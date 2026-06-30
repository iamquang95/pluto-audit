# Audit: dkg  (14840 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/dkg/src/*` (dkg, frost, frostp2p, exchanger, nodesigs, share, disk, aggregate, publish, signing, validators, node, sync)
- **Charon:** `dkg/{dkg,frost,frostp2p,exchanger,nodesigs,share,disk}.go`
- **Tier A** · model: **opus** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `dkg.rs`/`node.rs` ↔ `dkg/dkg.go` (ceremony orchestration)
- `frost.rs`/`frostp2p` ↔ `dkg/{frost,frostp2p}.go`
- `exchanger.rs` ↔ `dkg/exchanger.go`; `nodesigs.rs` ↔ `dkg/nodesigs.go`
- `share.rs` ↔ `dkg/share.go`; `disk.rs` ↔ `dkg/disk.go` (lock/keystore output)

## Crate-specific focus
- **Parity:** FROST DKG round sequencing, share verification, exchanger message ordering + completeness, nodesigs threshold collection, disk output format (cluster lock + keystores). Ceremony MUST abort on any verification failure.
- **Security (CRITICAL):** private-key-share handling, verify ALL peer contributions (reject invalid shares — no silent skip), no secret material in logs/errors, secure RNG, restrictive permissions on written keystores, zeroize secrets.
- **Quality:** ceremony state machine, secret cleanup, error transparency.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

Overall the FROST round sequencing, share/cast/p2p validation, partial-signature
verification (every peer contribution is verified, no silent skip), aggregate
verification, bcast signature verification (`sha256(type_url||value)` matches Go,
incl. an explicit Go-expected-value test) and sync def-hash signature verification
are faithful ports — several paths are *stricter* than charon. The findings below
are mostly divergences in error/control-flow semantics and a few quality items;
no key-share-leaking or verification-bypass bug was found in the scan.

