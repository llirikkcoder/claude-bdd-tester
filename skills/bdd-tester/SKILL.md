---
name: bdd-tester
description: Behaviour-Driven Development pipeline as agent-driven infrastructure. Three modes — (1) discovery — turn a requirement into approved Gherkin scenarios; (2) run-loop — outside-in TDD, red scenarios → green via bdd-implementer; (3) drift — check that scenarios remain a living document. Universal via .bdd-config.yaml — supports behave, pytest-bdd, cucumber-js, playwright-bdd, and event-driven variant. Use when the user says "write scenarios for X", "make this a BDD flow", "verify scenarios still describe the system", or via /bdd-discover, /bdd-run, /bdd-drift commands.
---

# bdd-tester

Orchestrator for BDD as **AI-Native Harness**: scenarios stay in sync with behavior because the harness verifies on every build, and agents use "all scenarios green" as their /goal stop-condition.

## When BDD is the right tool

**Yes** — when at least one is true:
- Product has user-facing flows a non-technical stakeholder needs to review
- Critical revenue/UX paths worth regression-locking (payment, onboarding, funnel)
- Agent /goal-loops need a hard, executable stop condition
- Regulated env where Gherkin doubles as a User Requirements Spec (GxP/HIPAA)

**No** — when:
- Pure backend/library code with unit-testable functions → use `unit-tester` skill instead
- LLM output quality → use RAGAS + GT-gate (statistical evals, stricter than boolean scenarios)
- No non-technical stakeholder and no critical user flow → BDD is overhead

See [[references/bdd-vs-tdd]] for the full decision matrix.

## The 6 agents

| # | Agent | Model | Role |
|---|-------|-------|------|
| 1 | `bdd-planner` | Opus | Requirement → scenario tree draft |
| 2 | `bdd-reviewer` | Opus | Reviews scenarios (readable, declarative, one-behavior). Score ≥9. |
| 3 | `bdd-step-writer` | Opus | Step definitions (behave/cucumber/playwright/…). No production code. |
| 4 | `bdd-harness-runner` | Sonnet | Mechanical run + compact per-scenario report |
| 5 | `bdd-implementer` | Opus | Outside-in TDD: red scenario → green. No touching .feature/steps. |
| 6 | `bdd-drift-detector` | Sonnet | Periodic: dead-refs, coverage gaps, trivially-green |

## Prerequisites

Before starting, **verify**:

1. Working directory is a git repo.
2. `.bdd-config.yaml` exists at repo root. If missing → tell user to run `/bdd-init` and stop.
3. The relevant paths (`feature_dirs`, `step_dirs`, `source_globs`) exist.

Load `.bdd-config.yaml` and pass fragments to sub-agents.

## Mode 1 — Discovery (`/bdd-discover <requirement>`)

Turn a requirement into approved scenarios.

```
requirement (text/ticket/PRD)
      │
      ▼
bdd-planner → scenario tree JSON
      │
      ▼
write .feature file to target_file
      │
      ▼
bdd-reviewer (loop max 3) ──approved──► ready for /bdd-run
      │       ▲
      │       │ revision feedback
      └───────┘
```

**Steps:**

1. Read `requirement` (path or inline text).
2. Delegate to `bdd-planner` with `requirement`, `bdd_config`, `context_files` (grep for related existing features).
3. If plan has `open_questions` — **surface to user, do not silently proceed**. Ask whether to answer them now (better) or proceed with assumptions marked.
4. Write the `.feature` file at `plan.feature.target_file` using the plan's structure.
5. Review loop (max 3 iterations of `bdd_config.max_revisions`):
   - Delegate to `bdd-reviewer`.
   - If approved → break.
   - If revision → delegate to `bdd-planner` again with `feedback` and current `.feature`; rewrite file.
   - If iteration 3 and still not approved → surface, ask approve-as-is or abort.
6. Report:
   ```
   ✅ features/returns/return_within_14_days.feature
      scenarios: 3, reviewer: 9.2/10 approved after 1 revision
      open_questions: 1 (surfaced above)
      next: /bdd-run features/returns/return_within_14_days.feature
   ```

