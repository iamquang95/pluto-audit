# Audit: parsigex  (1983 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/parsigex/src/*` (behaviour, handler, protocol, error)
- **Charon:** `core/parsigex/{parsigex,memory}.go`
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `protocol.rs`/`handler.rs` ↔ `core/parsigex/parsigex.go` (partial-sig exchange protocol)
- `behaviour.rs` ↔ libp2p behaviour wiring

## Crate-specific focus
- **Parity:** partial-sig exchange wire format, broadcast-to-peers fanout, dedup, protocol ID.
- **Security:** verify sender identity, validate partial sig before accept, message-size limits.
- **Quality:** libp2p request/response or gossip handler correctness.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Medium] Broadcast does not dial disconnected cluster peers (no on-demand dial); charon's SendAsync dials + retries
- **Rust:** `crates/parsigex/src/behaviour.rs:291` (`Behaviour::handle_command`)
- **Charon ref:** [`p2p/sender.go:337`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/sender.go#L337) (`Send`, via `NewStream`) and [`p2p/sender.go:152`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/sender.go#L152) (`SendAsync`); wired at [`app/app.go:583`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/app.go#L583)
- **Issue:** Pluto only sends to peers that already have an active connection (`connections_to_peer(&peer).is_empty()` → skip + record failure). Charon production wires `sendFunc = sender.SendAsync`, whose `Send` calls `p2pNode.NewStream(...)` which **dials the peer on demand** (with relay retry via `withRelayRetry`). A momentarily-disconnected cluster peer therefore never receives the partial-sig set in pluto, whereas charon establishes the connection and delivers.
- **Impact & likelihood:** A cluster peer that is briefly disconnected when a duty broadcasts misses that peer's partial signature, degrading liveness / threshold aggregation for that duty · only when a peer is transiently disconnected at broadcast time (mitigated but not eliminated by pluto's relay/connection manager keeping cluster peers connected).
- **PoC:** CONFIRMED — scratch `finding1_no_on_demand_dial_for_disconnected_peer` (drove `Behaviour::poll` with one connected + one known-but-store-absent peer) → `dial=0 notify_connected=1 notify_disconnected=0 berr_disconnected=1`: the disconnected known peer emits **zero** `ToSwarm::Dial`, gets no `NotifyHandler` send, only a `BroadcastError`; connected peer gets exactly one `NotifyHandler`. Charon side reasoned (Go not run): `app.go:583` wires `sender.SendAsync` → `Send` → `p2pNode.NewStream(WithAllowLimitedConn,...)` which dials on demand (`sender.go:160,345`), with one relay retry (`withRelayRetry`, `sender.go:184`). Pluto has no dial-on-send and no relay retry.
- **Fix:** Have the broadcast path trigger a dial for known-but-unconnected cluster peers (e.g. emit `ToSwarm::Dial` and defer the `Send` until connected) instead of immediately recording a failure, matching charon's dial-on-send semantics.

