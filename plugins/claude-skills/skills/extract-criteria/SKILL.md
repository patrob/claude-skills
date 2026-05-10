---
name: extract-criteria
description: |
  Decompose a spec, roadmap, PRD, or plan into workstreams with scope_globs,
  success criteria, and acceptance criteria. Use when:
  (1) You have a feature spec or roadmap and need to break it into workstreams
  (2) User says "decompose this", "extract criteria", "break down the spec"
  (3) Before parallel implementation, to establish what "done" means
  (4) `/orchestrate` Phase 1
  Always delegates the parsing to a single subagent — the caller never parses
  the spec itself. Output is structured JSON consumable by downstream skills.
---

# Extract Criteria

Delegate spec parsing to one subagent. Return `{workstreams[], ac_map,
per_workstream_criteria[]}` as JSON.

## Input

Accept one of:
1. A path to a roadmap / feature spec / PRD (`.md` file)
2. A `plan.md` from `pw-deliberate` or `/plan`
3. An inline description of multiple workstreams
4. Auto-detect: most recent `.pipeline/*/plan.md`

The caller does NOT read the spec. Pass its path (or the inline text) to the
subagent below.

## Delegation

Spawn ONE subagent. Do not parse the spec yourself.

```
Agent({
  description: "Decompose spec into workstreams + criteria",
  subagent_type: "general-purpose",
  prompt: <see prompt template>
})
```

### Prompt template

