# Audit: relay-server  (1721 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/relay-server/src/*` (config, p2p, web, metrics, error, utils)
- **Charon:** `cmd/relay.go`, `p2p/relay.go`, `p2p/bootnode.go`
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `p2p.rs` ↔ `p2p/relay.go` (libp2p circuit-relay v2)
- `web.rs` ↔ http status/ENR endpoints; `config.rs` ↔ `cmd/relay.go` flags

## Crate-specific focus
- **Parity:** relay reservation/circuit limits, web status + ENR endpoints, bootnode ENR serving.
- **Security:** relay resource limits (reservations/DoS), web endpoint exposure, rate limiting.
- **Quality:** server lifecycle, graceful shutdown.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [High] Relay rate limiters disabled — empty reservation/circuit rate-limiter vecs remove all per-peer/per-IP throttling
- **Rust:** `crates/relay-server/src/config.rs:49-64` (`create_relay_config`) — `reservation_rate_limiters: vec![]`, `circuit_src_rate_limiters: vec![]`
- **Charon ref:** [`cmd/relay/p2p.go:61-72`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L61-L72) (`startP2P`) — uses `relay.DefaultResources()` then `relay.New(... WithResources(relayResources))`, keeping the default rate limiters intact.
- **Issue:** rust-libp2p `relay::Config::default()` (libp2p-relay-0.21.1 `behaviour.rs:124-167`) ships four rate limiters: per-peer reservations (30/2min), per-IP reservations (60/min), per-peer circuits (30/2min), per-IP circuits (60/min). By building `relay::Config` literally instead of starting from `Default`, pluto sets both rate-limiter vectors to empty. In the behaviour, the rate-limit clause is `!self.config.<…>_rate_limiters.iter_mut().all(|l| l.try_next(...))` (`behaviour.rs:419-426`, `542-548`); `Iterator::all` over an empty iterator is `true`, so the deny branch is never taken — **all rate limiting is off**. Charon deliberately keeps go-libp2p's default rate limiters (DefaultResources). Also note Go enforces `MaxReservationsPerIP = MaxResPerPeer` (per-IP reservation cap); rust-libp2p has no per-IP reservation cap field, and pluto removed the per-IP rate limiters that would have partially compensated.
- **Impact & likelihood:** Internet-facing relay. A single peer/IP can churn reservations and circuit-open requests at line rate (bounded only by the hard caps `max_reservations`=16384 and `max_circuits`/`max_circuits_per_peer`=512), with no per-peer/per-IP rate ceiling. Enables cheap reservation/circuit-slot exhaustion and connection-handler churn (DoS) that Charon throttles by default · always exploitable on a reachable relay.
- **PoC:** CONFIRMED — scratch test `scratch_rate_limiters_empty` (built `create_relay_config`, asserted limiter vecs) → both `reservation_rate_limiters.len()==0` and `circuit_src_rate_limiters.len()==0`, while `relay::Config::default()` has 2 + 2 limiters. Verified rate-limit clause `!…rate_limiters.iter_mut().all(|l| l.try_next(...))` at `libp2p-relay-0.21.1/src/behaviour.rs:421-427` (reservations) and `:544` (circuits): empty `.all()`==`true` → `!true`==`false` → deny-on-rate branch never taken (reasoned, Go/libp2p runtime not run). Charon keeps go-libp2p defaults via `DefaultResources()` ([cmd/relay/p2p.go:61](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L61)).
- **Fix:** Start from `relay::Config::default()` and override only the fields that must differ (mirror Charon: `max_reservations = max_conns`, `max_reservations_per_peer`/`max_circuits` = `max_res_per_peer`, data=32MB, durations), preserving `reservation_rate_limiters` / `circuit_src_rate_limiters`. Do not pass empty vecs.

