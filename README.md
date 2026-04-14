# Claude Code Skills Collection

A collection of Claude Code skills for parallel development workflows and code review.

## Installation

Install as a Claude Code plugin directly from GitHub:

```bash
claude plugin install patrob/claude-skills@github
```

Once installed, all skills are available with the `claude-skills:` prefix:

```
/claude-skills:workflow       # Pipeline orchestrator
/claude-skills:orchestrate    # Multi-workstream parallel build
/claude-skills:pw-research    # Deep parallel research
/claude-skills:pw-review      # 5-lens code review
```

### Manual Installation

If you prefer to install manually, copy the skills and commands into your Claude Code configuration:

```bash
# Clone the repo
git clone https://github.com/patrob/claude-skills.git

# Copy all skills
cp -r claude-skills/skills/* ~/.claude/skills/

# Copy the workflow command
cp claude-skills/commands/workflow.md ~/.claude/commands/
```

### Local Development

To test changes locally before committing:

```bash
claude plugin install --local /path/to/claude-skills
```

## What's Included

### Development Pipeline Skills

These skills chain together into a complete development pipeline. Use them individually or via the `/workflow` orchestrator.

| Skill | Description |
|-------|-------------|
| **[workflow](commands/workflow.md)** | Pipeline orchestrator — triages work and chains the right skills together |
| **[orchestrate](skills/orchestrate/)** | Multi-workstream parallel orchestration with isolated worktree agents |
| **[pw-research](skills/pw-research/)** | Deep parallel research with 3 agents and validation |
| **[pw-deliberate](skills/pw-deliberate/)** | Product Owner + Tech Lead debate to produce checkpoint plans |
| **[pw-implement](skills/pw-implement/)** | Checkpoint-gated parallel implementation with verification gates |
| **[pw-review](skills/pw-review/)** | 5-lens parallel code review (UX, Security, Performance, Tests, Production) |
| **[pw-bug-hunt](skills/pw-bug-hunt/)** | TDD bug fix — investigate with parallel hypotheses, write failing test, then fix |
| **[verify-fix-loop](skills/verify-fix-loop/)** | Iterative verification and fix cycle until all checks pass |

### Pipeline Flow

```
/workflow triages → selects pipeline:

Bug:     pw-bug-hunt
Small:   pw-implement → verify-fix-loop → pw-review
Feature: pw-research → pw-deliberate → pw-implement → verify-fix-loop → pw-review

/orchestrate scales this to multiple parallel workstreams:
  Parse roadmap → decompose into workstreams → spawn worktree agents
  → each agent runs full /workflow pipeline → review → merge
```

## Customization

These skills are designed to be adapted to your project. Key things to customize:

- **Review lenses** (`skills/pw-review/references/review-lenses.md`): Update the review criteria to match your tech stack. The defaults reference Next.js/MongoDB/Tailwind — change these to match your project.
- **Verification commands**: Skills reference `make verify`, `npx tsc`, `npx vitest` — adapt to your project's tooling.
- **Checkpoint format** (`skills/pw-deliberate/references/checkpoint-format.md`): The plan template works for most projects but can be extended.

## How It Works

### Parallel Agent Architecture

The core insight: Claude Code can launch multiple subagents in parallel. These skills exploit this for:

- **Research**: 3 agents investigate different angles simultaneously (web, codebase, project context)
- **Review**: 5 agents review through different lenses concurrently
- **Implementation**: Multiple agents implement different files/modules in parallel
- **Orchestration**: Multiple worktree agents each run the full pipeline independently

### Worktree Isolation

The orchestrate skill uses `isolation: "worktree"` to give each agent its own git worktree. This means:

- Agents can't interfere with each other
- Each agent has its own working directory
- Changes are merged back via `git merge --no-ff`
- Failed workstreams don't pollute the main branch

### Quality Gates

Every implementation agent must pass three gates before reporting done:

1. **TypeScript compilation** (`tsc --noEmit`)
2. **Test execution** (run relevant tests)
3. **Prohibited pattern scan** (no `@ts-ignore`, no `any` escape hatches, no skipped tests)

See `skills/pw-implement/references/agent-protocol.md` for the full protocol.

## License

MIT
