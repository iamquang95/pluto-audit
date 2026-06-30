# Audit: cli  (13721 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/cli/src/*` (cli, commands/, error, duration, ascii, main)
- **Charon:** `cmd/*.go` (run, dkg, create*, enr, exit*, relay, test*, version, combine, addvalidators)
- **Tier B** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `commands/` ↔ individual `cmd/*.go` files
- `cli.rs`/`main.rs` ↔ `cmd/cmd.go` (root command wiring)
- `duration.rs` ↔ `cmd/duration.go`

## Crate-specific focus
- **Parity:** command set, subcommand names, flag names + shorthands, defaults, env-var bindings, config-file precedence must match charon's cobra setup. Each `test*` subcommand's checks.
- **Security:** path/secret flag handling, no secret echo to stdout/logs.
- **Quality:** clap structure, error UX, help text.

---
## Findings

### [High] `alpha test all` panics: `unimplemented!()`
- **Rust:** `crates/cli/src/commands/test/all.rs:45` (`run`)
- **Charon ref:** [`cmd/testall.go:65`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/testall.go#L65) (`runTestAll`)
- **Issue:** Pluto's `test all` body is `unimplemented!("all test not yet implemented")`. Charon fully runs beacon→validator→mev→infra→peers, aggregates `testCategoryResult`s and writes them. Pluto registers the subcommand (cli.rs:131, dispatched main.rs:112) so it is invokable.
- **Impact & likelihood:** `pluto alpha test all` panics/aborts the process at runtime (not a clean error) · always, the instant the command is run.
- **PoC:** REFINED — scratch `#[tokio::test] test_all_panics_unimplemented` (parse `["pluto","alpha","test","all"]` then call `run`). In a DEBUG build it panics *before* reaching `run`: clap debug-assert `Command all: Argument names must be unique, but 'output_json' is in use by more than one argument or group` — `TestAllArgs` flattens `TestConfigArgs` plus all five category arg structs (each of which re-flattens `TestConfigArgs`), so `output_json`/etc. are duplicated. This means `main.rs:31` (`Cli::command()`) actually debug-asserts on *every* invocation in debug builds, not just `all`. In a RELEASE build clap's debug-asserts are compiled out, parsing succeeds, and `run`'s `unimplemented!("all test not yet implemented")` then panics as the scanner described. Either way `alpha test all` aborts. Severity unchanged (High); note the second latent bug (duplicate flattened args).
- **Fix:** Implement orchestration mirroring `runTestAll` (set each sub-config `quiet=true`, run sequentially, collect results, honor `--quiet`/`--output-json`), or until then remove/hide the subcommand and return a clean `CliError` instead of `unimplemented!`. Also fix the duplicate-flattened-args structure (the category structs should not each re-flatten `TestConfigArgs` when also flattened into `TestAllArgs`).

### [Medium] `test validator` ignores the `--quiet` requires `--output-json` rule
- **Rust:** `crates/cli/src/commands/test/validator.rs:~93-119` (`run`)
- **Charon ref:** [`cmd/testvalidator.go:45`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/testvalidator.go#L45) (`PreRunE → mustOutputToFileOnQuiet`)
- **Issue:** beacon/mev/infra/peers all call `must_output_to_file_on_quiet(...)` first; validator does not — it only branches on `if !quiet`. Charon enforces the rule in the `PreRunE` of every test subcommand.
- **Impact & likelihood:** `pluto alpha test validator --quiet` (no `--output-json`) runs the suite, prints nothing, and exits 0 — results are silently lost, where charon errors `"on --quiet, an --output-json is required"` · always for that flag combination.
- **PoC:** confirmed by inspection (no PoC needed) — `grep must_output_to_file_on_quiet` over `crates/cli/src/commands/test/` returns calls in `peers.rs:223`, `beacon.rs`, `mev.rs`, `infra.rs`, but NONE in `validator.rs`; `validator::run` only branches on `if !args.test_config.quiet` (validator.rs:119) and `if !args.test_config.output_json.is_empty()` (validator.rs:123), so the `(quiet=true, output_json="")` case silently produces no output and returns `Ok`. The helper itself (`helpers.rs:544`) returns `Err` on `(true, "")` (its own unit test `must_output_to_file_on_quiet_output` asserts this). Charon enforces it in `PreRunE` of every test cmd (`testvalidator.go:45-47`). Matches the scanner exactly.
- **Fix:** Call `must_output_to_file_on_quiet(args.test_config.quiet, &args.test_config.output_json)?` at the top of `validator::run`, as in the other four.

### [Low] Test subcommands drop charon's `--log-format`/`--log-level`/`--log-color`/`--log-output-path` flags
- **Rust:** `crates/cli/src/commands/test/mod.rs:26` (`TestConfigArgs`) and per-category arg structs; init in `main.rs:90`
- **Charon ref:** [`cmd/test.go:88`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/test.go#L88) (`bindTestLogFlags`) bound to every test subcommand
- **Issue:** Charon binds 4 log flags to each `test *` command. Pluto's test arg structs expose none of them; main.rs initializes tracing with `TracingConfig::default()` for the whole alpha-test path, so log level/format/output-path can't be configured.
- **Impact & likelihood:** Scripts/docs passing `--log-level debug` (etc.) to `pluto alpha test …` fail to parse (unknown flag); no way to silence/redirect test logs · whenever a log flag is supplied.
- **PoC:** CONFIRMED — scratch `#[test] test_subcommands_reject_log_flags`: `Cli::try_parse_from(["pluto","alpha","test","beacon","--endpoints","http://localhost:5052","--log-level","debug"])` returns `Err` (`unexpected argument '--log-level'`). `TestConfigArgs` (test/mod.rs:27) defines none of the four log flags and `main.rs:90` inits tracing with `TracingConfig::default()` for the whole alpha-test path. Charon binds all four via `bindTestLogFlags` (test.go:88-93) to every test cmd. Matches the scanner.
- **Fix:** Add the four log flags to the shared test config and wire them through `build_console_tracing_config` before running.

### [Low] DKG/relay accept `--log-format` & `--log-output-path` but silently ignore them
- **Rust:** `crates/cli/src/commands/common.rs:32` (`build_console_tracing_config`); `dkg.rs:233` (`DkgLogArgs.format`), `dkg.rs:259` (`log_output_path`); `relay.rs:250` (`RelayLogFlags.format`), `relay.rs:268` (`log_output_path`)
- **Charon ref:** [`cmd/run.go:165`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/run.go#L165) (`bindLogFlags`) — `Format` (console/logfmt/json) and `LogOutputPath` both honored by `log.InitLogger`
- **Issue:** `build_console_tracing_config` only ever builds a console layer and never reads `format` or `log_output_path` (a `// TODO: wire log-output-path` is left at common.rs:31). DKG `try_from`/relay `try_into` pass `level`+`color` only; `--log-format=json` and `--log-output-path=…` are accepted then dropped.
- **Impact & likelihood:** Operators requesting JSON/logfmt logs or on-disk logs get plain console output to stderr with no error — silent misconfiguration · always when those flags are set.
- **PoC:** confirmed by inspection (no PoC needed) — `build_console_tracing_config` (common.rs:32-50) takes only `(level, color, loki)`, builds a console-only layer, never references `format` or `log_output_path` (explicit `// TODO: wire log-output-path` at common.rs:31). DKG `TryFrom<DkgArgs>` (dkg.rs:128) calls `build_console_tracing_config(args.log.level.clone(), &args.log.color, None)` — `args.log.format` (dkg.rs:240) and `args.log.log_output_path` (dkg.rs:263) are parsed then dropped. Relay `TryInto` (relay.rs:94) calls `build_console_tracing_config(self.log.level.clone(), &self.log.color, loki_config)` — `self.log.format` (relay.rs:255) and `self.log.log_output_path` (relay.rs:273) dropped. Charon's `log.InitLogger` honors both. Matches the scanner.
- **Fix:** Honor `format` (console/logfmt/json) and write a file layer for `log_output_path`, or reject unsupported values explicitly instead of ignoring.

### [Medium] `create dkg` drops `holesky` from network help and launchpad-link mapping
- **Rust:** `crates/cli/src/commands/create_dkg.rs:74` (network help text) and `create_dkg.rs:495` (`generate_launchpad_link`)
- **Charon ref:** [`cmd/createdkg.go:102`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/createdkg.go#L102) (help lists `holesky`) and [`cmd/createdkg.go:324`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/createdkg.go#L324) (`generateLaunchpadLink` maps `holesky → "holesky."`)
- **Issue:** Help text omits `holesky` ("mainnet, goerli, sepolia, hoodi, gnosis, chiado" vs charon's list that includes `holesky`). More importantly `generate_launchpad_link` only special-cases `hoodi`/`sepolia`; a holesky cluster gets `https://launchpad.obol.org/dv#…` instead of charon's `https://holesky.launchpad.obol.org/dv#…`.
- **Impact & likelihood:** On `--publish` with `--network=holesky` (still accepted by `valid_network`), operators are pointed at a wrong/mainnet launchpad URL · whenever a holesky cluster is published.
- **PoC:** confirmed by inspection (no PoC needed) — `generate_launchpad_link` (create_dkg.rs:495-506): prefix is set only `if network == HOODI.name || network == SEPOLIA.name`, else empty string → `holesky` yields `https://launchpad.obol.org/dv#…` (mainnet host). Charon's `generateLaunchpadLink` (createdkg.go:317-335) has an explicit `case "holesky": networkLink = "holesky."` → `https://holesky.launchpad.obol.org/dv#…`. `holesky` is in pluto's `SUPPORTED_NETWORKS` (eth2util/network.rs:139), so `valid_network("holesky")==true` (network.rs:223) and `--network=holesky` is accepted. Help text at create_dkg.rs:74 also omits `holesky` (charon's createdkg.go:102 lists it). Matches the scanner. [Go side `reasoned (Go not run)`.]
- **Fix:** Add `holesky` to the `generate_launchpad_link` prefix match and to the network help string (test `test_launchpad_link` should gain a holesky case).

### [Low] `beacon` HTTP status check stricter than charon (rejects 3xx)
- **Rust:** `crates/cli/src/commands/test/beacon.rs:459, 509, 554, 1559` (version/is-synced/peer-count, post)
- **Charon ref:** [`cmd/testbeacon.go:386, 431, 525, 575`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/testbeacon.go#L386) (`resp.StatusCode > 399`)
- **Issue:** Pluto fails the test when `!resp.status().is_success()` (anything outside 2xx), whereas charon only fails on `> 399`, i.e. accepts 3xx as success. (Note the shared `helpers::request_rtt` at helpers.rs:611 correctly uses `> 399`; only beacon.rs diverges.)
- **Impact & likelihood:** A beacon node answering with a 3xx redirect makes the pluto beacon test report `Fail` where charon would proceed · only for beacon endpoints that return 3xx (uncommon but real behind proxies/redirectors).
- **PoC:** confirmed by inspection (no PoC needed) — beacon.rs uses `if !resp.status().is_success()` (2xx-only) at the version (459), is-synced (509), peer-count (554) and get_current_slot (1559) checks, each with a self-acknowledging comment "More strict than the Charon check, which requires the status code to be > 399." Charon (testbeacon.go:386/431/525/575) uses `resp.StatusCode > 399`, accepting 3xx. So a 3xx response → pluto `Fail`, charon `OK`. NOTE: `reqwest::Client` follows redirects by default, so in practice a 3xx is usually followed before this check sees it; the divergence bites only when the redirect target itself is non-2xx or redirects are exhausted/disabled. Behavioral gap is real but practical likelihood is low because reqwest follows redirects by default — Severity lowered to Low. [Go side `reasoned (Go not run)`.]
- **Fix:** Replace `!resp.status().is_success()` with `resp.status().as_u16() > 399` to match charon.

### [Low] Relay command uses `PLUTO_*` env-var bindings instead of charon's `CHARON_*`
- **Rust:** `crates/cli/src/commands/relay.rs:131-292` (all relay arg `env = "PLUTO_*"`)
- **Charon ref:** [`cmd/cmd.go:30`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/cmd.go#L30) (`envPrefix = "charon"`) + `bindFlags` (cmd.go:132) — every flag binds to `CHARON_<FLAG>`
- **Issue:** dkg/enr/create-enr use `CHARON_*` env vars (matching charon), but the relay command binds `PLUTO_HTTP_ADDRESS`, `PLUTO_P2P_RELAYS`, `PLUTO_LOG_LEVEL`, etc. Inconsistent within pluto and divergent from charon.
- **Impact & likelihood:** Operators relying on `CHARON_*` env config for the relay (or expecting one consistent prefix) get defaults instead · whenever relay config is supplied via env.
- **PoC:** n/a
- **Fix:** Decide a single prefix; for charon parity use `CHARON_*` across all commands (or document the intentional `PLUTO_*` rename and apply it everywhere).

### [Low] DKG skips charon's up-front relay-URL validation
- **Rust:** `crates/cli/src/commands/dkg.rs:285` (`validate_p2p_args`) and `relay.rs` `try_into`
- **Charon ref:** [`cmd/run.go:180-197`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/run.go#L180-L197) (`bindP2PFlags` PreRunE: `url.Parse` each relay, warn on `http` scheme, `idna.Lookup.ToASCII(ExternalHost)`)
- **Issue:** Charon validates every `--p2p-relays` entry with `url.Parse` and validates `ExternalHost` via IDNA punycode in a PreRunE. Pluto's `validate_p2p_args` only checks `external_host` (via `url::Host::parse`, not IDNA), and relay parsing happens lazily during config conversion. Error timing/semantics differ (e.g. clap value-validation vs domain error; no IDNA normalization).
- **Impact & likelihood:** Mostly cosmetic divergence in when/how a bad relay or non-ASCII hostname is reported · only on malformed `--p2p-relays`/`--p2p-external-hostname`.
- **PoC:** n/a
- **Fix:** Validate relays/hostname in one place mirroring charon; use IDNA `ToASCII` for hostname parity if relevant downstream.

### [Low] `test` queued lists not sorted for peers/validator/mev (only beacon & infra sort)
- **Rust:** `crates/cli/src/commands/test/peers.rs:236-244`, `validator.rs:~93`, `mev.rs:115-125` (no `sort_tests`); cf. `beacon.rs:306`, `infra.rs:583` which do sort
- **Charon ref:** [`cmd/testpeers.go:157-165`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/testpeers.go#L157-L165), [`cmd/testvalidator.go:84`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/testvalidator.go#L84), [`cmd/testmev.go:121`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/testmev.go#L121) (all call `sortTests`)
- **Issue:** Charon sorts every filtered list by `testCaseName.order`; pluto only sorts beacon and infra. peers/validator/mev rely on the static declaration order of their `*_test_cases()`/`::all()` arrays.
- **Impact & likelihood:** Test execution/printed order can diverge from charon if the static arrays aren't already in `order` order; cosmetic, no correctness impact · always (ordering only).
- **PoC:** n/a
- **Fix:** Call `sort_tests` (or guarantee declaration order matches `order`) for peers/validator/mev.

### [Low] MEV block timestamp uses `unsigned_abs()` instead of signed Unix time
- **Rust:** `crates/cli/src/commands/test/mev.rs:295` (`create_mev_block` path)
- **Charon ref:** [`cmd/testmev.go:323`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/testmev.go#L323) (`time.Unix(latestBlockTSUnix, 0)`)
- **Issue:** Pluto parses the block timestamp as `i64` then `UNIX_EPOCH + Duration::from_secs(ts.unsigned_abs())`. A negative timestamp would be mapped to its magnitude (wrong) rather than a pre-epoch time; charon handles the signed value directly.
- **Impact & likelihood:** Beacon execution-payload timestamps are always positive (post-genesis), so this never triggers in practice · negligibly rare.
- **PoC:** n/a
- **Fix:** Reject negative timestamps with an error (or use signed handling) instead of `unsigned_abs()`.

### [Low] `TestnetConfig → network::Network` leaks heap memory via `Box::leak`
- **Rust:** `crates/cli/src/commands/create_cluster.rs:333, 341` (`From<TestnetConfig> for network::Network`)
- **Charon ref:** n/a (Rust-only design choice)
- **Issue:** Converts owned `String` testnet name / fork-version into `&'static str` with `Box::leak` to satisfy the `network::Network<'static>` field types.
- **Impact & likelihood:** Permanent (small) heap leak per `create cluster` invocation using a custom testnet · only with `--testnet-*` flags. Harmless for a one-shot CLI process but an anti-pattern.
- **PoC:** n/a
- **Fix:** Make `network::Network` own its strings (or use `Cow<'static, str>`) so no leak is needed.

### [Low] `version` does not emit per-dependency versions like charon
- **Rust:** `crates/cli/src/commands/version.rs:38` (`dependencies()`)
- **Charon ref:** [`cmd/version.go:51-66`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/version.go#L51-L66) (`debug.ReadBuildInfo()` → each dep `Path Version`)
- **Issue:** Charon's verbose output lists every build dependency with resolved version (and follows `Replace`). Pluto prints a static `pluto_core::version::dependencies()` list. Likely not the full resolved dependency graph.
- **Impact & likelihood:** Verbose `version --verbose` output is less complete than charon's; diagnostic only · always (verbose mode).
- **PoC:** n/a
- **Fix:** Confirm `dependencies()` reflects the real build graph; if not, generate it at build time. (Verify `pluto_core::version` — out of cli crate.)

### [Info] Several `commands::*` commands not ported (combine, exit*, addvalidators, run, unsafe)
- **Rust:** `crates/cli/src/cli.rs:34` (`Commands`), `main.rs:65`
- **Charon ref:** [`cmd/cmd.go:36-69`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/cmd.go#L36-L69) (`New()` registers version, enr, run, relay, dkg, create*, combine, alpha{addvalidators,test}, exit{list,sign,bcast,fetch,delete}, unsafe)
- **Issue:** Pluto registers only enr/create/version/relay/dkg/alpha-test. Missing: `run`, `combine`, `alpha addvalidators`, the whole `exit` subtree, and `unsafe`. Likely intentional for the current port stage, but a parity gap to track.
- **Impact & likelihood:** Those commands are unavailable in pluto · always (commands simply absent).
- **PoC:** n/a
- **Fix:** Track as porting backlog; ensure CLI help/docs don't promise unported commands.

### [Info] No viper-style config-file / env precedence layer
- **Rust:** `crates/cli/src/main.rs:31-33`, `cli.rs` (clap only)
- **Charon ref:** [`cmd/cmd.go:106-164`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/cmd.go#L106-L164) (`initializeConfig`/`bindFlags`: reads `charon.{yaml,toml,…}` from `.`, `AutomaticEnv`, `-`→`_` replacer, flag>config>env precedence)
- **Issue:** Charon supports a `charon` config file plus env autobinding with a defined precedence (CLI flag wins, then config file, then env). Pluto uses clap's per-flag `env=` only — no config-file support and per-command env prefixes.
- **Impact & likelihood:** Users with a `charon.yaml`/`.toml` config file get no effect under pluto · whenever a config file is relied upon.
- **PoC:** n/a
- **Fix:** If config-file parity is required, add a config layer feeding clap defaults; otherwise document the omission.

### [Info] DKG/relay redact behavior differs from charon's `redact()`
- **Rust:** `crates/cli/src/commands/relay.rs:347` (`info!(config = ?config)`), `dkg.rs` (no flag dump)
- **Charon ref:** [`cmd/cmd.go:178-228`](https://github.com/ObolNetwork/charon/blob/v1.7.1/cmd/cmd.go#L178-L228) (`printFlags`/`redact`: redacts `*auth-token*` → `xxxxx` and password in `*address*` URLs)
- **Issue:** Charon logs all parsed flags at startup with auth-token and URL-embedded passwords redacted. Pluto's relay logs the whole `config` via `?` Debug (relay.rs:347); dkg doesn't dump flags. If any address/token ends up in the relay `Config` Debug output, redaction parity isn't guaranteed (relay config currently has no auth-token field, so low risk).
- **Impact & likelihood:** Potential secret-in-logs if future config gains token/credential fields and Debug isn't redacted · low today (no secret fields in relay Config).
- **PoC:** n/a
- **Fix:** Avoid `?`-Debug-dumping whole configs; if dumping flags, port `redact()` for `*auth-token*`/`*address*` parity.

## Uncertain / needs-human
- **`version::dependencies()` completeness** (version.rs:38): whether it equals charon's full `debug.ReadBuildInfo` dep graph is defined in `pluto_core::version` (outside cli) — verify there.
- **Relay `monitor_addr`/`debug_addr` defaults**: pluto uses `Option<String>` (monitor `None`, debug `Some("")`) vs charon empty-string defaults; both mean "disabled", but confirm the relay-server config treats `Some("")` and `None` identically (relay-server crate, out of scope).
- **`duration.rs` rounding edge cases** (round, duration.rs:30): charon `RoundDuration` rounds on full `time.Duration` nanos via `d.Round`; pluto rounds on truncated `as_millis()` in the `>1s` branch. The included `test_round` cases pass, but tie/boundary cases near 5ms increments may differ — needs a differential test vs Go.
- **`--network` enum vs free string**: create_cluster uses a clap `ValueEnum` (rejects unknown at parse with a clap error) while create_dkg/charon use a free string with a domain error. Confirm no script depends on charon's exact unknown-network error text/exit semantics.
- **Test-case name parity**: per-category `*_test_cases()` name strings vs charon `supported*TestCases()` keys were not exhaustively diffed; a name mismatch would silently drop a `--test-cases` selection. Worth a Phase-2 name-set diff per category.

## Summary
The CLI is a faithful clap port of charon's cobra tree for the commands it implements; flag names/defaults/shorthands and the create-dkg/create-cluster/dkg/relay/enr/version flows largely match. Top issues are functional-parity gaps rather than memory-safety bugs: `alpha test all` is a runtime `unimplemented!()` panic (High); `test validator` skips the `--quiet`⇒`--output-json` guard (Medium); create-dkg mishandles `holesky` launchpad links (Medium). Log-flag gaps and beacon 3xx handling are Low after triage: real parity gaps, but operational/config-output only. Lower-severity: relay uses `PLUTO_*` env vars vs charon's `CHARON_*`, missing test-list sorting, `Box::leak` for testnet config, and absence of charon's viper config-file/env precedence layer. Several whole command subtrees (run, combine, exit*, addvalidators, unsafe) are not yet ported (Info). No crypto/secret-handling or panic-on-untrusted-input defects found in this crate.
Counts: 0C / 1H / 2M / 9L / 3I (15 findings).
