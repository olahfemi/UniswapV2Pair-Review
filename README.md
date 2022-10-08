# UniswapV2Pair-Review

```solidity
1 pragma solidity =0.5.16;
```
Specifies the compiler that should be used

```solidity
3 import './interfaces/IUniswapV2Pair.sol';
4 import './UniswapV2ERC20.sol';
5 import './libraries/Math.sol';
6 import './libraries/UQ112x112.sol';
7 import './interfaces/IERC20.sol';
8 import './interfaces/IUniswapV2Factory.sol';
9 import './interfaces/IUniswapV2Callee.sol';
```
These are the contract, libraries and interfaces the contract inherits either because it uses them in some way or calls a contract that uses them

```solidity
11 contract UniswapV2Pair is IUniswapV2Pair, UniswapV2ERC20 {
```
IUniswapV2Pair is an interface that defines functions needed by the UniswapV2Pair contract. UniswapV2ERC20 implements the ERC-20 liquidity (LP) tokens. In this contract, it provides ERC-20 functions for them. 

```solidity
12 using SafeMath for uint;
```
This library is used in order to avoid overflows and underflows

```solidity
13 using UQ112x112 for uint224;
```
This is a library that helps to remedy the inability of the EVM to work with fractions 

#### Variables
```solidity
15 uint public constant MINIMUM_LIQUIDITY = 10**3;
```
This is a constant of one thousand liquidity tokens that are owned by address zero. It helps to prevent a case of division by zero.

```solidity
16 bytes4 private constant SELECTOR = bytes4(keccak256(bytes('transfer(address,uint256)')));
```
The selector of the ERC-20 transfer function is being computed in this line stored in the constant SELECTOR variable

```solidity
18 address public factory;
```
This variable captures the address of the factory contract that creates a pool

```solidity
19 address public token0;
20 address public token1;
```
These variables store the addresses for the two ERC-20 tokens that can be exchanged in a pool

```solidity
22 uint112 private reserve0;           // uses single storage slot, accessible via getReserves
23 uint112 private reserve1;           // uses single storage slot, accessible via getReserves
```
Stores the reserve of the pool for each token type

```solidity
24 uint32  private blockTimestampLast; // uses single storage slot, accessible via getReserves
```
Stores the timestamp for the last block where an exchange happened. Variable packing (ensuring they fit into the 256 storage slot) has been done to the last three variables to help with gas optimisation.

```solidity
26 uint public price0CumulativeLast;
27 uint public price1CumulativeLast;
```
They hold the cumulative prices of each token. Useful for calculating the average exchange rate per time.

```solidity
28 uint public kLast; // reserve0 * reserve1, as of immediately after the most recent liquidity event
```
This represents the constant that is used to determine the exchange rate between token pairs. It is determined by multiplying their reserves.

#### Lock 
This section talks about the function and variable used to perform a lock on functions, preventing them from being called while they are running (within the same transaction), in order to avoid reentrancy abuse

```solidity
30 uint private unlocked = 1;
```
The`unlocked` variable is initially set to true

```solidity
31 modifier lock() {
```
This is called a function is a modifier, it wraps around a normal function to change its behavior

```solidity
32 require(unlocked == 1, 'UniswapV2: LOCKED');
33 unlocked = 0;
```
If `unlocked` is equal to one, set it to zero. If it is already zero revert the call, make it fail

```solidity
34 _;
```
In a modifier `_;` is the original function call (with all the parameters). Here it means that the function call only happens if `unlocked` was one when it was called, and while it is running the value of `unlocked` is zero.

```solidity
35      unlocked = 1;
36 }
```
After the main function returns, release the lock

#### Miscellaneous functions

```solidity
38 function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
39      _reserve0 = reserve0;
40      _reserve1 = reserve1;
41      _blockTimestampLast = blockTimestampLast;
42 }
```

This function provides callers with the current state of the exchange
```solidity
44  function _safeTransfer(address token, address to, uint value) private {
45      (bool success, bytes memory data) = token.call(abi.encodeWithSelector(SELECTOR, to, value));
```
This internal function transfers an amount of ERC20 tokens from the exchange to somebody else. `SELECTOR` specifies that the function we are calling is `transfer(address,uint)`.

