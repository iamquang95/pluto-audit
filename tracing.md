# Audit: tracing  (665 LOC) — STATUS: VERIFIED

- **Pluto:** `crates/tracing/src/*` (init, config, layers/, metrics)
- **Charon:** `app/tracer`, `app/log`, `app/promauto`
- **Tier C** · model: **sonnet** · READ-ONLY (no source edits, no builds)
- **How to run:** use the agent prompt + output schema in `shared/audit/README.md`; overall process in `.plans/repo-audit.md`.

## Module map (pluto ↔ charon)
- `init.rs`/`config.rs` ↔ `app/tracer` + `app/log` setup
- `metrics.rs` ↔ `app/promauto` metric registration

## Crate-specific focus
- **Parity:** tracing/log init, level/format, OTLP exporter config, metric registry setup.
- **Security:** NO secret material in logs/traces/spans.
- **Quality:** subscriber/layer composition correctness.

---
## Findings   ← audit agent appends findings below this line (schema in README.md)

### [Low] Distributed-tracing (OTLP / span exporter) subsystem entirely missing
- **Rust:** `crates/tracing/src/*` (whole crate — no equivalent module)
- **Charon ref:** [`app/tracer/trace.go:1-156`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/tracer/trace.go#L1-L156) (`Init`, `Start`, `RootedCtx`, `WithOTLPTracer`, `WithStdOut`, `newTraceProvider`)
- **Issue:** Charon's `app/tracer` is a full OpenTelemetry tracer: global `TracerProvider`, `Start(ctx, name)` span creation, `RootedCtx` to root spans to a `TraceID`, OTLP-gRPC exporter (`WithOTLPTracer`), stdout JSON exporter (`WithStdOut`), `AlwaysSample()`, service/namespace resource attributes. Pluto's "tracing" crate has none of this — it is logging (console + Loki) + a metrics layer only. There is no span export, no OTLP endpoint config, no trace propagation, no `RootedCtx` analogue (`rg` for `otlp|opentelemetry|otel|tracer|RootedCtx|trace_id` in the crate returns nothing). Charon's `log.Info/Warn/Error` also attach events/errors onto the active span (`trace.SpanFromContext(ctx).AddEvent/RecordError`, `log.go:88-159`); pluto's logging path has no span correlation at all.
- **Impact & likelihood:** No distributed traces are emitted; cross-node duty traces (Tempo/Grafana) that charon produces are absent, and log-to-trace correlation is lost. Always (feature is simply not implemented). Severity lowered to Low unless OTLP/span export is explicitly in current milestone scope; logging/metrics still work.
- **PoC:** confirmed by inspection (no PoC needed) — charon `app/tracer/trace.go` exists with `package tracer` (`Init`/`Start`/`RootedCtx`, otlptracegrpc + stdouttrace exporters, `noop.NewTracerProvider` default). Pluto crate: `rg -i 'otlp|opentelemetry|otel|RootedCtx|trace_id|tracer|TracerProvider'` over `crates/tracing/` → 0 hits; no `opentelemetry*`/`otlp` dep in `crates/tracing/Cargo.toml`; only `init.rs`/`config.rs`/`metrics.rs`/`layers/metrics.rs` exist. Structural absence confirmed.
- **Fix:** Port `app/tracer` as a dedicated module (e.g. `opentelemetry` + `opentelemetry-otlp` + `tracing-opentelemetry` layer), exposing `init`/`shutdown`, OTLP-gRPC and stdout exporters, service/namespace resource, `AlwaysSample`, and a `RootedCtx` equivalent; wire the OTel layer into the `Registry` built in `init.rs`. Or confirm with humans that span tracing is deferred and document the gap.

### [Low] Empty `topic` metric label diverges from charon's `"unknown"`
- **Rust:** `crates/tracing/src/layers/metrics.rs:46` (`event_topic` returns `String::new()`); consumed at `:69-78` (`on_event`)
- **Charon ref:** [`app/log/log.go:62-69`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/log.go#L62-L69) (`metricsTopicFromCtx`)
- **Issue:** Charon maps a missing/empty topic to the literal label `"unknown"` before incrementing `app_log_error_total`/`app_log_warn_total`. Pluto increments with an empty-string label (`topic=""`) when no enclosing span carries a topic. Same metric family, different label series.
- **Impact & likelihood:** Dashboards/alerts/health checks that key on `topic="unknown"` (the charon convention) will not match pluto's `topic=""` series, and untagged errors split into a different bucket. Always, for any error/warn logged outside a topic-bearing span (common — most `tracing::error!`/`warn!` call sites have no topic span).
- **PoC:** CONFIRMED — scratch `scratch_no_topic_uses_empty_not_unknown` in `layers/metrics.rs` (single-threaded) → no-topic `error!` increments `error_total[""]` (EMPTY 0→1) and leaves `error_total["unknown"]` untouched (UNKNOWN 0→0). Charon [`app/log/log.go:62-69`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/log.go#L62-L69) `metricsTopicFromCtx` returns literal `"unknown"` for empty topic (reasoned, Go not run). Divergent label series confirmed.
- **Fix:** In `event_topic`, return `"unknown".to_string()` instead of `String::new()` when no topic is found (and update the `events_without_topic_use_empty_label` test accordingly).

### [Low] Metrics layer reads `topic` only from spans, never from the event's own fields
- **Rust:** `crates/tracing/src/layers/metrics.rs:37-47` (`event_topic`), `:53-61` (`on_new_span`)
- **Charon ref:** [`app/log/log.go:43-46,62-69`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/log.go#L43-L46) (`WithTopic` → `topicKey` in ctx; `metricsTopicFromCtx`), [`app/log/metrics.go:30-35`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/metrics.go#L30-L35)
- **Issue:** Pluto derives the metric `topic` label exclusively from an enclosing span (recorded in `on_new_span`, walked via `event_scope`). It does not inspect the *event's own* fields. But many call sites attach `topic` directly on the event, e.g. `crates/app/src/sse/mod.rs:226,262,301,326` (`tracing::warn!(... topic = HEAD_EVENT, ...)`). In charon the topic lives in the context (`WithTopic`) and is read regardless of how the log line is structured, so those warns are correctly labelled. In pluto they fall through to the empty/"unknown" bucket.
- **Impact & likelihood:** Error/warn counts for event-field-tagged topics (SSE beacon-node events, etc.) are mis-labelled (lost to the no-topic bucket). Always, for every such call site (4+ in `sse/mod.rs` alone).
- **PoC:** CONFIRMED — scratch `scratch_event_field_topic_is_ignored` in `layers/metrics.rs` → `warn!(topic = "...", ...)` with NO enclosing span leaves the labelled series at 0 (LABELED 0→0) and increments the empty bucket instead (EMPTY 0→1). The 4 real call sites verified: `crates/app/src/sse/mod.rs:226,262,301,326` all use `tracing::warn!(... topic = <CONST>, ...)` as an event field (no topic span), so all fall through to `topic=""`. (Charon reads topic from ctx via `WithTopic`/`metricsTopicFromCtx`, independent of field plumbing — reasoned, Go not run.)
- **Fix:** In `on_event`, first run a `TopicVisitor` over the event itself (`event.record(&mut visitor)`); fall back to the span-scope lookup only if the event has no `topic` field. Mirrors charon reading topic from context independent of field plumbing.

### [Low] Default log level is `info`, charon's default is `debug`
- **Rust:** `crates/tracing/src/init.rs:124-126` (`default_env_filter` → `EnvFilter::new("info")`)
- **Charon ref:** [`app/log/config.go:141-147`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/config.go#L141-L147) (`DefaultConfig` → `Level: zapcore.DebugLevel`)
- **Issue:** When no `RUST_LOG`/override is set, pluto defaults to `info`; charon's `DefaultConfig` defaults to `debug`. Note the configuration surface also differs (env var / override string vs an explicit `Level string` field), so this is not a 1:1 mapping — pluto has no `Level` config field at all; level is controlled only through `RUST_LOG` or `override_env_filter`.
- **Impact & likelihood:** Out-of-the-box verbosity differs from charon (debug logs suppressed by default). Always, on default config. Likelihood of operational impact is moderate (operators usually set the level explicitly).
- **PoC:** CONFIRMED — scratch `scratch_default_filter_is_info` in `init.rs` → `default_env_filter().max_level_hint() == Some(LevelFilter::INFO)`; debug suppressed by default. Charon [`app/log/config.go:142-147`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/config.go#L142-L147) `DefaultConfig` sets `Level: zapcore.DebugLevel.String()` (reasoned, Go not run). Default verbosity diverges (info vs debug) as claimed.
- **Fix:** Either default to `debug` for parity, or (better) add a `level`/`format` config field mirroring charon's `Config.Level`/`Config.Format` and document the intended default. Confirm intended default with humans.

### [Low] No console/json/logfmt format selection or charon's opinionated formatting
- **Rust:** `crates/tracing/src/config.rs:60-93` (`ConsoleConfig`), `crates/tracing/src/init.rs:55-61`
- **Charon ref:** [`app/log/config.go:194-214,313-349,357-478`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/config.go#L194-L214) (`console`/`json`/`logfmt`, 4-char levels, topic-as-logger-name, message padding, concise stack traces)
- **Issue:** Charon supports three formats (`console`, `json`, `logfmt`) plus opinionated console rendering (4-char level codes `DEBG`/`ERRO`, green topic prefix, message padding to 40 cols, condensed stack traces, `pretty` field for structured logs). Pluto exposes only the default `tracing_subscriber::fmt` console layer with boolean toggles; no json/logfmt format switch, no level abbreviation, no topic-as-name, no padding. Output format and field set differ substantially from charon.
- **Impact & likelihood:** Log scrapers / golden-output expectations tuned to charon's format will not match; no structured (json/logfmt) output option for machine ingestion outside Loki. Always (cosmetic/format parity; functionally logs still emit).
- **PoC:** n/a
- **Fix:** If format parity matters, add format selection (json/logfmt via `tracing_subscriber::fmt().json()` / a logfmt formatter) and the console conventions; otherwise document the intentional divergence.

### [Low] `override_env_filter` parse failure silently falls back to `info`
- **Rust:** `crates/tracing/src/init.rs:47-51` (`init`)
- **Charon ref:** [`app/log/config.go:118-125`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/config.go#L118-L125) (`ZapLevel` returns an error on bad level)
- **Issue:** A malformed `override_env_filter` (or malformed `RUST_LOG`) is swallowed by `unwrap_or_else(|_| default_env_filter())` and silently downgraded to `info`. Charon surfaces an error (`parse level`) and refuses to init. Violates the "transparent failures" guardrail: an operator typo in the filter directive is masked rather than reported.
- **Impact & likelihood:** Misconfigured verbosity goes unnoticed; intended debug/trace logging silently lost. Only on malformed filter input (operator error).
- **PoC:** n/a
- **Fix:** For the explicit `override_env_filter` path, return an `Error` on parse failure rather than falling back. (Tolerant fallback for ambient `RUST_LOG` is defensible, but the explicit override should be strict.)

### [Low] `init` panics if called twice / a global subscriber is already set
- **Rust:** `crates/tracing/src/init.rs:92,96` (`registry.try_init()` → `Error::Init` is returned, not panicked) — see note
- **Charon ref:** [`app/log/config.go:150-151`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/log/config.go#L150-L151) (`InitLogger` calls `Stop(...)` then re-inits; idempotent re-init supported)
- **Issue:** `try_init` correctly returns `TryInitError` rather than panicking (good), but pluto offers no re-init / `Stop` path. Charon's `InitLogger` stops previously started loggers and can be called repeatedly (e.g. test harness, config reload). Pluto `init` can only ever succeed once per process; a second call errors with no way to tear down and replace the subscriber.
- **Impact & likelihood:** No runtime log reconfiguration; second init always fails. Rare in production (init once) but affects tests/multi-stage setup. Not a panic.
- **PoC:** n/a
- **Fix:** Document single-init contract explicitly, or provide a reload mechanism (e.g. `tracing_subscriber::reload`) if charon's re-init behaviour is needed.

### [Info] Loki credential handling is a deliberate security hardening (no leak) — verified good
- **Rust:** `crates/tracing/src/init.rs:68-112` (`extract_basic_auth`, `strip_userinfo`), `crates/tracing/src/config.rs:35-58` (`LokiConfig` `Debug` redaction, `redact_url_userinfo`)
- **Charon ref:** `app/log/loki/client.go` (n/a — pluto uses the `tracing-loki` crate)
- **Issue:** Not a defect. Pluto moves embedded basic-auth out of the Loki URL into an `Authorization` header (so `tracing-loki`'s error logging cannot echo credentials to stderr/Loki), percent-decodes userinfo before base64 (correct HTTP basic-auth semantics), and redacts userinfo in `LokiConfig`'s `Debug` impl. Tests cover all paths. No secret leakage found in logs/spans across the crate.
- **Impact & likelihood:** n/a (positive finding).
- **PoC:** n/a
- **Fix:** None. Retain.

### [Info] `TopicVisitor::record_debug` quotes non-string topic values
- **Rust:** `crates/tracing/src/layers/metrics.rs:28-32` (`record_debug`)
- **Charon ref:** n/a
- **Issue:** If a `topic` span field is recorded as a non-`&str`/`String` value, `record_debug` stores `format!("{value:?}")`, which for a string-like Debug yields a quoted form (`"foo"`) and for other types a Debug repr — a different label than charon's raw string. In practice all observed call sites pass `&'static str`/`String` (`retry.rs:144`, `health/checker.rs:60`, `metrics.rs` test), which hit `record_str` and are correct. Minor robustness note only.
- **Impact & likelihood:** Label mismatch only for non-string topic values (none currently in tree). Negligibly rare.
- **PoC:** n/a
- **Fix:** Optionally normalise (strip surrounding quotes) or restrict topic to string fields; low priority.

## Uncertain / needs-human
- **OTLP/span-tracing scope:** Is the absence of `app/tracer` (OpenTelemetry span export) an intentional milestone deferral or an oversight? The finding is Low after triage unless span tracing is explicitly in the current milestone. The crate is named `tracing` and docs say "logging, metrics, tracing", but only logging+metrics exist. Confirm whether span tracing is planned.
- **Default level (`info` vs `debug`):** Is `info` the deliberate pluto default, or should it match charon's `debug`? Pluto also lacks a `Level`/`Format` config field entirely (only `RUST_LOG`/override) — confirm whether the charon `Config.Level`/`Config.Format`/`Color`/`LogOutputPath` surface is intended to be ported.
- **`metrics: bool` config field is dead:** `TracingConfig.metrics` (config.rs:15) and its builder setters (`with_metrics`/`metrics`) are never read in `init.rs` — `MetricsLayer` is always added to the registry unconditionally (init.rs:66). Confirm whether metrics are meant to be toggleable (charon registers log counters unconditionally, so always-on may be intended, but then the config flag is misleading dead API). Borderline Low; flagged for human triage.

## Summary
Pluto's `tracing` crate ports charon's logging (console + Loki) and the per-topic error/warn counters, with notably good Loki credential hygiene (no secret leakage; userinfo redacted and moved to an Authorization header). The headline gap is that charon's entire `app/tracer` OpenTelemetry span-export subsystem (OTLP/stdout exporters, `Start`/`RootedCtx`, log-to-span correlation) is unimplemented, but this is Low unless span tracing is in current milestone scope. Metric-label and log-default parity bugs are also Low: they affect observability labels/defaults, not protocol behavior. Remaining gaps: no json/logfmt format options, silent fallback on malformed filter directives, and a likely-dead `metrics: bool` config flag. Counts: 0C / 0H / 0M / 7L / 2I.
