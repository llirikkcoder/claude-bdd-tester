---
description: Health-check живой документации BDD: dead references, coverage gaps, trivially-green сценарии. Usage — /bdd-drift [--since <ref>] [--full]
---

# /bdd-drift

Проверка что сценарии остаются living document, а не мёртвым грузом.

## Аргументы

- `--since <ref>` — опционально. Scope drift-check к изменениям с этого git ref. Default: последний тег или `HEAD~30d`.
- `--full` — опционально. Полное сканирование, игнорирует `--since`. Медленнее, но исчерпывающе.

## Действия

1. Проверь `.bdd-config.yaml`.
2. Invoke skill `bdd-tester` с `mode: drift, since_ref: <ref>, full_scan: <bool>`.
3. Skill вызовет `bdd-drift-detector`, вернёт 3-class отчёт.
4. Print:
   ```
   BDD Drift Report (since abc1234, 12 days ago)

   ❗ Dead references (critical): 1
     - features/steps/returns.py:42 → helpers.legacy.OldRefund (символ удалён в def5678)

   ⚠️ Coverage gaps (warning): 3
     - src/returns/refund.py (+84 lines, 0 сценариев обновлено)
     - src/pricing/promo.py (+56 lines, 0 сценариев)
     - src/auth/token.py (+31 lines, 0 сценариев)

   ⚠️ Trivially green (warning): 2
     - features/returns/edge.feature :: "Возврат бонусами" — helper пустой (pass)
     - features/pricing/tiers.feature :: "Дисконт корпорату" — assert not None only

   Total: 6 items требуют внимания.

   Рекомендуемые действия:
     - Dead ref: удалить/восстановить (человек решает)
     - Coverage gap: /bdd-discover для изменённого src
     - Trivially green: revisit планером
   ```
5. **Не авто-фиксит ничего.** Только сообщает.

## Пример

```
/bdd-drift                          # с последнего тега
/bdd-drift --since main             # относительно main
/bdd-drift --since HEAD~7d          # за неделю
/bdd-drift --full                   # всё, независимо от даты
```

## В cron / schedule

```
/schedule "0 9 * * 1" /bdd-drift --since HEAD~7d
```
Понедельник 09:00 — отчёт за неделю. Не автофикс, только показать.
