# Phase 1 — Decompose Into Workstreams

Load this reference when executing Phase 1 of `/orchestrate`.

Parse the input and extract discrete workstreams. A workstream is a unit of
work that can be independently researched, planned, implemented, verified, and
reviewed.

## 1a. Extract Workstreams

From the input, identify each workstream with:
- **Name**: short identifier (e.g., `auth-service`, `video-upload-api`)
- **Description**: full scope of what needs to be built
- **Domain**: which area of the codebase it touches (backend, frontend, infra)
- **Files likely affected**: best estimate of files created/modified (globs preferred)
- **Scope globs**: machine-readable form of "files likely affected" used by the
  Phase 3.5c scope-diff check. Every workstream MUST have a `scope_globs` entry
  — this is the contract the subagent is held to.
- **Dependencies**: does this workstream need another to complete first?
- **Acceptance criteria**: how do we know this workstream is done? Bullet list,
  verifiable form (each AC must map to a runnable check).

Record each workstream into `state.json` under `rounds[n].workstreams[w]`:

```json
{
  "name": "auth-service",
  "description": "OAuth2 login + JWT session",
  "scope_globs": ["src/auth/**", "tests/auth/**", "migrations/*auth*"],
  "dependencies": [],
  "acceptance_criteria": [
    { "id": "AC1", "text": "Login returns 200 + token", "check": "curl localhost:3000/login -d ... | jq .token" },
    { "id": "AC2", "text": "Session expires after 24h", "check": "npm test -- --grep 'session expiry'" }
  ]
}
```

## 1b. Dependency & Overlap Analysis

For each pair of workstreams:
- **File overlap check**: do their `scope_globs` intersect?
- **Interface dependency**: does one produce an API/type/schema the other consumes?
- **Shared infrastructure**: do they both need a shared piece (e.g., DB migration)?

Categorize relationships:
- **Independent**: no overlap, no dependency → parallel
- **Consumer/Producer**: one needs the other's output → sequential
- **Shared foundation**: both need a common base → extract foundation as its
  own workstream, run first

### Contract Extraction for Consumer/Producer Pairs

When a Consumer/Producer relationship is identified (service-to-service, job
queues, shared data shapes, backend-to-frontend flow):

1. Define the shared interface contract inline in the orchestration plan:
   - Data shape (in the project's native type system or as a schema)
   - Required fields vs optional
   - Error/status enumerations
2. Include the contract in BOTH agents' prompts (see `phase-2-spawn.md`)
3. The producer workstream MUST implement the contract as defined
4. The consumer workstream MUST consume the contract as defined

See `interface-contracts.md` for contract format and examples.

## 1c. Build Round Schedule

Group workstreams into rounds:

```
Round 0 (foundation — sequential if needed):
  - Shared DB migrations, shared types, shared config

Round 1 (parallel — independent workstreams):
  - [worktree] auth-service: full /workflow pipeline
  - [worktree] video-upload-api: full /workflow pipeline
  - [worktree] ui-components: full /workflow pipeline

Round 2 (parallel — depends on Round 1):
  - [worktree] api-integration: full /workflow pipeline
  - [worktree] upload-ui: full /workflow pipeline

Round 3 (integration):
  - Integration tests, E2E tests, cross-cutting concerns
```

## 1d. Present Orchestration Plan

Display to the user before proceeding (unless in auto mode):

```
Orchestration Plan: {input title}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Round 0 — Foundation (sequential):
  1. shared-types: Database schema + shared TypeScript types
     Scope: src/types/**, migrations/**
     Pipeline: implement → verify

Round 1 — Core Features (3 parallel agents, full /workflow):
  1. [worktree: auth-service] Authentication & authorization
     Scope: src/auth/**, tests/auth/**
     AC: 3 criteria
  2. [worktree: video-api] Video upload & processing API
     Scope: src/video/**, tests/video/**
     AC: 5 criteria
  3. [worktree: clip-editor] Clip editor UI components
     Scope: src/components/clip-editor/**, tests/ui/clip-editor/**
     AC: 4 criteria

Round 2 — Integration (2 parallel agents):
  1. [worktree: api-wiring] Wire frontend to APIs
  2. [worktree: auth-ui] Auth UI pages

Round 3 — Cross-Cutting (sequential):
  1. Integration tests + E2E tests

Estimated parallel agents: 3 max concurrent
Proceed? (y/n)
```

If `--dry-run` is set, render the plan using `dry-run.md` and STOP before
launching. If `auto` mode is set, skip the confirmation prompt.

## Acceptance-Criteria → Workstream Map

Before finishing Phase 1, build a reverse index mapping each acceptance
criterion from the original roadmap to the workstream responsible for it:

```json
"ac_map": {
  "AC1-user-can-log-in": "auth-service",
  "AC2-uploads-transcode": "video-upload-api",
  "AC3-clip-edit-saves": "clip-editor"
}
```

This is the foundation of the Phase 5 acceptance report. Every AC in the
roadmap MUST appear in this map — if one has no owning workstream, the plan
is incomplete and Phase 1 fails.
