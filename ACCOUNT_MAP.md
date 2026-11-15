# Account Map — Required Accounts, PDAs, Signers, Writability, Remaining Accounts

Date: 2026-11-15

All lists are transcribed from official IDLs / shank annotations / audited source
(`CPI_COMPATIBILITY.md` §1). W = writable, S = signer. For CPI calls, writable/signer
privileges cannot exceed those of the incoming transaction (Solana privilege extension);
requesting write access on a read-only outer account fails the CPI.

Conventions:
- **Optional account slots keep their position.** Anchor treats a slot filled with the
  *callee program ID* (read-only) as `None` (Jupiter Lend convention, documented in the
  official CPI guides).
- Jupiter Lend amounts are in the token's **native decimals** (no pre-scaling). Tokens with
  more than 9 decimals fail `VaultInvalidDecimals`.
- Manifest amounts are in **atoms** (base/quote native units) with explicit
  mantissa/exponent pricing.

---

## 1. Jupiter Lend — Flashloan (`jupgfSgfuAXv4B6R2Uxu85Z1qdzgju79s6MfZekN6XS`)

### 1.1 `flashloan_borrow(amount: u64)` and `flashloan_payback(amount: u64)`

**Identical account lists — order and keys must match exactly** (payback-time validation
compares the full 14-account list against the borrow's; any deviation ⇒ the borrow is
rejected at validation with `FlashloanInvalidInstruction`/`FlashloanPaybackNotFound`).

| # | Account | W | S | Notes / derivation |
| --- | --- | --- | --- | --- |
| 0 | `signer` | W | S | Borrower. Must be a **transaction signer** (top-level; see `CPI_COMPATIBILITY.md` §3). Payback transfers `amount + fee` from its token account. |
| 1 | `flashloan_admin` | W | | PDA `["flashloan_admin"]` of the flashloan program. Singleton state: `authority, liquidity_program, status, flashloan_fee (bps), flashloan_timestamp, is_flashloan_active, active_flashloan_amount, bump`. |
| 2 | `signer_borrow_token_account` | W | | ATA of `signer` for `mint`. `init_if_needed` (created by the instruction itself, payer = signer). Borrowed tokens land here; payback pulls from here. |
| 3 | `mint` | | | Borrowed token mint. |
| 4 | `flashloan_token_reserves_liquidity` | W | | Liquidity layer `TokenReserve` PDA — `["reserve", mint]` (liquidity program). |
| 5 | `flashloan_borrow_position_on_liquidity` | W | | Liquidity layer borrow position PDA — `["user_borrow_position", mint, protocol]` where `protocol` = `flashloan_admin`. |
| 6 | `rate_model` | | | `["rate_model", mint]` (liquidity program). |
| 7 | `vault` | W | | Liquidity layer token vault for `mint` (the program transfers payback funds here). |
| 8 | `liquidity` | | | Liquidity singleton PDA `["liquidity"]`. |
| 9 | `liquidity_program` | | | Must satisfy `flashloan_admin.liquidity_program` (`has_one`). Mainnet `jupeiUmn818Jg1ekPURTpr4mFo29p46vygyykFJ3wZC`. |
| 10 | `token_program` | | | SPL Token or Token-2022 (`Interface<TokenInterface>`). |
| 11 | `associated_token_program` | | | Optional slot (`ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL`). |
| 12 | `system_program` | | | `11111111111111111111111111111111`. |
| 13 | `instruction_sysvar` | | | `Sysvar1nstructions1111111111111111111111111`. **Load-bearing**: repayment verification reads it. |

Args: `amount: u64` (≥ `MIN_FLASHLOAN_AMOUNT` = 1000 atoms). Payback `amount` must equal
the borrow amount; the token transfer at payback is `amount + ceil(amount·fee_bps/10_000)`.
Data length of `flashloan_payback` must be exactly 16 bytes (8-byte disc
`[213,47,153,137,84,243,94,232]` + u64 LE) or the borrow-time matcher ignores/rejects it.

Internal flow (for understanding, not for CPI): borrow/payback CPI into the liquidity
program's `operate`/`pre_operate` as the `flashloan_admin` PDA (`invoke_signed` with
`["flashloan_admin", bump]`).

---

## 2. Jupiter Lend — Earn (`jup3YeL8QhtSx1e253b2FDvsMNC87fDrgQZivbrndc9`)

