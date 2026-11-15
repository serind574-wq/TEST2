# CPI Compatibility Report — Solana Atomic Delta-Neutral Execution Engine

**Status: FEASIBILITY INVESTIGATION COMPLETE — IMPLEMENTATION STOPPED (see §3).**
No execution-engine code has been written, per the task's stop-condition.

Date: 2026-11-15
Scope: Jupiter Lend (flashloan / earn / borrow) + Manifest (Bonasa-Tech) composability via
on-chain CPI from an Anchor program, for devnet and mainnet-beta.

---

## 1. Evidence base and verification methodology

Every verdict below is backed by artifacts inspected in this investigation. Three evidence
tiers are used:

| Tier | Meaning | Sources used |
| ---- | ------- | ------------ |
| **E1 — Official interface** | Official IDL, CPI guide or CPI reference code published by the protocol team | `jup-ag/jupiter-lend` @ `7960a14a5a760abc41db3ed1395dc5cb56a1f8db` (matches current `main`; contains `target/idl/*.json`, `docs/*/cpi.md`, `references/**/*.rs`); `Bonasa-Tech/manifest` @ `d4e908b34f7a050d63cad38cb7175b2e88008465` + current `main` @ `52a6f56b58995666d563d98f449f1285e8e9a3bf` (diff-checked; only wrapper Deposit/Withdraw gained `system_program`) |
| **E2 — Audited source snapshot** | Protocol source code from the official audit scope, cross-checked against E1 interfaces | `code-423n4/2026-02-jupiter-lend` audit scope (`programs/flashloan/src/{lib.rs,validate.rs,state/context.rs,state/state.rs,constants.rs,errors.rs}`), retrieved from the sponsor source tree; error set 6000–6013 and `declare_id!` values match the current `flashloan.json` IDL v0.1.4 byte-for-byte |
| **E3 — Corroboration** | Third-party/derived confirmation | Offside Labs "JupiterLend / Fluid Solana" audit (Oct 2025, `jup-ag/docs` audit PDFs) confirming flashloan design ("requires repayment within the same transaction"); audit scope file listing `programs/flashloan/src/validate.rs` (118 nSLoC) as the validation module |

**Explicit limitation:** this environment has no RPC access to Solana clusters. Program IDs
were verified against official repositories/IDLs/Anchor configs (triangulated, see
`PROGRAM_ADDRESSES.md`), **not** against on-chain account state. Where on-chain state matters
(e.g. devnet deployment drift) it is marked UNVERIFIED.

**Rules honored:** SDK instruction builders were *not* treated as proof of CPI compatibility;
no program IDs were invented; Jupiter Swap / any aggregator was not used or considered; no
off-chain substitute is proposed for required on-chain steps without being labeled as such.

---

## 2. Compatibility matrix

Legend: **SUPPORTED** = verified callable via CPI with protocol-sanctioned interface or
in-protocol proof. **UNSUPPORTED** = verified that the protocol forbids or structurally
prevents the call path. **UNVERIFIED** = cannot be confirmed from available artifacts.

### 2.1 Jupiter Lend — Flashloan program (`flashloan` v0.1.4)

| Operation | Top-level ix | CPI from Anchor program | Status | Evidence |
| --- | --- | --- | --- | --- |
| `flashloan_borrow` | OK | **Rejected by protocol** | **UNSUPPORTED (via CPI)** / SUPPORTED (top-level) | E1 IDL errors `FlashloanCPICallNotAllowed`, `FlashloanInvalidInstructionSysvar`, `FlashloanPaybackNotFound`; E2 `validate.rs` (see §3) |
| `flashloan_payback` | OK | **Rejected by protocol** | **UNSUPPORTED (via CPI)** / SUPPORTED (top-level) | E2 `validate_flashloan_payback` stack-height guard; E2 `validate_flashloan` sysvar scan only sees top-level ixs |
| Repayment verification satisfied when both are CPIs | n/a | **Impossible** | **UNSUPPORTED** | Instructions sysvar only enumerates top-level instructions; a CPI'd payback is invisible to the borrow-time scan (`FlashloanPaybackNotFound`) |
| ≥ 2 flashloans in one transaction (2 engines) | Blocked | n/a | **UNSUPPORTED** | E2 scan logic: any other flashloan-program instruction between/after a borrow and its payback → `FlashloanInvalidInstruction` or `FlashloanMultiplePaybacksFound`; `FlashloanAdmin` singleton enforces `FlashloanAlreadyActive` |
| `init_flashloan_admin` / `set_flashloan_fee` / `activate_protocol` / `pause_protocol` / `update_authority` | Admin only | n/a | N/A for engine | E1 IDL — authority-gated, not part of the engine surface |

