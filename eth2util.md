# Audit: eth2util  (6170 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/eth2util/src/*` (deposit/, eip712, enr, signing, keystore/, keymanager, registration, network, rlp, eth2exp, types, helpers)
- **Charon:** `eth2util/*.go` + subpkgs (deposit, enr, keystore, signing, eip712, registration, ...)
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `signing.rs` ↔ `eth2util/signing` (compute_domain, signing_root)
- `deposit/` ↔ `eth2util/deposit`; `enr.rs`/`rlp.rs` ↔ `eth2util/enr` (RLP encode)
- `keystore/` ↔ `eth2util/keystore` (EIP-2335); `network.rs` ↔ `eth2util/network.go`

## Crate-specific focus
- **Parity:** signing domains + roots (`compute_domain`, `compute_signing_root`), deposit message/data roots, ENR RLP encode/decode + sig, EIP-2335 keystore encrypt/decrypt, keymanager API shapes, network configs (fork versions, genesis validators root).
- **Security:** keystore KDF params (scrypt/pbkdf2 cost), ENR signature verification, deposit withdrawal-credential construction.
- **Quality:** centralize domain/fork constants; avoid magic numbers.

---
## Summary

Interop-critical paths are sound: `compute_signing_root`, domain computation (28-byte fork-data-root truncation), EIP-7044 Capella-for-voluntary-exit routing, deposit/registration signing roots, and the EIP-2335 keystore (PBKDF2+scrypt, AES-128-CTR, `SHA256(DK[16:32]||cipher)` checksum, NFKD normalization) all pass charon/spec vectors and match byte-for-byte. The most material divergences are non-crypto: `Network::is_non_zero` uses an OR-of-fields (`!= default`) where charon ANDs four specific fields, and RLP `from_big_endian` has an off-by-one bounds check (`>` vs Go's `>=`) that accepts edge-length blobs charon rejects — an ENR/RLP interop inconsistency. Remaining items are Low/Info (stderr warnings vs `tracing`, nil-vs-empty + saturating-mul in `eths_to_gweis`, stricter WC-length validation). Three items parked as needs-human (SignedEpoch JSON string-vs-number, `epoch_from_slot` zero-divisor, ENR compressed-pubkey byte form) are reasoned-OK but unexecuted. Net: 0C / 0H / 2M / 3L / 3I.

## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Medium] `Network::is_non_zero` truth table diverges from charon `IsNonZero`
- **Rust:** `crates/eth2util/src/network.rs:62` (`Network::is_non_zero`)
- **Charon ref:** [`eth2util/network.go:33`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/network.go#L33) (`Network.IsNonZero`)
- **Issue:** Rust returns `self != &Network::default()` → true if **any** field differs from its zero value. Charon requires **all** of `Name != "" && ChainID != 0 && GenesisTimestamp != 0 && GenesisForkVersionHex != ""` (an AND, ignoring `CapellaHardFork`). A `Network` with only `name` set (other fields zero) returns `true` in Rust but `false` in Go; conversely a fully-populated network with `CapellaHardFork == ""` returns `true` in both (Go ignores that field, Rust would also be non-default). The mismatch surfaces for partially-populated networks (e.g. a test/custom network missing `chain_id` or `genesis_timestamp`): Go treats it as "zero/invalid", Rust treats it as valid.
- **Impact & likelihood:** Validation/guard checks relying on `is_non_zero` accept partially-constructed networks Go would reject (or vice-versa) · only when callers build a `Network` with a strict subset of fields populated — uncommon in production, plausible in custom/test-network flows.
- **PoC:** CONFIRMED — scratch `scratch_is_non_zero_divergence` (reasoned Go from network.go:34) → name-only `Network{name:"custom"}`: Rust `is_non_zero()=true`, charon AND-of-four=`false`; capella-only `Network{capella_hard_fork:"0x03000000"}`: Rust=`true`, charon=`false` (Go ignores `CapellaHardFork`). Both truth-table divergences observed at the exact rows the finding predicts. Charon `Network.IsNonZero` AND-of-four-fields confirmed verbatim at [`eth2util/network.go:34`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/network.go#L34).
- **Fix:** Mirror Go exactly: `!self.name.is_empty() && self.chain_id != 0 && self.genesis_timestamp != 0 && !self.genesis_fork_version_hex.is_empty()`.

### [Medium] RLP `from_big_endian` bounds check accepts inputs charon rejects (off-by-one)
- **Rust:** `crates/eth2util/src/rlp.rs:148` (`from_big_endian`)
- **Charon ref:** [`eth2util/rlp/rlp.go:170`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/rlp/rlp.go#L170) (`fromBigEndian`)
- **Issue:** Charon rejects when `offset+length >= len(b)` (strict `>=`), Rust rejects only when `offset.wrapping_add(length) > bytes.len()` (`>`). For a long-form RLP prefix whose declared length-bytes occupy the slice exactly up to its end (`offset+length == len(b)`), Go errors `input too short` while Rust proceeds to read those bytes and returns a length. The two implementations therefore disagree on which malformed/edge ENR-or-RLP blobs are valid. (Note: Go's check is itself arguably too strict, but parity is the contract here. The Rust call sites then re-validate total bounds in `decode_length`/`decode_bytes_list`, so this is a divergence in *which error/acceptance* occurs, not necessarily memory unsafety.)
- **Impact & likelihood:** A crafted ENR/RLP string is accepted by pluto but rejected by charon → interop inconsistency in ENR handling · only on adversarial/edge-length RLP input. (Strengthened from finding's "divergence in which error/acceptance occurs" hedge: confirmed to be a genuine accept-vs-reject at the *public* API, not merely an internal error-path difference.)
- **PoC:** CONFIRMED — scratch `from_big_endian([0xb8,0x05],1,1)=Ok(5)` (Rust accepts the exact `offset+length==len` boundary). Surfaces at the public API: blob `[0xb8,0x00]` (long-form prefix declaring 0-length payload, length-byte is final byte) → Rust `decode_bytes=Ok([])` / `decode_bytes_list=Ok([])`; Go `DecodeBytes`: `fromBigEndian(item,1,1)` hits `offset+length=2 >= len=2` (rlp.go:171) → "input too short" → **rejects**. Also `[0xb9,0x00,0x00]` → Rust `Ok([])`, Go rejects (`1+2=3 >= 3`). Charon strict `>=` confirmed verbatim at [`eth2util/rlp/rlp.go:171`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/rlp/rlp.go#L171) (path is `eth2util/rlp/rlp.go`, not [`eth2util/rlp/rlp.go:170`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/rlp/rlp.go#L170) exactly — fn `fromBigEndian` at line 170, check at 171). Go side reasoned (not run).
- **Fix:** Match Go: reject when `offset >= bytes.len() || offset.wrapping_add(length) >= bytes.len()`.

### [Low] `eths_to_gweis` empty-input and overflow semantics differ from charon
- **Rust:** `crates/eth2util/src/deposit/mod.rs:141` (`eths_to_gweis`)
- **Charon ref:** [`eth2util/deposit/deposit.go:268`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/deposit/deposit.go#L268) (`EthsToGweis`)
- **Issue:** (1) Go returns `nil` for a `nil` slice; Rust returns an empty `Vec` (observable only if a caller distinguishes nil vs empty — Go JSON would differ). (2) Go computes `OneEthInGwei * ethAmount` on Go `int` (silent two's-complement wrap on 64-bit); Rust uses `ONE_ETH_IN_GWEI.saturating_mul(eth)` on `u64` (saturates at `u64::MAX`). Inputs large enough to overflow (>~1.8e10 ETH) yield different values. Also Go takes `[]int` (can be negative) while Rust takes `&[u64]`.
- **Impact & likelihood:** Divergent deposit-amount conversion for absurd inputs; nil-vs-empty difference is cosmetic · negligibly rare (requires unrealistic ETH amounts; partial-deposit amounts are validated elsewhere).
- **PoC:** n/a
- **Fix:** Acceptable as-is, but for strict parity document the saturating choice; inputs are bounded by `verify_deposit_amounts` upstream.

### [Low] Keystore weak-parameter warnings printed to stderr instead of structured logging
- **Rust:** `crates/eth2util/src/keystore/keystorev4.rs:298,338,389,414,419` (`validate_aes_iv`, `validate_parameters`, `validate_salt`)
- **Charon ref:** n/a (charon delegates KDF to `wealdtech/go-eth2-wallet-encryptor-keystorev4`; warnings differ)
- **Issue:** Weak-KDF / odd-IV / odd-salt conditions are surfaced with `eprintln!` rather than the crate-wide `tracing` facility used elsewhere (e.g. `helpers.rs` uses `tracing::info!`). Bypasses log levels, filtering, and structured fields; pollutes stderr in library contexts.
- **Impact & likelihood:** Cosmetic/operational only — warnings are unfilterable and inconsistent with the codebase's observability story · always (whenever weak params are decrypted).
- **PoC:** n/a
- **Fix:** Replace `eprintln!` with `tracing::warn!`.

### [Low] `read_deposit_data_files` validates withdrawal-credentials length; charon does not
- **Rust:** `crates/eth2util/src/deposit/mod.rs:279` (`read_deposit_data_files`)
- **Charon ref:** [`eth2util/deposit/deposit.go:416`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/deposit/deposit.go#L416) (`ReadDepositDataFiles`)
- **Issue:** Go decodes `withdrawal_credentials` into a variable-length `[]byte` and performs **no** length check (only pubkey and signature lengths are validated). Rust parses into a fixed `WithdrawalCredentials = [u8;32]` and errors `InvalidDataLength` if not exactly 32 bytes. Rust is stricter: a malformed file with a short/long WC that Go would load (and surface later as a hash/verify failure) is rejected at read time by Rust.
- **Impact & likelihood:** Different error point/behavior on malformed deposit-data files; Rust is arguably safer but not byte-for-byte equivalent · only on hand-edited/corrupt deposit-data JSON.
- **PoC:** n/a
- **Fix:** No change recommended (stricter is fine); note the intentional divergence if strict parity is required.

### [Info] `verify_address` accepts any-case hex (no EIP-55 checksum validation) — matches charon
- **Rust:** `crates/eth2util/src/helpers.rs:99` (`verify_address`) used by `deposit::withdrawal_creds_from_addr` and `registration::execution_address_from_str`
- **Charon ref:** [`eth2util/helpers.go:77`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/helpers.go#L77) (`ChecksumAddress`)
- **Issue:** Investigated as a potential interop break: charon's `ChecksumAddress` only checks `0x` prefix + exact 42-char length + hex-decodability (it re-checksums, never validates an incoming checksum). Rust's `verify_address` uses alloy `str::parse::<Address>()` which derives `FromStr` from `derive_more` on `FixedBytes` — confirmed (alloy-primitives 1.6.0, `src/bits/macros.rs:62`, `src/bits/address.rs`) to be a plain hex decode that does **not** validate the EIP-55 checksum (that is `Address::parse_checksummed`, which is not used here). Both therefore accept mixed/any-case input. No divergence.
- **Impact & likelihood:** None — behaviors match.
- **PoC:** n/a
- **Fix:** None. Recorded so a later pass does not re-flag it.

### [Info] Self-contained EIP-2335 keystore re-implementation passes spec vectors; file perms match charon
- **Rust:** `crates/eth2util/src/keystore/keystorev4.rs`, `store.rs:222` (`write_file`)
- **Charon ref:** [`eth2util/keystore/keystore.go:107,213`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/keystore/keystore.go#L107) (`storeKeysInternal`, `storePassword`)
- **Issue:** Pluto re-implements EIP-2335 (PBKDF2 encrypt; PBKDF2+scrypt decrypt) instead of wrapping `wealdtech` like charon. Verified: AES-128-CTR, `SHA256(DK[16:32] || cipher)` checksum, NFKD+control-strip password normalization, and both EIP-2335 PBKDF2/scrypt test vectors are exercised in-file. Keystore files written `0o444` and password files `0o400` via `OpenOptions().mode()` — matches charon's `0o444`/`0o400`. Private-key material wrapped in `Zeroizing`.
- **Impact & likelihood:** None — functionally equivalent and spec-correct.
- **PoC:** n/a
- **Fix:** None.

### [Info] `compute_domain` (interop-critical) lives in `eth2api`, not `eth2util`; truncation correct
- **Rust:** `crates/eth2api/src/extensions.rs:161` (`compute_domain`), `:227` (`resolve_domain`)
- **Charon ref:** [`eth2util/helper_capella.go:71`](https://github.com/ObolNetwork/charon/blob/v1.7.1/eth2util/helper_capella.go#L71) (`ComputeDomain`), signing domain in `eth2wrap`
- **Issue:** charon's `helper_capella.go` (`CapellaFork`, `ComputeDomain`, `CapellaDomain`) has no counterpart inside pluto's `eth2util`; the domain computation is instead in `eth2api`. Verified correct: `compute_domain` hashes `ForkData{current_version, genesis_validators_root}` and copies `domain_type || fork_data_root[..28]` into the 32-byte domain (matching Go's `fdt[:28]`), and `resolve_domain` applies EIP-7044 (voluntary exit → Capella fork version). `signing.rs::get_domain` routes `ApplicationBuilder` → genesis domain and others → `fetch_domain(_, epoch)`, matching charon's `GetDomain`. Signing-root vectors pass in `signing.rs`/`registration.rs`/`deposit/types.rs`.
- **Impact & likelihood:** None — module placement differs but signing-root/domain output is byte-identical to charon for the tested vectors.
- **PoC:** n/a
- **Fix:** None (out-of-crate placement is a layout choice; flagged for the eth2api audit's awareness).

## Uncertain / needs-human

- **`SignedEpoch` JSON: epoch as string vs number.** Rust (`types.rs:11`) serializes `epoch` via `serde_as(DisplayFromStr)` → `"42"` (quoted string). Charon's `signedEpochJSON` (`types.go:296`) uses `eth2p0.Epoch` directly. go-eth2-client's phase0 scalar types (Epoch/Slot/…) have custom `MarshalJSON` that emit **quoted strings**, so both should produce `{"epoch":"42",...}` — i.e. parity holds. Not flagged as a finding, but go-eth2-client source is not vendored locally, so the string-encoding assumption was reasoned, not executed. Confirm against go-eth2-client `spec/phase0/epoch.go` if a wire-format guarantee is needed.
- **`epoch_from_slot` zero-divisor path.** Rust (`helpers.rs:160`) uses `fetch_slots_config()` which already errors on `slots_per_epoch == 0` (`eth2api/extensions.rs:311`), then `slot.checked_div(slots_per_epoch)`. Charon (`helpers.go:62`) reads `SLOTS_PER_EPOCH` from spec and does `slot / slotsPerEpoch` with no zero guard (would panic on a malformed spec advertising `SLOTS_PER_EPOCH: 0`). Rust is safer; behavior on a zero-valued spec differs (Rust → typed error, Go → panic). Confirm this is acceptable rather than a parity requirement.
- **ENR pubkey serialization form.** Rust `Record::new` (`enr.rs:144`) stores `secret_key.public_key().to_sec1_bytes()`; charon uses `SerializeCompressed()`. `to_sec1_bytes()` on a k256 `PublicKey` defaults to compressed (33 bytes), matching Go, and the `new` test asserts a fixed ENR vector — appears correct, but worth a Phase-2 byte-equality check against a charon-generated ENR to be certain the compressed form and key ordering match.