### 2.1 `deposit(assets: u64)` — 17 accounts

| # | Account | W | S | Notes |
| --- | --- | --- | --- | --- |
| 0 | `signer` | W | S | Depositor (tx signer or engine PDA via `invoke_signed`). |
| 1 | `depositor_token_account` | W | | Underlying token account owned by `signer`. |
| 2 | `recipient_token_account` | W | | fToken account owned by `signer` (receives shares). |
| 3 | `mint` | | | Underlying mint. |
| 4 | `lending_admin` | | | PDA `["lending_admin"]`. |
| 5 | `lending` | W | | PDA `["lending", mint, f_token_mint]`. |
| 6 | `f_token_mint` | W | | PDA `["f_token_mint", mint]`. |
| 7 | `supply_token_reserves_liquidity` | W | | `["reserve", mint]` (liquidity). |
| 8 | `lending_supply_position_on_liquidity` | W | | `["user_supply_position", mint, protocol]`, protocol = `lending` program's protocol account (verified in-liquidity). |
| 9 | `rate_model` | | | `["rate_model", mint]`. |
| 10 | `vault` | W | | Liquidity vault for `mint`. |
| 11 | `liquidity` | W | | `["liquidity"]`. |
| 12 | `liquidity_program` | | | Must equal `lending_admin.liquidity_program`. |
| 13 | `rewards_rate_model` | | | `jup7TthsMgcR9Y3L277b8Eo9uboVSmu1utkuXHNUKar` (per pool). |
| 14 | `token_program` | | | Owner of `mint`. |
| 15 | `associated_token_program` | | | Optional slot. |
| 16 | `system_program` | | | |

Args: `assets: u64` (native decimals; `u64::MAX` = full balance). Returns shares minted
(Borsh `u64` via return data). Slippage variant `deposit_with_min_amount_out(assets,
min_amount_out)` — same accounts, disc `[116,144,16,97,118,109,40,119]`.
Token accounts are **not** created by the program — pre-create them.

### 2.2 `withdraw(amount: u64)` — 18 accounts

Note the ordering differs from `deposit`: `mint` sits at slot 5 (after `lending`), and an
optional `claim_account` slot appears after `vault`.

| # | Account | W | S | Notes |
| --- | --- | --- | --- | --- |
| 0 | `signer` | W | S | Must own the fTokens being burned. |
| 1 | `owner_token_account` | W | | fToken source (burned). |
| 2 | `recipient_token_account` | W | | Underlying destination. |
| 3 | `lending_admin` | | | `["lending_admin"]`. |
| 4 | `lending` | W | | `["lending", mint, f_token_mint]`. |
| 5 | `mint` | | | Underlying mint. |
| 6 | `f_token_mint` | W | | `["f_token_mint", mint]`. |
| 7 | `supply_token_reserves_liquidity` | W | | `["reserve", mint]`. |
| 8 | `lending_supply_position_on_liquidity` | W | | `["user_supply_position", mint, protocol]`. |
| 9 | `rate_model` | | | `["rate_model", mint]`. |
| 10 | `vault` | W | | Liquidity vault for `mint`. |
| 11 | `claim_account` | W | | **Optional**; unused by `withdraw`; pass the lending program ID read-only for `None` (the `@jup-ag/lend` SDK passes the claim PDA; both accepted). |
| 12 | `liquidity` | W | | `["liquidity"]`. |
| 13 | `liquidity_program` | | | Must equal `lending_admin.liquidity_program`. |
| 14 | `rewards_rate_model` | | | |
| 15 | `token_program` | | | |
| 16 | `associated_token_program` | | | Optional. |
| 17 | `system_program` | | | |

Args `amount: u64` (`u64::MAX` = full shares, rounded down; program does **not** enforce
liquidity-layer withdrawal limits — check max-withdrawable first). Returns shares burned
(Borsh `u64` return data). Slippage variant
`withdraw_with_max_shares_burn(amount, max_shares_burn)` — same accounts, disc
`[47,197,183,171,239,18,245,171]`.

---

## 3. Jupiter Lend — Borrow / Vaults (`jupr81YtYssSyPt8jbnGuiWon5f6x9TcDEFxYe3Bdzi`)

### 3.1 `init_position(vault_id: u16, next_position_id: u32)` — 13 accounts

