# Audit: p2p  (9317 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/p2p/src/*` (p2p, config, gater, peer, sender? , relay, bootnode, name, k1, behaviours, quic_upgrade, force_direct, conn_logger, bandwidth, manet)
- **Charon:** `p2p/{p2p,config,gater,peer,sender,receive,relay,bootnode,name,ping,errors,metrics,k1}.go`
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `p2p.rs`/`config.rs` ↔ `p2p/{p2p,config}.go` (swarm setup)
- `gater.rs` ↔ `p2p/gater.go` (connection allow/deny)
- `peer.rs`/`name.rs` ↔ `p2p/{peer,name}.go` (ENR/identity)
- `relay`/`bootnode.rs` ↔ `p2p/{relay,bootnode}.go`; `k1.rs` ↔ `p2p/k1.go` (uses k1util)

## Crate-specific focus
- **Parity:** ENR/peer-identity derivation, connection-gater allowlist rules, circuit-relay config, protocol negotiation IDs, send/receive framing + message-size limits, peer scoring/ping.
- **Security (network attack surface):** gater enforcement, message-size caps (DoS), connection/stream limits, multiaddr validation, no panic on malformed peer input.
- **Quality:** libp2p swarm event handling, backpressure, async correctness.

## Summary

Scanned all 22 source files in `crates/p2p/src` against charon v1.7.1 `p2p/*.go`. Identity derivation (`peer.rs`/`name.rs`), key handling (`k1.rs`), manet IP classification, message-size caps (`proto.rs`, 128MB matching `pbio`), and the QUIC-upgrade/force-direct behaviours port faithfully (name parity is test-verified). The headline issue is a **gater parity gap**: pluto only enforces the cluster allowlist on inbound connections, while charon's `InterceptSecured` gates both directions. One Medium parity item remains: `filter_advertised_addresses` sorts+per-list-dedups (charon preserves order with a shared dedup map, test-confirmed). HTTP-resolved relay identities are trusted unvalidated, but this is Low after triage because it matches Charon and only affects non-default plaintext relay URLs. Notably, charon's `sender.go`/`receive.go` request/response layer — including stream read/send deadlines and error-storm suppression — is **not** ported into this crate; the framing primitives in `proto.rs` have no read timeout and consumers (peerinfo) don't add one. Severity: 0C / 1H / 1M / 5L / 2I.

---
## Findings