To avoid having to import an interface for the token function, the call was "manually" created using one of the ABI functions

```solidity
46 require(success && (data.length == 0 || abi.decode(data, (bool))), 'UniswapV2: TRANSFER_FAILED');
47 }
```
There are two ways in which an ERC-20 transfer call can report failure:

1. Revert. If a call to an external contract reverts, then the boolean return value is `false`
2. End normally but report a failure. In that case the return value buffer has a non-zero length, and when decoded as a boolean value it is `false`

If either of these conditions happen, revert.

#### Events 

```solidity
49 event Mint(address indexed sender, uint amount0, uint amount1);
50 event Burn(address indexed sender, uint amount0, uint amount1, address indexed to);
```

These two events are emitted when a liquidity provider either deposits liquidity (`Mint`) or withdraws it (`Burn`). In either case, the amounts of token0 and token1 that are deposited or withdrawn are part of the event, as well as the identity of the account that called us (`sender`). In the case of a withdrawal, the event also includes the target that received the tokens (`to`), which may not be the same as the sender.

```solidity
51 event Swap(
52      address indexed sender,
53      uint amount0In,
54      uint amount1In,
55      uint amount0Out,
56      uint amount1Out,
57      address indexed to
58 );
```
This event is emitted when a trader swaps one token for the other. Again, the sender and the destination may not be the same.
Each token may be either sent to the exchange, or received from it.

```solidity
59 event Sync(uint112 reserve0, uint112 reserve1);
```
Finally, `Sync` is emitted every time tokens are added or withdrawn, regardless of the reason, to provide the latest reserve information (and therefore the exchange rate).

#### Setup Functions

These functions are supposed to be called once when the new pair exchange is set up.

```solidity
61 constructor() public {
62      factory = msg.sender;
63 }
```
The constructor makes sure we'll keep track of the address of the factory that created the pair. This information is required for `initialize` and for the factory fee (if one exists)

```solidity
65 // called once by the factory at time of deployment
66 function initialize(address _token0, address _token1) external {
67      require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
68      token0 = _token0;
69      token1 = _token1;
70  }
```
This function allows the factory (and only the factory) to specify the two ERC-20 tokens that this pair will exchange.

#### Internal Update Functions

##### \_update

```solidity
72 // update reserves and, on the first call per block, price accumulators
73 function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
```
This function is called every time tokens are deposited or withdrawn.

```solidity
74 require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
```
If either balance0 or balance1 (uint256) is higher than uint112(-1) (=2^112-1) (so it overflows & wraps back to 0 when converted to uint112) refuse to continue the \_update to prevent overflows. With a normal token that can be subdivided into 10^18 units, this means each exchange is limited to about 5.1\*10^15 of each tokens. So far that has not been a problem.

```solidity
75 uint32 blockTimestamp = uint32(block.timestamp % 2**32);
76 uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
77 if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
```
If the time elapsed is not zero, it means we are the first exchange transaction on this block. In that case, we need to update the cost accumulators.

```solidity
78 // * never overflows, and + overflow is desired
79      price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
80      price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
81 }
```

Each cost accumulator is updated with the latest cost (reserve of the other token/reserve of this token) times the elapsed time in seconds. To get an average price you read the cumulative price is two points in time, and divide by the time difference between them. For example, assume this sequence of events:

