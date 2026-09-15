# claude-bdd-tester

A [Claude Code](https://claude.com/claude-code) skill that turns **Behaviour-Driven Development into agent-driven infrastructure**. Requirements become approved Gherkin scenarios, an outside-in red-green loop drives implementation until every scenario is green, and a drift detector keeps the scenarios honest as the codebase evolves.

The scenarios aren't just tests — they're the **stop condition**. Wrap `/bdd-run --goal` as the terminal check in an agent `/goal` loop and the agent literally cannot claim "done" until the behavior it was asked to build actually passes.

## Why

Most AI coding agents will happily declare victory once the code compiles or a unit test passes, even if the resulting behavior doesn't match what was asked for. BDD scenarios close that gap:

- They're written in **Gherkin** (Given-When-Then), readable by a non-technical stakeholder who can confirm "yes, that's the behavior I meant."
- They're **executable** — a harness (behave, pytest-bdd, cucumber-js, playwright-bdd, or a parser-less event-driven variant) runs them for real.
- They double as a **living spec**: a periodic drift check flags scenarios that reference dead code, production code with no scenario coverage, and scenarios that are green but no longer actually assert anything meaningful.

## When to use BDD (and when not to)

**Use it when:**
- The product has user-facing flows a non-technical stakeholder needs to review.
- There's a critical revenue/UX path worth regression-locking (payment, onboarding, checkout funnel).
- An agent `/goal` loop needs a hard, executable stop condition.
- You're in a regulated environment where Gherkin can double as a User Requirements Spec (GxP/HIPAA).

**Skip it when:**
- It's pure backend/library code with unit-testable functions — use a unit-testing pipeline instead.
- You're evaluating raw LLM output quality — that calls for statistical evals (e.g. RAGAS + ground-truth gates), not boolean scenario pass/fail.
- There's no non-technical stakeholder and no critical user flow — BDD is pure overhead in that case.

Full decision matrix: [`skills/bdd-tester/references/bdd-vs-tdd.md`](skills/bdd-tester/references/bdd-vs-tdd.md).

## How it works

The skill orchestrates six sub-agents, each with a narrow, single-purpose role:

| # | Agent | Model | Role |
|---|-------|-------|------|
| 1 | `bdd-planner` | Opus | Turns a requirement into a draft scenario tree |
| 2 | `bdd-reviewer` | Opus | Scores scenarios on readability, declarative style, one-behavior-per-scenario, edge-case coverage. Requires ≥9/10, max 3 revision cycles |
| 3 | `bdd-step-writer` | Opus | Writes step definitions (behave/cucumber/playwright/…). Never touches production code |
| 4 | `bdd-harness-runner` | Sonnet | Mechanically runs the harness, returns a compact per-scenario report. No opinions, no fixes |
| 5 | `bdd-implementer` | Opus | Outside-in TDD: turns one failing scenario green with the smallest possible change. Never touches `.feature` files or step definitions |
| 6 | `bdd-drift-detector` | Sonnet | Periodic health check: dead references, coverage gaps, trivially-green scenarios |

Each agent is deliberately boxed in — the planner never writes code, the implementer never edits scenarios — so the loop can't quietly cheat its own stop condition.

## Three modes

### 1. Discovery — `/bdd-discover <requirement>`

Turns a requirement (PRD fragment, ticket, user story, or free-form text) into an approved `.feature` file.

```
requirement → bdd-planner → .feature draft → bdd-reviewer (loop, max 3) → approved
```

Open questions the planner surfaces are never silently resolved — they're handed back to you.

```
/bdd-discover docs/requirements/RETURNS.md
/bdd-discover "Customer gets an email confirmation on signup"
/bdd-discover tickets/JIRA-1234.md --feature-name checkout
```

### 2. Run loop — `/bdd-run [feature_path] [--goal]`

Without `--goal`: runs the harness once, prints a per-scenario pass/fail report.

With `--goal`: enters an **outside-in red-green loop** — picks the first failing scenario, writes missing step definitions if needed, delegates to `bdd-implementer`, re-runs, repeats. Stops on: all green, `max_goal_iterations` reached, the same scenario failing 3 times in a row, or the implementer explicitly escalating. The implementer is never allowed to edit `.feature` files or step definitions to force a pass.

```
/bdd-run                                          # all features
/bdd-run features/returns/ --goal                 # TDD loop
/bdd-run --scenario "Return within window"        # one scenario
```

### 3. Drift — `/bdd-drift [--since <ref>] [--full]`

A living-doc health check across three classes of drift:

- **Dead references** (critical) — a step definition points at code that no longer exists.
- **Coverage gaps** (warning) — production code changed with zero scenario updates.
- **Trivially green** (warning) — a scenario passes but its assertions no longer mean anything (empty helper, `assert not None` only).

Never auto-fixes — it reports, you decide.

```
/bdd-drift                    # since last tag
/bdd-drift --since main
/bdd-drift --full
```

## Getting started

1. Copy `skills/bdd-tester/`, `agents/*.md`, and `commands/*.md` into your Claude Code config — either globally (`~/.claude/`) or per-project (`.claude/`).
2. In your target repo, run:
   ```
   /bdd-init
   ```
   This auto-detects your stack (`pyproject.toml`/`package.json`) and driver (behave, pytest-bdd, cucumber-js, playwright-bdd), writes `.bdd-config.yaml`, and scaffolds `features/`, `features/steps/`, and an example `.feature` file.
3. Turn a requirement into scenarios:
   ```
   /bdd-discover "your requirement here"
   ```
4. Drive implementation to green:
   ```
   /bdd-run features/your_feature.feature --goal
   ```
5. Periodically check the scenarios still describe reality:
   ```
   /bdd-drift
   ```

## Supported stacks

Driven by `.bdd-config.yaml` — one config, five adapters:

- Python + [behave](skills/bdd-tester/references/adapters/python-behave.md)
- Python + pytest-bdd (shares the behave adapter's pytest guidance)
- JS/TS + [cucumber-js](skills/bdd-tester/references/adapters/js-cucumber.md)
- JS/TS + [playwright-bdd](skills/bdd-tester/references/adapters/playwright-bdd.md) (e2e web flows)
- [Event-driven](skills/bdd-tester/references/adapters/event-driven.md) — a parser-less variant (Abdullin-style)

A starter config with presets for each stack lives at [`skills/bdd-tester/templates/.bdd-config.yaml`](skills/bdd-tester/templates/.bdd-config.yaml).

## Wiring into a `/goal` loop

Because `/bdd-run --goal` exits non-zero until every scenario passes, it works as a hard stop condition for an autonomous agent loop:

```yaml
stop_condition:
  command: "/bdd-run --goal features/critical/*.feature"
  expected_exit: 0
```

The agent cannot declare the task done until the scenarios it was supposed to satisfy actually pass.

## Repository layout

```
skills/bdd-tester/
  SKILL.md                          — orchestrator: prerequisites, the 3 modes, adapter routing
  references/
    bdd-vs-tdd.md                   — decision matrix: BDD vs unit tests vs statistical evals
    gherkin-style.md                — declarative vs imperative, Background usage, Scenario Outlines
    outside-in-workflow.md          — how red-green-refactor works with scenarios as the outer loop
    bdd-config.md                   — full .bdd-config.yaml schema
    hooks-recipe.md                 — wiring BDD into commit/stop hooks
    adapters/                       — per-stack driver guidance
  templates/
    .bdd-config.yaml                — starter config with per-stack presets
    example.feature                 — Gherkin starter

agents/            — the 6 sub-agent definitions
commands/           — /bdd-discover, /bdd-run, /bdd-init, /bdd-drift
```

## Rules the orchestrator follows

- Never skip discovery ("just implement") — scenario drift becomes inevitable.
- Never mix run-loop and discovery in one call — the planner and the implementer need different context.
- The implementer never touches scenarios or step definitions, silently or otherwise.
- Drift reports are never auto-fixed, and never ignored just because the harness currently passes.

## License

No license specified yet — all rights reserved by default. Open an issue if you want to use this and need a specific license.
