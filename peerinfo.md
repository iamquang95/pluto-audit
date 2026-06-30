# Audit: peerinfo  (1977 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/peerinfo/src/*` (behaviour, handler, protocol, config, failure, metrics, peerinfopb)
- **Charon:** `app/peerinfo/{peerinfo,adhoc,metrics}.go`
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `protocol.rs`/`handler.rs` ↔ `app/peerinfo/peerinfo.go` (version/git-hash/clock-offset exchange)
- `failure.rs` ↔ failure tracking; `peerinfopb` ↔ protobuf messages

## Crate-specific focus
- **Parity:** peerinfo exchange fields (version, git hash, start time, clock offset), periodic interval, failure/disconnect tracking.
- **Security:** validate peer-reported values, bound clock-offset, no trust of peer strings.
- **Quality:** protobuf handling, metric parity.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Low] Nickname map keyed by wrong peer; seeded with wrong key → wrong nickname metrics & log spam
- **Rust:** `crates/peerinfo/src/protocol.rs:88-96` (`ProtocolState::new`), `:116-125` & `:207-213` (`validate_peer_info`/`metrics_submitter`)
- **Charon ref:** [`app/peerinfo/peerinfo.go:111`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L111) (`newInternal`), `:188`, `:208-218` (`sendOnce`)
- **Issue:** Charon keeps ONE shared `nicknames` map across all peers, seeded `{localName: localNickname}`, and on each response keys by the remote `name := p2p.PeerName(peerID)`, comparing prev-vs-new for that remote. Pluto creates a *fresh* `ProtocolState` per connection (`Handler::new` → `Arc::new(ProtocolState::new(...))`), and in `new` seeds the map with `nicknames.insert(self.name /* = remote peer name */, local_info.nickname)` — i.e. it stores the *local* nickname under the *remote* peer's name. Then `validate_peer_info` inserts the remote's real nickname under that same `self.name` key. Net effects: (a) the very first response always sees `prev_nickname = Some(localNickname)`, so `prev != new` is almost always true → `metrics_submitter` resets `nickname{peer, localNickname}=0` (a label that never legitimately existed for that peer) and emits the "Peer name to nickname mappings" info log on essentially every first exchange; (b) the map is per-connection, so it never accumulates the cluster-wide name→nickname view Go logs.
- **Impact & likelihood:** Wrong/garbage nickname gauge series (stale `localNickname` label per remote peer), and noisy info logging that Go suppresses · always on first exchange per connection, and on any genuine change.
- **PoC:** CONFIRMED — scratch `ProtocolState::new(remote_peer, local_info{nickname:"LOCAL-NICK"})` → seed map = `{"precious-leaves": "LOCAL-NICK"}` where `precious-leaves` IS the remote peer name (`state.name == peer_name(remote_peer)`) and the local node name (`agreeable-wood`) has NO entry. So the local nickname is keyed under the remote peer's name (matches claim). Effect (a) follows by inspection: `validate_peer_info` does `nicknames.insert(self.name /*=remote name*/, remote_nick)` → returns `Some("LOCAL-NICK")` (the seed) on the first response, which `!= remote_nick` in the common case → fires the "Peer name to nickname mappings" info log AND `metrics_submitter` resets `nickname{peer,"LOCAL-NICK"}=0`. Effect (b) (per-connection, never cluster-wide) confirmed by inspection: each `Handler::new` builds a fresh `Arc::new(ProtocolState::new(...))`.
- **Fix:** Seed the map with the *local* node's name → local nickname (key = `peer_name(local_peer_id)`, not the remote), matching Go; or drop the per-connection seed entirely and only insert/compare the remote entry. Better: move nickname tracking to behaviour-level shared state so the map reflects all peers as in Charon.

