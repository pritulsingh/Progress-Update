# Yield Harbor - Architecture Overview

**Project:** Automated Liquidation Protection for Leveraged Positions on Sui  
**Status:** Testnet Deployment Phase  
**Last Updated:** January 8, 2026

---

## What Is Yield Harbor?

Yield Harbor is a looping leverage protocol on Sui that enables users to create and maintain leveraged positions with automatic risk management. The protocol handles position creation, leverage looping, health monitoring, and automatic unwinding when positions become risky.

Key features:
- Create leveraged positions in a single transaction
- Automatic health factor monitoring
- Permissionless auto-unwinding when risk thresholds are crossed
- No manual intervention required after position creation

---

## The Problem We're Solving

Traditional leveraged positions require constant manual monitoring. If collateral value drops or debt increases, users must manually close or adjust positions to avoid liquidation. This creates:
- High cognitive load (constant monitoring)
- Risk of liquidation during sleep/work hours
- Need for external bots or services
- Potential loss of funds in volatile markets

Yield Harbor automates this entire process on-chain with trustless execution.

---

## Core Concept: Looping Leverage

### Traditional Lending (1x exposure)
```
User deposits 100 WETH
Can borrow up to 70 USDC (70% LTV)
Final exposure: 1x WETH
```

### Looping Leverage (2.5x exposure example)
```
Loop 1: Deposit 100 WETH → Borrow 70 USDC → Swap to 70 WETH → Supply
Loop 2: Now have 170 WETH → Borrow 49 USDC → Swap to 49 WETH → Supply  
Loop 3: Now have 219 WETH → Borrow 34 USDC → Swap to 34 WETH → Supply
Final exposure: 253 WETH from initial 100 WETH = 2.53x leverage
```

Each loop increases collateral, which allows more borrowing, creating multiplied exposure to the collateral asset.

---

## Architecture Overview

### System Components

**Core Module**  
The heart of the protocol, handling position lifecycle:
- Position creation and initialization
- Loop execution logic
- Health factor calculations
- Auto-unwind orchestration

**Unified Adapter Layer**  
Single adapter module that abstracts all external protocol interactions:
- Lending operations (supply, borrow, repay, withdraw)
- Token swaps (for looping conversions)
- Price feeds (for health factor calculations)

The adapter acts as a switchable interface layer - using mocks on testnet and real protocols on mainnet, with zero changes to core logic.

**Mock Implementations (Testnet)**  
Simulated versions of real protocols for testing:
- Mock lending protocol with simplified supply/borrow
- Mock DEX with fixed swap rates
- Mock oracle with manual price updates

**Automation Layer**  
Off-chain monitoring with on-chain enforcement:
- Keeper bots watch positions
- Permissionless functions anyone can call
- Chain enforces all safety rules

### Data Flow

**Position Creation:**
```
User submits transaction
  → Transfer initial collateral to protocol
  → Create Position object
  → Execute N loops (borrow → swap → supply)
  → Transfer Position object to user wallet
  → Emit event for indexers
```

**Auto-Unwind:**
```
Price changes in market
  → Bot detects position at risk
  → Bot calls auto_unwind(position)
  → Chain recalculates HF from current prices
  → If HF below threshold: execute unwind
  → If HF still safe: transaction aborts
```

---

## Technical Design Decisions

### 1. Position Model: Owned Objects

Positions are owned objects stored directly in user wallets, not in shared contract storage. This is a Sui-native pattern that provides:
- Clear ownership semantics
- Gas efficiency (no shared object contention)
- Easy composability
- Natural authorization model
```move
struct Position has key, store {
    id: UID,
    owner: address,
    collateral_balance: Balance<CollateralCoin>,
    debt_balance: Balance<DebtCoin>,
    total_supplied: u64,
    total_borrowed: u64,
    loop_count: u64,
    // config fields...
}
```

### 2. Internal Health Factor Calculation

The protocol calculates health factor independently rather than trusting external lending protocols. This ensures:
- Custom risk thresholds for our use case
- Independence from external protocol bugs
- Consistent behavior across different lending markets
- Ability to switch protocols without logic changes

**Formula:**
```
HF = (Collateral Value × Liquidation Threshold) / Borrowed Value

Example:
Collateral: 100 WETH at $2000 = $200,000
Liquidation Threshold: 80%
Debt: 100,000 USDC at $1 = $100,000

HF = (200,000 × 0.8) / 100,000 = 1.6
```

**Thresholds:**
- HF > 1.6: Safe, allow new loops
- 1.3 < HF ≤ 1.6: Warning, pause new loops
- 1.1 < HF ≤ 1.3: Risky, auto-unwind 25%
- HF ≤ 1.1: Critical, auto-unwind 50%
- HF ≤ 1.0: Emergency, full unwind

### 3. Unified Adapter Pattern

All external protocol interactions go through a single adapter module. This provides:
- Clean separation between core logic and external dependencies
- Simple mainnet migration (change imports in one file)
- Testability with mocks
- Flexibility to swap underlying protocols

**How it works:**

The adapter module contains forwarding functions for three categories:

**Lending operations:**
- `lending_supply()` - Supply collateral to lending market
- `lending_borrow()` - Borrow against collateral
- `lending_repay()` - Repay borrowed assets
- `lending_withdraw()` - Withdraw supplied collateral
- `lending_get_supplied()` - Query supplied balance
- `lending_get_borrowed()` - Query borrowed balance
- `lending_get_ltv()` - Get loan-to-value ratio
- `lending_get_liquidation_threshold()` - Get liquidation threshold

