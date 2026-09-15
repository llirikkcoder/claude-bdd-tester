# Hooks recipe — как встроить BDD в workflow

Хуки прописываются в `~/.claude/settings.json` (global) или `<repo>/.claude/settings.json` (per-project). Скилл сам не правит `settings.json` — это твоё решение.

## Рецепт 1 — Stop hook: напоминалка после сессии

Если Claude изменил production-файлы под `source_globs`, но не обновил ни одного .feature — напомнить проверить сценарии.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "if [ -f .bdd-config.yaml ]; then src=$(git diff --name-only 2>/dev/null | grep -E '\\.(py|ts|tsx|js)$' | grep -vE '(features/|tests/)'); feat=$(git diff --name-only 2>/dev/null | grep '\\.feature$'); if [ -n \"$src\" ] && [ -z \"$feat\" ]; then echo '💡 Изменены исходники, но .feature не тронуты. Проверь: /bdd-drift'; fi; fi",
            "timeout": 3
          }
        ]
      }
    ]
  }
}
```

## Рецепт 2 — PreCommit reminder

Перед `git commit` напомнить прогнать `/bdd-run` на features, ссылающихся на изменённые исходники (упрощённо — на все).

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if [ -f .bdd-config.yaml ] && echo \"$CLAUDE_TOOL_INPUT\" | grep -qE '^git commit'; then echo '💡 Reminder: /bdd-run перед commit чтобы убедиться что сценарии зелёные.' >&2; fi",
            "timeout": 2
          }
        ]
      }
    ]
  }
}
```

## Рецепт 3 — SessionStart: показать красные сценарии

При старте сессии в проекте с `.bdd-config.yaml` — показать статус (много красных = нужен /bdd-run --goal).

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "if [ -f .bdd-config.yaml ]; then echo '📋 BDD-проект. Быстрая проверка сценариев:'; timeout 20 behave --dry-run --format=progress3 features/ 2>/dev/null | tail -5 || echo '   (skip: harness недоступен)'; fi",
            "timeout": 25
          }
        ]
      }
    ]
  }
}
```

**Осторожно:** `behave --dry-run` не запускает step definitions, только проверяет что все шаги замаплены. Быстро, но не показывает pass/fail. Если хочешь реальный статус — `behave --format=progress3 features/` (может быть медленно).

## Рецепт 4 — Weekly cron: drift check

Раз в неделю запусти `/bdd-drift` через `schedule` skill (не hook, а cron):

```
/schedule "0 9 * * 1" /bdd-drift --since HEAD~7d
```

Понедельник 09:00 — отчёт по недельному дрифту. Не автопочинка, только показать.

## Не рекомендую (антипаттерны)

- **Auto-run /bdd-run --goal в PreCommit** — может идти минуты/десятки минут. Убьёт скорость commit'ов.
- **Блокировать commit если сценарии красные** — на срочных фиксах будешь материться. Лучше CI-gate, не local hook.
- **SessionEnd → auto-invoke /bdd-drift** — контекст сессии не даст времени, зря сожжёшь токены.
- **Auto-invoke /bdd-discover на новый ticket** — сценарии должны рождаться из человеческого понимания требования, не из авто-разбора тикета.

## Для регулируемых сред (GxP/HIPAA)

Hooks — audit signal. Если проект под GxP:
- Все runs должны логироваться → `SessionEnd` hook пишет в audit-file
- Сценарии + результаты = валидационный артефакт → включи в CI-report
- `/bdd-drift` — часть quarterly review, не ad-hoc

## Дефолт для нового проекта

Начни с **Рецепта 1** (Stop-reminder). Он самый безопасный: не блокирует, не жрёт токены, только напоминает.