### [Low] Per-connection `ProtocolState` defeats Charon's shared lock-hash/version log filters → unfiltered warn spam
- **Rust:** `crates/peerinfo/src/protocol.rs:178-193` (`validate_peer_info`); `handler.rs:90` (`Handler::new`)
- **Charon ref:** [`app/peerinfo/peerinfo.go:113-120,238,254`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L113-L120) (`lockHashFilters`/`versionFilters` via `log.Filter()`)
- **Issue:** Charon attaches a per-peer `log.Filter()` to the "Mismatching peer lock hash" and "Invalid peer version" logs so the warning/error is emitted at most periodically per peer, not every tick. Pluto has no equivalent filter and logs `warn!`/`error!` unconditionally on every exchange. With a 60s interval and a mismatching/unsupported peer this is a steady stream of warnings.
- **Impact & likelihood:** Log-volume amplification (one warn/error per peer per 60s indefinitely) vs Charon's rate-limited single line · always when a peer has a mismatching lock hash or unsupported version.
- **PoC:** confirmed by inspection (no PoC needed) — pluto `validate_peer_info` emits bare `tracing::error!(... "Invalid peer version")` (protocol.rs:151) and `warn!(... "Mismatching peer lock hash")` (protocol.rs:179) with no rate-limit/guard. Go passes `p.versionFilters[peerID]`/`p.lockHashFilters[peerID]` (`log.Filter()` z.Field) to the same two logs (peerinfo.go:238, 254). `ProtocolState` is per-connection (handler.rs:90) so even a per-state guard couldn't be the shared per-peer filter Go uses. Divergence real. (Go log.Filter rate-limiting reasoned, Go not run.)
- **Fix:** Port `log.Filter()` rate-limiting (or a per-peer "already warned" guard) for the lock-hash and version mismatch logs.

### [Low] `start_time` zero-sentinel mismatch: epoch-0 peer start time still recorded
- **Rust:** `crates/peerinfo/src/protocol.rs:228-230` (`metrics_submitter`)
- **Charon ref:** [`app/peerinfo/peerinfo.go:311-313`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L311-L313) (`newMetricsSubmitter`)
- **Issue:** Charon guards `if !startTime.IsZero()` where Go's zero time is `0001-01-01T00:00:00Z` (`time.Time` zero value). Pluto guards `if start_time != DateTime::<Utc>::UNIX_EPOCH`, i.e. `1970-01-01T00:00:00Z`. A peer sending `started_at = {seconds:0,nanos:0}` is the Unix-epoch in pluto and would be *suppressed* by pluto but the same wire value in Go decodes to a non-zero `time.Time` (1970) and would be *recorded*. Conversely Go's zero (`StartedAt` proto-nil) is already filtered earlier in pluto by the `started_at.is_none()` early return. The sentinel chosen differs from Go's semantics; behaviour diverges only for the degenerate `seconds:0` case.
- **Impact & likelihood:** `start_time_secs` gauge differs from Charon for a peer reporting epoch-0 start time · only on the (adversarial/degenerate) epoch-0 timestamp.
- **PoC:** CONFIRMED — scratch test: `DateTime::<Utc>::from_timestamp(0,0)` → `1970-01-01T00:00:00Z` and `== DateTime::<Utc>::UNIX_EPOCH` is `true`, i.e. the protocol.rs:228 guard SUPPRESSES the epoch-0 start time. Go side (reasoned, Go not run): `resp.GetStartedAt().AsTime()` for `{seconds:0,nanos:0}` decodes to `time.Unix(0,0)` = 1970, whose `IsZero()` is `false` (Go zero time is `0001-01-01`), so Go RECORDS it. Divergence confirmed, scoped to the degenerate `seconds:0,nanos:0` wire value only (proto-nil `started_at` is already filtered by the earlier `is_none()` return).
- **Fix:** Either drop the guard (the proto-nil case is already handled by the earlier `started_at` None check, matching the only realistic zero case) or document that pluto deliberately treats Unix-epoch as "unset".

