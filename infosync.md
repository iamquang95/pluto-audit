# Audit: infosync  (849 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/infosync/src/lib.rs`
- **Charon:** `core/infosync/infosync.go`
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `lib.rs` ↔ `core/infosync/infosync.go` (cluster-wide version/protocol sync, built on priority)

## Crate-specific focus
- **Parity:** which info is synced (supported protocols/versions), how consensus result is applied, interaction with priority component.
- **Security:** validate synced info before adoption.
- **Quality:** small crate; clean integration.

---
## Summary

infosync is a close, largely faithful port: topic routing, the empty-versions store gate, consecutive-identical dedup, the `>= maxResults` eviction (cap 99), `ProposalType` open-set handling, and the trigger payload all match charon. One behavioral divergence: pluto replaces charon's insertion-order linear scan with a `partition_point` binary search that silently assumes the result deque is sorted-by-slot — an invariant that holds only because of the (not-yet-wired) monotonic trigger and is neither documented nor asserted. Remaining items are cleanliness (write-only `versions` state, an overstated cap comment) and a benign mutex-poison panic surface with no untrusted-input path. No security-relevant input-validation gaps: all synced values flow through the already-signature-verified priority layer and unknown topics/proposal types are handled exactly as charon does. 0C/0H/1M/2L/1I.

## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Low] Slot-selection assumes a sorted-by-slot invariant charon never relies on
- **Rust:** `crates/infosync/src/lib.rs:128` (`ResultStore::protocols`), `:144` (`ResultStore::proposals`)
- **Charon ref:** [`core/infosync/infosync.go:99`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/infosync/infosync.go#L99) (`Protocols`), `:118` (`Proposals`)
- **Issue:** Charon picks the latest applicable result with a linear scan in *insertion order*: walk `c.results`, `break` on the first `result.slot > slot`, keep the last seen. Pluto replaces this with `results.partition_point(|r| r.slot <= slot)` (binary search), then takes `idx-1`. `partition_point` is only correct if the deque is partitioned by the predicate — i.e. sorted non-decreasing by slot. Nothing in `add_result` (`:102`) enforces that: it appends unconditionally (only consecutive *fully-identical* entries are deduped). The sorted order holds only because the single caller (`Component::trigger`) is in turn driven by a monotonic slot scheduler (charon `app.go:715` `SubscribeSlots`, last-slot-of-epoch). This invariant is undocumented and unasserted. If a duty for an earlier slot is ever stored after a later one (re-trigger of a past slot, replay, out-of-order decision delivery, future caller), `partition_point` returns a wrong index and selection silently diverges from charon's break-on-first-greater scan. Charon's linear scan degrades predictably (stops at the first out-of-order entry); pluto's binary search returns an arbitrary element.
- **Impact & likelihood:** Wrong cluster-agreed protocol/proposal set selected for a slot → e.g. wrong consensus protocol activated cluster-wide · negligibly rare under the current single monotonic trigger; becomes reachable if any non-monotonic trigger path is added. Latent correctness/maintenance hazard.
- **PoC:** CONFIRMED — scratch `#[test]` against the real `ResultStore`: inserted out-of-order slots `[8, 2, 5]` (protocols p8/p2/p5), queried `slot=6`. Predicate `r.slot <= 6` → `[false, true, true]` (not a valid partition), so `partition_point` returned `3` → `get(2)` = slot-5 result → pluto yielded `["p5"]`, while a replica of charon's break-on-first-greater linear scan returned the local default `["local"]` (breaks immediately on slot-8 > 6). Divergence reproduced exactly as claimed; the element pluto returns is an arbitrary artifact of the binary-search path. Charon-side linear scan reasoned from `infosync.go:99-105`/`:118-124` (Go not run). Severity lowered to Low: current trigger is monotonic and app wiring is not present; this is a latent invariant/documentation issue.
- **Fix:** Either (a) document + `debug_assert!` the sorted-by-slot invariant in `add_result` and at the partition_point sites, or (b) match charon exactly with a linear `rev()`/scan that breaks on the first `slot > target`, which is robust to insertion order and equally cheap at ≤99 entries.

### [Low] Doc comment overstates the history cap as a defect; off-by-one matches charon
- **Rust:** `crates/infosync/src/lib.rs:34` (`MAX_RESULTS` doc), `:115` (`add_result`)
- **Charon ref:** [`core/infosync/infosync.go:142`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/infosync/infosync.go#L142) (`addResult`)
- **Issue:** Pluto checks `if results.len() >= MAX_RESULTS { pop_front() }` after push, capping retained history at 99, and the doc comment frames the `>=` as producing an "effectively capped at MAX_RESULTS - 1 (99)" history. This is a faithful port — charon does the identical `if len(c.results) >= maxResults { c.results = c.results[1:] }` (also caps at 99). So parity is preserved; the only issue is the verbose comment presenting a 1:1 charon behavior as a pluto quirk, which can mislead a reader into "fixing" it and breaking parity.
- **Impact & likelihood:** None functionally (parity correct); minor doc-clarity / future-divergence risk · always (cosmetic).
- **PoC:** n/a
- **Fix:** Trim the comment to note this mirrors charon's `>= maxResults` eviction (cap 99) and must not be changed for parity.

### [Low] `versions` agreed by the cluster are stored but never exposed (matches charon, but dead in Rust)
- **Rust:** `crates/infosync/src/lib.rs:46` (`InfoResult.versions`), `:84`, `:94`; `:155` (`Component.versions`)
- **Charon ref:** [`core/infosync/infosync.go:50,58-59,71`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/infosync/infosync.go#L50) / `:83`
- **Issue:** `versions` is collected into each `InfoResult` and used only as the "did the cluster agree on anything" gate (`if !res.versions.is_empty()`), never read back — charon has no `Versions()` getter either, so this is faithful. In Rust the stored `versions` field on `InfoResult` and the `versions: Vec<SemVer>` field on `Component` are write-only beyond the trigger payload / gate. Faithful to charon, but worth flagging as carried-over dead state (no `#[allow]`/doc explains why it is retained).
- **Impact & likelihood:** None functional · always (cleanliness only).
- **PoC:** n/a
- **Fix:** Keep for parity; add a one-line note that `versions` is intentionally store-and-gate only (no getter), mirroring charon, so it is not mistaken for an oversight.

### [Info] `expect` on poisoned mutex panics; acceptable but is a port-introduced panic surface
- **Rust:** `crates/infosync/src/lib.rs:106,124,142` (`*.lock().expect("infosync results mutex poisoned")`)
- **Charon ref:** [`core/infosync/infosync.go:94,113,131`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/infosync/infosync.go#L94) (`c.mu.Lock()` — Go mutexes cannot poison)
- **Issue:** Each lock site `expect`s, panicking if a prior holder panicked while holding the lock. The only code under the lock is push/compare/clone of plain data — none can panic on untrusted input — so poisoning requires an unrelated panic, and re-panicking is the standard, defensible choice. Go has no equivalent (sync.Mutex never poisons), so this is a Rust-port artifact, not a divergence in observable behavior. No untrusted-input panic path identified.
- **Impact & likelihood:** Process abort only after an unrelated panic already corrupted state · negligibly rare.
- **PoC:** n/a
- **Fix:** None required; optionally use the lock guard via `lock().unwrap_or_else(PoisonError::into_inner)` if poison-tolerance is ever wanted. Current behavior is fine.

## Uncertain / needs-human

- **Out-of-order / duplicate-slot triggers reachable in pluto's wiring?** The Medium finding's severity hinges on whether `Component::trigger` can ever be invoked for a non-increasing slot. The integration test (`tests/infosync_integration.rs`) triggers a single slot, and charon drives it from a monotonic epoch-boundary `SubscribeSlots`. Pluto's app wiring for infosync is **not yet present** (the integration test notes "infosync is not yet wired into the app"). A human should confirm the future app trigger is strictly monotonic; if a re-org / catch-up path can re-trigger a past slot, the Medium escalates.
- **`trigger` error/cancellation forwarding:** `Component::trigger` forwards `prioritiser.prioritise(...).await` verbatim. The priority `Component::prioritise` maps both parent-token and deadline cancellation to `Ok(())` (`crates/priority/src/component.rs:371-388`); whether charon's `priority.Prioritise` likewise treats context cancellation as a non-error at this layer was not fully traced. Out of infosync scope but affects observable `Trigger` semantics — flag for the priority-crate audit.
