# Audit: cluster  (6366 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/cluster/src/*` (definition, lock, manifest/, operator, deposit, distvalidator, eip712sigs, registration, ssz, version)
- **Charon:** `cluster/*.go` (definition, lock, operator, deposit, distvalidator, eip712sigs, registration, ssz, version, helpers)
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `definition.rs`/`lock.rs` ↔ `cluster/{definition,lock}.go` (the canonical config/lock files)
- `manifest/` ↔ legacy-lock → manifest DAG conversion
- `eip712sigs.rs` ↔ `cluster/eip712sigs.go`; `ssz.rs` ↔ `cluster/ssz.go`

## Crate-specific focus
- **Parity (cross-impl-critical):** `definition_hash` / `config_hash` / `lock_hash` MUST match charon byte-for-byte (clusters are shared across charon+pluto operators). EIP-712 typed-data signing, SSZ encoding, operator/creator signatures, deposit data roots, versioned schema.
- **Security:** verify operator/creator signatures on definition+lock, hash domain separation, reject malformed manifests.
- **Quality:** versioned schema handling, serde correctness.

---
## Summary
Core config/lock hashing (v1.3–v1.10), EIP-712 digests, operator/creator signature verification, and versioned serde all closely mirror charon and are backed by passing test vectors. **Phase-2 verification:** one interop-critical hashing divergence is CONFIRMED in the legacy (v1.0–v1.2) path — `hash_lock_legacy` mixes in `definition.validator_addresses.len()` (via `Deref`) instead of `distributed_validators.len()`, producing a divergent `lock_hash` (and `CountGreaterThanLimit` panics) when the two counts differ. The second alleged High (legacy config-hash timestamp operator-precedence) is REFUTED as an observable bug and downgraded to Low: the mis-parenthesization only ever writes an *empty* timestamp, whose all-zero SSZ chunk is absorbed by merkleization zero-padding, so the config hash is byte-identical to charon (verified for 1- and 3-operator definitions). Both Mediums are confirmed by inspection — dropped legacy-input validations (v1.0–v1.5 lock fee-recipient rejection, v1.0/v1.1 non-zero operator nonce) where pluto accepts inputs charon rejects. Revised counts: 0C / 1H / 2M / 4L / 2I.

## Findings   ← audit agent appends findings below this line (schema in README.md)

