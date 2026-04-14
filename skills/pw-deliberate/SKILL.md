---
name: pw-deliberate
description: |
  Multi-persona plan deliberation. Simulates Product Owner + Tech Lead debate to produce
  a checkpoint-driven plan with explicit success criteria. Use when:
  (1) Planning a feature or significant change
  (2) User says "plan", "deliberate", "think through"
  (3) After research, before implementation
  (4) When multiple approaches are possible and tradeoffs need evaluation
  Produces a plan.md with 3-4 checkpoints, each with goals, files, success criteria,
  and verification commands.
---

# Multi-Persona Plan Deliberation

Simulate a Product Owner + Tech Lead debate to produce a checkpoint-driven implementation plan.

## Input

Accept one of:
1. A path to a `research.md` file (from `pw-research`)
2. A problem statement from the user
3. A GitHub issue number (fetch with `gh issue view {number}`)

## Setup

1. Create or reuse run directory:
   ```bash
   mkdir -p .pipeline/{run-name}
   ```
2. If input is a research.md path, read it for context
3. Read CLAUDE.md for project conventions

## Workflow

### 1. Launch 2 Parallel Perspective Agents

Launch both agents simultaneously using the Agent tool.

**Agent 1 — Product Owner Perspective** (subagent_type: `general-purpose`):
```
You are a Product Owner evaluating this work. Analyze the following problem/research
and provide your perspective.

Problem: {problem statement or research.md content}

Evaluate and output:

### User Value
- What user problem does this solve?
- Who benefits and how?
- What's the minimal viable scope that delivers value?

### Acceptance Criteria
- List 3-5 specific, testable acceptance criteria
- Each must be verifiable (not vague)
- Include mobile considerations (this is a mobile-first web app)

### Scope Risks
- What could cause scope creep?
- What should we explicitly NOT do in this iteration?
- What can be deferred to a follow-up?

### Priority Assessment
- How important is this relative to other work?
- Are there dependencies or blockers?
- What's the impact of NOT doing this?
```

**Agent 2 — Tech Lead Perspective** (subagent_type: `general-purpose`):
```
You are a Tech Lead evaluating this work. Analyze the following problem/research
and provide your technical assessment.

Problem: {problem statement or research.md content}

The project uses: Next.js App Router, TypeScript, Tailwind CSS v4, shadcn/ui, MongoDB/Mongoose,
JWT auth, LaunchDarkly feature flags. See CLAUDE.md for conventions.

Evaluate and output:

### Architecture Impact
- What existing patterns does this follow or extend?
- What files/modules are affected?
- Does this introduce new patterns? If so, justify.

### Technical Approach
- Recommended implementation approach (1-2 paragraphs)
- Key technical decisions and their rationale
- Database changes needed (schema, indexes, migrations)

### Risk Areas
- What could go wrong technically?
- Complex areas that need extra testing
- Performance implications

### Parallelization Opportunities
- What can be implemented concurrently (different files)?
- What must be sequential (shared dependencies)?
- Estimated agent count for implementation

### Feature Flag Needs
- Does this need a feature flag? (new features: yes, bug fixes: no)
- Suggested flag key (format: `faster-scale.{feature-name}`)
- Rollout strategy
```

### 2. Synthesize Deliberation

After both agents complete, synthesize their perspectives:

```markdown
## Deliberation

### Points of Agreement
- {Areas where PO and TL perspectives align}

### Points of Tension
- **{Tension}**: PO says {x}, TL says {y}
  - Resolution: {how we resolve this}

### Resolved Approach
{1-2 paragraph summary of the chosen approach, incorporating both perspectives}
```

### 3. Produce Checkpoint Plan

Create a plan with 3-4 checkpoints. Use the checkpoint format template from `{SKILL_DIR}/references/checkpoint-format.md`.

Each checkpoint should:
- Be completable in one focused session
- Have clear success criteria (not vague)
- Include a verification command that proves it works
- Note which tasks can run in parallel

**Guidelines for checkpoint design:**
- Checkpoint 1: Foundation (models, types, basic structure)
- Checkpoint 2: Core logic (API routes, business logic)
- Checkpoint 3: UI (components, pages, integration)
- Checkpoint 4: Polish (tests, edge cases, feature flags) — optional, combine with 3 if small

### 4. Write Plan

Save to `.pipeline/{run-name}/plan.md`:

```markdown
# Implementation Plan: {Title}

**Date**: {timestamp}
**Input**: {research.md path or problem statement}

## Summary
{2-3 sentence overview}

## Deliberation
{From step 2}

## Acceptance Criteria
{From PO perspective, refined}

## Checkpoints

### Checkpoint 1: {Name}
{Use checkpoint format template}

### Checkpoint 2: {Name}
{Use checkpoint format template}

### Checkpoint 3: {Name}
{Use checkpoint format template}

## Feature Flag
- Key: `{flag-key}`
- Default: off
- Rollout: {strategy}

## Out of Scope
- {Things explicitly deferred}

## Risks & Mitigations
- {Risk}: {Mitigation}
```

### 5. Present to User

Display the plan summary. Ask if the user wants to:
- Proceed to implementation (`pw-implement`)
- Adjust the plan
- Get more detail on a specific checkpoint

## Notes

- The deliberation is the differentiator — don't skip it or reduce it to a single perspective
- Success criteria must be concrete and testable (e.g., "clicking X navigates to Y" not "UX is good")
- Verification commands should be runnable (e.g., `cd app && npx vitest run src/features/X`)
- Always consider mobile-first — this is not optional for this project