**Oracle operations:**
- `oracle_get_price()` - Get current asset price
- `oracle_get_decimals()` - Get price decimals
- `oracle_has_price()` - Check if price feed exists

**Swap operations:**
- `swap()` - Execute token swap
- `swap_get_quote()` - Get expected output amount
- `swap_get_slippage()` - Get current slippage setting

**Testnet configuration:**
```move
// Adapter imports mock implementations
use looping_leverage::mock_lending;
use looping_leverage::mock_oracle;
use looping_leverage::mock_swap;

// Functions forward to mocks
public fun lending_supply<T>(...) {
    mock_lending::supply(...)
}
```

**Mainnet configuration:**
```move
// Switch imports to real protocols
use looping_leverage::navi_adapter;
use looping_leverage::pyth_adapter;
use looping_leverage::cetus_adapter;

// Same function signatures, different implementation
public fun lending_supply<T>(...) {
    navi_adapter::supply(...)
}
```

Core modules never know or care whether they're calling mocks or real protocols - they just call adapter functions with the same signatures.

### 4. Permissionless Automation

Auto-unwind functions are permissionless - anyone can call them, but the chain enforces safety:
- Bot identifies risky position
- Bot submits transaction
- Chain recalculates HF independently
- Chain only executes if conditions are met
- Transaction aborts if position is actually safe

This model provides:
- Decentralization (anyone can run a keeper)
- Security (bots cannot cheat)
- Liveness (protocol continues even if original bot fails)
- Trustlessness (no special permissions required)

---

## Module Structure

### Core Modules
```
sources/core/
  position.move          - Position struct and creation logic
  health_factor.move     - HF calculation and threshold checks
  looping.move           - Loop execution orchestration
  unwinding.move         - Partial and full unwind logic
```

### Single Adapter Module
```
sources/
  adapters.move          - Unified adapter layer with all forwarding functions
                          Contains lending, oracle, and swap adapters in one file
                          Switch between mock/real via import statements
```

### Mocks (Testnet Only)
```
sources/mocks/
  mock_lending.move      - Simulated lending protocol
  mock_swap.move         - Simulated DEX
  mock_oracle.move       - Manual price updates for testing
  mock_tokens.move       - MockWETH, MockUSDC test coins
```

### Automation
```
sources/automation/
  keeper.move            - Permissionless functions for bots
```

---

## Testnet vs Mainnet Strategy

### Testnet Phase (Current)
- Deploy mock implementations of all external protocols
- Adapter module imports and forwards to mocks
- Test core logic with simulated markets
- Validate position lifecycle end-to-end
- Gather user feedback on UX
- Identify and fix bugs in safe environment

### Mainnet Migration

The adapter pattern makes this trivial:

1. Open `adapters.move`
2. Comment out mock imports:
```move
   // use looping_leverage::mock_lending;
   // use looping_leverage::mock_oracle;
   // use looping_leverage::mock_swap;
```
3. Uncomment real protocol imports:
```move
   use looping_leverage::navi_adapter;
   use looping_leverage::pyth_adapter;
   use looping_leverage::cetus_adapter;
```
4. Update function bodies to call real protocols instead of mocks
5. Core modules remain completely unchanged

That's it. No changes to position logic, health factor calculations, looping execution, or unwinding logic. The adapter abstraction ensures clean separation.

---

## Security Model

### On-Chain Enforcement
All critical logic executes on-chain:
- Health factor calculations
- Threshold checks
- Unwind execution
- Balance updates
- Access control

### Off-Chain Components
Bots and indexers are untrusted:
- Bots can only submit transactions, not control outcomes
- Indexers record history but don't make decisions
- If bots fail, positions remain safe (just won't auto-unwind)
- Anyone can replace bots with their own

### Authorization Pattern
- User owns Position object (Sui object model)
- User-specific functions require position ownership
- Keeper functions are permissionless but enforce safety checks
- No approval mechanisms needed (Sui handles this natively)

---

## User Experience

### What Users Do
1. Connect wallet
2. Choose collateral and borrow assets
3. Set initial deposit and target loops
4. Click "Create Position"
5. Sign one transaction
6. Monitor position via dashboard

### What Users See
- Current leverage multiplier
- Health factor in real-time
- Total collateral and debt values
- Position profit/loss
- Auto-unwind notifications (if triggered)

### What Users Don't See
- Looping mechanics
- Keeper bots
- Oracle updates
- Adapter implementations
- Internal calculations

The goal is a "set it and forget it" experience where users create positions and trust the protocol to manage risk automatically.

---

## Current Status

**Completed:**
- Core protocol architecture design
- Position lifecycle logic
- Health factor calculation module
- Mock protocol implementations
- Unified adapter layer with mock forwarding
- Comprehensive test suite (178 passing tests)

**In Progress:**
- Testnet deployment
- Frontend development
- Keeper bot implementation

**Next Steps:**
- Deploy to Sui testnet
- Build position creation UI
- Test full flow with real users
- Gather feedback and iterate
- Prepare mainnet adapters (Navi, Pyth, Cetus)
- Security audit

---

## Key Takeaways

1. **Sui-native design**: Built around owned objects, not shared contract storage
2. **Independent risk management**: Calculate HF internally, never trust external protocols
3. **Single adapter abstraction**: One file handles all external protocol switching
4. **Permissionless automation**: Anyone can trigger auto-unwinds, chain enforces rules
5. **Production-ready**: Architecture designed for mainnet from day one, not a demo

The protocol bridges the gap between DeFi composability and user safety by automating risk management while maintaining trustless execution.

---

**Document Version:** 1.0  
**Architecture Status:** Finalized and implementation-ready