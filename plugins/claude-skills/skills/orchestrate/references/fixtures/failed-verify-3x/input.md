# Roadmap: Verify Gate Fails All 3 Retries (Fixture)

One workstream. Its verify gate fails repeatedly and none of the three
retry axes converge. The orchestrator must mark it FAILED, record all
three failure outputs, and exclude it from merge.

## Workstreams

### video-api
- **Scope**: `src/video/**`, `tests/video/**`
- **AC**:
  - AC1: Upload succeeds — check: `npm test -- video/upload`
  - AC2: Transcode runs — check: `npm test -- video/transcode`

## verify.final

```yaml
verify:
  final:
    commands:
      - { name: "test", cmd: "npm test -- --run" }
    smoke: { type: "none" }
```

## Simulated Agent Outcomes

- Attempt 1 (feedback-loop): tests fail with "ReferenceError: Buffer is not
  defined". Agent adds polyfill, re-runs. Tests still fail (lint now also
  complains about the polyfill import).
- Attempt 2 (reduced-scope): agent reverts polyfill, reduces changeset to
  just the transcode endpoint. Tests pass for transcode but upload test
  still fails ("connection refused" — missing service setup).
- Attempt 3 (fresh-agent): new agent inspects, determines the test harness
  requires a docker-compose service that isn't running, cannot start it
  from inside the worktree, fails the gate with that diagnostic.
- All 3 red → workstream FAILED.
