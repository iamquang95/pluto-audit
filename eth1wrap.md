# Audit: eth1wrap  (125 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/eth1wrap/src/*` (lib, build/)
- **Charon:** `app/eth1wrap/{runner,interface}.go`
- **Tier C** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `lib.rs` ↔ `app/eth1wrap/{runner,interface}.go` (execution-layer client wrapper)

## Crate-specific focus
- **Parity:** eth1/EL client connection + interface surface (tiny crate — likely partial port; note what's missing vs Go).
- **Security:** RPC endpoint URL validation.
- **Quality:** minimal; check for stub/unimplemented gaps.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [High] Contract address parsed with strict EIP-55 checksum; rejects valid lowercase/non-checksummed addresses that Go accepts
- **Rust:** `crates/eth1wrap/src/lib.rs:94` (`verify_smart_contract_based_signature`)
- **Charon ref:** [`app/eth1wrap/runner.go:52`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/eth1wrap/runner.go#L52) (`NewDefaultEthClientRunner` erc1271 factory → `common.HexToAddress`)
- **Issue:** Pluto uses `Address::parse_checksummed(contract_address, None)`, which REQUIRES an EIP-55 mixed-case checksum and rejects all-lowercase, all-uppercase, or any mis-cased hex. Go's `common.HexToAddress` is fully lenient — it never validates checksum, truncates/pads, and ignores case. The `contract_address` here is operator/creator-supplied data from a cluster definition (`crates/cluster/src/definition.rs:585,608,643` → `verify_contract_signature` → this fn), and is fed *unmodified* from `operator.address` / `creator.address` strings. Notably the EOA signature path for the *same* address field (`crates/cluster/src/helpers.rs:34` `verify_sig` → `verify_address` → bare `.parse()`) is lenient, so the two verification paths disagree on which addresses are valid.
- **Impact & likelihood:** A legitimate cluster definition carrying a smart-contract (ERC-1271) operator/creator address in lowercase or non-canonical case fails verification with `InvalidAddress` instead of performing the on-chain `isValidSignature` check. Result: valid clusters rejected (availability/parity break); the contract-signature fallback that Go supports is effectively unreachable for non-checksummed inputs. Likelihood: occurs whenever a contract-based signer's address is not stored in exact EIP-55 form (common — many tools emit lowercase). Definitions are not internally re-checksummed before this call. Verified: `verify_contract_signature` (`definition.rs:788`) propagates the `InvalidAddress` as `Err(FailedToVerifyContractSignature)` — so a lowercase contract address makes the whole `verify_signatures` return `Err`, not `Ok(false)`; the address flows unmodified from `operator.address` (`definition.rs:585,608`). The EOA path's `verify_address` (`eth2util/helpers.rs:106`) uses bare `.parse()` (lenient), confirming the two paths disagree.
- **PoC:** CONFIRMED — scratch `#[test]` parsing valid lowercase `0xdac17f958d2ee523a2206206994597c13d831ec7` (installed alloy 1.6.0): `Address::parse_checksummed(lower, None)` → `Err(InvalidChecksum)`; `Address::from_str(lower)` → `Ok(...)`; correctly mixed-case form accepted by `parse_checksummed`. Crate compiles; test deleted, worktree clean.
- **Fix:** Use lenient parsing to match Go's `HexToAddress`: `Address::from_str(contract_address)` / `contract_address.parse::<Address>()` (alloy's `FromStr` for `Address` is case-insensitive and does not enforce checksum), or `Address::parse_checksummed` only when a checksum is actually present. Mirror the same leniency used by `verify_address` in eth2util so both signature paths agree.

### [Low] No reconnect / liveness loop — `Run` and reconnect logic of the Go runner are entirely absent
- **Rust:** `crates/eth1wrap/src/lib.rs:51-105` (`EthClient` impl — no `run`/reconnect)
- **Charon ref:** [`app/eth1wrap/runner.go:76-111,146-161`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/eth1wrap/runner.go#L76-L111) (`Run`, `maybeReconnect`, `checkClientIsAlive`, `close`)
- **Issue:** Go models the EL client as a long-lived `EthClientRunner` with a `Run(ctx)` loop: it (re)dials with exponential backoff (`expbackoff.WithFastConfig`), parks on `reconnectCh`, probes liveness via `BlockNumber`, and tears the client down/reconnects on failure. On any `IsValidSignature`/factory error it signals `maybeReconnect()`. Pluto has none of this: `EthClient::new` dials once (alloy's `RetryBackoffLayer` only retries *rate-limit* errors, not connection loss/liveness), and there is no `Run`, no liveness probe, no reconnect channel, no `Close`. A dropped EL connection is never re-established for the lifetime of the `EthClient`.
- **Impact & likelihood:** After the EL endpoint restarts or the socket drops, every subsequent `verify_smart_contract_based_signature` fails with a transport error until the whole `EthClient` is recreated by the caller; Go would transparently reconnect. Likelihood: certain over a long-running process whenever the EL is restarted/flaps. Note: today's only caller (`cluster`/`dkg`/`cli` create-cluster) uses the client short-lived per command, so blast radius is currently limited — but this is a real functional-parity gap if the wrapper is used for any long-lived duty.
- **PoC:** CONFIRMED — confirmed by inspection (reasoned, Go not run). charon `runner.go:76-111` (`Run` loop), `146-161` (`maybeReconnect`/`checkClientIsAlive` via `BlockNumber`), `close` — none of this exists in `lib.rs`; `EthClient` is a 2-variant enum (`Connected`/`Noop`) with no `run`/reconnect/liveness/`Close`. alloy `RetryBackoffLayer` retries only rate-limit errors, not connection loss. Verified scope claim: callers create the client short-lived per CLI/DKG command.
- **Fix:** Either port the reconnect/liveness loop (a `run` task + liveness probe + reconnect signaling) to match Go, or explicitly document that this crate is intentionally a single-shot connector for short-lived CLI/DKG use and is NOT a drop-in for charon's long-lived `EthClientRunner`. Decide deliberately rather than silently dropping the behavior.

### [Low] RPC endpoint URL accepted without validation; no equivalent of Go's `DialContext` scheme handling, and the `url` dependency is unused
- **Rust:** `crates/eth1wrap/src/lib.rs:71-74` (`new`), `Cargo.toml:11` (`url.workspace = true`)
- **Charon ref:** [`app/eth1wrap/runner.go:43-49`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/eth1wrap/runner.go#L43-L49) (`ethclient.DialContext`)
- **Issue:** `address` is passed straight into alloy's `ClientBuilder::connect(address)`. The `EthClientError::UrlParseError(#[from] url::ParseError)` variant exists and `url` is a declared dependency, but nothing in the crate ever calls `url::Url::parse` — so the variant is dead and `url` is an unused dependency. The only documented "validation" is the empty-string → `Noop` branch. Whether a malformed/non-http(s) endpoint is rejected, and with what error, depends entirely on alloy's `connect` string parsing (which `RetryBackoffLayer` is layered over). Go's `ethclient.DialContext` does its own scheme dispatch (http/ws/ipc) and wraps failures in `"failed to connect to eth1 client"`.
- **Impact & likelihood:** Low correctness risk (a bad URL still errors somewhere), but: (a) misleading API surface — an error variant and dependency that suggest URL validation that does not occur; (b) the endpoint string typically comes from operator config/CLI (`--execution-engine-addr`), so unvalidated/unexpected schemes produce alloy-internal errors rather than a clear, wrapped message. Likelihood: only on malformed/misconfigured endpoints.
- **PoC:** CONFIRMED — confirmed by inspection. `rg url::|Url::` over `crates/eth1wrap/src/` matches only `UrlParseError(#[from] url::ParseError)` at `lib.rs:31`; nothing ever calls `url::Url::parse`, so the variant is never constructed (dead) and `url` is a functionally-unused dependency. `new` passes `address` straight into `ClientBuilder::connect`; only the empty-string→`Noop` branch validates. Matches the finding exactly.
- **Fix:** Either validate the endpoint up front (`url::Url::parse(address)?`, asserting an allowed scheme) so the `UrlParseError` variant and `url` dep are actually used and errors are clear; or, if alloy's parsing is deemed sufficient, remove the dead `UrlParseError` variant and the `url` dependency to keep the surface honest.

### [Low] `EthClient::new` connects eagerly inside the constructor; diverges from Go's lazy/uninitialized runner and conflates construction with I/O
- **Rust:** `crates/eth1wrap/src/lib.rs:56-79` (`new`)
- **Charon ref:** [`app/eth1wrap/runner.go:26-34,37-62`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/eth1wrap/runner.go#L26-L34) (`NewEthClientRunner` / `NewDefaultEthClientRunner` return an *uninitialized* runner; connection happens later in `Run`)
- **Issue:** Go's constructors never dial — `eth1client: nil` until `Run` connects. Pluto's `new` performs the network dial and fails the whole construction if the EL is unreachable. This changes the failure timing/semantics (caller must handle connect errors at construction, and there's no later retry — see the Medium reconnect finding) and makes `new` an async, fallible, side-effecting constructor.
- **Impact & likelihood:** Construction fails if the EL is momentarily down where Go would have started and retried. Mostly an API-shape/parity nit given current short-lived usage. Likelihood: only when EL unreachable at startup.
- **PoC:** n/a
- **Fix:** If matching Go semantics matters, separate construction from connection (lazy connect on first use or an explicit `run`/`connect`); otherwise document the eager-connect contract on `new`.

### [Info] `connect` is not given a timeout/context; Go threads `ctx` through `DialContext` and the whole runner
- **Rust:** `crates/eth1wrap/src/lib.rs:73` (`.connect(address).await`)
- **Charon ref:** [`app/eth1wrap/runner.go:44`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/eth1wrap/runner.go#L44) (`ethclient.DialContext(ctx, url)`), `runner.go:76` (`Run(ctx context.Context)`)
- **Issue:** The pluto dial and `verify_smart_contract_based_signature` take no `Context`/cancellation/timeout; cancellation is left to the caller's outer future. Go plumbs `ctx` through dial, the run loop, and liveness checks. Not a defect on its own, but a structural divergence worth noting for any future long-lived use.
- **Impact & likelihood:** No cooperative cancellation/timeout of a hanging dial except by dropping the future. Negligible for current short-lived callers.
- **PoC:** n/a
- **Fix:** Accept and thread a cancellation/timeout (e.g. `tokio::time::timeout` or a `CancellationToken`) if/when the wrapper becomes long-lived.

## Uncertain / needs-human
- **`parse_checksummed` exact behavior:** Confirmed by reading alloy that `Address::parse_checksummed` enforces EIP-55 and rejects all-lowercase; `Address::from_str` is lenient. Worth a Phase-2 `#[test]` against the installed alloy version to nail the precise rejection (e.g. lowercase 40-hex address) and confirm the High-severity claim. Also confirm whether any caller re-checksums `operator.address` before reaching this fn (searched — none found in `cluster`).
- **Reconnect intent:** Whether dropping charon's `Run`/reconnect loop is a deliberate scope decision (short-lived CLI/DKG usage only) or an unintended omission needs a maintainer call. If long-lived usage is planned, the Medium becomes High.
- **Go `HexToAddress` truncation:** `common.HexToAddress` silently truncates/pads to 20 bytes and ignores invalid hex (yields zero-ish address) rather than erroring. Pluto erroring on malformed input is arguably *better* security, but it is a behavioral divergence; flagging in case strict parity with Go's lenient acceptance is required.

## Summary
Tiny single-shot EL connector. Most important issue: contract addresses are parsed with strict EIP-55 (`parse_checksummed`) while Go (`common.HexToAddress`) and pluto's own EOA path are lenient — valid lowercase/non-checksummed operator/creator addresses fail ERC-1271 verification (High). The crate also omits charon's entire `Run`/reconnect/liveness loop (Low after triage because current callers are short-lived), does no endpoint URL validation despite a dead `UrlParseError` variant + unused `url` dep (Low), and eagerly dials in its constructor unlike Go's lazy runner (Low). No crypto/unsafe/panic-on-untrusted-input issues found.
