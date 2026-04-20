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
   - success_criteria: 3-8 verifiable statements about what the code must do
     (each statement must be testable with a single check command)
   - acceptance_criteria: 2-5 user-facing / behavioral statements the PO will
     walk through end-to-end. Must NOT contradict success_criteria but may
     be broader (end-to-end journeys, UX checks). If the spec has explicit
     ACs, use them verbatim; otherwise derive from the feature description.
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
            { "id": "SC1", "text": "...", "check": "npm run typecheck" }
          ],
          "acceptance_criteria": [
            { "id": "AC1", "text": "..." }
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

- Every success criterion has a runnable `check` command.
- Every acceptance criterion maps to ≥1 success criterion (check via
  `ac_map`).
- Scope globs are specific (`src/auth/**`, not `**`).
- No two workstreams in the same round have intersecting scope globs
  (foundation overlap must be extracted as its own Round 0 workstream).