### [High] hashLockLegacy uses validator_addresses count instead of distributed_validators count for mixin
- **Rust:** `crates/cluster/src/ssz.rs:790` (`hash_lock_legacy`)
- **Charon ref:** [`cluster/ssz.go:714`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/ssz.go#L714) (`hashLockLegacy`)
- **Issue:** Go computes `num := uint64(len(l.Validators))` (the distributed validators), then iterates `l.Validators`, and calls `MerkleizeWithMixin(subIndx, num, num)`. Rust sets `let num = lock.validator_addresses.len();` — `validator_addresses` resolves via `Deref<Target=Definition>` to `definition.validator_addresses` (the fee/withdrawal address list), NOT the distributed validators it iterates over. The mixin length (and the list-length mixed into the merkle root) therefore comes from the wrong field. For well-formed legacy clusters `validator_addresses.len() == distributed_validators.len() == num_validators`, so the bug is masked; when the two differ the legacy `lock_hash` diverges from charon byte-for-byte.
- **Impact & likelihood:** Interop-critical — wrong `lock_hash` for v1.0/v1.1/v1.2 locks silently breaks cluster formation / lock verification across charon↔pluto · latent: only when `definition.validator_addresses.len() != distributed_validators.len()` (malformed or hand-edited legacy lock, or any future caller that builds a Lock with mismatched counts). Always semantically wrong field even when count happens to match.
- **PoC:** CONFIRMED — scratch test `hash_lock_legacy` on a v1.2 Lock with `validator_addresses.len()=3` but `distributed_validators.len()=2` produced `0x509f0577…` vs a charon-faithful reference (mixin = `distributed_validators.len()`) `0x61896210…` → hashes diverge. Wrong field confirmed; `Deref` resolves `lock.validator_addresses` to `definition.validator_addresses`. (Note: also triggers `CountGreaterThanLimit` errors when `validator_addresses.len() < distributed_validators.len()`, since `merkleize_with_mixin(num,num)` uses the wrong count as the limit.)
- **Fix:** Use `let num = lock.distributed_validators.len();` to match charon.

### [Low] Legacy config-hash timestamp gating mis-parenthesized vs charon (functionally inert)
- **Rust:** `crates/cluster/src/ssz.rs:239` (`hash_definition_legacy`)
- **Charon ref:** [`cluster/ssz.go:146-155`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/ssz.go#L146-L155) (`hashDefinitionLegacy`)
- **Issue:** Go: `if configOnly { if d.Timestamp != "" { put } } else { if d.Version != v1_0 { put } }`. Rust condenses to `if config_only && !definition.timestamp.is_empty() || definition.version != V1_0 { put }`. Rust `&&` binds tighter than `||`, so this is `(config_only && !empty) || (version != V1_0)`. Divergence case: `config_only == true` (config hash) AND `timestamp.is_empty()` AND `version` is v1.1/v1.2 → Go puts NOTHING; Rust evaluates `false || true == true` and puts the (empty) timestamp field.
- **Impact & likelihood:** **No observable hash divergence** (see PoC). The diverging case only puts an *empty* timestamp string, which the SSZ hasher encodes as one all-zero 32-byte chunk appended immediately before the final `merkleize(indx)`; that trailing zero chunk is absorbed by the tree's zero-padding, so the config hash is identical whether or not it is written. Pure code-clarity/correctness divergence with zero interop impact; cannot cascade into `definition_hash`/EIP-712 digests. **Severity lowered High → Low** based on PoC.
- **PoC:** REFUTED (as an observable bug) — scratch test: v1.2 def, empty timestamp, `config_only=true`. Pluto config hash `0x5290619a…` == charon-faithful reference that skips the timestamp field entirely `0x5290619a…`. Positive control (non-empty timestamp `0x5394f10f…` ≠ empty) confirms the harness detects real timestamp changes; re-verified with a 3-operator definition (`0x9a9bcd30…` == reference). The empty/zero trailing field is absorbed by merkleization zero-padding. (Go branch reasoned, not run.)
- **Fix:** Mirror Go's branch structure explicitly for clarity/future-proofing: `if config_only { if !timestamp.is_empty() { put } } else if version != V1_0 { put }`. Not interop-blocking.

### [Medium] v1.0–v1.5 lock deserialization drops "distributed validator fee recipient not supported" rejection
- **Rust:** `crates/cluster/src/lock.rs:460` (`From<LockV1x0or1> for Lock`), `:515` (`From<LockV1x2to5> for Lock`); `crates/cluster/src/distvalidator.rs:168,211` (`From<DistValidatorV1x0or1/V1x2to5> for DistValidator`)
- **Charon ref:** [`cluster/lock.go:365-369,387-391`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/lock.go#L365-L369) (`unmarshalLockV1x0or1`, `unmarshalLockV1x2to5`)
- **Issue:** Go rejects any v1.0–v1.5 lock whose distributed validators carry a non-empty `fee_recipient_address` (`"distributed validator fee recipient not supported anymore"`). Rust deserializes `fee_recipient_address` into `DistValidatorV1x0or1`/`V1x2to5`, then the `From … for DistValidator` impls silently discard it with no validation, so such inputs are accepted (and round-trip to a different value).
- **Impact & likelihood:** Pluto accepts legacy locks charon rejects; the dropped field is excluded from the lock hash so verification can still pass, masking malformed/legacy data instead of surfacing it · only on legacy locks that actually carry the deprecated field.
- **PoC:** confirmed by inspection (no PoC needed) — `From<DistValidatorV1x0or1>` (distvalidator.rs:168) and `From<DistValidatorV1x2to5>` (:211) build `DistValidator` from `pub_key`/`pub_shares` only; the deserialized `fee_recipient_address` field is never read (no rejection). Charon `unmarshalLockV1x0or1`/`unmarshalLockV1x2to5` (lock.go:365-369, 387-391) explicitly `return … "distributed validator fee recipient not supported anymore"` when `len(FeeRecipientAddress) > 0`. (Go reasoned, not run.)
- **Fix:** In the v1x0or1 / v1x2to5 `From` conversions (or a dedicated deserialize path) return an error when any `fee_recipient_address` is non-empty, matching charon.

### [Medium] OperatorV1X1 non-zero nonce not rejected on deserialization
- **Rust:** `crates/cluster/src/operator.rs:61` (`From<OperatorV1X1> for Operator`)
- **Charon ref:** [`cluster/operator.go:47-64`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/operator.go#L47-L64) (`operatorsFromV1x1`)
- **Issue:** Go's `operatorsFromV1x1` returns `"non-zero operator nonce not supported"` if any v1.0/v1.1 operator JSON has `nonce != 0`. Rust deserializes `nonce: u64` into `OperatorV1X1` and the `From` impl drops it without checking, accepting non-zero nonces.
- **Impact & likelihood:** Pluto silently accepts malformed legacy operator records charon rejects (validation/parity gap); no hash impact since the SSZ path always writes `ZERO_NONCE` · only on hand-crafted/legacy input with non-zero nonce.
- **PoC:** confirmed by inspection (no PoC needed) — `From<OperatorV1X1> for Operator` (operator.rs:61-69) copies `address`/`enr`/`config_signature`/`enr_signature` and never reads the deserialized `nonce: u64` field, so any value is accepted. Charon `operatorsFromV1x1` (operator.go:47-64) returns `"non-zero operator nonce not supported"` when `o.Nonce != 0`. (Go reasoned, not run.)
- **Fix:** Reject `nonce != 0` in `From<OperatorV1X1>`/deserialization (convert to a fallible path), matching charon.

### [Low] verifyBuilderRegistrations "noRegistration" check omits fee-recipient emptiness test
- **Rust:** `crates/cluster/src/lock.rs:364` (`verify_builder_registrations`)
- **Charon ref:** [`cluster/lock.go:242-244`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/lock.go#L242-L244) (`verifyBuilderRegistrations`)
- **Issue:** Go treats a registration as absent when `len(Signature)==0 || len(FeeRecipient)==0 || len(PubKey)==0`. Rust checks only `signature == EMPTY_SIGNATURE || message.pub_key == EMPTY_VALIDATOR_PUBKEY` (fixed-size all-zero arrays), intentionally dropping the fee-recipient test (per the inline comment, since the zero address is a valid 20-byte fee recipient in Go). Because `fee_recipient`/`pub_key`/`signature` are fixed `[u8;N]` in Rust, a JSON-absent field deserializes to all-zeros rather than Go's len-0, so the trigger condition differs in edge cases (e.g. a registration with valid sig+pubkey but zero fee recipient: Go→absent/skip, Rust→present/verify).
- **Impact & likelihood:** Behavioral divergence only for the unusual case of a partially-populated registration; for fully-present or fully-absent registrations the two agree · negligibly rare in practice.
- **PoC:** n/a
- **Fix:** Document the deliberate divergence, or normalize to Go semantics by also treating a zero/absent fee recipient as "no registration" when sig+pubkey are also zero.

### [Low] Registration timestamp SSZ encoding errors on negative Unix time instead of wrapping
- **Rust:** `crates/cluster/src/ssz.rs:949-953` (`hash_registration`); also `lock.rs:390-395`, `distvalidator.rs:121-126`
- **Charon ref:** [`cluster/ssz.go:858-859`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/ssz.go#L858-L859) (`hashRegistration`)
- **Issue:** Go encodes `hh.PutUint64(uint64(r.Timestamp.Unix()))` — a negative Unix time wraps (two's complement) into a large u64. Rust uses `u64::try_from(timestamp.timestamp())` and returns `FailedToConvertTimestamp` on negative values, so a pre-epoch timestamp errors out where Go produces a (large) hash input.
- **Impact & likelihood:** Different behavior (error vs hash) only for pre-1970 registration timestamps, which never occur for real builder registrations · negligibly rare.
- **PoC:** n/a
- **Fix:** If strict parity is desired, use `as u64` cast (wrapping) to mirror Go; otherwise document the stricter behavior.

### [Low] NewDefinition timestamp format differs from charon
- **Rust:** `crates/cluster/src/definition.rs:446-449` (`Definition::new`)
- **Charon ref:** [`cluster/definition.go:94`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/definition.go#L94) (`NewDefinition`)
- **Issue:** Go uses `time.Now().Format(time.RFC3339)` (local tz, second precision). Rust uses `chrono::Local::now().with_nanosecond(0).to_rfc3339()`. Both are local-tz RFC3339 at second precision, so output format matches; flagged only because the timestamp string feeds directly into the config/definition hash and any divergence here would be interop-fatal. The inline `TODO` already notes this is error-prone.
- **Impact & likelihood:** No divergence found between the two for second-precision local RFC3339; risk is future drift (e.g. tz/offset formatting) · n/a at present.
- **PoC:** n/a
- **Fix:** None required now; consider a single controlled UTC RFC3339 helper to remove the risk (matches the existing TODO).

### [Info] get_operator_eip712_type panics via unreachable! on unsupported version
- **Rust:** `crates/cluster/src/eip712sigs.rs:121` (`get_operator_eip712_type`)
- **Charon ref:** [`cluster/eip712sigs.go:109`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cluster/eip712sigs.go#L109) (`getOperatorEIP712Type`)
- **Issue:** Both panic ("this should never happen") when called for a non-EIP712 version. Rust uses `unreachable!`. Callers (`verify_signatures`, `sign_operator`) only reach it after gating on `support_eip712_sigs`, so the invariant holds — but it is a panic on a public-ish path rather than a typed error.
- **Impact & likelihood:** Matches charon; only reachable via a future caller that bypasses the version gate · effectively never.
- **PoC:** n/a
- **Fix:** Optional: return `Result`/typed error instead of panicking to harden against future misuse.

### [Info] manifest/* modules retained though removed upstream
- **Rust:** `crates/cluster/src/manifest/mod.rs:1-32`
- **Charon ref:** n/a (charon PR #4130 removed manifest; not in v1.7.1 production path)
- **Issue:** `mod.rs` documents that load/materialise/mutation/mutationaddvalidator/mutationlegacylock/mutationnodeapproval/types are "no longer required" yet they remain compiled. Dead/legacy surface not exercised by the interop-critical config/lock hashing path.
- **Impact & likelihood:** Maintenance/clarity only; extra attack/parity surface with no production use · n/a.
- **PoC:** n/a
- **Fix:** Remove the obsolete manifest modules (or gate behind a feature) if they are truly unused.

## Uncertain / needs-human
- **`verify_aggregate` with empty pubkeys (lock.rs:285-294):** for a v1.x lock with zero distributed validators but a non-empty `signature_aggregate`, `pubkeys` is empty. Go's `tbls.VerifyAggregate` vs pluto's `BlstImpl.verify_aggregate` behavior on an empty pubkey set is library-dependent (may error, may treat as identity). Not validated; needs checking against the blst wrapper. Likely benign (zero-validator locks are degenerate) but unverified.
- **`DepositData.amount` JSON parity (deposit.rs:27):** Go marshals `Amount int` with `json:"amount,string"` (string-encoded). Rust uses `DisplayFromStr` (string) for deserialize/serialize. Matches for normal values; behavior for `amount` exceeding i64 range (Go `int`) vs Rust `u64` not verified — low risk, flagged for completeness.
- **Manifest hashing parity not audited:** per the work order's emphasis on definition/lock/ssz and mod.rs marking manifest as upstream-removed, the `manifest/*` DAG conversion + protobuf hashing was not compared against charon. If pluto still relies on it anywhere, a separate parity pass is warranted.