| Event                                                    |  reserve0 |  reserve1 | timestamp | Marginal exchange rate (reserve1 / reserve0) |       price0CumulativeLast |
| -------------------------------------------------------- | --------: | --------: | --------- | -------------------------------------------: | -------------------------: |
| Initial setup                                            | 1,000.000 | 1,000.000 | 5,000     |                                        1.000 |                          0 |
| Trader A deposits 50 token0 and gets 47.619 token1 back  | 1,050.000 |   952.381 | 5,020     |                                        0.907 |                         20 |
| Trader B deposits 10 token0 and gets 8.984 token1 back   | 1,060.000 |   943.396 | 5,030     |                                        0.890 |       20+10\*0.907 = 29.07 |
| Trader C deposits 40 token0 and gets 34.305 token1 back  | 1,100.000 |   909.090 | 5,100     |                                        0.826 |    29.07+70\*0.890 = 91.37 |
| Trader D deposits 100 token1 and gets 109.01 token0 back |   990.990 | 1,009.090 | 5,110     |                                        1.018 |    91.37+10\*0.826 = 99.63 |
| Trader E deposits 10 token0 and gets 10.079 token1 back  | 1,000.990 |   999.010 | 5,150     |                                        0.998 | 99.63+40\*1.1018 = 143.702 |

Let's say we want to calculate the average price of **Token0** between the timestamps 5,030 and 5,150. The difference in the value of `price0Cumulative` is 143.702-29.07=114.632. This is the average across two minutes (120 seconds). So the average price is 114.632/120 = 0.955.

This price calculation is the reason we need to know the old reserve sizes.

```solidity
82      reserve0 = uint112(balance0);
83      reserve1 = uint112(balance1);
84      blockTimestampLast = blockTimestamp;
85      emit Sync(reserve0, reserve1);
86 }
```
Finally, update the global variables and emit a `Sync` event.

##### \_mintFee

```solidity
88 // if fee is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
89 function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
```
In Uniswap 2.0 traders pay a 0.30% fee to use the market. Most of that fee (0.25% of the trade) always goes to the liquidity providers. The remaining 0.05% can go either to the liquidity providers or to an address specified by the factory as a protocol fee, which pays Uniswap for their development effort.

To reduce calculations (and therefore gas costs), this fee is only calculated when liquidity is added or removed from the pool, rather than at each transaction.

```solidity
90 address feeTo = IUniswapV2Factory(factory).feeTo();
91 feeOn = feeTo != address(0);
```
Read the fee destination of the factory. If it is zero then there is no protocol fee and no need to calculate it.

```solidity
92 uint _kLast = kLast; // gas savings
```

The `kLast` state variable is located in storage, so it will have a value between different calls to the contract.
Access to storage is a lot more expensive than access to the volatile memory that is released when the function call to the contract ends, so we use an internal variable to save on gas.

```solidity
        if (feeOn) {
            if (_kLast != 0) {
```

The liquidity providers get their cut simply by the appreciation of their liquidity tokens. But the protocol fee requires new liquidity tokens to be minted and provided to the `feeTo` address.

```solidity
                uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
                uint rootKLast = Math.sqrt(_kLast);
                if (rootK > rootKLast) {
```