### [High] ConnGater does not gate outbound connections (parity divergence)
- **Rust:** `crates/p2p/src/gater.rs:155` (`ConnGater::handle_established_outbound_connection`)
- **Charon ref:** [`p2p/gater.go:57`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/gater.go#L57) (`InterceptSecured`)
- **Issue:** Go's `InterceptSecured(direction, id, …)` is invoked by go-libp2p for **both** inbound and outbound secured connections and rejects any peer id not in the cluster/relay allowlist regardless of direction. Pluto's gater only checks `is_peer_allowed` on the inbound path; `handle_established_outbound_connection` unconditionally returns `Ok(...)`. A closed gater therefore enforces the allowlist on inbound only.
- **Impact & likelihood:** A node that is tricked/configured into dialing an out-of-cluster peer (e.g. via a poisoned peerstore address, relay routing to a substituted peer id, or a buggy/forced dial) will complete the outbound connection even though the cluster gater should block it. Weakens the "limit connections to DV peers" guarantee. · Triggers whenever an outbound dial targets a non-allowlisted peer id; in normal operation we only dial known peers so exploitation requires an address/routing manipulation, but the defense-in-depth the Go gater provides is lost.
- **PoC:** CONFIRMED — scratch `scratch_outbound_not_gated` (closed gater allowing one peer; dial a non-allowlisted `stranger`) → inbound `handle_established_inbound_connection(stranger)` = `Err` (correctly denied), outbound `handle_established_outbound_connection(stranger)` = `Ok` (NOT gated). Charon `InterceptSecured` (gater.go:57) is direction-agnostic and rejects the stranger regardless of direction (`TestInterceptSecured` gater_test.go:21 exercises it via direction `0`/inbound but the func ignores direction); reasoned (Go not run). Pluto gates inbound only → parity gap confirmed.
- **Fix:** Mirror inbound logic in `handle_established_outbound_connection`: if `!is_peer_allowed(&peer)` push `PeerBlocked` and return `Err(ConnectionDenied::new(PeerNotAllowed(peer)))`.

### [Medium] filter_advertised_addrs sorts addresses and dedups per-list, diverging from charon
- **Rust:** `crates/p2p/src/utils.rs:105` (`filter_advertised_addresses`)
- **Charon ref:** [`p2p/p2p.go:134`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/p2p.go#L134) (`filterAdvertisedAddrs`); test [`p2p/p2p_internal_test.go:12`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/p2p_internal_test.go#L12) (`TestFilterAdvertisedAddrs`)
- **Issue:** Go preserves insertion order (external addrs first in given order, then internal) and dedups across **both** lists with one shared `dedup` map; private filtering applies only to internal. Pluto instead `sort()`s each list and `dedup()`s each list **independently**, then chains external→internal. Two divergences: (1) advertised-address ordering no longer matches Go (Go's `TestFilterAdvertisedAddrs` "duplicate public" expects `[priv1, pub1]` preserved order; Rust would emit sorted order); (2) an address present in both external and internal is deduped by Go (shared map) but NOT by Rust (per-list dedup), so it is advertised twice.
- **Impact & likelihood:** Peers receive a different / duplicated advertised address set than charon would produce; AddrsFactory output drives what remote peers dial. Duplicates waste dial attempts; reordered addrs change dial priority. Functional-equivalence break. · Always, whenever external and internal address sets overlap or ordering matters.
- **PoC:** CONFIRMED — scratch `scratch_filter_advertised_addrs_parity` replicates charon's "duplicate public" case (`p2p_internal_test.go:40`: external=[priv1,pub1], internal=[pub1,priv1], excludePrivate=true). Charon → `[priv1, pub1]`. Rust → `[pub1, priv1, pub1]` (`/ip4/1.1.1.1/tcp/80, /ip4/192.168.1.1/tcp/80, /ip4/1.1.1.1/tcp/80`). Test FAILED the equality assert → BOTH divergences proven: (1) external reordered by `sort()` (pub1 before priv1), and (2) pub1 NOT deduped across lists so it is advertised twice. Charon parity (`TestFilterAdvertisedAddrs`) reasoned (Go not run).
- **Fix:** Replicate Go: single ordered pass with a shared `HashSet<String>` dedup keyed on `addr.to_string()`, push external (no private filter) then internal (private filter when `exclude_internal_private`), preserving encounter order; drop the `sort()`.

### [Low] Relay HTTP-resolved multiaddrs are not validated against the configured relay identity
- **Rust:** `crates/p2p/src/bootnode.rs:159` (`resolve_relay` / `query_relay_addrs`)
- **Charon ref:** [`p2p/bootnode.go:87`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/bootnode.go#L87) (`resolveRelay`); [`p2p/bootnode.go:136`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/bootnode.go#L136) (`queryRelayAddrs`) [scanner cited `relay.go`; corrected — these live in `bootnode.go`]
- **Issue:** Functional parity with Go (both trust whatever peer id the HTTP relay endpoint returns), but neither validates that the resolved peer id matches an expected pin. The relay URL response (`enr:` or JSON multiaddr list) is parsed and the first/only `AddrInfo` is turned into a relay `Peer` whose id then becomes an allowlisted gater entry (`new_conn_gater` adds relays). Note the warning-only handling when the URL is not `https` (`bootnode.rs:102`): a plaintext-HTTP relay endpoint can be MITM'd to substitute an attacker peer id, which is then trusted as a relay and allowed through the gater.
- **Impact & likelihood:** A network attacker on a non-https relay-resolution path can inject an arbitrary relay peer id that the node will reserve circuits on and gate-allow, enabling traffic interception/relay-position attacks. · Only when a relay is configured by `http://` URL (default relays in `config.rs:12` are https, so default config is safe) and an active MITM is present.
- **PoC:** confirmed by inspection (no PoC needed) — `query_relay_addrs` (bootnode.rs:226) parses the HTTP body's `enr:`/JSON multiaddrs and `resolve_relay` (bootnode.rs:190-205) turns the single resolved `AddrInfo` into a relay `Peer` via `mutable.set(peer)` with no peer-id pin/verification; `new_relays` only `warn!`s on non-https (bootnode.rs:102-104). That relay `MutablePeer` is added to the gater allowlist (`is_peer_allowed` iterates `config.relays`, gater.rs:117-123). Charon `bootnode.go` (NewRelays:33, resolveRelay:87, queryRelayAddrs:136) is identical: warn-only on non-https, trusts the returned id, no pin → functional parity confirmed; the unvalidated-trust security note holds for both. Charon reasoned (Go not run).
- **Fix:** Match charon's posture at minimum (document the trust assumption); ideally reject non-https relay URLs rather than warn, and where a relay ENR/peer-id is known from the cluster lock, verify the resolved id against it before trusting.

### [Low] new_relays does not propagate per-relay multiaddr parse style differences; resolve_relay swallows first-error permanently
- **Rust:** `crates/p2p/src/bootnode.rs:159` (`resolve_relay`)
- **Charon ref:** [`p2p/relay.go:87`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/relay.go#L87) (`resolveRelay`)
- **Issue:** On a non-retryable error from `query_relay_addrs`, Go logs and `return`s, ending that relay's goroutine (same as Rust). But Rust computes `new_addrs = format!("{sorted_addrs:?}")` using the `Debug` representation of `Vec<Multiaddr>` for change detection, whereas Go uses `fmt.Sprint(addrs)` (the multiaddr `String()`). The Debug form differs from the string form; functionally still detects change, but the comparison key is not the canonical string and is more allocation-heavy. Minor.
- **Impact & likelihood:** No correctness impact (still detects changes); cosmetic divergence + needless formatting. · Always.
- **PoC:** n/a
- **Fix:** Compare on the canonical multiaddr strings (e.g. join `a.to_string()`), matching Go.

### [Low] proto.rs read paths have no inbound read timeout (charon sets stream ReadDeadline)
- **Rust:** `crates/p2p/src/proto.rs:26` (`read_length_delimited`), `:80` (`read_fixed_size_delimited_with_max`)
- **Charon ref:** [`p2p/receive.go:52`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/receive.go#L52) (`RegisterHandler` → `s.SetReadDeadline(receiveTimeout)`); [`p2p/sender.go:288`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/sender.go#L288) (`s.SetDeadline(sendTimeout)`)
- **Issue:** The framing primitives enforce a max message size (good, matches `pbio` `maxMsgSize` 128MB) but have no built-in timeout: a peer that sends the length prefix then stalls leaves `read_exact` pending indefinitely. Charon's request/response layer (`sender.go`/`receive.go`) — which is NOT ported into the pluto p2p crate — wraps every stream with `SetReadDeadline`/`SetDeadline` (5s recv, 7s send). The pluto consumers that do call these primitives (`crates/peerinfo/src/protocol.rs:267,285`) do **not** wrap the reads in `tokio::time::timeout`, so a stalled/malicious peer can pin an inbound handler stream open.
- **Impact & likelihood:** Slow-loris style stream exhaustion: held-open streams consume the yamux stream budget (`YAMUX_MAX_NUM_STREAMS = 2048`, `p2p.rs:113`) and handler tasks. · Only on malicious/buggy peers that are already gater-allowed (cluster peers/relays), so blast radius is limited to cluster members.
- **Fix:** Either add an optional deadline to the proto read helpers, or (preferred, matches charon's layering) ensure every stream-handler call site wraps reads/writes in `tokio::time::timeout` with charon's 5s/7s defaults. Track as a cross-crate item; the p2p primitive itself mirrors `pbio`.

### [Low] Unbounded ConnGater event queue
- **Rust:** `crates/p2p/src/gater.rs:71` (`ConnGater.events: VecDeque<Event>`), pushed at `:150`
- **Charon ref:** n/a (Go gater returns a bool; has no event queue)
- **Issue:** Every blocked inbound connection pushes a `PeerBlocked` event onto an unbounded `VecDeque`. The queue is only drained in `poll`, which is driven by the swarm; if events are produced faster than the swarm polls the gater (or the generated event is ignored downstream), the queue grows without bound.
- **Impact & likelihood:** Memory growth under a flood of disallowed inbound connection attempts on a publicly reachable node. · Low — swarm polls behaviours promptly and each event is one `PeerId`; would require a sustained high-rate connection flood to matter, and the connections are rejected anyway.
- **Fix:** Cap the queue (drop oldest / bounded ring) or drop the event entirely and rely on metrics/logging, since the `PeerBlocked` event has no consumer requirement.

### [Low] `peer()`/`set()` watch borrow holds lock across clone; subscribe semantics differ from Go callback model
- **Rust:** `crates/p2p/src/peer.rs:130` (`MutablePeer`)
- **Charon ref:** [`p2p/peer.go:102`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/peer.go#L102) (`MutablePeer`, mutex + `subs []func(Peer)`)
- **Issue:** Go's `MutablePeer.Set` snapshots subscribers under lock then invokes callbacks outside the lock; subscribers are push-based functions. Rust reimplements via `tokio::sync::watch`, which is pull-based: `RelayManager` consumes updates by polling a `WatchStream`. Behaviourally close, but `watch` only retains the **latest** value — if two `set()` calls land between polls, intermediate relay peers are coalesced/lost. For relays whose id changes on restart this is the intended "latest wins" semantics, so it's acceptable, but it is a semantic divergence from Go's "call all subscribers for every Set" worth recording.
- **Impact & likelihood:** A rapid relay id change sequence could skip an intermediate id; in practice relays change rarely and only the latest matters for routing. · Negligible.
- **Fix:** None required; document the latest-wins semantics. If every transition must be observed, use a broadcast channel instead of `watch`.

### [Info] backoff_delay computes cap via iterative multiply loop (O(retry_count))
- **Rust:** `crates/p2p/src/relay/dial.rs:108` (`backoff_delay`)
- **Charon ref:** `app/expbackoff` (`DefaultConfig`: base 1s, mult 1.6, jitter 0.2, max 120s)
- **Issue:** For large `retry_count` the function loops `retry_count` times multiplying by 1.6 until it hits max. Functionally correct and bounded (breaks at max), but `retry_count` is a `u32` that `saturating_add(1)`s forever, so the loop can run up to ~the number of retries before the `>= max` break — wasteful for a never-connecting target. Charon uses `math.Pow`-style direct computation.
- **Impact & likelihood:** Trivial CPU waste on long-failing dial campaigns; capped by the early `break` once `delay >= max` (≈11 iterations for base 1s→120s), so effectively bounded. · Negligible.
- **Fix:** Compute directly: `delay = min(base * 1.6^retry, max)` then apply jitter; or cap the loop iteration count.

### [Info] peer_name uses `to_base58()` while charon uses `id.String()`
- **Rust:** `crates/p2p/src/name.rs:367` (`peer_name`)
- **Charon ref:** [`p2p/name.go:374`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/name.go#L374) (`PeerName`, `s := id.String()`)
- **Issue:** Go's `peer.ID.String()` returns the base58btc multibase encoding (no multibase prefix for the legacy CIDv0-style id), which is what `to_base58()` produces in rust-libp2p. The crate's tests (`name.rs:401,412`) assert exact name parity against known peer ids ("happy-floor", "different-course"), confirming equivalence. Recorded only because the two method names differ and the hash is input-sensitive.
- **Impact & likelihood:** None — verified equivalent by the parity tests. · n/a.
- **Fix:** None.

## Uncertain / needs-human

- **Sender/Receive layer not in p2p crate.** Charon's `p2p/{sender,receive,errors}.go` (synchronous `Send`/`SendReceive`, `RegisterHandler`, the `senderHysteresis`-based error-storm suppression, relay-error single-retry `withRelayRetry`, and stream deadlines) have no counterpart in `crates/p2p`. Equivalent behaviour appears partially in consumer crates (`peerinfo`, `dkg`, `priority`). Need a human to confirm whether this is an intentional architectural split and whether the deadline/error-suppression semantics are preserved end-to-end (see Low finding on read timeouts). Out of strict p2p-crate scope but a functional-parity gap for the module as a whole.
- **RelayManager is a ground-up reimplementation, not a port.** Charon implements relay reservation/routing as lifecycle-hook goroutines (`relay.go` `NewRelayReserver`/`NewRelayRouter`) driving `circuit.Reserve` + peerstore `AddAddrs`. Pluto reimplements it as a libp2p `NetworkBehaviour` with an `Established`-stuck watchdog (`manager.rs:40`) and dial-state streams. The logic is plausible and well-tested (639-line test file) but is genuinely different control flow; a human/Phase-2 reviewer should confirm the watchdog (force-close after 60s in Established) and the `DialPeerConditionFalse` handling for `RelayServer` (`manager.rs:559`) don't wedge a relay permanently in `Dialing`/`Established` under reservation-refresh-denied conditions.
- **`route_known_peers` circuit-dial fan-out.** `manager.rs:301` builds circuit addrs for every known peer × every reserved relay on each relay reservation; no explicit cap. Likely fine for small cluster sizes (DV clusters are small) but worth a human eye for very large peer/relay counts.

---
## Delta audit 2026-07-02 — commits since 2026-06-30 baseline (#505)
STATUS: VERIFIED (no Critical/High/Medium — Phase 2 not required)

Scope: commit 63f6603 "consolidate exp backoff config into pluto_core::expbackoff". Only `bootnode.rs::query_relay_addrs` changed behaviorally: its inline backoff builder (`ExponentialBuilder::default()...with_jitter()`, **without** `without_max_times()`) was replaced by `pluto_core::expbackoff::fast()`. The old builder inherited backon's default `max_times = Some(3)`, so it gave up after 3 retries (~0.5s) and returned `Err` → `resolve_relay` logged and **permanently killed the relay-resolution task** on a brief transient outage. `expbackoff::fast()` sets `without_max_times()`, so it now retries until cancellation — this correctly restores Charon `bootnode.go:136` (`queryRelayAddrs`: "It retries until the context is cancelled", `for ctx.Err() == nil` retrying on send/non-2xx/read/parse errors). The fix is sound and matches Go's retryable-error set exactly. Constants (100ms/5s/1.6) unchanged. The jitter-formula divergence is recorded once against `expbackoff.rs` in core-duty.md.

### [Low] bootnode query_relay_addrs: infinite retry not cancellable mid-attempt (no ctx-bound request), diverging from Go
- **Rust:** `crates/p2p/src/bootnode.rs:235-299` (`query_relay_addrs`), fetch closure `:237-291`, cancel check `:238`
- **Charon ref:** [`p2p/bootnode.go:148-209`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/bootnode.go#L148) (`queryRelayAddrs`: `http.NewRequestWithContext(ctx, ...)` + `for ctx.Err() == nil` + `backoff()` selecting on `ctx.Done()`)
- **Issue:** #505 correctly makes this retry forever (was capped at 3). But cancellation is only observed by the `cancel.is_cancelled()` check at the **top of the fetch closure** — it is not wired into the request or the backoff sleep. `resolve_relay` awaits `query_relay_addrs(...).await` directly (no outer `select!` against cancel), the reqwest client is built with `reqwest::Client::new()` (**no request timeout**), and backon's inter-attempt sleep is not cancel-aware. So on shutdown/cancel: (a) if between attempts, the task waits out the full backoff interval (up to ~2× the 5s cap with backon jitter) before the next fetch sees the token; (b) if a request is in-flight against a black-holed/slowloris endpoint, `send().await` (or `text().await`) hangs **indefinitely** and the task never terminates. Go binds the request to `ctx` and its `backoff()` selects on `ctx.Done()`, so cancellation aborts both promptly. Before #505 the task died after 3 retries so this was bounded; the now-unbounded loop makes the hang reachable in steady state.
- **Impact & likelihood:** Delayed or hung graceful shutdown for http(s)-URL relays (a relay task that cannot be stopped ties up a task slot until process exit). · Only for relays configured by http(s) URL (default relays are static multiaddrs → not affected), and only on cancel while erroring/hung; the indefinite case needs an endpoint that accepts the connection then stalls.
- **PoC:** n/a (Low — confirmed by inspection: `reqwest::Client::new()` has no timeout, `resolve_relay` awaits without `select!` on the token; Go side reasoned from `bootnode.go:148-209`)
- **Fix:** Bind the reqwest request to the cancel token — wrap the `fetch` HTTP calls in `tokio::select!` against `cancel.cancelled()` (or set a per-request timeout on the client), and/or `select!` the whole `query_relay_addrs` await in `resolve_relay` against the token — so an in-flight request and inter-attempt sleep both abort on cancellation, matching Go's ctx-bound request.

### Delta summary
#505's bootnode change (add `without_max_times` via `expbackoff::fast()`) is a correct fix restoring Go's retry-until-cancel semantics. One Low surfaced by it: the now-unbounded retry loop is only cancel-checked between attempts and uses a no-timeout, non-ctx-bound reqwest request, so shutdown can be delayed (≈backoff interval) or indefinite (hung endpoint) — vs Go's ctx-bound request.