| # | Account | W | S | Derivation / notes |
| --- | --- | --- | --- | --- |
| 0 | `signer` | W | S | Pays rent, receives the position NFT (or PDA via `invoke_signed`; PDA must hold lamports). |
| 1 | `vault_admin` | | | `["vault_admin"]`. |
| 2 | `vault_state` | W | | `["vault_state", vault_id (u16 LE)]`. |
| 3 | `position` | W | | `["position", vault_id (u16 LE), next_position_id (u32 LE)]`. |
| 4 | `position_mint` | W | | `["position_mint", vault_id, next_position_id]`. |
| 5 | `position_token_account` | W | | ATA of `signer` for `position_mint`. |
| 6 | `metadata_account` | W | | Metaplex PDA `["metadata", metaqbxx…, position_mint]`. |
| 7 | `token_program` | | | SPL Token. |
| 8 | `associated_token_program` | | | |
| 9 | `system_program` | | | |
| 10 | `sysvar_instruction` | | | `Sysvar1nstructions…` (used by the program). |
| 11 | `metadata_program` | | | `metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s`. |
| 12 | `rent` | | | `SysvarRent1111…`. |

`next_position_id` must equal `vault_state.next_position_id` at execution time
(`VaultInvalidNextPositionId` on stale values) — read it immediately before building.

### 3.2 `operate(new_col: i128, new_debt: i128, transfer_type: Option<TransferType>, remaining_accounts_indices: bytes)` — 35 fixed slots + remaining accounts

`new_col`/`new_debt`: positive = deposit/borrow, negative = withdraw/payback,
`i128::MIN` = max withdraw/payback. Returns `(nft_id: u32, new_col_final: i128,
new_debt_final: i128)` via return data. Standard (T1) vaults only.

