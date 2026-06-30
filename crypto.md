# Audit: crypto  (1455 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/crypto/src/*` (blst_impl, tbls, tblsconv, types)
- **Charon:** `tbls/{tbls,herumi}.go`, `tbls/tblsconv/tblsconv.go` (charon uses herumi/BLS)
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `tbls.rs`/`blst_impl.rs` ↔ `tbls/{tbls,herumi}.go` (threshold BLS)
- `tblsconv.rs` ↔ `tbls/tblsconv/tblsconv.go` (serialization conversions)

## Crate-specific focus
- **Parity (interop-critical):** charon uses herumi/BLS, pluto uses blst — scheme params MUST match (ciphersuite/domain, hash-to-curve, pubkey/sig point serialization) so signatures interoperate cross-impl. Lagrange interpolation for share recovery; `tblsconv` byte formats.
- **Security (CRITICAL):** subgroup/point-on-curve checks on every deserialized pubkey/sig, reject identity/zero, secret-share handling + zeroize, secure RNG.
- **Quality:** error types vs panics; constant-time.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [High] verify_aggregate: per-pubkey subgroup check skipped — only the aggregate is validated
- **Rust:** `crates/crypto/src/blst_impl.rs:206` (`verify_aggregate`) + `240` (`aggregate_public_keys`)
- **Charon ref:** [`tbls/herumi.go:321`](https://github.com/ObolNetwork/charon/blob/v1.7.1/tbls/herumi.go#L321) (`VerifyAggregate`) → Herumi `sig.FastAggregateVerify`
- **Issue:** Each share pubkey is parsed with `BlstPublicKey::from_bytes` (no group check), summed with raw `blst_p1_add_or_double_affine`, then only the **aggregate** is verified with `pk_validate=true`. Charon's `FastAggregateVerify` follows the BLS spec which `KeyValidate`s every individual pubkey (subgroup membership + non-identity). Validating only the sum is strictly weaker: a pubkey outside the G1 prime-order subgroup (small-subgroup / cofactor component) can pass if the malicious components cancel in the sum, so a caller can be made to accept an aggregate over keys that are individually invalid.
- **Impact & likelihood:** Accepts signatures verified against pubkey sets containing non-subgroup points → rogue-key / invalid-membership surface in `verify_aggregate` consumers (sigagg, validatormock). Divergence from charon spec behavior. · Only on adversarially crafted pubkey inputs (not natural data); requires an attacker controlling submitted pubkeys.
- **PoC:** CONFIRMED — scratch test `verify_aggregate_accepts_non_subgroup_members`: built a non-subgroup on-curve G1 point N (brute-forced compressed x, `blst_p1_affine_in_g1`=false) and its negation -N; `verify_aggregate([K, N, -N], sig_over_K, data)` → `Ok(())` (sum = K + N - N = K, only the aggregate is validated). Supporting tests: `from_bytes_accepts_non_subgroup_validate_rejects` (blst `PublicKey::from_bytes` accepts N; `validate()` → `BLST_POINT_NOT_IN_GROUP` — verified in blst 0.3.16 `from_bytes`→`deserialize`→`$pk_deser` does on-curve only, not subgroup); `spec_fastaggverify_rejects_non_subgroup_members` (`AggregatePublicKey::aggregate(&[K,N,-N], validate=true)` → `Err(BLST_POINT_NOT_IN_GROUP)`, i.e. charon's `KeyValidate`-equivalent path rejects the same set). Go not run but herumi `FastAggregateVerify` key-validates per spec (reasoned, Go not run).
- **Fix:** Validate each pubkey before aggregating — parse via `PublicKey::key_validate` (or call `pk.validate()` on each) which does the G1 subgroup + infinity check, matching `KeyValidate`; reject empty/identity. Alternatively use blst's `AggregatePublicKey::aggregate(&pks, true)` (groupcheck=true) instead of the raw point-add helper.

### [Medium] aggregate: single-signature path returns input bytes without validation
- **Rust:** `crates/crypto/src/blst_impl.rs:136` (`aggregate`)
- **Charon ref:** [`tbls/herumi.go:228`](https://github.com/ObolNetwork/charon/blob/v1.7.1/tbls/herumi.go#L228) (`Aggregate`)
- **Issue:** `if signatures.len() == 1 { return Ok(signatures[0]); }` returns the raw 96 bytes verbatim. Charon `Aggregate` always `Deserialize`s every signature (errors on malformed) and re-serializes a canonical encoding. Pluto's fast path skips deserialization entirely, so a malformed / non-canonically-encoded / non-subgroup single signature is propagated unchanged instead of being rejected, and the output is not normalized.
- **Impact & likelihood:** Divergent error semantics vs charon (charon errors, pluto succeeds) and a malformed signature passes through `aggregate` undetected → may surface later as a verify failure far from the source, or as a non-canonical byte string where charon would have produced canonical bytes. · Always for any single-element input; only matters when that input is malformed/non-canonical.
- **PoC:** CONFIRMED — scratch test `aggregate_single_returns_garbage_without_validation`: `BlstImpl.aggregate(&[[0xFF;96]])` → `Ok([0xFF;96])` (input returned verbatim), while `blst::min_pk::Signature::from_bytes(&[0xFF;96])` → `Err(BLST_BAD_ENCODING)`, proving the bytes are not a valid signature yet pass through unrejected. Charon `Aggregate` `Deserialize`s every sig and would error on these (reasoned, Go not run — herumi.go:228).
- **Fix:** Drop the short-circuit (or deserialize-then-reserialize the single sig with group check) so the validation/normalization path matches charon for n==1.

### [Medium] Secret material never zeroized (ikm, poly coeffs, recovered secrets, scalars)
- **Rust:** `crates/crypto/src/blst_impl.rs:33` (`generate_secret_key` `ikm`), `:79-88` (`threshold_split_insecure` `poly`/`ikm`), `:110-128` (`recover_secret` `share_secrets`/`recovered`), `:294-313`/`:404-423` (Lagrange scalar/secret intermediates); `crates/crypto/src/types.rs:19` (`PrivateKey = [u8;32]`)
- **Charon ref:** n/a (Go relies on GC; Rust audit focus calls out zeroize explicitly)
- **Issue:** Private keys, polynomial coefficients, per-share secret keys, the recovered master secret, IKM buffers, and intermediate `blst_scalar`/`blst_fr` values holding secret-derived data are dropped without zeroing. `PrivateKey` is a plain `[u8;32]` (Copy), so copies proliferate freely and none are wiped. `BlstSecretKey` from the `blst` crate is not zeroize-on-drop. Secret bytes linger in freed heap/stack.
- **Impact & likelihood:** Secret-share / root-key bytes remain recoverable from memory (core dumps, swap, heap reuse) → key-exfiltration surface for a DV node. · Always (every key op); exploitation requires memory disclosure.
- **PoC:** confirmed by inspection (no PoC needed) — `grep -rn 'zeroize|Zeroizing|ZeroizeOnDrop' crates/crypto/` returns nothing; `PrivateKey = [u8;32]` is `Copy`; the crate's `Cargo.toml` does not depend on `zeroize` (workspace declares `zeroize = "1.8.2"` at Cargo.toml:86 but crypto never pulls it); blst 0.3.16 `SecretKey` has no `Drop`/zeroize (the `serde-secret` zeroize path is feature-gated and unused here). `ikm`, `poly`, `share_secrets`, `recovered`, and `blst_scalar`/`blst_fr` intermediates are dropped without wiping.
- **Fix:** Wrap secret types in `zeroize::Zeroizing` (or derive `ZeroizeOnDrop`), zeroize `ikm`/`poly`/`share_secrets`/`recovered` and scalar intermediates after use; consider a dedicated `Secret`/`Zeroizing<[u8;32]>` newtype for `PrivateKey`.

### [Info] Not a finding as written: generate_insecure_secret acceptance criterion only diverges on all-zero insecure draw
- **Rust:** `crates/crypto/src/blst_impl.rs:42` (`generate_insecure_secret`)
- **Charon ref:** [`tbls/herumi.go:349`](https://github.com/ObolNetwork/charon/blob/v1.7.1/tbls/herumi.go#L349) (`generateInsecureSecret`)
- **Issue:** Both loop ≤100 times reading 32 random bytes and retry on rejection, but the acceptance test differs: pluto uses `BlstSecretKey::from_bytes` (blst: rejects zero and any value ≥ r, no reduction); Herumi uses `bls.SecretKey.Deserialize` (Herumi's own range/zero policy). For an identical RNG byte stream, the two impls can accept/reject different draws, so the i-th accepted secret — and hence every downstream insecure-split share/test vector — can diverge between pluto and charon. Test-only path, but `threshold_split_insecure` feeds reproducibility-sensitive fixtures.
- **Impact & likelihood:** Deterministic insecure-key/insecure-split test vectors generated by charon will not reproduce byte-for-byte in pluto only on the astronomically-rare divergent draw (all-zero 32 bytes) · ~2⁻²⁵⁶ per draw; a hard mismatch only if that exact draw occurs. (Down-weighted from "near-boundary"; the ≥r region is NOT divergent — see PoC.)
- **PoC:** REFINED — scratch test `insecure_secret_acceptance_boundary` pins blst's exact acceptance set: `BlstSecretKey::from_bytes` (big-endian, `blst_scalar_from_bendian` + `blst_sk_check`) gives `from_bytes(0)=false`, `from_bytes(r-1)=true`, `from_bytes(r)=false`, `from_bytes(r+1)=false` → accepts exactly `0 < x < r`. Herumi in ETH mode (`SetETHmode`) `SecretKey.Deserialize` is also big-endian 32-byte and rejects `x ≥ r` (canonical), so the ≥r region matches blst and is NOT a divergence (refutes the original "≥r reduction" worry). The ONLY residual divergence is the all-zero draw: herumi `Fr` Deserialize accepts 0 as a secret key whereas blst `blst_sk_check` rejects 0 → pluto retries, charon accepts (reasoned, Go not run — herumi binary binding not vendored). Probability ~2^-256; test-only path. Severity lowered to Info / not-a-finding as written.
- **Fix:** Accept as effectively non-divergent for practical RNG streams; if strict bit-for-bit reproducibility of charon insecure vectors is ever required, special-case the zero draw, or simply document that pluto's insecure (test-only) vectors are not guaranteed charon-reproducible. A shared fixed-RNG vector test against charon output would close the residual gap.

### [Low] aggregate / threshold_aggregate iterate a HashMap-derived order but result is order-independent — confirm parity, not a bug
- **Rust:** `crates/crypto/src/blst_impl.rs:156` (`threshold_aggregate`), `:131` (`aggregate`)
- **Charon ref:** [`tbls/herumi.go:228`](https://github.com/ObolNetwork/charon/blob/v1.7.1/tbls/herumi.go#L228)/`:252`
- **Issue:** Both pluto (`HashMap` iteration) and charon (`map[int]` iteration) aggregate in nondeterministic order. BLS aggregation (point addition) and Lagrange interpolation are commutative, so the final signature is order-independent — no parity break. Noted only because the audit flags ordering; verified commutative by inspection.
- **Impact & likelihood:** None (commutative) · n/a
- **PoC:** n/a
- **Fix:** None required.

### [Low] Misleading doc: claimed "max 255 shares / ID < 255" limit is not enforced and not a real Herumi limit
- **Rust:** `crates/crypto/src/tbls.rs:35-37`, `:54-56`, `:69-71` (trait doc comments)
- **Charon ref:** [`tbls/herumi.go:114`](https://github.com/ObolNetwork/charon/blob/v1.7.1/tbls/herumi.go#L114) (`SetDecString(strconv.Itoa(i))`)
- **Issue:** Docs assert a hard 255-share / ID<255 limit "due to underlying BLS library constraints". Herumi uses `ID.SetDecString` (arbitrary integer field element) and blst here uses `scalar_from_u64` (full u64) — neither imposes a 255 cap, and `threshold_split` accepts any `u64 total`. The comment describes a constraint that does not exist, risking incorrect caller assumptions.
- **Impact & likelihood:** Misleading API contract; no runtime effect · always (doc only)
- **PoC:** n/a
- **Fix:** Remove/correct the 255 claim; document the actual constraint (`2 <= threshold <= total`, indices must be unique nonzero field elements).

### [Low] threshold_split bound `threshold > total` checked, but total==0 / total==1 not rejected before share loop
- **Rust:** `crates/crypto/src/blst_impl.rs:70` (`threshold_split_insecure`)
- **Charon ref:** [`tbls/herumi.go:90`](https://github.com/ObolNetwork/charon/blob/v1.7.1/tbls/herumi.go#L90)/`:143` (only `threshold <= 1` checked)
- **Issue:** Pluto checks `threshold <= 1 || threshold > total`. With `threshold==2, total==... ` the `threshold > total` check already forces `total >= 2`, so `total` 0/1 is implicitly rejected (since `threshold>=2 > total`). Charon checks only `threshold <= 1` and would loop `for i:=1; i<=total` producing fewer shares than threshold for small total. Pluto is stricter (rejects `threshold>total`). Divergent error semantics: an input charon accepts (e.g. total=1, threshold=2 → 1 share, unrecoverable) pluto rejects. Minor; pluto behavior is safer.
- **Impact & likelihood:** Slightly different error surface vs charon on degenerate (total<threshold) inputs · only on misconfigured total/threshold
- **PoC:** n/a
- **Fix:** Acceptable as-is; if strict parity required, mirror charon's single `threshold <= 1` check. Recommend keeping the stricter check and documenting the divergence.

### [Info] Hardening: aggregate uses sigs_groupcheck=true; charon Herumi Aggregate does not group-check
- **Rust:** `crates/crypto/src/blst_impl.rs:150` (`aggregate`, `AggregateSignature::aggregate(&sigs, true)`)
- **Charon ref:** [`tbls/herumi.go:247`](https://github.com/ObolNetwork/charon/blob/v1.7.1/tbls/herumi.go#L247) (`sig.Aggregate` — no group check)
- **Issue:** Pluto group-checks each signature during aggregation; charon does not. Pluto is stricter (good), but it is a behavioral divergence: a non-subgroup partial signature that charon would aggregate is rejected by pluto. For honestly-produced signatures the result is identical.
- **Impact & likelihood:** None for honest inputs; stricter rejection on malformed input · negligible
- **PoC:** n/a
- **Fix:** None; keep the hardening. Note divergence for interop test design.

### [Info] BlsError mapping: BLST_AGGR_TYPE_MISMATCH and several variants collapse; some BlsError variants dead
- **Rust:** `crates/crypto/src/types.rs:176` (`From<BLST_ERROR>`), unused variants `:155-173`
- **Charon ref:** n/a
- **Issue:** `From<BLST_ERROR>` maps a subset; `BLST_PK_IS_INFINITY -> InvalidPublicKey` loses the "infinity" specificity, and `BlsError::{InvalidSecretKey, InvalidSignature, PointNotInGroup (only via from), Unknown}` plus `Error::BlsError(BlsError)` variant appear partly dead/duplicative (e.g. `InvalidPublicKey` exists both as a `BlsError` variant and an `Error` variant). No functional impact; error taxonomy is noisier than needed.
- **Impact & likelihood:** Cosmetic / maintainability · n/a
- **PoC:** n/a
- **Fix:** Prune unused `BlsError` variants; keep infinity distinct if callers care.

## Uncertain / needs-human

- **Polynomial-eval / Lagrange byte-for-byte parity with Herumi `sk.Set`/`pk.Recover`.** Pluto reimplements Shamir over the BLS scalar field (Horner eval at `x=i`, Lagrange at `x=0`) whereas charon delegates to Herumi `bls.SecretKey.Set(poly, id)` / `.Recover`. The interop test `verify_aggregate_from_data` (blst_impl.rs:531) hardcodes a Go-produced **aggregate** signature and passes, which strongly implies the recover/aggregate scheme is field-compatible. NOT directly verified: that an individual **share** `poly(i)` produced by pluto equals the share Herumi `Set` would produce for the same polynomial coefficients (only the recovered/aggregated result is pinned by the test). If charon ever persists or transmits individual shares cross-impl, confirm per-share equality. Phase 2 should add a fixed-polynomial vector comparing one pluto share to a Herumi share.
- **`scalar_from_u64` for share index vs Herumi `ID.SetDecString(strconv.Itoa(i))`.** Both should map integer `i` to the same field element; reasoned-equivalent by inspection but not executed against Herumi. Confirm for indices and for any non-1..n index set.
- **blst `Signature::from_bytes` / `PublicKey::from_bytes` group-check semantics.** Assumed `from_bytes` does NOT subgroup-check and the `true` flag in `verify(...)` / the `pk_validate` arg does. This assumption underpins the [High] `verify_aggregate` finding. A human should confirm against the pinned `blst` crate version that `from_bytes` alone is not a full `KeyValidate`.

## Summary

crypto is a from-scratch blst reimplementation of charon's Herumi-backed threshold BLS. Sign/verify DST and the recover/aggregate scheme appear interop-correct (a hardcoded Go aggregate-signature vector passes). Highest risk: `verify_aggregate` validates only the **summed** pubkey, skipping per-key subgroup checks that charon's spec `FastAggregateVerify` performs (High). Also: an unvalidated single-signature fast path in `aggregate` (Medium), no zeroization of any secret material (Medium). The insecure-key acceptance-criterion item is Info/not-a-finding as written: only an all-zero draw diverges, at ~2^-256, in test-only code. Per-share polynomial parity vs Herumi is unverified (only the aggregate is pinned by tests) and is the key Phase-2 item.
