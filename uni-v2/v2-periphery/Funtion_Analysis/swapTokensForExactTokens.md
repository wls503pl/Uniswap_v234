# SwapTokensForExactTokens Function Analysis

## Function Signature

```solidity
function swapTokensForExactTokens(
    uint amountOut,
    uint amountInMax,
    address[] calldata path,
    address to,
    uint deadline
) external virtual override ensure(deadline) returns (uint[] memory amounts)
```

## Overview

**Purpose**: Swap tokens to receive an exact amount of output tokens, with a maximum input amount limit.

**Key Difference from swapExactTokensForTokens**:

-   Input: You specify **output amount** (amountOut)
-   Output: Function calculates **required input amount**
-   Works backwards: output → input

**User Scenario**: "I need exactly 100 USDC. What's the maximum DAI I should spend?"

---

## Function Parameters

| Parameter     | Type        | Purpose                                    |
| ------------- | ----------- | ------------------------------------------ |
| `amountOut`   | `uint`      | Exact amount of output tokens user wants   |
| `amountInMax` | `uint`      | Maximum input tokens user willing to spend |
| `path`        | `address[]` | Swap route (e.g., [DAI, WETH, USDC])       |
| `to`          | `address`   | Recipient address for output tokens        |
| `deadline`    | `uint`      | Block timestamp deadline                   |

---

## Execution Flow

### Step 1: Validate Deadline

```solidity
ensure(deadline)
```

Checks current block timestamp ≤ deadline. Prevents stale transactions.

### Step 2: Calculate Required Input Amounts

```solidity
amounts = UniswapV2Library.getAmountsIn(factory, amountOut, path);
```

**How `getAmountsIn` works**:

```solidity
function getAmountsIn(address factory, uint amountOut, address[] memory path)
    internal view returns (uint[] memory amounts)
{
    require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
    amounts = new uint[](path.length);
    amounts[amounts.length - 1] = amountOut;  // Start with desired output

    // Work backwards through pairs
    for (uint i = path.length - 1; i > 0; i--) {
        (uint reserveIn, uint reserveOut) = getReserves(factory, path[i - 1], path[i]);
        amounts[i - 1] = getAmountIn(amounts[i], reserveIn, reserveOut);
    }
}
```

**Logic**:

1. Initialize last element with amountOut
2. Loop backwards through the path
3. For each pair, calculate input needed to get the output amount
4. Formula: `amountIn = (amountOut * reserveIn * 1000) / (reserveOut * 997)`

**Example** (path = [DAI, WETH, USDC], amountOut = 100 USDC):

```
Iteration 1: amounts[1] = getAmountIn(100 USDC, WETH_reserve, USDC_reserve)
            Result: 0.5 WETH needed to get 100 USDC

Iteration 2: amounts[0] = getAmountIn(0.5 WETH, DAI_reserve, WETH_reserve)
            Result: 150 DAI needed to get 0.5 WETH

Final: amounts = [150 DAI, 0.5 WETH, 100 USDC]
```

### Step 3: Verify Slippage Protection

```solidity
require(amounts[0] <= amountInMax, 'UniswapV2Router: EXCESSIVE_INPUT_AMOUNT');
```

Ensures calculated input amount doesn't exceed user's maximum acceptable amount.

**Why**: Protects against price slippage. If market price moves unfavorably and requires more input than `amountInMax`, transaction reverts.

### Step 4: Transfer Input Tokens to First Pair

```solidity
TransferHelper.safeTransferFrom(
    path[0],
    msg.sender,
    UniswapV2Library.pairFor(factory, path[0], path[1]),
    amounts[0]
);
```

Transfers input tokens from user to the first pair contract. Required before swap execution.

### Step 5: Execute Swaps

```solidity
_swap(amounts, path, to);
```

**How `_swap` works**:

```solidity
function _swap(uint[] memory amounts, address[] memory path, address _to)
    internal virtual
{
    for (uint i; i < path.length - 1; i++) {
        (address input, address output) = (path[i], path[i + 1]);
        (address token0,) = UniswapV2Library.sortTokens(input, output);
        uint amountOut = amounts[i + 1];

        // Determine amount0Out and amount1Out based on token ordering
        (uint amount0Out, uint amount1Out) = input == token0
            ? (uint(0), amountOut)
            : (amountOut, uint(0));

        // Route: intermediate swaps → next pair, last swap → recipient
        address to = i < path.length - 2
            ? UniswapV2Library.pairFor(factory, output, path[i + 2])
            : _to;

        // Execute swap on pair contract
        IUniswapV2Pair(
            UniswapV2Library.pairFor(factory, input, output)
        ).swap(amount0Out, amount1Out, to, new bytes(0));
    }
}
```

