# Audit: core-types  (9731 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/core/src/{types.rs,signeddata.rs,unsigneddata.rs,eth2signeddata.rs,ssz_codec.rs,corepb,lib.rs,version.rs,testutils.rs}`
- **Charon:** `core/{types,signeddata,unsigneddata,eth2signeddata,ssz,interfaces,proto}.go`
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `types.rs` ↔ `core/types.go` (Duty, DutyType, PubKey, etc.)
- `signeddata.rs`/`unsigneddata.rs`/`eth2signeddata.rs` ↔ `core/{signeddata,unsigneddata,eth2signeddata}.go`
- `ssz_codec.rs` ↔ `core/ssz.go`; `corepb` ↔ `core/proto.go` + `core/interfaces.go`

## Crate-specific focus
- **Parity:** SignedData/UnsignedData variant set + SSZ/proto encode round-trip equivalence; DutyType enum values/ordering; eth2 signing-domain handling. These types cross the wire — encodings must match charon byte-for-byte.
- **Security:** `ssz_codec`/proto decode bounds on untrusted bytes (offset/length validation, overflow).
- **Quality:** trait design for signed/unsigned data; enum exhaustiveness; clone discipline.

---
## Summary
Core domain types, the SignedData/Eth2SignedData trait family, signing-domain handling, DutyType enum values/ordering, proto conversions, and the version/semver port are all functionally equivalent to charon v1.7.1; the parsigex JSON-vs-SSZ codec and golden SSZ fixtures round-trip correctly. The one substantive parity gap is in the versioned-SSZ decoders, which require the value-offset field to equal the header exactly and always slice the inner payload at the constant header size, whereas charon accepts `offset >= header` and slices at the decoded offset — agreeing on all canonical Charon output but diverging on non-canonical bytes (mirrored by a same-shaped over-strict check in `AttestationData` SSZ decode). Remaining items are low/info: an off-by-one in `PubKey::abbreviated` log form, sentinel `Display` divergence, JSON-prefix-first codec ordering, and a dead error variant. No memory-safety, overflow, or panic-on-untrusted-input issues found (decode helpers length-check via `try_into` and all slices are bounded).

## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Medium] Versioned SSZ decoders require exact offset; Charon accepts `offset >= header`
- **Rust:** `crates/core/src/ssz_codec.rs:275` (`decode_versioned_attestation_no_val_idx`), `:379` (`decode_versioned_signed_aggregate_and_proof`), `:451` (`decode_versioned_signed_proposal`)
- **Charon ref:** [`core/ssz.go:804`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/ssz.go#L804) (`unmarshalSSZVersioned`, `if versionedOffset > o1`), `:737` (`unmarshalSSZVersionedBlinded`, `if versionedBlindedOffset > o1`)
- **Issue:** Go validates the value-offset field with `header > o1` (i.e. accepts any `o1 >= header`) and then slices the inner payload at the *decoded* offset `buf[o1:]`. Rust instead requires `offset == header` exactly and always slices the inner payload at the compile-time constant header size, ignoring the decoded offset value. (The val-idx attestation path `ssz.go:772` uses exact `o1 != 20`, which Rust matches — only the non-val-idx / blinded / aggregate paths diverge.) Two consequences: (a) a peer-encoded object with `o1 > header` (legal per Go's check) is rejected by Rust with `InvalidOffset`; (b) even if accepted, Rust would read the inner payload from the wrong start offset vs Go. Honest Charon encoders always emit `o1 == header`, so the two implementations agree on all well-formed Charon output.
- **Impact & likelihood:** Decode divergence on the wire-facing parsigex/SSZ path → a versioned attestation/aggregate/proposal that Go would accept is rejected (or misparsed) by Rust. · Negligible for honest Charon peers (offset is always exactly the header); only triggers on non-canonical/hand-crafted SSZ.
- **PoC:** CONFIRMED — scratch `scratch_versioned_offset_must_equal_header` (ssz_codec tests). Built a canonical no-val-idx `VersionedAttestation` SSZ (header=12, `o1==12`, decodes fine), then crafted a Go-LEGAL variant with `o1=16` (4 padding bytes after the offset field, `o1` pointed past them). Observed: `decode_versioned_attestation(crafted) → Err(InvalidOffset { expected: 12, got: 16 })`. Go (ssz.go:804) accepts any `o1 >= versionedOffset` and slices at the DECODED `buf[o1:]` (:815, skipping the padding); Rust requires `offset == header` exactly (ssz_codec.rs:275) and slices at the compile-time constant (:281). Confirms both consequences (rejection of `o1>header`, and wrong-start slice even if accepted). Honest Charon always emits `o1==header`, so they agree on well-formed output. Severity Medium retained (wire-facing, but non-canonical-only). Test deleted; worktree clean.
- **Fix:** Match Go: accept `offset >= header` and slice the inner payload at the decoded `offset` (`&bytes[offset as usize..]`) after bounds-checking `offset <= bytes.len()`, rather than at the constant.

### [Low] `PubKey::abbreviated()` selects the wrong trailing hex window vs Charon
- **Rust:** `crates/core/src/types.rs:454` (`PubKey::abbreviated`)
- **Charon ref:** [`core/types.go:288`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/types.go#L288) (`PubKey.String`, `string(k[2:5]) + "_" + string(k[94:97])`)
- **Issue:** Go's `PubKey` is the 98-char `"0x"+96 hex` string; its abbreviated form is `k[2:5]` + `_` + `k[94:97]`. Subtracting the 2-char `0x` prefix, the trailing window maps to hex indices `[92..95]`. Rust uses `hex[0..3]` (correct) but `hex[93..96]` for the tail — off by one (hex `[93..96]` instead of `[92..95]`). The existing unit test uses an all-`0x2a` key, so every hex char is identical and the off-by-one is masked.
- **Impact & likelihood:** Log/metric labels for a pubkey differ by one hex nibble from Charon, hindering cross-client log correlation. No wire/consensus effect. · Always (every abbreviated render of a non-uniform key).
- **PoC:** n/a
- **Fix:** Use `&hex[92..95]` for the trailing slice to match Go's `k[94:97]`.

### [Low] `AttestationData` SSZ decode requires `data_offset == 8` exactly; Charon accepts `>= 8`
- **Rust:** `crates/core/src/unsigneddata.rs:106` (`decode_attestation_data_ssz`)
- **Charon ref:** [`core/ssz.go:909`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/ssz.go#L909) (`AttestationData.UnmarshalSSZ`, `if size < o0 || minSize > o0`)
- **Issue:** Go requires the first offset `o0` to satisfy `o0 >= 8 && o0 <= size`; Rust requires `data_offset == 8` exactly and then decodes the inner attestation from `data[8..duty_offset]`. For honest Charon encoders `o0` is always exactly 8, so they agree; a non-canonical encoding with `o0 > 8` that Go accepts is rejected by Rust.
- **Impact & likelihood:** Stricter-than-Go decode of unsigned attestation data on the wire path · only on non-canonical SSZ (never from honest Charon).
- **PoC:** n/a
- **Fix:** Accept `data_offset >= ATTESTATION_DATA_SSZ_OFFSET` (still requiring `duty_offset >= data_offset` and bounds) to mirror Go.

### [Low] `deserialize_signed_data` decides JSON-vs-SSZ by prefix first; Charon tries SSZ first
- **Rust:** `crates/core/src/parsigex_codec.rs:189` (`deserialize_signed_data`, `let is_json = looks_like_json(bytes)` then JSON-only branch)
- **Charon ref:** [`core/proto.go:287`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/proto.go#L287) (`unmarshal`: tries `UnmarshalSSZ` first; only attempts JSON when SSZ fails *and* data has a `{` prefix)
- **Issue:** Go always attempts SSZ first for SSZ-capable types, falling back to JSON only on SSZ failure with a `{` prefix. Rust inverts the order: if the trimmed bytes start with `{` it decodes JSON and never tries SSZ. The two differ only for an input that simultaneously begins with `{` (0x7b) and is a valid SSZ encoding of the target type — not reachable for these fixed-layout eth2 types.
- **Impact & likelihood:** Behavioural divergence only for adversarial bytes that are valid SSZ *and* JSON-prefixed · effectively impossible for the concrete types involved.
- **PoC:** n/a
- **Fix:** Optional — reorder to try SSZ first then JSON-on-`{`-prefix to mirror Go exactly; document the intentional difference if kept.

### [Low] `DutyType::Display` diverges for the sentinel and `expect`s on serialize
- **Rust:** `crates/core/src/types.rs:55` (`impl Display for DutyType`), `:52` (`DutySentinel(Box<DutyType>)`)
- **Charon ref:** [`core/types.go:56`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/types.go#L56) (`DutyType.String`, map lookup → `""` for `dutySentinel`/unknown)
- **Issue:** Go's `String()` returns `""` for any value not in its map (including `dutySentinel`). Rust serializes via serde; the `DutySentinel(Box<DutyType>)` variant serializes to a JSON object, so `Display` falls through to `write!(f, "{}", v)` and prints `{"duty_sentinel":...}` rather than empty. Also the impl uses `.expect("failed to serialize duty type")` — sound for this enum but a latent panic-on-Display if the enum ever gains a non-serializable field. Modeling Go's plain sentinel constant as a recursive boxed variant is also unusual (quality).
- **Impact & likelihood:** Divergent string for the sentinel/unknown duty in logs; sentinel is internal and never displayed in normal flow · rare.
- **PoC:** n/a
- **Fix:** Map `Unknown` and `DutySentinel` to `""` in `Display` to match Go; consider replacing the boxed-recursive `DutySentinel` with a unit sentinel or dropping it (it exists only to bound `all()`/`is_valid()`).

### [Info] `SignedDataError::Custom` variant appears unused
- **Rust:** `crates/core/src/signeddata.rs:64` (`SignedDataError::Custom`)
- **Charon ref:** n/a (Rust-only)
- **Issue:** No constructor of `SignedDataError::Custom` is referenced in the crate; dead error variant.
- **Impact & likelihood:** Dead code / minor API noise · n/a.
- **PoC:** n/a
- **Fix:** Remove if unused, or wire it where a non-enumerated error is produced.

## Uncertain / needs-human
- **`ParSignedData` proto decode skips Charon's `protonil.Check`.** `types.rs:734` (`TryFrom<(&DutyType,&pbcore::ParSignedData)>`) does not replicate `proto.go:60`'s `protonil.Check(data)`. Inspection of `app/protonil/protonil.go` shows `Check` only validates *message*-typed sub-fields are non-nil; `ParSignedData`'s fields are all scalars/bytes, so `Check` is a no-op for it. Assessed as **not a divergence**, but flagging for a human to confirm no other proto entry point relies on `protonil.Check` semantics that Rust must mirror.
- **`share_idx` conversion narrowing.** Encode uses `i32::try_from(u64)` (`types.rs:719`) and decode uses `u64::try_from(i32)` (`:739`), erroring on out-of-range/negative; Go uses plain `int32(...)`/`int(...)` casts (`proto.go:184`, `:170`) which silently wrap/sign-extend. Practically share indices are small positive ints, so behaviour matches; confirm no path can legitimately carry a value outside `0..=i32::MAX`.

---
## Delta audit 2026-07-02 — commits since 2026-06-30 baseline (#508)
STATUS: VERIFIED

Scope: commit `c241b50` "feat(core): encode/decode UnsignedDataSet proto for all duty types". New/changed: `unsigneddata.rs` (proto encode + all-duty-type decode), `ssz_codec.rs` (unsigned versioned SSZ codecs), `signeddata.rs` (`Deserialize for VersionedProposal`, `UnsignedBlockContentsJson`), `parsigex_codec.rs` (SSZ-first reorder — findings in core-parsig.md).

### [Medium] New unsigned versioned SSZ decoders repeat the exact-offset divergence (Charon accepts `offset >= header`)
- **Rust:** `crates/core/src/ssz_codec.rs:756` (`decode_versioned_proposal`, offset check `:763`, slice `:770`), `:803` (`decode_versioned_aggregated_attestation`, offset check `:809`, slice `:816`)
- **Charon ref:** [`core/ssz.go:737`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/ssz.go#L737) (`unmarshalSSZVersionedBlinded`, `if versionedBlindedOffset > o1`, slice `buf[o1:]` `:748`), [`core/ssz.go:804`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/ssz.go#L804) (`unmarshalSSZVersioned`, `if versionedOffset > o1`, slice `buf[o1:]` `:815`)
- **Issue:** Same divergence already recorded in the top-level Medium for the *signed* versioned decoders, now repeated by #508's two *unsigned* decoders. Go validates the value-offset field with `header > o1` (accepts any `o1 >= header`) and slices the inner payload at the *decoded* `buf[o1:]`; both new Rust decoders require `offset == header` exactly (`InvalidOffset` otherwise) and always slice at the compile-time constant header, ignoring the decoded offset. `decode_versioned_proposal` reaches the unsigned `VersionedProposal` (DutyProposer) and `decode_versioned_aggregated_attestation` the `VersionedAggregatedAttestation` (DutyAggregator) proto/consensus decode paths. Two consequences: (a) a peer-encoded object with `o1 > header` (legal per Go) is rejected by Rust; (b) even if accepted, Rust reads the inner payload from the wrong start offset. Honest Charon always emits `o1 == header`, so they agree on canonical output.
- **Impact & likelihood:** Decode divergence on the wire-facing unsigned-duty consensus path → a proposal / aggregated attestation Go accepts is rejected (or misparsed) by Rust · Negligible for honest Charon peers (offset always exactly the header); triggers only on non-canonical / hand-crafted SSZ.
- **PoC:** CONFIRMED — scratch `scratch_unsigned_versioned_offset_must_equal_header` (pluto-core, ssz_codec tests). Encoded canonical unsigned `VersionedAggregatedAttestation` (header=12, `o1==12`) and Phase0 `VersionedProposal` (header=13, `o1==13`); both round-trip decode fine. Crafted Go-LEGAL variants with `o1 = header+4` (4 padding bytes inserted after the offset field, offset bumped past them). Observed: `decode_versioned_aggregated_attestation → Err(InvalidOffset { expected: 12, got: 16 })` and `decode_versioned_proposal → Err(InvalidOffset { expected: 13, got: 17 })`. Go (reasoned, not run): `unmarshalSSZVersioned` (ssz.go:804, `versionedOffset > o1`) and `unmarshalSSZVersionedBlinded` (:737, `versionedBlindedOffset > o1`) accept any `o1 >= header` and slice the inner payload at the DECODED `buf[o1:]` (:815 / :748, skipping the padding); both new Rust decoders require `offset == header` exactly (ssz_codec.rs:763 / :809) and slice at the compile-time constant header (:770 / :816). Confirms both consequences: (a) rejection of Go-legal `o1 > header`, (b) wrong-start constant slice even if it did not reject. Honest Charon always emits `o1 == header`, so they agree on canonical output. Severity Medium retained (wire-facing, non-canonical-only). Test deleted; `git status` clean.
- **Fix:** Same as the existing Medium: accept `offset >= header`, bounds-check `offset as usize <= bytes.len()`, and slice the inner payload at the decoded `&bytes[offset as usize..]`. Apply to both new decoders (and ideally consolidate all five versioned decoders onto one offset-checking helper).

### [Low] Attester SSZ encode zeroes the `AttesterDuty` pubkey — byte-divergent from Charon
- **Rust:** `crates/core/src/unsigneddata.rs:110` (`encode_attestation_data_ssz`, `out.extend_from_slice(&[0u8; 48])`)
- **Charon ref:** [`core/ssz.go:937`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/ssz.go#L937) (`attesterDutySSZ.MarshalSSZTo`, `dst = append(dst, a.PubKey[:]...)`)
- **Issue:** Charon writes the real 48-byte validator pubkey (`eth2v1.AttesterDuty.PubKey`) into the `AttesterDuty` SSZ body; pluto's `AttesterDuty` type has no pubkey field, so the new encoder writes 48 zero bytes. The unsigned-attester SSZ bytes therefore differ from Charon's for the same duty. Not a functional break: on decode both sides skip bytes `[0..48]` (pluto's `decode_attester_duty_ssz` reads only the trailing six u64s), and Charon keys attester data by `(slot, committeeIndex, validatorIndex)` in `dutydb` (`memory.go` `Store`/`PubKeyByAttestation`) and by the outer `UnsignedDataSet` map key — never by `Duty.PubKey`. So round-trip and cross-impl decode succeed despite the byte difference. (The docstring's "recovered from the aggregation bits downstream" rationale is imprecise — the pubkey is carried by the map key / dutydb index, not the aggregation bits.)
- **Impact & likelihood:** Wire bytes for unsigned attester data differ from Charon by the 48 pubkey bytes; tolerated by both decoders and by Charon's slot/idx-based mapping, so no observed consensus/mapping break · always (every attester encode), but functionally inert under current Charon usage.
- **PoC:** n/a
- **Fix:** None strictly required for interop; if exact byte parity is ever mandated, thread the validator pubkey into `AttesterDuty` and emit it. Correct the docstring to state the pubkey is recovered from the set map key / dutydb, not the aggregation bits.

### [Low] Deneb/Electra/Fulu proposal SSZ encode clones the whole block and all blobs
- **Rust:** `crates/core/src/ssz_codec.rs:617` (`encode_unsigned_proposal_block`, Deneb/Electra/Fulu arms — `(**block).clone()`, `kzg_proofs.clone()`, `blobs.clone()`)
- **Charon ref:** [`core/ssz.go:178`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/ssz.go#L178) (`VersionedProposal.MarshalSSZTo` — marshals fields in place, no copy)
- **Issue:** To reuse the `ssz_derive`-derived `DenebBlockContents`/`ElectraBlockContents`, the encoder deep-clones the boxed beacon block plus every KZG proof and blob into a temporary container before `as_ssz_bytes()`. A `deneb::Blob` is 128 KiB; up to `MAX_BLOBS` per block → up to ~0.75 MiB of blob data (plus the block body) copied per proposal encode purely to serialize.
- **Impact & likelihood:** Extra allocation + memcpy of the full block contents on every Deneb+ proposal encode · rare (proposals are infrequent) and bounded, hence Low.
- **PoC:** n/a
- **Fix:** Serialize the three fields directly into a manual offset-table layout (as `encode_attestation_data_ssz` already does), or have the container borrow (`&`) its fields, avoiding the clone.

### [Info] New unsigned aggregated-attestation decode always rewraps as versioned; re-encode is not byte-identical to a legacy non-versioned input
- **Rust:** `crates/core/src/unsigneddata.rs:182` (`decode_aggregated_attestation`), `:209` (`wrap_phase0_aggregated_attestation`)
- **Charon ref:** [`core/unsigneddata.go:674`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/unsigneddata.go#L674) (`unmarshalUnsignedData`, `DutyAggregator`: returns either `VersionedAggregatedAttestation` or non-versioned `AggregatedAttestation`)
- **Issue:** Charon can return either the versioned or the non-versioned (`AggregatedAttestation` = raw `phase0.Attestation`) type. Pluto models only the versioned variant, so a legacy non-versioned SSZ/JSON aggregate is wrapped as a Phase0-versioned attestation (documented). Decode is functionally equivalent, but if pluto later *re-marshals* that value it emits the 12-byte-header versioned form, not the raw phase0 form Charon would emit — the re-encoded bytes differ from the original legacy bytes. In the current flow unsigned aggregated data is decoded and consumed (not re-broadcast/compared as bytes), so no divergence is reachable; flagged for awareness only.
- **Impact & likelihood:** None in the current consensus flow (decode-only compat path; modern Charon emits versioned) · n/a.
- **PoC:** n/a
- **Fix:** None required; keep the documented wrapping. Note the re-encode asymmetry if a raw-aggregate re-marshal path is ever added.

### [Info] New SSZ block-contents containers do not enforce spec list-max lengths (systemic)
- **Rust:** `crates/core/src/ssz_codec.rs:600` (`DenebBlockContents`), `:609` (`ElectraBlockContents`) — `Vec<deneb::KZGProof>` / `Vec<deneb::Blob>` via `ssz_derive::Decode`
- **Charon ref:** go-eth2-client `deneb.BlockContents.UnmarshalSSZ` (fastssz — `ssz-max` on `KZGProofs`/`Blobs`, returns `ErrListTooBig` past the cap)
- **Issue:** `ethereum_ssz_derive` does not support `ssz-max`, so these lists decode however many fixed-size elements the body contains; Charon's fastssz rejects lists over the spec maximum. Not an unbounded-allocation DoS (element count is bounded by the already-in-memory body length ÷ fixed element size), but it is a validation-strictness divergence: an over-long-list proposal that Charon rejects is accepted by pluto. This is the same systemic gap as every other pluto SSZ `Vec` (pre-existing, not #508-specific); #508 only adds two more instances.
- **Impact & likelihood:** Pluto accepts non-canonical over-max blob/proof lists that Charon rejects · only on malformed/adversarial SSZ; systemic across pluto SSZ types.
- **PoC:** n/a
- **Fix:** Systemic — add post-decode length assertions against the fork's `MAX_BLOB_COMMITMENTS`/blob caps for these containers (and, ideally, for the spec list types generally) to match fastssz.

### Delta summary
#508 adds the all-duty-type unsigned proto codec; encode/decode round-trip within pluto (tests cover all four duty types + leading-`0x7B` SSZ and JSON fallbacks). One Medium: the two new unsigned versioned decoders (`decode_versioned_proposal`, `decode_versioned_aggregated_attestation`) repeat the existing exact-offset-vs-`>=` divergence (interop-safe for canonical Charon output). Lows: attester encode zeroes the 48-byte duty pubkey (byte-divergent but functionally inert), and Deneb+ proposal encode deep-clones block+blobs. Two Info notes (legacy-aggregate rewrap re-encode asymmetry; missing SSZ list-max enforcement, systemic).
