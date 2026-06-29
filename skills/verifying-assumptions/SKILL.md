---
name: verifying-assumptions
description: Use as Step 0 of executing any plan that has a ## Assumptions section - runs the two-tact verification gate (review the list, check env against reality, doc-ground behavioral claims) in parallel subagents before execution starts. Never auto-invoked outside the planning pipeline.
---

# Verifying Assumptions (the gate)

## Overview

The **hard gate** between a written plan and its execution. Verifies the plan's `## Assumptions` against **reality and docs** so execution never dies on "ой докера нет / версия другая / такого флага нет". Always present as Step 0 — no whole-plan skip. Empty Assumptions section → **instant PASS no-op**.

**Announce at start:** "I'm using the verifying-assumptions skill to run the Step 0 gate."

**Three pillars, one pass:** A — env state vs reality; B — review the list itself (completeness, checkability, severity) with reality in hand; C — doc-ground behavioral claims to the resolved version.

## D4 — everything runs in subagents

Every check and analysis runs in a SUBAGENT, never inline in this orchestrator. Tact 2 subagents fan out **in parallel**. The orchestrator only dispatches and merges verdicts.

## Reviewer selection (LOCKED)

- **Default reviewer = an independent Claude subagent** — NOT the plan's author.
- Use **DeepSeek** (`ds` CLI / ask-deepseek) only when Mikhail explicitly asks (third opinion or sole reviewer). Never by default.
- **Never switch model on your own.**

## §6 whitelist (no citation needed)

`cd, ls, mkdir, cp, mv, rm, echo, cat, pwd, exit` and `git {add, commit, push, pull, status, diff, log}`. Everything else MUST be doc-grounded. **Destructive commands are grounded even if whitelisted.** Anything verified earlier this session cites the session evidence instead of re-fetching.

## Tact 1 — single resolve-and-gather subagent

Dispatch **one** independent Claude subagent (it needs the **whole-plan view**):

1. **Version resolution (Pillar C, per host):** for each tool/lib resolve `pin > installed (live probe: `tool --version` / `pip show` / `npm ls` / lockfile) > latest stable`, per host. Record version + source. Set mismatch verdict: `pin>installed = BLOCKER`, `pin<installed = SOFT`.
2. **Dedup doc-fetch + cache:** build the DEDUP set of `(lib, version)` pairs across ALL behavioral claims; fetch each doc source **once**; cache as `(lib,version)→doc-text`. Source priority (local-first): (a) `tool --help` / `tool sub --help` for existence/spelling, `man` / versioned docs for semantics; (b) libraries w/o `--help` → recipe in Tact 2; (c) online versioned docs (WebFetch/WebSearch, version pinned) only for targeted-but-not-installed.
3. **Pillar B — review the list (reality in hand):**
   - **Completeness:** "What does this plan depend on that is NOT listed?" incl. intra-plan consistency (step N doesn't break N−1's output), idempotency, ordering. Add missing premises as new `A<n>`.
   - **Checkability:** "Does each `check` actually prove its premise?" For **behavioral** claims do NOT trust the author's check — flag `trust-author-check: no` for independent doc confirmation in Tact 2.
   - **Severity + Blast radius triage:** re-classify; author's severity is itself an assumption. Default when unsure → **BLOCKER**. Set `blast radius` high/low (drives Tact 2 depth).

**Tact 1 output (consumed by Tact 2):** resolved Versions table; `(lib,version)→doc-text` cache; triaged list (each with final Type, Severity, Blast radius, Gates steps, `trust-author-check`).

## Tact 2 — parallel subagents, fanned out by Type

Dispatch **one subagent per assumption, in parallel** (per-check timeout). Each uses ONLY Tact 1's cached results. Route by `Type`:

- **Type A (env/resource/partial-state/fs/code/version/data-contract/access/process):** run the check **against reality**; reality is source of truth, docs NOT used.
- **Type C (behavioral):** check the **cached doc** first.
  - If `trust-author-check: no`, independently confirm flag/subcommand/semantics from the cached `--help`/man/doc for the resolved version — do NOT rely on the author's grep (an author who assumes `--profile` and greps `--help | grep profile` rubber-stamps their own hallucination even when the real flag is `--profiles` or must follow `up`).
  - Escalate to a **sandboxed probe (read-only / crash-safe)** ONLY if the doc is silent/ambiguous or doc-trust is low — escalation, not default.
  - Still not safely verifiable → **UNKNOWN** (= BLOCKER).

**Blast-radius depth (§10):** high → always fully verified, skipping the expensive check is FORBIDDEN; low → expensive check MAY be skipped, recorded `"low blast radius — skipped"` (auditable, never silent).

### Library doc-grounding recipe (Type C, no `--help`)

- **Python:** version via `pip show <pkg>` / lockfile. Ground via `python -m pydoc <pkg>.<sym>` or `python -c "import <pkg>; help(<pkg>.<sym>)"`, or read installed source under `site-packages`. Not installed → versioned docs (PyPI/readthedocs, version in URL).
- **Node:** version via `npm ls <pkg>` / `package-lock.json`. Ground by reading `node_modules/<pkg>` (`.d.ts`/source) or unpkg/registry pinned to version.
- **Go:** version via `go list -m <mod>`; read module-cache source; `pkg.go.dev/<mod>@<version>`.
- **Other:** man page + `--version`, then official versioned docs.

## Verdicts and the gate decision

The orchestrator merges all Tact 2 verdicts (+ Tact 1 version-mismatch verdicts) into ONE table.

- **PASS** — confirmed. **BLOCKER-fail** — false + load-bearing → gating steps don't start. **SOFT-fail** — proceed with warning. **INFO** — non-blocking. **UNKNOWN** — couldn't run → **treated as BLOCKER**, never silent.

### Dependency-traced partial execution

Each premise lists `Gates steps`. Not a binary stop:
- Any BLOCKER/UNKNOWN on a step's premises → that step **and its dependents** do not start.
- Independent steps whose premises **all PASS** → may proceed.

### Write the verdict table into plan.md (top of the execution section)

```markdown
## Verification Results

| # | Premise | Type | Blast | Verdict | Notes |
|---|---------|------|-------|---------|-------|
| A1 | docker installed on paris | env | high | ✅ PASS | 26.1.0 |
| A2 | port 8080 free | env | low | ⚠️ SOFT | in use by nginx |
| C1 | `docker compose --profile` flag exists | behavioral | high | ✅ PASS | v2.27 docs confirmed |
| A3 | /data/backups writable | fs | high | ❌ BLOCKER | permission denied |

**SOFT warnings:** A2 — port 8080 occupied by nginx; verify compose port mapping.

> ❌ BLOCKER on A3 — Step 3 (and dependents) will not start. Choose: (a) Task 0 pre-remediation, (b) revise plan, (c) STOP.
```

### On any BLOCKER / UNKNOWN

Surface options: (a) add `Task 0: Environment Readiness` pre-remediation, (b) revise the plan so the premise isn't needed, (c) STOP + ask. SOFT → proceed with warning written in. INFO → table only.

### Re-verify-all on fix

After **any** fix, **re-run ALL checks**, not only the failed ones — a fix can break a previously-true premise.

### In-flight plan edits

Plan edited during execution (steps changed/added) → **re-run the gate for the changed/added steps** before they start.
