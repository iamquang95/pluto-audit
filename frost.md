# Audit: frost  (2445 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/frost/src/*` (curve, frost_core, kryptology, kryptology_interop/round_trip tests)
- **Charon:** `dkg/frost.go` + kryptology FROST (charon uses coinbase/kryptology)
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `frost_core.rs`/`curve.rs` ↔ FROST primitives
- `kryptology.rs` + interop tests ↔ byte-compat layer with coinbase/kryptology (charon's FROST lib)

## Crate-specific focus
- **Parity (interop-critical):** pluto FROST must be byte-compatible with kryptology's FROST — verify shared interop test vectors, challenge/nonce derivation, commitment + share serialization formats, curve choice and ops.
- **Security:** scalar/point validation on deserialization, RNG for nonces, constant-time concerns.
- **Quality:** interop test coverage breadth; error handling on malformed input.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

Parity baseline verified against coinbase/kryptology @ `1dcc062` (charon pins the
`ObolNetwork/kryptology v0.1.0` fork; fixtures are the byte-compat anchor). The
interop-critical primitives all MATCH kryptology and need no change:
challenge preimage `byte(id)||ctx||A0.compressed||R.compressed` → `kryptology_challenge`;
hash-to-scalar `ExpandMsgXmd(SHA-256, DST="BLS12381_XMD:SHA-256_SSWU_RO_", 48)` then
big-endian wide-reduce → `kryptology_hash_to_scalar`; `wi = k + a0·ci` (`s.MulAdd(ci,ki)`);
proof check `R' = wi·G − ci·A0`, recompute + compare, reject `ci==0`; scalars 32-byte BE,
points 48-byte compressed G1; Shamir share x-coordinate = 1-indexed `id`; threshold≥2,
limit≤255, limit≥threshold; group VK = Σ A_{i,0}, VkShare = G·sk_total. RFC 9380 vectors
for `expand_msg_xmd` pass. Findings below are the divergences/issues found.

> **Note — [PR #507](https://github.com/NethermindEth/pluto/pull/507) (secret-hardening):** This PR added, in the frost crate, explicit `Zeroize::zeroize` of the secret scalars in `kryptology::round1`/`round2` (nonce `k`, per-peer share scalars, reconstructed signing key) and redacting manual `Debug` impls for `SigningShare`, `KeyPackage`, and `ShamirShare` (dropping their derived `Debug` so secret bytes can't leak via logs/panics). This was **not** a finding in this file — the scan flagged no zeroize/Debug-leak issue in frost — so it is proactive hardening, not a remediation of a tracked item. Zeroization here is best-effort (`Scalar: Copy`, so blst arithmetic intermediates aren't wiped), as documented in `curve.rs`.

### [Low] round2: bcast upper-bound stricter than kryptology (rejects len == max_signers)
- **Rust:** `crates/frost/src/kryptology.rs:397-405` (`round2`)
- **Charon ref:** kryptology `pkg/dkg/frost/dkg_round2.go` Round2 length guard (`if uint32(len(bcast)) > dp.feldman.Limit || ... < Threshold-1`) | charon `dkg/frost.go` n/a (delegates to lib)
- **Issue:** Pluto bounds `received_bcasts.len()` to `[threshold-1, max_signers-1]` (`max_received = max_signers-1`). Kryptology bounds `len(bcast)` to `[Threshold-1, Limit]` where `Limit == max_signers` — i.e. kryptology accepts up to `max_signers` broadcasts, pluto caps at `max_signers-1`. (p2pSend bound `[Threshold-1, Limit-1]` does match.) Pluto's bcast cap is off-by-one stricter than the reference.
- **Impact & likelihood:** Behavioral divergence only at the `kryptology::round2` API boundary in isolation: kryptology's `DkgParticipant.Round2` accepts `len(bcast) == max_signers` (charon's transport delivers exactly that — `frostp2p.go:268` echoes the node's own bcast to itself and `getRound2Inputs` filters only by `ValIdx`, not `SourceID`, so the map fed to the lib includes the self-bcast → `max_signers` entries), whereas pluto's `kryptology::round2` rejects it. NOT hit in pluto's assembled pipeline: the pluto caller `crates/dkg/src/frost.rs:258-262` strips the self-bcast before calling `kryptology::round2`, passing exactly `max_signers-1`. So integrated pluto DKG is unaffected; the off-by-one only matters if `kryptology::round2` is called directly with the un-stripped charon-style input. No security loss (pluto is stricter). · only on direct API use with un-stripped/non-standard bcast counts.
- **PoC:** REFINED — scratch test drove `kryptology::round2` (3-of-3) with all `max_signers`=3 bcasts incl. self → `Err(IncorrectPackageCount)`; confirms cap is `max_signers-1` (kryptology allows `max_signers`). But charon parity is preserved because pluto's `dkg/frost.rs` caller strips self before the call, so the scanner's "never hit in normal flow" conclusion holds (for a different reason: caller-side self-strip, not "node receives max_signers-1 bcasts"). Go side reasoned (kryptology not vendored/run).
- **Fix:** To mirror kryptology exactly, allow `received_bcasts.len()` up to `max_signers` (keep the explicit `sender_id == secret.id` self-rejection). Or document the intentional tightening. Confirm whether charon's transport can ever deliver `max_signers` bcasts before changing.

### [Low] from_compressed rejects identity / always subgroup-checks; kryptology does neither
- **Rust:** `crates/frost/src/curve.rs:200-206` (`G1Projective::from_compressed`), `:265-276` (`G1Affine::from_compressed`)
- **Charon ref:** kryptology `pkg/core/curves/bls12381_curve.go` `PointBls12381G1.FromAffineCompressed` (delegates to `bls12381.G1.FromCompressed`, no explicit identity reject, no separate subgroup check)
- **Issue:** Pluto rejects the point-at-infinity encoding and unconditionally runs `blst_p1_affine_in_g1` (full subgroup check) on every deserialized commitment. Kryptology's wrapper does neither explicitly, so it would accept an infinity-encoded coefficient commitment that pluto rejects as `InvalidPoint`. Used in `from_commitments` / `deserialize_commitment` for peer commitments.
- **Impact & likelihood:** Pluto is strictly safer (subgroup-validates untrusted points). Pure-divergence risk: a peer commitment whose any coefficient is the identity point (or a non-G1 point kryptology happens to accept) makes pluto error where kryptology succeeds → a DKG that completes under Go would abort under pluto. Honest coefficients a_k are random-nonzero, so identity is ~2⁻²⁵⁵; non-subgroup points only arise from a malicious/buggy peer. · negligibly rare honestly / only on crafted input.
- **PoC:** n/a
- **Fix:** Keep the validation (security win); document the intentional hardening over kryptology so a future interop diff is understood. No code change recommended unless strict byte-for-byte accept-set parity is required.

### [Low] round2: non-canonical / out-of-range share value surfaces as InvalidScalar, not InvalidShare{culprit}
- **Rust:** `crates/frost/src/kryptology.rs:447` (`round2`, `scalar_from_be(&share.value)?`)
- **Charon ref:** kryptology `pkg/dkg/frost/dkg_round2.go` (`Verifiers.Verify` calls `Scalar.SetBytes(share.Value)`; a ≥r value yields a verification failure attributable to that participant)
- **Issue:** If a peer sends a share value ≥ field order, pluto returns `KryptologyError::InvalidScalar` (no culprit) instead of attributing it to the sending participant like the Feldman-verify failure path (`InvalidShare { culprit }`). Loses the attacker-attribution that kryptology/charon preserves.
- **Impact & likelihood:** Reduced diagnosability / weaker culprit attribution on malformed P2P share; functionally still rejects the DKG. · only on malformed/malicious share bytes.
- **PoC:** n/a
- **Fix:** Map the `scalar_from_be(&share.value)` error to `InvalidShare { culprit: sender_id }` (the `wi`/`ci` decode at lines 425-426 has the same pattern — those are arguably `InvalidProof { culprit }`).

### [Low] expand_msg_xmd panics on DST > 255 instead of hashing it (RFC 9380 long-DST path)
- **Rust:** `crates/frost/src/kryptology.rs:184` (`expand_msg_xmd`, `assert!(dst.len() <= 255 ...)`)
- **Charon ref:** kryptology `native` `ExpandMsgXmd` (RFC 9380 §5.3.3 maps DST>255 to `H("H2C-OVERSIZE-DST-" || DST)`)
- **Issue:** Per RFC 9380 a DST longer than 255 bytes must be replaced by its hash; pluto instead `assert!`-panics. The only caller passes the fixed 29-byte `KRYPTOLOGY_DST`, so unreachable today, but the helper is `fn`-private-general and an internal-only `assert` on a public-shaped routine is a latent panic if reused.
- **Impact & likelihood:** No impact for current DKG (DST fixed < 255). Latent panic if the helper is reused with a long DST. · never on current call sites.
- **PoC:** n/a
- **Fix:** Either implement the RFC 9380 oversize-DST hashing branch, or keep the assert and add a doc-comment that the helper is specialized to short DSTs. Same applies to the `ell <= 255` / `len_in_bytes <= 65535` asserts (both fine for len=48).

### [Info] Round1 self-share generation skipped vs kryptology generating all n then taking shares[id-1]
- **Rust:** `crates/frost/src/kryptology.rs:342-356` (`round1`), recomputed in `round2` at `:408-409`
- **Charon ref:** kryptology `dkg_round1.go` (`feldman.Split` evaluates all 1..=limit; `p2pSend[id]=shares[id-1]`, self share kept internally)
- **Issue:** Pluto skips `j == id` when emitting shares and re-derives its own share in round2 via `from_coefficients(coefficients, own_identifier)`. Functionally identical (both evaluate the polynomial at x=id), just a structural difference — noted to confirm the own-share x-coordinate equals the participant id (it does), which is what makes VkShare/aggregation match.
- **Impact & likelihood:** None — equivalent result. · n/a
- **PoC:** n/a
- **Fix:** None.

### [Info] BlsSignature::verify builds min_pk::PublicKey from raw affine without independent validation
- **Rust:** `crates/frost/src/kryptology.rs:626-635` (`BlsSignature::verify`)
- **Charon ref:** n/a (Rust-only; charon BLS verify lives in `tbls`/herumi)
- **Issue:** `PublicKey::from(pk_affine.0)` / `Signature::from(sig_affine)` wrap raw blst affines with no membership check, relying on `sig.verify(true, .., true)` (sig group-check + pk validate) for safety. The pk here is always an internally-derived `VerifyingKey`, so untrusted input never reaches it; acceptable. Documented for completeness.
- **Impact & likelihood:** None on internal call paths. · n/a
- **PoC:** n/a
- **Fix:** None required; if `verify` is ever exposed to externally-supplied keys, validate the pubkey is in G1 and non-identity first.

## Uncertain / needs-human

- **expand_msg_xmd exact multi-block byte-for-byte vs kryptology `native.ExpandMsgXmd`:** the kryptology `expand.go`/`expand_msg.go` source 404'd via WebFetch (path differs in this commit), so I verified `expand_msg_xmd` only against the RFC 9380 SHA-256 published vectors (which pass, including the 128-byte multi-block case). Kryptology calls the standard RFC 9380 construction with the same DST and len=48, so equivalence is near-certain, but a human with the vendored fork source should confirm there is no kryptology-specific quirk (e.g. DST_prime handling) — interop-critical if it ever diverges.
- **kryptology native `fq.Bytes()` endianness assumption:** I inferred `ScalarBls12381.Bytes()` yields big-endian (internal LE limbs `ReverseScalarBytes`-flipped) → matches pluto `scalar_to_be` and the BE fixture `value`/`wi`/`ci` fields. The fixtures (generated from the Obol fork) corroborate this end-to-end, but the underlying `bls12381.Bls12381Fq.Bytes()` ordering was not read directly. Treat as confirmed-by-fixture, flagged for completeness.

## Summary

FROST/kryptology byte-compat layer is sound: the interop-critical primitives
(challenge preimage, hash-to-scalar DST+expand+BE-reduce, Schnorr `wi`/proof-verify,
scalar BE / point compressed-G1 encodings, 1-indexed Shamir x-coords, threshold/limit
bounds, group-VK and VkShare aggregation) all match coinbase/kryptology `1dcc062`, and
golden fixtures plus RFC 9380 vectors anchor the encoding end-to-end. No Critical/High.
One Medium: an off-by-one Low direct-API-only `round2` broadcast-count upper bound vs kryptology
(`max_signers-1` vs `max_signers`) — divergence only, not a security loss. Remaining items
are Low/Info: extra (safe) point validation over kryptology, weaker culprit attribution on
out-of-range share scalars, and a latent long-DST panic in `expand_msg_xmd` (unreachable
with the fixed 29-byte DST).
