# UniswapV2Pair-Review

```
1 pragma solidity =0.5.16;
```
Specifies the compiler that should be used

```
3 import './interfaces/IUniswapV2Pair.sol';
4 import './UniswapV2ERC20.sol';
5 import './libraries/Math.sol';
6 import './libraries/UQ112x112.sol';
7 import './interfaces/IERC20.sol';
8 import './interfaces/IUniswapV2Factory.sol';
9 import './interfaces/IUniswapV2Callee.sol';
```
These are the contract, libraries and interfaces the contract inherits either because it uses them in some way or calls a contract that uses them

```
11 contract UniswapV2Pair is IUniswapV2Pair, UniswapV2ERC20 {
```
IUniswapV2Pair is an interface that defines functions needed by the UniswapV2Pair contract. UniswapV2ERC20 implements the ERC-20 liquidity (LP) tokens. In this contract, it provides ERC-20 functions for them. 

```
12 using SafeMath for uint;
```
This library is used in order to avoid overflows and underflows

```
13 using UQ112x112 for uint224;
```
This is a library that helps to remedy the inability of the EVM to work with fractions 

## Variables
```
15 uint public constant MINIMUM_LIQUIDITY = 10**3;
```
This is a constant of one thousand liquidity tokens that are owned by address zero. It helps to prevent a case of division by zero.

```
16 bytes4 private constant SELECTOR = bytes4(keccak256(bytes('transfer(address,uint256)')));
```
The selector of the ERC-20 transfer function is being computed in this line stored in the constant SELECTOR variable

```
18 address public factory;
```
This variable captures the address of the factory contract that creates a pool

```
19 address public token0;
20 address public token1;
```
These variables store the addresses for the two ERC-20 tokens that can be exchanged in a pool

```
22 uint112 private reserve0;           // uses single storage slot, accessible via getReserves
24 uint112 private reserve1;           // uses single storage slot, accessible via getReserves
```
Stores the reserve of the pool for each token type

```
25 uint32  private blockTimestampLast; // uses single storage slot, accessible via getReserves
```
Stores the timestamp for the last block where an exchange happened. Variable packing (ensuring they fit into the 256 storage slot) has been done to the last three variables to help with gas optimisation.




