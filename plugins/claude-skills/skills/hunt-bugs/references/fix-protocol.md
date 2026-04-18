# Fix Protocol

Loop spec for the Phase 3 fixer agent. One fixer agent per confirmed bug, up
to 5 attempts, sequential across bugs.

## Attempt loop

Each attempt is a closed cycle. The agent must complete all four steps before
starting the next attempt.

```
attempt N of 5
├─ 1. Theory
│    Read the failing test output and the hypothesis evidence.
│    State the root-cause theory in one sentence BEFORE editing anything.
│    If this is attempt ≥ 2, explicitly state what the previous theory got
│    wrong and what new information shifted the theory.
│
├─ 2. Edit
│    Make the MINIMAL production-code change.
│    Do not refactor adjacent code. Do not fix unrelated issues seen in the
│    same file. Do not rename symbols. Do not reformat.
│
├─ 3. Single-test verify
│    Run the single failing test using the exact command supplied.
│    PASS → proceed to step 4.
│    FAIL → back to step 1 with the new output (uses one of the 5 attempts).
│
└─ 4. Full-suite verify
     Run the full suite.
     PASS → report SUCCESS, stop.
     FAIL with test regressions → back to step 1. The regression is your
     signal that the theory was incomplete or the fix was too broad.
```

## Hard prohibitions

These break the contract. If the agent finds itself reaching for any of them,
it must stop and report `FAILED` instead.

| Prohibited | Why |
|---|---|
| Modifying the failing test to make it pass | Defeats the entire exercise |
| `.skip`, `.only`, `.todo`, `xit`, `@pytest.mark.skip`, `t.Skip`, `#[ignore]` | Hides bugs |
| `// @ts-ignore`, `// @ts-expect-error`, `# type: ignore`, `eslint-disable` | Hides real errors |
| `as any`, `: any`, untyped escape hatches | Hides real errors |
| Deleting or weakening another test to clear a regression | Converts one bug into two |
| Catching and swallowing the error the test asserts on | Masks the symptom |
| Rewriting the test to use a mock that the fix relies on | Circular — the mock validates the mock |
| Adding a feature flag | Bug fixes don't need feature flags (CLAUDE.md) |

## Between attempts: what "new information" looks like

If attempt 2's theory is the same as attempt 1's theory, the agent is
guessing, not reasoning. New information comes from:

- Reading a file that wasn't read in the previous attempt
- Running the code path with an added log and reading the actual output
- Comparing the failing path to a similar-but-working path in the codebase
- Re-reading the hypothesis evidence with the specific failure mode in mind

The agent should name, in one sentence, what it learned between attempts.

## Report format

At the end of the loop, the agent outputs exactly one report block:

```
---
FIXER REPORT
Hypothesis: H{n} — {title}
Status: SUCCESS | FAILED
Attempts used: {n}/5

Root cause (1 paragraph):
{explanation tying evidence → bug}

Fix summary (1 paragraph):
{what the minimal change is and why it addresses the root cause}

Files modified (production only — not the test):
- {path}: {one line on what changed}

Verification:
- Single test ({test path}): PASS
- Full suite: PASS ({passing}/{total}) | FAIL ({list of regressed tests})

Attempt trace (one line per attempt):
1. Theory: {sentence}. Result: fail — {one-line reason}.
2. Theory: {sentence}. Result: fail — {one-line reason}.
...
N. Theory: {sentence}. Result: pass.
---
```

If `Status: FAILED`, the `Fix summary` section instead describes the best
theory the agent converged on and why the minimal fix was not achievable
within budget. This goes into `.pipeline/{RUN_NAME}/failed-fixes.md` for the
user to pick up.

## Orchestrator responsibilities (outside the agent)

The orchestrator (the skill itself) is responsible for the steps the fixer
agent does NOT do:

- **Before launching the fixer**: stage the failing test (`git add {test}`)
  so the agent can see it in `git status` but hasn't yet committed it
- **On SUCCESS**: commit test + fix together with the `fix({slug}): ... [H{n}]`
  message, then run the full suite once more before launching the next fixer
- **On FAILED**: unstage and delete the failing test, append the fixer report
  to `failed-fixes.md`, continue to the next bug
- **Never**: let the fixer agent commit on its own — the orchestrator owns the
  commit boundary so each bug is exactly one commit
