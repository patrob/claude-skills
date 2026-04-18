# Orchestrate Self-Test Fixtures

These fixtures exercise the reconciliation logic of `/orchestrate` — the
rules that decide quarantine vs merge, retry outcomes, and scope-diff
classifications. They are used by `references/self-test.md`.

## Not Unit Tests

The skill is a markdown prompt, not executable code. There is no function
to call. Instead, each fixture is a **scenario**: a synthetic input
(`input.md`) and the expected state-transition outcome (`expected.json`).
A meta-agent walks the orchestration logic against the fixture and asserts
the outcome matches.

## Running

See `../self-test.md`. A reviewer or pre-release meta-agent runs all
fixtures and reports any mismatch between predicted and expected behavior.
Failure = the skill's rules diverged from this documented contract.

## Fixture Inventory

| Fixture | Tests |
|---------|-------|
| `clean-parallel/` | Three independent workstreams, all pass verify on attempt 1, no deviations, full green |
| `dirty-worktree-recovered/` | One workstream arrives dirty at merge time — must be quarantined with `[recovery]` commit, NOT merged, surfaced in Next Actions |
| `scope-spillover/` | Workstream touches a file outside its scope_globs without declaring it — must be classified as creep, reverted or halted |
| `failed-verify-3x/` | Verify gate fails all 3 retry axes — workstream must be marked FAILED, not merged, verify_failures captured |
| `foundation-overlap/` | Two same-round workstreams' scope_globs intersect — must halt at 3.5c and surface to user |

## What Each Fixture Asserts

Each `expected.json` contains a top-level `assertions` array. Every assertion
is a condition the self-test meta-agent checks against the run outcome:

```json
{
  "assertions": [
    { "path": "workstreams.clip-editor.status", "equals": "quarantined" },
    { "path": "workstreams.clip-editor.merge_skipped", "equals": true },
    { "path": "next_actions[*].type", "contains": "quarantine-review" },
    { "path": "acceptance_report.status", "equals": "mixed" }
  ]
}
```

Path syntax: dot-notation into the acceptance-report.json shape (see
`../phase-5-report.md`). `[*]` = match any array element. Supported operators:
`equals`, `contains`, `not_equals`, `greater_than`, `matches` (regex).

## Adding a Fixture

When adding a new safety rule or reconciliation behavior, add a fixture that
exercises it. The fixture name should describe the scenario (not the rule
number), so the catalog reads as a list of situations the skill handles.
