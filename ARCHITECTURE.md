# ARCHITECTURE — Solana Atomic Delta-Neutral Execution Engine

**Phase: Technical feasibility investigation — COMPLETE.**
**Verdict: implementation of the requested execution engine is STOPPED** per the task's
stop-condition: Jupiter Flashloan CPI cannot be verified — it is, in fact, provably
**unsupported** by the protocol. Details and evidence in `CPI_COMPATIBILITY.md`.

Date: 2026-11-15 · Branch `arena/3e5bcb28-test2` · No deployments performed.

---

## 1. Executive summary

| Question | Answer |
| --- | --- |
| Can Jupiter Flashloan `flashloan_borrow`/`flashloan_payback` be invoked via Anchor CPI? | **No** — the program rejects CPI calls (`FlashloanCPICallNotAllowed`, stack-height guard) and its repayment verification only recognizes top-level instructions (instructions-sysvar scan). |
| Can the repayment verification still be satisfied via CPI? | **No** — a CPI'd payback is invisible to the borrow-time matcher (`FlashloanPaybackNotFound` by construction). |
| Is Jupiter Earn (deposit/withdraw) CPI-capable? | **Yes** — official CPI guides + compiling reference code (`jup-ag/jupiter-lend`). |
| Is Jupiter Borrow (`init_position`, `operate`) CPI-capable? | **Yes** — official CPI guides + reference code (standard T1 vaults). |
| Is Manifest order creation / matching / cancellation / settlement CPI-composable? | **Yes** — the core program has no CPI restrictions and the official wrapper is itself a CPI program over the core (place/match/cancel/settle all included). |
| Program IDs verified per network? | **Yes, from official sources** (IDLs, `declare_id!`, Anchor.toml, CPI guides) — on-chain state not queried (no RPC here); devnet caveats apply. |

Compatibility matrix: `CPI_COMPATIBILITY.md` §2 (SUPPORTED / UNSUPPORTED / UNVERIFIED).
Accounts/PDAs: `ACCOUNT_MAP.md`. Addresses: `PROGRAM_ADDRESSES.md`. Limits:
`TRANSACTION_LIMITS.md`.

## 2. Evidence base

- **Jupiter Lend official integration repo** `jup-ag/jupiter-lend` @ `7960a14a…` (= current
  `main`): IDLs (`flashloan 0.1.4`, `lending 0.1.9`, `vaults 0.1.8`, `liquidity 0.1.9`,
  `oracle 0.1.8`, `dex 0.1.8`, `lending_reward_rate_model 0.1.9`), CPI guides for Earn and
  Borrow, and Rust CPI reference code compiled against `anchor-lang 0.31.1`.
- **Audited program source** (Code4rena `2026-02-jupiter-lend` scope snapshot): full
  `programs/flashloan/src` (incl. `validate.rs`), `programs/vaults`, `programs/lending`,
  `programs/liquidity`, `programs/oracle`; `declare_id!` values and error sets match the
  current IDLs exactly.
