# QuillCTF : Collect

## Objective of CTF

This CTF task is about Uniswap v3. You are defiMaster.

You have 1 LP from the owner.
At the end of the test, you collect commission from your LP.
But it's too small for you. Increase the amount of commission, which you can get.

In the given contract code, there is compile error :
```
Compiler run failed:
Error (7576): Undeclared identifier.
   --> test/collect.t.sol:122:9:
    |
122 |         paramsRemoveLiq = INonfungiblePositionManager.DecreaseLiquidityParams({
    |         ^^^^^^^^^^^^^^^

Error (7576): Undeclared identifier.
   --> test/collect.t.sol:129:9:
    |
129 |         collectParams = INonfungiblePositionManager.CollectParams({
    |         ^^^^^^^^^^^^^

Error (7576): Undeclared identifier.
   --> test/collect.t.sol:137:13:
    |
137 |             collectParams
    |             ^^^^^^^^^^^^^
```

So I changed those lines and fixed it

```solidity
        INonfungiblePositionManager.DecreaseLiquidityParams memory paramsRemoveLiq = INonfungiblePositionManager.DecreaseLiquidityParams({
            tokenId: defiMasterLP,
            liquidity: defiMasterLiquidity,
            amount0Min: 0,
            amount1Min: 0,
            deadline: block.timestamp
        });
        INonfungiblePositionManager.CollectParams memory collectParams = INonfungiblePositionManager.CollectParams({
            tokenId: defiMasterLP,
            recipient: defiMaster,
            amount0Max: type(uint128).max,
            amount1Max: type(uint128).max
        });
```

UniswapV3 uses ERC721 tokens as lp tokens and NonfungiblePositionManager to manage the tokens

The owner created a UniswapV3 pool DogToken and CatToken with 0.3% fee, then he add liquidity by minting 2 lp tokens on NonfungiblePositionManager

```solidity
        INonfungiblePositionManager.MintParams
            memory params = INonfungiblePositionManager.MintParams({
                token0: address(DogToken),
                token1: address(CatToken),
                fee: poolFee,
                tickLower: -887220,
                tickUpper: 887220,
                amount0Desired: 1000 ether,
                amount1Desired: 1000 ether,
                amount0Min: 0,
                amount1Min: 0,
                recipient: owner,
                deadline: block.timestamp
            });

        nonfungiblePositionManager.mint(params);
```

He minted 2 lp tokens, each of them having 1000 ether of DogToken and CatToken, and he sent one of the lp token to us, so one of the lp token is now owned by the owner and the another lp token is owned by us, so we own 50% of the total liquidity

Then the user will perform a swap on the pool, swapping 100 ether of CatToken to DogToken, and there is 0.3% fee on the pool, which around 0.3 ether of the token is used as fee for the liquidity provider to collect

If we do nothing, we will get around 0.15 ether, which is half of the fee collected, as we own 50% of the liquidity, and the owner owns another 50%

Our goal is to collect more than 298214374191364123 tokens, which is nearly all of the fee collected, but we only own 50% of the liquidity

Uniswap introduced concentrated liquidity in V3, and when the owner mint the 2 lp tokens, he provided liquidity for the full price range (-887220, 887220), which is just like providing liquidity on UniswapV2, most of the provided liquidity is not used by the swap

https://ethereum.stackexchange.com/questions/144793/why-does-uniswap-v3-use-ticks-887220-887220-to-represent-the-price-range-0-%E2%88%9E

The price is represented by the variable `sqrtPriceX96` and `sqrtPriceX96` is type `uint160` 

