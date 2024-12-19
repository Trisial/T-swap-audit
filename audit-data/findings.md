## Highs

### [H-1] `TSwapPool:getInputAmountBasedOnOutput` includes a different fee amount causing the protocol to collect higher amounts from users than initially stated, resulting in lost fees

**Description:** The function `TSwapPool:getInputAmountBasedOnOutput` indends to calculate the amount a user should deposit given the amount of tokens of output tokens. the return parameter of the function  returns this equation: `return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);` which multiplies by 10000 opposed to the previous calculations that were equal to 1000

**Impact:** It severely impacts the protocol as it completely changes the fee percentage compared to the one declared in the contract.

**Proof of Code**
<details>

    function testFlawedSwapExactOutput() public {
        uint256 initialLiquidity = 100e18;
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), initialLiquidity);
        poolToken.approve(address(pool), initialLiquidity);
        pool.deposit(initialLiquidity, 0, initialLiquidity, uint64(block.timestamp));
        vm.stopPrank();

        // User has 11 pool tokens
        address someUser = makeAddr("someUser");
        uint256 userInitialPoolTokenBalance = 11e18;
        poolToken.mint(someUser, userInitialPoolTokenBalance);
        vm.startPrank(someUser);

        // user buys one weth
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, 1 ether, uint64(block.timestamp));
        assertLt(poolToken.balanceOf(someUser), 1 ether);
        vm.stopPrank();

        vm.startPrank(liquidityProvider);
        pool.withdraw(initialLiquidity, 1, 1, uint64(block.timestamp));

        assertEq(weth.balanceOf(address(pool)), 0);
        assertEq(poolToken.balanceOf(address(pool)), 0);
    }
</details>
**Recommended Mitigation:** Consider setting constant variables to replace magic numbers, which would decrease the chances of possible typos from happening in the future.

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
-       return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
+       return ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
    }
    ```

### [H-2] Lack of slippage protection in `TSwapPool::swapExactOutput` causes users to potentially receive way fewer tokens

**Description:** `swapExactOutput` function does not include any sort of slippage protection. This function is similar to what is done in `TSwapPool::swapExactInput` where the function specifies `minOutputAmount`, the `swapExactOutput` function should specify `maxInputAmount`.

**Impact:** If market conditions change before the transaction processes, the user could get a much worse swap.

**Proof of Concept:**
1. The price of Weth is 1000 USDC
2. User in put `swapExactOutput` looking for 1 weth.
    1. inputToken = USDC
    2. outputToken = WETH
    3. outputAmount = 1
    4. deadline = whatever
3. The function does not allow a max input amount
4. As the transaction is pending in the mempool, the makret changes and the price of Weth moves to 2000 USDC
5. The transaction completes but the user sends 2000 USDC instead of expected 1000 USDC.

**Proof Of Code**
```javascript
    function testSwapExactOutputReceivesLessTokens() public {
        uint256 initialLiquidityPool = 10000e18;
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), initialLiquidityPool);
        poolToken.approve(address(pool), initialLiquidityPool);
        pool.deposit(initialLiquidityPool, 0, initialLiquidityPool, uint64(block.timestamp));
        vm.stopPrank();

        uint256 initialLiquidityWeth = 100e18;
        vm.startPrank(liquidityProvider);
        weth.approve(address(weth), initialLiquidityWeth);
        poolToken.approve(address(weth), initialLiquidityWeth);
        pool.deposit(initialLiquidityWeth, 0, initialLiquidityWeth, uint64(block.timestamp));
        vm.stopPrank();

        // Faster User has 5000 pool tokens
        address someUser = makeAddr("someUser");
        uint256 userInitialPoolTokenBalance = 5000e18;
        poolToken.mint(someUser, userInitialPoolTokenBalance);
        vm.startPrank(someUser);

        // user buys 100 weth
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, 100 ether, uint64(block.timestamp));
        assertLt(poolToken.balanceOf(someUser), 100 ether);
        vm.stopPrank();

        //The initial user wants to buy for the same amount
        address initialUser = makeAddr("initialUser");
        uint256 iUserInitialPoolTokenBalance = 10000e18;
        poolToken.mint(initialUser, iUserInitialPoolTokenBalance);
        vm.startPrank(initialUser);

        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, 100 ether, uint64(block.timestamp));
        assertLt(poolToken.balanceOf(initialUser), 100 ether);
        vm.stopPrank();

        assert(
            (userInitialPoolTokenBalance - poolToken.balanceOf(someUser))
                < (iUserInitialPoolTokenBalance - poolToken.balanceOf(initialUser))
        );
    }
```


**Recommended Mitigation:** 

```diff 
    function swapExactOutput(
        IERC20 inputToken,  
+       uint256 maxInputAmount
    inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);
