# Audit: priority  (4082 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/priority/src/*` (calculate, component, consensus, prioritiser, p2p/, error)
- **Charon:** `core/priority/{calculate,component,prioritiser}.go`
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `calculate.rs` ↔ `core/priority/calculate.go` (scoring/intersection)
- `component.rs`/`prioritiser.rs` ↔ `core/priority/{component,prioritiser}.go`
- `consensus.rs`/`p2p/` ↔ priority consensus over p2p

## Crate-specific focus
- **Parity:** priority-protocol result calculation (proposal intersection/ordering), prioritiser consensus rounds, p2p message exchange.
- **Security:** validate peer priority messages, bounded proposals.
- **Quality:** integration boundary with consensus crate.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Medium] calculate_result: stable tie-break diverges from charon's unstable sort (cross-impl consensus risk)
- **Rust:** `crates/priority/src/calculate.rs:92` (`calculate_result`, also doc at :89-91)
- **Charon ref:** [`core/priority/calculate.go:73`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/priority/calculate.go#L73) (`calculateResult` → `sort.Slice`)
- **Issue:** Rust orders priorities by score with `Vec::sort_by` (a **stable** sort) and the doc comment claims this "preserves first-seen order for equal scores ... so the output is deterministic and internally consistent." Charon orders with `sort.Slice`, which is explicitly **unstable** in Go — for equal scores Go does not guarantee first-seen order. Both impls are individually deterministic for a fixed input, but their tie-break orderings can differ once ≥3 priorities share the same score (Go's pattern-defeating quicksort may reorder ties that the stable sort leaves in insertion order). The result feeds QBFT consensus, where every peer must compute byte-identical `PriorityResult` to reach quorum.
- **Impact & likelihood:** In a mixed Go+Rust cluster, a tie among priorities that share an aggregate score in one topic yields different `PriorityResult` orderings on Go vs Rust nodes → no quorum on the priority value, instance fails to decide. · Triggers when a single tier of ≥13 equally-scored priorities exists alongside other (differently-scored) priorities in the same topic — i.e. the `allPriorities` slice handed to the sort exceeds Go's pdqsort insertion-sort threshold (n>12) so pdqsort reorders the tied run. A topic of ≤12 distinct priorities (or one where every priority is tied at the *same* score) is preserved identically by both impls (verified below). Pure-Rust clusters are always self-consistent.
- **PoC:** CONFIRMED — ran charon's exact sort (`sort.Slice`, strict `>` on score) in Go 1.25.7 against Rust's stable `sort_by`. (1) Direct comparator probe: identical insertion order, charon `sort.Slice` vs `sort.SliceStable` **diverge on every tested multi-tier / interleaved-tie input with n≥13** (n=13,16,24,32,64,…,1000) — pdqsort scrambles tied runs; they **agree** on all-equal-score inputs (pdqsort short-circuits) and on n≤12 (insertion-sort path, stable). (2) End-to-end: built 32 priorities in two score tiers (16×2000, 16×1000) via real `calculate_result` — Rust emits hi tier `e00,e02,…,e30` (first-seen, stable); feeding the identical `allPriorities=[e00..e31]`/scores through charon's `sort.Slice` emits `e16,e22,e02,e30,e04,…` → orderings differ byte-for-byte. Go executed (not merely reasoned). Refinement vs scanner: divergence threshold is a tied **tier width ≥13**, not "≥3-way"; ties of width 2–12 within a tier are preserved by both, so charon's own ≤12 tests are not affected.
- **Fix:** Match charon's tie semantics exactly. Since both sides intend "by score desc, then first-seen (lower peer id) order," make the comparator total and explicit: carry the first-seen index and sort by `(score desc, index asc)` — equivalently keep the stable sort but document that parity holds only because the comparator is effectively total given pre-sorted input. The real fix is to confirm charon's actual emitted order on ≥3-way ties and reproduce it (Phase 2), rather than asserting "matches Go" from a stable sort against an unstable one.

### [Low] duty_from_proto collapses distinct unknown duty-type integers to one buffer key
- **Rust:** `crates/priority/src/prioritiser.rs:73-76` (`duty_from_proto`); keyed into `req_buffers`/deadliner at :106,:183,:345
- **Charon ref:** [`core/proto.go:42`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/proto.go#L42) (`DutyFromProto` → `Type: DutyType(duty.GetType())`)
- **Issue:** Charon preserves the raw duty-type integer (`DutyType(int)`), so two duties differing only by an out-of-range type (e.g. 98 vs 99) are distinct map keys. Rust maps every unrecognised type to `DutyType::Unknown`, so such duties collapse to the same `Duty` key used for `req_buffers` and the deadliner. The "mismatching duties" validation itself is unaffected — `validate_msgs` (`calculate.rs:160`) compares the raw `Option<ProtoDuty>` like charon's `proto.Equal`, so parity holds there.
- **Impact & likelihood:** Two messages with same slot but distinct out-of-range duty types share one request buffer / deadliner entry, where charon keeps them separate · negligibly rare: the message must first pass `msg_validator` (valid signature from a known cluster peer) and carry a duty-type integer outside 0–13, which never occurs in normal protocol operation.
- **PoC:** n/a
- **Fix:** Key engine state on the raw `(slot, type_int)` (or store the proto duty) rather than the lossy `DutyType`, matching charon's lossless `DutyFromProto`. Acceptable to leave as-is if duty types are guaranteed in-range by an upstream contract; document that assumption at the boundary.

### [Low] handle_request / exchange do not bound priority/topic counts before the run loop
- **Rust:** `crates/priority/src/prioritiser.rs:175` (`handle_request`) and `:456` (`exchange`); `MAX_PRIORITIES` enforced only later in `calculate.rs:175` via `validate_msgs`
- **Charon ref:** [`core/priority/prioritiser.go:199`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/priority/prioritiser.go#L199) (`handleRequest`) — same: only `validateMsgs` enforces the 1000 cap, at calculate time
- **Issue:** Inbound messages are gated by frame size (`MAX_MESSAGE_SIZE = 128MB`, matching charon's `maxMsgSize`) and by `msg_validator` (peer + signature + duty present), but the per-topic 1000-priority cap and per-message topic count are not checked until `calculate_result`. A signed message from a cluster peer can therefore carry up to ~128MB of topics/priorities that are buffered and cloned (`own.clone()`, `add_msg`, score `HashMap`s) before validation rejects it.
- **Impact & likelihood:** Memory/CPU amplification per accepted exchange, bounded by 128MB/message · only from an authenticated cluster peer (signature-gated); this is the same surface charon exposes, so it is parity-equivalent, not a regression. Noted because the verifier is the natural early-rejection point and currently skips the size checks.
- **PoC:** n/a
- **Fix:** Optional hardening (would diverge from charon): move the `MAX_PRIORITIES` / topic-count checks into `msg_validator` so oversized messages are rejected before buffering. If strict parity is required, leave as-is and document that the bound is identical to charon's.

### [Low] buffer_capacity / responses channel buffering diverges from charon's unbuffered/zero-cap channels
- **Rust:** `crates/priority/src/prioritiser.rs:128-130` (`buffer_capacity` → `peers.len().max(1)*2`) and `:397-398` (`responses` channel = `mpsc::channel(peers.len().max(1))`)
- **Charon ref:** `prioritiser.go:249` (`make(chan request, 2*len(p.peers))`) and `:276` (`make(chan *pbv1.PriorityMsg)` — **unbuffered**)
- **Issue:** (a) `buffer_capacity` clamps to `max(1)`, so an empty `peers` set gives capacity 2 where charon gives 0 (unbuffered). (b) The `responses` channel is buffered to `peers.len().max(1)` whereas charon uses an unbuffered channel and relies on the `select{ctx.Done | responses<-}` to apply backpressure. Both Rust paths still deliver every response and dedup correctly; the difference is buffering/backpressure timing only.
- **Impact & likelihood:** None observed functionally — message set and dedup are identical; only channel-fill timing differs · always (structural), but no behavioural consequence since `peers` is non-empty in practice and senders are bounded by the instance token.
- **PoC:** n/a
- **Fix:** None required for correctness. If exact parity is desired, drop the `responses` buffer (use capacity matching charon's intent) and document the `max(1)` clamp as a tokio requirement (`mpsc` capacity must be ≥1).

### [Info] subscriber callback invoked while holding the subscribers mutex
- **Rust:** `crates/priority/src/prioritiser.rs:290-298` (consensus-subscription closure) and `:330-335` (`subscribe`)
- **Charon ref:** `prioritiser.go:107-115` (`newInternal` subscribe wiring); `:180` (`Subscribe` — plain slice, documented not-thread-safe, MUST NOT call after Run)
- **Issue:** Rust holds `subs` `Mutex` for the whole iteration that invokes each subscriber synchronously. A subscriber that re-entrantly calls `Prioritiser::subscribe` (which locks the same mutex) would deadlock. Charon uses a lock-free slice and documents the no-call-after-Run contract instead. The Rust doc on `subscribe` notes "call before any prioritise," so re-entrancy is contractually excluded.
- **Impact & likelihood:** Self-deadlock only if a subscriber re-subscribes during dispatch · effectively impossible given the documented usage contract.
- **PoC:** n/a
- **Fix:** Optionally snapshot/clone the subscriber list (or `Vec<Arc<..>>`) and release the lock before dispatch, so subscriber code never runs under the lock. Low priority given the contract.

## Uncertain / needs-human
- **Tie-break parity (Medium above) — RESOLVED in Phase 2 (Go executed).** Charon's `sort.Slice` (pdqsort) does reorder tied runs once `allPriorities` length n>12 (insertion-sort threshold), while Rust's stable sort preserves first-seen order — confirmed divergent on multi-tier ties at n≥13 and identical for all-equal-score inputs and n≤12. The Medium (cross-impl consensus break in a mixed cluster) stands, with the refined trigger condition (tied tier width ≥13). Note: charon's existing tests stay green because their topics have ≤12 priorities.
- **`min_required` sign/type.** Rust widens charon's `int minRequired` to `i64` and uses saturating arithmetic for `min_score`. For all realistic values (`min_required ≥ 1`) behaviour matches; saturating vs wrapping only differs at extreme/negative inputs that the caller never supplies. Confirm no caller passes a negative `min_required`.

## Summary
Strong, faithful port. Scoring/selection math, `validate_msgs` (empty/dup-peer/dup-topic/max-priority/dup-priority), topic-hash ordering, `Any`-envelope hashing (incl. empty-topic equivalence), signature verify, deadline gating, exchange→consensus orchestration, and the 128MB frame cap all match charon. The libp2p request/response transport (no charon analogue) is well-built and panic-safe (cluster-gated send path avoids the `Either` dummy-handler abort). No Critical/High issues. The one substantive risk is tie-break ordering (Medium): the Rust stable sort vs charon's unstable `sort.Slice` could diverge on ≥3-way score ties and break consensus in a mixed Go+Rust cluster — pure-Rust clusters are self-consistent. Remaining items are Low/Info parity nits (lossy duty-type keying, count-bound timing, channel buffering, locked subscriber dispatch). Counts: 0C/0H/1M/4L/1I.