![](https://i.stack.imgur.com/d1zLz.png)

![](https://i.stack.imgur.com/eFNBs.png)

This is the maximum value of `sqrtPriceX96` :
```
 »  type(uint160).max
1461501637330902918203684832716283019655932542975

>>> 2**160-1
1461501637330902918203684832716283019655932542975
```

So, by solving the equation the maximum value of price is around `2**((160-96)*2)` which is around `340282366920938463463374607431768211456`

And the equation of the price and tick (i) is this :

![](https://i.stack.imgur.com/x8smF.png)

So the maximum value of tick can be roughly calculated with this :

```python
>>> math.log(2**((160-96)*2), (1.0001))
887272.7517970635
```


Also, the pool has 0.3% fee, according to the whitepaper, it has a tick spacing of 60 :
https://uniswap.org/whitepaper-v3.pdf
```
The initial fee tiers and tick spacings supported are 0.05% (with a tick spacing of 10, approximately 0.10% between initializable ticks), 0.30% (with a tick spacing of 60, approximately 0.60% between initializable ticks), and 1% (with a tick spacing of 200, approximately 2.02% between ticks.
```

In solidity, there is no decimal places also the pool is having 0.3% fee which has 60 tick spacing, so the maximum of that pool can be calculated with this :

```python
>>> (math.log(2**((160-96)*2), (1.0001))) - (math.log(2**((160-96)*2), (1.0001)) % 60)
887220.0
```

That's why (-887220, 887220) represent the full price range

So we can remove the liquidity that we own and mint a new lp token that has concentrated liquidity, first I use console.log to see the tick changed on the swap, before the swap the tick is 0 and after the swap it's 5

Then, we can provide all of our liquidity on the range 0,60, so all of our liquidity will be concentrated on the range that the user's swap use, and most of the fee collected will be owned by us, as the owner's liquidity is diluted over the full range


## Proof of concept

### Foundry test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8;

import "forge-std/Test.sol";
import {ERC20} from "openzeppelin-contracts/token/ERC20/ERC20.sol";
import {INonfungiblePositionManager} from "v3-periphery/interfaces/INonfungiblePositionManager.sol";
import {TickMath} from "v3-core/libraries/TickMath.sol";
import {IUniswapV3Factory} from "v3-core/interfaces/IUniswapV3Factory.sol";
import {IUniswapV3Pool} from "v3-core/interfaces/IUniswapV3Pool.sol";
import {ISwapRouter} from "v3-periphery/interfaces/ISwapRouter.sol";

// install:
// forge install Openzeppelin/openzeppelin-contracts
// forge install Uniswap/v3-periphery@0.8
// forge install Uniswap/v3-core@0.8

contract Token is ERC20 {
    constructor(
        string memory name,
        string memory symbol,
        uint initialMint
    ) ERC20(name, symbol) {
        _mint(msg.sender, initialMint);
    }
}

contract NFTRent is Test {
    Token DogToken;
    Token CatToken;
    uint tokenId1;
    uint defiMasterLP;
    uint128 defiMasterLiquidity;
    uint liquidity;
    address owner = makeAddr("owner");
    address defiMaster = makeAddr("defiMaster");
    address user = makeAddr("user");
    uint24 poolFee = 3000;
    IUniswapV3Pool pool;
    ISwapRouter router =
        ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
    INonfungiblePositionManager nonfungiblePositionManager =
        INonfungiblePositionManager(0xC36442b4a4522E871399CD717aBDD847Ab11FE88);
    IUniswapV3Factory UNISWAP_FACTORY =
        IUniswapV3Factory(0x1F98431c8aD98523631AE4a59f267346ea31F984);

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth");

        vm.startPrank(owner);
        DogToken = new Token("DogToken", "DogToken", 1000000 ether);
        CatToken = new Token("CatToken", "CatToken", 1000000 ether);
        // gift from owner to user
        DogToken.transfer(user, 10000 ether);
        CatToken.transfer(user, 10000 ether);
        // owner lp
        nonfungiblePositionManager.createAndInitializePoolIfNecessary(
            address(DogToken),
            address(CatToken),
            3000,
            1 << 96
        );

        pool = IUniswapV3Pool(
            UNISWAP_FACTORY.getPool(address(DogToken), address(CatToken), 3000)
        );
        DogToken.approve(address(nonfungiblePositionManager), 10000 ether);
        CatToken.approve(address(nonfungiblePositionManager), 10000 ether);
        INonfungiblePositionManager.MintParams
            memory params = INonfungiblePositionManager.MintParams({
                token0: address(DogToken),
                token1: address(CatToken),
                fee: poolFee,
                tickLower: -887220,
                tickUpper: 887220,
                amount0Desired: 1000 ether,
                amount1Desired: 1000 ether,
                amount0Min: 0,
                amount1Min: 0,
                recipient: owner,
                deadline: block.timestamp
            });

        nonfungiblePositionManager.mint(params);

        (defiMasterLP, defiMasterLiquidity, , ) = nonfungiblePositionManager
            .mint(params);

        // owner send to defiMaster LP 721 token
        nonfungiblePositionManager.safeTransferFrom(
            owner,
            defiMaster,
            defiMasterLP
        );
        vm.stopPrank();
    }

 
    function test_solution() public {
        // solution
        vm.startPrank(defiMaster);
        
        console.log("defiMasterLP :", defiMasterLP);
        console.log("defiMasterLiquidity :", defiMasterLiquidity);
        
        INonfungiblePositionManager.DecreaseLiquidityParams memory decreaseLiquidityParams = INonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: defiMasterLP,
                liquidity: defiMasterLiquidity,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp
            });
        nonfungiblePositionManager.decreaseLiquidity(decreaseLiquidityParams);
        
        console.log("DogToken, CatToken balance of defiMaster before collect :", DogToken.balanceOf(defiMaster), CatToken.balanceOf(defiMaster));
        
        INonfungiblePositionManager.CollectParams memory _collectParams = INonfungiblePositionManager.CollectParams({
            tokenId: defiMasterLP,
            recipient: defiMaster,
            amount0Max: type(uint128).max,
            amount1Max: type(uint128).max
        });

        nonfungiblePositionManager.collect(_collectParams);
        
        
        console.log("DogToken, CatToken balance of defiMaster after collect :", DogToken.balanceOf(defiMaster), CatToken.balanceOf(defiMaster));
        
        DogToken.approve(address(nonfungiblePositionManager), type(uint256).max);
        CatToken.approve(address(nonfungiblePositionManager), type(uint256).max);
        
        INonfungiblePositionManager.MintParams memory mintParams = INonfungiblePositionManager.MintParams({
                token0: address(DogToken),
                token1: address(CatToken),
                fee: poolFee,
                tickLower: 0,
                tickUpper: 60,
                amount0Desired: DogToken.balanceOf(defiMaster),
                amount1Desired: CatToken.balanceOf(defiMaster),
                amount0Min: 0,
                amount1Min: 0,
                recipient: defiMaster,
                deadline: block.timestamp
            });
        (defiMasterLP, defiMasterLiquidity, , ) = nonfungiblePositionManager.mint(mintParams);
        
        
        vm.stopPrank();
        // end solution
        // --------------------------------------------------------

        vm.startPrank(user);
        CatToken.approve(address(router), 100 ether);
        DogToken.approve(address(router), 100 ether);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: address(CatToken),
                tokenOut: address(DogToken),
                fee: 3000,
                recipient: user,
                deadline: block.timestamp,
                amountIn: 100 ether,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
        router.exactInputSingle(params);
        vm.stopPrank();

        vm.startPrank(defiMaster);
        INonfungiblePositionManager.DecreaseLiquidityParams memory paramsRemoveLiq = INonfungiblePositionManager.DecreaseLiquidityParams({
            tokenId: defiMasterLP,
            liquidity: defiMasterLiquidity,
            amount0Min: 0,
            amount1Min: 0,
            deadline: block.timestamp
        });
        INonfungiblePositionManager.CollectParams memory collectParams = INonfungiblePositionManager.CollectParams({
            tokenId: defiMasterLP,
            recipient: defiMaster,
            amount0Max: type(uint128).max,
            amount1Max: type(uint128).max
        });

        (, uint collectAmount1) = nonfungiblePositionManager.collect(
            collectParams
        );

        assertGt(collectAmount1, 298214374191364123);
        vm.stopPrank();
    }
}
```

### Foundry test output

```
# forge test --match-path test/collect.t.sol -vv
[⠰] Compiling...
No files changed, compilation skipped

Running 1 test for test/collect.t.sol:NFTRent
[PASS] test_solution() (gas: 778439)
Logs:
  defiMasterLP : 512956
  defiMasterLiquidity : 1000000000000000000054
  DogToken, CatToken balance of defiMaster before collect : 0 0
  DogToken, CatToken balance of defiMaster after collect : 999999999999999999999 999999999999999999999

Test result: ok. 1 passed; 0 failed; finished in 12.40s
```