# Transaction & Composition Limits

Date: 2026-11-15

Runtime limits are from the Solana runtime (Agave v3.x values, as compiled in the repo's
`SolanaCPI.md` reference; SIMD-0268 / SIMD-0339 relaxations noted). Protocol-level limits
are from the inspected sources.

## 1. Solana runtime limits relevant to this engine

| Limit | Value | Impact on engine |
| --- | --- | --- |
| Transaction size | 1232 bytes (packet) | `operate` alone carries 35+ accounts; two swaps + lend + flashloan in **one tx** will pressure this — use address lookup tables (ALTs) and compact layouts |
| Max instruction stack depth | 5 (9 with SIMD-0268) | Engine → vaults → liquidity = depth 3 ✔; Engine → flashloan = depth 2 ✘ (rejected by protocol, see §3) |
| CPI invocation cost | 1,000 CU (946 w/ SIMD-0339) | Negligible vs. matching/operate logic |
| Max PDA signers per CPI | 16 | Sufficient (engine PDA only) |
| Max CPI instruction data | 10 KiB | `operate` payload: 8+16+1+2+`remaining_accounts_indices`… trivial; Manifest `BatchUpdate` with many orders must stay under this |
| Max return data | 1,024 bytes | `operate` returns 28 bytes, `deposit`/`withdraw` 8 bytes ✔ |
| Max CPI account infos | 128 (255 w/ SIMD-0339) | One `operate` call passes 35 + remaining; several CPIs per instruction fit comfortably |
| CPI serialization cost | 1 CU / 250 bytes | Budget in compute estimates |
| Max realloc per CPI | 10 KiB | Manifest market expansion is handled **inside** the core program (self-CPI not needed) |
| Compute budget | 200k default / 1.4M max per ix (`ComputeBudget` program) | `operate` + matching + oracle reads are CU-heavy; request 600k–1.4M and measure per path |
| Account locks | 64 writable accounts/tx (well-known practical limit is lower with contention) | See §3 singleton contention |

## 2. Instruction-data / account-shape constraints (protocol-level)

| Item | Constraint | Source |
| --- | --- | --- |
| `flashloan_payback` data | exactly 16 bytes (8-byte disc + u64 LE amount) — anything else makes the borrow-time matcher return `FlashloanInvalidInstruction` | `validate.rs::is_flashloan_payback_instruction` |
| `flashloan_borrow`/`payback` account lists | identical 14 accounts, identical order | `validate.rs` compares full list |
| One flashloan per transaction | any other flashloan-program ix after a borrow → `FlashloanInvalidInstruction` / `FlashloanMultiplePaybacksFound` | `validate.rs::validate_payback_instruction_exists` |
| Flashloan amount | ≥ 1,000 atoms; payback amount must equal `active_flashloan_amount`; fee ≤ 0.5% (50 bps) | `constants.rs`, `state.rs` |
| Vaults `operate` remaining-accounts indices | exactly 3 bytes `[oracle_sources, branches, tick_has_debt]` | `docs/borrow/cpi.md` |
| Vaults decimals | ≤ 9 decimals per token | `VaultInvalidDecimals` |
| `init_position` | `next_position_id` must match live `vault_state` (race window between read and execution) | `VaultInvalidNextPositionId` |
| Manifest `invoke_passthrough` helper | ≤ 16 accounts per forwarded call (helper cap in `programs/manifest/src/program/processor/shared.rs`) — `SwapV2` max 14 ✔ | source |
| Manifest advanced orders | BatchUpdate containing Global/Reverse orders pays `ADVANCED_ORDER_FEE_LAMPORTS` to the market (collected by Defrag) | `instruction.rs` docs |
| Manifest pricing | `price_exponent ≤ order_type.max_exponent()`; lot sizes atomic (base atoms) | `PlaceOrderParams::try_price` |

## 3. Composition constraints discovered in this investigation