- **Corroboration**: Offside Labs Oct-2025 audit of the flashloan/oracle programs ("atomic
  loans that must be borrowed and repaid within a same transaction"), published in
  `jup-ag/docs` audit PDFs.
- **Manifest** `Bonasa-Tech/manifest` @ `d4e908b3…` (provided snapshot) diffed against
  `main` @ `52a6f56b…`: core instruction set unchanged; wrapper `Deposit`/`Withdraw` gained
  a `system_program` account (slot 8). Includes core (`manifest`), `wrapper`, `ui-wrapper`,
  clients, shank IDLs, Certora formal-verification suite.

## 3. The blocker (summary — full proof in `CPI_COMPATIBILITY.md` §3)

The flashloan program enforces three independent, mutually reinforcing guards:

1. `get_stack_height() > 1 ⇒ FlashloanCPICallNotAllowed` in **both** `flashloan_borrow`
   and `flashloan_payback` (top-level-only, hard-coded).
2. `flashloan_borrow` asserts that the *current top-level instruction* (read from
   `Sysvar1nstructions…`) is itself a flashloan-program instruction — false under any CPI.
3. Repayment verification scans the **top-level instruction list** for exactly one
   `flashloan_payback` with identical 14-account list and identical amount. CPI'd
   instructions do not appear in that list.

Consequences: the requested pattern (flashloan borrow + payback inside an Anchor program's
instruction flow) cannot be built; also **at most one flashloan per transaction** (plus a
protocol-wide `FlashloanAdmin` singleton with `FlashloanAlreadyActive`), so the two engines
cannot share a transaction when flashloans are involved, and the borrower must be a real
keypair signer.

**Per the task: STOP. The complete execution engine is not implemented in this phase.**

## 4. Target architecture (revised) — for approval before implementation

Because the flashloan must be composed at the transaction level while everything else can be
CPI-composed, the recommended shape is a **top-level sandwich around an Anchor engine
program** (alternative A, with D's lending leg):

```
Transaction (single, atomic):
┌────────────────────────────────────────────────────────────────────────┐
│ ix1  flashloan_borrow(amount)                    [top-level, keypair]  │
│ ix2  engine.execute_engine_{a|b}(…)              [Anchor program]      │
│        ├─ CPI manifest::Swap / SwapV2   (swap 1, min-out protected)    │
│        ├─ CPI vaults::operate           (deposit/borrow leg)           │
│        │     └─ (vaults → liquidity)                                   │
│        ├─ CPI lending::deposit/withdraw (earn leg, if used)            │
│        └─ CPI manifest::Swap / SwapV2   (swap 2, min-out protected)    │
│ ix3  flashloan_payback(amount)                   [top-level, keypair]  │
│ ix4  engine.verify_and_finalize(…)               [Anchor program]      │
│        └─ min-profit / slippage / balance-delta checks; failure        │
│           reverts the ENTIRE transaction (incl. borrow & payback)      │
└────────────────────────────────────────────────────────────────────────┘
```

Properties preserved from the brief:

- **Atomicity**: any failure anywhere reverts everything (Solana tx semantics); flashloan
  repayment is additionally enforced by the protocol itself at borrow time.
- **Fully on-chain**: every state transition (swaps, lend operations, checks) executes in
  on-chain programs; only instruction *ordering* is fixed at submission time — which is
  exactly the pattern the flashloan protocol mandates.
- **Two independent engines**: Engine A and Engine B are separate transactions with
  separate flashloans, parameters, positions, token accounts, and rollback behavior —
  required anyway by the one-flashloan-per-tx constraint.
- **Manifest exclusively** for swaps; **Jupiter Lend** for flashloan + earn + collateralized
  borrow; **no aggregator** anywhere.

Structural consequences to accept (or reject at design review):

1. The flashloan borrower is a hot/session keypair (PDAs cannot sign top-level
   instructions); borrowed funds transit its ATA; engine PDA custody must be reconciled
   (e.g. engine PDA as Manifest `SwapV2` owner; borrower signs `ix2` as a co-signer to
   authorize token legs).
2. `flashloan_borrow`/`payback` must carry byte-identical account lists and amounts.
3. Position NFTs (`init_position`) should be created **before** the hot path.
4. `FlashloanAdmin` lock contention: serialize engines or handle `FlashloanAlreadyActive`.

Fallback alternatives (documented in `CPI_COMPATIBILITY.md` §3): **B** — replace the
flashloan with collateralized borrow entirely via CPI (introduces LTV/oracle risk); **C** —
pre-funded float, no leverage; **D** — hybrid.

## 5. Gated implementation plan (does not start until the topology is approved)

1. **Validation harness (localnet / mainnet-clone)** — no devnet reliance:
   - `solana-test-validator` with cloned mainnet Lend programs + markets (per Jupiter's own
     guidance) and Manifest built from source (`--bpf-program MNFSTqt…/wMNFSTkir…`).
   - Prove on the clone: (i) CPI'd flashloan fails with `FlashloanCPICallNotAllowed`;
   (ii) sandwich succeeds end-to-end; (iii) double-flashloan tx fails; (iv) min-profit
   `verify` reverts cleanly.
2. **Anchor workspace skeleton** (`programs/atomic_engine`) with integrations modules
   (`jupiter_flashloan` → transaction-builder-side adapter only; `jupiter_earn`,
   `jupiter_borrow`, `manifest` → CPI adapters using the account maps in
   `ACCOUNT_MAP.md`), `execution/{settlement,profit,validation,risk}.rs`, errors.
3. **CPI adapters**: mirror the official reference code (`references/earn/*.rs`,
   `references/borrow/operate.rs`) with anchor-lang 0.31.x; Manifest via raw
   `Instruction{program_id: MNFSTqt…, data: [tag…]}` + `invoke`/`invoke_signed`.
4. **TypeScript SDK**: transaction builder emitting the sandwich layout; account resolvers
   for every PDA in `ACCOUNT_MAP.md`; market reader (Manifest book depth → best bid/ask);
   two independent engine clients (A/B) with their own configs
   (`config/devnet.json`, `config/mainnet.json` — cluster-scoped addresses only).
5. **Tests**: unit (discriminators, PDA derivations, profit math), integration (sandwich
   atomicity, rollback on each failure mode), security (slippage, oracle freshness,
   allowance of `u64::MAX` cases, delegate misuse).
6. **Deployment**: devnet scripts where meaningful (Manifest has no devnet — use localnet),
   mainnet script **gated behind explicit confirmation**; never automatic.
7. **Docs**: `SECURITY.md`, `DEPLOYMENT.md`, runbook for `FlashloanAlreadyActive` and
   `VaultInvalidNextPositionId` races.

Ordering note: step 1 exists precisely because this investigation could not query live
clusters; the flashloan conclusion is already proven from protocol source + IDL, so step 1
is confirmation, not discovery. If — contrary to all evidence — the clone shows CPI'd
flashloans succeeding, re-open `CPI_COMPATIBILITY.md` §3 before any code is written.

## 6. Explicit non-actions of this phase

- No Anchor program, SDK, or test code written (stop-condition honored).
- No deployments to any cluster (devnet/mainnet/localnet).
- No Jupiter Swap / aggregator usage in any design.
- No invented program IDs, discriminators, or PDA seeds (every one is sourced in
  `ACCOUNT_MAP.md` / `PROGRAM_ADDRESSES.md`).
- `operate_dex`/`operate_perfect_dex` (smart-debt vaults), the `dex` program, liquidations,
  and the ethena market are out of scope for v1 (UNVERIFIED territory, listed in
  `CPI_COMPATIBILITY.md` §2).

## 7. Deliverables index

| File | Content |
| --- | --- |
| `ARCHITECTURE.md` | This document — verdict, revised architecture, gated plan |
| `CPI_COMPATIBILITY.md` | Compatibility matrix (SUPPORTED/UNSUPPORTED/UNVERIFIED) + flashloan blocker proof |
| `PROGRAM_ADDRESSES.md` | Verified program IDs per network with sources and devnet caveats |
| `ACCOUNT_MAP.md` | Accounts, PDAs, signers, writability, remaining-accounts layouts |
| `TRANSACTION_LIMITS.md` | Runtime limits and composition constraints |
