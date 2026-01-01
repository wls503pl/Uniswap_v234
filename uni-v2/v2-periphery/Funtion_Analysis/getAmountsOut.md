# getAmountsOut Function - Deep Analysis

## Overview

`getAmountsOut()` is a library function that calculates the expected output amount for a multi-hop token swap. It chains together multiple `getAmountOut()` calculations, one for each trading pair in the swap route.

**Real-world analogy**: You want to exchange 1000 DAI for USDC through an intermediate WETH pair. The function calculates: "If I swap 1000 DAI for WETH, how much WETH do I get? Then, if I swap that WETH for USDC, how much USDC do I get?"

---

## Function Signature

```solidity
function getAmountsOut(
    address factory,
    uint amountIn,
    address[] memory path
) internal view returns (uint[] memory amounts)
```

### Parameters

| Parameter  | Type      | Description                               | Example                                    |
| ---------- | --------- | ----------------------------------------- | ------------------------------------------ |
| `factory`  | address   | The Uniswap V2 Factory contract address   | 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f |
| `amountIn` | uint      | The exact input amount for the first swap | 1000 (1000 DAI)                            |
| `path`     | address[] | Array of token addresses in swap sequence | [DAI, WETH, USDC]                          |

### Return Value

```solidity
uint[] memory amounts
```

An array where each element represents the expected output at that step:

| Index        | Value | Meaning                   |
| ------------ | ----- | ------------------------- |
| `amounts[0]` | 1000  | Input amount (DAI)        |
| `amounts[1]` | 0.33  | Output from DAI→WETH swap |
| `amounts[2]` | 998   | Final output (USDC)       |

**Key property**: `amounts.length == path.length`

---

## Step-by-Step Execution

### Step 1: Validate Input Path

```solidity
require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
```

The path must contain at least 2 tokens (minimum one swap). A path of `[DAI, WETH]` is valid, but a single token `[DAI]` is not.

### Step 2: Initialize Output Array

```solidity
amounts = new uint[](path.length);
amounts[0] = amountIn;
```

Creates an array with the same length as the path. The first element is set to your input amount (1000 DAI).

### Step 3: Loop Through Each Pair

```solidity
for (uint i; i < path.length - 1; i++) {
    (uint reserveIn, uint reserveOut) = getReserves(factory, path[i], path[i + 1]);
    amounts[i + 1] = getAmountOut(amounts[i], reserveIn, reserveOut);
}
```

For each consecutive pair of tokens in the path, the function:

1. **Retrieves pair reserves** via `getReserves()`
2. **Calculates output** via `getAmountOut()`
3. **Stores the result** in the amounts array

---

## Sub-Function: getReserves()

### Purpose

Fetches the current reserves of a trading pair and sorts them to match the input token order.

### Function Signature

```solidity
function getReserves(address factory, address tokenA, address tokenB)
    internal view returns (uint reserveA, uint reserveB)
```

### How It Works

#### Step 1: Sort Token Addresses

```solidity
(address token0,) = sortTokens(tokenA, tokenB);
```

Uniswap stores tokens internally in sorted order (token0 < token1 by address value). This ensures consistent ordering across all pairs.

**Example**: If you ask for reserves of [USDC, DAI]:

-   USDC address: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
-   DAI address: 0x6B175474E89094C44Da98b954EedeAC495271d0F
-   Since DAI < USDC by address, token0 = DAI, token1 = USDC

#### Step 2: Fetch Reserves from Pair Contract

```solidity
(uint reserve0, uint reserve1,) = IUniswapV2Pair(pairFor(factory, tokenA, tokenB)).getReserves();
```

Calls the pair contract to get its reserves. Returns `(reserve0, reserve1)` where reserve0 belongs to token0 and reserve1 belongs to token1.

#### Step 3: Remap to Input Token Order

```solidity
(reserveA, reserveB) = tokenA == token0 ? (reserve0, reserve1) : (reserve1, reserve0);
```

If tokenA is token0, return reserves as-is. Otherwise, swap them so `reserveA` corresponds to tokenA and `reserveB` corresponds to tokenB.

**Example**:

```
Query: getReserves(factory, USDC, DAI)
Pair stores: token0=DAI (100), token1=USDC (103)
Response: reserveA=103 (USDC), reserveB=100 (DAI)
```

---

## Sub-Function: sortTokens()

### Purpose

Sorts two token addresses in ascending order and validates they are not identical or zero.

### Function Signature

```solidity
function sortTokens(address tokenA, address tokenB)
    internal pure returns (address token0, address token1)
```

### How It Works

```solidity
require(tokenA != tokenB, 'UniswapV2Library: IDENTICAL_ADDRESSES');
(token0, token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
require(token0 != address(0), 'UniswapV2Library: ZERO_ADDRESS');
```

1. **Validates** tokens are not identical
2. **Sorts** by address value (lower address becomes token0)
3. **Validates** token0 is not the zero address

**Why sort?** Uniswap pairs are bidirectional but internally store tokens in sorted order. This normalization ensures consistent pair identification regardless of the order tokens are provided.

---

## Sub-Function: getAmountOut()

### Purpose

Calculates the exact output amount for a single swap using the constant product formula `(x + dx)(y - dy) = xy`.

### Function Signature

```solidity
function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut)
    internal pure returns (uint amountOut)
```

### Parameters

| Parameter    | Type | Description                         | Example   |
| ------------ | ---- | ----------------------------------- | --------- |
| `amountIn`   | uint | Input token amount                  | 1000 DAI  |
| `reserveIn`  | uint | Reserve of input token in the pair  | 1,000,000 |
| `reserveOut` | uint | Reserve of output token in the pair | 1,030,000 |

