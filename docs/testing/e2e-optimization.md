# Оптимизация и ускорение E2E-тестов

E2E-тесты дают максимальную уверенность, но могут превратить CI в марафон. Если прогон E2E занимает 40 минут, разработчики начинают обходить CI, а тесты теряют ценность.

Эта статья — про то, как сократить время E2E без потери качества.

---

## Содержание

1. [Почему E2E медленные](#почему-e2e-медленные)
2. [Параллельный запуск](#параллельный-запуск)
3. [Шардинг (sharding) в CI](#шардинг-sharding-в-ci)
4. [Готовь состояние через API, а не через UI](#готовь-состояние-через-api-а-не-через-ui)
5. [Reuse auth state](#reuse-auth-state)
6. [Мокируй внешние сервисы](#мокируй-внешние-сервисы)
7. [Стабильные селекторы](#стабильные-селекторы)
8. [Избегай sleep и waitForTimeout](#избегай-sleep-и-waitfortimeout)
9. [Разделяй тесты по тегам](#разделяй-тесты-по-тегам)
10. [Retries и flaky-тесты](#retries-и-flaky-тесты)
11. [Запускай E2E против production-сборки](#запускай-e2e-против-production-сборки)
12. [Селективный запуск E2E](#селективный-запуск-e2e)
13. [Мониторинг времени тестов](#мониторинг-времени-тестов)
14. [Чеклист оптимизации](#чеклист-оптимизации)

---

## Почему E2E медленные

| Фактор | Влияние |
|---|---|
| Запуск браузера | Секунды на каждый тест |
| Навигация по страницам | Сеть, рендеринг, гидратация |
| Заполнение форм через UI | Медленнее, чем API-вызов |
| Ожидание элементов и анимаций | Добавляет задержки |
| Последовательный запуск | Не использует мощности CI |
| Нестабильные тесты | Retries умножают время |

Чтобы ускорить E2E, нужно уменьшить время каждого теста **и** увеличить параллелизм.

---

## Параллельный запуск

Playwright запускает тесты в изолированных браузерных контекстах, поэтому параллелизм безопасен.

```ts
// playwright.config.ts
export default defineConfig({
  fullyParallel: true,
  workers: process.env.CI ? 4 : undefined,
  retries: process.env.CI ? 1 : 0,
});
```

| Параметр | Назначение |
|---|---|
| `fullyParallel: true` | Тесты в одном файле тоже идут параллельно |
| `workers` | Количество параллельных воркеров |
| `retries` | Повторный запуск упавших тестов в CI |

> ⚠️ `fullyParallel` требует полной изоляции тестов. Если тесты делят состояние — будут flaky.

---

## Шардинг (sharding) в CI

Шардинг разбивает набор тестов на части и запускает их на разных runner'ах.

```yaml
# .gitlab-ci.yml
playwright-shard-1:
  script:
    - npx playwright test --shard=1/3

playwright-shard-2:
  script:
    - npx playwright test --shard=2/3

playwright-shard-3:
  script:
    - npx playwright test --shard=3/3
```

Playwright автоматически распределяет тесты по шардам. Если в одном шарде много тяжёлых тестов — время будет неравномерным.

### Балансировка по времени

Для равномерного распределения используй `--shard` вместе с предварительным анализом длительности тестов. Playwright в новых версиях умеет балансировать по истории (`blob` + `merge` reports).

```bash
# Запуск с blob-отчётами
npx playwright test --shard=1/3 --reporter=blob
npx playwright test --shard=2/3 --reporter=blob
npx playwright test --shard=3/3 --reporter=blob

# Merge отчётов
npx playwright merge-reports --from blob-dir
```

---

## Готовь состояние через API, а не через UI

Самая частая ошибка: логиниться через UI перед каждым тестом.

```ts
// ❌ Плохо: 5–10 секунд на логин в каждом тесте
test.beforeEach(async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("user@example.com");
  await page.getByLabel("Password").fill("password");
  await page.getByRole("button", { name: /login/i }).click();
});

// ✅ Хорошо: логинимся через API один раз
const authFile = "playwright/.auth/user.json";

test("user can access dashboard", async ({ page }) => {
  await page.goto("/dashboard");
  await expect(page.getByRole("heading", { name: /dashboard/i })).toBeVisible();
});
```

Аналогично создавай тестовые данные через API или seed базы данных, а не через клики по формам.

---

## Reuse auth state

Playwright позволяет сохранить состояние аутентификации и переиспользовать его.

```ts
// playwright.config.ts
export default defineConfig({
  use: {
    storageState: "playwright/.auth/user.json",
  },
});
```

Создание auth-файла:

```ts
// e2e/auth.setup.ts
import { test as setup } from "@playwright/test";

setup("authenticate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel("Password").fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole("button", { name: /login/i }).click();

  await page.waitForURL("/dashboard");
  await page.context().storageState({ path: "playwright/.auth/user.json" });
});
```

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    { name: "setup", testMatch: /auth\.setup\.ts/ },
    {
      name: "e2e",
      dependencies: ["setup"],
      use: { storageState: "playwright/.auth/user.json" },
    },
  ],
});
```

---

## Мокируй внешние сервисы

Внешние сервисы — платёжки, аналитика, CDN, сторонние виджеты — замедляют и дестабилизируют тесты.

```ts
// Блокируем аналитику
await page.route("**/google-analytics.com/**", (route) => route.abort());
await page.route("**/sentry.io/api/**", (route) => route.abort());

// Мокируем платёжный шлюз
await page.route("**/api/payment", (route) => {
  route.fulfill({
    status: 200,
    body: JSON.stringify({ status: "success" }),
  });
});
```

Для собственного бэкенда решай сам: реальные данные дают уверенность, моки — скорость. Частый компромисс: использовать staging API, но seed'ить данные.

---

## Стабильные селекторы

Тесты падают не потому, что сломан функционал, а потому, что поменялся CSS-класс.

```ts
// ❌ Плохо
await page.click(".btn-primary");

// ✅ Хорошо
await page.getByRole("button", { name: /checkout/i }).click();

// ⚠️ Приемлемо
await page.getByTestId("checkout-button").click();
```

Flaky-тесты из-за селекторов — один из главных источников потерь времени.

---

## Избегай sleep и waitForTimeout

Фиксированные задержки — убийца скорости и стабильности.

```ts
// ❌ Плохо
await page.waitForTimeout(2000);

// ✅ Хорошо: ждём конкретного состояния
await expect(page.getByText("Order confirmed")).toBeVisible();

// ✅ Хорошо: ждём сетевого ответа
const responsePromise = page.waitForResponse("**/api/order");
await page.getByRole("button", { name: /pay/i }).click();
await responsePromise;
```

Playwright автоматически ждёт элементы перед действиями. Явные `waitFor` нужны только для сетевых событий, навигации и исчезновения элементов.

---

## Разделяй тесты по тегам

Не все E2E нужны на каждом коммите. Используй теги:

```ts
// e2e/checkout.spec.ts
test.describe("checkout", () => {
  test("completes order with card @smoke", async ({ page }) => {
    // ...
  });

  test("applies promo code @regression", async ({ page }) => {
    // ...
  });

  test("handles paypal @slow", async ({ page }) => {
    // ...
  });
});
```

Запуск по тегам:

```bash
# Только smoke на каждый PR
npx playwright test --grep @smoke

# Полная регрессия ночью
npx playwright test --grep-invert @slow
```

---

## Retries и flaky-тесты

Retries — это пластырь, не лечение. Но в CI они необходимы, чтобы отделить случайные падения от системных.

```ts
export default defineConfig({
  retries: process.env.CI ? 1 : 0,
});
```

Если тест падает регулярно:

1. Собери trace (`trace: "on-first-retry"`).
2. Найди причину: race condition, нестабильный селектор, сеть.
3. Исправь тест или код.
4. Не увеличивай retries без причины — это замаскирует проблему.

---

## Запускай E2E против production-сборки

Dev-сервер медленнее production-сборки из-за компиляции на лету.

```ts
// playwright.config.ts
webServer: {
  command: "npm run build && npm run start",
  url: "http://localhost:3000",
  timeout: 120000,
  reuseExistingServer: !process.env.CI,
},
```

Для Next.js: `next build && next start` вместо `next dev`.

---

## Селективный запуск E2E

Не запускай все E2E, если изменился один компонент.

### По изменённым файлам

```bash
# Запускаем только тесты, относящиеся к изменённым файлам
npx playwright test $(git diff --name-only HEAD~1 | grep "e2e/")
```

### По областям приложения

```bash
# PR меняет корзину — запускаем только e2e/cart/
npx playwright test e2e/cart/
```

### По impact analysis

Некоторые команды используют статический анализ зависимостей, чтобы определить, какие E2E затронуты изменениями.

---

## Мониторинг времени тестов

Следи за метриками:

- среднее время E2E-прогона;
- время отдельных тестов (топ самых медленных);
- flaky rate;
- распределение по шардам.

Инструменты:

- Playwright HTML report
- Playwright blob + merge reports
- GitLab CI analytics
- Внешние dashboards: Currents, Playwright Cloud

> 💡 Регулярно просматривай самые медленные тесты. Обычно 10% тестов занимают 50% времени.

---

## Чеклист оптимизации

- [ ] Включён `fullyParallel` и адекватное число `workers`.
- [ ] E2E запускаются в шардах на нескольких CI-раннерах.
- [ ] Аутентификация подготавливается один раз через `storageState`.
- [ ] Тестовые данные создаются через API/seed, а не через UI.
- [ ] Внешние сервисы мокируются или блокируются.
- [ ] Используются стабильные селекторы (роли, label, data-testid).
- [ ] Нет `waitForTimeout` / `sleep`.
- [ ] Тесты размечены тегами (@smoke, @regression, @slow).
- [ ] Retries настроены, но flaky-тесты исправляются, а не маскируются.
- [ ] E2E запускаются против production-сборки.
- [ ] В CI используется селективный запуск по изменениям.
- [ ] Время прогона мониторится и анализируется.
