---
name: po-acceptance
description: |
  Run product-owner end-to-end acceptance against success + acceptance
  criteria on a feature branch. Use when:
  (1) Implementation is green but you need PO sign-off before merging
  (2) User says "PO review", "acceptance test", "validate against criteria"
  (3) Inside `/orchestrate`, after all workstreams report COMPLETE
  (4) You want showstopper vs fast-follow triage in one structured verdict
  Spawns a bounded PO-persona subagent (read-only + verify.final smoke). Emits
  a verdict JSON the caller routes: showstoppers loop back to TDD with updated
  criteria; fast-follows are written to a markdown file and do not block.
---

# Product-Owner Acceptance

Delegate end-to-end acceptance validation to a bounded subagent. The caller
never runs the app, writes reports, or triages findings itself — it reads
the subagent's verdict and routes.

## Input

```
{
  "feature_branch": "orchestrate/{run}",
  "run_name": "{run}",
  "success_criteria": [ ... all, across workstreams ... ],
  "acceptance_criteria": [ ... all, across workstreams ... ],
  "ac_map": { "AC1": "auth-service", ... },
  "verify_final": { ... the verify.final config ... }
}
```

## Delegation

Spawn ONE subagent with a PO persona and BOUNDED tools.

```
Agent({
  description: "PO acceptance on {feature_branch}",
  subagent_type: "general-purpose",
  prompt: <see PO prompt template>
})
```

### Tool bounds (enforce in the prompt — the agent self-polices)

The PO agent MUST NOT:
- Install new dependencies (no `npm install`, `pip install`, `apt install`,
  `npx playwright install`). If the smoke needs browsers that are not yet
  installed, the agent reports `blocked: "missing browser install"` and
  STOPS — the caller decides.
- Modify source files (no Edit, Write, NotebookEdit).
- Make commits, merges, or pushes.

The PO agent MAY:
- Read files (Read, Glob, Grep).
- Run the commands declared in `verify_final.commands`.
- Run the smoke described in `verify_final.smoke` IF all prerequisites are
  already installed.
- Run read-only git commands (`git log`, `git diff`, `git status`).
- Run the `check` command declared inside each success criterion.

## PO Prompt Template

```
You are a Product Owner running end-to-end acceptance on a feature branch.
You did NOT implement this work. Your job is to prove each criterion holds
and catch regressions the developers missed.

## Branch under test
  feature_branch: {feature_branch}
  run: {run_name}

## Success Criteria (developer-facing, N total)
{JSON list}

## Acceptance Criteria (user-facing, M total)
{JSON list}

## AC → Workstream map
{JSON}

## verify.final contract
{paste of verify.final YAML/JSON}

## Method

1. For each SUCCESS criterion: run its `check` command, capture exit code
   and the last 20 lines of output. Classify:
     PASS — command exit 0 AND output matches expected signal
     FAIL — command exit != 0 OR output shows wrong signal
     BLOCKED — prerequisite missing (e.g. service not running)

2. For each ACCEPTANCE criterion: walk the user journey it describes.
   - Prefer the verify.final.smoke if it covers the AC path.
   - Otherwise, describe the concrete steps you would take (read code,
     invoke CLI, query the API) and run the read-only ones.
   - Classify PASS / FAIL / BLOCKED / MANUAL.
     MANUAL is reserved for ACs that literally require a human (e.g. 'UX
     feels responsive on a real phone'). Use MANUAL sparingly.

3. Triage every FAIL or BLOCKED finding into one of:
   - SHOWSTOPPER — violates a core acceptance criterion, OR breaks a
     previously-passing success criterion (regression). Release cannot
     ship with this open.
   - FAST_FOLLOW — degrades UX or skips an edge case but all core ACs
     still pass. Can ship with a documented follow-up.

4. For each SHOWSTOPPER: propose an `updated_criteria` delta — add a new
   success criterion that would have caught this (e.g. a missing edge-case
   test). The caller feeds these back into the next TDD loop.

## Do not

- Do NOT edit source files.
- Do NOT install anything.
- Do NOT commit or push.
- Do NOT make up evidence. If a command could not run, mark BLOCKED.

## Deliverable — return EXACTLY this JSON

{
  "verdict": "APPROVED" | "SHOWSTOPPERS" | "FAST_FOLLOWS_ONLY",
  "success_criteria_results": [
    { "id": "SC1", "status": "PASS|FAIL|BLOCKED", "check": "...",
      "exit": 0, "output_tail": "..." }
  ],
  "acceptance_criteria_results": [
    { "id": "AC1", "status": "PASS|FAIL|BLOCKED|MANUAL",
      "method": "smoke|read-code|cli|manual",
      "evidence": "..." }
  ],
  "showstoppers": [
    { "ref": "AC3", "finding": "...", "evidence": "...",
      "proposed_criterion": {
        "id": "SC-NEW-1",
        "text": "...",
        "check": "...",
        "owning_workstream": "..."
      }
    }
  ],
  "fast_follows": [
    { "ref": "AC4", "finding": "...", "severity": "minor|moderate",
      "recommendation": "..." }
  ],
  "updated_criteria": [ /* same shape as proposed_criterion entries above */ ],
  "blocked_preflight": [ "missing browser install", ... ]
}

Rules for the verdict field:
  APPROVED           — no showstoppers, no fast_follows (pure green)
  SHOWSTOPPERS       — showstoppers is non-empty
  FAST_FOLLOWS_ONLY  — no showstoppers but fast_follows is non-empty
```