```
You are a spec-decomposition agent. Read the input and return a single JSON
document describing every workstream, its scope, and its criteria.

## Input
{spec path OR inline text}

## Your job

1. Read the input end-to-end.
2. Extract every discrete workstream — a unit that can be researched,
   implemented, verified, and reviewed independently.
3. For each workstream produce:
   - name: short identifier (kebab-case, e.g. "auth-service")
   - description: one paragraph of scope
   - scope_globs: file globs the workstream is allowed to touch
     (e.g. ["src/auth/**", "tests/auth/**", "migrations/*auth*"])
   - dependencies: names of workstreams that must merge first
   - success_criteria: 3-8 verifiable statements about what the code must do.
     Each statement must carry a `check` command, a `rubric`, or BOTH:
       - `check` — a single shell command that exits 0 iff the criterion
         holds. Required for any criterion whose verification is naturally
         binary (status code, return value, file exists, lint clean).
       - `rubric` — a markdown document with `## Aspect` sections and
         per-aspect bullets. Required for any criterion whose verification
         is qualitative or has multiple sub-aspects (response shape +
         logging behavior + error message quality, "report renders
         correctly," "error message is helpful"). The rubric is graded by a
         bias-isolated `criterion-grader` subagent at Stage 5 of
         `tdd-cycle`.
       - BOTH is allowed and recommended for criteria that have a clear
         binary check but where the test could pass for the wrong reason —
         the rubric becomes a sanity check that the test actually exercises
         the criterion.
   - acceptance_criteria: 2-5 user-facing / behavioral statements the PO will
     walk through end-to-end. Must NOT contradict success_criteria but may
     be broader (end-to-end journeys, UX checks). If the spec has explicit
     ACs, use them verbatim; otherwise derive from the feature description.
     Acceptance criteria may also carry a `rubric` (no `check` required).
4. Identify Consumer/Producer pairs. For each pair, define a contract:
   producer workstream name, consumer workstream name, data shape.
5. Build an ac_map: every AC id from the original spec → owning workstream.
   Every AC in the spec MUST appear exactly once. If any is unmapped, the
   decomposition is wrong — fix it before returning.
6. Group workstreams into rounds:
   - Round 0 (foundation): shared types, migrations, config
   - Round N (parallel): no dependencies within the round
   - Later rounds consume outputs of earlier rounds

## Output shape

Return EXACTLY this JSON (no prose, no markdown fences):

{
  "title": "{short title from input}",
  "rounds": [
    {
      "round": 0,
      "label": "foundation",
      "workstreams": [
        {
          "name": "shared-types",
          "description": "...",
          "scope_globs": ["src/types/**"],
          "dependencies": [],
          "success_criteria": [
            { "id": "SC1", "text": "...", "check": "npm run typecheck" },
            {
              "id": "SC2",
              "text": "Login rejection response is well-formed and safe",
              "check": "npx vitest run auth/login.test.ts",
              "rubric": "## Status code\n- Returns HTTP 401 on expired token\n## Response body\n- `error` is a non-empty string\n- Does NOT leak the decoded token payload\n## Logging\n- Logs at info with the `sub` claim only"
            }
          ],
          "acceptance_criteria": [
            { "id": "AC1", "text": "..." },
            {
              "id": "AC2",
              "text": "Failed-login error message is helpful and non-leaky",
              "rubric": "## Tone\n- Plain English, no stack trace\n## Information disclosure\n- Does not name which field was wrong (username vs password)\n- Does not reveal whether the account exists"
            }
          ]
        }
      ]
    }
  ],
  "contracts": [
    {
      "name": "auth-token-shape",
      "producer": "auth-service",
      "consumer": "ui-header",
      "shape": "{ token: string; expiresAt: ISODateTime }"
    }
  ],
  "ac_map": {
    "AC1": "shared-types",
    "AC2": "auth-service"
  },
  "unmapped_acs": []
}

If `unmapped_acs` is non-empty, your output is INCOMPLETE. Re-read the spec
and fix the decomposition before returning.
```

## After the subagent returns

1. Write the JSON to `.pipeline/$RUN_NAME/criteria.json` via a second
   "writer" subagent if the caller's delegation contract forbids direct
   writes, OR (when invoked standalone outside `/orchestrate`) write it
   directly with `Write`.
2. Verify `unmapped_acs` is `[]`. If not, re-spawn the decomposer with the
   unmapped list appended to its prompt. Max 2 retries.
3. Return the JSON to the caller.

## Standalone use

`/extract-criteria path/to/spec.md` reads the spec, writes
`.pipeline/criteria-{timestamp}/criteria.json`, and prints the round
schedule so the user can review before any agents are launched.

## Quality bar

- Every success criterion has at least one of: a runnable `check` command,
  a `rubric` (markdown with `## Aspect` sections), or both. A criterion with
  neither is rejected — the verifier and the grader would have nothing to
  evaluate.
- Every acceptance criterion is either covered by ≥1 success criterion (via
  `ac_map`) OR carries its own `rubric` so the PO grader can score it
  directly.
- Rubrics are written as markdown with `## Aspect` sections and `-` bullets
  per gradeable item. Each bullet is a single, testable assertion. Avoid
  vague bullets like "the response is good" — the grader scores per bullet,
  so vague bullets produce noisy verdicts.
- Scope globs are specific (`src/auth/**`, not `**`).
- No two workstreams in the same round have intersecting scope globs
  (foundation overlap must be extracted as its own Round 0 workstream).

## When to add a rubric (decomposer guidance)

Add a rubric when the criterion involves any of:

- **Multiple sub-aspects** — "returns 401 AND logs the rejection AND does
  not leak the token." A `check` collapses these into one pass/fail; a
  rubric scores each independently and tells you exactly which one failed.
- **Qualitative judgment** — "error message is helpful," "report layout is
  scannable," "API shape is intuitive." No CLI exits 0 for these.
- **Information-disclosure / safety properties** — "does NOT log the full
  token," "does NOT reveal which field was wrong." These are absence
  properties that tests rarely cover but that a grader reading the diff
  catches naturally.
- **Cross-cutting concerns** — "all endpoints emit a request_id header."
  The rubric forces the grader to enumerate, not just sample.

Skip the rubric (use `check` only) when the criterion is a single binary
behavior with a clean test (a function returns the right value, a migration
adds the right column, a feature flag toggles correctly).
