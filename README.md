# UniswapV2Pair-Review

```
pragma solidity =0.5.16;
```
Specifies the compiler that should be used

```
import './interfaces/IUniswapV2Pair.sol';
import './UniswapV2ERC20.sol';
import './libraries/Math.sol';
import './libraries/UQ112x112.sol';
import './interfaces/IERC20.sol';
import './interfaces/IUniswapV2Factory.sol';
import './interfaces/IUniswapV2Callee.sol';
```
These are the contract, libraries and interfaces the contract inherits either because it uses them in some way or calls a contract that uses them

```
contract UniswapV2Pair is IUniswapV2Pair, UniswapV2ERC20 {
```
IUniswapV2Pair is an interface that defines functions needed by the UniswapV2Pair contract. UniswapV2ERC20 implements the ERC-20 liquidity (LP) tokens. In this contract, it provides ERC-20 functions for them. 
