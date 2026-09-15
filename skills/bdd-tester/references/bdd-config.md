# `.bdd-config.yaml` — full schema

Лежит в корне репо. Orchestrator читает, передаёт фрагменты sub-agents.

## Полный пример

```yaml
# Stack identity
stack: python                          # python | js | polyglot
driver: behave                         # behave | pytest-bdd | cucumber-js | playwright-bdd | event-driven

# Директории
feature_dirs: ["features/"]
step_dirs: ["features/steps/"]
helper_dirs: ["features/helpers/"]
source_globs: ["src/**/*.py"]          # для drift-detector
ignore_globs: ["**/migrations/**", "**/vendor/**"]

# Harness
harness:
  command: "behave {feature_path} --no-capture --format=progress3"
  command_scenario: "behave {feature_path} -n \"{scenario_name}\""     # для одиночного сценария
  parallel: false
  timeout_sec: 300

# Философия — передаётся planner + reviewer + step-writer + implementer verbatim.
philosophy: |
  - Declarative Given-When-Then. Никаких кликов, URL, полей формы в тексте сценария.
  - Одна поведенческая единица на сценарий.
  - Конкретные значения, не абстрактные.
  - Background — только если общее для >=3 сценариев.
  - Step definitions: DRY-библиотека. Fuzzy-match → rename scenario, не дублируй.
  - Implementer никогда не редактирует .feature или step defs, чтобы сделать зелёным.
  - Тесты — контракт, но недостаточны: security + arch review отдельным контуром.

  step_style: functional            # functional | class_based
  mocks_in_steps: false             # моки должны быть в fixture/conftest

# Discovery loop
max_revisions: 3                    # reviewer iterations
reviewer_threshold: 9               # 1–10

# Goal loop
max_goal_iterations: 8              # /bdd-run --goal ceiling
same_scenario_retry_limit: 3        # тот же сценарий 3 раза красный → эскалация

# Drift
drift_thresholds:
  significant_change_lines: 30
  coverage_gap_severity: warning    # warning | critical

# Cost / speed
model_overrides: {}
```

## Поля

| Field | Required | Default | Notes |
|-------|----------|---------|-------|
| `stack` | ✓ | — | Выбор adapter reference |
| `driver` | ✓ | — | behave/pytest-bdd/cucumber-js/playwright-bdd/event-driven |
| `feature_dirs` | ✓ | — | Куда пишет planner |
| `step_dirs` | ✓ | — | Куда пишет step-writer |
| `helper_dirs` | ✗ | `step_dirs + '/helpers'` | Где живут stub-хелперы |
| `source_globs` | ✓ | — | Для drift-detector |
| `ignore_globs` | ✗ | `[]` | Для drift-detector |
| `harness.command` | ✓ | — | С `{feature_path}` |
| `harness.command_scenario` | ✗ | fallback to `command` | С `{scenario_name}` |
| `harness.parallel` | ✗ | `false` | Позволяет ли driver |
| `harness.timeout_sec` | ✗ | `300` | Cap per run |
| `philosophy` | ✗ | (defaults) | Переопределяет [[gherkin-style]] |
| `max_revisions` | ✗ | `3` | Reviewer iterations |
| `reviewer_threshold` | ✗ | `9` | 1–10 |
| `max_goal_iterations` | ✗ | `8` | /bdd-run --goal |
| `same_scenario_retry_limit` | ✗ | `3` | |
| `drift_thresholds.*` | ✗ | (see above) | |
| `model_overrides` | ✗ | `{}` | Rare |

## Где держать

- **Репо root** — `.bdd-config.yaml`. Коммитим в git.
- **Локальные оверрайды** — `.bdd-config.local.yaml` (git-ignored). Merged поверх.

## Событийный вариант (Абдуллин)

Если `driver: event-driven`, добавить блок:

```yaml
event_driven:
  event_store: "src/events/store.py"
  scenario_dir: "scenarios/"                    # .py вместо .feature
  runner: "python -m scenarios.runner {scenario_file}"
  serializer: "json"                            # json | msgpack | protobuf
```

Тогда сценарии — Python-модули с `given = [Event1, Event2]`, `when = Command(...)`, `then = [Event3]`. См. [[adapters/event-driven]].
