# Uniswap V2 Functions:

## Overview

This guide explains two core functions in Uniswap V2 Router: `swapExactTokensForTokens()` and `_swap()`. Together, they enable you to trade one token for another through automated market makers.

**Real-world analogy**: You go to a currency exchange and say "I want to trade 1000 DAI for as much USDC as possible." The exchange tells you the current rate, you agree, and the trade happens.

---

## 1. swapExactTokensForTokens() Function

### Purpose

Execute a token swap where you specify the **input amount** and receive the **maximum possible output** based on current market prices.

### Function Signature

```solidity
function swapExactTokensForTokens(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external virtual override ensure(deadline) returns (uint[] memory amounts)
```

### Parameters

| Parameter      | Type      | Description                                          | Example                      |
| -------------- | --------- | ---------------------------------------------------- | ---------------------------- |
| `amountIn`     | uint      | The exact amount of input token you're sending       | 1000 DAI                     |
| `amountOutMin` | uint      | Minimum acceptable output (slippage protection)      | 24000 USDC                   |
| `path`         | address[] | Array of token addresses representing the swap route | [DAI, WETH, USDC]            |
| `to`           | address   | Wallet address that receives the final output tokens | Your wallet address          |
| `deadline`     | uint      | Unix timestamp when transaction expires if not mined | current_time + 600 (10 mins) |

### Return Value

```solidity
uint[] memory amounts
```

An array containing the amount at each step of the swap:

| Index        | Value | Meaning                     |
| ------------ | ----- | --------------------------- |
| `amounts[0]` | 1000  | Input amount (DAI)          |
| `amounts[1]` | 25    | Output from 1st swap (WETH) |
| `amounts[2]` | 24000 | Final output (USDC)         |

**Key relationship**: `amounts.length == path.length` (both have 3 elements)

---

### How It Works (Step-by-Step)

#### Step 1: Calculate Expected Outputs

```solidity
amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
```

The library function calculates how much output you'll get at each step using Uniswap's constant product formula (`x * y = k`).

For our example:

-   You send 1000 DAI
-   DAI/WETH pair outputs 25 WETH
-   WETH/USDC pair outputs 24000 USDC

#### Step 2: Verify Slippage Protection

```solidity
require(amounts[amounts.length - 1] >= amountOutMin,
        'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
```

This checks: `amounts[2] (24000) >= amountOutMin (24000)` ✓

**Why is this important?** Between when you submit your transaction and when it gets mined, market prices can change. This is called **slippage**. If the output drops below your minimum (e.g., to 23000 USDC), the transaction is rejected and you lose nothing. This protects you from sandwich attacks.

#### Step 3: Transfer Input Tokens to the First Pair

```solidity
TransferHelper.safeTransferFrom(
    path[0],  // DAI token address
    msg.sender,  // Your wallet address
    UniswapV2Library.pairFor(factory, path[0], path[1]),  // DAI/WETH pair contract
    amounts[0]  // 1000 DAI
);
```

**What happens**: Your wallet → transfers 1000 DAI → to the DAI/WETH Pair contract

**What is `pairFor()`?** It retrieves the contract address of the DAI/WETH trading pair from the factory registry.

#### Step 4: Execute the Actual Swaps

```solidity
_swap(amounts, path, to);
```

This internal function performs the actual token exchanges through each pair. Details below.

---

### Security Mechanisms

#### ✓ Slippage Protection (`amountOutMin`)

Protects against price fluctuations between transaction submission and execution.

```
Scenario: You expect 24000 USDC but get only 23500 USDC due to price changes
Solution: Set amountOutMin = 24000
Result: Transaction rejected, your DAI is returned safely
```

#### ✓ Expiration Protection (`deadline` + `ensure()`)

```solidity
modifier ensure(uint deadline) {
    require(deadline >= block.timestamp, 'UniswapV2Router: EXPIRED');
    _;
}
```

Prevents your transaction from being mined after your specified deadline.

```
Scenario: Malicious miner deliberately delays your transaction to execute at worse price
Solution: Set deadline = current_block_time + 600 seconds (10 minutes)
Result: If not mined within 10 minutes, transaction automatically fails
Defense: Protects against sandwich attacks
```

