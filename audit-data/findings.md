## High
### [H-1] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput` causes protocol to take too many tokens from users, resulting in lost fees.

**Description:** The `getInputAmountBasedOnOutput` function is intended to calculate the amount of tokens a user should deposit given an amount of tokens of output tokens. however , the function currently miscalcuate the resulting amount. when calculating the fee, it scales the amount by **10_000** instead of **1_000** 

**Impact:** Protocol takes more fees than expected from users

**Recommended Mitigation:**
```diff
 function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-       return ((inputReserves * outputAmount) * 10_000) / ((outputReserves - outputAmount) *       997);
+       return ((inputReserves * outputAmount) * 1_000) / ((outputReserves - outputAmount) * 997);
    }
```

### [H-2] Lack of slippage protection in `TSwapPool::swapExactOutput` causes users to potentially receive way fewer tokens

**Description:** The `swapExactOutput` function does not include any sort of slippage protection. This function is similar to what is done in `TswapPool::swapExactInput` where the function specifies a `minOutputAmount`, the `swapExactOutput` function should specify a `maxInputAmount`.

**Impact:** if market conditions change before the transaction processes, the user could get a much worse swap.

**Proof Of Concept:**
1. The price of 1 WETH right now is 1000 USDC
2. User inputs a `swapExactOutput` looking for 1 WETH
   1. inputToken = USDC
   2. outputToken = WETH
   3. outputAmount = 1
   4. dealine = whatever
3. The function does not offer a maxInput amount
4. As the transcation is pending in the mempool, the market changes! and the price moves HUGE -> 1 WETH is now 10,000 USDC. 10x more than the user expected.
5. The transcation completes, but the user sent the protocol 10,000 USDC instead of the expected 1,000 USDC. 

**Recommended Mitigation:** We should include a `maxInputAmount` so the user only has to spend up to a specific amount, and can predict how much they will spent on the protocol.

```diff
    function swapExactOutput(
        IERC20 inputToken,
+       uint256 maxInputAmount,
.
.
.
    inputAmount = getInputAmountBasedOnOutput(
            outputAmount,
            inputReserves,
            outputReserves
        );

+   if(inputAmount > maxInputAmount){
+    revert()
+   }    
```

### [H-3] `TSwapPool::sellPoolTokens` mismatches input and output tokens causing users to receive the incorrect amount of tokens

**Description:** The `sellPoolTokens` function is intended to allow users to easily sell pool tokens and receive WETH in exchnage. Users indicate how many pool tokens they are willing to sell in the `poolTokenAmout` parameter. However, the function currently miscalculates the swapped amount.

This is due to the fact that the `swapExactOutput` functio is called, whereas the `swapExactInput` function is the one that should be called. Beacuse users specify the exact amount of input tokens, not output tokens.

**Impact:** Users will swap the wrong amount of tokens, which is a severe disruption of protocol functionality.

**Proof Of Concept:** 

**Recommended Mitigation:**
Consider changing the implementation to use `swapExactInput` instead of `swapExactOutput`. Note that this would also require changing the `sellPoolTokens` function to accept a new parameter (i.e `minWethToReceive` to be passed to `swapExactInput`)

```diff
    function sellPoolTokens(uint256 poolTokenAmount) external returns (uint256 wethAmount) {
-        return swapExactOutput(i_poolToken,i_wethToken,poolTokenAmount,uint64(block.timestamp));
+        return swapExactInput(i_poolToken,poolTokenAmount,i_wethToken,minWethToReceive,uint64(block.timestamp));
    }
```

Additinally, it might be wise to add a deadline to the function, as there is currently NO deadline. 

## Medium
### [M-1] `TswapPool::deposit` is missing deadline chcek causing transcations to complete even after the deadline

**Description:** The `deposit` function accepts a deadline parameter, which according to the documentation is "The deadline for the transaction to be completed by". However, this parameter is never used. As a consequence, oprators that add liquidity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavorable.

<!-- MEV attacks -->

**Impact:** Transactions could be sent when market conditions are unfavorable to deposit, even when adding a deadline parameter.

**Proof Of Concept:** The `deadline` parameter is unused.

**Recommended Mitigation:** Consider making the following change to the function.

```diff
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint, // LP Token to mint
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfDead(deadline)
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {
```

## Low

### [L-1] `TSwapPool::LiquidityAdded` event has parameters out of order causing event to emit incorrect information

**Description:** When the `LiquidityAdded` event is emitted in the `TSwapPool::_addLiquidityMintAndTransfer` function, it logs values in an incorrect order. The `poolTokensToDeposit` value should go in the third parameter position, whereas the `wethToDeposit` value should go second.

**Impact:** Event emission is incorrect, leading to off-chain functions potentially malfunctioning.

**Recommended Mitigation:**
```diff
- emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+ emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
```

### [L-2] Default value returned by `TSwapPool::swapExactInput` results in incorrect return value given

**Description:** The `swapExactInput` function is expected to return the actual amount of tokens bought by the caller. However, while it declares the named return value `output` it is never assigned a value, nor uses an explict return statement.

**Impact:** The return value will always be 0, giving incorrect information to the caller.

**Recommended Mitigation:** 

```diff
    uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-        uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount,inputReserves,
-            outputReserves);

+        output = getOutputAmountBasedOnInput(inputAmount,inputReserves,
+            outputReserves);

-        if (outputAmount < minOutputAmount) {
-            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
-        }

+        if (output < minOutputAmount) {
+            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
+        }

-        _swap(inputToken, inputAmount, outputToken, outputAmount);
+        _swap(inputToken, inputAmount, outputToken, output);
```


## Informationals

### [I-1] `PoolFactory::PoolFactory__PoolDoesNotExist` is not used and should be removed

```diff
- error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-2] Lacking zero address checks 

```diff
    constructor(address wethToken) {
+        if(wethToken == address(0)){
+            revert();
+        }
        i_wethToken = wethToken;
    }
```

### [I-3] `PoolFactory::createPool` should use `.symbol()` instead of `name()`
```diff
-    string memory liquidityTokenSymbol = string.concat(
-            "ts",
-            IERC20(tokenAddress).name()
-        );

+    string memory liquidityTokenSymbol = string.concat(
+            "ts",
+            IERC20(tokenAddress).symbol()
+        );
```

### [I-4] Event is missing `indexed` fields

**Description:**
Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

- Found in src/PoolFactory.sol [Line: 35](src/PoolFactory.sol#L35)

	```solidity
	    event PoolCreated(address tokenAddress, address poolAddress);
	```

- Found in src/TSwapPool.sol [Line: 43](src/TSwapPool.sol#L43)

	```solidity
	    event LiquidityAdded(address indexed liquidityProvider, uint256 wethDeposited, uint256 poolTokensDeposited);
	```

- Found in src/TSwapPool.sol [Line: 44](src/TSwapPool.sol#L44)

	```solidity
	    event LiquidityRemoved(address indexed liquidityProvider, uint256 wethWithdrawn, uint256 poolTokensWithdrawn);
	```

- Found in src/TSwapPool.sol [Line: 45](src/TSwapPool.sol#L45)

	```solidity
	    event Swap(address indexed swapper, IERC20 tokenIn, uint256 amountTokenIn, IERC20 tokenOut, uint256 amountTokenOut);
	```



### [S-#] TITLE (Root cause + Impact)

**Description:**

**Impact:**

**Proof Of Concept:**

**Recommended Mitigation:**

### [S-#] TITLE (Root cause + Impact)

**Description:**

**Impact:**

**Proof Of Concept:**

**Recommended Mitigation:**