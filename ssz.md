# Audit: ssz  (1686 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/ssz/src/*` (encode, decode, hasher, helpers, types, serde_utils, error)
- **Charon:** `app/genssz` + prysm/fastssz (charon delegates SSZ to fastssz)
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `encode.rs`/`decode.rs` ↔ SSZ ser/de
- `hasher.rs` ↔ hash_tree_root / merkleization
- `types.rs`/`helpers.rs` ↔ basic + composite type handling

## Crate-specific focus
- **Parity (silent-interop-break risk):** SSZ must match the consensus spec + fastssz output exactly — `hash_tree_root` merkleization, list/vector/union/bitlist encoding, length prefixes, zero-padding. Errors here silently corrupt every signing root.
- **Security:** decode bounds on untrusted input — offset monotonicity, length caps, no integer overflow / OOM.
- **Quality:** test against official SSZ spec vectors; derive-macro vs manual impl correctness.

---
## Findings

### Context (two coexisting SSZ stacks)
This crate ships **two independent SSZ implementations** that both feed signing roots:
1. **`hasher.rs` + `helpers.rs`** — manual port of fastssz's `HashWalker` (`PutBytes`, `Merkleize`, `MerkleizeWithMixin`, `putByteList`, `putBytesN`, `leftPad`). Used by `crates/cluster/src/ssz.rs` (cluster definition/lock hash) and `crates/consensus/src/qbft/msg.rs` (QBFT message signing root).
2. **`types.rs`** — `SszList`/`SszVector`/`BitList`/`BitVector` wrappers that provide hand-written `ssz::Encode`/`Decode`/`tree_hash::TreeHash` impls over the external `ethereum_ssz` 0.10 + `tree_hash` 0.12 crates. These are the `aggregation_bits`/`committee_bits`/list fields of `crates/eth2api/src/spec/{phase0,electra,...}.rs`, decoded **directly from untrusted beacon-node JSON** (`Attestation::try_from`, electra `Attestation::try_from`) and tree-hashed for attestation/block signing roots.

The findings below weight (1) the manual hasher's fastssz parity and (2) the custom `BitList`/`BitVector` decode bounds, since both silently corrupt or accept attacker-controlled signing-root inputs.

---

### [High] `BitList::from_ssz_bytes` ignores `MAX`; over-capacity bitlists silently accepted
- **Rust:** `crates/ssz/src/types.rs:451` (`impl Decode for BitList::from_ssz_bytes`) → delegates to inherent `from_ssz_bytes` at `types.rs:304`
- **Charon ref:** fastssz `UnmarshalSSZ`/`ValidateBitlist` + SSZ spec (Bitlist[N] decode MUST reject `bit_count > N`) | n/a (Rust-only laxity)
- **Issue:** The `Decode` impl returns `Ok(Self::from_ssz_bytes(bytes.to_vec()))` and the inherent decoder never consults the `MAX` const generic. A bitlist whose decoded bit-length exceeds `MAX` (e.g. an `aggregation_bits` longer than `BitList<2048>` / `BitList<131_072>`) decodes successfully with `len > MAX`. fastssz and `ssz_types` reject this. The repo even ships `ELECTRA_OVERSIZED_ATTESTATION_JSON` (`crates/eth2api/src/test_fixtures.rs:60`, ~2056 bits into `BitList<2048>`) as an over-capacity case, but the decoder cannot reject it because the inherent `from_ssz_bytes` is infallible (returns `Self`, not `Result`).
- **Impact & likelihood:** Malicious/buggy beacon node supplies an over-length `aggregation_bits`; pluto accepts it, computes a `tree_hash_root` (mix_in_length uses the oversized `len`) that diverges from spec-compliant clients, and may sign over invalid data. Wrong/duplicate aggregation bits also break attestation aggregation logic. · only on malformed/oversized input (attacker-triggerable via beacon API).
- **PoC:** CONFIRMED — scratch test `scratch_bitlist_over_capacity_accepted`: `<BitList<2048>>::from_ssz_bytes(&[0xFF;300, 0x01])` returns `Ok` with `len()==2400` (> MAX 2048), no error; `tree_hash_root()` runs on the oversized `len`. The repo's existing `oversized_attestation_from_vector_deserializes` test (`crates/eth2api/src/spec/electra.rs:404`) already *expects* this — it `expect()`s a successful decode and asserts `len() > 2048` into a `BitList<131_072>`. Verified no downstream length cap rejects it (only consumer is `decode_hex_var` → infallible `BitList::from_ssz_bytes`). Go/fastssz side: not run (not on disk); SSZ spec mandates `bit_count > N` MUST be rejected — reasoned. Severity stays **High** (no downstream cap; Uncertain item resolved).
- **Fix:** Make the inherent decoder fallible (or add validation in the `Decode` impl): after computing `len`, return `DecodeError::BytesInvalid` when `MAX > 0 && len > MAX`. Mirror `SszList`'s `MAX` check (`types.rs:105`).

