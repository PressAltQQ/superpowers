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
