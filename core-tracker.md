# Audit: core-tracker  (4675 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/core/src/{tracker,gater.rs}`
- **Charon:** `core/tracker/{tracker,inclusion,reason,metrics}.go`, `core/gater.go`
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `tracker` ↔ `core/tracker/*.go` (duty lifecycle tracking, inclusion checks, failure-reason classification)
- `gater.rs` ↔ `core/gater.go` (duty/message gating)

## Crate-specific focus
- **Parity:** duty-inclusion detection, failure `reason` classification taxonomy (must match `reason.go`), tracker event ordering, gater allow/deny rules.
- **Security:** gater enforcement (reject out-of-window duties); bounded tracker state.
- **Quality:** event-stream modeling; metric labels parity.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Low] `report_par_sigs` deliberately diverges from charon's pubkey-count gate, changing which inconsistencies get logged
- **Rust:** `crates/core/src/tracker/reporters.rs:296` (`report_par_sigs`)
- **Charon ref:** [`core/tracker/tracker.go:851`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/tracker/tracker.go#L851) (`reportParSigs`)
- **Issue:** Go gates per-pubkey logging on `len(parsigMsgs) <= 1` — the **outer** map size (number of pubkeys with any parsig), so it logs nothing whenever only a single validator is tracked, regardless of how many distinct roots that validator has. Pluto instead checks `by_root.len() <= 1` (the per-pubkey distinct-root count) and the in-code comment frames it as an "intentional fix over Go". The audit mandate is functional equivalence with charon; this changes observable log output (Pluto logs single-pubkey inconsistencies that Go suppresses, and Go would log a pubkey with exactly 1 root if ≥2 pubkeys exist whereas Pluto skips it). The `inconsistent_parsigs_total` metric increment is unaffected; only the warn/debug log differs.
- **Impact & likelihood:** Diagnostic-log divergence only (no metric or control-flow effect); triggers on every inconsistent-parsig duty. Operators comparing charon vs pluto logs see different "Inconsistent partial signed data" entries.
- **PoC:** CONFIRMED — confirmed by inspection. Go (tracker.go:850-853) gates with `if len(parsigMsgs) <= 1` inside `for pubkey, parsigsByMsg := range parsigMsgs` — i.e. the OUTER map size (# pubkeys), so it skips ALL logging whenever ≤1 pubkey is tracked, regardless of per-pubkey root count (also a quirk: the check is loop-invariant). Pluto (reporters.rs:296) gates with `if by_root.len() <= 1` — the PER-PUBKEY distinct-root count — and the code comment explicitly labels it an "intentional fix over Go (tracker.go:851)". Metric `inconsistent_parsigs_total` increments identically (reporters.rs:289 vs tracker.go:848) — only the warn/debug log set differs. Output-only divergence; Severity lowered to Low because metrics/control flow are unchanged and the code intentionally fixes a Go logging quirk.
- **Fix:** Either restore Go's exact gate (`parsigs.len() <= 1` on the outer map) for strict parity, or keep the improved behaviour but reclassify the code comment as a documented intentional deviation approved by maintainers (it is currently asserted as a bug-fix without sign-off).

### [Low] Tracker uses a buffered channel + `biased` select; charon uses an unbuffered channel + fair select
- **Rust:** `crates/core/src/tracker/mod.rs:171` (`EVENT_BUFFER = 1024`), `mod.rs:466` (`biased;` in `run`)
- **Charon ref:** [`core/tracker/tracker.go:112`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/tracker/tracker.go#L112) (`input: make(chan event)` — unbuffered), `tracker.go:135` (`select` — fair/random)
- **Issue:** Two related concurrency-model divergences. (1) Charon's input channel is **unbuffered**, so producers block until the loop receives each event; pluto buffers 1024, so producers can outrun the loop and the loop drains the buffer in bursts. End-state is equivalent (same events stored/analysed) but timing/ordering relative to deadliner ticks differs. (2) `biased;` makes the loop poll cancel → analyser → deleter → input in fixed priority every iteration, whereas Go's `select` picks a ready case at random. Under a continuously-ready analyser/deleter channel the biased order could defer input draining; in practice deadliner channels only become ready at duty deadlines, so starvation is unlikely.
- **Impact & likelihood:** No correctness divergence found; only event-interleaving timing differs. Negligible likelihood of observable effect under normal deadline cadence.
- **PoC:** n/a
- **Fix:** None required for correctness. If strict charon timing parity is desired, document the buffer/bias choice (the buffer rationale is already documented at mod.rs:166–170); the bias is justified by the shutdown/cleanup-priority comment at mod.rs:464.

### [Low] Gater clamps pre-genesis time to epoch 0; charon underflow-wraps to a huge epoch (allow-all)
- **Rust:** `crates/core/src/gater.rs:122` (`current_epoch`), `gater.rs:132` (`u64::try_from(elapsed_ms).unwrap_or(0)`), `gater.rs:107` (`saturating_add`)
- **Charon ref:** [`core/gater.go:60`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/gater.go#L60) (`currentSlot := o.nowFunc().Sub(genesisTime) / slotDuration`), `gater.go:61` (`uint64(currentSlot)`)
- **Issue:** When `now < genesisTime`, Go computes a negative `currentSlot` then casts to `uint64`, wrapping to a near-`MAX` value → `currentEpoch` huge → `dutyEpoch <= currentEpoch+budget` is true for everything (allow-all). Pluto clamps negative elapsed to 0 (reject-future) and uses `saturating_add` so no wrap. Both treat pre-genesis as unreachable in production (gater built only after genesis), but the behaviours are opposite: charon allows all, pluto allows only the first few epochs. Pluto's behaviour is the safer one (a malicious/early duty is rejected rather than blanket-accepted), so this is a deliberate hardening, not a regression.
- **Impact & likelihood:** Only reachable with an injected pre-genesis clock (tests) — unreachable in production per the documented contract. Divergent gating decision if ever reached.
- **PoC:** n/a
- **Fix:** None needed; keep the safer clamp. Optionally note the intentional deviation in the doc comment alongside the existing "unreachable in practice" note (gater.rs:129).

### [Info] Missing legacy `participation_total` metric present in charon
- **Rust:** `crates/core/src/tracker/metrics.rs:14` (`TrackerMetrics`), `crates/core/src/tracker/reporters.rs:188` (`MetricsParticipationReporter::new`)
- **Charon ref:** [`core/tracker/metrics.go:20`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/tracker/metrics.go#L20) (`participationSuccessLegacy`, name `participation_total`), `tracker.go:638,654` (zero-init + increment)
- **Issue:** Charon still emits a deprecated `core_tracker_participation_total` counter (duplicate of `participation_success_total`) for old dashboards, explicitly TODO-marked for removal (metrics.go:19). Pluto omits it entirely. Since it is deprecated and slated for deletion in charon, omission is reasonable, but any dashboard still keyed on `participation_total` will see no data.
- **Impact & likelihood:** Dashboard-compat only; no functional effect. Affects only legacy Prometheus queries.
- **PoC:** n/a
- **Fix:** Intentional omission acceptable; flag to whoever owns dashboard migration so no panel silently breaks.

### [Info] Divergent error string for the `Bcast`-missing-inclusion bug path
- **Rust:** `crates/core/src/tracker/analysis.rs:227` (`analyse_duty_failed`, `Step::Bcast` arm)
- **Charon ref:** [`core/tracker/tracker.go:277`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/tracker/tracker.go#L277) (`bcast` case)
- **Issue:** When a `Bcast` step has no error (the "should never happen" path), both set a synthetic error. Go sets `"bug: missing chain inclusion event"` and keeps `reason = reasonUnknown`. Pluto sets the identical string and keeps `REASON_UNKNOWN` — parity holds. Listed only to confirm the bug-path strings were checked and match (both `Bcast` and `ChainInclusion` arms match charon verbatim).
- **Impact & likelihood:** None; parity confirmed.
- **PoC:** n/a
- **Fix:** None.

### [Low] Participation "absent set changed" test logs a spurious "All peers participated" on the first duty of each type (nil-vs-empty divergence) (new — 2026-07-02)
- **Rust:** `crates/core/src/tracker/reporters.rs:249` (`MetricsParticipationReporter::report` — `if self.prev_absent.get(&duty.duty_type) != Some(&absent)`)
- **Charon ref:** [`core/tracker/tracker.go:669`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/tracker/tracker.go#L669) (`newParticipationReporter`'s returned closure — `if fmt.Sprint(prevAbsent[duty.Type]) != fmt.Sprint(absentPeers)`)
- **Issue:** Go compares the **string form** of the previous vs current absent-peer slices. For a duty type seen for the first time, `prevAbsent[duty.Type]` is a nil `[]string` and `fmt.Sprint(nil-slice) == "[]" == fmt.Sprint(empty-slice)`, so when the first duty of a type has full participation (empty absent set) Go's condition is false and it logs **nothing**. Rust seeds `prev_absent` lazily via `HashMap::get`, which returns `None` for an unseen type; `None != Some(&empty_vec)` is `true`, so Rust logs `"All peers participated in duty"`. Every subsequent duty agrees once the key holds the empty vec (`Some(empty) == Some(empty)`), so it is a one-time-per-duty-type divergence.
- **Impact & likelihood:** Diagnostic-log only; no metric or control-flow effect. Fires once per duty type (~13 types) over the process lifetime, and only when the first duty of that type has full participation (common on a healthy cluster). Operators diffing charon vs pluto logs see extra early "All peers participated in duty" entries. · always, once per duty type, on a healthy cluster.
- **PoC:** confirmed by inspection (Go not run). Rust side verified verbatim at `reporters.rs:248-260`: the `!= Some(&absent)` comparison is `true` for an unseen `duty_type` because `get` returns `None`, and the `absent.is_empty()` branch then emits the info log; `prev_absent.insert(...)` seeds the key only afterward. Go's `fmt.Sprint`-based nil==empty equivalence reasoned from `tracker.go:669`.
- **Fix:** Mirror Go's nil==empty semantics, e.g. `if self.prev_absent.get(&duty.duty_type).map(Vec::as_slice).unwrap_or(&[]) != absent.as_slice()`, or pre-seed `prev_absent` with an empty `Vec` for every `DutyType` in `new()`.

## Uncertain / needs-human

- **Electra committee-offset positional indexing (parity, needs confirmation).** `inclusion.rs:467-476` (`check_attestation_inclusion`) sums `beacon_committees.iter().take(committee_index)` validator counts to offset into the aggregation bits, mirroring Go's `for idx := range subCommIdx { previousCommsValidatorsLen += len(block.BeaconCommitees[idx].Validators) }` (inclusion.go:454-456). Both index committees **positionally** (vec/slice order), assuming `beacon_committees` is ordered by committee index with no gaps. The Go networked driver guarantees that ordering when building the slice; pluto's `InclusionCore` takes the `Block` pre-built, so the ordering guarantee moves to the (still-unported) driver. Equivalent given the same precondition, but cannot be fully verified until the networked `InclusionChecker` is ported. Flagging so the driver port preserves index-ordering.
- **Unported networked inclusion driver + Electra bit-merging (`conjugateAggregationBits*`).** Go's `InclusionChecker.checkBlockAndAtts` (inclusion.go:695-1061) fetches beacon committees, zeroes signatures, and **merges duplicate attestations' aggregation bits per committee** before handing them to the core. Pluto's `InclusionCore` (inclusion.rs) is explicitly the I/O-free core only; the driver and the bit-merging are deferred (TODO at inclusion.rs:13-15). If the driver port omits the per-committee `Or`-merge of duplicate attestations, Electra/Fulu inclusion checks could miss attestations split across multiple block-attestation objects. Not a current defect (code intentionally absent) — a parity item to track for the follow-up.
- **Pluto guards `par_sig = None` on ParSigDB events; charon dereferences unconditionally.** `analysis.rs:492` (`analyse_participation`) and `analysis.rs:428` (`extract_par_sigs`) skip events whose `par_sig` is `None`. Go (tracker.go:572,576) accesses `e.parSig.ShareIdx` for `parSigDBExternal`/`parSigDBInternal` events with no nil check — a `None`/nil there would panic in Go but is silently skipped in pluto. The producing handlers always set `par_sig` for these steps (mod.rs:243-260), so the field is non-None by construction in both languages; the guard is defensive and currently unreachable. Confirm no future code path can submit a ParSigDB event with `par_sig = None`.

## Summary

The core failure-reason taxonomy (`reason.rs`), step ordering (`step.rs`), `analyse_duty_failed`/`analyse_fetcher_failed_*` control flow, participation accounting, metric labels/names, and the `UnsupportedIgnorer` state machine are faithful, well-tested ports of charon and match the Go behaviour (including the dead `context.Canceled` branch in `analyseFetcherFailed`, where pluto's two-tier mapping is functionally equivalent). No security defects: gater enforcement, bounded tracker state (deleter-driven), and inclusion bit-handling are sound; no `unsafe`, no untrusted-input panics, no integer-overflow surface. The one notable parity item is a **deliberate, comment-justified deviation** in `report_par_sigs` logging gate (Low) — output-only, no metric/control impact. Remaining findings are Low/Info (buffered vs unbuffered channel + biased select, pre-genesis epoch clamp, omitted deprecated `participation_total` metric). The networked `InclusionChecker` driver (and its Electra per-committee bit-merging) is intentionally unported and tracked under Uncertain for the follow-up.

**Delta scan (2026-07-02):** added one **[Low]** — the participation reporter's "absent set changed" gate uses `HashMap::get() != Some(&absent)`, which (unlike charon's `fmt.Sprint` nil==empty comparison) fires on the **first** duty of each type when all peers participated, emitting a spurious one-time "All peers participated in duty" log per duty type. Log-only. Revised counts: 0C / 0H / 0M / 4L / 2I.