### 2.2 Jupiter Lend — Earn (`lending` v0.1.9)

| Operation | CPI from Anchor program | Status | Evidence |
| --- | --- | --- | --- |
| `deposit` (`assets: u64`) | Yes — official CPI reference | **SUPPORTED** | E1 `docs/earn/cpi.md`, `references/earn/deposit.rs`, `lending.json` disc `[242,35,198,137,82,225,242,182]` (cross-checked IDL↔docs↔reference) |
| `withdraw` (`amount: u64`) | Yes — official CPI reference | **SUPPORTED** | E1 `docs/earn/cpi.md`, `references/earn/withdraw.rs`, disc `[183,18,70,156,148,109,161,34]` |
| `deposit_with_min_amount_out` | Yes (same accounts + 1 arg) | **SUPPORTED** | E1 `docs/earn/cpi.md` §5, disc `[116,144,16,97,118,109,40,119]` |
| `withdraw_with_max_shares_burn` | Yes (same accounts + 1 arg) | **SUPPORTED** | E1 `docs/earn/cpi.md` §5, disc `[47,197,183,171,239,18,245,171]` |
| Return-data readback (shares minted/burned) | `get_return_data()` after CPI | **SUPPORTED** | E1 reference code `read_u64_return_data` |

### 2.3 Jupiter Lend — Borrow / Vaults (`vaults` v0.1.8)

| Operation | CPI from Anchor program | Status | Evidence |
| --- | --- | --- | --- |
| `init_position` (`vault_id: u16`, `next_position_id: u32`) | Yes — official CPI reference | **SUPPORTED** | E1 `docs/borrow/cpi.md` §1, `references/borrow/operate.rs`, disc `[197,20,10,1,97,160,177,91]` |
| `operate` (deposit / withdraw / borrow / payback / combos, `i128` signed, `i128::MIN` = max) | Yes — official CPI reference | **SUPPORTED** | E1 `docs/borrow/cpi.md` §2, `references/borrow/operate.rs` (incl. `deposit`, `withdraw`, `borrow`, `payback`, `deposit_and_borrow`, `payback_and_withdraw` wrappers), disc `[217,106,208,99,116,151,42,135]` |
| `operate` with `TransferType::Claim` | Yes (claim accounts required) | **SUPPORTED** | E1 `docs/borrow/cpi.md` §5 |
| `operate` via PDA-owned position (`invoke_signed`) | Yes | **SUPPORTED** | E1 `docs/borrow/cpi.md` §2 note 2 ("use `invoke_signed` with the PDA's seeds") |
| `operate_dex` / `operate_perfect_dex` (smart-collateral / smart-debt vaults) | Not covered by CPI guide; `operate` fails on these vaults (`VaultInvalidVaultTypeForOperate`) | **UNVERIFIED** (out of scope) | E1 `docs/borrow/cpi.md` overview; `vaults.json` discs `[223,122,223,181,133,132,116,33]` / `[88,34,180,77,152,186,166,162]` exist in IDL but no CPI reference ships |
| Liquidation path (`utils/liquidate.rs`) | Not in CPI guide | **UNVERIFIED** (out of scope) | E2 source exists; no official CPI interface doc |

### 2.4 Manifest (Bonasa-Tech) — core program (`manifest`)

