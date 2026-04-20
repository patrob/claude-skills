# Acceptance Report (Proof)

Load when spawning the reporter agent. The main orchestrator never writes
this report itself — it spawns a reporter subagent with this reference as
guidance, and the subagent generates `acceptance-report.md` +
`acceptance-report.json`.

## Top-of-Report Rule (non-negotiable)

First thing the user sees: **"Next Actions for Human"**. Quarantined
branches, blocked reviews, scope creep, red verify, unresolved
showstoppers — each with the exact next step. If no outstanding actions,
the section still appears with "None — run is fully green, safe to PR".

## Template

```markdown
# Acceptance Report — {RUN_NAME}

## ⚠ Next Actions for Human

{quarantined branches:} Review quarantined workstream `{name}` — recovered
from dirty state at merge time, NOT merged. Recovery diff:
`.pipeline/{RUN_NAME}/recovery/{name}.diff`. Reason: {quarantine_reason}.
Decide: merge manually after review, or discard.

{blocked reviews:} Unblock review for `{name}` — review agent BLOCKED
after 3 iterations. Feedback: `.pipeline/{RUN_NAME}/reviews/{name}.md`.

{reverted scope creep:} Reverted out-of-scope changes in `{name}` at
`{file}`. If legitimate, re-request as its own workstream.

{verify-final red:} Final verification RED — `{command}` failed. Output:
`.pipeline/{RUN_NAME}/verify-final.json`. Fix attempts exhausted.

{showstopper unresolved:} PO rejected 2 rounds running. Criteria may be
wrong — rewrite before another /orchestrate run.

{fast-follows recorded:} See `.pipeline/{RUN_NAME}/fast-follows.md`.

{all green:} None — run is fully green, safe to PR to base branch.

## Run Summary

| Field | Value |
|-------|-------|
| Run / Feature branch / Base | {values} |
| Started / Completed / Duration | {values} |
| Merge commit | {SHA} |
| Verify status | GREEN / RED / NO-VERIFY |
| Showstopper rounds used | 0-2 |
| PO verdict | APPROVED / FAST_FOLLOWS_ONLY / SHOWSTOPPER_UNRESOLVED |

## Acceptance Criteria

One row per AC from the original spec. Every AC appears.

| AC | Status | Evidence | Source |
|----|--------|----------|--------|
| AC1: ... | ✅ PASS | `npm test -- login` (12/12, 1.3s) | verify.final.test |
| AC3: ... | ❌ FAIL | `npm test -- clip` (3 failing) | verify.final.test |
| AC4: ... | ⚠ MANUAL | no automated check | — |

Each PASS row's full `stdout_tail` lives in the JSON twin.

## Per-Workstream Results

One sub-section per round; one table row per workstream. Columns:
workstream, status, verify attempts, scope deviations, review, merge.

## Verify Final Transcript

From `verify-final.json` — per command name, exit status, duration,
stdout tail.

## Recovery / Quarantine Log

One entry per quarantined workstream (or "None."). Include: reason,
recovery commit SHA, recovery diff path, files touched, recommended next
action.

## Merge Log

```
{git log --oneline --first-parent $BASE..$FEATURE_BRANCH}
```

## Scope Deviations

Table of workstream / file / declared? / action.

## Files Changed

```
{git diff --name-only $BASE...$FEATURE_BRANCH}
```

## How to Proceed

- Review: `git log --oneline $BASE..$FEATURE_BRANCH`
- Diff: `git diff $BASE...$FEATURE_BRANCH`
- PR: `gh pr create --base $BASE --head $FEATURE_BRANCH`
- Merge: `git checkout $BASE && git merge $FEATURE_BRANCH --no-ff`

{Repeat quarantined workstream names so the user does not forget them.}
```

## JSON Twin

`acceptance-report.json` — same info, structured for programmatic
consumers:

```json
{
  "run_name": "...",
  "schema_version": 2,
  "status": "green|red|mixed|showstopper_unresolved",
  "po_verdict": "APPROVED|FAST_FOLLOWS_ONLY|SHOWSTOPPER_UNRESOLVED",
  "showstopper_rounds_used": 0,
  "next_actions": [ { "type": "...", "workstream": "...", "action": "..." } ],
  "feature_branch": "...", "base_branch": "...", "merge_commit": "...",
  "acceptance_criteria": [
    { "id": "AC1", "text": "...", "status": "pass|fail|manual",
      "source": "verify.final.commands.test",
      "evidence": { "command": "...", "exit": 0, "duration_ms": 1342, "stdout_tail": "..." } }
  ],
  "workstreams": [ ... ],
  "verify_final": { ... paste of verify-final.json ... },
  "quarantined": [ ... ],
  "scope_deviations": [ ... ],
  "fast_follows_file": ".pipeline/{RUN_NAME}/fast-follows.md"
}
```

## Success Criteria

Phase is DONE when:
1. `acceptance-report.md` exists and is non-empty.
2. `acceptance-report.json` exists and validates against the shape above.
3. Every AC from the Phase 1 `ac_map` has a row.
4. Every quarantined / blocked workstream is under "Next Actions".
5. The report was printed to the user's terminal (not just written).

A run with no quarantines, no scope deviations, all ACs passing via
automated evidence, `verify_final.status == "green"`, and PO verdict
`APPROVED` is the "proof" the user asked for.
