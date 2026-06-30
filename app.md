# Audit: app  (6469 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/app/src/*` (eth2wrap, health, log, monitoringapi, obolapi, sse, privkeylock, retry, utils)
- **Charon:** `app/{health,log,obolapi,sse,privkeylock,retry}.go`, `app/lifecycle`, `app/health/*`
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `health/` ↔ `app/health/{checker,checks,reducers,select,metrics}.go`
- `monitoringapi/` ↔ monitoring/metrics endpoints
- `obolapi/` ↔ `app/obolapi.go`; `privkeylock.rs` ↔ `app/privkeylock.go`; `sse/` ↔ `app/sse.go`

## Crate-specific focus
- **Parity:** startup/lifecycle wiring + ordering, health checks & reducers logic, monitoring endpoint shapes, obolapi (launchpad) integration, privkeylock semantics, sse handling.
- **Security:** monitoring/obol API exposure + TLS, privkeylock race conditions, no secret logging.
- **Quality:** lifecycle abstraction vs Go's; graceful shutdown.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [High] privkeylock: lock-file timestamp wire format diverges from charon (string vs integer)
- **Rust:** `crates/app/src/privkeylock.rs:52-56` (`struct Metadata`), `:55` (`timestamp: u64`)
- **Charon ref:** [`app/privkeylock/privkeylock.go:97-101`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/privkeylock/privkeylock.go#L97-L101) (`type metadata`, `Timestamp time.Time`)
- **Issue:** Go serializes `Timestamp` as a Go `time.Time`, which `encoding/json` marshals to an RFC3339 **string** (`"2026-06-30T..."`). Rust serializes `timestamp` as a `u64` unix-seconds **number**. The on-disk JSON lock file is therefore mutually incompatible: a lock written by charon (`{"command":...,"timestamp":"2026-..."}`) fails `serde_json::from_slice::<Metadata>` in pluto (`Json` error → `Service::new` returns `Err` → pluto refuses to start), and a pluto-written file (`{"timestamp":1750000000}`) fails charon's `json.Unmarshal` into `time.Time`. This is an external on-disk format that the two implementations are expected to interoperate on (same `.charon/<...>.lock` path, "another charon instance may be running").
- **Impact & likelihood:** Mixed charon/pluto deployment sharing a data dir: stale-lock detection breaks and startup may abort, OR (worse) the parse error path is hit and a genuinely-active lock from the other binary is not honored, allowing two instances to run on the same key. · always, whenever a lock file written by the other implementation is read.
- **PoC:** CONFIRMED — scratch `scratch_timestamp_wire_format` → pluto serializes `{"command":"charon","timestamp":1750000000}` (integer); deserializing a charon-style `{"timestamp":"2026-06-30T12:00:00Z"}` returns `Err` (`parse charon file ok = false`). Note: in pluto the parse error does NOT fall through to "ignore lock" — `Service::new` propagates the `Json` error (`?` at privkeylock.rs:96) so pluto *refuses to start* on a charon-written file rather than ignoring a live lock; the two-instances scenario only arises in the reverse direction (charon reading a pluto integer file). Direction-of-impact noted; severity unchanged (High — interop breakage on shared data dir, always).
- **Fix:** Serialize the timestamp as an RFC3339 string to match Go (e.g. store `chrono::DateTime<Utc>` with default serde, or a custom serializer producing RFC3339). Add a round-trip test against a charon-produced fixture.

### [Low] privkeylock: stale comparison uses integer-second `<=` vs Go's sub-second `time.Since`
- **Rust:** `crates/app/src/privkeylock.rs:98-100` (`Service::new`), `now_secs().saturating_sub(meta.timestamp)` then `elapsed <= STALE_DURATION.as_secs()`
- **Charon ref:** [`app/privkeylock/privkeylock.go:35`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/privkeylock/privkeylock.go#L35) (`time.Since(meta.Timestamp) <= staleDuration`)
- **Issue:** Go compares full-resolution durations (`<= 5s`). Rust truncates both timestamps to whole seconds before subtracting, so the boundary differs by up to ~1s and the comparison is on integer seconds. Combined with the format divergence above this is mostly moot cross-impl, but even pluto-to-pluto the staleness window is coarser/shifted vs charon semantics. Minor.
- **Impact & likelihood:** A lock ~5s old is classified stale/active off-by-up-to-1s vs charon. · rare boundary timing only.
- **PoC:** CONFIRMED — scratch `scratch_stale_integer_truncation` → with `now=100`, `meta_ts=95`, `elapsed = now.saturating_sub(meta_ts) = 5` and `5 <= 5` ⇒ classified active. Both timestamps are whole-second (`now_secs()` truncates via `.as_secs()`), so sub-second resolution is lost vs Go's `time.Since`. Boundary off-by-up-to-1s confirmed. Severity lowered to Low: real boundary divergence, but only ~1s timing drift.
- **Fix:** Store and compare full-resolution timestamps (RFC3339 / `DateTime<Utc>`) and compute `Utc::now() - meta_ts <= STALE_DURATION`.

### [Low] retry::do_async: an already-expired deadline yields unbounded retries instead of immediate timeout
- **Rust:** `crates/app/src/retry.rs:128-142` (`do_async`), `:135` `deadline.and_then(|deadline| (deadline - now).to_std().ok())`
- **Charon ref:** [`app/retry/retry.go:40-47`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/retry/retry.go#L40-L47) (`ctxTimeoutFunc` → `context.WithDeadline`) + `:156-175` (`ctx.Err()` checks)
- **Issue:** When `deadline_fn` returns a time **in the past** (`deadline < now`), `chrono::Duration::to_std()` returns `Err` for the negative value, so `.ok()` yields `None`. `with_total_delay(None)` plus `without_max_times()` (line 54) means the backoff iterator is never exhausted → with no cancellation token and a persistently `RetryableError` function, retries run **forever**. Go instead creates a context whose deadline is already passed: the first `fn(ctx)` runs, and the `ctx.Err() != nil` branch fires → returns "timeout" after 1 attempt. The `one_attempt_timeout` test only covers `deadline == now` (→ `Some(ZERO)` → exhausts immediately), not `deadline < now`.
- **Impact & likelihood:** A retry whose deadline is already past (e.g. an expired duty slot, the exact case charon's slot-deadline retryer handles) loops indefinitely consuming a task slot, instead of failing fast. · whenever a caller's `deadline_fn` returns a past instant and the op keeps failing retryably with no cancellation token.
- **PoC:** CONFIRMED — two scratch tests. (1) `scratch_past_deadline_no_cap`: replicating `do_async`'s arithmetic with `deadline = now - 10s` gives `(deadline-now).to_std().ok() = None` (vs `deadline == now` ⇒ `Some(0ns)`), so `with_total_delay(None)` = no cap. (2) `scratch_past_deadline_runs_forever`: ran the real `do_async` with a past deadline, persistently-`RetryableError` fn, no cancellation token, wrapped in a 200ms `tokio::time::timeout` — it did NOT return on its own (`returned within window = false`) and executed 91 attempts in the window. Unbounded retries confirmed; Go (`context.WithDeadline` already-passed → first `fn(ctx)` runs, `ctx.Err()!=nil` → "timeout" after 1 attempt) reasoned (Go not run). Severity lowered to Low: no production caller currently installs a deadline function, so this is latent module behavior.
- **Fix:** Treat a past/zero deadline as an immediate timeout: if `deadline.is_some()` and `(deadline - now) <= 0`, set `total_delay = Some(Duration::ZERO)` (or short-circuit to a single attempt). Do not fall through to the `None` (=no cap) path.

### [Low] No `log.Filter()` equivalent: health/version/retry warnings are not rate-limited
- **Rust:** `crates/app/src/health/checker.rs:121-138` (`instrument`, logs every 30s tick with no filter); `crates/app/src/log/mod.rs` (no filter API); `crates/app/src/eth2wrap/version.rs:66-89`; `crates/app/src/retry.rs:172-174`
- **Charon ref:** [`app/health/checker.go:36`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/health/checker.go#L36),`:80` (`logFilter: log.Filter()`, passed to `log.Warn`); `app/log/filter.go` (`Filter()` rate-limits to once per 12s slot)
- **Issue:** Charon throttles repeated identical warnings (failed health checks, etc.) via a stateful `log.Filter()` field appended to the log call. Pluto has no `log.Filter()` port (the `log` module only re-exports `loki`), and `health::checker::instrument` explicitly logs every failing check every tick "no rate-limiting" (comment at checker.rs:127). A persistently-failing check therefore emits a warning every 30s indefinitely; other ported call sites that Go rate-limits are likewise unthrottled.
- **Impact & likelihood:** Log spam / noisier operator logs under sustained failure; not a correctness bug. · always, while any check fails continuously.
- **PoC:** confirmed by inspection (no PoC needed) — `crates/app/src/log/mod.rs` contains only `pub mod loki;` (no `Filter`/rate-limiter port; `grep -rln "Filter|rate_limit"` over `crates/app/src/log/` is empty). `health/checker.rs:127` carries an explicit comment "Logged every tick (no rate-limiting)" and calls `warn!(...)` unconditionally each tick. Charon's `app/log/filter.go` `Filter()` is absent. Severity lowered to Low: log-noise only, no control-flow or metric impact.
- **Fix:** Port `log.Filter()` (e.g. a per-call-site rate limiter keyed by message, ~1/slot) and apply it in `instrument` and other warn sites that Go filters.

### [Low] eth2wrap version: Grandine minimum-version pre-release string differs from charon
- **Rust:** `crates/app/src/eth2wrap/version.rs:31` (`("grandine", SemVer::parse("v2.0.0-rc0").unwrap())`)
- **Charon ref:** [`app/eth2wrap/version.go:20`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/eth2wrap/version.go#L20) (`minGrandineVersion, _ = version.Parse("v2.0.0.rc0")`)
- **Issue:** Charon's Grandine minimum is `v2.0.0.rc0` (dot before `rc`); pluto's is `v2.0.0-rc0` (hyphen). These are different pre-release encodings and may parse/order differently in the two SemVer impls. Note the *client* version is always reduced to `X.Y.Z` by the extract regex (pre-release dropped), so a released `Grandine/2.0.0` is compared against the minimum's pre-release: under standard semver `2.0.0` (no pre-release) > `2.0.0-rc0`, but charon's `version.Parse`/`Compare` semantics for `v2.0.0.rc0` must be confirmed. Divergent constant → potential off-by-one in the "too old" boundary for Grandine.
- **Impact & likelihood:** A Grandine version exactly at the rc boundary could be flagged too-old in one impl but OK in the other. Warning-only (no functional gate). · only for Grandine builds near v2.0.0-rc.
- **PoC:** REFINED — scratch `scratch_grandine_minversion_parse`. Both impls use the IDENTICAL regex `^v(\d+)\.(\d+)(?:\.(\d+))?(?:-(.+))?$` (charon [`app/version/version.go:167`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/version/version.go#L167), pluto `core/version.rs:109`) which requires a hyphen before the pre-release. Pluto's `v2.0.0-rc0` parses fine. Charon's literal `v2.0.0.rc0` (dot, not hyphen) does NOT match the regex (`charon v2.0.0.rc0 parse ok = false`), so charon's `minGrandineVersion, _ = version.Parse(...)` discards the error and stores the ZERO value `SemVer{}` (v0.0). Net effect (Go-side reasoned, not run): charon's Grandine minimum is effectively v0.0, so charon NEVER flags any Grandine version as too-old; pluto enforces a real v2.0 floor (`Compare` ignores pre-release, so `Grandine/2.0.0` ⇒ equal ⇒ not-too-old: scratch `client v2.0.0 >= min v2.0.0-rc0`, but `Grandine/1.x` ⇒ too-old in pluto, OK in charon). This is a divergent *constant intent*, not merely an rc-boundary off-by-one — but pluto's behavior is arguably the correct one and charon's is a latent bug. Severity lowered to Low: warning-only, and pluto arguably fixes a charon parsing bug.
- **Fix:** Use the identical string charon uses (`v2.0.0.rc0`), or confirm both parse to the same ordering and document the intentional normalization.

### [Low] eth2wrap version: client-name match is case-insensitive in pluto but case-sensitive in charon
- **Rust:** `crates/app/src/eth2wrap/version.rs:52-54` (`.get(&name.to_lowercase().as_str())`, all-lowercase keys at `:26-31`)
- **Charon ref:** [`app/eth2wrap/version.go:22-29`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/eth2wrap/version.go#L22-L29) (mixed-case keys `"Lighthouse"`,`"teku"`,`"Lodestar"`,`"Nimbus"`,`"Prysm"`,`"Grandine"`; lookup `minimumBeaconNodeVersion[client]` with raw regex capture)
- **Issue:** Charon's map keys are inconsistently cased and matched exactly against the client name as reported (`Lighthouse/...`, but `teku/...` lowercase). Pluto lowercases both sides, so it matches all casings. Divergence: a BN reporting a non-canonical case (e.g. `Teku/...` capital-T, or `LIGHTHOUSE/...`) is recognized by pluto (may warn too-old) but is `VersionUnknownClient` in charon (different warning). Warning text only.
- **Impact & likelihood:** Different warning emitted for oddly-cased client strings; no functional effect. · only on non-canonical client-name casing.
- **PoC:** n/a
- **Fix:** Acceptable as-is (pluto's is arguably better); if strict parity wanted, match charon's exact-case keys.

### [Low] obolapi: parent context/cancellation not propagated into HTTP calls
- **Rust:** `crates/app/src/obolapi/client.rs:80-179` (`http_post`/`http_get`/`http_delete`), `:38-58` (`Client::new` bakes timeout into reqwest client)
- **Charon ref:** [`app/obolapi/api.go:85-171`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/obolapi/api.go#L85-L171) (`httpPost`/`httpGet`/`httpDelete` take `ctx`), [`app/obolapi/exit.go:116`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/obolapi/exit.go#L116),`:144` (`context.WithTimeout(ctx, c.reqTimeout)` per call)
- **Issue:** Charon threads a `context.Context` through every request and derives a per-call timeout from it, so a cancelled parent context (shutdown, duty expiry) aborts an in-flight request. Pluto's `Client` methods take no context/cancellation token; the only bound is the reqwest client's fixed 10s timeout. The per-call *timeout* magnitude matches (10s), but parent-driven cancellation is lost — a shutdown cannot abort an outstanding obolapi call early; it waits up to the full timeout.
- **Impact & likelihood:** Slower graceful shutdown / no early-abort on cancellation for obolapi operations (lock publish, full-exit fetch, partial-exit submit). · whenever the app cancels while an obolapi request is in flight.
- **PoC:** confirmed by inspection (no PoC needed) — pluto's exit methods `post_partial_exits` / `get_full_exit` / `delete_partial_exit` (exit.rs:248,292,375) and the client HTTP helpers `http_post`/`http_get`/`http_delete` (client.rs:80-179) take NO `ctx`/`CancellationToken` param; the only bound is the reqwest client's fixed timeout (`Client::new`, default 10s). Charon's `httpPost`/`httpGet`/`httpDelete` (api.go:85,115,146) all take `ctx context.Context`, and `exit.go:116,144` wrap it with `context.WithTimeout(ctx, c.reqTimeout)`, so a cancelled parent aborts in-flight. Parent-cancellation propagation is lost in pluto. Severity lowered to Low: graceful-shutdown latency only, no correctness loss.
- **Fix:** Accept a `CancellationToken` (or `&Context`) in the obolapi client methods and `tokio::select!` the request against it, mirroring charon's ctx propagation.

### [Medium] obolapi: partial-exit sort re-parses validator_index string with `unwrap_or_default()`
- **Rust:** `crates/app/src/obolapi/exit.rs:261-267` (`post_partial_exits`)
- **Charon ref:** [`app/obolapi/exit.go:88-91`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/obolapi/exit.go#L88-L91) (`sort.Slice` on native `ValidatorIndex` field)
- **Issue:** The exit blobs are sorted ascending by validator index. Charon sorts on the already-typed `ValidatorIndex` (numeric). Pluto re-parses `validator_index` from its **string** form via `.parse::<u64>().unwrap_or_default()`, so any non-numeric/empty index silently sorts as `0`. Because the sorted set is SSZ `hash_tree_root`-ed and signed (`:274-275`), a divergent sort order produces a different signed payload than charon would for the same input, and a malformed index is masked rather than surfaced.
- **Impact & likelihood:** Mis-ordered/mis-signed partial-exit request, or silent acceptance of a malformed index. · only if `validator_index` is ever non-numeric (internally-produced strings are likely always valid, lowering likelihood).
- **PoC:** CONFIRMED — scratch `scratch_validator_index_unwrap_or_default` → `""` and `"abc"` both `.parse::<u64>().unwrap_or_default()` to `0`, while `"42"`→42; sorting `["10","5","bad",""]` by this key yields `["bad","","5","10"]` (the two unparseable indices silently collapse to the minimum). Field type confirmed `String`: `signed_exit_message.message.validator_index` is `.parse::<u64>()`-ed at exit.rs:60 and exit.rs:265, so it is a string on the wire (charon sorts the already-numeric `ValidatorIndex`). Malformed-index masking + divergent sort order (hence divergent signed `hash_tree_root`) confirmed. Severity unchanged (Medium; gated on a non-numeric index ever reaching this path).
- **Fix:** Parse the index with `?` (propagate error) before sorting, and sort on the parsed `u64`; reject unparseable indices.

### [Low] obolapi: failed DELETE error reports method GET (`Method::default()`)
- **Rust:** `crates/app/src/obolapi/client.rs:171-175` (`http_delete`, `method: Method::default()`)
- **Charon ref:** [`app/obolapi/api.go:167`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/obolapi/api.go#L167) (`errors.New("http DELETE failed", ...)`)
- **Issue:** On a non-2xx DELETE, the error is constructed with `Method::default()` which is `GET`, so the surfaced error mislabels the failed method. Cosmetic but misleading in logs/diagnostics.
- **Impact & likelihood:** Misleading error text for failed DELETE (delete partial exit). · always, on any non-2xx DELETE.
- **PoC:** n/a
- **Fix:** Use `Method::DELETE`.

### [Low] obolapi: unused `_req_timeout` field
- **Rust:** `crates/app/src/obolapi/client.rs:25` (`_req_timeout: Duration`)
- **Charon ref:** n/a (Rust-only)
- **Issue:** Stored but never read (timeout is applied to the reqwest client at build). Dead field, underscore-prefixed to silence warnings.
- **Impact & likelihood:** Maintainability noise only. · n/a.
- **PoC:** n/a
- **Fix:** Remove the field.

### [Low] monitoringapi current_epoch: negative chain age clamps to 0 vs Go's wrap
- **Rust:** `crates/app/src/monitoringapi/checker.rs:316-333` (`current_epoch`), `:323` `.to_std().unwrap_or(Duration::ZERO)`
- **Charon ref:** [`app/monitoringapi.go:146-151`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/monitoringapi.go#L146-L151) (`currentEpochFunc`: `chainAge := clock.Since(genesisTime); uint64(currentSlot)`)
- **Issue:** If `now < genesis_time` (genesis in the future), Go computes a negative `time.Duration`, integer-divides (still negative), then `uint64(...)` wraps to a huge epoch. Pluto clamps the negative duration to `ZERO` → epoch 0. Different value, but only pre-genesis. Pluto's behavior is saner; flagged as a parity divergence, not a defect.
- **Impact & likelihood:** Pre-genesis only (testnets/dev); readiness epoch differs from charon. · negligibly rare in production.
- **PoC:** n/a
- **Fix:** None required; optionally document the intentional divergence.

### [Low] SSE: reorg subscriber semantics diverge (try_send+prune vs synchronous blocking)
- **Rust:** `crates/app/src/sse/mod.rs:351-366` (`notify_chain_reorg`, `try_send` + drop-on-full warn + prune-on-closed)
- **Charon ref:** [`app/sse/listener.go:253-265`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/sse/listener.go#L253-L265) (`notifyChainReorg` calls each handler synchronously under the lock)
- **Issue:** Charon invokes subscriber callbacks synchronously (a slow subscriber blocks the listener and never loses an event). Pluto sends over bounded channels with `try_send`: a full subscriber **drops** the reorg event (warn) and a closed one is pruned. So under backpressure pluto silently loses reorg notifications where charon would block/deliver. Deliberate design divergence in the actor model.
- **Impact & likelihood:** A lagging reorg subscriber misses epochs. · only under sustained subscriber backpressure (uncommon; reorgs are infrequent).
- **PoC:** n/a
- **Fix:** Document the divergence; if delivery must be guaranteed, use `send().await` from a dedicated task or an unbounded channel with monitoring.

### [Low] SSE head-event slot not range-checked against i64 (saturates) vs Go's explicit reject
- **Rust:** `crates/app/src/sse/mod.rs:378` (`compute_delay`, `i64::try_from(slot).unwrap_or(i64::MAX)`)
- **Charon ref:** [`app/sse/listener.go:125-127`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/sse/listener.go#L125-L127) (`if slot > math.MaxInt64 { return errors.New(...) }`)
- **Issue:** Charon rejects a head-event slot exceeding `MaxInt64`; pluto saturates the conversion to `i64::MAX` and proceeds, producing a meaningless delay metric instead of an error. Untrusted BN input.
- **Impact & likelihood:** Corrupt delay metric for an absurd slot value; no panic, no crash. · only on a malicious/garbage beacon node slot > 2^63.
- **PoC:** n/a
- **Fix:** Reject slots `> i64::MAX` (return early) to match charon, or document the saturation as acceptable.

### [Info] health: high-cardinality label count cast is checked (`try_from`) vs Go's silent float cast
- **Rust:** `crates/app/src/health/checker.rs:103` (`i64::try_from(max_labels)?`)
- **Charon ref:** [`app/health/checker.go:118`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/health/checker.go#L118) (`float64(maxLabelsCount)`)
- **Issue:** Pluto propagates an error if a series somehow has > i64::MAX labels; Go silently casts. Pluto is strictly safer; non-issue, noted for completeness. (`max_label_count` correctly excludes the `le` bucket label for histograms — checker.rs:145-163 — matching the protobuf model.)
- **Impact & likelihood:** None in practice. · n/a.
- **PoC:** n/a
- **Fix:** None.

### [Info] health select: regex compiled per-call (label match) — minor inefficiency, behavior-correct
- **Rust:** `crates/app/src/health/select.rs:88-90` (`labels_contain`, `Regex::new(&c.value)...unwrap_or(false)`)
- **Charon ref:** [`app/health/select.go:98`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/health/select.go#L98) (`regexp.MatchString`)
- **Issue:** Both compile the regex on each label comparison (Go's `MatchString` also compiles per call), so this is parity, not a regression. Pluto additionally treats an invalid regex as "no match" (graceful) rather than erroring. Optional optimization: precompile selectors once. Pluto's invalid-regex handling is safer than a panic.
- **Impact & likelihood:** Negligible CPU on the 30s health tick. · n/a.
- **PoC:** n/a
- **Fix:** Optional: cache compiled regexes in the selector closure.

## Uncertain / needs-human
- **Grandine version-string ordering (Medium finding above):** confirming whether charon's `version.Parse("v2.0.0.rc0")` + `version.Compare` orders identically to pluto's `SemVer::parse("v2.0.0-rc0")` requires running both parsers. The pluto `core::version::SemVer` parse/compare semantics for hyphen vs dot pre-release were not exercised here. Phase 2 should diff actual parse + ordering for both literals against a real `Grandine/2.0.0` and `Grandine/2.0.0-rc0` input.
- **retry past-deadline (Medium finding above):** confirmed by inspection of the `to_std().ok()` → `None` path and `without_max_times()`, but the real-world trigger depends on which `deadline_fn`s pluto's wiring installs (whether any can return a past instant). The retry module itself is generic and currently has no production call sites in this crate; Phase 2 should check downstream callers (e.g. core/scheduler) for past-deadline inputs.
- **TLS / HTTP-scheme enforcement (obolapi + SSE):** neither pluto nor charon enforces HTTPS for the obol API or the beacon-node SSE stream (both accept plain HTTP). This is a shared design property, not a pluto regression, so it is not recorded as a finding — but a human should decide whether pluto should harden this beyond charon parity.

## Summary
The pluto `app` port is largely faithful; no Critical defects found. Highest-impact issue is the **privkeylock on-disk timestamp format** (Go RFC3339 string vs Rust integer seconds), which makes lock files mutually unreadable and can defeat the single-instance guard in mixed charon/pluto deployments. Other real divergences: obolapi re-parses validator_index with a silent default in the signed sort key (Medium); retry, obolapi cancellation, Grandine version warning, and missing `log.Filter()` are now Low because they are latent, warning-only, or log/shutdown-only. Several scanner-flagged "Critical/High" items (SSE backoff reset, SSE epoch-0 dedup, unrecognized-topic drop) were refuted on inspection as exact charon parity or sound deliberate design. Counts: 0C / 1H / 1M / 11L / 2I.