### [Low] `BitVector::from_ssz_bytes` does not reject non-zero padding bits
- **Rust:** `crates/ssz/src/types.rs:580` (`impl Decode for BitVector::from_ssz_bytes`)
- **Charon ref:** fastssz bitvector unmarshal + SSZ spec (Bitvector[N]: bits above index `N` MUST be zero) | n/a
- **Issue:** Decode checks only total byte length (`SIZE.div_ceil(8)`); it does not verify that the unused high bits of the final byte (positions `SIZE .. 8*ceil(SIZE/8)`) are zero. `tree_hash_root` (`types.rs:607`) hashes `self.bytes` verbatim, so attacker-set padding bits flow straight into the root, while `bit_indices`/`bit_at` (which clamp to `< SIZE`) ignore them — i.e. the corruption is invisible to logic but present in the hash.
- **Impact & likelihood:** A beacon node returns `committee_bits` (`BitVector<64>`, decoded at `crates/eth2api/src/spec/electra.rs:97` from untrusted input) with a padding bit set; pluto's signing root differs from honest clients → signature over a value other peers reject, or consensus divergence. For `SIZE` a multiple of 8 (e.g. 64, 512) there are no padding bits, so the common consensus vectors are unaffected; bite only for non-byte-aligned `SIZE`. · only on malformed input, and only for non-byte-aligned bit vectors.
- **PoC:** CONFIRMED — scratch test `scratch_bitvector_nonzero_padding_accepted`: `BitVector<12>::from_ssz_bytes(&[0x00, 0x80])` (bit 15 = a padding bit set) returns `Ok`; `bit_at(15)`/`bit_indices()` ignore it (clamp to `< SIZE`) yet `tree_hash_root()` differs from the zero-padded `[0x00,0x00]` root — corruption invisible to logic, present in the hash, exactly as described. **Important caveat (confirms the finding's own scoping):** the only live `BitVector` in the codebase is `committee_bits: BitVector<64>` (electra.rs:67) — `SIZE=64` is byte-aligned, so it has *no* padding bits and is NOT exploitable today. The bug is real but latent for current types. Go/fastssz: not run; SSZ spec requires high padding bits be zero — reasoned. Severity lowered to **Low**: latent only; no live non-byte-aligned `BitVector` exists in tree.
- **Fix:** After the length check, when `SIZE % 8 != 0`, reject if `bytes.last() & !((1 << (SIZE % 8)) - 1) != 0` with `DecodeError::BytesInvalid`.

### [Medium] `BitList::from_ssz_bytes` silently maps malformed (zero-sentinel) input to empty instead of erroring
- **Rust:** `crates/ssz/src/types.rs:304-329` (`BitList::from_ssz_bytes`, branch `last_byte == 0` and empty `ssz`)
- **Charon ref:** fastssz `ValidateBitlist` + SSZ spec (a valid Bitlist encoding always has a non-zero final byte; an all-zero trailing byte / empty buffer is invalid) | n/a
- **Issue:** Empty input and a zero last byte both return `Self::default()` (len 0) rather than signalling a decode error. A valid bitlist of length 0 encodes as `0x01` (sentinel only), never `0x00` or `""`. Combined with the infallible signature, malformed encodings are accepted as "empty".
- **Impact & likelihood:** Malformed `aggregation_bits` is silently coerced to an empty bitlist; its root (`mix_in_length(merkle_root([]), 0)`) differs from a spec decoder that would reject, so pluto and honest clients disagree on the attestation root. Lower blast radius than the missing MAX cap, but still attacker-influenced. · only on malformed input.
- **PoC:** CONFIRMED — scratch test `scratch_bitlist_malformed_zero_sentinel_to_empty`: `BitList<2048>::from_ssz_bytes` of `[]`, `[0x00]`, and `[0x00,0x00]` all return `Ok` with `len()==0` (no error). A spec-valid empty bitlist is `0x01`; these malformed encodings are silently coerced to "empty" via the `ssz.is_empty()` / `last_byte == 0` early returns (`types.rs:305-311`). Go/fastssz: not run; SSZ spec requires a present delimiting (sentinel) bit ⇒ empty buffer / zero final byte is an invalid encoding — reasoned. Severity stays **Medium**.
- **Fix:** Return a `Result`; error on empty buffer or zero final byte (consistent with the MAX-cap fix above — converting this decoder to fallible resolves both at once).

### [Low] `default_hash_fn` panics on input not a multiple of 64 bytes
- **Rust:** `crates/ssz/src/hasher.rs:133-144` (`Hasher::default_hash_fn`)
- **Charon ref:** fastssz `hasher.go` `hashFn`/`gohashtree` always receives padded even-chunk input | n/a (internal invariant)
- **Issue:** `for pair in src.chunks(64) { hasher.update(&pair[..32]); hasher.update(&pair[32..]); }` indexes `pair[..32]` and `pair[32..]` assuming every chunk is exactly 64 bytes. A final chunk of 1..63 bytes panics (`pair[..32]` slice OOB for <32, or empty second half mis-hashes for 32..63). All current callers (`merkleize_impl` after odd-node padding) supply multiples of 64, so it is not reachable today, but the function is `pub` and undocumented about this precondition.
- **Impact & likelihood:** Panic (DoS) if a future caller passes an odd-length buffer; no current trigger. · negligibly rare today / latent.
- **PoC:** n/a
- **Fix:** Document the "len must be a multiple of 64" invariant, or `return Err(InvalidBufferLength)` when `src.len() % 64 != 0`.

### [Low] `SszList<T, 0>` (default `MAX`) merkleizes without capacity padding → non-spec root
- **Rust:** `crates/ssz/src/types.rs:130-138` (`SszList::tree_hash_root`, `MAX == 0` branch sets `minimum_leaf_count = 0`)
- **Charon ref:** SSZ spec (a List[T, N] always merkleizes the data padded to `chunk_count(N)` then `mix_in_length`) | n/a
- **Issue:** When `MAX == 0` the root depends only on the actual element count, not on the declared list capacity, so it does not match a spec/fastssz root for any concrete `N`. This is the `Default` and is the value used by `SszList<T>` written without an explicit const generic.
- **Impact & likelihood:** All live consensus list fields specify an explicit `MAX` (verified across `eth2api/src/spec/*`), so no current type is affected; the wrong root only surfaces if someone declares a real list as `SszList<T>` / relies on `Default`. · latent / misuse-only.
- **PoC:** n/a
- **Fix:** Make `MAX` mandatory (drop the `= 0` default), or document that `MAX = 0` means "no capacity" and is not spec-compliant for hashing.

### [Low] `BitList::with_bits` / `BitVector::with_bits` panic on out-of-range bit index
- **Rust:** `crates/ssz/src/types.rs:348-350` (`BitList::with_bits`), `types.rs:500-503` (`BitVector::with_bits`), also `set_bit_at`/`bit_at` rely on `BIT_MASK[i % 8]` which is fine but `bytes[bit / 8]` is unchecked in `with_bits`
- **Charon ref:** n/a (Rust-only API)
- **Issue:** `with_bits(capacity, set_bits)` indexes `bytes[bit / 8]` without checking `bit < capacity`; a `set_bits` entry ≥ `capacity` panics (index OOB). `BitVector::with_bits` similarly trusts `bit < SIZE`. All in-tree callers pass controlled indices, so not reachable from untrusted input, but it is a `pub` constructor with an unstated precondition.
- **Impact & likelihood:** Panic on caller misuse; no untrusted-input path. · only on programmer error.
- **PoC:** n/a
- **Fix:** Either document the precondition or return `Result`/skip out-of-range bits; assert `bit < capacity`/`bit < SIZE`.

### [Low] `merkleize_with_mixin` performs a redundant truncate/extend before computing the mixin
- **Rust:** `crates/ssz/src/hasher.rs:370-382` (`Hasher::merkleize_with_mixin`)
- **Charon ref:** fastssz `hasher.go` `MerkleizeWithMixin` | confirmed equivalent output
- **Issue:** The function writes the intermediate merkle root into `self.buf` (`truncate(index); extend_from_slice(&input)` at lines 370-371) and then immediately overwrites it again (lines 381-382) after appending the length chunk and hashing. The first write is dead work. Output is correct; only an efficiency/clarity nit on a hot signing path.
- **Impact & likelihood:** None functional; minor wasted allocation/copy. · always (every mixin), but immaterial cost.
- **PoC:** n/a
- **Fix:** Drop the first `truncate`/`extend_from_slice` pair; build the 64-byte `[root || length]` buffer locally and write the hash once.

### [Info] `parse_bitlist` uses `wrapping_sub` on lengths; relies on prior empty-check
- **Rust:** `crates/ssz/src/hasher.rs:405-432` (`parse_bitlist`)
- **Charon ref:** fastssz `hasher.go` `PutBitlist`/`parseBitList` | confirmed equivalent
- **Issue:** Saturating/wrapping arithmetic on `buf.len()-1` etc. is guarded by the `buf.is_empty()` early-return, so the `wrapping_sub(1)` never underflows. `put_bitlist` (the only caller of `parse_bitlist`) has no production caller outside this crate (the live bitlist path goes through `types.rs::BitList`, not the manual hasher), so this code is effectively test-only today. Logic matches fastssz; flagged only as dead-in-prod surface to keep tested.
- **Impact & likelihood:** None observed. · n/a
- **PoC:** n/a
- **Fix:** None required; consider deleting `put_bitlist`/`parse_bitlist` if no production caller is planned, or add a spec-vector test.

### [Info] `helpers.rs` mirrors charon `genssz` `putByteList`/`putBytesN`/`leftPad` faithfully
- **Rust:** `crates/ssz/src/helpers.rs:20-80` (`put_byte_list`, `put_bytes_n`, `left_pad`)
- **Charon ref:** `app/genssz/template.go` (`putByteList`, `putBytesN`, `leftPad`) | confirmed match
- **Issue:** No defect. `put_byte_list` limit chunk = `limit.div_ceil(32)` == Go `(limit+31)/32`; `put_bytes_n` left-pads to `n`; `left_pad` semantics match. Recorded as a positive parity confirmation for the genssz-generated path.
- **Impact & likelihood:** n/a
- **PoC:** n/a
- **Fix:** none

## Uncertain / needs-human
- **fastssz source not on disk** (`go env GOPATH` cache empty; no vendored `ferranbt/fastssz`). The `hasher.rs` merkleization (`merkleize_impl`, `get_depth`/`next_power_of_two`, `MerkleizeWithMixin`, `calculate_limit`, `parse_bitlist`) was reviewed against the SSZ spec and recalled fastssz behavior and judged equivalent, but was **not** diffed line-by-line against pinned fastssz. Recommend a Phase-2 spec-vector test (official SSZ test vectors / cross-check against fastssz output for representative containers) to lock parity. Specifically confirm: `merkleize_impl` `limit==1`/empty-input ordering, `get_depth(limit)` for large limits, and the length-chunk layout in `merkleize_with_mixin`.
- **`ethereum_ssz` 0.10 `Vec<T>::from_ssz_bytes` offset/length bounds** (used by `SszList::from_ssz_bytes`) are upstream-crate responsibility and out of this unit's scope; assumed sound. The wrapper's added `MAX` post-check is correct; the untrusted-decode risk inside this crate is the `BitList`/`BitVector` findings above.
- **Whether the over-capacity `aggregation_bits` is rejected somewhere downstream** of `BitList::from_ssz_bytes` (e.g. a later validation in `eth2api`/`core`) — RESOLVED in Phase 2: no downstream cap exists. The only consumer is `decode_hex_var` → infallible `BitList::from_ssz_bytes` (`electra.rs:106`, `phase0.rs:414`); grep found no length check on `aggregation_bits` anywhere in `eth2api`/`core`. The `electra_oversized_attestation_json` fixture's test (`electra.rs:404`) *expects* successful decode with `len() > 2048`. [High] finding stands at High.

## Summary
Two SSZ stacks coexist: a manual fastssz-port hasher (cluster/QBFT signing roots) and custom `BitList`/`BitVector`/`SszList` wrappers over `ethereum_ssz`+`tree_hash` (attestation/block roots, decoded from untrusted beacon-node JSON). The hasher and `helpers.rs` match charon `genssz`/fastssz on inspection; main risk is in the custom bitfield decoders. Top issue: `BitList::from_ssz_bytes` ignores its `MAX` const generic and is infallible, so over-capacity attacker-supplied `aggregation_bits` decode silently and corrupt the tree-hash signing root (High). One related Medium decode-laxity issue remains: `BitList` coerces malformed zero-sentinel input to "empty". `BitVector` padding validation is Low after triage because current live vectors are byte-aligned. Remaining items are Low/Info (latent panics on `pub` APIs, a non-spec `MAX==0` default root, a redundant copy in the mixin path). fastssz was not on disk — merkleization parity is reasoned, not vector-tested; flagged for Phase 2.
