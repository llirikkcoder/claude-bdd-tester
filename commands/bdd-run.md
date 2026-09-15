---
description: Прогнать BDD-харнесс. С --goal — outside-in red-green loop до всех зелёных или ceiling. Usage — /bdd-run [feature_path] [--goal] [--scenario "name"]
---

# /bdd-run

Запуск harness либо одноразово (simple run), либо в TDD-цикле.

## Аргументы

- `<feature_path>` — опционально. Один `.feature` файл или glob. Default: все features из `feature_dirs`.
- `--scenario "<name>"` — опционально. Только один сценарий (для точечного дебага).
- `--goal` — опционально. Enter outside-in red-green loop до всех зелёных или `max_goal_iterations`.
- `--dry` — опционально. Показать план прогона без реального запуска.

## Действия

**Simple run (без --goal):**
1. Проверь `.bdd-config.yaml`.
2. Invoke skill `bdd-tester` с `mode: run, feature_path: ..., scenario: ...`.
3. Skill вызовет `bdd-harness-runner`, вернёт per-scenario отчёт.
4. Print:
   ```
   Feature: Возврат товара
     ✅ Возврат в срок              (1.2s)
     ❌ Возврат в последний день    (1.4s)
        failing step: Тогда деньги возвращаются на карту в течение 3 дней
        reason: expected ≤3, got 5
        stack: helpers/returns.py:42
     ✅ Отказ на 15-й день          (0.9s)

   Total: 2 passed, 1 failed
   ```

**Goal mode (--goal):**
1. Всё как выше, но если есть красные:
2. Skill enters loop: пиcать первый красный → `bdd-implementer` → re-run harness → повторить.
3. Stop conditions (из [[outside-in-workflow]]): все зелёные / max_goal_iterations / same_scenario_retry_limit / implementer escalate / regression.
4. Финальный вывод:
   ```
   Iterations: 4/8
   Result:     все сценарии зелёные ✅
   Changes:    src/returns/refund.py, src/returns/eligibility.py
   Follow-ups: 2 (magic literal 5, deprecated log call)
   Wall time:  3m 12s
   ```

## Пример

```
/bdd-run                                             # все features
/bdd-run features/returns/                           # директория
/bdd-run features/returns/return_within_14_days.feature
/bdd-run features/returns/ --goal                    # TDD-цикл
/bdd-run --scenario "Возврат в срок"                 # один сценарий
/bdd-run --dry                                       # план без запуска
```

## В /goal-цикле

Стоп-условие в yaml goal-цикла:
```yaml
stop_condition:
  command: "/bdd-run --goal features/critical/*.feature"
  expected_exit: 0
```
Агент не может объявить готово, пока `/bdd-run --goal` не вернёт 0.
