# Yield Harbor

> Automated liquidation protection for leveraged positions on Sui

Yield Harbor is a looping leverage protocol that allows users to create leveraged positions with automatic risk management. The protocol monitors health factors and automatically unwinds positions before liquidation occurs.

---

## Overview

Traditional leveraged DeFi positions require constant manual monitoring. Users risk liquidation during market volatility, especially during off-hours. Yield Harbor solves this by automating risk management on-chain.

**Key Features:**
- One-click leveraged position creation
- Automatic health factor monitoring
- Permissionless auto-unwinding before liquidation
- Sui-native architecture using owned objects
- Clean adapter pattern for protocol abstraction

---

## How It Works

### Looping Leverage

Users deposit collateral once, and the protocol automatically loops it multiple times:
```
Initial: 100 WETH
Loop 1: Deposit → Borrow 70 USDC → Swap to WETH → Supply
Loop 2: 170 WETH → Borrow 49 USDC → Swap to WETH → Supply
Loop 3: 219 WETH → Borrow 34 USDC → Swap to WETH → Supply
Result: 2.53x leverage from single deposit
```

### Automatic Risk Management

The protocol continuously monitors health factors:
- HF > 1.6: Safe, normal operations
- HF 1.3-1.6: Warning, pause new loops
- HF 1.1-1.3: Auto-unwind 25%
- HF < 1.1: Auto-unwind 50%
- HF < 1.0: Emergency full unwind

Keeper bots submit unwind transactions, but all logic is enforced on-chain. The bot cannot cheat or manipulate outcomes.

---

## Architecture

### Core Components

**Position System**  
Positions are owned objects stored in user wallets. Each position tracks collateral, debt, and configuration independently.

**Health Factor Engine**  
Independent HF calculation using current prices and lending parameters. Never relies on external protocol health factors.

**Unified Adapter Layer**  
Single module that abstracts all external protocol interactions (lending, swaps, oracles). Mocks on testnet, real protocols on mainnet.

**Automation Layer**  
Permissionless keeper functions that anyone can call. On-chain logic enforces all safety rules.

### Module Structure
```
looping_leverage/
├── sources/
│   ├── core/
│   │   ├── position.move           # Position struct and lifecycle
│   │   ├── health_factor.move      # HF calculation logic
│   │   ├── looping.move            # Loop execution
│   │   └── unwinding.move          # Unwind logic
│   │
│   ├── mocks/
│   │   ├── mock_lending.move       # Simulated lending protocol
│   │   ├── mock_swap.move          # Simulated DEX
│   │   ├── mock_oracle.move        # Price feed simulation
│   │   └── mock_tokens.move        # Test tokens
│   │
│   ├── adapters.move               # Unified adapter layer
│   └── keeper.move                 # Automation functions
│
└── tests/
    └── *.move                      # 178 passing tests
```

---

## Getting Started

### Prerequisites

- Sui CLI installed
- Rust toolchain
- Node.js (for frontend/keeper bot)

### Installation
```bash
# Clone the repository
git clone <repo-url>
cd yield-harbor

# Build the project
sui move build

# Run tests
sui move test
```

### Running Tests
```bash
# Run all tests
sui move test

# Run specific test file
sui move test --filter position_tests

```

Expected output: 178 tests passing

---

## Deployment

### Testnet Deployment

The protocol uses mock implementations on testnet for safe testing:
```bash
# Publish to testnet
sui client publish --gas-budget 500000000

# Initialize mock protocols
sui client call --function init_mock_market --package <PACKAGE_ID>
```

Contract addresses and deployment logs are tracked in the `deployments/` directory.

### Mainnet Migration

Mainnet deployment only requires changing imports in `adapters.move`:

1. Comment out mock imports
2. Uncomment real protocol imports (Navi, Pyth, Cetus)
3. Republish

Core logic remains unchanged. See `docs/architecture.md` for details.

---

## Testing

The project includes comprehensive test coverage:

- **Unit Tests:** Individual function validation
- **Integration Tests:** Full position lifecycle testing
- **Edge Case Tests:** Boundary conditions and error paths
- **Mock Protocol Tests:** Adapter layer validation

### Test Categories
```
Core Module Tests:        45 tests
Position Management:      32 tests
Auto-Unwind Logic:        28 tests
Access Control:           18 tests
Mock Protocols:           55 tests
Total:                   178 tests
```

All tests must pass before deployment.

---

## Documentation

- `docs/architecture.md` - Detailed technical architecture
- `docs/milestones.md` - Development roadmap and progress
- `docs/technical-decisions.md` - Key design decisions explained
- `progress/` - Weekly development updates

---

## Security

### Design Principles

**On-Chain Enforcement**  
All critical logic executes on-chain. Bots and indexers are untrusted components that cannot manipulate outcomes.

**Independent Calculations**  
Health factors are calculated internally using raw data. External protocol values are never trusted.

**Permissionless Safety**  
Anyone can call keeper functions, but the chain enforces strict safety checks. Bots cannot trigger unwinding unless conditions are genuinely met.

**Object Ownership**  
Positions are owned by users as Sui objects. Authorization is handled through the object model, not approval mechanisms.

### Audit Status

- Internal testing: Complete (178/178 tests passing)
- Testnet deployment: In progress
- External audit: Planned before mainnet

---

## Roadmap

### Phase 1: Core Development (Complete)
- Protocol architecture design
- Core module implementation
- Mock protocol development
- Comprehensive test suite

### Phase 2: Testnet (Current)
- Deploy to Sui testnet
- Build frontend interface
- Implement keeper bot
- Community testing and feedback

### Phase 3: Mainnet Preparation
- Real protocol adapters (Navi, Pyth, Cetus)
- Security audit
- Performance optimization
- Documentation finalization

### Phase 4: Mainnet Launch
- Mainnet deployment
- Liquidity mining program
- Partnership integrations
- Ongoing monitoring and updates

---


## Resources

**Sui Ecosystem:**
- [Sui Documentation](https://docs.sui.io)
- [Move Language Book](https://move-book.com)
- [Sui Testnet Explorer](https://suiscan.xyz/testnet)

**Related Protocols:**
- Navi Protocol (Lending)
- Cetus Protocol (DEX)
- Pyth Network (Oracles)

**Inspiration:**
- Aave V3 (Health factor model)
- Instadapp (Leverage automation)
- DeFi Saver (Position management)

---

## License

MIT License - see LICENSE file for details

---

## Contact

- Website: [Coming Soon]
- Twitter: [Coming Soon]
- Discord: [Coming Soon]
- Email: [pritulsingh225@gmail.com]

---

## Acknowledgments

Built during the Sui ecosystem growth phase. Special thanks to the Sui Foundation, Move community, and DeFi protocol teams for their open-source contributions and documentation.

---

**Current Status:** Testnet Deployment Phase  
**Last Updated:** January 8, 2026  
**Version:** 0.1.0-testnet