## After the subagent returns

The caller routes based on `verdict`:

### If `APPROVED`
- Write `.pipeline/$RUN_NAME/po-acceptance.json` (via a writer subagent
  under orchestrate's delegation contract, or directly for standalone use).
- Return the verdict to the caller's caller.

### If `SHOWSTOPPERS`
- Merge `showstoppers[*].proposed_criterion` into the criteria set (these
  become new success criteria).
- Route back to the caller — typically `/orchestrate` restarts Phase 3
  (parallel-workstreams) with the updated criteria.
- Showstopper loop has a cap (2 rounds in `/orchestrate`); the caller
  enforces it. This skill only emits the verdict; it does not loop.

### If `FAST_FOLLOWS_ONLY`
- Spawn ONE "triage-writer" subagent to append the fast-follows to
  `.pipeline/$RUN_NAME/fast-follows.md`:

```
Agent({
  description: "Write fast-follows.md",
  prompt: "Append the following to .pipeline/{run}/fast-follows.md
(create if missing). Each entry becomes a markdown section with
heading '## {ref} — {finding}' and body containing severity,
recommendation, and evidence. Do not modify any other file.

Entries: {JSON array}
"
})
```
- Return verdict APPROVED-with-fast-follows to the caller so it knows to
  proceed to merge and include the fast-follows under 'Next Actions for
  Human' in the acceptance report.

### If `blocked_preflight` is non-empty (regardless of verdict)
- Surface it as a top-of-report item. Do NOT retry the PO until the caller
  resolves the preflight (e.g. installs the browser in a pre-hook).

## Standalone use

`/po-acceptance {feature-branch} {criteria.json}` runs the PO subagent
against an arbitrary branch and criteria set. Useful outside
`/orchestrate` for sign-off on any feature branch.

## Notes

- The PO agent is bounded on purpose. Giving it install privileges leads to
  runaway Playwright installs, gigabytes of browser binaries pulled mid-run,
  and confused CI pipelines. Prerequisites are the caller's job.
- Showstopper triage deliberately lives inside this skill (returned in the
  verdict shape) to avoid a separate triage skill for ≤10 lines of routing.
  The caller does the routing inline.
- `updated_criteria` is the mechanism for closing the TDD loop around
  regressions — the PO's "what would have caught this" answers become the
  first test in the next cycle.
