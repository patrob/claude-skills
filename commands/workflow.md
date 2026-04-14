# Personal Workflow Orchestrator

Chain research, planning, implementation, and review skills into a complete pipeline.

## Invocation

`/workflow <problem description>`
`/workflow resume` — resume from last incomplete phase

## Triage

Analyze the problem description to determine scope:

**Bug keywords** (fix, bug, broken, error, crash, not working, regression, failing):
→ Run: `pw-bug-hunt`

**Small change** (tweak, typo, rename, update text, < 20 lines estimated):
→ Run: `pw-implement` → `verify-fix-loop` → `pw-review`

**Research-only** (investigate, explore, research, compare, evaluate, understand):
→ Run: `pw-research`

**Feature / large change** (add, implement, build, create, feature, refactor, redesign):
→ Run: `pw-research` → `pw-deliberate` → `pw-implement` → `verify-fix-loop` → `pw-review`

## State Management

Create a run directory for the pipeline:
```bash
RUN_NAME=$(echo "$ARGUMENTS" | tr ' ' '-' | tr '[:upper:]' '[:lower:]' | cut -c1-40)-$(date +%s)
mkdir -p .pipeline/$RUN_NAME
```

Write `.pipeline/{run-name}/state.json` tracking pipeline progress:
```json
{
  "run_name": "{run-name}",
  "problem": "{problem description}",
  "triage": "{bug|small|research|feature}",
  "phases": {
    "research": "pending|running|complete|skipped",
    "deliberate": "pending|running|complete|skipped",
    "implement": "pending|running|complete|skipped",
    "verify": "pending|running|complete|skipped",
    "review": "pending|running|complete|skipped"
  },
  "current_phase": "{phase name}",
  "started_at": "{ISO timestamp}",
  "artifacts": {
    "research": ".pipeline/{run-name}/research.md",
    "plan": ".pipeline/{run-name}/plan.md",
    "implementation_log": ".pipeline/{run-name}/implementation-log.md",
    "review": ".pipeline/{run-name}/review-summary.md"
  }
}
```

## Pipeline Execution

### Phase Transitions

Each phase produces output that feeds the next:

1. **pw-research** → produces `research.md`
2. **pw-deliberate** → reads `research.md`, produces `plan.md`
3. **pw-implement** → reads `plan.md`, produces code changes + `implementation-log.md`
4. **verify-fix-loop** → runs `make verify`, fixes until passing
5. **pw-review** → reviews all changes, produces `review-summary.md`

After each phase completes:
1. Update `state.json` with phase status
2. Display phase summary to user
3. Ask: "Continue to next phase?" (unless running in auto mode)

### Auto Mode

If the user says `/workflow auto <problem>`, run all phases without pausing between them. Only stop if a phase fails.

### Resume

When `/workflow resume` is invoked:
1. Find the most recent `.pipeline/*/state.json`
2. Read the state to determine the last completed phase
3. Resume from the next incomplete phase
4. Display: "Resuming pipeline '{run-name}' from {phase} phase"

If multiple pipelines exist, show the user a list and ask which to resume.

## Pipeline Report

After the final phase, display a summary:

```markdown
## Pipeline Complete: {problem description}

**Triage**: {bug|small|research|feature}
**Phases completed**: {list}
**Duration**: {total time}

### Phase Results
- Research: {summary or "skipped"}
- Plan: {summary or "skipped"}
- Implementation: {files modified, tests passing}
- Verification: {PASS/FAIL}
- Review: {verdict — PASS/PASS WITH NOTES/NEEDS CHANGES}

### Next Steps
- {Suggested actions: commit, PR, fix review findings, etc.}
```

## Error Handling

If any phase fails:
1. Update state.json with failure status
2. Display what failed and why
3. Ask user how to proceed:
   - Retry the phase
   - Skip to next phase
   - Abort the pipeline
   - Fix manually and resume later (`/workflow resume`)

## Notes

- State files in `.pipeline/` persist across sessions for resumability
- Each skill invocation passes the run directory path for artifact coordination
- The triage step should be quick — don't over-analyze, just categorize
- For ambiguous cases, ask the user which pipeline path to take
- Bug fixes always skip research and deliberation phases (CLAUDE.md: bugs don't need feature flags)
