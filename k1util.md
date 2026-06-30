# Audit: k1util  (336 LOC) — STATUS: DONE

**Charon counterpart:** app/k1util/k1util.go
**Audited:** 2026-06-30 · **Dimensions:** parity, security, quality

## Summary
Core sign/verify/recover logic is functionally close to charon and the shared test vector (PRIV_KEY_1/DIGEST_1 → SIG_1) passes, confirming byte-for-byte signature parity. Top risks: `save` writes the private key world/group-readable (0o644) where charon enforces 0o600 (security); `verify_64` rejects high-S signatures that charon's dcrd `Verify` accepts (verification parity); and `recover` validates the recovery byte more loosely than charon (accepts 2/3; different error type for invalid bytes).

## Findings

### [High] Saved private key file is world/group-readable
- **Where:** crates/k1util/src/k1util.rs:202-208 (`save`)
- **Charon ref:** [app/k1util/k1util.go:147](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L147) (`os.WriteFile(file, ..., 0o600)`)
- **Issue:** `std::fs::write` creates the file with mode `0o666 & ~umask`, typically `0o644`, leaving the hex-encoded secp256k1 private key readable by group and other. Charon deliberately writes `0o600` (owner-only). This is the ENR/operator signing key at rest (saved via p2p key handling); on a multi-user host it is exposed.
- **Verified:** PoC `poc_save_file_permissions` (added, run, deleted). Observed `saved key file mode = 644`; the `== 0o600` assertion FAILED (left 420 / right 384). Confirmed.
- **Recommend:** Write with explicit owner-only perms on Unix: `OpenOptions::new().write(true).create(true).truncate(true).mode(0o600).open(file)` then write hex; on non-Unix `fs::set_permissions` after write. Mirrors charon's `0o600`.

### [Medium] verify_64 rejects high-S signatures that charon accepts
- **Where:** crates/k1util/src/k1util.rs:133-155 (`verify_64`)
- **Charon ref:** [app/k1util/k1util.go:78-94](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L78-L94) (`Verify64` → dcrd `ecdsa.NewSignature(r,s).Verify`)
- **Issue:** k256 `verify_prehashed` rejects non-low-S (high-S) signatures, whereas dcrd's `Signature.Verify` does not enforce low-S — it verifies raw (r,s). A counterparty signing with charon (or any non-normalizing signer) can produce a 64-byte signature that charon's `Verify64` accepts but pluto's `verify_64` returns `false` for, rejecting otherwise-valid peer signatures. Pluto's own `sign` always emits low-S, so self-produced sigs are unaffected.
- **Verified:** PoC `poc_high_s_malleability` (added, run, deleted). Observed: `sign()` output is low-S; its n−s counterpart is high-S; `verify_64` on the high-S sig returned `Ok(false)` — confirms k256 rejects high-S. The "charon accepts" half is dcrd's documented Verify semantics (reasoned, not executed against Go).
- **Recommend:** Decide the contract. For charon parity, normalize before verifying: `let sig = Signature::from_slice(sig)?; let sig = sig.normalize_s().unwrap_or(sig);`. If strict low-S is intentional, document the divergence explicitly.

### [Low] recover accepts recovery bytes 2 and 3 (parity gap)
- **Where:** crates/k1util/src/k1util.rs:170-182 (`recover`)
- **Charon ref:** [app/k1util/k1util.go:109-111](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L109-L111) (rejects `sig[64]` unless in {0,1,27,28} with "invalid recovery id")
- **Issue:** Pluto only remaps 27/28→0/1, then hands the raw byte to `RecoveryId::from_byte`, which accepts 0,1,2,3 (2/3 = x-reduced variants). Bytes 2 and 3 bypass pluto's validation; charon rejects them up front. In practice recovery for 2/3 almost always fails inside `recover_from_prehash` (returning `InvalidSignature`), so the usual observable difference is a wrong/less-specific error type vs charon's "invalid recovery id"; on the astronomically rare x-reduced signature pluto could return a key where charon errors. Low real-world risk, genuine parity gap. Severity lowered to Low because normal observable impact is error-type specificity.
- **Verified:** PoCs `poc_recover_accepts_recid_2_3`, `poc_recid_2_3_can_recover_wrong_key`, `poc_recover_27_28_recovers_correct_key` (added, run, deleted). Observed: byte=2 → `Err("signature error")` (NOT `InvalidSignatureRecoveryId`), proving `from_byte(2)` is accepted and the byte reaches recovery; 50-trial search found no successful 2/3 recovery (x-reduction negligibly rare); 27/28→0/1 remap recovers the correct key.
- **Recommend:** Reject up front exactly like charon, before remapping: `if !matches!(sig[K1_REC_IDX], 0 | 1 | 27 | 28) { return Err(InvalidSignatureRecoveryId { invalid_recovery_byte: sig[K1_REC_IDX] }) }`.

