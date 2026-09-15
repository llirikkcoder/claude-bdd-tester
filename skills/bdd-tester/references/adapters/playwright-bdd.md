# Adapter — Playwright + BDD (e2e для веб-воронок)

Стек: **`playwright-bdd`** (npm-пакет, склеивает Playwright с Gherkin).

## Когда использовать

- Клиентские сайты-воронки (форма → оплата)
- SaaS/продукт с UI, где нужен end-to-end контракт
- CogniAvatar / любой веб-продукт со стейкхолдером-читателем

## Пример `.bdd-config.yaml`

```yaml
stack: js
driver: playwright-bdd
feature_dirs: ["features/"]
step_dirs: ["features/steps/"]
source_globs: ["src/**/*.{ts,tsx}", "app/**/*.{ts,tsx}"]
ignore_globs: ["**/node_modules/**", "**/.next/**"]

harness:
  command: "npx bddgen && npx playwright test {feature_path}"
  command_scenario: "npx bddgen && npx playwright test {feature_path} -g \"{scenario_name}\""
  timeout_sec: 600
  parallel: true
```

`bddgen` — pre-step: генерирует `.spec.ts` из `.feature`, дальше стандартный playwright test.

## Структура

```
features/
├── funnel/
│   └── checkout.feature
├── steps/
│   ├── funnel.ts
│   └── fixtures.ts               # Playwright fixtures + шаги
playwright.config.ts
package.json
```

## Пример шага (Playwright-fixture-стиль)

```ts
// features/steps/funnel.ts
import { createBdd } from 'playwright-bdd';

const { Given, When, Then } = createBdd();

Given('пользователь на странице оплаты', async ({ page }) => {
  await page.goto('/checkout');
});

When('он заполняет форму валидной картой', async ({ page }) => {
  await page.getByLabel('Номер карты').fill('4242 4242 4242 4242');
  await page.getByLabel('Срок').fill('12/28');
  await page.getByLabel('CVV').fill('123');
  await page.getByRole('button', { name: 'Оплатить' }).click();
});

Then('он видит подтверждение оплаты', async ({ page }) => {
  await expect(page.getByText('Оплата прошла успешно')).toBeVisible();
});

Then('на email приходит чек в течение {int} минут', async ({ mailbox }, minutes) => {
  const email = await mailbox.waitFor({ subject: /чек/i, timeout: minutes * 60000 });
  expect(email.body).toContain('Спасибо за покупку');
});
```

## Playwright config

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';

const testDir = defineBddConfig({
  features: 'features/**/*.feature',
  steps:    'features/steps/**/*.ts',
});

export default defineConfig({
  testDir,
  timeout: 60_000,
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
  },
});
```

## Фикстуры (расширение Playwright)

Если нужен доступ к почте / БД / внешним API — расширь Playwright test fixture:

```ts
// features/steps/fixtures.ts
import { test as base } from '@playwright/test';
import { createMailboxClient } from '../support/mailbox';

export const test = base.extend({
  mailbox: async ({}, use) => {
    const mb = createMailboxClient();
    await use(mb);
    await mb.close();
  },
});
```

## Мокинг

- API → `page.route(...)` (Playwright native) или `msw` через service worker
- Внешние платежные шлюзы → stub endpoint в тестовом окружении
- Email → mailhog / mailpit в docker-compose

## Anti-patterns

- Императивные Given ("клиент на странице /login с параметром ?foo=bar" → это URL, не бизнес-намерение)
- `page.waitForTimeout(5000)` — заменить на явный `waitFor(selector)`
- Одинаковый setup в каждом Given → вынеси в `Before` или в `Background`
- `test.slow()` без причины — сначала диагностируй, потом марки

## Что показать стейкхолдеру

`.feature` файл. Не код step definitions, не Playwright конфиг. Продакт открывает `checkout.feature`, читает 5 сценариев на человеческом языке, комментирует. Это и есть главная фича — она достигается за счёт декларативного стиля шагов.
