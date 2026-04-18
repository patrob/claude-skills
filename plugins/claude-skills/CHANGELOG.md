# Changelog

All notable changes to the `claude-skills` plugin. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this plugin
adheres to semantic versioning.

## [1.2.0] — 2026-04-17

Autonomous-workflow upgrade for `/orchestrate`. Introduces the Merge-Back
Safety Checklist together with the recovery-quarantine path, verify gate,
`verify.final` hook, acceptance report, `--dry-run`, and progressive
disclosure of phase detail. Outcome of a Tech Lead / Product Owner
deliberation that reconciled the "stay autonomous" goal with the "never
silently merge garbage" safety guarantee.

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

- **Retry semantics formalized**. Three distinct retry axes with explicit
  termination conditions; state.json captures the axis used per attempt.

- **Merge-Back Safety Checklist (Phase 3.5)**: 3.5a (operate from main repo
  root), 3.5b (dirty worktree → recovery + quarantine, not merged), 3.5c
  (scope diff classified as declared-spillover / creep / foundation-overlap),
  3.5d (record outcome in state.json). Includes cross-boundary contracts
  for producer/consumer pairs, regression guard (catalog pre-existing
  public interfaces before edits, verify after), and an integration-fix
  review gate (fix commits during final integration must be reviewed before
  re-verify).

- Merge protocol prerequisites now require the Phase 3.5 checklist to pass.
  Removed the old auto-commit fallback in `merge-protocol.md` Step 1b in
  favor of the recovery-quarantine behavior.

### Non-Goals (explicitly not shipping)

- Playwright login smoke as a universal default — left configurable per the
  PO recommendation; many callers (plugins, CLIs, libraries) have no login.
- Unit tests of reconciliation logic — the skill is markdown, not code.
  Scenario fixtures cover the same fear (regression of safety rules).
- Auto-merge of recovered worktrees if their post-recovery verify passes —
  proposed by the Tech Lead, deferred to a future `--autonomous-aggressive`
  flag. Default is conservative: human must review quarantined branches.

## [1.1.0]

Initial public marketplace release of the plugin (parallel development
workflow skills: research, deliberation, implementation, code review, and
multi-workstream orchestration with worktree isolation).
