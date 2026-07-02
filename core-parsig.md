# Audit: core-parsig  (4393 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/core/src/{parsigdb,parsigex_codec.rs,sigagg.rs,aggsigdb,bcast}`
- **Charon:** `core/parsigdb/{memory,metrics}.go`, `core/sigagg/sigagg.go`, `core/aggsigdb/{memory,memory_v2}.go`, `core/bcast/{bcast,recast,metrics}.go`
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Summary
Core-parsig is a faithful, well-structured port. No memory-safety, crypto-correctness, or panic-on-untrusted-input issues found; arithmetic is checked throughout (bcast delay/slot math is strictly safer than Go's unchecked `time.Duration`). The main parity risks are two equality-semantics divergences (parsigdb dedup + aggsigdb mismatch use structural `PartialEq` where charon uses JSON-bytes equality), an incomplete aggsigdb v1/v2 two-impl parity, and a sigagg template-scan whose loop semantics + an inaccurate comment diverge from Go (functionally inert under consensus invariants). Most findings hinge on serde/consensus invariants verifiable only outside this unit — see Uncertain.

## Module map (pluto ↔ charon)
- `parsigdb` ↔ `core/parsigdb/*.go` (partial-sig store, threshold detection)
- `sigagg.rs` ↔ `core/sigagg/sigagg.go` (BLS aggregation)
- `aggsigdb` ↔ `core/aggsigdb/{memory,memory_v2}.go`
- `bcast` ↔ `core/bcast/{bcast,recast,metrics}.go`
- `parsigex_codec.rs` ↔ wire codec for parsigex

## Crate-specific focus
- **Parity:** threshold BLS aggregation correctness, partial-sig dedup + threshold trigger, aggsigdb v2 semantics, recast logic, broadcast retry/backoff.
- **Security:** verify each partial sig before aggregation; reject duplicate/equivocating partials; memory bounds on dbs.
- **Quality:** error propagation on agg failure; avoid needless sig-bytes clones.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Low] sigagg: validator-index template scan stops at first non-attestation parSig
- **Rust:** `crates/core/src/sigagg.rs:217-227` (`Aggregator::aggregate_one`)
- **Charon ref:** [`core/sigagg/sigagg.go:120-135`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/sigagg/sigagg.go#L120-L135) (`Aggregator.aggregate`)
- **Issue:** Both pick a template parSig that carries a `validator_index` (VC-supplied), falling back to `parSigs[0]`. Go's loop `break`s out of the scan on the FIRST parSig that is not a `VersionedAttestation` (`if !ok { break }`), so for non-attestation duties it stops immediately and uses `parSigs[0]`. Rust uses `find_map` over ALL parSigs and additionally `att.0.validator_index?` short-circuits the closure with `None` (the comment "return an error" is wrong — `?` on `Option` returns `None`, continuing the search), so behaviour matches for the attestation case. BUT the divergence is in ordering/short-circuit: Go breaks on the first non-attestation entry (effectively only inspects up to the first non-att), whereas Rust scans the whole vec. For a homogeneous attestation set the result is identical; for a mixed/garbage set the chosen template can differ. More importantly, `par_sigs[0]` indexing on line 227 is reached only when `find_map` yields `None`; `par_sigs` is guaranteed non-empty by the threshold check at line 177 (`len < threshold` returns early, threshold ≥ 1), so the index is safe. Net: functional divergence is confined to mixed-type sets, which consensus rules out, but the inverted-from-Go scan semantics plus the misleading comment are a latent correctness/maintenance hazard worth flagging.
- **Impact & likelihood:** Wrong template selection only if a validator's parSig set mixes `VersionedAttestation` with other types within one pubkey — ruled out by consensus producing one unsigned payload per duty · negligibly rare in practice; comment is actively misleading to future maintainers · always for code-reading correctness.
- **PoC:** CONFIRMED — confirmed by inspection (reasoned, Go not run). Verified verbatim: Go (sigagg.go:121-135) `break`s the loop on the first non-`VersionedAttestation` (`if !ok { break }`), so it inspects only up to the first non-att entry, then falls back to `parSigs[0]`. Rust (sigagg.rs:217-227) uses `find_map` over the WHOLE vec, and `att.0.validator_index?` short-circuits the *closure* with `None` (so it continues scanning, NOT "return an error" as the line-224 comment claims). For a homogeneous attestation set both pick the same template. The `par_sigs[0]` index (line 227) is provably safe (threshold check at :177 guarantees non-empty). Severity DOWNGRADED High→Low: the only functional divergence (template choice on a mixed-type set) is ruled out by consensus (one unsigned payload per duty per pubkey); residual issue is the misleading comment + inverted scan semantics = comment/quality fix only. This matches the Uncertain note's stated downgrade condition.
- **Fix:** Match Go: iterate `par_sigs`, `break` on first downcast failure, only adopt entries whose `validator_index.is_some()`. Fix the comment on line 224 — `?` yields `None` and continues, it does not "return an error".

### [Low] parsigdb dedup uses structural PartialEq, not JSON-bytes equality
- **Rust:** `crates/core/src/parsigdb/memory.rs:339-348` (`MemDB::store`)
- **Charon ref:** [`core/parsigdb/memory.go:149-161,232-244`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/parsigdb/memory.go#L149-L161) (`store`, `parSignedDataEqual`)
- **Issue:** Go decides duplicate-vs-equivocation by `json.Marshal(x)`/`json.Marshal(y)` + `bytes.Equal`. Rust compares `s == &value` where `ParSignedData: PartialEq` delegates to `share_idx ==` and `signed_data ==` (`dyn_eq::DynEq`, i.e. each concrete type's derived structural `PartialEq`). For the same share_idx, structural equality and JSON-bytes equality can diverge: fields excluded from / added by JSON (e.g. a `version` discriminant not in the struct, default/optional fields serialized differently, byte-vector vs hex encoding) make two values structurally-equal but JSON-different (→ Go errors `mismatching partial signed data`, Rust silently dedups) or vice-versa (→ Go dedups, Rust raises `ParsigDataMismatch`). A spurious `ParsigDataMismatch` is surfaced as a hard error to the caller and a spurious dedup silently drops a distinct partial.
- **Impact & likelihood:** False equivocation error rejects a legitimate distinct partial, or a genuine equivocation is silently merged · only when JSON-roundtrip equality ≠ structural equality for a `SignedData` impl — depends on serde derives; plausible for versioned types carrying non-serialized state.
- **PoC:** REFINED — confirmed by inspection that the mechanism diverges (Rust `s == &value` structural `DynEq` at memory.rs:340 vs Go `json.Marshal`+`bytes.Equal` at memory.go:232-244), BUT could NOT reproduce an actual struct-eq ≠ JSON-eq divergence with the current concrete types. Scratch test `scratch_struct_eq_vs_json_eq_versioned_attestation` exercised `VersionedAttestation` (the impl with the most complex CUSTOM serde — string-encoded `validator_index`, legacy version mapping) across equal / differ-by-validator_index / differ-by-version-independent-payload pairs: in every case `struct_eq == json_eq` (observed: case0 both true; cases1-3 both false). Every other `SignedData` impl derives `PartialEq` + standard `serde` (serde emits exactly the compared fields), so structural-eq ⇔ JSON-eq holds. Severity DOWNGRADED Medium→Low: no divergence reachable with present types; remains a latent hazard (a future `#[serde(skip)]`/computed field would silently break the equivalence) and a documented deviation from Go's comparison basis. (Sample is representative, not exhaustive across all impls — test deleted, worktree clean.)
- **Fix:** Confirm every `SignedData` impl's `PartialEq` is byte-identical to its JSON round-trip, or compare via the parsigex `serialize_signed_data`/`MarshalJSON`-equivalent bytes to match Go exactly. Document the chosen invariant.

### [Low] aggsigdb mismatch check uses structural PartialEq, not JSON-bytes equality
- **Rust:** `crates/core/src/aggsigdb/memory.rs:91-94` (`MemoryDBActor::store`)
- **Charon ref:** [`core/aggsigdb/memory.go:151-176`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/aggsigdb/memory.go#L151-L176) (`execCommand`, `dataEqual`); `memory_v2.go:45-51`
- **Issue:** Same class as the parsigdb finding. Go's `dataEqual` marshals both `SignedData` to JSON and compares bytes; Rust uses `slot.get() != &signed_data` (boxed `DynEq` structural equality). Divergence when JSON equality ≠ structural equality yields either a spurious `Error::MismatchingData` (rejecting a valid idempotent re-store) or a missed mismatch (overwriting/accepting differing aggregate data silently).
- **Impact & likelihood:** Spurious `MismatchingData` blocks a legitimate duplicate store, or a real conflict is accepted as equal · only when serde round-trip ≠ structural eq for the stored aggregate type.
- **PoC:** REFINED — same class and same verdict as the parsigdb finding above. Mechanism divergence confirmed by inspection (Rust `slot.get() != &signed_data` structural at memory.rs:91 vs Go `dataEqual`→`MarshalJSON`+`bytes.Equal` at memory.go:151-176). No reachable struct-eq ≠ JSON-eq divergence with current concrete types (see the `VersionedAttestation` scratch test result under the parsigdb finding — struct-eq agreed with JSON-eq in all cases). Severity DOWNGRADED Medium→Low: latent maintenance hazard / documented deviation, not currently exploitable.
- **Fix:** Align comparison with Go (JSON-bytes), or assert/document that all aggregate `SignedData` impls satisfy structural-eq ⇔ JSON-eq.

### [Info] aggsigdb v2 (busy-loop Await) variant not ported; only the channel-actor MemDB exists
- **Rust:** `crates/core/src/aggsigdb/memory.rs` (whole file — only `MemoryDBHandle`/actor model)
- **Charon ref:** [`core/aggsigdb/memory_v2.go:81-117`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/aggsigdb/memory_v2.go#L81-L117) (`MemDBV2.Await`, `runtime.Gosched` poll loop)
- **Issue:** Work order lists `aggsigdb/{memory,memory_v2}.go` as the parity target. Charon ships two implementations: `MemDB` (goroutine + blocked-query channel) and `MemDBV2` (RWMutex + spin-on-`Gosched` Await). Pluto implements a single actor model (closest to `MemDB`) and does NOT provide a v2 equivalent. If charon selects V2 by config/default at the call site, pluto's single implementation may not match the deployed semantics (notably: V2's `Await` returns `ctx.Err()`/`ErrStopped` synchronously on every poll iteration; the actor model resolves cancellation only via dropped oneshot → `Terminated`). Behavior is close but the two-implementation parity is incomplete. Confirm which charon impl is wired in v1.7.1 production path.
- **Impact & likelihood:** Divergent Await cancellation/closed semantics vs whichever charon impl is the production default · always (structural), but impact depends on whether V2 is the live path.
- **PoC:** REFUTED (premise) — confirmed by inspection of charon v1.7.1 wiring. [`app/app.go:592-595`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/app.go#L592-L595) selects `NewMemDBV2` only when `featureset.Enabled(AggSigDBV2)`, else `NewMemDB` (V1). `app/featureset/featureset.go`: `AggSigDBV2 = statusAlpha` (value 1), `minStatus = statusStable` (value 2), and `Enabled` returns `state[feature] >= minStatus` — so `AggSigDBV2` is DISABLED by default. The v1.7.1 production default is `NewMemDB` (the channel-blocked-query model), which is exactly what pluto's single actor impl mirrors. So this is an intended consolidation onto the production-default impl, NOT a divergence. Severity DOWNGRADED Medium→Info. Resolves the matching Uncertain item.
- **Fix:** Confirm the charon production AggSigDB impl for v1.7.1; if V2, verify the actor model reproduces its cancellation/closed/await semantics, else document that the single impl is the intended consolidation.

### [Low] sigagg/parsigdb subscriber notification omits Go's per-subscriber deep clone
- **Rust:** `crates/core/src/sigagg.rs:164-166` (`aggregate`); `crates/core/src/parsigdb/memory.rs:242-244,295-297` (`store_internal`, `store_external`)
- **Charon ref:** [`core/sigagg/sigagg.go:71-81`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/sigagg/sigagg.go#L71-L81); [`core/parsigdb/memory.go:67-76,113-118`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/parsigdb/memory.go#L67-L76)
- **Issue:** Go clones the output set (`output.Clone()` / `clone(output)` / `signedSet.Clone()`) before EACH subscriber call so a subscriber cannot mutate data seen by the next. Rust passes a shared `&` reference (`&output`, `&signed_set`) to all subscribers. Because the callbacks take immutable borrows (`&Duty, &AggSignedDataSet`/`&ParSignedDataSet`), they cannot mutate the shared data — the clone is unnecessary in Rust and the borrow model already provides the isolation Go achieves by cloning. No behavioural divergence; flagged only because it is a deliberate, non-obvious deviation from the Go source the reviewer should be aware of.
- **Impact & likelihood:** None functionally; a future subscriber signature change to `&mut`/owned could reintroduce the aliasing Go guards against · n/a.
- **PoC:** n/a
- **Fix:** None required. Optionally add a comment noting the immutable-borrow rationale for omitting Go's clone.

### [Low] resolve_active_validators_indices drops Go's explicit nil-validator error
- **Rust:** `crates/core/src/bcast/mod.rs:663-689` (`resolve_active_validators_indices`)
- **Charon ref:** [`core/bcast/bcast.go:454-477`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/bcast/bcast.go#L454-L477) (`resolveActiveValidatorsIndices`)
- **Issue:** Go returns `errors.New("validator data cannot be nil")` when `val == nil || val.Validator == nil`. Rust's `complete_validators()` yields typed non-nullable data, so the nil case is unrepresentable and the check is correctly omitted (trust proven guarantees). Logged as Info-grade parity note. Rust instead surfaces `InvalidValidatorField` if `activation_epoch` fails to parse — a different (stricter) check Go lacks here.
- **Impact & likelihood:** None; type system rules out the nil state · n/a.
- **PoC:** n/a
- **Fix:** None.

### [Low] first_slot_in_current_epoch rejects pre-genesis time instead of wrapping like Go
- **Rust:** `crates/core/src/bcast/mod.rs:719-775` (`first_slot_in_current_epoch`)
- **Charon ref:** [`core/bcast/bcast.go:435-451`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/bcast/bcast.go#L435-L451) (`firstSlotInCurrentEpoch`)
- **Issue:** Go computes `currentSlot := chainAge / slotDuration` then `uint64(currentSlot)`; if `time.Now()` precedes genesis, `chainAge` is negative, the division is negative, and the `uint64` cast WRAPS to a huge slot number (silent garbage). Rust rejects negative `current_slot_i64` via `u64::try_from` → `Error::InvalidTime`. Rust is strictly safer/more correct; the divergence only manifests pre-genesis (test/devnet), where Go would emit a nonsense first-slot and Rust errors cleanly. Same hardening also present in `DelayCalculator::delay` (checked arithmetic vs Go's unchecked `time.Duration` math).
- **Impact & likelihood:** Behavioural divergence only when wall-clock < genesis (misconfigured/devnet) · negligibly rare in production.
- **PoC:** n/a
- **Fix:** None required; Rust behaviour is preferable. Note the intentional divergence if strict Go parity is ever mandated.

### [Info] parsigex codec relies on serde round-trip == Go JSON for wire compatibility
- **Rust:** `crates/core/src/parsigex_codec.rs:98-287` (`serialize_signed_data`, `deserialize_signed_data`)
- **Charon ref:** [`core/proto.go:265-304`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/proto.go#L265-L304) (`marshal`, `unmarshal`); `core/parsigex/parsigex.go`
- **Issue:** Codec faithfully mirrors Go's "SSZ-if-capable else JSON; on decode, `{`-prefix → JSON else SSZ" branch (`looks_like_json` ↔ `bytes.HasPrefix(TrimSpace,"{")`). Cross-client wire compatibility hinges on each type's serde JSON matching go-eth2-client's JSON exactly (field names, hex/base64 encodings, optional-field presence). That is verified for `Signature` via the snapshot test (line 494) but not exhaustively for every JSON-only type. Out-of-unit per-partial signature verification (charon `parsigex.NewEth2Verifier`) is correctly NOT in these files.
- **Impact & likelihood:** Wire incompatibility with charon peers if any JSON-only type's serde diverges from go-eth2-client JSON · depends on serde fidelity (largely covered elsewhere).
- **PoC:** n/a
- **Fix:** Ensure golden cross-client fixtures cover every JSON-only parsigex type (registration, exit, randao, selections), not just `Signature`.

## Uncertain / needs-human
- **aggsigdb v2 selection:** RESOLVED in Phase 2 — `AggSigDBV2` is `statusAlpha` and disabled by default in v1.7.1 (`minStatus = statusStable`), so the production-default impl is `NewMemDB` (V1), which pluto's actor model matches. Finding downgraded to Info. No human action needed.
- **PartialEq vs JSON-equality (parsigdb & aggsigdb):** PARTIALLY RESOLVED in Phase 2 — for the only custom-serde `SignedData` impl (`VersionedAttestation`) struct-eq empirically agreed with JSON-bytes-eq across tested pairs, and all other impls are derive-PartialEq + standard-serde (so equivalence holds). Both findings downgraded Medium→Low. The cross-check was representative, not exhaustive over every impl — a human/maintainer confirming every `SignedData` impl avoids `#[serde(skip)]`/computed fields would close this fully and let the findings collapse to Info.
- **sigagg template-scan break semantics:** RESOLVED in Phase 2 (reasoned) — divergence is confined to mixed-type per-pubkey parSig sets, ruled out by consensus (one unsigned payload per duty); High downgraded to Low (comment + scan-semantics fix only).

---
## Delta audit 2026-07-02 — commits since 2026-06-30 baseline (#508)
STATUS: VERIFIED (no Critical/High/Medium — Phase 2 not required)

Scope: commit `c241b50`. In `parsigex_codec.rs`, #508 (a) hoists `looks_like_json` from a nested fn to a `pub(crate)` helper shared with `unsigneddata.rs`, and (b) reorders `deserialize_signed_data` for every SSZ-capable duty type to try SSZ first and only fall back to JSON when SSZ fails *and* the payload has a `{` prefix.

### [Info] Prior "JSON-prefix-first" divergence is now FIXED — decode tries SSZ before JSON
- **Rust:** `crates/core/src/parsigex_codec.rs:180` (`deserialize_signed_data`), `:176` (`looks_like_json`)
- **Charon ref:** [`core/proto.go:287`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/proto.go#L287) (`unmarshal`: SSZ first, JSON only on SSZ failure with `{` prefix)
- **Issue:** The existing Low in this dir's sibling (`core-types.md`: "`deserialize_signed_data` decides JSON-vs-SSZ by prefix first") is resolved by #508. All SSZ-capable branches (Attester, Proposer, Aggregator, SyncMessage, SyncContribution) now attempt SSZ decode first and only use the `{`-prefix as a JSON *fallback gate*, matching Charon's `unmarshal` exactly. Two new regression tests (`ssz_sync_message_with_leading_brace_decodes_as_ssz`, `ssz_signed_sync_contribution_with_leading_brace_decodes_as_ssz`) confirm SSZ payloads whose leading `u64` byte is `0x7B` are no longer misrouted to JSON. The per-type SSZ-then-JSON interleaving differs cosmetically from Charon's per-value `unmarshal` (pluto tries [phase0 SSZ, versioned SSZ] then [phase0 JSON, versioned JSON]; Charon tries [phase0 SSZ, phase0 JSON] then [versioned SSZ, versioned JSON]) but is behaviourally equivalent for the concrete eth2 types (no input is both valid-SSZ and JSON-prefixed for a given type). No new finding.
- **Impact & likelihood:** Positive — removes a real (if low-probability) misroute of SSZ payloads beginning with `0x7B` · n/a.
- **PoC:** n/a
- **Fix:** None; change is correct. The prior Low can be considered closed.

### Delta summary
`parsigex_codec.rs` change is a net improvement: the SSZ-first reorder fixes the previously-recorded JSON-prefix-first Low and matches Charon's `unmarshal`, with regression coverage for leading-`0x7B` SSZ. No new parsig findings.