| # | Account | W | S | Derivation / notes |
| --- | --- | --- | --- | --- |
| 0 | `signer` | W | S | Must hold (or be delegate of) the position NFT for withdraw/borrow. Anyone may deposit/payback for any position. |
| 1 | `signer_supply_token_account` | W | | Optional. Required for deposit. Owned by `signer`. |
| 2 | `signer_borrow_token_account` | W | | Optional. Required for payback. Owned by `signer`. |
| 3 | `recipient` | | | Optional; defaults to `signer`. |
| 4 | `recipient_borrow_token_account` | W | | Optional; borrow payout target. |
| 5 | `recipient_supply_token_account` | W | | Optional; withdraw payout target. |
| 6 | `vault_config` | | | `["vault_config", vault_id]`. |
| 7 | `vault_state` | W | | `["vault_state", vault_id]`. |
| 8 | `supply_token` | | | Supply mint. |
| 9 | `borrow_token` | | | Borrow mint. |
| 10 | `oracle` | | | Must equal `vault_config.oracle`. |
| 11 | `position` | W | | `["position", vault_id, nft_id (u32 LE)]`. |
| 12 | `position_token_account` | | | NFT token account. |
| 13 | `current_position_tick` | W | | `["tick", vault_id, (tick − MIN_TICK) (i16/u16 LE)]` (MIN_TICK = −16383). |
| 14 | `final_position_tick` | W | | Same derivation, post-op tick. |
| 15 | `current_position_tick_id` | | | `["tick_id_liquidation", vault_id, (tick − MIN_TICK), ((total_ids + 2) / 3)]`. |
| 16 | `final_position_tick_id` | W | | Same pattern for final tick. |
| 17 | `new_branch` | W | | `["branch", vault_id, branch_id (u32 LE)]`. |
| 18 | `supply_token_reserves_liquidity` | W | | `["reserve", supply_mint]`. |
| 19 | `borrow_token_reserves_liquidity` | W | | `["reserve", borrow_mint]`. |
| 20 | `vault_supply_position_on_liquidity` | W | | `["user_supply_position", supply_mint, protocol]` (protocol = vaults' protocol account). |
| 21 | `vault_borrow_position_on_liquidity` | W | | `["user_borrow_position", borrow_mint, protocol]`. |
| 22 | `supply_rate_model` | | | `["rate_model", supply_mint]`. |
| 23 | `borrow_rate_model` | | | `["rate_model", borrow_mint]`. |
| 24 | `vault_supply_token_account` | W | | Liquidity program's supply-token account. |
| 25 | `vault_borrow_token_account` | W | | Liquidity program's borrow-token account. |
| 26 | `supply_token_claim_account` | W | | Optional. `["user_claim", user, supply_mint]`. Required for withdraw with `TransferType::Claim`. |
| 27 | `borrow_token_claim_account` | W | | Optional. Required for borrow with `TransferType::Claim`. |
| 28 | `liquidity` | | | `["liquidity"]`. |
| 29 | `liquidity_program` | | | Must equal `vault_config.liquidity_program`. |
| 30 | `oracle_program` | | | `jupnw4B6Eqs7ft6rxpzYLJZYSnrpRgPcr589n5Kv4oc` (mainnet). |
| 31 | `supply_token_program` | | | |
| 32 | `borrow_token_program` | | | |
| 33 | `associated_token_program` | | | Optional. |
| 34 | `system_program` | | | |

**Remaining accounts** (after slot 34), ordered and counted by `remaining_accounts_indices`
(Borsh `bytes`: u32 LE length + 3 bytes `[oracle_sources_count, branch_count,
tick_has_debt_count]`):

1. `oracle_sources_count` oracle-source accounts — read-only;
2. `branch_count` branch accounts — writable;
3. `tick_has_debt_count` tick-has-debt array accounts — writable
   (`["tick_has_debt", vault_id, index (u32 LE)]`).

Read/write-rounding notes (protocol-favoring): deposit rounds up +1 native unit (unless
depositing the exact source balance), payback rounds up +1 (fund with amount + 1), withdraw
and borrow round down. Max-payback/withdraw return actual amounts in the return data.

---

## 4. Manifest — core (`MNFSTqtC93rEfYHB6hF82sKdZpUDFWkViLByLd1k1Ms`)

Instruction data is a **1-byte tag** (0..14) + Borsh payload. Market vault PDAs:
`["vault", market, mint]` (base & quote). `market` and `wrapper_state` are ordinary fresh
accounts initialized by the programs (not PDAs). Accounts below from shank annotations
(`programs/manifest/src/program/instruction.rs`).

| Ix (tag) | Accounts (#, W, S) | Payload |
| --- | --- | --- |
| `CreateMarket` (0) | 0 payer W,S · 1 market W · 2 system_program · 3 base_mint · 4 quote_mint · 5 base_vault W · 6 quote_vault W · 7 token_program · 8 token_program_22 | — |
| `ClaimSeat` (1) | 0 payer W,S · 1 market W · 2 system_program | — |
| `Deposit` (2) | 0 payer W,S · 1 market W · 2 trader_token W · 3 vault W · 4 token_program · 5 mint | amount |
| `Withdraw` (3) | same as Deposit | amount |
| `Swap` (4) | 0 payer W,S · 1 market W · 2 system_program · 3 trader_base W · 4 trader_quote W · 5 base_vault W · 6 quote_vault W · 7 token_program_base · 8 base_mint *(optional, Token-2022)* · 9 token_program_quote *(optional)* · 10 quote_mint *(optional)* · 11 global W *(optional)* · 12 global_vault W *(optional)* | `SwapParams { in_atoms, out_atoms, is_base_in, is_exact_in }` — exact-in enforces min-out; exact-out enforces max-in |
| `Expand` (5) | 0 payer W,S · 1 market W · 2 system_program | — |
| `BatchUpdate` (6) | 0 payer W,S · 1 market W · 2 system_program · 3 base_mint *(opt)* · 4 base_global W *(opt)* · 5 base_global_vault W *(opt)* · 6 base_market_vault W *(opt)* · 7 base_token_program *(opt)* · 8 quote_mint *(opt)* · 9 quote_global W *(opt)* · 10 quote_global_vault W *(opt)* · 11 quote_market_vault W *(opt)* · 12 quote_token_program *(opt)* | `BatchUpdateParams { trader_index_hint: Option<DataIndex>, cancels: Vec<CancelOrderParams>, orders: Vec<PlaceOrderParams> }`; `PlaceOrderParams { base_atoms, price_mantissa, price_exponent, is_bid, last_valid_slot, order_type }`; `CancelOrderParams { order_sequence_number, order_index_hint }`. Global/reverse orders pay `ADVANCED_ORDER_FEE_LAMPORTS`. |
| `GlobalCreate` (7) | 0 payer W,S · 1 global W · 2 system_program · 3 mint · 4 global_vault W · 5 token_program | — |
| `GlobalAddTrader` (8) | 0 payer W,S · 1 global W · 2 system_program | — |
| `GlobalDeposit` (9) | 0 payer W,S · 1 global W · 2 mint · 3 global_vault W · 4 trader_token W · 5 token_program | amount |
| `GlobalWithdraw` (10) | 0 payer W,S · 1 global W · 2 mint · 3 global_vault W · 4 trader_token W · 5 token_program | amount |
| `GlobalEvict` (11) | 0 payer W,S · 1 global W · 2 mint · 3 global_vault W · 4 trader_token W · 5 evictee_token W · 6 token_program | — |
| `GlobalClean` (12) | 0 payer W,S (fee collector) · 1 market W · 2 system_program · 3 global W | — |
| `SwapV2` (13) | 0 payer W,S · 1 owner W,S · 2 market W · 3 system_program · 4 trader_base W · 5 trader_quote W · 6 base_vault W · 7 quote_vault W · 8 token_program_base · 9 base_mint *(opt)* · 10 token_program_quote *(opt)* · 11 quote_mint *(opt)* · 12 global W *(opt)* · 13 global_vault W *(opt)* | `SwapParams` — `owner` may be a PDA signing via `invoke_signed` (router hop design) |
| `Defrag` (14) | 0 collector W,S · 1 market W · 2 base_vault W · 3 quote_vault W · 4 base_token_program · 5 quote_token_program · 6 system_program | — |

Matching/settlement: taker fills execute inside `Swap`/`SwapV2`; maker fills execute inside
`BatchUpdate` place-order; proceeds settle immediately (taker-favoring rounding). No crank.

## 5. Manifest — wrapper (`wMNFSTkir3HgyZTsB7uqu3i7FA73grFCptPXgrZjksL`) and ui-wrapper (`UMnFStVeG1ecZFc2gc5K3vFy3sMpotq8C91mXBQDGwh`)

Wrapper state = fresh account initialized by `CreateWrapper` (holds `trader` pubkey +
market-info tree). All trading ixs take `manifest_program` as a pass-through account and
CPI the core (`invoke_passthrough`, ≤ 16 accounts per call — helper-level cap).

- wrapper `CreateWrapper` 0: owner W,S · system_program · wrapper_state W.
- wrapper `ClaimSeat` 1: manifest_program · owner W,S · market W · system_program ·
  wrapper_state W.
- wrapper `Deposit` 2 / `Withdraw` 3: manifest_program · owner W,S · market W ·
  trader_token_account W · vault W · token_program · wrapper_state W · mint ·
  **system_program (added upstream `d4e908b..52a6f56`; read optionally by processors)**.
- wrapper `BatchUpdate` 4 (+ 5 base-global / 6 quote-global variants): wrapper_state W ·
  manifest_program · owner W,S · market W · system_program · then the core BatchUpdate
  global-account block (optional slots identical to core ix 6).
- wrapper `Collect` 7: wrapper_state W · system_program · collector W,S.
- wrapper `Defrag` 8: wrapper_state W · authority S.

UI wrapper (`PlaceOrder` 2 / `EditOrder` 3 / `CancelOrder` 4 / `SettleFunds` 5 / `Collect` 6)
adds `payer` and fee accounts (`fee_authority` S, `platform_token_account` W,
`referrer_token_account` W optional) and always passes explicit base/quote global blocks;
`EditOrder` is documented as not-implemented upstream. Reference only — the engine should
CPI the **core** directly (fewer accounts, no UI fees).

## 6. Signer / PDA summary for the engine

| Role | Who signs | Mechanism |
| --- | --- | --- |
| Flashloan `signer` | Real keypair (tx signer) | Top-level ix only (PDAs cannot sign top-level ixs) |
| Earn `signer`, Vaults `operate`/`init_position` `signer` | User keypair **or** engine PDA | `invoke` / `invoke_signed` (`["<seed>", bump]`) |
| Manifest `payer`/`owner` | User keypair or engine PDA | `invoke` / `invoke_signed`; `SwapV2` explicitly supports PDA hops |
| Vaults position NFT | Held by (or delegated to) the operate signer | ATA of signer/PDA |
| Liquidity-layer protocol accounts (`user_supply_position`, `user_borrow_position`, `user_claim`) | PDAs of the **liquidity** program keyed by *protocol* (flashloan_admin / lending / vaults), never signed by the engine | passed as accounts |
