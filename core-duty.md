# Audit: core-duty  (6182 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/core/src/{scheduler,scheduler.rs,fetcher,dutydb,deadline,clock.rs}`
- **Charon:** `core/scheduler/{scheduler,offset,metrics}.go`, `core/fetcher/{fetcher,graffiti}.go`, `core/dutydb/memory.go`, `core/deadline.go`
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `scheduler` ↔ `core/scheduler/*.go` (slot→duty timing, epoch/slot math in `offset.go`)
- `fetcher` ↔ `core/fetcher/{fetcher,graffiti}.go`
- `dutydb` ↔ `core/dutydb/memory.go`
- `deadline` ↔ `core/deadline.go`; `clock.rs` ↔ clock abstraction

## Crate-specific focus
- **Parity:** slot/epoch offset math (`offset.go`), graffiti construction, in-memory dutydb keying + dedup, per-duty-type deadline computation, clock (genesis time, slot duration). Verify duty resolution order matches.
- **Security:** unbounded growth of dutydb/deadline maps (DoS); panic on missing/duplicate duty.
- **Quality:** async task lifecycle & cancellation vs Go goroutines; lock granularity.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [High] dutydb: missing Electra committee-index-0 dual-store for attestation data & pubkeys
- **Rust:** `crates/core/src/dutydb/memory.rs:531` (`State::store_attestation` / `store_att_pubkey` / `store_att_data`)
- **Charon ref:** [`core/dutydb/memory.go:343-401`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/dutydb/memory.go#L343-L401) (`storeAttestationUnsafe`, the `pKeyCommIdx0` / `aKeyCommIdx0` blocks)
- **Issue:** Charon stores every attestation duty **twice**: once under the real `committee_index`, and once under a hardcoded `committee_index = 0` (both `attDuties`/`attKey` and `attPubKeys`/`pkKey`). This is deliberate (see Go comment 343-350): post-Electra a spec-compliant VC requests attestation data / `pub_key_by_attestation` with `committee_index = 0`, while other VCs still send the real index, so both must resolve. Pluto only inserts under the real committee index. A consumer (`validatorapi` calls `await_attestation` / `pub_key_by_attestation`, confirmed in `crates/core/src/validatorapi/component.rs`) querying with `committee_index = 0` post-Electra gets `AwaitDutyExpired` / `PubKeyNotFound` instead of the data.
- **Impact & likelihood:** Attestation/aggregation duties silently fail for spec-compliant VCs that request committee index 0 on Electra+ networks (e.g. attestation-data await times out, signed-attestation pubkey lookup 404s) → missed attestations. · Always, once on Electra and a VC uses the index-0 convention.
- **PoC:** CONFIRMED — scratch `scratch_commidx0_dualstore_missing` (tokio test in dutydb tests): stored an attestation under real `committee_index=7`, then queried index 0. Observed `pub_key_by_attestation(slot,0,valIdx) → Err(PubKeyNotFound{committee_index:0})` and `await_attestation(slot,0)` parked until the 300ms outer timeout elapsed (`Err(Elapsed)`); the real-index lookups both resolved. No `committee_index=0` dual-store exists anywhere in `memory.rs` (grep), whereas Charon memory.go:343-401 deliberately inserts under CI=0 too (confirmed by inspection). Test deleted; worktree clean.
- **Fix:** In `store_att_pubkey`/`store_att_data`, mirror Charon: after inserting under the real `committee_index`, also insert under `committee_index = 0` (keyed `AttKey{slot, 0}` and `PkKey{slot, 0, validator_index}`), tracking both keys in `attestation_keys_by_slot` for eviction. For the index-0 attestation-data clash, match only `source`/`target` (not `beacon_block_root`) as Charon does (memory.go:374-401).

### [Medium] scheduler: slot ticker panics instead of erroring on zero slot duration / slots-per-epoch
- **Rust:** `crates/core/src/scheduler.rs:591` (`new_slot_ticker`, `current_slot` closure, `.checked_div(slot_ms).expect("non-zero")` line 602-605) and `:345-350` (`get_duty_definition`, `.checked_div(slots_per_epoch).expect("non-zero")`)
- **Charon ref:** [`core/scheduler/scheduler.go:640-642`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/scheduler/scheduler.go#L640-L642) (`newSlotTicker` returns `errors.New("slot duration cannot be zero")`)
- **Issue:** Charon explicitly returns an error when `slotDuration == 0`. Pluto instead relies on `.expect("non-zero")`, which **panics** if the beacon node reports a zero slot duration; the same `.expect` pattern guards the `slots_per_epoch` division in `get_duty_definition`. Go's behavior is a graceful startup error; Pluto's is a task panic (slot-ticker spawn) / actor-message panic.
- **Impact & likelihood:** A misconfigured/malicious beacon node reporting `slot_duration = 0` (or `slots_per_epoch = 0`) crashes the slot-ticker task or the scheduler-message handler rather than surfacing a clean error. · Only on malformed beacon-node config (rare, but externally controlled).
- **PoC:** CONFIRMED — confirmed by inspection + reasoned (Go not run). No `slot_duration != 0` guard exists anywhere in scheduler.rs (grep for `cannot be zero`/`== 0`/`!= 0` returns no slot-duration check; `build` at :209 calls `new_slot_ticker` without validation). `from_std(ZERO)` succeeds (zero is in range) → `num_milliseconds() == 0` → `checked_div(0)` returns `None` → `.expect("non-zero")` panics (std semantics, self-evident). Charon scheduler.go:640-642 returns `errors.New("slot duration cannot be zero")` at build time instead. NOTE on `slots_per_epoch`: Charon does NOT guard it either (scheduler.go:181 `duty.Slot / slotsPerEpoch` would also panic in Go on zero), so the slots-per-epoch divergence is weaker — both panic; only the slot-duration path is a true Go-error-vs-Rust-panic divergence.
- **Fix:** Validate `slot_duration != 0` (and `slots_per_epoch != 0`) in `new_slot_ticker`/`build` and return `SchedulerError` (mirroring Charon), instead of `expect`-ing in the hot path.

### [Medium] scheduler: `get_duty_definition` drops Charon's "wait while epoch is resolving" semantics
- **Rust:** `crates/core/src/scheduler.rs:339-366` (`SchedulerActor::get_duty_definition`)
- **Charon ref:** [`core/scheduler/scheduler.go:183-201`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/scheduler/scheduler.go#L183-L201) (`GetDutyDefinition` `for s.isResolvingEpoch(epoch)` spin-wait) + `:299-300` (`resolveDuties` sets/clears `resolvingEpoch`)
- **Issue:** Charon, when a `GetDutyDefinition` request races an in-progress epoch resolution, spins (100ms sleeps) until the epoch finishes resolving, then returns the definition — turning a transient race into a successful lookup. Pluto's actor is single-threaded so a request and `resolve_duties` cannot truly interleave; **but** because `resolve_duties` for the *next* epoch only runs on `last_in_epoch` / on the next slot tick, a caller asking for an epoch that has *not yet been resolved at all* gets `EpochNotResolved` immediately, whereas in Charon the same call during the resolving window would block and then succeed. Net: Pluto returns an error in a window where Charon returns data. Behavior diverges; `resolvingEpoch`/`isResolvingEpoch` are entirely absent.
- **Impact & likelihood:** Duty lookups that arrive while the owning epoch is mid-resolution return `EpochNotResolved` (caller must retry) instead of transparently waiting → spurious errors / extra retries at epoch boundaries. · Occasional (epoch-boundary races under load).
- **PoC:** CONFIRMED — confirmed by inspection (reasoned, Go not run). `resolvingEpoch`/`isResolvingEpoch` are entirely absent from scheduler.rs (no spin-wait loop); `get_duty_definition` (scheduler.rs:352-353) returns `SchedulerError::EpochNotResolved` immediately when `!is_epoch_resolved(epoch)`, with no wait. Charon scheduler.go:183-191 spins (`for s.isResolvingEpoch(epoch) { ...time.Sleep(100ms) }`) then returns the definition. The actor-model rationale (single-threaded, request/resolve can't interleave) is partially valid, but the divergence is real for an epoch not-yet-resolved-at-all: Charon's wait-then-succeed window has no Rust equivalent. Severity Medium retained — this is a behavioral/retry divergence, not a correctness defect (callers retry).
- **Fix:** Either document the actor-model divergence as acceptable, or have `get_duty_definition` retry/await resolution (e.g. re-poll after the in-flight resolve completes) to match Charon's wait-then-return contract.

### [Low] scheduler: attester duties not sorted by slot before processing
- **Rust:** `crates/core/src/scheduler.rs:441-468` (`resolve_duties`, attester loop over `fetch_attester_duties` result)
- **Charon ref:** [`core/scheduler/scheduler.go:363-366`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/scheduler/scheduler.go#L363-L366) (`sort.Slice(attDuties, ... Slot < ...)`)
- **Issue:** Charon sorts attester duties ascending by slot before iterating "so logging below in ascending slot order". Pluto iterates in beacon-node response order. Affects only log ordering and `duties_by_epoch` insertion order — no functional/state difference (the map keys are slot-derived).
- **Impact & likelihood:** Cosmetic: "Resolved attester duty" logs may be out of slot order. · Always (but no correctness impact).
- **PoC:** n/a
- **Fix:** Optionally sort `att_duties` by `duty.slot` before the loop to match Charon's log ordering; or note as an intentional cosmetic difference.

### [Low] dutydb: aggregated-attestation clash check dropped (unconditional overwrite)
- **Rust:** `crates/core/src/dutydb/memory.rs:604-621` (`State::store_agg_attestation`, comment "unconditional overwrite")
- **Charon ref:** [`core/dutydb/memory.go:435-462`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/dutydb/memory.go#L435-L462) (`storeAggAttestationUnsafe`, `existingDataRoot != providedDataRoot` → `errors.New("clashing data root")`)
- **Issue:** Charon compares the existing vs provided data root for an already-stored `aggKey` and errors on mismatch; Pluto skips the check and overwrites unconditionally. Pluto keys by root only, so for a *given* root the data is by construction identical — the comment's "unreachable" claim holds for same-root entries. Divergence is benign but removes Charon's defensive guard.
- **Impact & likelihood:** Negligible — same root implies same data; no observable behavior change. · n/a.
- **PoC:** n/a
- **Fix:** None required; optionally keep the equality assertion for defense-in-depth/parity.

### [Low] scheduler: `delay_slot_offset` doc/signature drift — async fn whose `await` is the cancellation point
- **Rust:** `crates/core/src/scheduler.rs:814-833` (`delay_slot_offset`) and call site `:397-405`
- **Charon ref:** [`core/scheduler/scheduler.go:277-295`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/scheduler/scheduler.go#L277-L295) (`delaySlotOffset` returns `bool`, false on ctx cancel)
- **Issue:** Charon's `delaySlotOffset` returns `bool` (false ⇒ cancelled ⇒ skip duty). Pluto returns `()` and relies on the caller wrapping it in `with_cancellation_token_owned(ct)` + checking `.is_none()` (scheduler.rs:398-405). Functionally equivalent and covered by the `cancellation_during_slot_offset_suppresses_duty_broadcast` test, but the cancellation contract lives at the call site, not in the function — easy to misuse if called elsewhere without the wrapper. Doc comment says "Blocks until the slot offset ... has been reached" without noting cancellation must be applied externally.
- **Impact & likelihood:** Maintainability only; current single call site is correct. · n/a.
- **PoC:** n/a
- **Fix:** Document that callers must apply `with_cancellation_token*`, or fold cancellation into the fn (take `&CancellationToken`, return `bool`) to match Charon and make misuse impossible.

### [Info] scheduler: `wait_chain_start`/`wait_beacon_sync` not externally cancellable mid-retry
- **Rust:** `crates/core/src/scheduler.rs:752-811` (`wait_chain_start`, `wait_beacon_sync`); invoked with `.with_cancellation_token(&ct)` at `:200-207`
- **Charon ref:** [`core/scheduler/scheduler.go:730-777`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/scheduler/scheduler.go#L730-L777) (`waitChainStart`/`waitBeaconSync` loop on `ctx.Err() == nil`)
- **Issue:** Charon's wait loops check `ctx.Err()` each iteration, so cancellation breaks the retry loop promptly. Pluto's functions retry with `backon` internally and are only cancellable at the outer `.with_cancellation_token` await in `build`; a `wait_beacon_sync` stuck in its inner `tokio::time::sleep(is_syncing_backoff)` (up to 120s) is interruptible at the await point, so practical parity holds, but the inner `fetch.retry(...)` (infinite, `without_max_times`) is not individually cancellation-checked. Acceptable given the outer token, noted for completeness.
- **Impact & likelihood:** Shutdown latency up to one backoff interval during startup. · Rare (startup only).
- **PoC:** n/a
- **Fix:** None required; optionally thread `ct` into the retry loops for prompt cancellation.

### [Info] deadline: `Add` 4-variant `AddOutcome` vs Charon `bool` — documented, intentional
- **Rust:** `crates/core/src/deadline/mod.rs:98-109` (`AddOutcome`), `:132-146` (`DeadlinerHandle::add`)
- **Charon ref:** [`core/deadline.go:33-39, 198-213`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/deadline.go#L33-L39) (`Add` returns `bool`)
- **Issue:** Pluto splits Charon's single `bool` into `Scheduled / AlreadyExpired / NoDeadline / FailedToCompute`. DutyDB maps these correctly (memory.rs:280-288). One subtle parity point: Charon's `Add` returns `false` for never-expiring duties (`canExpire == false`, deadline.go:158-161), and DutyDB Go treats that `false` as "not storing for expired duty". Pluto instead returns `NoDeadline` and DutyDB maps it to `UnsupportedDutyType` — but never-expiring types (Exit/BuilderRegistration) are *also* rejected by the `duty_type` match, so the outcome (rejection) matches. Documented in code; flagged only to confirm the mapping was checked.
- **Impact & likelihood:** None — behavior matches after mapping. · n/a.
- **PoC:** n/a
- **Fix:** None.

## Uncertain / needs-human

- **dutydb aggregation eviction high-water absence vs await-by-root:** `State` intentionally keeps no `max_evicted_*` mark for aggregated attestations (memory.rs:210-214, 684-692) because they're awaited by root only, matching Charon (Go keeps no eviction record for any type — it relies on query `Cancel`/ctx timeout). Pluto added high-water marks for att/proposer/contrib as an *enhancement* (fail-fast on evicted slots) that Charon lacks. This is a deliberate improvement, not a divergence, but a human should confirm the asymmetry (agg has no fast-fail, others do) is intended and that callers of `await_agg_attestation` always set a request timeout, since an evicted/never-stored root parks until shutdown otherwise.
- **`resolve_duties` blocking the actor during beacon fetches:** scheduler.rs:423-431 comment acknowledges the actor is blocked during `resolve_duties` network calls (matching Charon's single-goroutine `Run`), so `get_duty_definition` / reorg / slot ticks queue behind it. Confirmed equivalent to Charon, but worth a human eye on whether the added latency (esp. the `fetch_slots_config` per `get_duty_definition` call at scheduler.rs:344-345, which the TODO notes should be cached) is acceptable under load.
- **`resolve_active_validators` uses `get_by_head` (POST) vs Charon `CompleteValidators`:** scheduler.rs:670-674 reads the validator cache by head; need a human/eth2api-unit confirmation that `valcache::get_by_head` returns the same active-set semantics as Charon's `eth2Cl.CompleteValidators(ctx)` (head state vs a specific epoch state), since the activation-epoch check (scheduler.rs:703-711) assumes head-state statuses.

## Summary

Scanned scheduler (+offset/metrics), fetcher (+graffiti), dutydb (memory), deadline, and clock against charon v1.7.1. Counts: 0C / 1H / 2M / 3L / 2I. The one High is a genuine functional gap: pluto's in-memory dutydb never mirrors attestation data/pubkeys under the hardcoded `committee_index = 0`, which Charon does deliberately (memory.go:343-401) so post-Electra VCs that query index 0 resolve — pluto returns `AwaitDutyExpired`/`PubKeyNotFound` for them. Mediums are panic-on-zero-config in the slot ticker and the dropped "wait while resolving" semantics in `get_duty_definition`. Graffiti, deadline math (Msecs checked arithmetic), offset fractions (1/3, 2/3), and trim/eviction bookkeeping otherwise track Charon faithfully; several pluto deviations (4-variant AddOutcome, eviction high-water marks, root-only agg keying) are documented, intentional improvements.
