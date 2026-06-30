# Audit: testutil  (7939 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/testutil/src/*` (beaconmock/, validatormock/, random, lib)
- **Charon:** `testutil/*` (beaconmock, validatormock, ...)
- **Tier C** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `beaconmock/` ↔ `testutil/beaconmock` (mock beacon node driving integration tests)
- `validatormock/` ↔ `testutil/validatormock`; `random.rs` ↔ deterministic test data

## Crate-specific focus
- **Parity:** beaconmock endpoint fidelity (integration-test correctness depends on it), validatormock duty behavior, deterministic seed reproducibility.
- **Security:** low (test-only) — confirm no insecure/test RNG path leaks into production crates.
- **Quality:** ensure test-only (not compiled into release), fixture maintainability.

## Summary
Strong, faithful port: deterministic duty assignment (attester/proposer/sync), attestation store + epoch-0 wrap, meta slot/epoch arithmetic, and the async primitives (`CloseOnce`, `FakeClock`, scheduler) all match charon. Highest-value issue is `random_bit_list`: it emits a 32-byte array with no SSZ Bitlist length sentinel, where charon's `NewBitlist(256)` yields a valid 33-byte bitlist — a fixture-shape divergence in attestation/fuzz flows. Remaining items are low/informational: a `seed=255` u8-wrap that defeats the zero-seed guard, an inherent Go↔Rust secp256k1 seed→key divergence (no current golden depends on it), and a couple of stale doc comments. No security exposure: insecure RNG/key helpers are confined to test consumers and not reachable from production crates. STATUS: VERIFIED.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Low] random_bit_list: 32-byte output missing SSZ Bitlist length sentinel (wrong shape vs charon)
- **Rust:** `crates/testutil/src/random.rs:135` (`random_bit_list`)
- **Charon ref:** [`testutil/random.go:1405`](https://github.com/ObolNetwork/charon/blob/v1.7.1/testutil/random.go#L1405) (`RandomBitList`)
- **Issue:** Charon builds `bitfield.NewBitlist(256)` then sets `length` random bits; a `Bitlist(256)` marshals to **33 bytes** (256 data bits + the terminating length sentinel bit at position 256 → ceil(257/8)=33) and is a valid SSZ Bitlist. Pluto fills a fixed `[u8; 32]` array (256 bits, **no** sentinel) → emits 32 bytes / `0x`+64 hex. Result: different byte length and an SSZ-invalid bitlist (a Bitlist needs a high-order 1 bit to encode its length). Used by `random_phase0_attestation` (`random.rs:183`) and the fuzzer (`fuzzer.rs:330` `random_bit_list(0)` → all-zero, no sentinel at all).
- **Impact & likelihood:** Any consumer that decodes `aggregation_bits`/`sync_committee_bits` as an SSZ Bitlist sees the wrong length / rejects it; byte-for-byte golden comparison against a charon-shaped attestation diverges. Test fixtures only (Tier C), but weakens attestation-shape fidelity in fuzz/random flows · always (every call) · `random_bit_list(0)` yields a degenerate zero-length-marker bitlist.
- **PoC:** CONFIRMED — scratch `random::tests::scratch_bitlist_shape` (asserting charon-shaped 33-byte/sentinel) → Rust emits **32 bytes** for all `length` (hex_len=66 incl `0x`); `length=0` is all-zeros (no sentinel byte at all). No 33rd byte, no length-sentinel bit. Go side reasoned (Go not run): `bitfield.NewBitlist(256)` allocates `256/8 + 1 = 33` bytes and sets the length-marker bit at index 256 (byte 32, bit 0), untouchable by `SetBitAt` over `[0,256)` → always 33 bytes. Confirms shape divergence (32 vs 33 bytes, missing SSZ length sentinel). Severity lowered to Low: fixtures/fuzz helpers only, no production reachability.
- **Fix:** Allocate 33 bytes, set the length sentinel bit at index 256 (`bytes[32] |= 1`), and place the `length` random data bits in indices 0..256, matching `bitfield.NewBitlist(256)` semantics; update the `len == 66` test assertions accordingly.

### [Low] generate_insecure_k1_key: seed=255 wraps to 0, re-introducing the zero-seed path the +1 was meant to avoid
- **Rust:** `crates/testutil/src/random.rs:53` (`generate_insecure_k1_key`)
- **Charon ref:** [`testutil/random.go:1684`](https://github.com/ObolNetwork/charon/blob/v1.7.1/testutil/random.go#L1684) (`GenerateInsecureK1Key`)
- **Issue:** Go does `constReader(seed+1)` on an `int` (seed 255 → reader byte 256 truncated to 0 in Go's `byte(c)`... actually Go `constReader` is `byte`, so `constReader(256)` = 0 too — but Go's seed domain is `int` and tests never pass 255). Rust uses `seed.wrapping_add(1)` on a `u8`: seed `255` → `ConstReader(0)` → an all-zero RNG. `SecretKey::random` over an all-zero reader reduces to the zero scalar (invalid), which k256 rejects/retries — re-creating exactly the infinite-loop/panic risk the `+1` comment says it is avoiding. The Go comment "Add 1 to seed to avoid passing 0 … infinite loop" is defeated for `seed=255` in the u8 port.
- **Impact & likelihood:** Panic or hang only if a caller passes `seed = 255`. All current callers use small fixed seeds (0–99), so unreached today · negligibly rare in practice but a latent footgun.
- **PoC:** n/a
- **Fix:** Widen seed handling or special-case the wrap (e.g. `if seed == u8::MAX { ... }`), or document the forbidden value; simplest is to compute the reader byte in a wider type and never let it reach 0.

### [Low] generate_insecure_k1_key produces different key bytes than charon for the same seed (cross-impl divergence)
- **Rust:** `crates/testutil/src/random.rs:53` (`generate_insecure_k1_key`, via `SecretKey::random`)
- **Charon ref:** [`testutil/random.go:1684`](https://github.com/ObolNetwork/charon/blob/v1.7.1/testutil/random.go#L1684) (`GenerateInsecureK1Key`, via `ecdsa.GenerateKey`)
- **Issue:** Go derives the key with `ecdsa.GenerateKey(k1.S256(), constReader(seed+1))` then `PrivKeyFromBytes(k.D.Bytes())`; k256's `SecretKey::random(ConstReader)` uses a different field-element draw/reduction. For the same seed the two implementations yield **different** secret keys (hence different peer IDs / ENRs / cluster hashes).
- **Impact & likelihood:** All current Rust consumers (`crates/priority`, `crates/dkg`, `crates/infosync`) only need determinism+distinctness *within* Rust and never compare against a Go-generated golden key, so no parity break today. Becomes a real interop break only if a golden fixture or cross-client test encodes a specific Go-seeded key · would-trigger only in that scenario.
- **PoC:** n/a
- **Fix:** If cross-impl seed→key parity is ever required, replicate Go's `ecdsa.GenerateKey` rejection-sampling draw exactly; otherwise document that seeds are not portable across the Go/Rust implementations.

### [Info] Register: charon switches on a zero-valued struct field (Go bug); Rust switches on the input version
- **Rust:** `crates/testutil/src/validatormock/propose.rs:183` (`register`)
- **Charon ref:** [`testutil/validatormock/propose.go:256-265`](https://github.com/ObolNetwork/charon/blob/v1.7.1/testutil/validatormock/propose.go#L256-L265) (`Register`)
- **Issue:** Go writes `signedRegistration := new(...)` then `switch signedRegistration.Version` — the freshly-`new`'d struct has the zero Version (== `BuilderVersionV1`), so Go **always** takes the V1 branch regardless of the input `registration.Version`. Rust switches on the *input* `registration.version`, returning `UnsupportedVariant` for a non-V1 input. Rust is arguably more correct, but diverges from Go for a non-V1 input registration.
- **Impact & likelihood:** Only the V1 path is meaningful in either impl and inputs are V1 in practice, so behavior matches for all realistic inputs · negligible.
- **PoC:** n/a
- **Fix:** None needed; optionally note the deliberate divergence in the comment.

### [Info] meta::slot_start_time clamps slot offset at u32::MAX before multiplying by slot_duration
- **Rust:** `crates/testutil/src/validatormock/meta.rs:26` (`slot_start_time`)
- **Charon ref:** [`testutil/validatormock/meta.go:14`](https://github.com/ObolNetwork/charon/blob/v1.7.1/testutil/validatormock/meta.go#L14) (`SlotStartTime`)
- **Issue:** Rust does `u32::try_from(slot).unwrap_or(u32::MAX)` then `slot_duration.saturating_mul(u32)`. For `slot > u32::MAX` (~4.29e9) the offset is clamped, diverging from Go's `time.Duration(slot) * SlotDuration` (which itself overflows int64 nanos at large slots). Both are degenerate beyond any real chain age.
- **Impact & likelihood:** Unreachable for any realistic slot in tests · negligible.
- **PoC:** n/a
- **Fix:** None required; could widen to `u64`/`u128` math for exactness if desired.

### [Info] attest::prepare doc comment describes a stale `set`-returns-Err idempotence mechanism
- **Rust:** `crates/testutil/src/validatormock/attest.rs:163-167` (`prepare`)
- **Charon ref:** [`testutil/validatormock/attest.go:77`](https://github.com/ObolNetwork/charon/blob/v1.7.1/testutil/validatormock/attest.go#L77) (`Prepare`) | n/a (Rust doc-only)
- **Issue:** Comment says calling twice makes "the `set` calls on the close-once cells return `Err`, which we silently swallow." The current code uses `CloseOnce::close()` (`attest.rs:228/236/244`), which is idempotent no-op (`close_once.rs:35`) — there is no `set`/`Err` swallowing. Documentation mechanics out of date; behavior is fine.
- **Impact & likelihood:** None (comment only) · n/a.
- **PoC:** n/a
- **Fix:** Update the comment to reference `CloseOnce::close`'s idempotent semantics.

---
**Verified as NON-issues (Explore-agent claims refuted by reading Go):**
- `component.rs:375` BuilderRegistration — Go's `dutyStartTimeFuncsByDuty` (component.go:428) **does** schedule `DutyBuilderRegistration`, and `runDuty`'s `default:` arm (component.go:307) returns "unexpected duty". Rust schedules it and errors identically; duty errors are logged via `warn!` (`component.rs:309`), matching Go's non-fatal logging. Parity correct.
- `synccomm.rs:531` double divide-by-zero check — the `if divisor == 0` branch is reachable when `comm_size==0 && subnet_count>0`; defensive, not dead. Matches Go's `commSize/subnetCount` semantics.
- `attest.rs:675` `response.text().await.unwrap_or_default()` — still surfaces `SubmitStatus { endpoint, status, body }`; only the (rare) body-read failure loses the body text, not the error itself. Transparent enough.
- Deterministic attester/proposer/sync-committee duties (`defaults.rs:280-478`) match `options.go:298-481` exactly: slot-offset `(i*factor)%slotsPerEpoch`, `committee_length`/`valCommIndex`, active-only proposer filter, `slotsAssigned` break, `factor==0` break, `epoch%k>=n` sync window, `idx` sync-committee index.
- Attestation store (`attestation.rs`) prune predicate `old.slot+32 >= data.slot` matches Go's `old.Slot+cleanAfter < data.Slot` deletion (negation), epoch-0 `previous_epoch` wrap to `u64::MAX` matches `newAttestationData`.
- `getSubcommittees`, `meta` slot/epoch arithmetic (`SlotsForLookAhead/Back`, `InSlot`, `FirstInEpoch`), `active_validators` filter — all faithful ports.

## Uncertain / needs-human
- **random_bit_list exact charon byte length (33 vs 32):** RESOLVED in Phase 2. Rust side executed: 32 bytes confirmed (scratch test). Go side reasoned (not executed): `bitfield.NewBitlist(256)` = `n/8+1 = 33` bytes, sentinel bit at index 256, per `prysmaticlabs/go-bitfield` semantics (lib not present locally). Divergence stands, but severity lowered to Low because this is fixture-only. Still open for a human: whether charon's HTTP/JSON consumers tolerate the 32-byte form (would justify Low/Info), and a Go-executed `len(RandomBitList(1).Bytes())==33` cross-check.
- **headproducer pseudo-random roots:** Rust deliberately uses ChaCha `StdRng::seed_from_u64(slot)` instead of Go's `math/rand` LCG (documented at `headproducer.rs:214-219`); root bytes differ from charon. Accepted because no test asserts specific head/state/dependent-root values — confirm no downstream golden depends on these exact roots.