**Execution for path = [DAI, WETH, USDC]**:

**Swap 1**: DAI/WETH pair

-   Input: 150 DAI (already in pair)
-   Output: 0.5 WETH
-   Send to: WETH/USDC pair (not final recipient)

**Swap 2**: WETH/USDC pair

-   Input: 0.5 WETH (received from previous swap)
-   Output: 100 USDC
-   Send to: recipient address (\_to)

---

## Complete Example

**Setup**:

-   Path: [DAI, WETH, USDC]
-   amountOut: 100 USDC
-   amountInMax: 110 DAI
-   User address: 0xUser...
-   Recipient: 0x1234...

**Execution**:

```
1. Check deadline ✓

2. Calculate amounts:
   amounts = [150 DAI, 0.5 WETH, 100 USDC]

3. Slippage check:
   150 DAI <= 110 DAI? NO → Revert
   (If it was 100 DAI, would continue)

4. Transfer 100 DAI:
   DAI.transferFrom(0xUser, DAI/WETH_pair, 100)

5. Execute swaps:
   DAI/WETH.swap(0, 0.5WETH, WETH/USDC_pair, ...)
   WETH/USDC.swap(0, 100USDC, 0x1234, ...)

6. Result:
   User receives: 100 USDC at 0x1234
   User spent: 100 DAI
   Return: [100, 0.5, 100]
```

---

## Key Design Points

### 1. Backwards Calculation

Different from swapExactTokensForTokens:

-   Starts with desired output
-   Calculates backwards to required input
-   Suitable when user has output amount in mind

### 2. Slippage Protection

User controls max input spend instead of min output receive.

### 3. Two-Stage Architecture

1. Transfer input tokens to first pair
2. Swaps execute in pair contracts
   Follows Uniswap V2 "pull-push" pattern.

### 4. Multi-Hop Support

Works with any path length, calculating intermediate amounts correctly.

---

## Comparison Table

| Feature               | swapExactTokensForTokens | swapTokensForExactTokens |
| --------------------- | ------------------------ | ------------------------ |
| Input Amount          | Known                    | Calculated               |
| Output Amount         | Calculated               | Known                    |
| Calculation Direction | Forward (in → out)       | Backward (out → in)      |
| Slippage Check        | `amountOutMin`           | `amountInMax`            |
| Use Case              | "Spend 100 DAI"          | "Get 100 USDC"           |
| Formula               | getAmountOut             | getAmountIn              |

---

## Security Considerations

1. **Deadline Protection**: Prevents transaction execution after user-specified time
2. **Slippage Guard**: `require(amounts[0] <= amountInMax)` protects against price movements
3. **Atomic Execution**: All swaps succeed or entire transaction reverts
4. **No Reentrancy**: Token transfers isolated to pair contracts
5. **Reserves Validation**: Calculations based on current pair reserves

---

## Gas Cost

| Operation              | Cost               |
| ---------------------- | ------------------ |
| getAmountsIn (view)    | ~5000 per pair     |
| Slippage check         | ~100               |
| transferFrom           | ~50000             |
| swap (per pair)        | ~45000-50000       |
| **Total (2-hop path)** | **~150000-200000** |

---

## Common Issues

**Issue**: "EXCESSIVE_INPUT_AMOUNT"

-   Cause: Price moved unfavorably, amounts[0] > amountInMax
-   Solution: Increase amountInMax and retry

**Issue**: Transaction reverts at deadline

-   Cause: Network congestion delayed execution
-   Solution: Set longer deadline or retry with higher block timestamp

**Issue**: Insufficient liquidity

-   Cause: Swap amount too large relative to pair reserves
-   Solution: Use smaller amount or find better route

---

## Summary

`swapTokensForExactTokens` is the inverse of `swapExactTokensForTokens`. It calculates backwards from desired output to required input, protecting users through `amountInMax` slippage checks. The function maintains Uniswap V2's two-stage swap pattern: transfer inputs first, then execute swaps atomically through pair contracts.
