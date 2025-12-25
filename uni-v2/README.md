# Uniswap V2 Architecture Overview

## Introduction

Uniswap V2 is built on two main repositories: **v2-core** and **v2-periphery**. These layers separate concerns between core protocol logic and user-facing interfaces.

---

## V2-Core (Core Layer)

The core layer contains the fundamental smart contracts that implement the Uniswap V2 protocol.

### Key Contracts

-   **UniswapV2ERC20.sol**
    -   Implements the ERC20 standard for liquidity provider (LP) tokens
    -   Handles token transfers and balance management
-   **UniswapV2Factory.sol**

    -   Factory contract responsible for deploying new trading pairs
    -   Maintains a registry of all created pair contracts
    -   Controls pair creation and fee structure

-   **UniswapV2Pair.sol**
    -   The heart of the protocol: implements each trading pair's logic
    -   Implements the constant product formula: `x × y = L²`
    -   Handles swaps, liquidity provision (mint/burn), and flash loans
    -   Manages reserves and price data

### Structure

```
v2-core/
├── contracts/
│   ├── UniswapV2ERC20.sol
│   ├── UniswapV2Factory.sol
│   └── UniswapV2Pair.sol
```

### Characteristics

-   **Minimal and efficient** - only essential logic
-   **Protocol layer** - defines how trading works
-   **Direct blockchain interaction** - no user-friendly abstractions

---

## V2-Periphery (Periphery Layer)

The periphery layer provides convenient interfaces and helper functions that make it easier for users and applications to interact with v2-core.

### Key Contracts

-   **UniswapV2Router01.sol & UniswapV2Router02.sol**

    -   Primary user interface for interacting with Uniswap V2
    -   Handles trade execution with slippage protection
    -   Manages liquidity addition and removal
    -   Supports multi-hop routing (e.g., A→B→C trades)
    -   Router02 is the improved version with additional features

-   **UniswapV2Migrator.sol**
    -   Facilitates migration of liquidity from Uniswap V1 to V2
    -   Helps users transition their positions seamlessly

### Structure

```
v2-periphery/
├── contracts/
│   ├── UniswapV2Migrator.sol
│   ├── UniswapV2Router01.sol
│   └── UniswapV2Router02.sol
```

### Characteristics

-   **User-friendly abstractions** - simplifies complex operations
-   **Application layer** - provides convenient APIs
-   **Safety features** - includes slippage controls and output guarantees
-   **Complex routing logic** - handles multi-step trades

---

## Relationship Between Layers

```
Users/Applications
        ↓
V2-Periphery (Router)  ← Convenient Interface
        ↓
V2-Core (Pair/Factory) ← Core Protocol Logic
        ↓
Blockchain
```

### Separation of Concerns

| Aspect               | V2-Core        | V2-Periphery                 |
| -------------------- | -------------- | ---------------------------- |
| **Responsibility**   | Protocol logic | User interface               |
| **Focus**            | What to trade  | How to trade conveniently    |
| **Complexity**       | Minimal        | More complex abstractions    |
| **User interaction** | Not direct     | Primary interaction point    |
| **Gas efficiency**   | Optimized      | Convenient but may cost more |

---

## Key Concepts

### Constant Product Formula

```
x × y = L²
```

Where:

-   `x` = amount of token A in the pool
-   `y` = amount of token B in the pool
-   `L` = liquidity (a constant value)

### Flow of Operations

1. **Swap**: User calls Router → Router calls Pair → Pair executes swap using constant product formula
2. **Add Liquidity**: User calls Router → Router deposits tokens → Pair mints LP tokens
3. **Remove Liquidity**: User calls Router → Router burns LP tokens → Pair returns underlying tokens

---

## Summary

-   **V2-Core** is the foundation - it defines the rules and mechanics
-   **V2-Periphery** is the interface - it makes the protocol usable and safe
-   Together they create a modular, secure, and efficient decentralized exchange protocol
