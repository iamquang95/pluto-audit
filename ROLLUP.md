# Pluto repo audit — severity scoreboard (Phase 3 rollup)

**Date:** 2026-06-30 · **Scope:** 26 production units of pluto (Rust port of charon v1.7.1), three dimensions (parity / security / Rust quality). **Map-only:** the per-unit `shared/audit/<unit>.md` files are the deliverable; this is a one-glance scoreboard, not a merged/deduped findings file.

**Pipeline:** Phase 1 read-only scan (1 agent/unit) → Phase 2 independent skeptical verification (fresh agent/crate, isolated worktree, real `cargo test` PoCs). Every unit reached **VERIFIED** (k1util = pilot, **DONE**). No `⏳ pending` PoCs remain.

## Phase 0 — baseline gates

| Gate | Result |
| --- | --- |
| `cargo +nightly fmt --all --check` | ✅ clean |
| `cargo clippy --workspace --all-targets --all-features -- -D warnings` | ✅ clean (exit 0) |
| `cargo test --workspace --all-features` | ⚠️ blocked by `No space left on device` (env, not code) — disk was reclaimed before Phase 2; clippy had already compiled the whole workspace clean, so no code defect implied |

## Severity scoreboard (post-verification)

Counts are **final** (after Phase-2 severity adjustments). All units VERIFIED.

| Unit | C | H | M | L | I | Status |
| --- | --- | --- | --- | --- | --- | --- |
| app | 0 | 1 | 1 | 11 | 2 | VERIFIED |
| cli | 0 | 1 | 2 | 9 | 3 | VERIFIED |
| cluster | 0 | 1 | 2 | 4 | 2 | VERIFIED |
| consensus | 0 | 0 | 1 | 3 | 2 | VERIFIED |
| core-duty | 0 | 1 | 2 | 3 | 2 | VERIFIED |
| core-parsig | 0 | 0 | 0 | 6 | 2 | VERIFIED |
| core-tracker | 0 | 0 | 0 | 3 | 2 | VERIFIED |
| core-types | 0 | 0 | 1 | 4 | 1 | VERIFIED |
| core-validatorapi | 0 | 0 | 1 | 4 | 3 | VERIFIED |
| crypto | 0 | 1 | 2 | 3 | 3 | VERIFIED |
| dkg | 0 | 0 | 0 | 4 | 2 | VERIFIED |
| eth1wrap | 0 | 1 | 0 | 3 | 1 | VERIFIED |
| eth2api | 0 | 2 | 1 | 5 | 3 | VERIFIED |
| eth2util | 0 | 0 | 2 | 3 | 3 | VERIFIED |
| featureset | 0 | 0 | 0 | 2 | 2 | VERIFIED |
| frost | 0 | 0 | 0 | 4 | 2 | VERIFIED |
| infosync | 0 | 0 | 0 | 3 | 1 | VERIFIED |
| k1util | 0 | 1 | 1 | 4 | 2 | DONE (pilot) |
| p2p | 0 | 1 | 1 | 5 | 2 | VERIFIED |
| parsigex | 0 | 0 | 2 | 2 | 2 | VERIFIED |
| peerinfo | 0 | 0 | 1 | 7 | 2 | VERIFIED |
| priority | 0 | 0 | 1 | 3 | 1 | VERIFIED |
| relay-server | 0 | 1 | 3 | 3 | 2 | VERIFIED |
| ssz | 0 | 1 | 1 | 5 | 2 | VERIFIED |
| testutil | 0 | 0 | 0 | 3 | 3 | VERIFIED |
| tracing | 0 | 0 | 0 | 7 | 2 | VERIFIED |
| **TOTAL** | **0** | **12** | **25** | **113** | **54** | 26 units |

## High-severity findings — surface immediately (12)

Interop-critical (✦ = silent cross-impl breakage risk):

