# Outside-in workflow

Как режим `/bdd-run --goal` превращает BDD в стоп-условие агентного цикла.

## Общая идея

Классический TDD: red → green → refactor **на уровне unit-тестов**.

Outside-in BDD: тот же цикл, но **внешний контур — сценарий**, внутренний — unit-tests (опционально).

```
┌───────────────────────────── outer loop (сценарий) ───────────────────────────────┐
│                                                                                    │
│   red scenario                                                                     │
│      │                                                                             │
│      ▼                                                                             │
│   bdd-implementer ──── может внутри запустить ──► inner loop (unit-tests) ──┐     │
│      │                                                                       │     │
│      ▼                                                                       │     │
│   re-run harness  ◄──────────────────────────────────────────────────────────┘     │
│      │                                                                             │
│      ├── green → следующий red scenario                                            │
│      └── red → снова bdd-implementer, iteration++                                  │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

## Инвариант: сценарии — контракт, не переменная

**Правило #1:** implementer никогда не редактирует .feature или step defs, чтобы сделать красный сценарий зелёным. Если сценарий кажется невыполнимым — эскалируй, не двигай цель.

**Правило #2:** одна итерация — один сценарий. Не «параллельно всё зелёное». Иначе теряешь ясность что именно починил implementer.

**Правило #3:** re-run harness после каждого implementer-изменения. Дешевле, чем chase-a-flake через 5 итераций.

## Стоп-условия /bdd-run --goal

1. **Все сценарии зелёные** → done, exit success.
2. **Достигнут `max_goal_iterations` (default 8)** → эскалация: покажи прогресс, спроси continue/abort.
3. **Один и тот же сценарий красный 3 итерации подряд** → эскалация: implementer не может это починить, вероятно проблема в проектировании (сценарий/архитектура/step def).
4. **`bdd-implementer` вернул `escalate: true`** → сразу surface причину.
5. **Внесение изменений сломало ранее зелёный сценарий** → эскалация: regression, обычно сигнал что implementer не учёл side-effect.

## Взаимодействие с /goal skill

Если проект использует `/goal` skill из vault ([[Циклы кода — loops, goal, loop, schedule]]):

```yaml
# в yaml goal-цикла
stop_condition:
  type: command
  command: "/bdd-run --goal features/critical/*.feature"
  expected_exit: 0
```

Тогда агент буквально не может «объявить готово», пока `/bdd-run --goal` не вернёт 0. Это и есть AI-Native Harness из заметки vault.

## Как implementer работает внутри

`bdd-implementer` получает один failing scenario + failure stack. Его алгоритм:

1. Прочитать step def, который дошёл до `AssertionError`.
2. Прочитать helper, к которому обращается step def.
3. Прочитать production-код, куда helper тянется.
4. Сделать **минимальное** изменение (не рефакторинг!) чтобы зелёный сценарий.
5. Пометить `follow_ups` — что стоит подчистить потом.

Внутри implementer'а может быть unit-TDD (написать unit-test → фикс → зелёный → unit-tester полный прогон), но это опция, не обязательство. Внешний контур — сценарий.

## Anti-patterns outside-in

- **Implementer правит сценарий, потому что "требование неполное"** — эскалируй, попроси человека уточнить, не додумывай
- **"Быстро сделать все зелёные разом"** — потеря контроля; всегда по одному
- **Обход step definition** через прямой вызов production-функции в hook'е — цепь `feature → step → helper → prod` должна работать целиком
- **Пропуск harness re-run** между итерациями — накопление ошибок
- **Уменьшение max_goal_iterations в горячке** — если 8 не хватает, значит проблема не в лимите

## Метрика прогресса на выходе

При завершении `/bdd-run --goal` (успешно или по эскалации):

```
Iterations: 5/8
Started:    12 red / 3 green
Ended:      2 red / 13 green
Δ:          +10 green
Files:      src/returns/refund.py, src/returns/eligibility.py, features/steps/returns.py (steps only, no scenarios touched)
Follow-ups: 2 (magic literals, deprecated log call)
```
