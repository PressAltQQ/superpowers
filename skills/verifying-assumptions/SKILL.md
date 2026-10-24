---
name: verifying-assumptions
description: "Use when starting execution of a plan that has an Assumptions section (Step 0) - runs the two-tact premise gate (review the list, check env against reality, confirm external behavior) in parallel subagents before any task starts. Never auto-invoked outside the planning pipeline."
---

# Verifying Assumptions (the gate)

## Overview

The **hard gate** between a written plan and its execution. Verifies the plan's `## Assumptions` against **reality and docs** so execution never dies on "ой докера нет / версия другая / такого флага нет". Always present as Step 0 — no whole-plan skip. Empty Assumptions section → **instant PASS no-op**.

**Announce at start:** "I'm using the verifying-assumptions skill to run the Step 0 gate."

**Three pillars, one pass:** A — env state vs reality; B — review the list itself (load-bearing completeness, checkability, severity) with reality in hand; C — confirm premises about external behavior against the resolved version.

**Premises only.** The gate checks what is true before Task 1 starts. It never implements, builds, or replays the plan's own code to prove a claim. A claim about code or data the plan itself will create is acceptance: **DEFERRED** to the step whose test proves it. Be cheap: check the listed premises, not everything imaginable.

## D4 — everything runs in subagents

Every check and analysis runs in a SUBAGENT, never inline in this orchestrator. Tact 2 subagents fan out **in parallel**. The orchestrator only dispatches and merges verdicts.

## Reviewer selection (LOCKED)

- **Default reviewer = an independent Claude subagent** — NOT the plan's author.
- Use **DeepSeek** (`ds` CLI / ask-deepseek) only when Mikhail explicitly asks (third opinion or sole reviewer). Never by default.
- **Never switch model on your own.**

## Probe budget (Tact 2 anti-materialization rule)

Every Tact 2 checker runs read-only probes of about 60 s or less each. Forbidden: installing the project or its dependency set, builds, running against data corpora, creating worktrees, writing stubs, extracting or replaying the plan's code blocks. Patch applicability is `git apply --check` against the pinned commit, nothing more.

**One exception.** For a high-blast dependency-version premise that docs, changelog, and source do not settle, the checker may create ONE throwaway isolated env in `/tmp` containing only that single library at the version in question, and run a minimal snippet. This is how a real BLOCKER ("PyMuPDF 2.0 breaks `Table.extract()` cell text") gets caught.

A premise that cannot be settled within budget: external → **UNKNOWN** (= BLOCKER); plan-internal → **DEFERRED**.

## §6 whitelist (never premises)

`cd, ls, mkdir, cp, mv, rm, echo, cat, pwd, exit` and `git {add, commit, push, pull, status, diff, log}`, plus standard shell/POSIX/coreutils semantics, are never premises: do not check or add them. Existence of the tool on a host is still a premise if a step needs it there. **Destructive commands still need a premise on their target.** Anything verified earlier this session cites the session evidence instead of re-fetching.

## Tact 1 — single resolve-and-gather subagent

Dispatch **one** independent Claude subagent (it needs the **whole-plan view**):

1. **Version resolution (Pillar C, per host):** for each tool/lib resolve `pin > installed (live probe: `tool --version` / `pip show` / `npm ls` / lockfile) > latest stable`, per host. For ranges or unpinned deps resolve BOTH the locally installed version AND what the target will install (CI / fresh install / highest version the range allows, e.g. `pymupdf>=1.24` → latest release; read-only `pip index versions` / `npm view`). Record version + source. Set mismatch verdict: `pin>installed = BLOCKER`, `pin<installed = SOFT`. If the version the plan was grounded on and the version that will run differ by a major version on a high-blast premise, the single-library /tmp probe below is mandatory even if docs look clear (regressions are not documented).
2. **Dedup doc-fetch + cache:** build the DEDUP set of `(lib, version)` pairs across ALL external-behavior premises; fetch each doc source **once**; cache as `(lib,version)→doc-text`. Source priority (local-first): (a) `tool --help` / `tool sub --help` for existence/spelling, `man` / versioned docs for semantics; (b) libraries w/o `--help` → recipe in Tact 2; (c) online versioned docs (WebFetch/WebSearch, version pinned) only for targeted-but-not-installed.
3. **Pillar B — review the list (reality in hand):**
   - **Completeness (must not inflate):** add a missing premise only if it is an external or pre-existing fact a step depends on AND its failure would break a step (load-bearing); at most a few as new `A<n>`. Never add premises about standard shell semantics. Anything about output the plan itself creates (intra-plan consistency, step N not breaking N−1, idempotency) is acceptance → DEFERRED, not a probe.
   - **Checkability:** "Does each `check` actually prove its premise?" For **external-behavior** premises do NOT trust the author's check — flag `trust-author-check: no` for independent doc confirmation in Tact 2.
   - **Severity + Blast radius triage:** re-classify; author's severity is itself an assumption. Default when unsure → **BLOCKER**. Set `blast radius` high/low (drives Tact 2 depth); default high when unsure. Premises about external behavior arrive from the plan WITHOUT `Blast radius` or `Gates steps` (the author's `### Premises about external behavior` list omits them) — assign both here during triage so Type-C blast-radius depth and dependency-traced partial execution are well-defined.

**Tact 1 output (consumed by Tact 2):** resolved Versions table; `(lib,version)→doc-text` cache; triaged list (each with final Type, Severity, Blast radius, Gates steps, `trust-author-check`).

## Tact 2 — parallel subagents, fanned out by Type

Dispatch **one subagent per assumption, in parallel** (per-check timeout). Each uses ONLY Tact 1's cached results. Route by `Type`:

