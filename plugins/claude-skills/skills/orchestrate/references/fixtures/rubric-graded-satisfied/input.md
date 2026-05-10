# Roadmap: Rubric-Graded Satisfied (Fixture)

One workstream with a mix of binary criteria (Stage 3 verifier) and
rubric-bearing criteria (Stage 5 grader). Rubric criteria are satisfied
on the first grader pass — exercises the happy path of bias-isolated
grading without hitting a revision loop.

## Workstreams

### auth-service
- **Scope**: `src/auth/**`, `tests/auth/**`
- **Success Criteria**:
  - SC1 (binary): Login returns 200 with token on valid credentials
    - check: `npm test -- auth/login.test.ts -t 'valid credentials'`
  - SC2 (binary + rubric): Login rejection response is well-formed and safe
    - check: `npm test -- auth/login.test.ts -t 'rejection'`
    - rubric: |
        ## Status code
        - Returns HTTP 401 when password is wrong
        - Returns HTTP 401 when account does not exist
        ## Response body
        - `error` field is a non-empty string
        - Does NOT include the decoded token payload in any field
        ## Logging
        - Logs the rejection at info level with the `sub` claim
        - Does NOT log the full token, even at debug level

- **Acceptance Criteria**:
  - AC1: Failed-login error message is helpful and non-leaky (rubric-only)
    - rubric: |
        ## Tone
        - Plain English, not a stack trace
        - Single sentence, ≤120 characters
        ## Information disclosure
        - Does NOT name which field was wrong (username vs. password)
        - Does NOT reveal whether the account exists

## verify.final

```yaml
verify:
  final:
    commands:
      - { name: "typecheck", cmd: "npm run typecheck" }
      - { name: "lint",      cmd: "npm run lint" }
      - { name: "test",      cmd: "npm test -- --run" }
    smoke:
      type: "none"
```

## Simulated Outcomes (for the self-test)

- SC1: implemented cleanly, Stage 3 GREEN, no rubric → DONE at Stage 3.
  `grader_iterations: 0`, `per_aspect_results: null`.
- SC2: implemented cleanly, Stage 3 GREEN, rubric present → Stage 5 runs,
  grader returns `satisfied` first pass.
  `grader_iterations: 1`, `per_aspect_results` contains all 3 aspects
  (Status code / Response body / Logging) with every bullet `satisfied`.
- AC1: rubric-only acceptance criterion. PO grader runs Stage-5-style
  grading post-merge (no `check` to run via verify.final). Returns
  `satisfied` first pass. `grader_iterations: 1`, `per_aspect_results`
  populated with Tone + Information disclosure aspects.
- Final verify: all 3 commands exit 0.
- PO verdict: APPROVED. No showstoppers, no fast-follows.
