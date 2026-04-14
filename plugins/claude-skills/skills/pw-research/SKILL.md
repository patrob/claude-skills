---
name: pw-research
description: |
  Deep parallel research with validation. Launch 3 parallel agents to understand a problem
  from multiple angles, then validate findings against the codebase. Use when:
  (1) Starting a new feature or significant change
  (2) User says "research", "investigate", "explore options"
  (3) Before planning, to understand the problem space
  (4) Evaluating a technology choice or approach
  Produces confidence-rated findings in research.md.
---

# Deep Parallel Research

Launch 3 parallel agents to understand a problem from multiple angles, then validate findings.

## Input

Accept one of:
1. A problem statement from the user
2. A GitHub issue number (fetch with `gh issue view {number}`)
3. A general topic to research

## Setup

1. Create run directory:
   ```bash
   mkdir -p .pipeline/{run-name}
   ```
2. Read CLAUDE.md for project context

## Workflow

### 1. Launch 3 Parallel Research Agents

Launch all 3 agents simultaneously using the Agent tool.

**Agent 1 — Web Research** (subagent_type: `rpi:web-research-specialist`):
```
Research the following problem from a web/industry perspective:

Problem: {problem statement}

Find and report on:
1. Best practices and common patterns for solving this
2. Relevant libraries or tools (with version compatibility for Node 18+, Next.js App Router)
3. Documentation and examples from official sources
4. Known pitfalls or gotchas from community experience
5. Similar implementations in open-source projects

Output format:
### Web Research Findings
- **{Finding 1}**: {description with source}
- **{Finding 2}**: {description with source}
...
### Recommended Approach: {summary}
### Sources: {list of URLs}
```

**Agent 2 — Codebase Analysis** (subagent_type: `Explore`):
```
Analyze the existing codebase to understand how it relates to this problem:

Problem: {problem statement}

Investigate and report on:
1. Affected files and modules (search broadly, don't assume)
2. Existing patterns that solve similar problems in this codebase
3. Architecture constraints or conventions to respect
4. Current implementations that would be impacted
5. Test patterns used for similar features
6. Database models and collections involved

Search in: app/src/ (main source), app/e2e/ (E2E tests)

Output format:
### Codebase Analysis
- **Affected files**: {list with paths}
- **Existing patterns**: {how similar things are done}
- **Architecture notes**: {constraints, conventions}
- **Database impact**: {models, collections, schema changes}
- **Test patterns**: {how similar features are tested}
```

**Agent 3 — Project Context** (subagent_type: `Explore`):
```
Gather project-specific context relevant to this problem:

Problem: {problem statement}

Investigate:
1. CLAUDE.md conventions that apply (read the full file)
2. Recent git history related to this area:
   ```bash
   git log --oneline -20 -- {relevant paths}
   ```
3. Feature flags in use (LaunchDarkly, project: faster-scale)
4. Database gotchas from CLAUDE.md (legacy collections, email arrays, collation, etc.)
5. Any existing TODO comments or known issues in related code
6. Environment variables relevant to this feature

Output format:
### Project Context
- **Applicable conventions**: {from CLAUDE.md}
- **Recent changes**: {relevant git history}
- **Feature flags**: {existing flags, need for new ones}
- **Database gotchas**: {relevant warnings}
- **Known issues**: {TODOs, comments, open issues}
- **Environment**: {relevant env vars}
```

### 2. Synthesize Findings

After all 3 agents complete, combine their findings into a unified research document.

Organize by topic rather than by agent:
- Group related findings together
- Note where agents agree or disagree
- Identify gaps (things none of the agents found)

### 3. Launch Validation Agent

Launch a single validation agent (subagent_type: `Explore`):

```
Validate these research findings against the actual codebase. For each finding,
verify it's accurate and assign a confidence rating.

Findings to validate:
{synthesized findings from step 2}

For each finding, check:
1. Do referenced files actually exist? Read them.
2. Do described patterns actually match the code?
3. Are version numbers and compatibility claims accurate?
4. Are the described conventions actually followed in the codebase?

Rate each finding:
- HIGH: Verified against actual code/docs, definitely accurate
- MEDIUM: Partially verified, likely accurate but some assumptions
- LOW: Unverified or contradicted by actual code

Output format:
### Validation Results
| Finding | Confidence | Notes |
|---------|-----------|-------|
| {finding} | HIGH/MEDIUM/LOW | {verification details} |
```

### 4. Generate Research Document

Write `.pipeline/{run-name}/research.md`:

```markdown
# Research: {Topic}

**Date**: {timestamp}
**Problem**: {problem statement}

## Executive Summary
{3-5 sentence summary of findings and recommended direction}

## Findings

### {Topic Area 1}
- **[HIGH]** {Finding}: {details}
- **[MEDIUM]** {Finding}: {details}

### {Topic Area 2}
- **[HIGH]** {Finding}: {details}
- **[LOW]** {Finding}: {details, why low confidence}

## Codebase Impact
- **Files affected**: {list}
- **Patterns to follow**: {list}
- **Database changes**: {if any}

## Recommended Approach
{1-2 paragraphs based on highest-confidence findings}

## Open Questions
- {Things that still need clarification}

## Sources
- {Web sources with URLs}
- {Codebase files referenced}
```

### 5. Present to User

Display the executive summary and key findings. Ask if the user wants to:
- Proceed to planning (`pw-deliberate`)
- Dig deeper into a specific area
- Adjust the research direction

## Notes

- The validation pass is critical — it catches stale information and incorrect assumptions
- Web research agents should focus on current (2025+) sources for library/framework advice
- Codebase analysis should search broadly, not just in expected directories
- Project context agent should always read CLAUDE.md fully, not just relevant sections
- Total run time: typically 60-90 seconds for 3 parallel agents + validation
