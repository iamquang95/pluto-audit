# Audit: core-validatorapi  (11678 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/core/src/validatorapi`
- **Charon:** `core/validatorapi/{router,validatorapi,eth2types,metrics}.go`
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `validatorapi` ↔ `core/validatorapi/*.go` — beacon-node-facing HTTP router + duty types

## Crate-specific focus
- **Parity:** every endpoint, path, query param, and response shape MUST match the Eth2 beacon API + charon's `router.go`. Attestation/proposal/sync-committee duty flows, error→HTTP-code mapping, versioned (fork) request/response types.
- **Security:** PRIMARY external HTTP attack surface — input validation on all params/bodies, size limits, no panic on malformed JSON/SSZ, auth/TLS expectations, DoS.
- **Quality:** router framework usage, handler error mapping, async.

## Summary
Severity: 0C / 0H / 1M / 4L / 3I. Overall the port is faithful: route table order, paths/methods, versioned fork dispatch, pubshare↔root rewriting, partial-sig verification domains, duty-slot derivation, and JSON response envelopes all match Charon, and several Pluto-specific input guards (val-index count cap, per-endpoint body limits, single-vs-first query semantics, uniform 400 body shape) are well-reasoned additions. The remaining Medium is a media-type contract divergence (Pluto accepts SSZ on the two v2 submit endpoints that Charon marks JSON-only). The `.expect()` panic on an unregistered AggSigDB hook is Low after triage because it requires a wiring bug and is not user-triggerable under normal registration. Lows/Infos are proxy/timeout hardening gaps (2 MiB buffered proxy body, no router-level deadline, `Connection`-header hop-by-hop stripping) plus cosmetic nits. Note: `new_router` has no call site yet, so serving-time concerns (TLS/auth/global limits) are latent. No memory-safety, crypto, or unsafe issues found.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Medium] submit_attestations / submit_aggregate_attestations accept SSZ bodies; Charon is JSON-only
- **Rust:** `crates/core/src/validatorapi/router.rs:176-179, 227-230` (route table — `submit_attestations`, `submit_aggregate_attestations`); handlers at `router.rs:411` (`submit_attestations`) and `router.rs:805` (`submit_aggregate_attestations`), both call `request_is_ssz(&headers)` (`router.rs:1450`) and decode SSZ when `application/octet-stream`.
- **Charon ref:** [`core/validatorapi/router.go:137-142`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/validatorapi/router.go#L137-L142) (`submit_attestations_v2`) and `:263-268` (`submit_aggregate_and_proofs_v2`) — both `Encodings: []contentType{contentTypeJSON}`; `wrap` (`router.go:381`) rejects any non-listed encoding with `415` via `slices.Contains(encodings, typ)`.
- **Issue:** Pluto registers these two POST routes with bare `post(handler)` (no `enforce_json_content_type` layer) and the handlers branch on the content type, decoding SSZ when present. Charon's endpoint table marks both JSON-only, so a `Content-Type: application/octet-stream` request returns `415 Unsupported Media Type`. Pluto instead decodes the SSZ body and proceeds (200 / decode-error 400). Divergent media-type contract for the externally reachable submit endpoints.
- **Impact & likelihood:** A VC (or attacker) sending SSZ to `/eth/v2/beacon/pool/attestations` or `/eth/v2/validator/aggregate_and_proofs` gets accepted/processed where Charon would 415. Behavioral divergence, not a memory-safety issue (SSZ decode is length-checked). · only when a client sends SSZ to these two endpoints (uncommon; most VCs use JSON here).
- **PoC:** CONFIRMED — confirmed by inspection. Routes registered bare: router.rs:176-179 `post(submit_attestations)` and :227-230 `post(submit_aggregate_attestations)` — no `enforce_json_content_type` layer (contrast the duties/selections routes which use `bounded_post`/explicit guards). Handlers call `request_is_ssz(&headers)` (router.rs:1450) and decode SSZ on `application/octet-stream`. Charon router.go:137-142 & :263-268 mark both `Encodings: []contentType{contentTypeJSON}`; `wrap` (:381) returns 415 via `slices.Contains` for any non-listed encoding. Severity Medium retained (intent question — see Uncertain — may be a forward-looking deviation).
- **Fix:** Wrap both routes with `enforce_json_content_type` (as the duties/selections routes are) and drop the `request_is_ssz` branch in the two handlers, or — if SSZ support is intended ahead of Charon — document the deliberate deviation in the route comments like the other documented deviations.

### [Low] Selection handlers panic (`.expect`) on unregistered AggSigDB hook instead of returning a status
- **Rust:** `crates/core/src/validatorapi/component.rs:1427-1430` (`beacon_committee_selections`) and `:1519-1522` (`sync_committee_selections`) — `self.await_agg_sig_db_fn.as_ref().expect("await_agg_sig_db hook must be registered before serving requests")`.
- **Charon ref:** [`core/validatorapi/validatorapi.go:849`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/validatorapi/validatorapi.go#L849) (`BeaconCommitteeSelections`) / `:1123` (`SyncCommitteeSelections`) — call `c.awaitAggSigDBFunc(...)` directly (nil-func call also panics in Go, but Charon wires it unconditionally in `NewComponent`).
- **Issue:** These two handlers `.expect()` (panic) on a missing hook, while every other optional-hook consumer in the same file (`lookup_proposer_pubkey` `component.rs:437`, `lookup_attester_definitions` `:562`, `pub_key_by_attestation` `:598`, `aggregate_attestation` `:1261`, `sync_committee_contribution` `:1762`) gracefully returns `SERVICE_UNAVAILABLE`/`INTERNAL_SERVER_ERROR`. The panic fires on a request-handling task: a partial-sig set has already been fanned out to subscribers before the panic, so the request aborts mid-flow (axum drops the connection / 500) after side effects, rather than cleanly failing.
- **Impact & likelihood:** If the component is ever served with the AggSigDB hook unregistered (wiring/order bug), the first selections request panics the handler task after fanout. Inconsistent failure mode vs the rest of the module. · only on a wiring bug (hook never registered); not directly triggerable by untrusted input. Lower likelihood, but worse failure mode than the sibling 503 paths.
- **PoC:** CONFIRMED — confirmed by inspection. component.rs:1427-1430 (`beacon_committee_selections`) and :1519-1522 (`sync_committee_selections`) call `self.await_agg_sig_db_fn.as_ref().expect(...)` → panic if `None`. Verified the panic fires AFTER side effects: the subscriber fanout loop (component.rs:1407-1423) runs the per-slot `sub(&duty, set).await` for every subscriber BEFORE the `.expect()` at :1430. Every sibling optional-hook consumer (lookup_proposer_pubkey :437, lookup_attester_definitions :562, pub_key_by_attestation :598, aggregate_attestation :1261, sync_committee_contribution :1762) instead returns a graceful 503/500. Inconsistent failure mode confirmed. Reachable only via a wiring bug (hook never registered), not untrusted input — Severity lowered to Low.
- **Fix:** Replace both `.expect(...)` with `ok_or_else(|| ApiError::new(StatusCode::SERVICE_UNAVAILABLE, "aggsigdb lookup not registered"))?` to match the other hook consumers, and resolve the hook before the fanout loop so the request fails before producing side effects.

### [Low] Proxy fallback buffers the request body and inherits axum's 2 MiB default limit
- **Rust:** `crates/core/src/validatorapi/router.rs:252` (`.fallback(proxy_handler)`) + `:1048-1054` (`proxy_handler(..., body: Bytes)`); no `DefaultBodyLimit` override on the fallback.
- **Charon ref:** [`core/validatorapi/router.go:322`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/validatorapi/router.go#L322) (`r.PathPrefix("/").Handler(proxyHandler(...))`) + `:1622` (`proxyHandler`) — uses `httputil.NewSingleHostReverseProxy`, which streams the request body with no size cap.
- **Issue:** Pluto's proxy extracts the whole request body into `Bytes` (buffered, not streamed) and, having no per-route `DefaultBodyLimit`, is subject to axum's 2 MiB default. Charon proxies the request body as a stream with no limit. A proxied POST larger than 2 MiB (e.g. a large `prepare_beacon_proposer` set if it were proxied, or any future large upstream POST) would be rejected with `413` where Charon forwards it. Response body is correctly streamed (`router.rs:1158`); only the request side is buffered.
- **Impact & likelihood:** Large proxied request bodies fail with 413; also buffers each proxied request fully in memory. Note: `new_router` is not yet wired into a served binary (no call site found outside the module), so impact is latent until serving lands. · only for proxied requests >2 MiB.
- **PoC:** n/a
- **Fix:** Apply `DefaultBodyLimit::disable()` (or a high explicit cap) to the fallback route and stream the request body via `reqwest::Body::wrap_stream` rather than collecting `Bytes`, matching Charon's streaming proxy.

### [Low] No overall per-request timeout on the router; Charon enforces a 10s deadline on every endpoint
- **Rust:** `crates/core/src/validatorapi/router.rs:161-254` (`new_router`) — no request-timeout layer; per-leg timeouts live inside the Component (`UPSTREAM_REQUEST_TIMEOUT` 10s, `ATTESTATION_DATA_TIMEOUT`/`PROPOSAL_TIMEOUT`/`DUTY_AWAIT_TIMEOUT` 24s, `component.rs:131-150`).
- **Charon ref:** [`core/validatorapi/router.go:61`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/validatorapi/router.go#L61) (`defaultRequestTimeout = 10 * time.Second`) + `:353` (`context.WithTimeout(ctx, defaultRequestTimeout)`) — a single 10s deadline wraps every handler.
- **Issue:** Charon bounds the *whole* request at 10s. Pluto has no router-level deadline; instead each handler bounds only its own blocking legs, and the proposal/attestation waits are intentionally 24s (documented deviation). Handlers with no upstream call (e.g. `submit_attestations`, which only fans out to subscribers, `component.rs:1032`) have no deadline at all — a slow/hanging subscriber can park the task indefinitely. The 24s waits also exceed Charon's 10s, so a VC's request can hang ~2.4× longer.
- **Impact & likelihood:** A stuck subscriber or upstream leg lacking its own timeout holds a handler task longer than Charon would. Bounded for the upstream/dutydb legs; unbounded for pure-fanout handlers. · only when a downstream/subscriber stalls.
- **PoC:** n/a
- **Fix:** Add a `tower_http::timeout::TimeoutLayer` (or `tower::timeout`) at the router level mirroring Charon's 10s, or document and justify the per-handler-only model and add a fanout timeout to the submit handlers.

### [Low] `Connection`-listed hop-by-hop headers are not stripped by the proxy
- **Rust:** `crates/core/src/validatorapi/router.rs:1166-1181` (`is_hop_by_hop_header`) — matches a fixed static list only.
- **Charon ref:** [`core/validatorapi/router.go:1657`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/validatorapi/router.go#L1657) (`proxy.ServeHTTP`) — Go's `httputil.ReverseProxy` additionally removes every header named in the request's `Connection` header (per RFC 7230 §6.1) before forwarding.
- **Issue:** RFC 7230 requires a proxy to drop any header field named in the `Connection` header, in addition to the well-known hop-by-hop set. Pluto strips only the fixed list and ignores `Connection`-named fields, so connection-scoped headers leak end-to-end on proxied requests/responses.
- **Impact & likelihood:** Minor protocol-correctness gap; could confuse a strict upstream/downstream. No security impact for the trusted-upstream proxy. · always, but only matters when a client/upstream sends a non-empty `Connection` listing custom headers (rare).
- **PoC:** n/a
- **Fix:** Parse the `Connection` request/response header and add each comma-separated token to the strip set, mirroring `httputil`.

### [Info] node_version arch/os tokens differ in spelling from Charon (`x86_64`/`macos` vs `amd64`/`darwin`)
- **Rust:** `crates/core/src/validatorapi/component.rs:846-857` (`node_version`) — `std::env::consts::ARCH` / `std::env::consts::OS`.
- **Charon ref:** [`core/validatorapi/validatorapi.go:1297-1301`](https://github.com/ObolNetwork/charon/blob/v1.7.1/core/validatorapi/validatorapi.go#L1297-L1301) (`NodeVersion`) — `runtime.GOARCH` / `runtime.GOOS`.
- **Issue:** Rust's `ARCH`/`OS` constants yield `x86_64`/`aarch64` and `linux`/`macos`, while Go yields `amd64`/`arm64` and `linux`/`darwin`. The `charon`→`pluto` substring is an intentional identity change; the arch/os token spelling difference is incidental. Version string is identification-only.
- **Impact & likelihood:** Cosmetic; the field is informational. · always (every `/eth/v1/node/version` call) but no functional effect.
- **PoC:** n/a
- **Fix:** If exact token parity matters, map `x86_64→amd64`, `aarch64→arm64`, `macos→darwin`; otherwise leave as-is and note the deviation.

### [Info] Stale `#[allow(dead_code)]` on `await_proposal_fn`
- **Rust:** `crates/core/src/validatorapi/component.rs:185` (`#[allow(dead_code, reason = "consumed by proposal handler in later PRs")]` on `await_proposal_fn`).
- **Charon ref:** n/a (Rust-only).
- **Issue:** The field is actively read at `component.rs:478` (`await_proposal_for_handler`) and registered at `:302`; the allow + "later PRs" reason are stale and now misleading.
- **Impact & likelihood:** None functional; misleads readers and suppresses a lint that would otherwise pass. · always (cosmetic).
- **PoC:** n/a
- **Fix:** Remove the `#[allow(dead_code, ...)]` attribute.

### [Info] Duplicated subscriber-fanout loop with divergent error mapping
- **Rust:** `crates/core/src/validatorapi/component.rs:419-430` (`fanout` helper) used by `submit_sync_committee_contributions`/`submit_sync_committee_messages`/`submit_voluntary_exit`, but `submit_attestations` (`:1068-1076`), `proposal` (`:1128-1133`), `submit_proposal` (`:1191-1196`), `submit_blinded_proposal` (`:1244-1249`), `submit_aggregate_attestations` (`:1352-1360`), `beacon_committee_selections` (`:1409-1422`), `sync_committee_selections` (`:1500-1515`) each inline their own fanout loop with slightly different error messages and source-attachment (`with_boxed_source` vs `subscriber_error_to_api_error`).
- **Charon ref:** `core/validatorapi/validatorapi.go` — Charon repeats `for _, sub := range c.subs { sub(...) }` in each method too, so this is not a parity gap; purely a Rust-quality dedup opportunity.
- **Issue:** Seven near-identical fanout loops; the `fanout` helper already exists but is used by only three handlers. Inconsistent error messages/sources for the same downstream-failure condition.
- **Impact & likelihood:** Maintenance/consistency only. · n/a.
- **PoC:** n/a
- **Fix:** Route every submit handler's fanout through `self.fanout(&duty, set)` (or a per-slot variant) for one error-mapping path.

## Uncertain / needs-human

- **Per-handler 24s timeouts vs Charon's 10s:** `ATTESTATION_DATA_TIMEOUT`/`PROPOSAL_TIMEOUT`/`DUTY_AWAIT_TIMEOUT` are 24s (≈2 slots), documented as deliberate so a duty has time to flow through the pipeline. Charon's blanket deadline is 10s. Need a maintainer to confirm the 24s deviation is intended cluster-wide (it affects how long a VC blocks) and that no VC client treats >10s as a hard failure. (Tied to the Low timeout finding.)
- **SSZ on submit_attestations/aggregate (Medium #1):** Is accepting SSZ on these two endpoints an intentional forward-looking deviation (newer beacon-API specs do allow SSZ for v2 attestation submission) or an oversight? If intentional, downgrade to Info + document; if not, it is a real media-type contract divergence. Needs maintainer intent.
- **`validators` flow restructure:** Charon's component `Validators` uses a `CompleteValidators` cache + `getPubKeyFunc` (with a specific "mismatching key share index" diagnostic error) and only queries the BN for non-cached pubkeys. Pluto's `validators` (`component.rs:1555`) inverts `pub_share_by_pubkey` and forwards all ids to the BN in one POST, dropping the cache-hit fast path and the share-index-mismatch diagnostic. Functionally equivalent for correctness, but the perf characteristic (always hits BN) and the lost diagnostic differ. Flagging for a maintainer to confirm the cache path is intentionally deferred and the diagnostic loss is acceptable.