- **Type acceptance:** routed straight to **DEFERRED**, never probed. Tact 1 first rejects/splits any DEFERRED/acceptance item that actually rests on external behavior (e.g. "Task 3's extractor returns correct cells" rests on a library's behavior: split out the library premise and check it). The gate also confirms the `Deferred to: Task N, Step M` step exists and is a test step; if not → **SOFT** with a note telling the executor to add that assertion.

- **Type A (env/resource/partial-state/fs/code/version/data/contract/access/process):** run the check **against reality**; reality is source of truth, docs NOT used.
- **Type C (behavioral):** check the **cached doc** first.
  - If `trust-author-check: no`, independently confirm flag/subcommand/semantics from the cached `--help`/man/doc for the resolved version — do NOT rely on the author's grep (an author who assumes `--profile` and greps `--help | grep profile` rubber-stamps their own hallucination even when the real flag is `--profiles` or must follow `up`).
  - Escalate to a probe ONLY if the doc is silent/ambiguous or doc-trust is low, and only within the Probe budget above — escalation, not default.
  - Still not verifiable within budget → external premise: **UNKNOWN** (= BLOCKER); claim about code or data the plan will create: **DEFERRED**.

**Blast-radius depth (§10):** high → always fully verified, skipping the expensive check is FORBIDDEN; low → expensive check MAY be skipped with the visible verdict **SKIPPED** (non-PASS, shown in the table, note `low blast radius — skipped`; never silent).

### Library doc-grounding recipe (Type C, no `--help`)

- **Python:** version via `pip show <pkg>` / lockfile. Ground via `python -m pydoc <pkg>.<sym>` or `python -c "import <pkg>; help(<pkg>.<sym>)"`, or read installed source under `site-packages`. Not installed → versioned docs (PyPI/readthedocs, version in URL).
- **Node:** version via `npm ls <pkg>` / `package-lock.json`. Ground by reading `node_modules/<pkg>` (`.d.ts`/source) or unpkg/registry pinned to version.
- **Go:** version via `go list -m <mod>`; read module-cache source; `pkg.go.dev/<mod>@<version>`.
- **Other:** man page + `--version`, then official versioned docs.

## Verdicts and the gate decision

The orchestrator merges all Tact 2 verdicts (+ Tact 1 version-mismatch verdicts) into ONE table.

- **PASS** — confirmed. **BLOCKER-fail** — false + load-bearing → gating steps don't start. **SOFT-fail** — proceed with warning. **INFO** — non-blocking. **SKIPPED** — low-blast check skipped, non-PASS, non-blocking. **UNKNOWN** — couldn't run → **treated as BLOCKER**, never silent. **DEFERRED** — concerns code or data the plan itself creates; non-blocking, records the `Deferred to: Task N, Step M` whose test proves it. DEFERRED is not UNKNOWN and not BLOCKER; `UNKNOWN = BLOCKER` still holds for external premises.

### Dependency-traced partial execution

Each premise lists `Gates steps`. Not a binary stop:
- DEFERRED never gates a step.
- Any BLOCKER/UNKNOWN on a step's premises → that step **and its dependents** do not start.
- Independent steps whose premises **all PASS** → may proceed.

### Write the verdict table into plan.md (top of the execution section)

The gate computes and stamps `gate-fingerprint` LAST, after every write into the ass-fp region (verifier ticks, Pillar B additions). Stamp the table with a **freshness fingerprint** of the fingerprinted assumptions region (the lines between the `<!-- ass-fp:start -->` / `<!-- ass-fp:end -->` markers) so an execution gate can skip a redundant re-run while nothing changed. Compute it (canonical command — must match the execution gates byte-for-byte):

```bash
awk '/<!-- ass-fp:start -->/{f=1;next} /<!-- ass-fp:end -->/{f=0} f' <plan.md> | shasum -a 256 | cut -c1-16
```

If the plan has no markers (e.g. a no-premise plan), skip the stamp — the gate is an instant PASS no-op anyway. Otherwise write the fingerprint as an HTML comment on the line directly under the `## Verification Results` heading.

```markdown
## Verification Results
<!-- gate-fingerprint: <hex16> — reuse this table while the ## Assumptions section stays byte-identical -->

| # | Premise | Type | Blast | Verdict | Notes |
|---|---------|------|-------|---------|-------|
| A1 | docker installed on paris | env | high | ✅ PASS | 26.1.0 |
| A2 | port 8080 free | env | low | ⚠️ SOFT | in use by nginx |
| C1 | `docker compose --profile` flag exists | behavioral | high | ✅ PASS | v2.27 docs confirmed |
| A3 | /data/backups writable | fs | high | ❌ BLOCKER | permission denied |
| C2 | new `parse()` handles empty input | acceptance | low | ⏭ DEFERRED | Task 2, Step 4 |
| A4 | optional lint config present | fs | low | ⏭ SKIPPED | low blast radius — skipped |

**SOFT warnings:** A2 — port 8080 occupied by nginx; verify compose port mapping.

> ❌ BLOCKER on A3 — Step 3 (and dependents) will not start. Choose: (a) Task 0 pre-remediation, (b) revise plan, (c) STOP.
```

### On any BLOCKER / UNKNOWN

Surface options: (a) add `Task 0: Environment Readiness` pre-remediation (Task 0 is outside the fingerprinted region, so the fingerprint won't change: after it runs, re-run the gate for the premises it fixed), (b) revise the plan so the premise isn't needed, (c) STOP + ask. SOFT → proceed with warning written in. INFO → table only.

### Re-verify-all on fix

After **any** fix, **re-run ALL checks**, not only the failed ones — a fix can break a previously-true premise.

### In-flight plan edits

Plan edited during execution (steps changed/added) → **re-run the gate for the changed/added steps** before they start.
