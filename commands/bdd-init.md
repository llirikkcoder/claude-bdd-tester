---
description: Создать .bdd-config.yaml + feature-директории в текущем репо. Автодетект стека. Usage — /bdd-init [--stack python|js] [--driver behave|cucumber-js|playwright-bdd|event-driven]
---

# /bdd-init

Bootstrap BDD-инфраструктуры в проекте.

## Аргументы

- `--stack python|js` — опционально. Автодетект по `pyproject.toml` / `package.json`.
- `--driver <name>` — опционально. Автодетект по установленным пакетам (`behave` в reqs → `behave`, `@cucumber/cucumber` в deps → `cucumber-js`, `playwright-bdd` в deps → `playwright-bdd`).
- `--force` — перезаписать существующий `.bdd-config.yaml`.

## Действия

1. Найди корень репо (`git rev-parse --show-toplevel`).
2. Проверь наличие `.bdd-config.yaml`. Если есть и нет `--force` — покажи, спроси overwrite/keep/abort.
3. **Автодетект стека**:
   - `pyproject.toml` / `setup.py` → `python`
   - `package.json` → `js`
   - Ничего → спроси
4. **Автодетект драйвера**:
   - Python: `behave` в deps → `behave`; `pytest-bdd` → `pytest-bdd`; иначе спроси
   - JS: `playwright-bdd` → `playwright-bdd`; `@cucumber/cucumber` → `cucumber-js`; иначе спроси
5. Прочитай шаблон `~/.claude/skills/bdd-tester/templates/.bdd-config.yaml`, активируй нужный preset.
6. Подстрой `source_globs` под фактическую структуру.
7. Запиши `.bdd-config.yaml` в корень.
8. Создай стартовые директории (если их нет):
   - `features/` + пример из `~/.claude/skills/bdd-tester/templates/example.feature`
   - `features/steps/` (Python: с пустым `__init__.py`)
   - `features/helpers/` (Python)
   - `features/environment.py` (behave) — с пустыми `before_all`/`before_scenario`
   - Для playwright-bdd: `playwright.config.ts` scaffold
9. Проверь установлен ли harness-пакет:
   - `behave --version` / `npx cucumber-js --version` / `npx bddgen --version`
   - Если нет — напечатай команду установки (не запускай сам без разрешения)
10. Print следующие шаги:
    ```
    ✅ .bdd-config.yaml создан.
    ✅ features/example.feature — пример сценария.
    ✅ features/steps/ — сюда попадут step definitions.

    Дальше:
      /bdd-discover <requirement>   — превратить требование в сценарии
      /bdd-run features/example.feature  — прогнать пример
      /bdd-drift                    — health-check живой документации

    Полная схема: ~/.claude/skills/bdd-tester/references/bdd-config.md
    Когда BDD не нужен: ~/.claude/skills/bdd-tester/references/bdd-vs-tdd.md
    ```

## Пример

```
/bdd-init                                # автодетект
/bdd-init --stack python --driver behave
/bdd-init --stack js --driver playwright-bdd
/bdd-init --force
```
