# Orchestrate Self-Test

This reference is the "test" analog for a markdown-only skill. A meta-agent
walks the orchestration rules against each fixture in `fixtures/` and checks
that predicted behavior matches the expected assertions.

## When to Run

- Before tagging a new version of the skill
- After modifying any phase reference (`phase-*.md`)
- When debugging a report of unexpected orchestration behavior

## How to Run

Launch a meta-agent with the following prompt:

```
You are the orchestrate self-test agent. Do NOT execute a real /orchestrate
run. Instead, for each fixture in this directory, simulate what the skill's
rules would produce and check against the fixture's expected.json.

Fixtures to run:
- clean-parallel
- dirty-worktree-recovered
- scope-spillover
- failed-verify-3x
- foundation-overlap

For each fixture:
1. Read fixtures/{name}/input.md. Treat it as the input to /orchestrate.
2. Read fixtures/{name}/expected.json.
3. For each phase (1 → 5), apply the rules from the corresponding
   references/phase-*.md against the fixture's simulated agent outcomes
   (described in the input.md under "Simulated Agent Outcomes").
4. Produce a simulated acceptance-report.json in memory.
5. For each assertion in expected.json.assertions:
   - Resolve the `path` against your simulated report
   - Apply the operator (equals, contains, matches, not_equals, greater_than)
   - Record PASS or FAIL with the actual vs expected value

Output format:

# Self-Test Results

## clean-parallel
  ✓ status equals "green"
  ✓ workstreams.auth-service.status equals "complete"
  ...
  RESULT: 13/13 PASS

## dirty-worktree-recovered
  ✓ status equals "mixed"
  ✓ workstreams.clip-editor.status equals "quarantined"
  ✗ workstreams.clip-editor.recovery_commit matches "^[a-f0-9]{7,40}$"
    actual: null (simulated recovery did not run)
  ...
  RESULT: 14/15 PASS (1 rule divergence)

## Summary

  Fixtures run: 5
  All assertions: 67
  Passing: 66
  Failing: 1
  Divergences (skill rules differ from documented contract):
    - dirty-worktree-recovered: recovery_commit not generated in simulated path

Each FAIL indicates EITHER:
  (a) the skill rule was changed and the fixture needs updating, OR
  (b) the rule change introduced a regression and the skill needs reverting.
Never silently update a fixture to match a changed rule — surface the
divergence to the human reviewer first.
```

## Path Syntax Reference

Used in fixture assertion `path` fields:

| Syntax | Meaning |
|--------|---------|
| `status` | Top-level field `status` |
| `workstreams.auth-service.status` | Nested field |
| `next_actions[*].type` | Any element of array |
| `acceptance_criteria[id=AC1].status` | Array element where `id==AC1` |
| `workstreams.auth-service.verify_attempts[0].axis` | Indexed array access |

## Operators

| Operator | Meaning |
|----------|---------|
| `equals` | Deep equality |
| `not_equals` | Inverse of equals |
| `contains` | Substring (string) or element-in (array) |
| `matches` | Regex match (string only) |
| `greater_than` | Numeric > |

## What Passing Means

All 5 fixtures passing = the skill's documented rules are internally
consistent and unchanged from their last approved state. It does NOT mean
the rules produce a real green orchestration — that requires a live run on
a real codebase.

## What Failing Means

A fixture divergence is a signal to investigate, not an automatic failure.
Review the divergence with a human before:
- Updating the fixture to match new behavior, OR
- Reverting the rule change that caused the divergence.

The self-test is a tripwire, not an oracle.
