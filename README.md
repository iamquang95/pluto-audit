# Pluto repo-level audit — playbook

**Output:** one findings file per unit at `shared/audit/<unit>.md` (repo-relative path), schema below. Map-only — no merged/triaged report; the per-unit files are the deliverable.
**Dimensions:** (1) parity vs charon Go, (2) security, (3) Rust quality & idiom.
**Reference:** charon v1.7.1 at `shared/charon/…` — READ it, never fetch github.

**Two-pass pipeline:**
1. **Scan (Phase 1)** — read-only fan-out, one agent per unit. Finds issues and fills every field of the schema EXCEPT the PoC (Critical/High/Medium get `PoC: ⏳ pending`). No builds, no source edits — fast, parallel, cheap.
2. **Verify (Phase 2)** — one *independent* agent per unit that has Critical/High/Medium findings. In an isolated worktree it tries to *reproduce or refute* each such finding with a minimal `#[test]`/snippet, then writes CONFIRMED / REFUTED / REFINED + observed output into the PoC field (adjusting severity if the evidence warrants). A fresh agent mandated to disprove catches scanner overstatements.

Full process + waves: `.plans/repo-audit.md`.

## Finding schema — the inline report for each issue

```
### [SEV] <short title>
- **Rust:** `crates/<crate>/<file>.rs:LINE` (`fn / struct`)
- **Charon ref:** `<go file>:LINE` (`FuncName`) | n/a (Rust-only issue)
- **Issue:** <what diverges from charon / what is wrong>
- **Impact & likelihood:** <consequence if triggered> · <how likely: always / only on malformed input / negligibly rare (~2⁻¹²⁸) / …>
- **PoC:** <test name + observed result | code snippet | ⏳ pending (Phase 2) | confirmed by inspection (no PoC needed)>
- **Fix:** <concrete fix>
```

**Worked example:**

```
### [High] save: private key written world-readable
- **Rust:** `crates/k1util/src/k1util.rs:202` (`save`)
- **Charon ref:** [`app/k1util/k1util.go:147`](https://github.com/ObolNetwork/charon/blob/v1.7.1/app/k1util/k1util.go#L147) (`Save`)
- **Issue:** `std::fs::write` uses umask perms (0644); Go passes 0o600
- **Impact & likelihood:** ENR signing key readable by any local user · always (every save)
- **PoC:** ⏳ pending (Phase 2)        ← scanner leaves this; verifier fills it
- **Fix:** OpenOptions + `.mode(0o600)` on Unix
```

Severity: **Critical / High / Medium / Low / Info**. File header: `# Audit: <unit>  (<LOC> LOC) — STATUS: SCANNED|VERIFIED`. Each file also has a `## Summary` (2–4 lines) and `## Uncertain / needs-human`.

## Phase 1 — scan agent prompt (template)

> You are auditing the **<CRATE>** crate of **pluto**, a Rust port of ObolNetwork/charon (Go)
> v1.7.1. This is a **READ-ONLY scan**: do NOT edit source, do NOT run builds. Find issues only —
> reproduction/PoC happens in a separate later pass.
>
> Cover three dimensions:
> 1. **Parity vs charon Go** (golden rule = functional equivalence). Find the charon counterpart
>    under `shared/charon/<PATH>` (confirm by reading)
>    and compare: missing logic, divergent control flow, wrong constants/defaults, error
>    semantics, edge cases handled in Go but not here (or vice versa), ordering, concurrency
>    model. Read Go — never guess.
> 2. **Security** — crypto correctness, key/secret handling, input validation, panics on
>    untrusted input, `unsafe`, integer overflow, DoS surface, randomness, timing.
> 3. **Rust quality & idiom** — `unwrap`/`expect`/`panic` on non-invariants, API design,
>    dead/duplicated code, needless clones/allocs, missing docs on non-obvious invariants.
>
> For EVERY finding fill the inline schema in this dir's README EXACTLY: **Rust** `file:line`
> (+ fn/struct), **Charon ref** `file:line` (+ func, or `n/a`), **Issue**, **Impact &
> likelihood** (consequence + how likely it triggers), **Fix**. Set **PoC** = `⏳ pending (Phase 2)`
> for Critical/High/Medium and `n/a` for Low/Info. Severity Critical/High/Medium/Low/Info; mark
> genuinely uncertain items under `## Uncertain / needs-human`.
>
> Fill `## Findings` + `## Uncertain` + a `## Summary` in
> `shared/audit/<CRATE>.md`; set `STATUS: SCANNED`.
> Nothing else. Concrete + deduplicated, no filler. Reply with the file path, severity counts,
> and the single most important finding.

## Phase 2 — verify agent prompt (template)

> You independently verify the audit findings in
> `shared/audit/<CRATE>.md` for pluto's **<CRATE>**
> crate (Rust port of charon v1.7.1). You are in an **isolated git worktree** and a FRESH
> reviewer — do NOT trust a finding; try to REPRODUCE and, where you can, REFUTE it.
>
> For each Critical/High/Medium finding whose **PoC** is `⏳ pending`:
> - Write a minimal `#[test]` (or standalone snippet) exercising the exact claim — prefer a test
>   that would FAIL if the finding were wrong. Run `cargo test -p <CRATE> <name>`, capture the
>   observed result, then DELETE the scratch test (worktree must end clean).
> - For Go-side behavior you cannot execute, read the charon source and reason; mark it
>   `reasoned (Go not run)`.
> - If the build env can't compile the crate (protoc, C deps), say so and fall back to reasoned.
>
> Update each finding's **PoC** field in place with ONE of: `CONFIRMED — <test> → <observed>`,
> `REFUTED — <observed>`, `REFINED — <what actually happens>`, or
> `confirmed by inspection (no PoC needed)` for self-evident issues. If evidence changes the
> picture, adjust **Severity** and **Impact & likelihood** and note the change. Touch nothing
> else in the file; set `STATUS: VERIFIED`.
>
> Reply: per finding, CONFIRMED/REFUTED/REFINED + one-line observed result + any severity change.

## Unit inventory

Authoritative unit list, LOC, tier, and wave order live in `.plans/repo-audit.md`
(26 units; `core` split into 5, `build-proto` skipped). Each unit's charon ref +
module map + crate-specific focus live in its own `shared/audit/<unit>.md`.

**Model:** all units run **opus**. **Concurrency:** waves of ~5–6 agents, order A → B → C.
