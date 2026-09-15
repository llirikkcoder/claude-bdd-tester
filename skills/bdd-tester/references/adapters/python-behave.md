# Adapter — Python (behave / pytest-bdd)

Стек: **Python + behave** или **Python + pytest-bdd**.

## Пример `.bdd-config.yaml` (behave)

```yaml
stack: python
driver: behave
feature_dirs: ["features/"]
step_dirs: ["features/steps/"]
helper_dirs: ["features/helpers/"]
source_globs: ["src/**/*.py"]
ignore_globs: ["**/migrations/**"]

harness:
  command: "behave {feature_path} --no-capture --format=progress3"
  command_scenario: "behave {feature_path} -n \"{scenario_name}\""
  timeout_sec: 300
```

## Пример `.bdd-config.yaml` (pytest-bdd)

```yaml
stack: python
driver: pytest-bdd
feature_dirs: ["tests/bdd/features/"]
step_dirs: ["tests/bdd/steps/"]
source_globs: ["src/**/*.py"]

harness:
  command: "pytest tests/bdd --gherkin-terminal-reporter -q {feature_path}"
  command_scenario: "pytest tests/bdd -k \"{scenario_name}\" -q"
```

## Структура features/

```
features/
├── returns/
│   ├── return_within_14_days.feature
│   └── partial_refund.feature
├── steps/
│   ├── __init__.py
│   ├── returns.py                # @given/@when/@then
│   └── common.py                 # общие шаги
├── helpers/
│   ├── __init__.py
│   ├── purchase.py               # domain helpers (stubs при генерации)
│   └── returns.py
└── environment.py                # before_all/before_scenario hooks
```

## Пример шага (behave)

```python
# features/steps/returns.py
from behave import given, when, then
from features.helpers import purchase, returns as returns_helpers

@given("клиент купил товар {days:d} дней назад")
def step_impl(context, days):
    context.purchase = purchase.make(days_ago=days)

@when("он оформляет возврат")
def step_impl(context):
    context.refund = returns_helpers.request(context.purchase)

@then("деньги возвращаются на карту в течение {days:d} дней")
def step_impl(context, days):
    assert context.refund.arrival_days <= days, \
        f"expected ≤{days}, got {context.refund.arrival_days}"
```

## Пример helper-stub (для bdd-step-writer, чтобы bdd-implementer потом заполнил)

```python
# features/helpers/purchase.py
# TODO: bdd-implementer — реализовать
def make(days_ago: int):
    raise NotImplementedError("bdd-implementer: создать Purchase с датой в прошлом")
```

## Environment.py (behave-specific)

```python
# features/environment.py
def before_all(context):
    # инициализация тестовой БД, LLM test-double, etc.
    pass

def before_scenario(context, scenario):
    # чистая БД / сброс fixtures
    pass

def after_scenario(context, scenario):
    # cleanup
    pass
```

## Парсинг вывода harness-runner'ом

**behave (progress3):**
```
Feature: Возврат товара                 # features/returns/return_within_14_days.feature:1
  Scenario: Возврат в срок              # features/returns/return_within_14_days.feature:5
    Given клиент купил товар 10 дней назад       # features/steps/returns.py:5 0.001s
    When он оформляет возврат                    # features/steps/returns.py:9 0.002s
    Then деньги возвращаются на карту в течение 3 дней  # features/steps/returns.py:13 0.001s
      Assertion Failed: expected ≤3, got 5
```

harness-runner извлекает: `Scenario:` заголовок → статус (`Assertion Failed` = failed), `# path:line` = failing step, tail = reason.

**pytest-bdd:** стандартный pytest-репорт с префиксом сценария.

## Мокинг

- HTTP → `pytest-httpx` или `responses`
- LLM → test-double (для AZ-RAG: путь из `bdd-config.mocks.llm`)
- Time → `freezegun`
- FS → `pyfakefs`
- DB → `pytest-postgresql` / `testcontainers`

**Никогда** не мокай внутри step definition. Моки — в `environment.py` (behave) или `conftest.py` (pytest-bdd) fixtures.

## AZ-RAG-специфика (если применяется)

- `before_all`: подключить MLflow test-tracker (не реальный), YandexGPT test-double, dummy audit-sink вместо ChatAuditEvent
- Сценарии для chat-UI: safety-warning, source-citation, refusal on off-topic
- GxP: сценарии = User Requirements Spec fragment; хранить в `features/gxp/` отдельно