### [Low] `make_shares` builds public-share map only from received round-2 casts; missing/under-count not detected
- **Rust:** `crates/dkg/src/frost.rs:414` (`make_shares`)
- **Charon ref:** [`dkg/frost.go:203`](https://github.com/ObolNetwork/charon/blob/v1.7.1/dkg/frost.go#L203) (`makeShares`)
- **Issue:** Both Go and Rust build `pub_shares[val_idx][source_id]` purely from whatever round-2 casts arrived, with no check that each validator received a full set of `num_nodes` public shares before constructing the `Share`. Rust additionally uses `pub_shares.get(&v_idx).cloned().unwrap_or_default()` (line 444), so a validator with *zero* received vk-shares silently yields a `Share` with an empty `public_shares` map instead of erroring. The round-2 transport (`transport.rs:384`) does wait for exactly `num_peers` cast messages and `validate_round2_casts` enforces one cast per validator, so in the honest path the map is complete; the gap is the absence of a defensive completeness assertion at share-construction time (parity with Go, which is equally permissive). A later code change to the transport that loosened the count check would silently produce shares with incomplete `public_shares`, which feed `DistValidator.pub_shares` written to the cluster lock.
- **Impact & likelihood:** Cluster lock could be written with an empty/short `public_shares` set for a DV → downstream threshold verification of that DV fails for everyone · only if transport count invariants are weakened or a peer omits a validator's cast while still passing per-cast validation (not reachable today) → low, but high blast radius (corrupt lock).
- **PoC:** CONFIRMED — scratch test `scratch_make_shares_missing_validator` (real `DkgParticipant`s via in-mem 3-node DKG; all `val_idx==0` casts pruned from `r2_result` before `make_shares`) → `make_shares` returns `Ok` with `shares[0].public_shares.len()==0` (the `unwrap_or_default()` at frost.rs:444 silently yields an empty map, no error). Honest-path sanity in same test: every validator's `public_shares.len()==3` (complete). Confirms the silent-empty behavior is exactly the defensive gap described — real but not reachable today: `validate_round2_casts` (frostp2p/transport.rs:266) enforces `casts.len()==num_validators` + per-`val_idx` uniqueness per peer, and `round2` (transport.rs:384) waits for exactly `num_peers` such messages, so the honest/validated path always fills all `num_nodes` entries per validator. Go side reasoned (Go not run): charon `makeShares` (frost.go:202) is equally permissive (`pubShares[uint32(vIdx)]` direct map index, nil if absent) — divergence is only Rust's silent empty vs Go's nil map, both non-erroring; severity lowered to Low: real defensive gap, but not reachable under current transport invariants and Go is equally permissive.
- **Fix:** After collecting, assert `pub_shares[v_idx].len() == num_nodes` for every validator and return `FrostError::MissingRoundState` (or a dedicated error) otherwise; drop the `unwrap_or_default()` so a missing entry is a hard error.

### [Low] `nodesigs` adds a `None`-key path with NONE_DATA sentinel + filtering not present in charon (dead in real flow, divergent if ever used)
- **Rust:** `crates/dkg/src/nodesigs.rs:105` (`NodeSigBcast::exchange`), `:160` (`all_sigs`)
- **Charon ref:** [`dkg/nodesigs.go:162`](https://github.com/ObolNetwork/charon/blob/v1.7.1/dkg/nodesigs.go#L162) (`exchange`), `:87` (`allSigs`)
- **Issue:** Charon's `exchange` takes a non-optional `*k1.PrivateKey` and always contributes a real signature; `allSigs` returns all N slots once each is non-empty. Pluto adds an `Option<&SecretKey>`: when `None`, the local node and any peer sending `NONE_DATA` are stored as the sentinel `[0xde,0xad,0xbe,0xef]`, and `all_sigs` *filters out* sentinel slots, returning fewer than N signatures. The sole caller (`dkg.rs:742`) always passes `Some(&key)`, so the sentinel/filter path is unreachable in the real ceremony and behaviour matches charon there. As written it is divergent dead code: if the `None` path were ever wired up, `lock.node_signatures` would silently be shorter than the peer count (charon always emits N), changing lock contents/verification.
- **Impact & likelihood:** No effect today (path unreachable); future misuse would silently drop node signatures from the lock · negligible now.
- **PoC:** n/a
- **Fix:** Either remove the `Option`/`NONE_DATA`/filter machinery to mirror charon's always-sign contract, or document and gate it explicitly; do not leave a divergent silent-drop path on key-bearing output.

### [Low] `Config::has_test_config` includes `p2p_key`; charon's `HasTestConfig` does not
- **Rust:** `crates/dkg/src/dkg.rs:307` (`Config::has_test_config`)
- **Charon ref:** [`dkg/dkg.go:84`](https://github.com/ObolNetwork/charon/blob/v1.7.1/dkg/dkg.go#L84) (`Config.HasTestConfig`)
- **Issue:** Charon returns true iff `StoreKeysFunc|SyncCallback|Def|P2PNodeCallback` is set — pointedly *not* `P2PKey`. Pluto returns true if `def.is_some() || p2p_key.is_some()`. `has_test_config()` gates skipping `check_clear_data_dir` (`dkg.rs:503`) and the mainnet-test-config rejection (`:510`). So a pluto run with only `p2p_key` set (no `def`) skips the clean-data-dir preflight and would be rejected on mainnet, whereas charon would run the preflight and allow mainnet. In practice tests set `def` and `p2p_key` together, so the difference is not exercised, but the gating semantics differ.
- **Impact & likelihood:** Divergent preflight/mainnet gating when only `p2p_key` is set · only in custom embeddings that set `p2p_key` without `def` → low.
- **PoC:** n/a
- **Fix:** Exclude `p2p_key` from `has_test_config()` to match charon (`self.test_config.def.is_some()` only), unless the broader gate is intended — then document why.

### [Low] bcast inbound verifies sender signatures before checking the message ID is registered
- **Rust:** `crates/dkg/src/bcast/handler.rs` (`handle_inbound_msg` / `verify_signatures` call path), `crates/dkg/src/bcast/protocol.rs:30` (`verify_signatures`)
- **Charon ref:** `dkg/bcast/server.go` (`verifyFunc`, which checks `msgIDAllowed` as part of verification)
- **Issue:** On a fully-signed `BCastMessage`, pluto runs `verify_signatures` (N secp256k1 verifications) before confirming the message ID corresponds to a registered handler; charon folds the message-ID-allowed check into its verify function so unknown IDs are rejected earlier. Functionally both reject unknown IDs; pluto just spends verification work first.
- **Impact & likelihood:** Minor CPU-amplification DoS surface: a connected peer can force N signature verifications with an unregistered message ID · only from an already-authenticated cluster peer (gater restricts to cluster) → low.
- **PoC:** n/a
- **Fix:** Reject unregistered message IDs before running `verify_signatures` (or include the registry check inside it) to mirror charon's ordering.

### [Info] `write_lock` uses `set_readonly(true)` instead of explicit 0o444
- **Rust:** `crates/dkg/src/disk.rs:213` (`write_lock`), also `check_writes` `:293`
- **Charon ref:** [`dkg/disk.go:148`](https://github.com/ObolNetwork/charon/blob/v1.7.1/dkg/disk.go#L148) (`writeLock`, `os.WriteFile(..., 0o444)`)
- **Issue:** Charon writes the lock with explicit mode `0o444`. Pluto writes with the default mode then calls `Permissions::set_readonly(true)`, which on Unix clears the write bits (0o644→0o444). Result is equivalent on Unix; on non-Unix the semantics differ but DKG targets Unix. Not a security regression (the just-written content was world-readable during the write either way; group/other read is intended for the lock). Noted for exactness.
- **Impact & likelihood:** None on supported platforms · n/a.
- **PoC:** n/a
- **Fix:** Optional: write with explicit `OpenOptions().mode(0o444)` for byte-for-byte parity and platform independence.

### [Info] `dkg_context_byte` truncates parsed context to its lowest little-endian byte (matches charon's `strconv.Atoi`-and-truncate, but obscure)
- **Rust:** `crates/dkg/src/frost.rs:474` (`dkg_context_byte`)
- **Charon ref:** `coinbase/kryptology` `NewDkgParticipant` (`strconv.Atoi(ctx)` ignoring error, low byte used as FROST `ctx`)
- **Issue:** Production passes the hex definition hash ("0x..."), which fails integer parse and intentionally becomes `ctx = 0` on both sides — so the FROST domain-separation context byte is always 0 in practice (a known charon quirk faithfully reproduced). The Rust uses `to_le_bytes()[0]`; for the always-0 production case this is correct. Flagged only so a reader does not mistake the truncation for a bug.
- **Impact & likelihood:** None — faithful to charon · n/a.
- **PoC:** n/a
- **Fix:** None; the comment already explains the intent.

## Uncertain / needs-human

- **`make_shares` completeness (Medium above):** confirm whether the round-2
  transport's `num_peers` wait + `validate_round2_casts` per-cast invariants make
  an incomplete `public_shares` map truly unreachable today (I believe so), and
  whether the defensive assert is warranted vs. preserving exact charon behaviour.
- **bcast hash determinism:** `hash_any` hashes `type_url || value` where `value`
  is the already-serialized proto bytes carried in the `Any`. Parity depends on
  both sides producing identical `Any.value` bytes for the same logical message
  (deterministic prost vs. Go `proto.Marshal`). The unit test pins one vector;
  a human should confirm prost encoding is byte-identical to charon's for the
  real `FrostRound1Casts`/`MsgNodeSig`/etc. payloads (map/field ordering), since
  any mismatch breaks cross-impl signature verification. Likely fine (single
  vector passes) but not exhaustively proven here.
- **sync signature verify error semantics:** libp2p `PublicKey::verify` returns
  `bool` (infallible) so the Go `(bool, error)` two-arm path collapses to one in
  Rust; this is correct, not a bug. Noted only because an earlier sub-scan flagged
  it — treat as refuted.

## Summary

Scanned all 13 dkg modules + bcast/sync/frostp2p submodules against charon v1.7.1.
The crypto-critical paths (FROST round1/round2 sequencing, per-peer share/cast
validation, partial-signature verification with no silent skips, aggregate
verification, bcast/sync signature verification) are faithful and in several
places stricter than charon; no key-leak, RNG, zeroize, or verification-bypass
defect surfaced. Severity counts: 0C / 0H / 1M / 3L / 2I. The one Medium is a
defensive completeness gap in `make_shares` public-share assembly; Lows are the
divergent dead `nodesigs` None/sentinel path, `has_test_config` including
`p2p_key`, and bcast verifying signatures before the msg-ID registry check; Infos
are lock-file perms style and the FROST ctx-byte truncation. The Medium PoC is
left pending for Phase 2.
