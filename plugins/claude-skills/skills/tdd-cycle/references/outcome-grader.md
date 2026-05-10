# Outcome Grader (Stage 5)

Bias-isolated per-criterion grading. Models the Managed Agents "Outcomes"
pattern in a Claude Code skill: a fresh subagent reads the artifact and the
rubric, returns per-aspect verdicts, and the orchestrator iterates until
satisfied or capped.

## Why a separate stage?

Stage 3 (Verifier) answers "do the tests pass?" Stage 5 answers "is the
criterion actually met?" Those are different questions, and they fail in
different ways:

| Failure mode | Caught by Stage 3? | Caught by Stage 5? |
|---|---|---|
| Test fails (functionality missing) | yes | n/a (won't reach 5) |
| Test passes but doesn't exercise the criterion | no | yes |
| Test passes but the criterion has multiple sub-aspects, only one is covered | no | yes (per-aspect) |
| Criterion is qualitative — no test possible | no | yes (rubric-only) |
| Implementation leaks information the rubric forbids (absence property) | rarely | yes |

Stage 3 is a tight loop on the agent's own tests. Stage 5 is the
independent reviewer that asks "did you actually do what the criterion
asked, in all the ways it asked?"

## When to add a rubric

See `extract-criteria/SKILL.md#when-to-add-a-rubric` for the canonical
guidance. Short version: rubric required when the criterion has multiple
sub-aspects, qualitative judgment, information-disclosure properties, or
cross-cutting concerns. Rubric optional (skip Stage 5) for clean binary
behaviors.

## Rubric format

Markdown. Top-level structure:

```markdown
## {Aspect name}
- {Single, testable assertion}
- {Single, testable assertion}

## {Another aspect}
- {Single, testable assertion}
```

Rules the decomposer (and humans) must follow:

- **One assertion per bullet.** "Returns 401 AND logs the rejection" should
  be two bullets, not one. The grader scores per-bullet, so combined
  bullets degrade to coarse verdicts.
- **No vague bullets.** "The response looks good" is not gradeable. Prefer
  "The response body has an `error` field that is a non-empty string."
- **Absence properties are first-class.** "Does NOT log the full token" is
  exactly the kind of bullet a unit test rarely covers but a grader reading
  the diff catches naturally.
- **3-7 bullets per aspect, 1-5 aspects per criterion.** Beyond that the
  criterion is too broad — split it.

## Worked example — binary criterion with rubric

Criterion: `SC3 — Login endpoint returns 401 on expired tokens`

```markdown
## Status code
- Returns HTTP 401 when token `exp` claim is in the past
- Returns HTTP 401 when token `exp` claim is missing entirely

## Response body
- `error` field is present and is a non-empty string
- Does NOT include the decoded token payload in any field

## Logging
- Logs the rejection at info level with the token's `sub` claim
- Does NOT log the full token (any field), even at debug level
```

A single passing test for "expired token returns 401" satisfies one bullet
of the Status code aspect. The grader catches that the missing-`exp` case
is unhandled, that the response body shape isn't asserted, and that the
log-content properties aren't checked anywhere. It returns
`needs_revision` with concrete `gap` strings the fix-loop can act on.

## Worked example — rubric-only criterion (no `check`)

Criterion: `AC2 — Failed-login error message is helpful and non-leaky`

```markdown
## Tone
- Plain English, not a stack trace or framework error
- Single sentence, ≤120 characters

## Information disclosure
- Does NOT name which field was wrong (username vs. password)
- Does NOT reveal whether the account exists
- Does NOT include the request id in user-visible text (it goes in the log)

## Localization
- Sourced from the i18n catalog, not a string literal
```

No CLI exits 0 for any of these. The grader is the verification, full
stop. Stage 3 still runs (typecheck/lint/full suite) so the change doesn't
break anything else, but the criterion's own pass/fail comes from Stage 5.

## Iteration semantics

- Cap: 3 grader iterations per criterion, separate from the fix-loop's 3.
- Combined worst case: 6 revisions (3 fix-loop reds + 3 grader
  needs_revision). Most cycles terminate at 1 grader iteration or fewer.
- Each grader iteration:
  1. Stage 5 spawns a fresh `criterion-grader` (fresh context every time).
  2. If `needs_revision`, route into Stage 4 (fix-loop) with the per-aspect
     `gap` strings as the work to do — NOT a failing test.
  3. After the fix-loop returns (Stage 3 GREEN again), re-spawn Stage 5.
- The grader does NOT see prior iterations' verdicts, by design. Each call
  is independent. The orchestrator accumulates the history; the grader
  evaluates the current artifact.

## Failed grader contradiction

If the grader returns `result: failed`, it means the rubric and the
criterion `text` describe different behaviors (e.g. text says "returns
401" but the rubric demands HTTP 403). This is terminal — no iteration
can fix it because the work itself is ambiguous. Surface the
`explanation` to the human and stop.

This matches the Managed Agents Outcomes `failed` result: "the rubric
fundamentally does not match the task."

## Cost considerations

Each grader spawn is a fresh subagent — no prompt caching across
iterations. For a criterion with a large diff and a long rubric, expect
the grader to read several KB of source. Bound the cost by:

- Keeping rubrics under ~20 bullets total.
- Using `scope_globs` aggressively (the grader is told to ignore
  out-of-scope files).
- Ending the cycle on `satisfied` — do NOT run a "second opinion" grader.

## Bias isolation — what makes it work

The grader's tools are restricted at the agent definition level (Read,
Glob, Grep, Bash for read-only git only). It cannot run tests (so it can't
"validate" passing tests as proof). It cannot read the implementer's
report (so it can't be charmed by the implementer's reasoning). It comes
in cold to a diff and a rubric and grades.

The orchestrator MUST NOT pass implementer or test-writer reports into the
grader's prompt. If it does, bias isolation collapses and the grader
becomes a rubber stamp. The grader's prompt template in
`tdd-cycle/SKILL.md#stage-5--outcome-grader` is the only allowed shape.
