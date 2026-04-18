# Phase 3 — Review Each Branch

Load this reference when executing Phase 3 of `/orchestrate`.

After each round completes, review all branches from that round before merging.
See `review-loop.md` for the full protocol.

## 3a. Launch Review Agents (parallel)

Launch a review agent for EACH branch in the round simultaneously:

```
Agent({
  description: "Review: {workstream name}",
  subagent_type: "rpi:code-reviewer",
  prompt: "Review the complete implementation of workstream: {workstream name}

  Context: This workstream was developed in isolation as part of a parallel
  orchestration. It ran its own research/plan/implement/verify pipeline and
  passed the verify gate ({attempts used} attempts).

  The agent's self-review notes: {paste from workstream report}

  Assigned scope globs: {scope_globs}
  Actual files changed: {git diff --name-only output}

  Review all changes on this branch. Focus on:
  1. Correctness — does it meet the acceptance criteria?
  2. Security — no vulnerabilities, proper auth, no secrets
  3. Conventions — follows project patterns
  4. Test quality — adequate coverage, meaningful assertions
  5. Integration readiness — will this merge cleanly with the broader feature?
  6. Scope discipline — any changes outside scope_globs?

  Output APPROVE or REQUEST_CHANGES with specific actionable items.
  Only request changes for CRITICAL and HIGH severity items."
})
```

## 3b. Fix Cycle (max 3 iterations per branch)

If `REQUEST_CHANGES`:
1. Launch fix agent in the worktree branch with the review feedback
2. Fix agent re-runs verify gate after changes (see `retry-semantics.md`)
3. Re-review
4. Repeat up to 3 times
5. After 3 iterations: merge with warning flag; outstanding items go into the
   Phase 5 acceptance report under "Outstanding review items"

See `review-loop.md` for full details.

## 3c. Review Completion Criteria

A branch exits Phase 3 in one of three states:

| State | Meaning | Next |
|-------|---------|------|
| APPROVED | No blocking issues | Proceed to Phase 3.5 |
| APPROVED-WITH-WARNINGS | Review exhausted 3 iterations, non-critical issues remain | Proceed to Phase 3.5 but flag in report |
| BLOCKED | Critical issue unresolved after 3 iterations | Skip Phase 3.5 and Phase 4; mark `status: review-blocked` in state.json |

Blocked branches do NOT merge. They appear in the Phase 5 acceptance report
under "Next actions for human" with the review feedback attached.