---

## 2. \_swap() Function

### Purpose

Perform the actual token exchanges through each pair in the swap path. This is an internal function called by `swapExactTokensForTokens()`.

### Function Signature

```solidity
function _swap(
    uint[] memory amounts,
    address[] memory path,
    address _to
) internal virtual
```

### Parameters

| Parameter | Description                         | Example           |
| --------- | ----------------------------------- | ----------------- |
| `amounts` | Array of token amounts at each step | [1000, 25, 24000] |
| `path`    | Array of token addresses            | [DAI, WETH, USDC] |
| `_to`     | Final recipient address             | Your wallet       |

### How It Works

The function loops through each trading pair in the path and executes swaps:

#### Loop Iteration Logic

```
Iteration 0:
  input = path[0] (DAI)
  output = path[1] (WETH)
  amountOut = amounts[1] (25 WETH)
  → Send output to: next pair (DAI/WETH → WETH/USDC)

Iteration 1:
  input = path[1] (WETH)
  output = path[2] (USDC)
  amountOut = amounts[2] (24000 USDC)
  → Send output to: user's address (final recipient)
```

#### Key Trading Pair Details

For each step, the function determines `amount0Out` and `amount1Out` based on token ordering:

```solidity
(address token0,) = UniswapV2Library.sortTokens(input, output);
```

Uniswap stores tokens in sorted order within each pair contract. The function calculates which output goes to which position (token0 or token1).

#### Routing Logic

```solidity
address to = i < path.length - 2 ?
    UniswapV2Library.pairFor(factory, output, path[i + 2]) :
    _to;
```

-   **For intermediate swaps** (i < path.length - 2): Send output to the next pair contract
-   **For the final swap** (i == path.length - 2): Send output to the user's address

---

## Complete Example: DAI → WETH → USDC

### Initial Transaction Call

```solidity
swapExactTokensForTokens(
    1000,                                    // amountIn: 1000 DAI
    24000,                                   // amountOutMin: Accept at least 24000 USDC
    [0x6B17..., 0xC02a..., 0xA0b8...],      // path: DAI, WETH, USDC
    0xYourAddress,                           // to: Your wallet
    1735286400                               // deadline: Jan 1, 2025
)
```

### Execution Flow

**Step 1**: Calculate amounts

-   `amounts[0]` = 1000 DAI (input)
-   `amounts[1]` = 25 WETH (intermediate)
-   `amounts[2]` = 24000 USDC (output)

**Step 2**: Check slippage

-   `amounts[2]` (24000) >= `amountOutMin` (24000) ✓

**Step 3**: Transfer input

-   Your wallet → 1000 DAI → DAI/WETH Pair

**Step 4**: Execute swaps via `_swap()`

| Step | Input Pair | What Happens                     | Output Goes To |
| ---- | ---------- | -------------------------------- | -------------- |
| 0    | DAI/WETH   | Send 1000 DAI, receive 25 WETH   | WETH/USDC Pair |
| 1    | WETH/USDC  | Send 25 WETH, receive 24000 USDC | Your Wallet    |

### Final Result

Your wallet receives 24000 USDC

---

## Key Takeaways

1. **`swapExactTokensForTokens()` is the entry point** - It orchestrates the entire swap process
2. **`_swap()` handles the mechanics** - It executes individual pair swaps
3. **`amountOutMin` protects you** - Rejects trades with excessive slippage
4. **`deadline` prevents delays** - Ensures transaction executes within your timeframe
5. **Multi-hop routing is automatic** - The path array enables trading through multiple pairs
6. **Security is built-in** - Multiple layers protect against sandwich attacks and price manipulation

---

## Common Questions

**Q: What happens if I don't reach `amountOutMin`?**  
A: The entire transaction reverts. You lose nothing (no gas wasted on failed swap).

**Q: Why does my transaction fail sometimes even with high slippage tolerance?**  
A: Check your `deadline`. If it's too soon (e.g., 30 seconds on mainnet), network congestion might delay mining past your deadline.

**Q: Can I swap directly between any two tokens?**  
A: Only if a pair exists and has liquidity. If no direct pair exists, you need an intermediate token (like WETH), hence the multi-hop route.
