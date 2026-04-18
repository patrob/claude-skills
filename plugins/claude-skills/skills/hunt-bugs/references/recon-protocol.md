# Recon Protocol

Full prompt for the Phase 1 recon agent. The agent produces a ranked list of
5-10 bug hypotheses for a named subsystem, with file/line evidence for each.

## Agent launch

```
Agent({
  description: "Recon: {subsystem} bug hypotheses",
  subagent_type: "Explore",
  prompt: <the prompt below, with placeholders filled in>
})
```

Use `Explore` so the agent can search broadly but cannot edit files. It should
cost context, not repo state.

## Agent prompt template

```
You are the recon phase of a subsystem bug hunt. You do NOT write code or
tests — your only deliverable is a ranked hypothesis list that later agents
will use to write failing integration tests.

## Subsystem
{subsystem name — e.g., "Clerk auth", "upload pipeline", "subscription tier logic"}

## Bounding hints (if provided)
{paths / globs / issue context — otherwise: "none, infer from the name"}

## Task

1. Map the subsystem
   - Find its entry points (routes, handlers, jobs, CLI commands, UI actions)
   - Find its boundaries (DB tables/models, external APIs, queues, caches)
   - Read the 5-10 most central files thoroughly, not skimmed
   - Skim tests in the subsystem to understand what IS already covered —
     the gaps are where bugs live
   - Check recent git history in the subsystem:
     `git log --oneline -30 -- {subsystem paths}`
     — recent churn is a bug-rich area
   - Read CLAUDE.md for project-wide gotchas (auth patterns, DB mock chains,
     tier logic quirks, timezone handling, etc.)

2. Generate 5-10 ranked hypotheses

   Cover a spread of failure modes — do not cluster all hypotheses around one
   theme. Aim for at least one hypothesis in each category that applies:

   - **Auth / authorization edges**: role checks missing, token staleness,
     session vs. user mismatch, impersonation, webhook signature verification
   - **Input validation**: unvalidated or weakly-validated inputs reaching
     the DB / external API (length, type, enum, null, unicode, control chars)
   - **Tier / quota / billing logic**: off-by-one on limits, wrong tier
     resolution for legacy/grandfathered users, free-tier bypass, proration
   - **Concurrency / races**: double-submit, concurrent writes without a
     transaction, read-modify-write, idempotency-key collisions
   - **Error / retry paths**: silent catch, retries amplifying side effects,
     partial failure leaving inconsistent state, timeouts too short/long
   - **Data integrity**: missing foreign key cascades, soft-delete leaks,
     pagination off-by-one, ordering instability, stale cache
   - **Timezone / date**: UTC vs. local, DST, "today" computed wrong,
     cron expression misread, relative date math at month/year boundaries
   - **Migration / version drift**: optional-to-required columns,
     enum additions, shape changes not honored by old rows
   - **External API coupling**: assumed-synchronous webhooks, assumed-ordered
     events, missing retry/backoff, rate-limit handling
   - **Contract mismatches**: API response shape drift between producer and
     consumer, frontend expecting fields the backend renamed, event schema
     vs. consumer schema

   For each hypothesis, produce evidence, not vibes. "The auth middleware
   might not check role" is not a hypothesis — find the specific handler
   that lacks the check and cite `file:line`.

3. Rank by likelihood × impact

   - **Likelihood**: HIGH if the evidence is concrete (missing check, visible
     code path, recent change); MED if the pattern is risky but you haven't
     confirmed the gap; LOW if it's speculative from context alone
   - **Impact**: HIGH if it corrupts data, leaks data across tenants, bypasses
     auth, or charges the wrong amount; MED if it degrades UX or produces
     wrong but recoverable results; LOW if cosmetic

   Rank HIGH×HIGH first. Aim for the top 5 to be mostly HIGH or HIGH×MED.

4. For each hypothesis, describe the test that would prove it

   - **Entry point**: the real code the integration test calls (route handler,
     service function, job entrypoint — NOT a mock)
   - **Fixtures needed**: DB state, auth context, feature flags
   - **Action**: what the test does (one sentence)
   - **Expected correct behavior**: what SHOULD happen
   - **Expected buggy behavior**: what you think WILL happen today

## Output

Write to: `.pipeline/{RUN_NAME}/hypotheses.md`

Use this exact structure so the downstream test-writer agents can parse it:

    # Hypotheses: {subsystem}

    Generated: {ISO timestamp}
    Total: {N}

    ---

    ## H1 — {one-line title}
    **Likelihood**: HIGH | **Impact**: HIGH | **Rank**: 1

    **Evidence**:
    - {file:line} — {what this code does / fails to do}
    - {file:line} — {corroborating evidence}
    - Git: {commit sha or "no recent churn"} — {relevance}

    **Reproduction sketch**:
    - Entry point: {function or route}
    - Fixtures: {DB state, auth, flags}
    - Action: {one sentence}
    - Correct: {expected}
    - Buggy: {what actually happens today}

    **Suggested test shape**:
    - Test type: {integration — service-level / route-level / job-level}
    - Key assertion: {the single assertion that will fail if the bug is real}

    ---

    ## H2 — ...

## Quality bar

- Fewer than 5 hypotheses → you have not looked broadly enough. Go back and
  cover more of the failure-mode categories above.
- More than 10 → you are padding. Cut the weakest.
- Any hypothesis without a `file:line` pointer is not a hypothesis. Replace it
  or remove it.
- Any hypothesis whose "suggested test" would require testing the UI end-to-end
  via a browser is out of scope — re-scope it as a service-level integration
  test, or drop it.
```

## Retry prompt (if recon returns < 5 hypotheses)

Send the agent back exactly once with an appended instruction:

```
Your previous output had fewer than 5 hypotheses. Do not repeat the ones you
already produced. Expand specifically into these categories you missed:
{list of uncovered categories from the list above}

Produce {5 - current_count} additional hypotheses that meet the evidence bar
(file:line pointers, concrete reproduction sketch). If you genuinely cannot
find more real hypotheses in this subsystem, say so explicitly rather than
fabricating them.
```

If it still can't reach 5, trust the agent — do not fabricate hypotheses to
hit the quota.
