# Program Addresses — Verified per Network

Date: 2026-11-15

**Policy:** every address below is copied verbatim from an official artifact (IDL
`address` field, `declare_id!`, Anchor.toml, or official CPI guide). No address was
invented. Verification is *source-triangulation*; this environment has no RPC access, so
on-chain account state was not queried (see `CPI_COMPATIBILITY.md` §1 and §4.1).

Evidence keys:
- **[IDL]** — `address` field of an official IDL in `jup-ag/jupiter-lend` `target/idl/`
  (commit `7960a14a…`, equals current `main`).
- **[GUIDE]** — official CPI guide table (`docs/earn/cpi.md`, `docs/borrow/cpi.md`).
- **[SRC]** — `declare_id!` in audited protocol source (`code-423n4/2026-02-jupiter-lend`
  scope snapshot; flashloan/lending/liquidity/vaults/oracle/lendingRewardRateModel).
- **[ANCHOR]** — `[programs.mainnet]` / `[programs.devnet]` of the audit workspace
  `Anchor.toml`.
- **[MANIFEST]** — `declare_id!` in `Bonasa-Tech/manifest` `programs/*/src/lib.rs` and
  `client/*/constants` (commit `d4e908b3…`, verified unchanged at `main` `52a6f56b…`).

---

## 1. Jupiter Lend — mainnet-beta (`main` market)

| Program | Address | Version (IDL) | Corroboration |
| --- | --- | --- | --- |
| `flashloan` | `jupgfSgfuAXv4B6R2Uxu85Z1qdzgju79s6MfZekN6XS` | 0.1.4 | [IDL] `flashloan.json` · [SRC] `declare_id!(not(staging))` · [ANCHOR] `[programs.mainnet]` |
| `lending` (Earn) | `jup3YeL8QhtSx1e253b2FDvsMNC87fDrgQZivbrndc9` | 0.1.9 | [IDL] `lending.json` · [GUIDE] earn · [SRC] · [ANCHOR] |
| `liquidity` (shared liquidity layer) | `jupeiUmn818Jg1ekPURTpr4mFo29p46vygyykFJ3wZC` | 0.1.9 | [IDL] `liquidity.json` · [GUIDE] earn/borrow · [SRC] · [ANCHOR] |
| `vaults` (collateralized Borrow) | `jupr81YtYssSyPt8jbnGuiWon5f6x9TcDEFxYe3Bdzi` | 0.1.8 | [IDL] `vaults.json` · [GUIDE] borrow · [SRC] · [ANCHOR] |
| `oracle` | `jupnw4B6Eqs7ft6rxpzYLJZYSnrpRgPcr589n5Kv4oc` | 0.1.8 | [IDL] `oracle.json` · [SRC] · [ANCHOR] |
| `lending_reward_rate_model` | `jup7TthsMgcR9Y3L277b8Eo9uboVSmu1utkuXHNUKar` | 0.1.9 | [IDL] · [SRC] · [ANCHOR] |
| `dex` (smart-debt/DEX vault layer) | `jupZ4m2GqUCJ5iueMfzQf8khFfH31d4XAQt3RzCT9Vd` | 0.1.8 | [IDL] `dex.json` — **out of scope** for this engine (see `CPI_COMPATIBILITY.md` §2.3) |

Notes:

- These are the IDs of the Jupiter Lend **`main` market**. The SDK's `market` parameter
  also selects an **`ethena` market** — a *separate deployment with its own program IDs*
  (not published in this repo). If ethena support is ever needed, source those IDs from
  Jupiter's official deployment config; do not reuse the table above. (Status for ethena:
  **UNVERIFIED**.)
- Flashloan governance authority (per audited source `constants.rs`): multisig
  `HqPrpa4ESBDnRHRWaiYtjv4xe93wvCS9NNZtDwR89cVa` (`GOVERNANCE_MS`), stored in
  `FlashloanAdmin.authority`.
- Official README warning: *"The devnet programs listed in the CPI guides were last
  deployed in May 2025 and do not match these IDLs. The `@jup-ag/lend` SDK does not target
  devnet."*

## 2. Jupiter Lend — devnet

| Program | Address | Source | Status |
| --- | --- | --- | --- |
| `flashloan` | `aaae6RAZNe4wLgKkjKnJxqQ15B58yPUdcygDbTUpXo3` | [SRC] `declare_id!(feature="staging")` · [ANCHOR] `[programs.devnet]` | **UNVERIFIED** — not listed in the official CPI guides; staging build id; deployment state unknown |
| `lending` | `7tjE28izRUjzmxC1QNXnNwcc4N82CNYCexf3k8mw67s3` | [GUIDE] earn · [SRC] · [ANCHOR] | **STALE** (May 2025; does not match current IDL) |
| `liquidity` | `5uDkCoM96pwGYhAUucvCzLfm5UcjVRuxz6gH81RnRBmL` | [GUIDE] earn · [SRC] · [ANCHOR] | **STALE** (May 2025) |
| `vaults` | `Ho32sUQ4NzuAQgkPkHuNDG3G18rgHmYtXFA8EBmqQrAu` | [GUIDE] borrow · [SRC] · [ANCHOR] | **STALE** (May 2025) |
| `oracle` | `C59ahnV7xHuW1Q9E87yrYbCHMHaYnkqRdix8BfLp3vnk` | [SRC] · [ANCHOR] | **STALE** (May 2025) |
| `lending_reward_rate_model` | `68LHLkpgjAvo6Lgd9FT6KYEX4FWn1911EohSXxHYMFjc` | [SRC] · [ANCHOR] | **STALE** (May 2025) |

