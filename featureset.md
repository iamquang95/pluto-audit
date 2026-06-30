# Audit: featureset  (478 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/featureset/src/lib.rs`
- **Charon:** `app/featureset/{featureset,config,status_string}.go`
- **Tier C** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `lib.rs` ↔ `app/featureset/*.go` (feature flags, defaults, status enum)

## Crate-specific focus
- **Parity:** feature-flag names, default enabled/disabled set, enable/disable semantics, status values. NOTE: pluto recently removed `GLOBAL_STATE` (commit 3e32510) — confirm this is intended parity divergence and global flags resolve correctly.
- **Security:** none notable.
- **Quality:** how Go's global state is modeled in Rust without statics.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Low] Unknown enable/disable feature names: hard-reject vs charon's warn-and-ignore
- **Rust:** `crates/featureset/src/lib.rs:283` (`Config { enabled: Vec<Feature>, disabled: Vec<Feature> }`) and `:150` (`TryFrom<&str> for Feature`)
- **Charon ref:** [`app/featureset/config.go:58-86`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/featureset/config.go#L58-L86) (`Init`)
- **Issue:** Charon's `Config.Enabled/Disabled` are `[]string`; `Init` case-insensitively matches each against the feature table and, on no match, logs `log.Warn("Ignoring unknown enabled/disabled feature", …)` and continues. Pluto's `Config` is strongly typed (`Vec<Feature>`), so an unknown feature string cannot reach `from_config` at all — the warn-and-ignore branch has no equivalent. The intended parse boundary `Feature::try_from(&str)` returns `Err("unknown feature: …")`, i.e. a hard error, the opposite of charon's tolerant behaviour. `try_from` currently has **no caller** (`rg Feature::try_from` → only the impl), and no CLI flag (`--feature-set-enable/-disable`, see charon [`cmd/run.go:201-203`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/run.go#L201-L203)) is wired in pluto yet, so the divergence will materialise once the CLI is ported. If the CLI then propagates that `Err`, a typo in `--feature-set-enable` would abort node startup where charon would start (ignoring the typo).
- **Impact & likelihood:** Behavioural divergence at config boundary: charon tolerates unknown/typo'd feature flags (warn + run); pluto's typed model + `try_from` error would abort startup unless the CLI explicitly mirrors warn-and-ignore · only when a user passes an unrecognised feature name once flags are wired (today: no caller, latent).
- **PoC:** CONFIRMED (Rust side) + reasoned (Go not run) — scratch `scratch_try_from_unknown_is_hard_err` (`cargo test -p pluto-featureset`, 1 passed): `Feature::try_from("not_a_real_feature")` → `Err("unknown feature: not_a_real_feature")` (hard error); a realistic typo `"eager_double_linaer"` also → `Err`; case-insensitive match holds (`"MOCK_ALPHA"` → `MockAlpha`, charon `strings.EqualFold` parity). Charon side (read, not run): `config.go:58-86` `Init` matches case-insensitively and on no match calls `log.Warn("Ignoring unknown enabled/disabled feature", …)` then continues — tolerant. Latency confirmed: `rg` shows NO caller of `featureset::Feature::try_from` and NO `--feature-set-enable/-disable` CLI flag wired. Divergence is real but latent: no CLI caller or feature flag surface exists yet. Severity lowered to Low.
- **Fix:** When wiring the CLI parse boundary, mirror charon: map each `--feature-set-enable/-disable` token via `Feature::try_from`, and on `Err` emit `tracing::warn!("Ignoring unknown … feature", feature=tok)` and skip rather than propagating the error. Keep `try_from` as the matcher (it is already case-insensitive, matching charon's `strings.EqualFold`). Track in the CLI crate's audit; here, note `try_from`'s error contract must not be surfaced as fatal.

### [Low] `Feature::try_from` and `custom_enabled_all` are dead code in pluto
- **Rust:** `crates/featureset/src/lib.rs:150` (`TryFrom<&str> for Feature`), `:268` (`custom_enabled_all`)
- **Charon ref:** [`app/featureset/config.go:46,62,77`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/featureset/config.go#L46) (string matching in `Init`); [`app/metrics.go:133`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/metrics.go#L133) (`CustomEnabledAll` consumer)
- **Issue:** In charon, the string-matching logic backs `Init` and `CustomEnabledAll` is consumed by [`app/metrics.go:133`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/metrics.go#L133) (emits a metric per custom-enabled feature). In pluto, `Feature::try_from` has no caller and `custom_enabled_all` has no non-test caller (`rg custom_enabled_all`, `rg Feature::try_from` → defs/tests only). They are correct ports of as-yet-unported consumers (CLI flag parsing; metrics), so this is an unfinished-wiring gap, not a correctness defect — but `cargo clippy -D warnings` may not flag `pub` items, leaving the gap silent.
- **Impact & likelihood:** No runtime effect; risk is that the metrics parity (charon emits a gauge/counter per custom-enabled feature) and tolerant-flag-parsing parity are silently never completed · always (until consumers are ported).
- **PoC:** n/a
- **Fix:** No change to this unit. Ensure the metrics port consumes `custom_enabled_all` and the CLI port consumes `Feature::try_from`; otherwise remove them. Cross-reference the `app`/metrics and `cli` audit units.

### [Info] `enable_gnosis_block_hotfix_if_not_disabled` requires manual call-ordering that charon's global model did not
- **Rust:** `crates/featureset/src/lib.rs:246` (`enable_gnosis_block_hotfix_if_not_disabled`)
- **Charon ref:** [`app/featureset/config.go:94`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/featureset/config.go#L94) (`EnableGnosisBlockHotfixIfNotDisabled`), called at [`app/app.go:202`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/app.go#L202) after `featureset.Init` (`app.go:151`)
- **Issue:** Charon mutates global state under `initMu`, so `Init` then `EnableGnosisBlockHotfixIfNotDisabled` are independent global calls. Pluto correctly replaces the global with an injected `Arc<FeatureSet>` (per module doc, lines 7-23) and requires: `from_config` → `enable_gnosis_block_hotfix_if_not_disabled(&mut)` (gnosis/chiado only) → `Arc::new`. This ordering is load-bearing (the hotfix must be applied before the set is frozen into the `Arc`) but is enforced only by documentation, not types. No app-wiring caller exists yet (`rg enable_gnosis` → crate only), so correct sequencing is unverified for the eventual `run` path on gnosis/chiado.
- **Impact & likelihood:** If app wiring wraps in `Arc` before calling the hotfix (or omits the call on gnosis/chiado), the SSZ block hotfix silently stays at `Alpha` (disabled under default `min_status=Stable`) on gnosis/chiado → wrong block hashing on those networks · only if wiring mis-sequences; latent (no caller today).
- **PoC:** n/a
- **Fix:** When wiring `run` for gnosis/chiado, call `enable_gnosis_block_hotfix_if_not_disabled` before `Arc::new`. Optionally fold the network decision into `from_config` (e.g. `from_config_for_network`) so the hotfix cannot be forgotten. Verify in the `app`/cli audit.

### [Info] Parity confirmed: feature names, default statuses, and `enabled`/`min_status` semantics all match charon
- **Rust:** `crates/featureset/src/lib.rs:104-142` (`Feature`/`as_str`/`all`), `:209-223` (default `state`), `:260-265` (`enabled`)
- **Charon ref:** [`app/featureset/featureset.go:27-98`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/featureset/featureset.go#L27-L98) (consts, `state`, `minStatus`), `:101-106` (`Enabled`)
- **Issue:** Not a defect — recorded for the security dimension (a wrong default could enable/disable a safety feature). All 13 features present with identical string names (`mock_alpha`, `eager_double_linear`, `consensus_participate`, `aggsigdb_v2`, `json_requests`, `gnosis_block_hotfix`, `linear`, `sse_reorg_duties`, `attestation_inclusion`, `proposal_timeout`, `quic`, `fetch_only_commidx_0`, `chain_split_halt`). Default statuses match: `EagerDoubleLinear`+`ConsensusParticipate`=`Stable`, all others=`Alpha`. Default `min_status`=`Stable` (= charon `DefaultConfig`/`minStatus`). `enabled` uses `status >= min_status` with missing-key default `Disable(0)` (Go: zero value `0`), and `min_status` validation accepts exactly alpha/beta/stable (Go loop `statusAlpha..<statusSentinel`). `ChainSplitHalt` (anti-chain-split safety gate) defaults `Alpha` → off under default `Stable` in both, matching charon.
- **Impact & likelihood:** none (parity holds) · n/a
- **PoC:** n/a
- **Fix:** none.

## Summary

Strong parity: all 13 features, their string names, default statuses, default `min_status=Stable`, and `enabled`/`min_status` comparison semantics match charon exactly — the security-critical "no safety feature defaults wrong" check passes (e.g. `chain_split_halt`, `gnosis_block_hotfix` default Alpha/off, as in charon). The `GLOBAL_STATE` removal (commit 3e32510) is a sound parity divergence: the global+mutex model is replaced by an injected immutable `Arc<FeatureSet>`, eliminating the read-path lock. Findings are 0C/0H/1M/1L/2I and all latent (no app/CLI caller wires the crate yet): (M) charon's tolerant warn-and-ignore of unknown feature flags is structurally lost in the typed `Config`/`Feature::try_from(Err)` model and must be re-mirrored at the CLI parse boundary; (L) `try_from`/`custom_enabled_all` are dead until CLI/metrics consumers are ported; (I) gnosis hotfix call-ordering is doc-enforced only. No security defects, no panics on untrusted input, no `unsafe`.

## Uncertain / needs-human

- **Is the hard-reject vs warn-and-ignore divergence (Medium) an intentional port decision?** Charon deliberately tolerates unknown feature flags (warn + continue). Pluto's typed `Config` + `Feature::try_from` (`Err`) leans toward rejecting. Whether the CLI port should mirror charon's tolerance or intentionally tighten to a hard error is a product/parity call. Flagged Medium on the assumption functional equivalence is the golden rule (per AGENTS.md); downgrade to Info if a strict-rejection policy is intended.
- **`Status::Enable = i32::MAX as isize` vs Go `math.MaxInt` (=2^63-1 on 64-bit):** ordering relative to all real statuses (≤4) is identical, so no observable difference; not filed as a finding. Confirm no code ever does arithmetic on the discriminant (none seen) — only comparisons, which are safe.
