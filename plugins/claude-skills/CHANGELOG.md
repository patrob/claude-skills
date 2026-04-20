# Changelog

All notable changes to the `claude-skills` plugin. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this plugin
adheres to semantic versioning.

## [1.4.0] — 2026-04-20

Thin-delegator refactor of `/orchestrate`. The orchestrator is now a pure
coordinator — every code change, test run, and commit is delegated to a
spawned subagent or a sub-skill. Verbosity reduced via progressive
disclosure into four standalone, independently-invocable skills. Switches
implementation to strict one-test-at-a-time TDD per success criterion and
adds a bounded product-owner acceptance gate with showstopper / fast-follow
triage.

### Added — standalone skills (composable by `/orchestrate`, usable alone)

- **`extract-criteria`** — delegate spec decomposition to one subagent that
  returns `{workstreams, ac_map, per_workstream_criteria}` as JSON. Every
  success criterion carries a runnable `check`. Replaces
  `phase-1-decompose.md`. Invocable directly against any spec/PRD.
- **`tdd-cycle`** — TDD one criterion via test-writer → implementer →
  verifier → fix-loop, each a separate subagent. The implementer receives
  the failing test's content, path, and expected-error as structured
  context (no handoff guesswork). Wraps `verify-fix-loop`. Hard cap 3
  fix-loop iterations per criterion with distinct retry axes.
- **`parallel-workstreams`** — create worktrees, assemble worktree-agent
  prompts, launch all agents in a single message for true concurrency.
  Each worktree agent loops `Skill(tdd-cycle)` over its criteria.
  Absorbs `phase-2-spawn.md`. Supports Round 0 foundation on the feature
  branch, then parallel rounds.
- **`po-acceptance`** — bounded PO-persona subagent (read-only + configured
  smoke only; no Playwright install mid-run). Emits
  `{verdict, showstoppers, fast_follows, updated_criteria}`. Showstoppers
  carry a proposed new criterion the caller merges into the next TDD
  round. Triage routing is 10 lines inline in `orchestrate/SKILL.md`.

### Changed — `/orchestrate`

- **Delegation Contract** at the top of `SKILL.md`: the main agent MUST
  NOT `Edit`/`Write`/`NotebookEdit` source, run build/test/lint/install/
  commit bash, or merge directly. Every mutation is a spawned subagent.
- **Showstopper loop cap: 2 rounds.** After two rounds of PO rejection,
  the run halts with `SHOWSTOPPER_UNRESOLVED`; the same failure class
  appearing twice means the criteria themselves need human rewriting.
- **`state.json` schema_version: 2.** Adds `showstopper_rounds_used`.
  Resume on a pre-v2 state emits "schema v1 detected — re-run required"
  and stops; no silent migration.
- **Progressive disclosure deeper.** `SKILL.md` trimmed to 149 lines
  (down from 202 with new content added). Reference files consolidated:
  `phase-3.5-merge-safety.md` + `merge-protocol.md` +
  `phase-4-verify-final.md` → single `phase-merge-safety.md`;
  `phase-5-report.md` renamed `phase-report.md`; `retry-semantics.md` and
  `dry-run.md` condensed.
- **Worktree coordination preserved intact.** Dirty-worktree quarantine,
  scope-diff classification, foundation-overlap detection, conflict-
  resolution agent, and `verify.final` hook all continue to work — now
  invoked from the merge-coordinator subagent rather than by the main
  orchestrator.

### Removed

- `skills/orchestrate/references/phase-1-decompose.md` → `extract-criteria`
- `skills/orchestrate/references/phase-2-spawn.md` → `parallel-workstreams`
- `skills/orchestrate/references/phase-3-review.md` (pw-review covers it)
- `skills/orchestrate/references/review-loop.md` (pw-review covers it)
- `skills/orchestrate/references/merge-protocol.md` → `phase-merge-safety.md`
- `skills/orchestrate/references/phase-4-verify-final.md` → `phase-merge-safety.md`
- `skills/orchestrate/references/phase-3.5-merge-safety.md` → `phase-merge-safety.md`
- `skills/orchestrate/references/phase-5-report.md` → `phase-report.md`
- `skills/orchestrate/references/self-test.md` (reduced to a pointer in fixtures/README.md)

### Migration

User-facing invocation is unchanged (`/orchestrate {input}`, `fast`,
`thorough`, `auto`, `--dry-run`, `resume`). Pre-v2 `state.json` files are
not auto-migrated. To resume an in-flight 1.3.x run: re-run from the spec,
or keep 1.3.x installed until the run completes.

## [1.3.0] — 2026-04-17

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