**Devnet verdict for Jupiter Lend:** official guidance is to test against **mainnet or a
local validator with the mainnet programs cloned** (`solana-test-validator --clone` /
`--bpf-program`), not devnet. The devnet IDs above are recorded for completeness and must
not be assumed interface-compatible with the current IDLs. Never reuse devnet addresses on
mainnet.

## 3. Manifest (Bonasa-Tech) — mainnet-beta / localnet

| Program | Address | Source | Status |
| --- | --- | --- | --- |
| `manifest` (core orderbook) | `MNFSTqtC93rEfYHB6hF82sKdZpUDFWkViLByLd1k1Ms` | [MANIFEST] `programs/manifest/src/lib.rs` `declare_id!`; `client/rust/slim` `MANIFEST_PROGRAM_ID`; `client/ts` `PROGRAM_ADDRESS`; README "Upgrade Authority" section | **SUPPORTED** |
| `wrapper` (reference wrapper) | `wMNFSTkir3HgyZTsB7uqu3i7FA73grFCptPXgrZjksL` | [MANIFEST] `programs/wrapper/src/lib.rs` `declare_id!`; `client/ts/src/wrapper/index.ts` `PROGRAM_ADDRESS` | **SUPPORTED** |
| `ui-wrapper` | `UMnFStVeG1ecZFc2gc5K3vFy3sMpotq8C91mXBQDGwh` | [MANIFEST] `programs/ui-wrapper/src/lib.rs` `declare_id!`; `local-validator-test.sh` | **SUPPORTED** (reference only) |

Notes:

- The core program's upgrade authority is a **Squads multisig** (4-of-5):
  multisig account `6o29zFofTxn8nM5o83JTY12G3cGgFy7mrtM2Cp3GQJXg`, vault
  `CDFU8tEWsVU2ZMiek57Sgk3Huha2yBNcSHLAts3V3Cbf` *(copied verbatim from the official
  README; re-verify on-chain before relying on it)*.
- The same three IDs are used on **localnet** by the official test harness
  (`local-validator-test.sh` deploys `manifest.so` / `wrapper.so` / `ui_wrapper.so` under
  these IDs with `solana-test-validator --bpf-program`). This is the recommended way to
  integration-test Manifest CPI without a devnet deployment.

## 4. Manifest — devnet

| Program | Address | Status |
| --- | --- | --- |
| `manifest` | none found | **UNVERIFIED / not published** |
| `wrapper` | none found | **UNVERIFIED / not published** |
| `ui-wrapper` | none found | **UNVERIFIED / not published** |

A repo-wide search of `Bonasa-Tech/manifest` (programs, clients, scripts, docs, CI)
found **no devnet program IDs**; the TS cluster helper only distinguishes
mainnet/devnet/localnet genesis hashes and the clients hardcode the mainnet IDs. Conclusion:
there is **no official Manifest devnet deployment** in the current repository. Per the
no-invented-addresses rule, no devnet addresses are provided. For devnet-flavored testing of
the engine, deploy Manifest locally (`solana-test-validator --bpf-program`) or run against
mainnet with read-only/simulated flows.

## 5. Well-known programs and sysvars used by the integration

| Program / sysvar | Address | Used by |
| --- | --- | --- |
| System Program | `11111111111111111111111111111111` | all |
| SPL Token | `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` | all token movements |
| Token-2022 | `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` | Manifest (optional), Lend (`token_interface`) |
| Associated Token Program | `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL` | flashloan (`init_if_needed` ATA), earn, vaults |
| Sysvar Instructions | `Sysvar1nstructions1111111111111111111111111` | **flashloan (repayment verification — top-level only!)**, vaults `init_position` |
| Sysvar Rent | `SysvarRent1111111111111111111111111` | vaults `init_position` |
| Metaplex Token Metadata | `metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s` | vaults `init_position` (position NFT metadata) |

## 6. Network configuration policy (from the project brief, retained)

- Separate config files per cluster (`config/devnet.json`, `config/mainnet.json`) — every
  program ID, mint, market, vault and oracle address must be cluster-scoped.
- Never reuse a devnet address on mainnet; never assume a devnet deployment matches the
  mainnet IDL (Jupiter's own docs prove it does not).
- Require explicit confirmation before any mainnet deployment (none was performed, and none
  is proposed in this phase).
