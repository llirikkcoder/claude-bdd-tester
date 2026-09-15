---
description: Discovery-режим bdd-tester: требование → одобренные Gherkin-сценарии. Usage — /bdd-discover <requirement_path_or_text>
---

# /bdd-discover

Превращает требование (PRD-фрагмент, тикет, user story, свободный текст) в approved `.feature` файл.

## Аргументы

- `<requirement>` — обязательно. Либо путь к markdown/txt файлу, либо inline-текст в кавычках.
- `--feature-name <name>` — опционально. Явное имя feature (иначе planner придумает).
- `--target <path>` — опционально. Явный target-путь `.feature` (иначе planner подберёт по `feature_dirs`).

## Действия

1. Проверь git repo + `.bdd-config.yaml`. Если нет config — предложи `/bdd-init`.
2. Прочитай requirement:
   - Если путь → Read
   - Если inline → передай как есть
3. Invoke skill `bdd-tester` с параметрами:
   - `mode: discovery`
   - `requirement: <text>`
   - `feature_name_hint: <name or null>`
   - `target_path_override: <path or null>`
4. Skill сам вызовет `bdd-planner` → `bdd-reviewer` (loop) и запишет `.feature` файл.
5. Если планер вернул `open_questions` — они surface'ятся, ты решаешь: ответить сейчас (лучше) или proceed with assumptions.
6. Вывод:
   ```
   ✅ features/returns/return_within_14_days.feature
      scenarios: 4 (1 happy_path, 2 boundary, 1 error)
      reviewer: 9.4/10 approved after 1 revision
      open questions:
        - Что делать с частичным возвратом?
      next: /bdd-run features/returns/return_within_14_days.feature
   ```

## Пример

```
/bdd-discover docs/requirements/RETURNS.md
/bdd-discover "Клиент должен получать email-подтверждение при регистрации"
/bdd-discover tickets/JIRA-1234.md --feature-name checkout
```