If there is new liquidity on which to collect a protocol fee. You can see the square root function [later in this article](#Math)

```solidity
                    uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                    uint denominator = rootK.mul(5).add(rootKLast);
                    uint liquidity = numerator / denominator;
```

This complicated calculation of fees is explained in [the whitepaper](https://uniswap.org/whitepaper.pdf) on page 5. We know that between the time `kLast` was calculated and the present no liquidity was added or removed (because we run this calculation every time liquidity is added or removed, before it actually changes), so any change in `reserve0 * reserve1` has to come from transaction fees (without them we'd keep `reserve0 * reserve1` constant).

```solidity
                    if (liquidity > 0) _mint(feeTo, liquidity);
                }
            }
```

Use the `UniswapV2ERC20._mint` function to actually create the additional liquidity tokens and assign them to `feeTo`.

```solidity
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }
```

If there is no fee set `kLast` to zero (if it isn't that already). When this contract was written there was a [gas refund feature](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-3298.md) that encouraged contracts to reduce the overall size of the Ethereum state by zeroing out storage they did not need.
This code gets that refund when possible.

#### Externally Accessible Functions {#pair-external}

Note that while any transaction or contract _can_ call these functions, they are designed to be called from the periphery contract. If you call them directly you won't be able to cheat the pair exchange, but you might lose value through a mistake.

##### mint

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function mint(address to) external lock returns (uint liquidity) {
```

This function is called when a liquidity provider adds liquidity to the pool. It mints additional liquidity tokens as a reward. It should be called from [a periphery contract](#UniswapV2Router02) that calls it after adding the liquidity in the same transaction (so nobody else would be able to submit a transaction that claims the new liquidity before the legitimate owner).

```solidity
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
```

This is the way to read the results of a Solidity function that returns multiple values. We discard the last returned values, the block timestamp, because we don't need it.

```solidity
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));
        uint amount0 = balance0.sub(_reserve0);
        uint amount1 = balance1.sub(_reserve1);
```

Get the current balances and see how much was added of each token type.

```solidity
        bool feeOn = _mintFee(_reserve0, _reserve1);
```

Calculate the protocol fees to collect, if any, and mint liquidity tokens accordingly. Because the parameters to `_mintFee` are the old reserve values, the fee is calculated accurately based only on pool changes due to fees.

```solidity
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
           _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
```

If this is the first deposit, create `MINIMUM_LIQUIDITY` tokens and send them to address zero to lock them. They can never be redeemed, which means the pool will never be emptied completely (this saves us from division by zero in some places). The value of `MINIMUM_LIQUIDITY` is a thousand, which considering most ERC-20 are subdivided into units of 10^-18'th of a token, as ETH is divided into wei, is 10^-15 to the value of a single token. Not a high cost.

In the time of the first deposit we don't know the relative value of the two tokens, so we just multiply the amounts and take a square root, assuming that the deposit provides us with equal value in both tokens.

We can trust this because it is in the depositor's interest to provide equal value, to avoid losing value to arbitrage.
Let's say that the value of the two tokens is identical, but our depositor deposited four times as many of **Token1** as of **Token0**. A trader can use the fact the pair exchange thinks that **Token0** is more valuable to extract value out of it.

| Event                                                        | reserve0 | reserve1 | reserve0 \* reserve1 | Value of the pool (reserve0 + reserve1) |
| ------------------------------------------------------------ | -------: | -------: | -------------------: | --------------------------------------: |
| Initial setup                                                |        8 |       32 |                  256 |                                      40 |
| Trader deposits 8 **Token0** tokens, gets back 16 **Token1** |       16 |       16 |                  256 |                                      32 |

As you can see, the trader earned an extra 8 tokens, which come from a reduction in the value of the pool, hurting the depositor that owns it.

```solidity
        } else {
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
```

With every subsequent deposit we already know the exchange rate between the two assets, and we expect liquidity providers to provide equal value in both. If they don't, we give them liquidity tokens based on the lesser value they provided as a punishment.

Whether it is the initial deposit or a subsequent one, the number of liquidity tokens we provide is equal to the square root of the change in `reserve0*reserve1` and the value of the liquidity token doesn't change (unless we get a deposit that doesn't have equal values of both types, in which case the "fine" gets distributed). Here is another example with two tokens that have the same value, with three good deposits and one bad one (deposit of only one token type, so it doesn't produce any liquidity tokens).

| Event                     | reserve0 | reserve1 | reserve0 \* reserve1 | Pool value (reserve0 + reserve1) | Liquidity tokens minted for this deposit | Total liquidity tokens | value of each liquidity token |
| ------------------------- | -------: | -------: | -------------------: | -------------------------------: | ---------------------------------------: | ---------------------: | ----------------------------: |
| Initial setup             |    8.000 |    8.000 |                   64 |                           16.000 |                                        8 |                      8 |                         2.000 |
| Deposit four of each type |   12.000 |   12.000 |                  144 |                           24.000 |                                        4 |                     12 |                         2.000 |
| Deposit two of each type  |   14.000 |   14.000 |                  196 |                           28.000 |                                        2 |                     14 |                         2.000 |
| Unequal value deposit     |   18.000 |   14.000 |                  252 |                           32.000 |                                        0 |                     14 |                        ~2.286 |
| After arbitrage           |  ~15.874 |  ~15.874 |                  252 |                          ~31.748 |                                        0 |                     14 |                        ~2.267 |

```solidity
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        _mint(to, liquidity);
```

Use the `UniswapV2ERC20._mint` function to actually create the additional liquidity tokens and give them to the correct account.

```solidity

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```

Update the state variables (`reserve0`, `reserve1`, and if needed `kLast`) and emit the appropriate event.

##### burn

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function burn(address to) external lock returns (uint amount0, uint amount1) {
```

This function is called when liquidity is withdrawn and the appropriate liquidity tokens need to be burned.
It should also be called [from a periphery account](#UniswapV2Router02).

```solidity
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        address _token0 = token0;                                // gas savings
        address _token1 = token1;                                // gas savings
        uint balance0 = IERC20(_token0).balanceOf(address(this));
        uint balance1 = IERC20(_token1).balanceOf(address(this));
        uint liquidity = balanceOf[address(this)];
```

The periphery contract transferred the liquidity to be burned to this contract before the call. That way we know how much liquidity to burn, and we can make sure that it gets burned.

```solidity
        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        amount0 = liquidity.mul(balance0) / _totalSupply; // using balances ensures pro-rata distribution
        amount1 = liquidity.mul(balance1) / _totalSupply; // using balances ensures pro-rata distribution
        require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
```

The liquidity provider receives equal value of both tokens. This way we don't change the exchange rate.

```solidity
        _burn(address(this), liquidity);
        _safeTransfer(_token0, to, amount0);
        _safeTransfer(_token1, to, amount1);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Burn(msg.sender, amount0, amount1, to);
    }

```

The rest of the `burn` function is the mirror image of the `mint` function above.

##### swap

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
```

This function is also supposed to be called from [a periphery contract](#UniswapV2Router02).

```solidity
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        { // scope for _token{0,1}, avoids stack too deep errors
```

Local variables can be stored either in memory or, if there aren't too many of them, directly on the stack.
If we can limit the number so we'll use the stack we use less gas. For more details see [the yellow paper, the formal Ethereum specifications](https://ethereum.github.io/yellowpaper/paper.pdf), p. 26, equation 298.

```solidity
            address _token0 = token0;
            address _token1 = token1;
            require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
```

This transfer is optimistic, because we transfer before we are sure all the conditions are met. This is OK in Ethereum because if the conditions aren't met later in the call we revert out of it and any changes it created.

```solidity
            if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
```

Inform the receiver about the swap if requested.

```solidity
            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }
```

Get the current balances. The periphery contract sends us the tokens before calling us for the swap. This makes it easy for the contract to check that it is not being cheated, a check that _has_ to happen in the core contract (because we can be called by other entities than our periphery contract).

```solidity
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
            uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
            require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
```

This is a sanity check to make sure we don't lose from the swap. There is no circumstance in which a swap should reduce `reserve0*reserve1`. This is also where we ensure a fee of 0.3% is being sent on the swap; before sanity checking the value of K, we multiply both balances by 1000 subtracted by the amounts multiplied by 3, this means 0.3% (3/1000 = 0.003 = 0.3%) is being deducted from the balance before comparing its K value with the current reserves K value.

```solidity
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```

Update `reserve0` and `reserve1`, and if necessary the price accumulators and the timestamp and emit an event.

##### Sync or Skim {#sync-or-skim}

It is possible for the real balances to get out of sync with the reserves that the pair exchange thinks it has.
There is no way to withdraw tokens without the contract's consent, but deposits are a different matter. An account can transfer tokens to the exchange without calling either `mint` or `swap`.

In that case there are two solutions:

- `sync`, update the reserves to the current balances
- `skim`, withdraw the extra amount. Note that any account is allowed to call `skim` because we don't know who deposited the tokens. This information is emitted in an event, but events are not accessible from the blockchain.

```solidity
    // force balances to match reserves
    function skim(address to) external lock {
        address _token0 = token0; // gas savings
        address _token1 = token1; // gas savings
        _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
        _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
    }



    // force reserves to match balances
    function sync() external lock {
        _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)), reserve0, reserve1);
    }
}
```