## Mode 2 — Run-loop (`/bdd-run [feature_path] [--goal]`)

Runs the harness. Optional `--goal` flag enters outside-in red-green loop.

**Without --goal (simple run):**
1. Delegate to `bdd-harness-runner` scoped to `feature_path` (or entire feature_dirs).
2. Report per-scenario pass/fail.
3. Stop.

**With --goal (outside-in TDD loop):**
1. Delegate to `bdd-harness-runner`.
2. If all green → done, print `✅ all scenarios green`.
3. Else pick the **first failing scenario** in file order (BDD outside-in — one at a time).
4. If step definitions are missing → delegate to `bdd-step-writer` for that scenario.
5. Delegate to `bdd-implementer` with the failure. Increment iteration count.
6. Re-run `bdd-harness-runner`. Loop.
7. **Stop conditions:**
   - All scenarios green → success
   - `bdd_config.max_goal_iterations` reached → escalate to user with progress summary
   - Same scenario fails 3 times in a row → escalate (probably a deeper design issue)
   - `bdd-implementer` returns `escalate: true` → surface immediately
8. **Never touch .feature or step defs to make failing scenarios pass.** If tempted, escalate.

## Mode 3 — Drift (`/bdd-drift [--since <ref>]`)

Living-doc health check.

1. Delegate to `bdd-drift-detector` with `since_ref` (default: last tag).
2. Print the 3-class report (dead references / coverage gaps / trivially green).
3. **Do not auto-fix.** Ask the user which items to address, then route:
   - Dead references → delete step or restore code — human decision
   - Coverage gaps → optional `/bdd-discover` on the changed source
   - Trivially green → surface for review; probably needs planner revisit

## Adapter routing

Based on `bdd_config.stack + bdd_config.driver`, read the matching adapter reference before invoking sub-agents:

- `python + behave` → [[references/adapters/python-behave]]
- `python + pytest-bdd` → [[references/adapters/python-behave]] (shared pytest guidance)
- `js + cucumber-js` → [[references/adapters/js-cucumber]]
- `js + playwright-bdd` → [[references/adapters/playwright-bdd]]
- `event-driven` → [[references/adapters/event-driven]]

## Interaction with /goal cycles

BDD's power in AI-Native workflow ([[Циклы кода — loops, goal, loop, schedule]]): `/bdd-run --goal` is a **hard stop condition**. Wrap it as the terminal check in a /goal loop and the agent literally cannot claim done until scenarios pass. This is the promise the linked note cites — "docs must be accurate, harness must verify".

## Cost & speed knobs

- `bdd_config.max_revisions` (default 3) — reviewer iterations
- `bdd_config.max_goal_iterations` (default 8) — /bdd-run --goal ceiling
- `bdd_config.harness.parallel` (bool) — whether harness supports parallel scenarios
- `bdd_config.model_overrides` — rare

## References

- [[references/gherkin-style]] — declarative vs imperative, background usage, scenario outlines
- [[references/bdd-vs-tdd]] — when to use BDD vs unit tests (or both, or neither)
- [[references/outside-in-workflow]] — how red-green-refactor works with scenarios as the outer loop
- [[references/bdd-config]] — full schema of `.bdd-config.yaml`
- [[references/hooks-recipe]] — how to wire BDD into commit/stop hooks
- [[references/adapters/python-behave]] — Python behave/pytest-bdd
- [[references/adapters/js-cucumber]] — cucumber-js
- [[references/adapters/playwright-bdd]] — Playwright BDD wrappers
- [[references/adapters/event-driven]] — Abdullin's parser-less variant
- [[templates/.bdd-config.yaml]] — starter config
- [[templates/example.feature]] — Gherkin starter

## Do not

- Skip discovery ("just implement") — scenarios drift becomes inevitable
- Mix Mode 2 and Mode 1 in one call — planner and implementer have different contexts
- Let the implementer touch scenarios (silently or otherwise)
- Ignore drift reports because "the harness passed"
