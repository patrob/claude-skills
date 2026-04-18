# Changelog

All notable changes to the `claude-skills` plugin. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this plugin
adheres to semantic versioning.

## [1.3.0] — 2026-04-17

Autonomous-workflow upgrade for `/orchestrate`. Outcome of a Tech Lead /
Product Owner deliberation that reconciled the "stay autonomous" goal with
the v1.2 "never silently merge garbage" safety rule.

### Added — `/orchestrate`

- **Verify gate in every workstream agent**. Tests + lint + typecheck must
  all pass before the agent can claim COMPLETE. Capped at 3 retries, each
  using a distinct axis:
  1. Feedback loop (same agent, failure output in context)
  2. Reduced scope (same agent, revert unrelated changes, minimal slice)
  3. Fresh agent (new subagent, clean context, explicit "choose differently")
  Spec: `skills/orchestrate/references/retry-semantics.md`.

- **Recovery-quarantine path for dirty worktrees**. A worktree arriving at
  merge time with uncommitted changes is now auto-committed with a
  `[recovery]` prefix, marked `quarantined: true`, and **excluded from the
  merge**. The recovery diff is preserved at
  `.pipeline/{RUN_NAME}/recovery/{workstream}.diff` and surfaced in the
  acceptance report's "Next Actions for Human" section. This replaces v1.2's
  hard-stop behavior while preserving its safety guarantee — recovered
  work is never silently laundered into the feature branch.
  Spec: `skills/orchestrate/references/phase-3.5-merge-safety.md § 3.5b`.

- **`verify.final` hook** — configurable final-verification contract in
  roadmap YAML frontmatter. Supports multiple `commands` plus a `smoke` of
  type `playwright` / `http` / `cli` / `none`. Auto-detection defaults per
  project type (npm, pyproject, cargo, go, plugin). Playwright is one option,
  never hardcoded. Spec: `skills/orchestrate/references/phase-4-verify-final.md`.

- **Acceptance report** at `.pipeline/{RUN_NAME}/acceptance-report.md` plus
  JSON twin. Opens with "Next Actions for Human" — quarantined branches,
  blocked reviews, scope creep, red verify steps — each with the exact next
  step the user must take. One row per original AC with PASS/FAIL/MANUAL +
  validating command + output excerpt + commit SHA. This is the proof-of-
  acceptance deliverable.
  Spec: `skills/orchestrate/references/phase-5-report.md`.

- **`--dry-run` mode**. Renders the full orchestration plan (workstream
  decomposition, scope globs, dependency order, AC→workstream map,
  `verify.final` configuration, conflicts with current working tree) and
  exits without launching agents or mutating git state.
  Spec: `skills/orchestrate/references/dry-run.md`.

- **Scope globs as a declared contract**. Every workstream must declare
  `scope_globs` in Phase 1. Phase 3.5c diffs actual changed files against
  them and classifies out-of-scope files as declared-spillover, creep, or
  foundation-overlap.

- **Self-test fixtures** at `skills/orchestrate/references/fixtures/`:
  five named scenarios (`clean-parallel`, `dirty-worktree-recovered`,
  `scope-spillover`, `failed-verify-3x`, `foundation-overlap`) with
  `input.md` + `expected.json`. A meta-agent prompt in `self-test.md` walks
  the skill's rules against each fixture and asserts outcomes. These serve
  as both regression tripwires and living documentation of reconciliation
  behavior. Not unit tests — the skill is markdown prompts, not executable
  code — but serve the same role for pre-release review.

### Changed — `/orchestrate`

- **Progressive disclosure**. `SKILL.md` reduced from ~750 lines to ~200.
  Phase detail moved to `references/phase-{1..5,3.5}-*.md`, loaded on demand
  per phase. The top-level SKILL.md now reads as an index of non-negotiable
  safety rules and the phase sequence. Reduces context load when only a
  single phase is executing.

- **Retry semantics formalized**. v1.2 mentioned "3 attempts" loosely; v1.3
  specifies three distinct retry axes and their termination conditions, with
  state.json capturing the axis used per attempt.

- **Phase 3.5b behavior**. In v1.2 a dirty worktree was a hard stop. In v1.3
  it is a recovery-quarantine. Both versions refuse to merge unreviewed
  work; v1.3 keeps the overall run alive so other workstreams can continue.

### Non-Goals (explicitly not shipping in v1.3)

- Playwright login smoke as a universal default — left configurable per the
  PO recommendation; many callers (plugins, CLIs, libraries) have no login.
- Unit tests of reconciliation logic — the skill is markdown, not code.
  Scenario fixtures cover the same fear (regression of safety rules).
- Auto-merge of recovered worktrees if their post-recovery verify passes —
  proposed by the Tech Lead, deferred to a future `--autonomous-aggressive`
  flag. v1.3 default is conservative: human must review quarantined branches.

## [1.2.0] — 2026-04-17

### Added — `/orchestrate`

- **Phase 3.5: Merge-Back Safety Checklist** enforced before every worktree
  merge, with four sub-steps:
  - 3.5a — Leave the worktree: compute `$MAIN_ROOT`, verify working
    directory is the main repo root, never run merges from inside a
    worktree being merged or deleted.
  - 3.5b — Fail loudly on uncommitted work in the worktree (refuse to merge;
    no auto-commit). *Note: revised to recovery-quarantine in v1.3.*
  - 3.5c — Scope diff: compare changed files against the workstream's
    declared scope and flag spillover / creep / foundation overlap.
  - 3.5d — Record `merge_safety` block in state.json.
- Cross-boundary contracts for producer/consumer workstream pairs.
- Regression guard in the workstream agent prompt: catalog pre-existing
  public interfaces before edits, verify they still work after.
- Integration fix review gate: any fix commits made during the final
  integration pass must be reviewed before re-verify.

### Changed

- Merge protocol prerequisites now require the Phase 3.5 checklist to pass.
- Removed the old auto-commit fallback in `merge-protocol.md` Step 1b in
  favor of the hard-stop / recovery behavior.

## [1.1.0]

Initial public marketplace release of the plugin (parallel development
workflow skills: research, deliberation, implementation, code review, and
multi-workstream orchestration with worktree isolation).
