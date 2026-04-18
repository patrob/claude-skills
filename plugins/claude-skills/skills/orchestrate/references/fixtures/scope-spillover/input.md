# Roadmap: Scope Spillover (Fixture)

One workstream. Its agent edits `docs/unrelated.md` during implementation
without declaring it in the workstream report's "Scope deviations" section.
The orchestrator must flag it as creep.

## Workstreams

### auth-service
- **Scope**: `src/auth/**`, `tests/auth/**`
- **AC**:
  - AC1: Login returns 200 — check: `npm test -- auth/login`

## verify.final

```yaml
verify:
  final:
    commands:
      - { name: "test", cmd: "npm test -- --run" }
    smoke: { type: "none" }
```

## Simulated Agent Outcomes

- auth-service: implementation touches `src/auth/login.ts` (in scope),
  `tests/auth/login.test.ts` (in scope), AND `docs/unrelated.md` (out of
  scope, not declared). Agent commits everything. Verify passes attempt 1.
- At Phase 3.5c, the scope-diff check identifies `docs/unrelated.md` as
  out-of-scope-not-declared → classification: creep.
