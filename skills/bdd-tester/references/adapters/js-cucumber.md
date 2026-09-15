# Adapter — JS/TS (cucumber-js)

Стек: **cucumber-js** (без Playwright — для чисто-JS API/logic сценариев).

## Пример `.bdd-config.yaml`

```yaml
stack: js
driver: cucumber-js
feature_dirs: ["features/"]
step_dirs: ["features/step_definitions/"]
helper_dirs: ["features/support/"]
source_globs: ["src/**/*.{ts,js}"]
ignore_globs: ["**/node_modules/**", "**/dist/**"]

harness:
  command: "npx cucumber-js {feature_path} --format progress-bar --format json:cucumber-report.json"
  command_scenario: "npx cucumber-js {feature_path} --name \"{scenario_name}\""
  timeout_sec: 300
```

## Структура features/

```
features/
├── returns/
│   └── return_within_14_days.feature
├── step_definitions/
│   ├── returns.ts
│   └── common.ts
├── support/
│   ├── world.ts                  # Custom World для контекста
│   ├── hooks.ts                  # Before/After hooks
│   └── helpers/
│       └── purchase.ts
```

## Пример шага

```ts
// features/step_definitions/returns.ts
import { Given, When, Then } from '@cucumber/cucumber';
import { makePurchase, requestRefund } from '../support/helpers/purchase';
import assert from 'node:assert';

Given('клиент купил товар {int} дней назад', function (days: number) {
  this.purchase = makePurchase({ daysAgo: days });
});

When('он оформляет возврат', function () {
  this.refund = requestRefund(this.purchase);
});

Then('деньги возвращаются на карту в течение {int} дней', function (days: number) {
  assert.ok(this.refund.arrivalDays <= days,
    `expected ≤${days}, got ${this.refund.arrivalDays}`);
});
```

## Custom World

```ts
// features/support/world.ts
import { setWorldConstructor, World } from '@cucumber/cucumber';

class CustomWorld extends World {
  purchase: unknown;
  refund: unknown;
}
setWorldConstructor(CustomWorld);
```

## Hooks

```ts
// features/support/hooks.ts
import { Before, After, BeforeAll, AfterAll } from '@cucumber/cucumber';

BeforeAll(async function () { /* init DB / seeds */ });
Before(async function () { /* per-scenario reset */ });
After(async function () { /* cleanup */ });
```

## Парсинг вывода (JSON reporter)

`--format json:cucumber-report.json` → harness-runner парсит JSON:

```json
[
  {
    "uri": "features/returns/return_within_14_days.feature",
    "elements": [
      {
        "name": "Возврат в срок",
        "type": "scenario",
        "steps": [
          {"keyword": "Given ", "name": "...", "result": {"status": "passed", "duration": 1000000}},
          {"keyword": "Then ",  "name": "...", "result": {"status": "failed", "error_message": "..."}}
        ]
      }
    ]
  }
]
```

## Мокинг

- HTTP → `msw` (mock service worker)
- Time → `sinon.useFakeTimers()` или `@sinonjs/fake-timers`
- FS → `memfs`
- DB → testcontainers / in-memory sqlite

Моки — в `Before` hooks, не в step definitions.

## Anti-patterns

- Мутировать `this` в middle-step (замусоривает World)
- `Promise.all` в step definitions без `await` (флейки)
- Хранить состояние в модуль-скоуп переменных (сломает parallel)
- Regex в step patterns вместо cucumber-expressions (`{int}` > `(\d+)`)