- ✦ **cluster** — `hash_lock_legacy` uses `validator_addresses` count instead of `distributed_validators` count for the merkle mixin → latent legacy `lock_hash` divergence (also panics when counts cross). CONFIRMED.
- ✦ **crypto** — `verify_aggregate` skips per-pubkey subgroup check; only the summed key is validated → a non-subgroup pubkey whose components cancel can be accepted. CONFIRMED (PoC: `[K,N,−N]` over sig(K) → `Ok`).
- ✦ **ssz** — `BitList::from_ssz_bytes` ignores `MAX`; over-capacity `aggregation_bits` decode silently and corrupt the attestation signing root. CONFIRMED.
- ✦ **eth2api** — EIP-7044 voluntary-exit domain trusts the beacon-node-reported Capella version (+ wrong genesis fallback) instead of charon's hard-coded per-network table → wrong exit-signing domain on networks where Capella≠genesis. CONFIRMED.
- ✦ **app** — privkeylock timestamp wire format (integer seconds) vs charon RFC3339 string → mutually unreadable lock file; single-instance guard defeatable across a mixed charon/pluto data dir. CONFIRMED (direction refined: pluto refuses to start on a charon file).

Security / availability:

- **relay-server** — rate limiters disabled (empty reservation/circuit limiter vecs) → all per-peer/per-IP throttling off on the internet-facing relay. CONFIRMED (empty vec → `.all()`==true → deny branch unreachable).
- **eth2api** — validator cache never invalidated in production (`trim()` test-only; prod cache built with empty pubkeys, never repopulated) → stale validator set/status for the whole process lifetime. CONFIRMED (worse than scanned).
- **k1util** — saved secp256k1 private key file is world/group-readable (0o644 vs charon 0o600). CONFIRMED (pilot). → ✅ **FIXED in [PR #507](https://github.com/NethermindEth/pluto/pull/507)** (see Remediation).
- **eth1wrap** — ERC-1271 contract address parsed with strict EIP-55 checksum → valid lowercase contract-signer addresses (that Go accepts) fail verification, whole verify returns `Err`. CONFIRMED.

Parity / functional:

- **core-duty** — dutydb missing Electra committee-index-0 dual-store → post-Electra VCs querying index 0 get `PubKeyNotFound`/timeout instead of data → silently missed attestations. CONFIRMED.
- **p2p** — `ConnGater` gates inbound only; outbound connections to non-allowlisted peers are accepted (charon's `InterceptSecured` is direction-agnostic). CONFIRMED.
- **cli** — `pluto alpha test all` aborts: clap duplicate-flattened-args debug-assert (every invocation) and `unimplemented!()` in release. CONFIRMED/REFINED (extra bug found).

**Critical:** none found.

## Verification value — adjustments made by the skeptical Phase-2 pass

The independent verifier and follow-up High/Medium triage downgraded/refined several scanner claims (the reason Phase 2 exists):

- **cluster** legacy config-hash timestamp precedence bug: **High → Low** — real mis-parenthesization but functionally inert (empty timestamp's all-zero SSZ chunk absorbed by merkleization zero-padding; config hash byte-identical to charon, verified).
- **core-parsig** sigagg template scan: **High → Low** (divergence ruled out by consensus's one-payload-per-duty invariant; comment/quality only).
- **core-parsig** parsigdb + aggsigdb `PartialEq` dedup: **2× Medium → Low** (no divergence reachable with current types; latent hazard).
- **core-parsig** aggsigdb-V2-not-ported: **Medium → Info** (premise refuted — V2 is `statusAlpha`, disabled by default in v1.7.1; pluto matches the production V1 default).
- **crypto** `generate_insecure_secret`: likelihood down-weighted to ~2⁻²⁵⁶ (only the all-zero draw diverges; the ">=r" worry refuted — herumi is also big-endian reject-≥r).
- **frost** round2 bcast off-by-one: mechanism **REFINED** (scanner's "max_signers−1 peers" reason wrong; real story = pluto's caller strips self-bcast, so off-by-one only bites a direct un-stripped call).
- **priority** stable-vs-unstable sort: trigger **REFINED** to tied-tier width ≥13 (Go pdqsort uses stable insertion sort ≤12); divergence confirmed by actually running Go.
- **eth2api** dropped nil-validator guard: **REFINED** to a coverage-check gap (serde already rejects null inner validator).
- **follow-up High/Medium triage** downgraded peerinfo nickname and tracing OTLP from High to Low (metrics/log-only or scope-dependent), downgraded latent/fixture-only/warning-only/Charon-equivalent Mediums, and marked crypto insecure-secret + `compute_domain` length indexing as Info/not-a-finding-as-written. Current post-triage totals: 0C / 12H / 25M / 113L / 54I.

## Method notes

- Phase 2 ran as parallel, isolated-worktree verifiers (waves of 3). Several crate builds genuinely compiled and ran scratch `#[test]`s (blst, alloy, libp2p behaviours; priority even cross-checked Go 1.25); Go/kryptology/herumi/fastssz behavior not vendored locally was marked `reasoned (Go not run)`.
- Open items per unit live under each file's `## Uncertain / needs-human` — these need maintainer intent (e.g. whether SSZ-on-submit, BN-Capella-version trust, span-tracing deferral, broadcast-dial semantics are deliberate).

## Remediation (post-audit fixes)

Tracks audit findings addressed by merged/open PRs. The scoreboard counts above are the *original* audit tally and are left unchanged for provenance.

### [PR #507](https://github.com/NethermindEth/pluto/pull/507) — harden secret key material against Debug leakage and zeroize gaps

| Unit | Finding | Status | Notes |
| --- | --- | --- | --- |
| k1util | [High] Saved private key file is world/group-readable | ✅ **Fixed** | `save` now writes `0o600` on Unix (+ forces perms on overwrite, stricter than charon); regression tests added. |
| crypto | [Medium] Secret material never zeroized | ⚠️ **Partial** | `ikm` buffers wrapped in `Zeroizing`; `poly`/`share_secrets`/`recovered` already zeroize-on-drop via blst (see below). **Open:** `PrivateKey = [u8;32]` (Copy) still not a `Zeroizing` newtype — returned key bytes unwiped. |
| frost | *(not a prior finding)* | ➖ proactive | Added scalar `zeroize()` in `round1`/`round2` + redacting `Debug` for `SigningShare`/`KeyPackage`/`ShamirShare`. |

**Audit correction surfaced during this PR review:** `crypto.md`'s zeroize finding claimed *"blst 0.3.16 `SecretKey` has no `Drop`/zeroize (the `serde-secret` zeroize path is feature-gated…)"* — this is **false**. blst depends on `zeroize` non-optionally and derives `#[zeroize(drop)]` on `SecretKey` unconditionally; `serde-secret` gates only serde serialization, not the drop-wipe. So blst-held secrets (`poly`, `share_secrets`, `recovered`) were already wiped pre-PR, and the finding overstated its scope. Corrected inline in `crypto.md`.

---

## Delta audit 2026-07-02 — commits since the 2026-06-30 baseline

**Scope:** commits `68114fa..c241b50` — #506 (anyhow bump, dep-only), #507 (crypto hardening, see Remediation above; operator regression skim found no new issues), #505 (expbackoff consolidation), #511 (parsigex eth2 verifier), #508 (UnsignedDataSet proto/SSZ codecs for all duty types). Same pipeline, scaled down: 3 read-only scan agents (opus) + 1 independent Phase-2 verifier (real `cargo test` PoC) for the single Medium. Findings appended per-unit under each file's `## Delta audit 2026-07-02` section; baseline scoreboard above unchanged (provenance).

**Delta severity counts (all delta sections VERIFIED):**

| Unit | C | H | M | L | I | Delta source |
| --- | --- | --- | --- | --- | --- | --- |
| core-types | 0 | 0 | 1 | 2 | 2 | #508 |
| core-parsig | 0 | 0 | 0 | 0 | 1 | #508 |
| parsigex | 0 | 0 | 0 | 0 | 3 | #511 |
| core-duty | 0 | 0 | 0 | 1 | 0 | #505 |
| p2p | 0 | 0 | 0 | 1 | 0 | #505 |
| **TOTAL** | **0** | **0** | **1** | **4** | **6** | |

**The one Medium (CONFIRMED):** ✦ **core-types** — #508's two new *unsigned* versioned SSZ decoders (`decode_versioned_proposal`, `decode_versioned_aggregated_attestation`) repeat the baseline exact-offset divergence: they require the value-offset field to equal the header exactly and slice at the constant, where charon accepts `offset >= header` and slices at the decoded offset. PoC: crafted Go-legal `o1 = header+4` bytes → `Err(InvalidOffset)` from both decoders. Interop-safe for canonical charon output (always emits `o1 == header`); same fix as the baseline Medium — consolidate all five versioned decoders onto one `>=`-checking helper.

**Notable positives / closures in the delta:**

- **#511** `new_eth2_verifier` is a clean, **fail-closed** 1:1 port of charon's `NewEth2Verifier` (same lookup order, error strings, unsupported-family rejection; inbound-verified before parsigdb). ⚠️ Integration item: it is exported/tested but **not yet wired into any production caller** — eth2 partial-sig verification is inactive in a running node until the app crate constructs the parsigex `Config`/`Behaviour` with it.
- **#508** fixes the baseline core-types Low "JSON-prefix-first codec ordering": `deserialize_signed_data` now tries SSZ first with JSON as `{`-prefix-gated fallback, matching charon's `unmarshal` (regression tests for leading-`0x7B` SSZ added). That Low can be considered closed.
- **#505** fixes a real availability bug: `query_relay_addrs`'s old inline backoff inherited backon's default 3-retry cap, so a ~0.5s transient relay outage permanently killed relay resolution; `expbackoff::fast()` restores charon's retry-until-cancel. New Low surfaced by it: the now-unbounded loop is only cancel-checked between attempts and the reqwest request has no timeout/token binding → shutdown can hang against a stalled endpoint (Go binds the request to `ctx`).
- **#505/L** the shared `expbackoff` module doc mischaracterizes backon's jitter: actual jitter is one-sided `[d, 2d)` proportional to the current delay (charon: symmetric ±20%) — timing-only divergence, doc should be corrected.

## New-issue scan 2026-07-02 — additional findings beyond baseline + deltas

A follow-up targeted scan (not commit-driven) surfaced five **new** parity findings not in the frozen scoreboard above. Each is written into its unit file's Findings section (`## Delta scan (2026-07-02)` note in each). The frozen table is left unchanged for provenance.

| Unit | Sev | New finding | PoC |
| --- | --- | --- | --- |
| eth2api | **Medium** | `fork_schedule_from_spec` hard-requires **every** fork in `FORKS` (incl. `Fulu`) in the BN `/eth/v1/config/spec`, so a beacon node predating a fork pluto knows about makes **all** signing-domain computation fail (total, fail-closed liveness break). Contradicts the crate's own tolerant `resolve_fork_version` (which `continue`s past absent forks). Charon/go-eth2-client tolerate missing forks. | confirmed by inspection (Go not run); tests mask it because the beacon mock injects a full fork set incl. `FULU_FORK_VERSION` + `FULU_FORK_EPOCH=u64::MAX` |
| eth2util | **Low** | ENR TCP/UDP ports `1..=255` diverge from charon both ways: pluto encodes them non-minimally (2-byte leading-zero via `u16::to_be_bytes`) and its `tcp()`/`udp()` require exactly 2 bytes, so they return `None` for a charon ENR that minimally encodes such a port (1 byte) — silently dropping the peer's dial address in `p2p::bootnode::multi_addr_from_enr_str`. | **CONFIRMED** — scratch test: signature-valid charon-style ENR with 1-byte port 80 → `tcp()==None`, `udp()==None` |
| priority | **Low** | Inbound exchange times the request read and the `handle_request` enqueue/wait phases with two independent `RECEIVE_TIMEOUT` timers, so a slow cluster peer can pin an inbound stream ~2× longer (~10s) than charon's single shared receive-timeout context (~5s). | confirmed by inspection (Go not run) |
| core-tracker | **Low** | Participation "absent set changed" gate uses `HashMap::get() != Some(&absent)`; unlike charon's `fmt.Sprint` nil==empty comparison it fires on the **first** duty of each type when all peers participated → one spurious "All peers participated in duty" log per duty type. Log-only. | confirmed by inspection (Go not run) |
| core-validatorapi | **Low** | Wrong-method request to a registered DV path returns `405` (axum `MethodRouter`, no `method_not_allowed_fallback`) instead of being proxied to the beacon node as charon does (mux `ErrMethodMismatch` → `PathPrefix("/")` passthrough). Latent (`new_router` not yet served). | confirmed by inspection (Go not run) |

**Headline:** the eth2api **Medium** is the highest-value new item — a realistic total-liveness break during any fork-rollout window against a lagging beacon node, hidden by the test mock's complete fork set. The eth2util **Low** is the only CONFIRMED-by-execution new finding.
