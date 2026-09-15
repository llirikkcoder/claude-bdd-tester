# Adapter — Event-driven BDD (вариант Абдуллина)

**Идея**: G-W-T как данные/код поверх event-driven архитектуры, без Gherkin-парсера. Given = список начальных событий, When = команда, Then = ожидаемые исходящие события. Масштабируется на 10k+ сценариев.

## Когда использовать

- Проект уже на event-driven архитектуре (CQRS / Event Sourcing / Kafka)
- Ожидаешь **сотни-тысячи** сценариев (Gherkin-парсер сам становится боттлнеком)
- Продакт готов читать `.py` или `.json` вместо `.feature`
- Хочешь строгую типизацию сценариев

Если проект классический CRUD и сценариев ≤200 — используй behave/cucumber. Event-driven — тяжёлая артиллерия.

## Пример `.bdd-config.yaml`

```yaml
stack: python
driver: event-driven
feature_dirs: ["scenarios/"]           # тут .py, а не .feature
step_dirs: []                          # не нужны
helper_dirs: ["scenarios/helpers/"]
source_globs: ["src/**/*.py"]

event_driven:
  event_store: "src/events/store.py"
  runner: "python -m scenarios.runner {feature_path}"
  scenario_module_pattern: "*_scenario.py"
  serializer: "json"

harness:
  command: "python -m scenarios.runner {feature_path}"
  command_scenario: "python -m scenarios.runner {feature_path}::{scenario_name}"
```

## Пример сценария

```python
# scenarios/returns/return_within_14_days_scenario.py
from datetime import datetime, timedelta
from src.events import PurchaseCreated, RefundRequested, RefundIssued, RefundRejected
from src.commands import RequestRefund
from scenarios.framework import scenario, given, when, then

@scenario("Возврат в срок")
def happy_path():
    given(
        PurchaseCreated(purchase_id="p1", customer_id="c1", amount=1000,
                        created_at=datetime(2026, 1, 1))
    )
    when(RequestRefund(purchase_id="p1", requested_at=datetime(2026, 1, 11)))
    then(
        RefundRequested(purchase_id="p1", customer_id="c1", amount=1000),
        RefundIssued(purchase_id="p1", arrival_days=3)
    )

@scenario("Отказ на 15-й день")
def rejected_after_deadline():
    given(
        PurchaseCreated(purchase_id="p2", customer_id="c1", amount=500,
                        created_at=datetime(2026, 1, 1))
    )
    when(RequestRefund(purchase_id="p2", requested_at=datetime(2026, 1, 16)))
    then(
        RefundRejected(purchase_id="p2", reason="deadline_exceeded")
    )
```

## Мини-framework

```python
# scenarios/framework.py
from dataclasses import dataclass, field
from typing import Any, Callable

@dataclass
class ScenarioSpec:
    name: str
    given_events: list = field(default_factory=list)
    when_command: Any = None
    then_events: list = field(default_factory=list)

_current: ScenarioSpec | None = None

def scenario(name: str) -> Callable:
    def decorator(fn: Callable):
        global _current
        _current = ScenarioSpec(name=name)
        fn()
        return _current
    return decorator

def given(*events): _current.given_events.extend(events)
def when(command):  _current.when_command = command
def then(*events):  _current.then_events.extend(events)
```

## Runner

```python
# scenarios/runner.py
import importlib
from src.aggregate import Aggregate

def run(scenario_spec):
    agg = Aggregate.replay(scenario_spec.given_events)
    resulting_events = agg.handle(scenario_spec.when_command)
    assert resulting_events == scenario_spec.then_events, \
        f"{scenario_spec.name}: expected {scenario_spec.then_events}, got {resulting_events}"
```

## Плюсы vs Gherkin

| | Gherkin | Event-driven |
|---|---|---|
| Читаемость продактом | ✅ хорошо | ⚠️ средне (нужно объяснить события) |
| Типизация | ❌ строки | ✅ dataclass/pydantic |
| Refactor-safety | ❌ строки-паттерны | ✅ mypy ловит |
| Скорость на 10k сценариев | ⚠️ парсер тяжелеет | ✅ pure Python |
| Порог входа | низкий | высокий |
| Инструментарий | зрелый (Cucumber-семейство) | самопис |

## Как продакту прочитать

Сгенерируй `.md` из сценариев (декоратор может хранить и читаемое имя, и структуру):

```
# Feature: Возврат товара

## Сценарий: Возврат в срок
Дано: клиент купил товар 1 января 2026 за 1000
Когда: клиент запрашивает возврат 11 января 2026
Тогда: возврат оформлен, приход через 3 дня

## Сценарий: Отказ на 15-й день
...
```

Это опция. Но саму истину держит `.py` файл — он же живой контракт.

## Anti-patterns

- Смешивать event-driven и Gherkin сценарии в одном проекте (два паттерна = два места правды)
- Использовать этот подход если у тебя нет реальной event-sourcing архитектуры (мимикрия под неё = боль)
- Ленивое сравнение `then_events` (порядок vs множество) — определи явно, `==` или `set(...) ==`
