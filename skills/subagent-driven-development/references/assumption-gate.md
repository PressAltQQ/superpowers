# Step 0: Assumption-Verification Gate

Shared by superpowers:subagent-driven-development and
superpowers:executing-plans. Hard gate: it runs before the first task, every
time. In this file "task" means a dispatched task (subagent-driven) or a task
you work yourself (executing-plans).

## Freshness check (skip-guard)

Only when the plan contains a `<!-- ass-fp:start -->` marker **and** a
`## Verification Results` table with a `gate-fingerprint` stamp, recompute the
current fingerprint with the canonical command:

```bash
awk '/<!-- ass-fp:start -->/{f=1;next} /<!-- ass-fp:end -->/{f=0} f' <plan.md> | shasum -a 256 | cut -c1-16
```

- **Match:** the gate already ran and the assumptions region is unchanged.
  Reuse the existing table and honor its verdicts (do not start tasks gated
  by a BLOCKER or UNKNOWN). Do NOT re-dispatch verification subagents.
  Announce "Assumptions unchanged since verification — reusing gate results."
- **Mismatch, or any of {no `ass-fp` markers, no table, no stamp}:** run the
  full gate below. A plan without markers hashes to the empty-input value
  `e3b0c44298fc1c14`; never treat that as a match, it means the plan predates
  fingerprinting.

## Full gate

1. **REQUIRED SUB-SKILL:** Use superpowers:verifying-assumptions.
2. It verifies the plan's `## Assumptions` in subagents and writes a
   `## Verification Results` table (with a fresh `gate-fingerprint`) into the
   plan.
3. Start a task only when its gating premises are PASS. BLOCKER or UNKNOWN on
   a task's premises: that task and its dependents do not start; independent
   all-PASS tasks may proceed. DEFERRED blocks nothing: the deferred-to
   step's test is the proof.
4. Empty Assumptions is an instant PASS no-op; continue.
5. After any fix, re-verify ALL checks. A plan edited mid-execution
   fingerprints differently: re-run the gate for the changed or added tasks
   before they start.

Never skipped, never run inline: the gate runs in subagents. When executing
inline (executing-plans) you still dispatch the gate to subagents; if your
harness has no subagent tool, say so to your human partner rather than
running it yourself.

Run Step 0 before `task-start` for Task 1, and before the pre-flight scan.
It does not change how `task-start` and `task-done` work.
