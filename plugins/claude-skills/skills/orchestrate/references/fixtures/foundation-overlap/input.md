# Roadmap: Foundation Overlap (Fixture)

Two same-round workstreams whose scope_globs intersect. The orchestrator
must detect the overlap at Phase 1b (ideally) or Phase 3.5c (at worst) and
halt before a merge stomps parallel work.

## Workstreams

### auth-service
- **Scope**: `src/auth/**`, `src/shared/**`
- **AC**:
  - AC1: Login works — check: `npm test -- auth/login`

### sessions-api
- **Scope**: `src/sessions/**`, `src/shared/**`
- **AC**:
  - AC2: Session store works — check: `npm test -- sessions/store`

Both claim `src/shared/**`. Phase 1b should extract the overlap into a
Round 0 foundation workstream. If not extracted (Phase 1 missed it), Phase
3.5c must catch it when the second branch to merge tries to touch files
the first one already modified.

## verify.final

```yaml
verify:
  final:
    commands:
      - { name: "test", cmd: "npm test -- --run" }
    smoke: { type: "none" }
```

## Simulated Agent Outcomes

- Both agents modify `src/shared/types.ts` in their own ways.
- auth-service merges first, cleanly.
- sessions-api reaches 3.5c — its changed files include `src/shared/types.ts`
  which now contains merged changes from auth-service, flagged as
  foundation-overlap.
