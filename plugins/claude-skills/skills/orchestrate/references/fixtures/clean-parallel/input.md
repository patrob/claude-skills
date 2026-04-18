# Roadmap: Clean Parallel (Fixture)

Three independent workstreams. All implement cleanly, verify on attempt 1,
no scope deviations, all ACs green.

## Workstreams

### auth-service
- **Scope**: `src/auth/**`, `tests/auth/**`
- **AC**:
  - AC1: Login returns 200 with token — check: `npm test -- auth/login`
  - AC2: Bad password returns 401 — check: `npm test -- auth/bad-password`

### video-api
- **Scope**: `src/video/**`, `tests/video/**`
- **AC**:
  - AC3: Upload succeeds — check: `npm test -- video/upload`
  - AC4: Transcode runs — check: `npm test -- video/transcode`

### clip-editor
- **Scope**: `src/components/clip-editor/**`, `tests/ui/clip-editor/**`
- **AC**:
  - AC5: Clip save persists — check: `npm test -- ui/clip-editor/save`

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

## Simulated Agent Outcomes (for the self-test)

- All 3 agents commit cleanly, scope-diff matches their globs, verify passes
  on attempt 1, review APPROVED first pass.
- Final verify: all 3 commands exit 0.