### [Medium] Broadcast is synchronous & failure-propagating; charon production fire-and-forgets (SendAsync) and never fails the broadcast on a send error
- **Rust:** `crates/parsigex/src/behaviour.rs:344` (`finish_broadcast_result`) / `behaviour.rs:146` (`Handle::broadcast` / `broadcast_and_wait`)
- **Charon ref:** [`core/parsigex/parsigex.go:117`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/parsigex/parsigex.go#L117) (`Broadcast`) + [`p2p/sender.go:152`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/sender.go#L152) (`SendAsync` returns `nil` immediately, sends in goroutine)
- **Issue:** Pluto tracks per-peer outbound success/failure and reports `BroadcastFailed` / returns `Err` if *any* peer fails. Charon production passes `SendAsync`, which always returns `nil`; charon's `Broadcast` loop thus never observes a per-send error, never aborts, and returns `nil` (fire-and-forget with background relay retry and error-storm log filtering). Pluto callers waiting on `broadcast_and_wait` will see failures that charon callers never surface, and there is no relay-retry-once equivalent (`withRelayRetry`, [`p2p/sender.go:184`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/sender.go#L184)).
- **Impact & likelihood:** Divergent error semantics — pluto may treat a partially-delivered broadcast as a hard failure and (depending on caller) retry/abort the duty differently than charon; also missing the single relay retry · whenever any single peer send fails.
- **PoC:** CONFIRMED — scratch `finding2_broadcast_fails_when_a_peer_disconnected` (drove `Behaviour::poll` after enqueuing a broadcast to an unreachable known peer) → `broadcast_failed_events=1 broadcast_error_events=1`, and `broadcast_and_wait` returned `Err(BroadcastFailed { request_id: 0, error: Io("peer … is not connected") })`. Pluto synchronously surfaces the per-peer failure as a terminal `BroadcastFailed` event + `Err`. Charon side reasoned (Go not run): `app.go:583` passes `sender.SendAsync`, which spawns a goroutine and returns `nil` immediately (`sender.go:152-166`); `Broadcast`'s loop (`parsigex.go:130-141`) therefore never observes a send error and returns `nil` (fire-and-forget). No relay-retry-once equivalent exists in pluto (verified absent in `behaviour.rs`).
- **Fix:** Confirm intended semantics with the core-workflow caller. If charon parity is required, make broadcast fire-and-forget (do not fail the duty on individual peer send errors) and add a one-shot relay retry; otherwise document the deliberate divergence.

### [Low] Receive/send timeout differs from charon defaults (20s vs 5s receive / 7s send)
- **Rust:** `crates/parsigex/src/behaviour.rs:207` (`Config::new`, `timeout: Duration::from_secs(20)`); applied in `handler.rs:110,135`
- **Charon ref:** [`p2p/sender.go:28-29`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/sender.go#L28-L29) (`defaultRcvTimeout = 5s`, `defaultSendTimeout = 7s`); parsigex passes no timeout opts at [`app/app.go:583`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/app.go#L583), so defaults apply
- **Issue:** Pluto uses a single 20s timeout for both inbound recv and outbound send. Charon production uses 5s receive / 7s send. A single value also can't reproduce charon's asymmetric send-vs-receive deadlines.
- **Impact & likelihood:** Slower failure detection; a slow/stalled peer ties up an inbound future ~4x longer than charon, slightly enlarging the (cluster-peer-only) resource-hold window · always (every stream uses this timeout).
- **PoC:** n/a
- **Fix:** Default receive timeout to 5s and send to 7s (or expose both), matching charon; keep `with_timeout` for overrides.

### [Low] decode_message validates only top-level proto fields; charon recursively rejects any nil nested message field (protonil.Check)
- **Rust:** `crates/parsigex/src/protocol.rs:26` (`decode_message`)
- **Charon ref:** [`p2p/receive.go:90`](https://github.com/ObolNetwork/charon/blob/v1.7.1/p2p/receive.go#L90) (`protonil.Check(req)`) → [`app/protonil/protonil.go:25`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/protonil/protonil.go#L25) (`Check`, recursive over all message/map/list fields)
- **Issue:** Pluto checks only that `duty` and `data_set` are `Some`, then relies on `Duty::try_from` / `ParSignedDataSet::try_from` for deeper validation. Charon's `protonil.Check` recursively rejects *any* nil non-optional nested message field across the whole proto before the handler runs. Coverage is largely equivalent for the fields the `TryFrom` conversions touch, but malformed-but-decodable nested messages not exercised by the conversion could slip past where charon would reject early.
- **Impact & likelihood:** Possible acceptance of a structurally-incomplete message that charon rejects up front; effect is bounded because downstream verification still runs · only on crafted malformed input from a cluster peer.
- **PoC:** n/a
- **Fix:** Confirm the `TryFrom` paths reject every nil nested field protonil would; if gaps exist, add explicit nil-field checks or a protonil-equivalent pass.

### [Info] Pluto accepts parsigex streams only from known cluster peers (stricter than charon's global handler)
- **Rust:** `crates/parsigex/src/behaviour.rs:249` (`connection_handler_for_peer`: unknown peer → `dummy::ConnectionHandler`)
- **Charon ref:** [`core/parsigex/parsigex.go:45`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/parsigex/parsigex.go#L45) (`RegisterHandler`, global `SetStreamHandlerMatch`); handler ignores sender `peer.ID` (`parsigex.go:69` param `_`)
- **Issue:** Pluto installs the real handler (and advertises the protocol) only for peers in `known_peers`; unknown peers get a dummy handler and cannot open parsigex streams. Charon registers the handler for all peers and lets `verifyFunc` reject non-cluster pubkeys. Neither side binds the *sending peer identity* to the share index (both verify only the BLS partial sig against the claimed `shareIdx`), so a malicious cluster peer could relay another peer's valid partial sig in both implementations.
- **Impact & likelihood:** Pluto reduces DoS/parsing surface vs charon (positive divergence); no functional regression · always.
- **PoC:** n/a
- **Fix:** None required; note as an intentional hardening. If strict parity is desired, document the difference.

### [Info] Unbounded per-connection inbound future queue
- **Rust:** `crates/parsigex/src/handler.rs:86,107` (`active_futures: FuturesUnordered`, pushed per inbound stream)
- **Charon ref:** n/a (libp2p Go enforces per-conn stream limits in the muxer)
- **Issue:** Each fully-negotiated inbound stream pushes a future into an unbounded `FuturesUnordered`; there is no cap on concurrent in-flight inbound verifications per connection. Bounded in practice by libp2p substream limits and by the known-peer gate (only cluster peers reach the handler), but no explicit backpressure.
- **Impact & likelihood:** A cluster peer opening many concurrent streams could grow the future set and verification load · negligible in normal operation (cluster peers only; muxer-limited).
- **PoC:** n/a
- **Fix:** Optionally cap concurrent inbound futures per connection; low priority given the known-peer gate.

## Uncertain / needs-human
- **Intended broadcast semantics (parity vs deliberate change).** Charon production uses `SendAsync` (fire-and-forget, dials on demand, relay-retry-once, async). Pluto is synchronous, connected-peers-only, failure-propagating. Both Medium findings hinge on whether pluto intentionally chose stricter synchronous semantics. Needs confirmation from the core-workflow caller (the `parsigdb`/`sigagg` wiring) on whether `BroadcastFailed`/`Err` is acted on (retry, abort duty) — that determines real-world impact and whether the dial-on-demand gap is a true regression.
- **Connection-manager guarantee.** The Medium "no on-demand dial" finding's likelihood depends on whether pluto's relay/connection manager keeps *all* cluster peers continuously connected during duty windows. If it provably does, downgrade to Low.

## Summary
parsigex is a faithful port of the wire format (protocol ID `/charon/parsigex/2.0.0`, unsigned-varint length-delimited framing matching go-msgio `pbio`, 128 MiB cap) and the validate→gate→verify→subscribe pipeline. The two material divergences are in broadcast semantics: pluto is synchronous, sends only to already-connected peers (no on-demand dial), and fails the broadcast on any peer error, whereas charon production fire-and-forgets via `SendAsync`, dials disconnected peers on demand, and retries relay errors — risking missed partial-sig delivery to transiently-disconnected peers and divergent failure handling. Minor: timeout defaults (20s vs 5s/7s) and shallower top-level proto nil-checking vs charon's recursive `protonil.Check`. Pluto's known-peer-only handler gating is a positive hardening; no crypto/`unsafe`/overflow issues found.

---
## Delta audit 2026-07-02 — commits since 2026-06-30 baseline (#511)
STATUS: VERIFIED (no Critical/High/Medium — Phase 2 not required)

Commit `8c6325c` adds `new_eth2_verifier` (behaviour.rs:62-91), the `VerifyError::InvalidSignature { duty, source }` variant (error.rs:60-69, replacing the old free-form `Other(String)`), re-export in lib.rs, and 4 tests. The verifier is a near line-for-line port of charon `NewEth2Verifier` (parsigex.go:150-175): same lookups, same order, same error strings, and it delegates the domain/epoch/BLS work to the already-audited core `verify_eth2_signed_data`. No Critical/High/Medium divergences found.

### [Info] Verifier is a faithful, fail-closed port of `NewEth2Verifier` (positive)
- **Rust:** `crates/parsigex/src/behaviour.rs:62` (`new_eth2_verifier`)
- **Charon ref:** `core/parsigex/parsigex.go:150` (`NewEth2Verifier`)
- **Issue:** Not a defect — recorded for parity coverage. Step order and rejection semantics match charon exactly: `pubShares[pubkey]` miss → `UnknownPubKey` ("unknown pubkey, not part of cluster lock"); `pubShares[share_idx]` miss → `InvalidShareIndex` ("invalid shareIdx"); non-eth2 `SignedData` (`as_eth2_signed_data` → `None`, mirroring Go's `data.(core.Eth2SignedData)` assertion) → `InvalidSignedDataFamily` ("invalid eth2 signed data") — i.e. an unsupported/raw signed-data family is **rejected (fail-closed)**, same as charon; verify failure → `InvalidSignature`. Direction parity also holds: verification runs on INBOUND sigs only (invoked per-entry in `handler.rs:262-265`, before `Received`/subscriber notification, so no unverified sig reaches parsigdb), and `Broadcast`/`handle_command` does not verify — identical to charon (`handle` verifies, `Broadcast` does not). Threshold/pubshare lookup keyed by the message's `share_idx` into `HashMap<u64,PublicKey>` matches `map[int]tbls.PublicKey`.
- **Impact & likelihood:** None (correct behavior) · n/a
- **PoC:** n/a
- **Fix:** None.

### [Info] `InvalidSignature` conflates transient beacon-node lookup failures with genuine BLS-verify failures (charon-parity, fail-closed)
- **Rust:** `crates/parsigex/src/behaviour.rs:86` (`new_eth2_verifier`, `map_err(|source| VerifyError::InvalidSignature{..})`) via `core/eth2signeddata.rs:60` (`verify_eth2_signed_data`)
- **Charon ref:** `core/eth2signeddata.go` (`VerifyEth2SignedData`) + `eth2util/signing/signing.go:107` (`Verify`→`GetDataRoot`); wrapped at `core/parsigex/parsigex.go:170`
- **Issue:** `verify_eth2_signed_data` resolves the signing epoch (`epoch_from_slot`) and domain (`get_data_root`) via beacon-node API calls; a transient network/beacon error surfaces as `Eth2SignedDataError::{Helper,Signing}` and is folded into `InvalidSignature`, then flattened by `handler.rs:265` to `Failure::InvalidPartialSignature` — so a *valid* partial signature is rejected (and the whole inbound set dropped) on a transient beacon failure. Charon does the identical thing (`VerifyEth2SignedData` + `signing.Verify` both call `eth2Cl`; `handle` wraps as "invalid partial signature"), so this is parity-preserving and errs toward rejection (safe direction), not acceptance.
- **Impact & likelihood:** A cluster peer's valid partial sig is discarded during a beacon-node blip, mildly hurting liveness; never accepts an invalid sig · only during beacon-node unavailability.
- **PoC:** n/a
- **Fix:** None required for parity. If desired beyond charon, distinguish infrastructure errors (retry/log) from cryptographic verify failures (reject) in `Eth2SignedDataError`; would be a core-crate change, not parsigex-local.

### [Info] Doc comment says share is "the sending peer's public share" but selection is by the message's claimed `share_idx`, not the authenticated peer identity
- **Rust:** `crates/parsigex/src/behaviour.rs:48-59` (`new_eth2_verifier` doc)
- **Charon ref:** `core/parsigex/parsigex.go:69` (`handle` ignores sender `peer.ID`)
- **Issue:** The pubshare is looked up by `par_signed_data.share_idx` carried in the message, not bound to the libp2p peer that sent it — so a cluster peer can relay another member's valid partial sig. Behavior matches charon (documented in the baseline `[Info]` peer-identity finding above); only the doc wording ("sending peer's public share") could mislead a reader into assuming peer-identity binding exists.
- **Impact & likelihood:** Documentation-accuracy only; no behavioral divergence · n/a
- **PoC:** n/a
- **Fix:** Reword to "the share selected by the partial signature's `share_idx`" to avoid implying peer-identity binding.

### Delta summary
The #511 verifier is a clean, fail-closed 1:1 port of charon's `NewEth2Verifier` — matching lookup order, error strings, unsupported-type rejection, and inbound-only verification-before-storage. No Critical/High/Medium issues; the only notes are charon-parity error conflation (fail-closed, safe) and a minor doc-wording nit. Open integration item (not a delta defect): `new_eth2_verifier` is exported/tested but not yet wired into any production caller — the parsigex `Config`/`Behaviour` is not constructed in the `app` crate yet, so eth2 verification is not active in a running node until integration lands.