### [Low] InvalidSignatureRecoveryId reports post-normalization byte
- **Where:** crates/k1util/src/k1util.rs:170-182
- **Charon ref:** [app/k1util/k1util.go:110](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L110) (reports original `sig[k1RecIdx]`)
- **Issue:** The `invalid_recovery_byte` field holds `recovery_byte` after `wrapping_sub(27)`, so the reported value can differ from the caller-supplied byte, hurting debuggability. (Coupled with the Medium fix above, which also keys off the original byte.)
- **Verified:** reasoned-only (no PoC).
- **Recommend:** Capture `let original = sig[K1_REC_IDX];` and report `original` in the error.

### [Low] public_key_from_libp2p: divergent error semantics + gratuitous clone
- **Where:** crates/k1util/src/k1util.rs:87-91
- **Charon ref:** [app/k1util/k1util.go:34-41](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L34-L41) (`errors.New("invalid public key type")`)
- **Issue:** Charon does an explicit type assertion returning a domain error. Pluto calls `pk.clone().try_into_secp256k1()` → `OtherVariantError` (wrapped). Same outcome (error on non-secp256k1 key), different error type/message — matters only if callers pattern-match to distinguish "wrong type" from "malformed key". `pk.clone()` is unnecessary churn given the function takes `&Libp2pPublicKey`.
- **Verified:** reasoned-only (no PoC).
- **Recommend:** Acceptable error variant; confirm no caller relies on a distinct wrong-type code. Avoid the clone by taking `pk` by value or cloning only on the branch that needs it.

### [Low] #[allow(deprecated)] FieldBytes::from_slice in verify_64
- **Where:** crates/k1util/src/k1util.rs:149-150
- **Charon ref:** n/a
- **Issue:** Deprecated `FieldBytes::from_slice` behind an allow + TODO. Safe here only because the preceding `hash.len() != K1_HASH_LEN` check makes the panic-on-wrong-length path unreachable; the invariant is non-obvious and undocumented.
- **Verified:** reasoned-only (no PoC).
- **Recommend:** Replace with a length-checked conversion (return existing `InvalidHashLength`) or bump k256 per the TODO; keep the length-check adjacent and comment the invariant.

### [Info] sign/recover compact-format parity confirmed
- **Where:** crates/k1util/src/k1util.rs:96-119, 157-188
- **Charon ref:** [app/k1util/k1util.go:43-59](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L43-L59), 96-126
- **Issue:** Pluto emits [R||S||V] with V = recovery_id.to_byte() (0/1); charon does `recovery-27`→0/1. Equivalent; shared test vector passes including low-S canonicalization (both dcrd SignCompact and k256 emit low-S; both use RFC 6979 deterministic nonces).
- **Verified:** existing test `k1_util` passes (matches charon vector).
- **Recommend:** none.

### [Info] verify_64 zero/overflow scalar handling matches charon
- **Where:** crates/k1util/src/k1util.rs:145
- **Charon ref:** [app/k1util/k1util.go:154-174](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L154-L174) (`to32Scalar` rejects zero R, zero S, overflow)
- **Issue:** k256 `Signature::from_slice` already rejects zero r/s and out-of-range scalars, giving the same reject-set as charon's `to32Scalar`. No observable divergence. The explicit `hash.len()==32` check in pluto is stricter than charon (charon has none on Verify64) — a safe, intentional improvement.
- **Verified:** reasoned-only (k256 Signature invariants).
- **Recommend:** none; optionally comment the intentionally-stricter hash-length check.

## Uncertain / needs-human
- High-S parity (Medium): pluto-rejects-high-S is confirmed by PoC; the "charon accepts it" half rests on dcrd's documented `Signature.Verify` not enforcing low-S — not executed against the Go binary. Worth a cross-impl test if peer signatures may come from non-normalizing signers.
- Intended `verify_64` contract: strict low-S enforcement (stronger than charon) vs exact parity — drives whether the fix is "normalize_s before verify" or "document divergence".
- `save` atomicity/durability (no fsync, non-atomic write) was not flagged as a finding; confirm whether key-write atomicity matters for the deployment model.