### [Medium] ENR HTTP server has no request/header read timeout (slowloris DoS)
- **Rust:** `crates/relay-server/src/web.rs:142-169` (`enr_server`) — `TcpListener::bind` + `axum::serve(listener, router)` with no timeout layer.
- **Charon ref:** [`cmd/relay/relay.go:93`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/relay.go#L93) (HTTP), `:107` (monitoring), `:125` (debug) — every `http.Server` sets `ReadHeaderTimeout: time.Second`.
- **Issue:** axum/hyper apply no default read or header timeout. Charon sets `ReadHeaderTimeout: 1s` on all three servers specifically to bound slow-header attacks. Pluto's ENR server (publicly reachable when `--http-address` is set to a non-loopback addr) accepts connections with unbounded header/body read time and no concurrency cap.
- **Impact & likelihood:** A handful of slow-sending TCP clients can hold connections open indefinitely, exhausting the accept loop / file descriptors on the ENR endpoint · triggers whenever the HTTP port is exposed and an attacker can reach it.
- **PoC:** CONFIRMED by inspection — `enr_server` (web.rs:142-169) is bare `TcpListener::bind` + `axum::serve(listener, router)`; `grep` of `crates/relay-server/{src,Cargo.toml}` shows no `tower_http`/`TimeoutLayer`/`ConcurrencyLimitLayer` and no per-request timeout (the two `tokio::time::timeout` uses in p2p.rs are shutdown-join timeouts). `axum::serve`/hyper apply no default read-header/read timeout. Charon sets `ReadHeaderTimeout: time.Second` on all three servers (relay.go:93,107,125) (reasoned, Go not run). Note: Charon's protection is header-read-only (Go also has no default body/idle timeout); pluto has none.
- **Fix:** Wrap the router with `tower_http::timeout::TimeoutLayer` (or a header-read timeout) ~1s to match Charon, and consider a `ConcurrencyLimitLayer`.

### [Medium] `max_circuits_per_peer` set to 512 instead of go-libp2p default — per-peer circuit cap far higher than Charon
- **Rust:** `crates/relay-server/src/config.rs:58-59` (`create_relay_config`) — `max_circuits = max_res_per_peer`, `max_circuits_per_peer = max_res_per_peer` (512 by default).
- **Charon ref:** [`cmd/relay/p2p.go:61,67`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L61) — `relayResources := relay.DefaultResources()` then only `relayResources.MaxCircuits = config.MaxResPerPeer`. `MaxCircuitsPerPeer` is **not** overridden, so it stays at go-libp2p's default (`MaxCircuitsPerPeer = 4`).
- **Issue:** Pluto overrides both `max_circuits` and `max_circuits_per_peer` to `max_res_per_peer` (512). Charon leaves `MaxCircuitsPerPeer` at the small default (4) and only raises the global `MaxCircuits`. So one source peer can hold up to 512 simultaneous relayed circuits in pluto vs 4 in Charon. The inline `todo(varex83)` comment flags this exact uncertainty. (The author also conflated the global cap with the per-peer cap.)
- **Impact & likelihood:** A single peer can monopolize circuit slots (up to `max_circuits`), starving other cluster members of relayed connectivity — a per-peer fairness/DoS regression vs Charon · whenever one peer opens many circuits.
- **PoC:** CONFIRMED — scratch test `scratch_max_circuits_per_peer` → `max_circuits_per_peer == 512` and `max_circuits == 512` (== `max_res_per_peer`), while `relay::Config::default().max_circuits_per_peer == 4[`. Charon ([cmd/relay/p2p.go:61](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L61),67) leaves `](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L61)MaxCircuitsPerPeer` at the go-libp2p default (4) and only sets `MaxCircuits = MaxResPerPeer` (reasoned, Go not run). Confirms both the inflated per-peer cap and the global/per-peer conflation.
- **Fix:** Mirror Charon: keep `max_circuits_per_peer` at the libp2p default (4) — i.e. base on `relay::Config::default()` and only set `max_circuits = max_res_per_peer`. Resolve the `todo`.

### [Medium] `max_circuit_duration` is 1 minute vs Charon's 1 hour
- **Rust:** `crates/relay-server/src/config.rs:60` (`create_relay_config`) — `max_circuit_duration: Duration::from_secs(ONE_MINUTE_SECONDS)` (60s).
- **Charon ref:** [`cmd/relay/p2p.go:63`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L63) — `relayResources.Limit.Duration = time.Hour` (the circuit data/duration limit).
- **Issue:** Go sets the *circuit* duration limit to 1 hour; pluto sets it to 1 minute. (rust-libp2p default is 2 min; pluto further lowered it.) A relayed connection (hole-punch fallback / direct relay path between cluster peers) is force-closed after 60s in pluto, vs up to 1 hour in Charon.
- **Impact & likelihood:** Functional divergence: long-lived relayed cluster connections are torn down every minute, forcing constant circuit re-establishment; degrades connectivity for peers that depend on the relay data path · always, for any circuit lasting >60s. (Not a security issue — pluto is *more* restrictive; flagged for parity.)
- **PoC:** CONFIRMED — scratch test `scratch_max_circuit_duration` → `max_circuit_duration == 60s` (`ONE_MINUTE_SECONDS`), `!= 3600s`. Charon sets `relayResources.Limit.Duration = time.Hour` ([cmd/relay/p2p.go:63](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L63)); libp2p default is 120s (reasoned, Go not run).
- **Fix:** Set `max_circuit_duration` to 1 hour to match Charon (`relayResources.Limit.Duration = time.Hour`).

### [Low] Monitoring server bind/serve failure is swallowed, not propagated to shutdown
- **Rust:** `crates/relay-server/src/web.rs:180-188` (`monitoring_server`) — `MetricsExporter::… .start(bind_addr).await.unwrap_or_else(|e| warn!(...))`.
- **Charon ref:** [`cmd/relay/relay.go:97-110`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/relay.go#L97-L110) — monitoring server runs `serverErr <- server.ListenAndServe()`; any error is sent on the shared error channel and the relay's main `select` returns it (process exits).
- **Issue:** In Charon a monitoring-server failure (e.g. port in use) terminates the relay via `serverErr`. In pluto the monitoring server only logs a warning; the relay keeps running with metrics silently unavailable. The ENR server *does* report bind/serve errors via `server_errors` (`web.rs:144-147,166-168`), so the two HTTP servers behave inconsistently.
- **Impact & likelihood:** Operator may run a relay believing monitoring is up when it failed to bind; no crash/alert. Masks a failure (contra "transparent failures") · only when the monitoring port is unavailable.
- **PoC:** n/a
- **Fix:** Thread a `server_errors` sender into `monitoring_server` and emit on start failure, matching the ENR server and Charon.

### [Low] `--debug-address` flag accepted but no pprof/debug server is started
- **Rust:** `crates/relay-server/src/config.rs:28` (`debug_addr` field) and `crates/cli/src/commands/relay.rs:108,188-194` (flag wired into config) — `debug_addr` is never read in `relay-server`.
- **Charon ref:** [`cmd/relay/relay.go:112-128`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/relay.go#L112-L128) — when `DebugAddr != ""`, serves `net/http/pprof` (`/debug/pprof/*`).
- **Issue:** Pluto exposes the `--debug-address` flag and stores it in `Config.debug_addr`, but `run_relay_p2p_node` never starts a debug server. The flag is silently inert. (Functionally this is *safer* — no pprof exposure — but it is a parity gap and a misleading no-op flag.)
- **Impact & likelihood:** Operator sets `--debug-address` expecting pprof; nothing listens. Misleading UX; no security exposure · always when the flag is used.
- **PoC:** n/a
- **Fix:** Either implement a debug/pprof-equivalent server gated on `debug_addr`, or remove the flag/field and document the divergence.

### [Low] Per-peer `active_connections` gauge can underflow on unmatched ConnectionClosed
- **Rust:** `crates/relay-server/src/p2p.rs:227-232` (`handle_swarm_event`, `ConnectionClosed`) — `RELAY_METRICS.active_connections[&labels].dec_by(1)` with no guard.
- **Charon ref:** [`cmd/relay/p2p.go:149-161`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L149-L161) (`monitorConnections`) — decrements an in-map counter; per-peer state is deleted at 0 (`:164-172`), and the gauge is keyed by peer *name* not raw peer ID.
- **Issue:** Every `ConnectionEstablished` does `inc_by(1)` and every `ConnectionClosed` does `dec_by(1)`, but there is no invariant guaranteeing a 1:1 pairing per peer label (e.g. close events for connections established before metrics labels existed, or label/name collisions). vise `Gauge` is i64 so this won't panic, but the per-peer gauge can go negative, producing misleading/garbage metrics. Pluto also never removes stale per-peer label series (unbounded label cardinality over the relay's lifetime — minor memory growth), whereas Charon deletes idle peers from its map.
- **Impact & likelihood:** Negative / unbounded-cardinality metrics; monitoring noise, slow memory growth in the metrics registry on a long-running public relay · low, accumulates over time.
- **PoC:** n/a
- **Fix:** Track active count and only decrement when >0 (or mirror Charon's per-peer state map with deletion at zero); prune label series for fully-disconnected peers.

### [Info] CLI help says reservations valid 30min but actual reservation_duration is 1 hour
- **Rust:** `crates/relay-server/src/config.rs:53` (`reservation_duration = ONE_HOUR_SECONDS`) vs `crates/cli/src/commands/relay.rs:159` (help: "each valid for 30min").
- **Charon ref:** [`cmd/relay.go:53`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay.go#L53) — identical stale help text ("each valid for 30min") while `DefaultResources` reservation TTL is 1h.
- **Issue:** Help text is inaccurate (faithfully copied from Charon's equally-stale text). Reservations actually last 1 hour.
- **Impact & likelihood:** Cosmetic; operator confusion only · n/a.
- **PoC:** n/a
- **Fix:** Update help text to "each valid for 1h" (optionally upstream-divergent, but accurate).

### [Info] No required-TCP-address check; Charon errors when TCP addrs empty
- **Rust:** `crates/relay-server/src/p2p.rs:30-67` (`run_relay_p2p_node`) — no check; `apply_config` (`crates/p2p/src/p2p.rs:357-361`) only `warn!`s when no listen addrs.
- **Charon ref:** [`cmd/relay/p2p.go:31-33`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/relay/p2p.go#L31-L33) (`startP2P`) — `if len(config.P2PConfig.TCPAddrs) == 0 { return errors.New("p2p TCP addresses required") }`.
- **Issue:** A circuit-relay-v2 server must accept inbound TCP; Charon hard-fails fast if no TCP listen addresses are configured. Pluto starts anyway and only warns, yielding a relay that accepts no inbound connections.
- **Impact & likelihood:** Misconfigured relay silently does nothing useful instead of failing fast · only on misconfiguration.
- **PoC:** n/a
- **Fix:** Return an error when `config.p2p_config.tcp_addrs` is empty, matching Charon.

## Uncertain / needs-human
- **`max_conns` → `max_reservations` mapping & absent swarm connection limit.** Charon sets `MaxReservations = MaxConns` (16384) under a `NullResourceManager` (no rcmgr fd/memory accounting). Pluto maps `max_conns → max_reservations` identically and rust-libp2p has no resource manager either, so reservations are the only cap. Whether 16384 reservations with no global fd/memory guard is acceptable for an internet-facing relay (Go has the same posture) is a design question — flag for human review of whether a swarm-level `ConnectionLimits` should be added beyond Charon parity.
- **IPv6 ENR support.** `enr_handler` (`web.rs:227-285`) and `utils::extract_ip_and_*` only handle `Ipv4Addr`; an IPv6-only listener can never produce an ENR (handler returns 500). Charon's `manet.ToNetAddr` + `enr.WithIP` handle both. This is rooted in `eth2util::enr` being IPv4-only (`EnrEntry::Ipv4`), so it may be an intentional/global scope decision rather than a relay-server bug — confirm whether IPv6 relays are in scope.
- **`max_circuit_bytes` semantics (32MB).** Pluto sets `max_circuit_bytes = 32MB` matching Charon's `Limit.Data = 32MB`. Need confirmation that rust-libp2p's `max_circuit_bytes` counts the same direction/total as go-libp2p's `Limit.Data` (assumed equivalent by name; not verified against go-libp2p source, which was not available locally).

## Summary
relay-server is a faithful structural port but diverges from Charon on resource-limit defaults in ways that matter for an internet-facing service. The headline issue (High): `create_relay_config` builds `relay::Config` from scratch with **empty rate-limiter vectors**, silently disabling the per-peer/per-IP reservation and circuit rate limiting that go-libp2p (and thus Charon, via `DefaultResources`) enables by default — removing the relay's primary DoS throttle. Two further parity divergences raise per-peer circuit fairness limits far above Charon (`max_circuits_per_peer` 512 vs 4) and shorten circuit lifetime (60s vs 1h). The ENR HTTP server also lacks Charon's 1s header-read timeout (slowloris surface). Remaining findings are lower-severity parity/quality gaps (swallowed monitoring failure, inert `--debug-address`, gauge underflow, stale help text).