1. **Flashloan CPI is impossible** (`CPI_COMPATIBILITY.md` §3): both instructions enforce
   `get_stack_height() == 1` and the repay-matcher only sees top-level instructions. The
   engine's flashloan leg **must** be composed as top-level instructions in the same
   transaction as the engine's body instructions (sandwich layout):
   `[flashloan_borrow] [engine.execute …] [flashloan_payback] [engine.verify]`.
2. **One flashloan per transaction** ⇒ the two directional engines **cannot run in the same
   transaction**. Independence requirement from the brief is satisfiable only across
   separate transactions (which also matches "independent failure and rollback behavior").
3. **`FlashloanAdmin` global singleton**: `is_flashloan_active` is protocol-wide; concurrent
   flashloans from different users/engines contend on this account (writable lock) and
   `FlashloanAlreadyActive` if overlapping. Expect lock contention; serialize engine
   submissions or retry on `FlashloanAlreadyActive`.
4. **Signer topology**: the flashloan borrower must be a real signer (keypair). PDAs cannot
   sign top-level instructions. If the engine PDA owns the trading capital, funds must be
   arranged (ATA of the borrower; delegate or pre-staged balances) so the borrower can repay
   `amount + fee` from `signer_borrow_token_account`.
5. **Vaults depth**: engine → vaults `operate` → liquidity `operate` = stack depth 3 with
   2 nested CPI invocation costs; fine under the 5-depth limit even alongside token CPIs
   inside the liquidity program (depth 4).
6. **CPI cannot add signer privilege not present in the outer instruction**: every top-level
   engine instruction that will CPI to accounts needing `signer`/`writable` must request
   those privileges at the transaction level (privilege extension only downgrades).
7. **Optional-account convention** (Jupiter Lend): fill `None` slots with the callee program
   ID (read-only) and include its `AccountInfo` in the `invoke` list — otherwise Anchor
   account decoding fails.
8. **Return-data races**: `operate`/`deposit`/`withdraw` return data is global to the
   transaction; read it immediately after each CPI and check the producing program ID
   (the official reference code does this).
9. **Manifest matching is deterministic within an ix but cross-ix state (book) can change
   between the two engine swaps** — min-out parameters on `Swap`/`SwapV2` and the final
   `verify` instruction are the protection; never rely on pre-trade quotes alone.
10. **`init_position` race**: `next_position_id` is a global counter per vault; create
    positions ahead of the hot path (not inside the atomic sandwich) to avoid
    `VaultInvalidNextPositionId` failures mid-execution.

## 4. Recommended budgets (initial, to be measured in simulation)

| Path | Est. CU | Notes |
| --- | --- | --- |
| `flashloan_borrow` (top-level) | 50–150k | includes liquidity `operate` CPI + ATA init |
| Engine body: Manifest taker swap | 50–200k | depth-dependent (taker walk) |
| Engine body: vaults `operate` (deposit+borrow) | 150–400k | oracle sources + branch/tick updates dominate |
| `flashloan_payback` (top-level) | 80–200k | pre_operate + transfer + operate |
| `engine.verify` | 10–30k | balance-delta math |
| Whole sandwich tx | ≤ 1.4M per ix; total CU ≤ transaction cap (set `ComputeBudget`; several instructions share the 1.4M-per-ix cap individually) | measure on a mainnet-clone validator |

## 5. Failure / rollback semantics

- Solana transactions are all-or-nothing: if any of `[borrow, body…, payback, verify]`
  fails, the borrow reverts too — the flashloan's "repay-or-revert" invariant is preserved
  for top-level compositions and **only** for top-level compositions.
- `verify` must therefore run **after** `payback` and enforce min-profit/slippage; its
  failure reverts the entire sandwich (safe by construction).
- Never build a transaction where `payback` is omitted or made un-matchable (identical
  accounts/amount): the borrow would fail validation anyway — but any tooling that reorders
  or splits instructions must treat `[borrow, payback]` as an indivisible pair.
