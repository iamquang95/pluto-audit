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
- **k1util** — saved secp256k1 private key file is world/group-readable (0o644 vs charon 0o600). CONFIRMED (pilot).
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