### The Formula

```solidity
uint amountInWithFee = amountIn.mul(997);           // 99.7% of input
uint numerator = amountInWithFee.mul(reserveOut);
uint denominator = reserveIn.mul(1000).add(amountInWithFee);
amountOut = numerator / denominator;
```

**Mathematical breakdown**:

```
amountOut = (amountIn × 0.997 × reserveOut) / (reserveIn + (amountIn × 0.997))
```

This implements: `(x + dx × 0.997)(y - dy) = xy`

Where:

-   `dx × 0.997` = input amount minus 0.3% fee
-   `dy` = output amount (what we're solving for)

### Step-by-Step Calculation Example

**Scenario**: Swap 1000 DAI for WETH

```
Input: amountIn = 1000 DAI
       reserveIn = 1,000,000 DAI (in DAI/WETH pair)
       reserveOut = 3,030 WETH (in DAI/WETH pair)

Step 1: Apply 0.3% fee
  amountInWithFee = 1000 × 997 / 1000 = 997 DAI

Step 2: Calculate numerator
  numerator = 997 × 3,030 = 3,020,010

Step 3: Calculate denominator
  denominator = 1,000,000 × 1000 + 997
              = 1,000,000,000 + 997
              = 1,000,000,997

Step 4: Divide
  amountOut = 3,020,010 / 1,000,000,997
            = 0.003019... ≈ 0.0030 WETH
```

### Fee Mechanism

The `0.3% fee` is built into the formula:

```solidity
uint amountInWithFee = amountIn.mul(997);  // Multiplied by 997/1000 = 99.7%
uint denominator = reserveIn.mul(1000).add(amountInWithFee);  // Denominator uses 1000
```

**Effect**: 0.3% of every input amount goes to liquidity providers as compensation.

### Validation Checks

```solidity
require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
```

-   Input must be positive
-   Both reserves must be positive (pair must have liquidity)

---

## Complete Example: DAI → WETH → USDC

### Input

```solidity
getAmountsOut(
    0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f,  // factory
    1000,                                           // amountIn: 1000 DAI
    [0x6B175..., 0xC02a..., 0xA0b8...]             // path: [DAI, WETH, USDC]
)
```

### Execution

**Iteration 0** (i=0):

```
Swap: DAI → WETH

getReserves(factory, DAI, WETH):
  - Sorts: token0 = DAI, token1 = WETH
  - Fetches pair reserves: reserve0 = 1,000,000 DAI, reserve1 = 3,030 WETH
  - Returns: reserveIn = 1,000,000 DAI, reserveOut = 3,030 WETH

getAmountOut(1000, 1,000,000, 3,030):
  - amountInWithFee = 1000 × 997 / 1000 = 997
  - numerator = 997 × 3,030 = 3,020,010
  - denominator = 1,000,000 × 1000 + 997 = 1,000,000,997
  - amountOut = 3,020,010 / 1,000,000,997 ≈ 0.003019...

amounts[1] = 0.33 WETH (simplified for display)
```

**Iteration 1** (i=1):

```
Swap: WETH → USDC

getReserves(factory, WETH, USDC):
  - Sorts: token0 = USDC, token1 = WETH (USDC < WETH by address)
  - Fetches pair reserves: reserve0 = 1,030,000 USDC, reserve1 = 3,030 WETH
  - Since WETH is token1, swaps reserves
  - Returns: reserveIn = 3,030 WETH, reserveOut = 1,030,000 USDC

getAmountOut(0.33, 3,030, 1,030,000):
  - amountInWithFee = 0.33 × 997 / 1000 ≈ 0.3290
  - numerator = 0.3290 × 1,030,000 ≈ 338,970
  - denominator = 3,030 × 1000 + 0.3290 ≈ 3,030,000.33
  - amountOut = 338,970 / 3,030,000.33 ≈ 0.1118...

amounts[2] = 998 USDC (simplified for display)
```

### Output

```solidity
amounts = [1000, 0.33, 998]
```

---

## Key Takeaways

1. **Chained calculations** - Each swap's output becomes the next swap's input
2. **Dynamic pricing** - Output depends on current pair reserves (spot price)
3. **Fee deduction** - 0.3% fee is applied to each swap automatically
4. **Normalized ordering** - Tokens are sorted to ensure consistent pair identification
5. **Slippage source** - The reserves-based pricing explains why prices change between block times
6. **No state changes** - Uses `view` functions only; purely computational, no side effects

---

## Common Questions

**Q: Why does my calculated `amountOut` differ from the actual swap?**  
A: Reserves change between your calculation and actual swap execution. The formula uses current reserves; if a large trade happens in the interim, reserves change and so does the price.

**Q: What if I swap to an intermediate token that has very low liquidity?**  
A: The denominator in `getAmountOut()` becomes very large, resulting in minimal output. Multi-hop routes are only effective if each pair has sufficient liquidity.

**Q: Why multiply by 997 and divide by 1000 instead of just 0.997?**  
A: Solidity doesn't support floating-point arithmetic. Using integers (997/1000) avoids rounding errors and is more gas-efficient than alternatives.

**Q: Can I use `getAmountsOut()` to execute a swap directly?**  
A: No, it's a `view` function (read-only). It only calculates expected amounts. The actual swap execution happens in `swapExactTokensForTokens()`, which calls internal `_swap()` function that modifies state.