+   if(inputAmount > maxInputAmount) {
    revert();
}    
+
```

### [H-3] `TSwapPoll:sellPoolTokens` mismatches the input and output tokens causing users to receive incorrect amount of tokens

**Description:** `sellPoolTokens` allows user to easily sell the pool tokens and receive Weth in exchange. Users indicate how many poolToken they want to sell in the `poolTokenAmount` parameter. However the function calculate wrongly due to the fact that the `swapExactOutput` is called instead of `swapExactInput`, because users specify exact amount of input tokens.

**Impact:** Users will swap wrong amount of tokens, which is a severe disruption 

**Proof of Concept:** Write POC

**Recommended Mitigation:** Consider changing the implementation to use `swapExactInput` instead of `swapExactOutput`. Note that this would require to change the `sellPoolTokens` function to accept a new parameter (`minwethToReceive` to be passed in `swapExactInput`)

```diff
    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint256 minWethToReceive,    
        ) external returns (uint256 wethAmount) {
-        return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+        return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minWethToReceive, uint64(block.timestamp));
    }
```

### [H-4] In `TSwapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invaraint `x * y = k`

**Description:** The invariant `x * y = k`
`x` - the balance of the pool token
`y` - the balance of Weth
`k` - The constant product of two balances

This means whenever the balance changes in the protocol, the ratio between two amounts should remain constant. However it is broken due to the extra incentive in the `_swap` function. Meaning that overtime the protocol funds will be drained.

The following block of code is responsible for this issue
```javascript
        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }
```

**Impact:** A user could maliciously drain the contract by swapping multiple times and collecting the extra incentive.

**Proof of Concept:**
1. A user swaps 10 times, and collects the extra incentive of 1_000_000_000_000_000_000 tokens
2. That user continues to swap untill all the protocol funds are drained

**Proof of Code**
<details>

```javascript
    function testInvariantBreaks() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputWeth = 1e17;

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        poolToken.mint(user, 100e18);
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));

        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));

        int256 actualDeltaY = int256(endingY) - int256(startingY);

        assertEq(actualDeltaY, expectedDeltaY);

    }

```

</details>

**Recommended Mitigation:** Remove the extra incentive, however if you want to leave it you should change the invariant

## Medium
### [M-1] `PoolFactory::deposit` is missing deadline check causing transaction to complete even after the deadline

**Description:** The `deposit` function accepts a deadline parameter, which accroding to natspec is: The deadline for the transaction to be completed by. Although in the code there is no deadline check the user is not time bound to a time limit.
As a consequence the operations that add liquidity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavorable

**Impact:** Transaction could be where the deposit rate is unfavorable to deposit, even when adding the deadline parameter

**Proof of Concept:** the `deadline` parameter is unused

**Recommended Mitigation:** Consider making the following change to the function:
```diff
    function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfDeadlinePassed(deadline)
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {}
```

## Lows

### [L-1] The event `LiquidityAdded` emitted is giving the wrong return/information when emitted

**Description:** The event passess the following parameters: `event LiquidityAdded(address indexed liquidityProvider, uint256 wethDeposited, uint256 poolTokensDeposited);` however in the `_addLiquidityMintAndTransfer` function the emit passess down in the following order, which is different `emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);`, which causes to return wrong information

**Impact:** It causes the event to return wrong information than settled in the initial event

**Recommended Mitigation:** 

```diff 
-   emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+   emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
```

### [L-2] Default value returned by `PoolSwapToken::swapExactInput` results in incorrect return value given

**Description:** `PoolSwapToken::swapExactInput` is expected to return the actual amount of tokkens bought by the caller. However while it declares the named return value `output` it is never assigned a valye nor uses an explicit return statement.

**Impact:**  The return value will always be 0, giving incorrect information to the caller.

**Proof of Concept:** Do it. (check no matter what we swap we always get 0)

**Recommended Mitigation:**  

```diff
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        // @audit-info this should be external
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (
            uint256 output
        )
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-       uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);
+       output = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);
       
-       if (outputAmount < minOutputAmount) {
-           revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
        }
+      if (output < minOutputAmount) {
+           revert TSwapPool__OutputTooLow(output, minOutputAmount);
        }

-       _swap(inputToken, inputAmount, outputToken, outputAmount);
+       _swap(inputToken, inputAmount, outputToken, output);

    }

```

## Informationals

### [I-1] `PoolFactory::error PoolFactory__PoolDoesNotExist(address tokenAddress)` is not used in the codebase

**Description:** The following error is not used anywhere in the codebase which increases the gas costs. It should be deleted

```diff
- error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-2] Lacking zero address checks

```diff
    constructor(address wethToken) {
+       if(wethToken == address(0)) {
+       revert();
+       }
        i_wethToken = wethToken;
    }
```

### [I-3] `PoolFactory::liquidityTokenSymbol` should be `.symbol()` instead of `.name()`

```diff
-   string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+   string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

### [I-4] Event is missing 'indexed' fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>4 Found Instances</summary>


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

</details>