# Project: Solana Atomic Delta-Neutral Execution Engine

Act as a senior Solana protocol engineer, Rust/Anchor developer, DeFi integration engineer, and security auditor.

Build a production-oriented Solana on-chain execution protocol integrating:

1. Jupiter Lend flashloans.
2. Jupiter Lend collateralized borrowing and repayment.
3. Manifest on-chain CLOB for swapping and order placement.
4. Solana Anchor for the core execution program.
5. TypeScript SDK and off-chain transaction construction.
6. Independent directional execution engines.
7. Devnet and mainnet deployment configurations.

## A. Core architecture

Build two completely independent execution engines.

### Engine A

Token1 → Token2 → Jupiter Lend operations → Token2 → Token1.

### Engine B

Token2 → Token1 → Jupiter Lend operations → Token1 → Token2.

Each engine must have:

- Independent flashloan.
- Independent execution parameters.
- Independent lending position.
- Independent token accounts.
- Independent minimum-profit calculation.
- Independent slippage protection.
- Independent transaction execution.
- Independent failure and rollback behavior.

Neither engine must depend on the success of the other.

## B. Jupiter Lend integration

Integrate the official Jupiter Lend interfaces.

Implement:

- Flashloan borrow.
- Flashloan repayment.
- Collateral supply.
- Debt borrowing.
- Partial repayment.
- Full repayment.
- Collateral withdrawal.
- Position validation.

Use official SDKs and CPI interfaces where supported.

Never assume that a borrow position can be withdrawn without satisfying its debt and collateral requirements.

## C. Manifest integration

Integrate Manifest's on-chain orderbook.

Implement:

- Market account discovery.
- Orderbook depth reading.
- Best bid and ask calculation.
- Market execution.
- Limit order placement.
- Order cancellation.
- Fill verification.
- Minimum-output protection.
- Maximum-slippage protection.

Use the official Manifest program interface and wrapper architecture.

Do not recreate the Manifest matching engine.

## D. Atomic execution

Each directional strategy must execute as a single atomic Solana transaction.

The execution sequence must validate:

1. Flashloan received.
2. First swap successfully executed.
3. Lending operation successfully executed.
4. Borrowing and repayment balances are correct.
5. Required collateral can be withdrawn.
6. Second swap successfully executed.
7. Flashloan fully repaid.
8. Minimum profit condition satisfied.

If any instruction fails, the entire transaction must revert.

Never allow an incomplete flashloan repayment.

## E. Profit protection

Implement:

- Minimum profit in Token1.
- Minimum profit in Token2.
- Maximum slippage.
- Maximum flashloan amount.
- Minimum orderbook liquidity.
- Maximum price deviation.
- Lending LTV validation.
- Oracle freshness validation.
- Token balance validation.
- Transaction compute budget validation.

Do not execute if the projected net result is below the configured threshold.

## F. Network configurations

Support:

- Devnet.
- Mainnet-beta.

Use separate configuration files for:

- RPC endpoints.
- Program IDs.
- Token mint addresses.
- Market addresses.
- Lending vault addresses.
- Oracle accounts.
- Flashloan program addresses.

Never reuse devnet addresses on mainnet.

Require an explicit mainnet deployment confirmation.

## G. Deliverables

Create:

1. Anchor Rust program.
2. Jupiter Lend CPI adapters.
3. Manifest CPI adapter.
4. TypeScript SDK.
5. Two independent execution clients.
6. Transaction builder.
7. Simulation engine.
8. Unit tests.
9. Integration tests.
10. Devnet deployment scripts.
11. Mainnet deployment scripts.
12. Environment configuration examples.
13. Security documentation.
14. Complete README.

## H. Development rules

- Inspect official protocol source code and IDLs before implementing integrations.
- Do not invent program IDs or instruction discriminators.
- Do not fabricate devnet deployments.
- Do not substitute mocked integrations for production CPI code without clearly identifying them.
- Verify every CPI account, signer, writable flag and PDA.
- Compile the Anchor program.
- Run tests and transaction simulations.
- Document every unsupported feature.
- Do not deploy to mainnet automatically.

First produce the architecture, integration compatibility report, and implementation plan. Do not generate the entire codebase before validating the protocol interfaces.




solana-atomic-engine/
│
├── programs/
│   └── atomic_engine/
│       └── src/
│           ├── lib.rs
│           ├── state/
│           │   ├── config.rs
│           │   ├── engine_state.rs
│           │   └── position.rs
│           │
│           ├── instructions/
│           │   ├── initialize.rs
│           │   ├── execute_engine_a.rs
│           │   ├── execute_engine_b.rs
│           │   ├── close_position.rs
│           │   └── update_config.rs
│           │
│           ├── integrations/
│           │   ├── jupiter_flashloan.rs
│           │   ├── jupiter_earn.rs
│           │   ├── jupiter_borrow.rs
│           │   └── manifest.rs
│           │
│           ├── execution/
│           │   ├── settlement.rs
│           │   ├── profit.rs
│           │   ├── validation.rs
│           │   └── risk.rs
│           │
│           └── errors.rs
│
├── client/
│   ├── src/
│   │   ├── engine-a.ts
│   │   ├── engine-b.ts
│   │   ├── transaction-builder.ts
│   │   ├── market-reader.ts
│   │   └── account-resolver.ts
│
├── tests/
│   ├── flashloan-cpi.ts
│   ├── manifest-cpi.ts
│   ├── jupiter-borrow.ts
│   ├── engine-a.ts
│   ├── engine-b.ts
│   ├── atomicity.ts
│   └── security.ts
│
├── config/
│   ├── devnet.json
│   └── mainnet.json
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── CPI_COMPATIBILITY.md
│   ├── SECURITY.md
│   └── DEPLOYMENT.md
│
├── Anchor.toml
├── Cargo.toml
└── README.md