| Operation (core ix) | CPI from Anchor program | Status | Evidence |
| --- | --- | --- | --- |
| `CreateMarket` (0) | Yes (any program may create) | **SUPPORTED** | E1 `programs/manifest/src/program/instruction.rs`; official wrapper design (core is an "infrastructure layer primitive", meant to be CPI'd) |
| `ClaimSeat` (1) | Yes — wrapper does it | **SUPPORTED** | E1 `programs/wrapper/src/processors/claim_seat.rs` (`invoke` into core) |
| `Deposit` (2) / `Withdraw` (3) | Yes — wrapper does it | **SUPPORTED** | E1 `processors/deposit.rs`, `processors/withdraw.rs` |
| `Swap` (4) — taker, wallet funds, exact-in w/ min-out or exact-out w/ max-in | Yes | **SUPPORTED** | E1 `SwapParams { in_atoms, out_atoms, is_base_in, is_exact_in }` w/ `minimum_output_satisfied` check; wrapper `invoke_passthrough(&manifest::id(), …)` forwards core ixs |
| `Expand` (5) | Yes | **SUPPORTED** | E1 instruction table; auto-expansion also occurs inside core ixs |
| `BatchUpdate` (6) — **order creation + cancellation + matching + settlement in one ix** | Yes — wrapper does it | **SUPPORTED** | E1 `processors/batch_upate.rs` → `invoke_passthrough(&manifest::id(), &accounts[2..], &data)`; `BatchUpdateParams { cancels, orders }`, `OrderType { Limit, ImmediateOrCancel, PostOnly, Global, Reverse, ReverseTight }` |
| `GlobalCreate/AddTrader/Deposit/Withdraw/Evict/Clean` (7–12) | Yes | **SUPPORTED** (not required for engine v1) | E1 instruction table + wrapper `BatchUpdateBaseGlobal`/`BatchUpdateQuoteGlobal` |
| `SwapV2` (13) — `owner` separate from `payer` (router/PDA hops) | Yes | **SUPPORTED** | E1 instruction table ("allows routers to … have intermediate hops go through PDAs") |
| `Defrag` (14) | Yes (collector-gated) | **SUPPORTED** (ops only) | E1 instruction table |
| Stack-height / CPI-call guard in core | None exists | **SUPPORTED** (no blocker) | Grep of `programs/manifest` + `lib/`: no `get_stack_height`, no instructions-sysvar checks, no CPI-call restrictions; pinocchio entrypoint dispatches on a 1-byte tag |

**Manifest matching/settlement semantics (verified E1):** there is no separate crank/match
instruction. Matching executes *inside* `Swap`/`SwapV2` (taker) and inside `BatchUpdate`'s
place-order phase (maker: `add_order` walks the opposite book, fills settle immediately
against market vaults; rounding is in taker favor). Cancellation is `BatchUpdate` cancels
(`CancelOrderParams { order_sequence_number, order_index_hint }`). Settlement of a fill is
atomic within the placing instruction; settlement of resting balances is `Withdraw` (or the
UI wrapper's `SettleFunds`). Therefore **order creation, matching, cancellation and
settlement are all composable through CPI** — demonstrated in production by both official
wrappers (which are themselves CPI programs over the core).

### 2.5 Manifest — wrapper programs

| Wrapper ix (`wrapper`, `wMNFSTkir3HgyZTsB7uqu3i7FA73grFCptPXgrZjksL`) | Status | Notes |
| --- | --- | --- |
| `CreateWrapper` 0, `ClaimSeat` 1, `Deposit` 2, `Withdraw` 3, `BatchUpdate` 4, `BatchUpdateBaseGlobal` 5, `BatchUpdateQuoteGlobal` 6, `Collect` 7, `Defrag` 8 | **SUPPORTED** | E1 `programs/wrapper/src/instruction.rs`; Deposit/Withdraw gained a required `system_program` at slot 8 upstream (commit range `d4e908b..52a6f56`) — processors still read it optionally |

| UI-wrapper ix (`ui-wrapper`, `UMnFStVeG1ecZFc2gc5K3vFy3sMpotq8C91mXBQDGwh`) | Status | Notes |
| --- | --- | --- |
| `CreateWrapper` 0, `ClaimSeatUnused` 1, `PlaceOrder` 2, `EditOrder` 3 (marked not-implemented upstream), `CancelOrder` 4, `SettleFunds` 5, `Collect` 6 | **SUPPORTED** (reference) | E1 `programs/ui-wrapper/src/instruction.rs`; UI wrapper adds fee machinery (platform/referrer) — not needed for the engine |

### 2.6 Matrix summary (task-required format)

| Integration point | Status |
| --- | --- |
| Jupiter Flashloan `flashloan_borrow` via CPI | **UNSUPPORTED** |
| Jupiter Flashloan `flashloan_payback` via CPI | **UNSUPPORTED** |
| Jupiter Flashloan borrow+payback as **top-level** ixs with protocol repayment verification | **SUPPORTED** |
| Jupiter Flashloan inside a single top-level engine instruction (both ops as CPIs) | **UNSUPPORTED** |
| Jupiter Earn `deposit` / `withdraw` (+ slippage variants) via CPI | **SUPPORTED** |
| Jupiter Borrow `init_position` via CPI | **SUPPORTED** |
| Jupiter Borrow `operate` (all standard-vault patterns) via CPI | **SUPPORTED** |
| Jupiter Borrow `operate_dex` / `operate_perfect_dex` via CPI | **UNVERIFIED** |
| Manifest core order create / match / cancel / settle via CPI | **SUPPORTED** |
| Manifest `Swap`/`SwapV2` (min-out / max-in protected) via CPI | **SUPPORTED** |
| Manifest wrapper / UI wrapper instructions via CPI | **SUPPORTED** |
| Flashloan devnet deployment parity with current IDL | **UNVERIFIED** |
| `FlashloanNotAllowedInThisSlot` (error 6006) semantics | **UNVERIFIED** (declared in IDL and source, unreferenced in audited snapshot) |
| Manifest devnet deployment | **UNVERIFIED** (none found in official repo — see `PROGRAM_ADDRESSES.md`) |

---

## 3. THE BLOCKER — Jupiter Flashloan cannot be invoked through CPI

**Both `flashloan_borrow` and `flashloan_payback` hard-reject CPI invocation, and the
repayment verification structurally cannot see CPI'd instructions.** This is not an SDK
limitation; it is enforced inside the deployed program. Three independent guards, from the
audited source (`programs/flashloan/src/validate.rs`, E2, consistent with the official
v0.1.4 IDL error set):

### Guard 1 — explicit stack-height check (both instructions)

```rust
// constants.rs
pub const FLASHLOAN_STACK_HEIGHT: usize = 1;

// validate_flashloan() and validate_flashloan_payback()
if get_stack_height() > constants::FLASHLOAN_STACK_HEIGHT {
    return Err(ErrorCodes::FlashloanCPICallNotAllowed.into());   // error 6012
}
```

`get_stack_height()` returns 1 only while executing a **top-level** transaction instruction.
Any CPI raises the height to ≥ 2 and is rejected with `FlashloanCPICallNotAllowed`.

### Guard 2 — instructions-sysvar current-instruction check (`flashloan_borrow`)

```rust
let current_index = load_current_index_checked(ix_sysvar_account)?;
let current_ixs   = load_instruction_at_checked(current_index as usize, ix_sysvar_account)?;
if current_ixs.program_id != *ctx.program_id {
    return Err(ErrorCodes::FlashloanInvalidParams.into());
}
```

The instructions sysvar enumerates **top-level instructions only**. When `flashloan_borrow`
is reached via CPI from our Anchor program, "current instruction" is *our* instruction
(program id = engine), so this check fails even if Guard 1 were absent.

### Guard 3 — repayment verification scans only top-level instructions

`validate_flashloan` requires that a matching `flashloan_payback` instruction exists **later
in the top-level instruction list** of the same transaction:

- scans `current_index + 1 .. instruction_count` (reverse) for instructions whose
  `program_id` is the flashloan program;
- a candidate must have exactly 16 bytes of data (`[213,47,153,137,84,243,94,232]` +
  `u64` amount), the **exact same 14-account list in the exact same order** as the borrow,
  and `amount == borrow_amount`;
- exactly one match allowed (`FlashloanMultiplePaybacksFound` on two), zero matches →
  `FlashloanPaybackNotFound`, any other flashloan-program ix in the window →
  `FlashloanInvalidInstruction`.

Consequences (all verified against the scan logic):

1. A `flashloan_payback` reached via CPI is **invisible** to this scan — the sysvar only
   contains the outer engine instruction. Even if a runtime quirk let the CPIs execute, the
   borrow would fail with `FlashloanPaybackNotFound`. The repayment invariant and CPI
   invocation are mutually exclusive **by construction**.
2. **At most one flashloan per transaction.** A second borrow/payback pair anywhere after
   the first borrow trips `FlashloanInvalidInstruction` (discriminator mismatch) or
   `FlashloanMultiplePaybacksFound`. Additionally the `FlashloanAdmin` singleton state
   (`is_flashloan_active`, `active_flashloan_amount`) rejects overlapping loans with
   `FlashloanAlreadyActive`. The two engines therefore **cannot share a transaction** when
   flashloans are involved.
3. Borrow and payback must carry **identical account lists** (same keys, same order, 14
   accounts each) and identical amounts. The engine cannot smuggle extra accounts into the
   flashloan instructions.
4. The repayment is enforced *at borrow time* (missing payback → transaction fails before
   any funds move) and *at payback time* (`amount == active_flashloan_amount`, fee =
   `amount + ceil(amount·fee_bps/10_000)`). If any instruction in the transaction fails,
   Solana's transaction atomicity reverts the borrow as well. The protocol's safety
   argument only works for top-level compositions.

### Answer to task item 3

> Can both flashloan operations be invoked through CPI from an Anchor program and still
> satisfy the protocol's repayment verification?

**No.** `flashloan_borrow` and `flashloan_payback` reject CPI calls
(`FlashloanCPICallNotAllowed`), and the repayment verification (`FlashloanPaybackNotFound`
machinery over `Sysvar1nstructions…`) can only pair **top-level** borrow/payback
instructions. The requested pattern — flashloan wrapped inside one atomic Anchor
instruction — is **UNSUPPORTED** and unimplementable against the deployed protocol.

### Architectural alternatives (investigated, not implemented)

| # | Alternative | Atomic? | On-chain? | Notes |
| --- | --- | --- | --- | --- |
| **A** | **Top-level sandwich transaction**: `flashloan_borrow` → `engine.execute_*` (all Manifest + Lend steps via CPI) → `flashloan_payback` → `engine.verify_profit` (reverts whole tx if min-profit/slippage checks fail) | Yes — transaction-level all-or-nothing | Yes | Preserves the flashloan and full on-chain execution of the delta-neutral body. Flashloan instructions are top-level (as the protocol requires), engine logic is still a single Anchor program. Constraint: borrower must be a real transaction signer (PDAs cannot sign top-level ixs); borrowed funds transit the signer's ATA; one flashloan per tx → engines run as separate transactions. |
| **B** | **Collateralized borrow instead of flashloan**: Jupiter Borrow `init_position` + `operate` (deposit collateral, borrow debt) entirely via CPI; repay/withdraw later | Per-tx atomic; debt persists across txs | Yes | Fully CPI-compatible today (§2.3). Removes same-tx repayment constraint; introduces LTV/oracle/liquidation risk that the engine must manage (`VaultPositionAboveCF`, oracle freshness). |
| **C** | **Pre-funded PDA float** (no flashloan): engine PDA holds inventory, all steps via CPI | Yes (single instruction) | Yes | Simplest CPI graph; capital-inefficient; no leverage. |
| **D** | **Hybrid**: A for capital efficiency on the flashloan leg + B for the lending leg | Yes for body | Yes | Matches the brief's "Lend operations" better than A alone. |

Recommended direction if/when implementation proceeds: **A (+ D's lending leg)**, with B as
the fallback if top-level flashloan composition conflicts with other requirements.

---

## 4. What remains UNVERIFIED and how to verify before implementation

1. **On-chain parity of program IDs / versions** (no RPC in this environment): run
   `solana program show <id> --url mainnet-beta` for every address in
   `PROGRAM_ADDRESSES.md`; compare deployed Anchor versions with the IDLs
   (`flashloan 0.1.4`, `lending 0.1.9`, `liquidity 0.1.9`, `vaults 0.1.8`, `oracle 0.1.8`).
2. **Flashloan behavior on a live cluster**: build the sandwich of §3-A against a
   mainnet-fork local validator (`solana-test-validator --clone` of the Lend programs +
   accounts, as `docs/earn/cpi.md` recommends) and assert: (i) CPI'd borrow fails with
   `FlashloanCPICallNotAllowed`; (ii) top-level sandwich succeeds; (iii) a transaction with
   two flashloans fails.
3. **Devnet drift**: official docs state the devnet programs (May 2025) *do not match* the
   current IDLs; the flashloan devnet address comes from the audit build config (staging
   `declare_id!`) and is **not** listed in the official CPI guides. Treat devnet as
   non-representative; never reuse devnet addresses on mainnet.
4. **`operate_dex` / `operate_perfect_dex`** if smart-debt vaults are ever required (out of
   scope for v1).
5. **Manifest devnet**: no official devnet deployment found; localnet testing uses the
   mainnet IDs via `solana-test-validator --bpf-program …` (see
   `PROGRAM_ADDRESSES.md`).

---

## 5. Consequences for the requested engine design

Per the task's stop-condition, **implementation of the complete execution engine is halted
here.** The requested design (both flashloan operations inside an Anchor CPI flow) is not
achievable; any implementation must first adopt alternative A/B/C/D above and re-run this
compatibility review against the chosen topology. Deliverables produced at this stage:

- `ARCHITECTURE.md` — feasibility verdict, proposed revised topologies, implementation plan
  gated on the verdict.
- `CPI_COMPATIBILITY.md` — this document (matrix + blocker).
- `PROGRAM_ADDRESSES.md` — verified program IDs per network with sources.
- `ACCOUNT_MAP.md` — accounts / PDAs / signers / writability / remaining-accounts layouts.
- `TRANSACTION_LIMITS.md` — runtime limits and composition constraints.
