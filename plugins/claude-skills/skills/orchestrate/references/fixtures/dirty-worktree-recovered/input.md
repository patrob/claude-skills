# Roadmap: Dirty Worktree Recovered (Fixture)

Two workstreams. One (clip-editor) arrives at the merge-safety checklist
with uncommitted changes (agent's Phase 3.5 Commit Gate failed or agent
was interrupted). The orchestrator must:
1. Commit the leftover work with `[recovery]` prefix
2. Mark clip-editor quarantined
3. Skip its merge
4. Continue merging auth-service
5. Surface clip-editor in Next Actions

## Workstreams

### auth-service
- **Scope**: `src/auth/**`, `tests/auth/**`
- **AC**:
  - AC1: Login returns 200 — check: `npm test -- auth/login`

### clip-editor
- **Scope**: `src/components/clip-editor/**`
- **AC**:
  - AC2: Clip save persists — check: `npm test -- ui/clip-editor/save`

## verify.final

```yaml
verify:
  final:
    commands:
      - { name: "test", cmd: "npm test -- --run" }
    smoke: { type: "none" }
```

## Simulated Agent Outcomes

- auth-service: clean commits, verify passes attempt 1, merges cleanly.
- clip-editor: makes 2 commits, leaves `src/components/clip-editor/save.ts`
  modified and staged but uncommitted. Reports COMPLETE anyway (Commit Gate
  bug — the exact failure mode this fixture exists to catch).