### [Medium] Stale `version` / `git_commit` gauge series: missing per-peer reset (Charon uses ResetGaugeVec)
- **Rust:** `crates/peerinfo/src/protocol.rs:232-246` (`metrics_submitter`)
- **Charon ref:** [`app/peerinfo/peerinfo.go:326-329`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L326-L329) (`newMetricsSubmitter`), [`app/peerinfo/metrics.go:20,28,64`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/metrics.go#L20) (`peerVersion`/`peerGitHash`/`peerNickname` are `NewResetGaugeVec`)
- **Issue:** Charon resets all label series for a peer before setting the new one: `peerVersion.Reset(peerName)` then `WithLabelValues(peerName, version).Set(1)`, same for `peerGitHash` and `peerNickname`. This guarantees exactly one active `version{peer,…}=1` / `git_commit{peer,…}=1` series per peer. Pluto only does the reset for *nickname* (via the `prev_nickname`→0 dance, protocol.rs:210-213). For `version` (line 238) and `git_commit` (line 246) it does a bare `.set(1)` with no reset of the previously-set label. When a peer changes version or git hash (upgrade/redeploy), the old `version{peer, oldVer}=1` series remains at 1 alongside the new one → multiple simultaneous "active" version series per peer, breaking the "constant gauge" invariant Charon maintains.
- **Impact & likelihood:** Stale/duplicate `version` and `git_commit` series accumulate per peer across version changes; dashboards/queries assuming one active series per peer get wrong results · whenever a peer's version or git hash changes during a node's lifetime (every peer upgrade).
- **PoC:** confirmed by inspection (no PoC needed) — pluto `metrics_submitter` does bare `version[...].set(1)` (protocol.rs:238) and `git_commit[...].set(1)` (protocol.rs:246) with no prior reset; only nickname gets the `prev`→0 dance (protocol.rs:210-213). Go does `peerVersion.Reset(peerName)` then set (peerinfo.go:326-327) and `peerGitHash.Reset(peerName)` then set (peerinfo.go:328-329) on `NewResetGaugeVec` metrics (metrics.go:20,28). Confirmed re the `## Uncertain` note: vise `Family` (rev 73c6543) is backed by a `FrozenMap` (wrappers.rs `FamilyInner`) exposing only `contains`/`get`/`get_lazy`/`to_entries`/`Index` — NO remove/clear/reset. So there is no `vise` equivalent of `Reset(peerName)`; the recommended fix (track prev version/git-hash label per peer and set it to 0, mirroring nickname) is the correct approach.
- **Fix:** Track and reset the previous version/git-hash label per peer (mirror the nickname `prev`→0 logic), or use a metric type with `Reset`/clear-by-peer semantics equivalent to Charon's `NewResetGaugeVec`.

### [Low] No validation/clamp of peer `started_at`/`sent_at` magnitude before feeding gauges
- **Rust:** `crates/peerinfo/src/protocol.rs:159-165` (`started_at`), `:225,:229` (gauge `.set`)
- **Charon ref:** [`app/peerinfo/peerinfo.go:247,311-313`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L247) (`metricSubmitter`)
- **Issue:** A peer can report any in-range `started_at` (e.g. seconds far in the future or `i64`-extreme but still within chrono's valid `DateTime` window, ~year 262143). `clock_offset` is clamped to ±1h before export (good), but `start_time_secs` is set directly to the peer-controlled `timestamp()` with no bound. This is parity with Charon (Charon also sets it unbounded), so it is not a divergence — but it is a genuine untrusted-input → metrics-label/value surface worth flagging: a malicious peer can push absurd `start_time_secs` values into Prometheus.
- **Impact & likelihood:** Misleading/garbage `start_time_secs` series driven by untrusted peer input · only with a hostile/buggy peer; low real-world impact (gauge value, not label cardinality).
- **PoC:** confirmed by inspection (no PoC needed) — `start_time_secs[&peer_name].set(start_time.timestamp())` (protocol.rs:229) is peer-controlled and unbounded, whereas `clock_offset` IS clamped to ±1h (protocol.rs:215-225). Go is likewise unbounded (peerinfo.go:312 `peerStartGauge...Set(float64(startTime.Unix()))`), so this is parity-preserving, NOT a divergence — a trust-surface note only, as the finding already states. No severity change.
- **Fix:** Optionally bound start time to a plausible window before export; at minimum note the divergence-from-trust assumption. (Matches Charon, so a no-op for strict parity.)

### [Low] RTT measurement window differs from Charon (excludes stream-close)
- **Rust:** `crates/peerinfo/src/protocol.rs:264-269` (`send_peer_info`)
- **Charon ref:** [`p2p/sender.go:305,331`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/sender.go#L305) (`SendReceive`), [`app/peerinfo/peerinfo.go:226-228`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L226-L228)
- **Issue:** Charon's RTT = `time.Since(t0)` where `t0` is just before `WriteMsg` and the callback fires after write + `CloseWrite` + `ReadMsg` + `Close`. Pluto measures `start` just before `write_protobuf` and `rtt = start.elapsed()` immediately after `read_protobuf_with_max_size`, excluding the stream close. Clock-offset = `actualSentAt - (now - rtt/2)`; a smaller RTT shifts the estimate slightly. Minor and the offset is clamped to ±1h anyway.
- **Charon ref note:** functional but not exact.
- **PoC:** n/a
- **Fix:** Acceptable; if exactness matters, measure across the full exchange including close.

### [Low] Negative/out-of-range proto `nanos` silently coerced to 0 (no parity issue, but masks malformed input)
- **Rust:** `crates/peerinfo/src/protocol.rs:139-145,159-165` (`u32::try_from(sent_at.nanos).unwrap_or(0)`)
- **Charon ref:** [`app/peerinfo/peerinfo.go:227,247`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L227) (`AsTime()`)
- **Issue:** proto `nanos` is `i32`; a peer can send negative or >1e9 nanos. Pluto coerces invalid nanos to 0 via `unwrap_or(0)` rather than rejecting. Go's `timestamppb.AsTime()` normalizes/handles this internally too, so the observable result is similar, but pluto's silent `unwrap_or(0)` discards the signal that input was malformed. No panic risk (try_from + from_timestamp both fallible-safe).
- **PoC:** n/a
- **Fix:** Acceptable; optionally `warn!` when nanos is out of `[0,1e9)` to surface malformed peers.

### [Low] `protocols()` / `PROTOCOL_NAME` parity OK but no negotiation precedence list
- **Rust:** `crates/peerinfo/src/lib.rs:64-70`
- **Charon ref:** [`app/peerinfo/peerinfo.go:30,36-38`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L30) (`protocolID2`, `Protocols`)
- **Issue:** Both expose a single `/charon/peerinfo/2.0.0`. Parity OK. Note the charon `peerinfo_test.go` still references the legacy `/charon/peerinfo/1.0.0` (stale test); pluto correctly does not implement 1.0.0. No action — recorded so Phase 2 doesn't flag the test mismatch as a gap.
- **PoC:** n/a
- **Fix:** None.

### [Info] `adhoc.go` `DoOnce` has no pluto port
- **Rust:** n/a (absent)
- **Charon ref:** [`app/peerinfo/adhoc.go:19-36`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/adhoc.go#L19-L36) (`DoOnce`)
- **Issue:** Charon exposes `DoOnce` for a one-shot peerinfo query (used by `charon` cmds). No equivalent in pluto's peerinfo crate. May be intentionally out of scope; flag for completeness.
- **PoC:** n/a
- **Fix:** Port `DoOnce` if/when a caller needs ad-hoc peerinfo queries.

### [Info] Builder-API mismatch warning fires every tick (no filter)
- **Rust:** `crates/peerinfo/src/protocol.rs:187-193`
- **Charon ref:** [`app/peerinfo/peerinfo.go:259-265`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/peerinfo/peerinfo.go#L259-L265)
- **Issue:** Charon's builder-API-mismatch warn also has no `log.Filter()` (matches pluto). Recorded only to note this particular warn is intentional parity (unlike the lock-hash/version ones which DO use filters in Go).
- **PoC:** n/a
- **Fix:** None (parity).

## Uncertain / needs-human

- **Per-connection vs shared state model.** Charon runs ONE `PeerInfo` instance with shared `nicknames`/`*Filters` maps and an explicit ticker loop (`Run`/`sendOnce`) iterating `p.peers`. Pluto implements peerinfo as a libp2p `NetworkBehaviour`/`ConnectionHandler` with per-connection `ProtocolState` and `Delay`-driven ticks. This is a deliberate architectural re-port, not a line-by-line translation. The High (nickname seeding) and Medium (log filters) findings flow from this model gap; a human should confirm whether the behaviour-level design is intended to also host the shared nickname map + per-peer log filters, or whether those Charon features were consciously dropped.
- **`failures > 1` "first failure free" logic** (`handler.rs:182-187`) has no Charon counterpart — it is borrowed from libp2p's `ping` handler to tolerate single-substream peers. Confirm this is acceptable (it means the first failed exchange per peer is silently swallowed and never surfaced to the behaviour/metrics, unlike Charon where `sendFunc` errors are logged each time). Possible parity gap in failure visibility; flagged as uncertain because it depends on intended libp2p integration semantics.
- **`vise` reset semantics for the version/git_commit fix.** The stale-series Medium above assumes `vise`'s `Family<Labels, Gauge>` has no built-in per-peer reset equivalent to Charon's `NewResetGaugeVec`/`Reset(peerName)`. A human/Phase-2 should confirm the exact `vise` API for clearing label series so the recommended fix uses the right primitive; if `vise` exposes a reset, the fix is mechanical.

## Summary

Pluto's peerinfo is a faithful re-port of charon's wire protocol (varint-framed protobuf, `/charon/peerinfo/2.0.0`, version-compat check, ±1h clock-offset clamp, git-hash regex) but re-architected from Charon's single shared-state ticker loop into a libp2p `NetworkBehaviour` with per-connection `ProtocolState`. That model shift drives the top findings: the `version`/`git_commit` gauges miss Charon's per-peer reset so stale series accumulate (Medium). Nickname seeding, per-peer log filters, epoch-0 start time, and timestamp magnitude are Low after triage because they affect metrics/logs only or degenerate peer input. No memory-safety or panic risks found on untrusted input — timestamp parsing is fully fallible-safe and message size is bounded to 64 KiB. 0 Critical / 0 High / 1 Medium / 7 Low / 2 Info.
