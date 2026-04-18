# Phase 5 — Acceptance Report (Proof)

Load this reference when executing Phase 5 of `/orchestrate`, after the
final verification agent has written `verify-final.json`.

Phase 5 produces the artifact the user reads at the end of a run:
`.pipeline/$RUN_NAME/acceptance-report.md` (plus a machine-readable twin,
`acceptance-report.json`). This is the proof-of-acceptance deliverable.

## Top-of-Report Requirement (non-negotiable)

The first thing a user sees must be **"Next actions for human"** — not
metadata, not a summary prose paragraph. If the run had any quarantined
branches, blocked reviews, scope deviations, or red verify steps, the user
must see what they need to do before they see anything else.

If there are no outstanding actions, the section still appears with "None —
run is fully green, safe to PR to base branch."

## Full Report Template

```markdown
# Acceptance Report — {RUN_NAME}

## ⚠ Next Actions for Human

{if any quarantined branches:}
- **Review quarantined workstream `{name}`** — recovered from dirty state at
  merge time, NOT merged. Recovery diff: `.pipeline/{RUN_NAME}/recovery/{name}.diff`.
  Reason: {quarantine_reason}. Decide: merge manually after review, or discard.

{if any review-blocked branches:}
- **Unblock review for `{name}`** — review agent marked BLOCKED after 3
  iterations. Review feedback: `.pipeline/{RUN_NAME}/reviews/{name}.md`.

{if scope creep was reverted:}
- **Reverted out-of-scope changes** — in workstream `{name}`, the agent
  touched `{file}` outside its declared scope. Revert was applied. If the
  change was legitimate, re-request it as a separate workstream.

{if verify-final is red:}
- **Final verification RED** — {command name} failed. Output:
  `.pipeline/{RUN_NAME}/verify-final.json`. Fix attempts exhausted.

{if all green:}
- None — run is fully green, safe to PR to base branch.

## Run Summary

| Field | Value |
|-------|-------|
| Run | {RUN_NAME} |
| Feature branch | {FEATURE_BRANCH} |
| Base branch | {base} |
| Started | {ISO timestamp} |
| Completed | {ISO timestamp} |
| Duration | {human} |
| Merge commit | {SHA} |
| Verify status | GREEN / RED / NO-VERIFY |

## Acceptance Criteria

One row per AC from the original roadmap. Every AC must appear.

| AC | Status | Evidence | Source |
|----|--------|----------|--------|
| AC1: User can log in | ✅ PASS | `npm test -- login.test.ts` (12/12 passed, 1.3s) | workstream `auth-service` + verify.final.test |
| AC2: Upload transcodes video | ✅ PASS | smoke: playwright `tests/smoke/upload.spec.ts` (1.8s) | verify.final.smoke |
| AC3: Clip edit saves | ❌ FAIL | `npm test -- clip-edit.test.ts` (3 failing) | verify.final.test |
| AC4: Sessions expire | ⚠ MANUAL | no automated check configured | — |

Each `PASS` row includes a `stdout_tail` excerpt (last 5 lines of the
validating command's output) in the JSON twin — the markdown table links to
it via the Source column.

## Per-Workstream Results

### Round 0 — Foundation
| Workstream | Status | Verify attempts | Scope deviations | Merge |
|-----------|--------|-----------------|------------------|-------|
| shared-types | COMPLETE | 1 | none | clean |

### Round 1 — Core Features
| Workstream | Status | Verify attempts | Scope | Review | Merge |
|-----------|--------|----------------|-------|--------|-------|
| auth-service | COMPLETE | 1 | clean | APPROVED | clean |
| video-api | COMPLETE | 3 (axis: fresh-agent) | 1 declared | APPROVED | clean |
| clip-editor | QUARANTINED | — | — | — | SKIPPED (dirty) |

### Round 2 — Integration
...

## Verify Final Transcript

{From verify-final.json, per command:}

### `typecheck` — PASS (1.1s)
```
{stdout_tail, last 5 lines}
```

### `test` — PASS (3.2s, 147/147 passed)
```
{stdout_tail}
```

### `smoke` (playwright) — PASS (1.8s)
```
{stdout_tail}
```

## Recovery & Quarantine Log

{One entry per quarantined workstream, else "None."}

### `clip-editor`
- Reason: uncommitted work at merge time
- Recovery commit: {SHA} on branch `worktree/clip-editor`
- Recovery diff: `.pipeline/{RUN_NAME}/recovery/clip-editor.diff`
- Files touched in recovery: {list}
- Recommended next action: review the diff, then either
  `git merge worktree/clip-editor --no-ff` after manual validation,
  or `git branch -D worktree/clip-editor` to discard.

## Merge Log

Sequential merge history (git log --oneline --first-parent):
```
{paste}
```

## Scope Deviations

| Workstream | File | Declared? | Action |
|-----------|------|-----------|--------|
| video-api | src/shared/uuid.ts | yes | kept, flagged for final review |

## Review Log

{Per workstream: review verdict, iterations, outstanding items.}

## Files Changed (total)

{deduplicated list from `git diff --name-only {base}...{FEATURE_BRANCH}`}

## How to Proceed

- Review the feature branch: `git log --oneline {base}..{FEATURE_BRANCH}`
- Full diff: `git diff {base}...{FEATURE_BRANCH}`
- Create PR: `gh pr create --base {base} --head {FEATURE_BRANCH}`
- Merge to {base}: `git checkout {base} && git merge {FEATURE_BRANCH} --no-ff`

{If any quarantined branches, repeat their names here so the user does not
forget them.}
```

## JSON Twin

`acceptance-report.json` contains the same information in structured form
for programmatic consumers. Shape:

```json
{
  "run_name": "orchestrate-20260417-211230",
  "status": "green|red|mixed",
  "next_actions": [ { "type": "quarantine-review", "workstream": "clip-editor", ... } ],
  "feature_branch": "...",
  "base_branch": "...",
  "merge_commit": "...",
  "acceptance_criteria": [
    {
      "id": "AC1",
      "text": "User can log in",
      "status": "pass|fail|manual",
      "source": "verify.final.commands.test",
      "evidence": {
        "command": "npm test -- login.test.ts",
        "exit": 0,
        "duration_ms": 1342,
        "stdout_tail": "..."
      }
    }
  ],
  "workstreams": [ ... ],
  "verify_final": { ... paste of verify-final.json ... },
  "quarantined": [ ... ],
  "scope_deviations": [ ... ]
}
```

## Success Criteria for Phase 5

Phase 5 is COMPLETE when:
1. `acceptance-report.md` exists and is non-empty
2. `acceptance-report.json` exists and validates against the shape above
3. Every AC from the Phase 1 `ac_map` has a row in the report
4. Every quarantined or blocked workstream is listed under "Next Actions"
5. The report is printed to the user's terminal (not just written to disk)

A green run with no quarantine entries, no scope deviations, all ACs passing
via automated evidence, and `verify_final.status == "green"` is the "proof"
the user asked for.
