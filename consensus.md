# Audit: consensus  (11713 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/consensus/src/*` (controller, instance, debugger, timer, protocols, wrapper, qbft/)
- **Charon:** `core/consensus/{controller,debugger,wrapper}.go`, `core/qbft/qbft.go` (+~4.4K)
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `controller.rs`/`instance.rs`/`wrapper.rs` ↔ `core/consensus/{controller,wrapper}.go`
- `qbft/` ↔ `core/qbft/qbft.go` (generic QBFT state machine)
- `debugger.rs` ↔ `core/consensus/debugger.go`; `timer.rs`/`protocols.rs` ↔ round timers + protocol IDs
- NOTE: pluto has `qbft` in both `consensus` and `core`; confirm which is the algorithm vs the wrapper.
- RESOLVED: the generic QBFT **state machine** (↔ `core/qbft/qbft.go`) lives in **`crates/core/src/qbft/mod.rs`** (+`callbacks.rs`), not `crates/consensus/src/qbft/`. `crates/consensus/src/qbft/` is the **wrapper/transport** (↔ `core/consensus/qbft/{qbft,transport,sniffer}.go`): `component.rs`=consensus component, `runner.rs`=run-instance bridge, `definition.rs`=leader/compare/decide callbacks, `msg.rs`=proto adapter, `transport.rs`=broadcast/receive, `sniffer.rs`. Safety-critical algorithm audited = `crates/core/src/qbft/mod.rs`.

## Crate-specific focus
- **Parity (HIGHEST VALUE — consensus safety):** port the QBFT state machine EXACTLY — pre-prepare/prepare/commit transitions, round-change + justification rules, leader election, message validation predicates, timer/round-timeout backoff. Compare `qbft.go` transition-by-transition.
- **Security:** Byzantine handling — reject malformed/duplicate/equivocating messages, verify signatures on consensus msgs, DoS via round flooding / unbounded buffers.
- **Quality:** state-machine modeling, timer cancellation, no `unwrap` on network input.

---
## Summary
The QBFT state machine (`crates/core/src/qbft/mod.rs`) and the consensus wrapper (`crates/consensus/src/qbft/*`) are a faithful, transition-by-transition port of charon v1.7.1: all upon-rules, leader election (`(slot+type+round) % nodes`), message validation, signature verification, quorum/faulty formulas, FIFO buffering, timers (default EagerDoubleLinear), sniffer, debugger, and instance lifecycle match. No Critical/High parity or safety break found. The one notable deviation is an *added* `prepared_round < round` filter on ROUND-CHANGE justification that charon's core lacks — paper-compliant hardening that is safe vs honest traffic but is a state-machine behavioral divergence worth confirming for mixed Rust/Go clusters. Remaining findings are Low/Info (Alpha-only Linear timer ns→ms change, panic-recover semantics, lenient late-message drop, integer quorum arithmetic).
Counts: 0C / 0H / 1M / 3L / 2I.

## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Medium] QBFT adds a `prepared_round < round` ROUND-CHANGE filter absent in charon
- **Rust:** `crates/core/src/qbft/mod.rs:1002` (`valid_round_change_prepared_round`); applied at `:965` (`is_justified_round_change`), `:1073-1076` (`contains_justified_qrc`), `:1170-1173` (`get_justified_qrc`)
- **Charon ref:** [`core/qbft/qbft.go:607`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/qbft/qbft.go#L607) (`isJustifiedRoundChange`), `:691` (`containsJustifiedQrc`), `:772` (`getJustifiedQrc`) — none filter on `pr < round`
- **Issue:** Rust adds a predicate `pr >= 0 && pr < msg.round()` and rejects/filters ROUND-CHANGEs that violate it. Charon's QBFT core has no such check (the Rust comment at `:962-964` acknowledges this divergence). It changes three things vs Go: (1) the inbound `is_justified` gate now *drops* a ROUND-CHANGE whose `prepared_round >= round` that Go would buffer; (2) `contains_justified_qrc` and (3) `get_justified_qrc` compute the justified-QRC set over a *filtered* round-change list, so the Qrc a Rust leader derives can differ from a Go leader's.
- **Impact & likelihood:** Honest QBFT traffic always satisfies `pr < round` (a node prepares only in rounds < the round it round-changes to), so against honest peers this is paper-compliant hardening with no liveness loss. Risk is a Rust↔Go *divergence* in the safety-critical state machine: a crafted/edge ROUND-CHANGE accepted by Go but dropped by Rust could in principle yield a different justified-QRC / leader proposal between mixed Rust+Go clusters, splitting consensus. · only on adversarial/edge `pr>=round` messages; never on honest traffic.
- **PoC:** CONFIRMED — scratch `#[test]` in `crates/core/src/qbft/internal_test.rs` (4 nodes, quorum 3): a ROUND-CHANGE with `round=2, pr=2, pv=7` carrying a valid 3-PREPARE quorum at `round=2 value=7` → `is_justified_round_change` returns `false`. Every charon-side check in `qbft.go:617-646` (prepares non-empty, `len>=quorum`, each PREPARE type/round/value matches) passes, so filter-less Go would return `true`; Rust rejects solely via `valid_round_change_prepared_round` (`pr<round` fails at `pr==round`). The existing `valid_round_change_prepared_round_boundaries` / `invalid_round_change_prepared_rounds_are_filtered_from_call_sites` tests (`internal_test.rs:1765-1826`) corroborate the same drop at all three call sites. Go-side acceptance is `reasoned (Go not run)` from `qbft.go:607-647`. Severity unchanged (Medium): divergence is real but triggers only on `pr>=round` messages, which honest traffic never emits.
- **Fix:** Confirm the IBFT-2.0 safety argument that no honest+Byzantine combination can make a `pr>=round` ROUND-CHANGE consensus-relevant in Go but not Rust (and vice-versa). If parity with charon is required, gate the filter behind a feature or remove it; if the hardening is intended, document it as an explicit, reviewed safety deviation from charon v1.7.1 and keep the boundary tests (`internal_test.rs:1770-1834`).

### [Low] LinearRoundTimer subsequent-round timeout uses milliseconds; charon v1.7.1 uses (buggy) nanoseconds
- **Rust:** `crates/consensus/src/timer.rs:369-387` (`linear_subsequent_round_timeout`) → `Duration::from_millis(200*(round-1)+200)`
- **Charon ref:** [`core/consensus/timer/roundtimer.go:243`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/consensus/timer/roundtimer.go#L243) (`linearRoundTimer.Timer`) → `time.Duration(200*(round-1) + 200)` (nanoseconds — charon v1.7.1 still has the un-fixed ns bug; the Rust comment at `timer.rs:372-373` claims #4537 fixed it, but the pinned v1.7.1 source does not)
- **Issue:** For rounds > 1 on the `Linear` proposer timer, Rust waits 200–N00 ms while charon v1.7.1 effectively waits ~sub-microsecond (immediate timeout). Divergent round-timeout durations on this path.
- **Impact & likelihood:** Round-boundary misalignment between Rust and charon proposers on the Linear path; charon round-changes almost instantly, Rust does not. `Linear` is `Alpha`/off-by-default (`featureset` Status::Alpha) and only affects `DutyProposer`; default path is `EagerDoubleLinear` (whole-second timeouts) which matches Go. · only when `Linear` feature enabled.
- **PoC:** n/a
- **Fix:** Either match charon v1.7.1 exactly (ns) to preserve bug-for-bug parity, or — if intentionally fixing the charon bug — correct the code comment (v1.7.1 is unfixed) and flag it as a deliberate, reviewed deviation.

### [Low] No panic→error recovery boundary equivalent to charon's `recover()`
- **Rust:** `crates/core/src/qbft/mod.rs:332` (`run`) — `panic!("bug: ...")` at `:399,:417,:650,:901,:911,:919,:921,:945,:953,:1010,:1035,:1384` unwinds the `spawn_blocking` thread
- **Charon ref:** [`core/qbft/qbft.go:188-198`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/qbft/qbft.go#L188-L198) (`Run` deferred `recover()`) — converts panics containing "bug" into a returned `"qbft sanity check: %v"` error and re-panics others
- **Issue:** Go turns internal sanity-check panics into a graceful instance error. Rust has no in-`run` recover; a `bug:` panic unwinds across the `tokio::task::spawn_blocking` boundary (`runner.rs:379`) and surfaces as `JoinError` → `runner::Error::Join`. Functionally the instance still fails without crashing the process (workspace uses default `panic=unwind`, no `panic="abort"` profile), but the error type/message and the "only catch bug-panics, re-panic others" semantics differ.
- **Impact & likelihood:** Minor behavioral/observability divergence; instance fails either way. Would become a process-abort divergence only if a `panic="abort"` profile were ever introduced. · only if an internal invariant is violated (should be never).
- **PoC:** n/a
- **Fix:** Optionally catch `bug:`-prefixed panics in `run_instance_inner` and map to a dedicated error to mirror Go; or document that thread-unwind→`Join` is the intended equivalent and assert `panic=unwind` is required.

### [Low] `handle` admits late messages to a completed instance; charon does not special-case this
- **Rust:** `crates/consensus/src/qbft/component.rs:381-393` (`handle`) — `Err(_) if inst.has_started() => Ok(())`
- **Charon ref:** [`core/consensus/qbft/qbft.go:669-675`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/consensus/qbft/qbft.go#L669-L675) (`handle`) — only `select { recvBuffer<-msg | ctx.Done }`; no started/closed branch
- **Issue:** Rust silently treats a send failure to a *started* instance's receive channel as success (drop the late message). Go has no such branch; it relies on the buffered channel (kept until the deadliner expires the duty) and ctx.
- **Impact & likelihood:** Rust is more lenient — a late inbound message after the receiver task ended returns Ok (dropped) instead of erroring. Behavior is benign (late messages are not needed post-decision) and documented at `:385-387`, but it is an inbound-path divergence from charon. · only for messages arriving after an instance's receiver has been torn down.
- **PoC:** n/a
- **Fix:** None required if intentional; note the deviation. Keep regression test `handle_drops_late_message_after_started_receiver_closed`.

### [Info] `quorum`/`faulty` use integer arithmetic vs charon's float `ceil`/`floor`
- **Rust:** `crates/core/src/qbft/mod.rs:154-160` (`quorum` = `(2N+2)/3`), `:170-175` (`faulty` = `(N-1)/3`)
- **Charon ref:** [`core/qbft/qbft.go:59-67`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/qbft/qbft.go#L59-L67) — `Quorum()` = `int(math.Ceil(2N/3))`, `Faulty()` = `int(math.Floor((N-1)/3))`
- **Issue:** Rust replaces float `ceil/floor` with integer identities `ceil(a/3)==(a+2)/3` and `floor((N-1)/3)==(N-1)/3`. Mathematically equal for all non-negative N; avoids float. No divergence for any realistic peer count.
- **Impact & likelihood:** None (equivalent); float path could only differ at N large enough to lose f64 precision, far beyond any cluster size. · never in practice.
- **PoC:** n/a
- **Fix:** None.

### [Info] Per-justified-PRE-PREPARE compare spawns an OS thread (matches Go goroutine, but heavier)
- **Rust:** `crates/core/src/qbft/mod.rs:728` (`compare`) — `thread::spawn` per `UPON_JUSTIFIED_PRE_PREPARE`, detached by design
- **Charon ref:** [`core/qbft/qbft.go:449`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/qbft/qbft.go#L449) (`compare`) — `go d.Compare(...)` (goroutine)
- **Issue:** Faithful port of Go's detached compare goroutine, but uses a full OS thread rather than a lightweight task. The thread can outlive the call if a caller-provided compare callback ignores cancellation (documented at `:726-727`).
- **Impact & likelihood:** Functionally equivalent; potential thread-churn/leak only under a misbehaving compare callback or rapid round changes. The in-tree compare callback respects cancellation, so bounded in practice. · negligible with the default attester compare callback.
- **PoC:** n/a
- **Fix:** None required; consider a bounded blocking-pool if compare frequency ever grows.

## Uncertain / needs-human
- **`get_fplus1_round_changes` early-break determinism** (`core/src/qbft/mod.rs:1212-1248` ↔ `qbft.go:816-849`): both break at exactly `faulty+1` distinct sources while iterating an *unordered* map, so neither guarantees it picked each source's globally-highest round before breaking. This Go quirk is faithfully reproduced (parity preserved), but the "returns the highest round per process to jump furthest" doc-claim is only partially true in both. Not a Rust regression — flagging because it affects how far a node jumps on F+1 round-changes and could be worth a human/spec review for both implementations.
- **`compare` detached-thread cancellation under teardown**: on instance teardown `core_cts.cancel()` cancels the compare token, but a compare callback that blocks on `input_value_source_ch` and ignores its `ct` could leave a parked OS thread. Matches Go's goroutine-leak risk; human should confirm the production attester compare callback (`definition.rs:212-238`) always observes cancellation (it uses `ct.run` with a cancel channel — appears correct, but worth confirming under all paths).
- **Mixed Rust/Go cluster safety of the `pr<round` filter** (see Medium finding): needs a human with the IBFT-2.0 safety proof to confirm the added filter cannot cause a safety/liveness split between charon and pluto nodes in the same cluster.
