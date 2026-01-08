# Yield Harbor - Technical Decisions

**Project:** Automated Liquidation Protection for Leveraged Positions on Sui  
**Purpose:** Document key architectural and implementation decisions  
**Last Updated:** January 8, 2026

---

## Table of Contents

1. [Position Storage Model](#position-storage-model)
2. [Health Factor Calculation](#health-factor-calculation)
3. [Adapter Architecture](#adapter-architecture)
4. [Automation Model](#automation-model)
5. [Token and Balance Tracking](#token-and-balance-tracking)
6. [Access Control](#access-control)
7. [Error Handling](#error-handling)
8. [Gas Optimization](#gas-optimization)
9. [Testing Strategy](#testing-strategy)
10. [Testnet to Mainnet Migration](#testnet-to-mainnet-migration)

---

## Position Storage Model

### Decision: Use Owned Objects

**Choice Made:** Positions are stored as owned objects in user wallets, not in shared protocol storage.

**Rationale:**

Sui's object model provides two main options for storing state:
- Shared objects (accessible by multiple transactions simultaneously)
- Owned objects (exclusive access by owner)

We chose owned objects because:

1. **Gas Efficiency**: Owned objects avoid consensus overhead. Shared objects require coordination across validators, increasing costs.

2. **Sui-Native Pattern**: Most successful Sui DeFi protocols use owned objects for user-specific data. This aligns with ecosystem best practices.

3. **Clear Ownership**: Users literally own their positions as objects in their wallet. No ambiguity about who controls what.

4. **Composability**: Owned objects can be passed to functions easily. Other protocols can integrate by accepting Position objects.

5. **No Contention**: Multiple users can interact with their positions simultaneously without blocking each other.

**Alternative Considered:**

Shared object with mapping of user addresses to position data (Ethereum-style).

Rejected because:
- Higher gas costs
- Worse scaling characteristics
- Not idiomatic Sui
- Unnecessary complexity

**Implementation:**
```move
struct Position has key, store {
    id: UID,
    owner: address,
    collateral_balance: Balance<CollateralCoin>,
    debt_balance: Balance<DebtCoin>,
    // ... other fields
}
```

**Trade-offs:**

Pros:
- Lower gas costs
- Better UX (positions in wallet)
- Natural authorization
- Scales well

Cons:
- Cannot query "all positions" on-chain (need indexer)
- Slightly more complex for keeper bots (must track positions off-chain)

---

## Health Factor Calculation

### Decision: Calculate Internally, Never Trust External

**Choice Made:** Protocol calculates health factor from raw on-chain data. Never trusts health factors from external lending protocols.

**Rationale:**

External lending protocols (Navi, Scallop) have their own health factor calculations and liquidation thresholds. We could query their values, but we don't.

Reasons for independent calculation:

1. **Custom Risk Parameters**: Our protocol needs different thresholds than underlying protocols. We want to unwind at HF 1.3, even if Navi liquidates at 1.0.

2. **Security**: External protocol could have bugs in their HF calculation. We need independent verification.

3. **Protocol Independence**: If we switch from Navi to Scallop, our core logic shouldn't change. Calculating HF ourselves ensures consistency.

4. **Auditability**: Auditors expect critical safety logic to be in our code, not external dependencies.

5. **Transparency**: Users and auditors can verify our exact risk model.

**Formula Used:**
```
Health Factor = (Collateral Value × Liquidation Threshold) / Borrowed Value

Where:
- Collateral Value = Amount × Price (from oracle)
- Liquidation Threshold = Protocol parameter (e.g., 0.8 for 80%)
- Borrowed Value = Debt Amount × Price (from oracle)
```

**Implementation:**
```move
public fun calculate_health_factor(
    collateral_amount: u64,
    collateral_price: u64,
    debt_amount: u64,
    debt_price: u64,
    liquidation_threshold_bps: u64
): u64 {
    let collateral_value = (collateral_amount as u128) * (collateral_price as u128);
    let debt_value = (debt_amount as u128) * (debt_price as u128);
    let threshold_value = collateral_value * (liquidation_threshold_bps as u128) / 10000;
    ((threshold_value * PRECISION) / debt_value) as u64
}
```

**Alternative Considered:**

Query health factor from Navi: `navi::get_health_factor(user)`

Rejected because:
- Loss of control over risk parameters
- Dependency on external implementation
- Cannot customize thresholds
- Harder to audit and verify

**Trade-offs:**

Pros:
- Full control over risk model
- Protocol-independent
- Auditable and transparent
- Can customize per use case

Cons:
- Need to maintain oracle integrations
- Must handle price feed failures
- Slight gas overhead (but minimal)

---

## Adapter Architecture

### Decision: Single Unified Adapter Module

**Choice Made:** One adapter module handles all external protocol interactions (lending, swaps, oracles).

**Rationale:**

We need to interact with three types of external protocols:
- Lending markets (Navi, Scallop)
- DEXs (Cetus, Turbos)
- Oracles (Pyth, Supra)

Options considered:

1. **Separate adapter modules** (lending_adapter.move, swap_adapter.move, oracle_adapter.move)
2. **Unified adapter module** (adapters.move) ← CHOSEN
3. **Direct integration** in core modules

We chose unified adapter because:

1. **Simple Mainnet Migration**: Change imports in ONE file to switch from mocks to real protocols.

2. **Consistency**: All external calls follow same pattern, making code easier to review.

3. **Centralized Configuration**: Easy to see which protocols are being used.

4. **Less Code Duplication**: Shared helper functions, error handling, etc.

5. **Testing**: Mock all externals by swapping one module.

**Implementation Pattern:**
```move
// adapters.move

// TESTNET imports
use looping_leverage::mock_lending;
use looping_leverage::mock_oracle;
use looping_leverage::mock_swap;

// Forwarding functions
public fun lending_supply<T>(...) {
    mock_lending::supply(...)  // ← Just change this line for mainnet
}
```

**Mainnet Migration:**
```move
// Comment out mocks
// use looping_leverage::mock_lending;

// Uncomment real protocols
use looping_leverage::navi_adapter;

// Same function, different implementation
public fun lending_supply<T>(...) {
    navi_adapter::supply(...)  // ← One line change
}
```

**Alternative Considered:**

Separate adapter files with trait-like interfaces.

Rejected because:
- Move doesn't have traits/interfaces
- More files to manage
- Harder to see full picture
- Over-engineering for current scope

**Trade-offs:**

Pros:
- Single point of configuration
- Easy mainnet migration
- Simple to understand
- Less file management

Cons:
- One file gets larger
- All protocols bundled together
- Less modular (but not an issue for our use case)

---

## Automation Model

### Decision: Permissionless Functions + Keeper Bots

**Choice Made:** Auto-unwind functions are permissionless (anyone can call), with off-chain bots monitoring and submitting transactions.

**Rationale:**

For automatic position unwinding, we need some form of automation. Options:

1. **Permissioned keepers**: Only whitelisted addresses can call unwind
2. **On-chain automation**: Chainlink Keepers / similar
3. **Permissionless + bots**: Anyone can call, bots submit transactions ← CHOSEN

We chose permissionless with bots because:

1. **Decentralization**: Anyone can run a keeper bot. No single point of failure.

2. **Sui Reality**: Sui doesn't have on-chain automation like Chainlink Keepers (yet).

3. **Economic Incentives**: Bots are incentivized to unwind risky positions (future: keeper rewards).

4. **Trustlessness**: Bot cannot cheat. Chain enforces all rules.

5. **Proven Model**: Used by Aave liquidations, MakerDAO keepers, etc.

**How It Works:**
```
Off-chain Bot:
  1. Monitor positions (via indexer or RPC)
  2. Calculate HF off-chain (optimization)
  3. If HF < threshold, submit unwind transaction

On-chain Function:
  1. Recalculate HF from current prices
  2. Verify HF is actually below threshold
  3. If yes: execute unwind
  4. If no: abort transaction (bot wasted gas)
```

**Implementation:**
```move
// PERMISSIONLESS - anyone can call
public fun auto_unwind(
    position: &mut Position,
    oracle: &Oracle,
    // ... other params
) {
    // Chain enforces safety, not caller identity
    let current_hf = calculate_hf(position, oracle);
    assert!(current_hf < UNWIND_THRESHOLD, EPositionStillSafe);
    
    // Safe to unwind
    execute_unwind_logic(position);
}
```

**Security Model:**

- Bot is UNTRUSTED: It's just a transaction relay
- Chain is TRUSTED: All logic executed on-chain
- Bot can't manipulate: Can only trigger, not control outcome

**Alternative Considered:**

Permissioned keepers with whitelist.

Rejected because:
- Centralization risk
- Single point of failure
- Admin key management burden
- Against DeFi ethos

**Trade-offs:**

Pros:
- Fully decentralized
- No admin keys
- Anyone can contribute
- Resilient to failures

Cons:
- Bots can submit invalid transactions (waste their own gas)
- Need good off-chain infrastructure
- Relies on economic incentives (future keeper rewards)

---

## Token and Balance Tracking

### Decision: Internal Balance Tracking (No Receipt Tokens)

**Choice Made:** Track collateral and debt as Balance objects inside Position. Do not mint receipt tokens (aTokens/debtTokens).

**Rationale:**

Lending protocols like Aave mint tokens representing deposits (aTokens) and debt (debtTokens). We could do the same, but don't.

Reasons:

1. **Simplicity**: No need to manage token minting/burning logic.

2. **Testnet First**: Receipt tokens add complexity without benefit for testnet.

3. **Clear Accounting**: Balance fields directly show position state.

4. **Easy Queries**: Balance is already in the Position struct.

5. **Mainnet Compatible**: Real protocols already track balances. We query them directly.

**Implementation:**
```move
struct Position {
    collateral_balance: Balance<CollateralCoin>,  // Actual coins
    debt_balance: Balance<DebtCoin>,              // Debt owed
    total_supplied: u64,                          // Accounting
    total_borrowed: u64,                          // Accounting
}
```

**How It Works:**

When user supplies:
```move
public fun supply<T>(position: &mut Position, coins: Coin<T>) {
    let amount = coin::value(&coins);
    balance::join(&mut position.collateral_balance, coin::into_balance(coins));
    position.total_supplied = position.total_supplied + amount;
}
```

When protocol borrows on behalf of user:
```move
public fun borrow<T>(position: &mut Position, amount: u64, ...) {
    let borrowed_coins = lending_adapter::borrow<T>(..., amount);
    position.total_borrowed = position.total_borrowed + amount;
    // Use borrowed_coins for swapping
}
```

**Alternative Considered:**

Mint aTokens for deposits, debtTokens for borrows (like Aave).

Rejected because:
- Adds complexity
- Not needed for core functionality
- Can add later if needed for composability
- Testnet doesn't need it

**Trade-offs:**

Pros:
- Simpler codebase
- Easier to audit
- Direct balance queries
- Less gas (no minting/burning)

Cons:
- Can't transfer position ownership easily (not a requirement)
- No external composability via receipt tokens (not needed for MVP)

---

## Access Control

### Decision: Object Ownership + Function Guards

**Choice Made:** Use Sui's object ownership model for authorization. Position owner verified via owner field, not via object ownership alone.

**Rationale:**

Sui provides authorization through object ownership. If you can pass an object to a function, you have access to it. But we add explicit checks.

**Implementation Pattern:**
```move
// User functions - require owner check
public fun user_close_position(
    position: Position,
    ctx: &TxContext
) {
    assert!(position.owner == tx_context::sender(ctx), ENotOwner);
    // ... close logic
}

// Keeper functions - permissionless
public fun auto_unwind(
    position: &mut Position,
    oracle: &Oracle
) {
    // Anyone can call, but logic enforces safety
    let hf = calculate_hf(position, oracle);
    assert!(hf < THRESHOLD, ENotRisky);
    // ... unwind logic
}
```

**Why Explicit Owner Checks:**

Even though Sui's object model provides some protection, we add explicit checks for:

1. **Clarity**: Code reviewers see authorization explicitly
2. **Defense in Depth**: Multiple layers of protection
3. **Future Proofing**: If objects become transferable, checks still work

**Admin Functions:**

For protocol admin operations (updating parameters, emergency pause), we use capability pattern:
```move
struct AdminCap has key, store {
    id: UID
}

public fun update_threshold(
    _: &AdminCap,  // Proves caller has admin capability
    new_threshold: u64
) {
    // Update protocol parameters
}
```

**Alternative Considered:**

Role-based access control (RBAC) with complex permission system.

Rejected because:
- Over-engineering for current needs
- Sui's object model sufficient
- More complexity = more attack surface

**Trade-offs:**

Pros:
- Simple and clear
- Leverages Sui's strengths
- Easy to audit
- Minimal code

Cons:
- Less flexible than RBAC
- Can't delegate partial permissions easily (not needed)

---

## Error Handling

### Decision: Descriptive Error Codes with Assert

**Choice Made:** Use descriptive error constants with assert statements for all failure cases.

**Rationale:**

Move handles errors through abort codes. We define clear, descriptive error codes for all failure scenarios.

**Implementation:**
```move
// Error codes
const EInsufficientCollateral: u64 = 1;
const EPositionNotRisky: u64 = 2;
const ESlippageTooHigh: u64 = 3;
const EInvalidHealthFactor: u64 = 4;
const ENotOwner: u64 = 5;

// Usage
public fun auto_unwind(...) {
    let hf = calculate_hf(...);
    assert!(hf < UNWIND_THRESHOLD, EPositionNotRisky);
    // ...
}
```

**Error Code Organization:**

- 1-99: Core logic errors
- 100-199: Position management errors
- 200-299: Health factor errors
- 300-399: Adapter errors
- 400-499: Access control errors

**Benefits:**

1. **Debuggability**: Clear error messages in failed transactions
2. **Testing**: Can test for specific errors
3. **Documentation**: Error codes document failure cases
4. **Frontend**: Can display user-friendly messages

**Alternative Considered:**

Generic error handling with minimal error codes.

Rejected because:
- Harder to debug
- Poor UX
- Difficult testing
- Not production-grade

**Trade-offs:**

Pros:
- Clear failure reasons
- Easy debugging
- Good UX
- Testable

Cons:
- More constants to maintain
- Need to document error codes

---

## Gas Optimization

### Decision: Optimize Critical Paths, Accept Trade-offs

**Choice Made:** Optimize gas for frequently called functions (looping, unwinding). Accept higher gas for rare operations (position creation).

**Rationale:**

Gas optimization requires trade-offs between code clarity and efficiency. We optimize where it matters most.

**Optimization Strategies:**

1. **Batch Operations**: Execute multiple loops in one transaction
```move
// Instead of N transactions
for loop in 1..target_loops {
    borrow();
    swap();
    supply();
}
// Do all in one transaction
```

2. **Minimal Storage**: Only store essential data in Position struct
```move
struct Position {
    // Essential only
    id: UID,
    owner: address,
    collateral_balance: Balance<T>,
    debt_balance: Balance<T>,
    total_supplied: u64,
    total_borrowed: u64,
    loop_count: u64,
    config: PositionConfig,
}
```

3. **Avoid Redundant Calculations**: Cache computed values when safe

4. **Owned Objects**: Use owned objects to avoid shared object consensus costs

**Where We DON'T Optimize:**

- Position creation: Happens once, can be slower
- Admin functions: Rare operations
- Test-only code: Clarity over efficiency

**Benchmarking Plan:**

Track gas costs on testnet:
- Position creation: Target < 0.5 SUI
- Single loop execution: Target < 0.1 SUI
- Auto-unwind: Target < 0.3 SUI

**Alternative Considered:**

Aggressive optimization everywhere.

Rejected because:
- Premature optimization is root of evil
- Hurts code readability
- Testnet first, optimize later
- Can always optimize more after mainnet data

**Trade-offs:**

Pros:
- Reasonable gas costs
- Maintainable code
- Can optimize more later

Cons:
- Not maximally optimized
- Some inefficiency accepted

---

## Testing Strategy

### Decision: Comprehensive Unit + Integration Tests

**Choice Made:** Write extensive tests covering all code paths, edge cases, and error conditions.

**Rationale:**

DeFi protocols handle user funds. Bugs = lost money. Comprehensive testing is non-negotiable.

**Test Categories:**

1. **Unit Tests**: Individual function validation
   - Test each function in isolation
   - Cover all branches and edge cases
   - Test error conditions

2. **Integration Tests**: End-to-end workflows
   - Full position lifecycle
   - Multiple loops
   - Auto-unwind scenarios

3. **Edge Case Tests**: Boundary conditions
   - Zero values
   - Maximum values
   - Precision limits

4. **Error Tests**: Failure paths
   - Invalid inputs
   - Unauthorized access
   - Insufficient funds

**Test Organization:**
```
tests/
├── position_tests.move          # Position creation and management
├── health_factor_tests.move     # HF calculation edge cases
├── looping_tests.move           # Loop execution
├── unwinding_tests.move         # Auto-unwind scenarios
├── access_control_tests.move    # Authorization
└── mock_tests.move              # Mock protocol validation
```

**Current Status:**

- Total tests: 178
- All passing: Yes
- Estimated coverage: 90-95%

**Testing Principles:**

1. **Test Behavior, Not Implementation**: Test what functions do, not how
2. **Test Errors**: Use #[expected_failure] for error cases
3. **Isolate Tests**: Each test independent
4. **Readable Tests**: Clear test names and structure

**Example Test:**
```move
#[test]
fun test_auto_unwind_when_risky() {
    // Setup: Create position with low HF
    let position = create_test_position();
    set_health_factor(&mut position, 1_200_000); // HF = 1.2
    
    // Execute: Call auto unwind
    auto_unwind(&mut position, &oracle);
    
    // Verify: Position partially unwound
    assert!(get_debt(&position) < initial_debt);
    assert!(get_health_factor(&position) > 1_300_000);
}

#[test]
#[expected_failure(abort_code = EPositionNotRisky)]
fun test_auto_unwind_fails_when_safe() {
    // Setup: Create safe position
    let position = create_test_position();
    set_health_factor(&mut position, 1_800_000); // HF = 1.8
    
    // Execute: Should fail
    auto_unwind(&mut position, &oracle); // Aborts here
}
```

**Alternative Considered:**

Minimal testing, rely on audit to find bugs.

Rejected because:
- Irresponsible for DeFi
- Audits complement tests, don't replace them
- Catch bugs early = cheaper to fix
- Tests = documentation

**Trade-offs:**

Pros:
- High confidence in code
- Catch bugs early
- Tests document behavior
- Easier refactoring

Cons:
- Time investment
- Maintenance burden
- Test code volume

---

## Testnet to Mainnet Migration

### Decision: Zero-Change Core, Adapter Swap Only

**Choice Made:** Core protocol logic never changes between testnet and mainnet. Only adapter imports change.

**Rationale:**

The adapter pattern exists specifically to make mainnet migration trivial and safe.

**Migration Process:**

1. Open `adapters.move`
2. Comment out mock imports
3. Uncomment real protocol imports
4. Update function bodies to call real protocols
5. Deploy

That's it. Zero changes to:
- Position logic
- Health factor calculations
- Looping execution
- Unwinding logic
- Access control
- Error handling

**Example Migration:**

Before (Testnet):
```move
// adapters.move
use looping_leverage::mock_lending;

public fun lending_supply<T>(...) {
    mock_lending::supply(...)
}
```

After (Mainnet):
```move
// adapters.move
use looping_leverage::navi_adapter;

public fun lending_supply<T>(...) {
    navi_adapter::supply(...)
}
```

**Why This Matters:**

1. **Risk Reduction**: Core logic already battle-tested on testnet
2. **Audit Efficiency**: Auditors review core once, adapters separately
3. **Confidence**: If it works on testnet, it works on mainnet
4. **Time Saving**: No core rewrites needed

**Validation:**

Before mainnet:
- Run full test suite with real adapters
- Deploy to testnet with real adapters first
- Validate behavior matches mock behavior
- Only then deploy to mainnet

**Alternative Considered:**

Rewrite core logic for mainnet "properly".

Rejected because:
- Defeats purpose of abstraction
- Re-introduces bugs
- Wastes development time
- Invalidates testnet testing

**Trade-offs:**

Pros:
- Safe migration
- Tested core logic
- Fast deployment
- Low risk

Cons:
- Adapters must perfectly match interfaces
- Requires discipline to maintain separation

---

## Summary

These technical decisions form the foundation of Yield Harbor's architecture. Key themes:

1. **Sui-Native Design**: Leverage Sui's unique features (owned objects, object model)
2. **Safety First**: Independent calculations, comprehensive testing, clear errors
3. **Simplicity**: Avoid over-engineering, optimize for clarity
4. **Future-Proof**: Adapter pattern enables evolution without rewrites
5. **Production-Grade**: Decisions made for mainnet from day one

---

**Last Updated:** January 8, 2026  
**Document Version:** 1.0  
**Next Review:** Before mainnet deployment