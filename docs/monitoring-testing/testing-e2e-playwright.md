# E2E-тестирование с Playwright

## Содержание

1. [Что такое Playwright](#что-такое-playwright)
2. [Playwright vs Cypress](#playwright-vs-cypress)
3. [Установка и настройка](#установка-и-настройка)
4. [Базовый API: test, page, expect](#базовый-api-test-page-expect)
5. [Locators — поиск элементов](#locators--поиск-элементов)
6. [Действия: click, fill, type, press](#действия-click-fill-type-press)
7. [Ожидания: waitForSelector, waitForResponse](#ожидания-waitforselector-waitforresponse)
8. [Assertions — проверки](#assertions--проверки)
9. [Тестирование форм](#тестирование-форм)
10. [Тестирование навигации](#тестирование-навигации)
11. [Сетевые запросы: intercept, mock](#сетевые-запросы-intercept-mock)
12. [Визуальное тестирование: toHaveScreenshot](#визуальное-тестирование-tohavescreenshot)
13. [Page Object Model](#page-object-model)
14. [Тестирование аутентификации](#тестирование-аутентификации)
15. [Параллельный запуск и конфигурация](#параллельный-запуск-и-конфигурация)
16. [CI-интеграция](#ci-интеграция)
17. [Трейсы и отладка](#трейсы-и-отладка)
18. [Лучшие практики](#лучшие-практики)
19. [Антипаттерны](#антипаттерны)

---

## Что такое Playwright

**Playwright** — это фреймворк для E2E-тестирования, разработанный Microsoft. Он поддерживает все современные браузеры (Chromium, Firefox, WebKit) и предоставляет единый API для управления ими.

### Ключевые возможности

- **Мультибраузерность** — тесты работают в Chrome, Firefox, Safari
- **Автоожидание** — автоматически ждёт элементов перед действиями
- **Сетевой контроль** — перехват и мокирование запросов
- **Параллельный запуск** — тесты выполняются параллельно в изолированных контекстах
- **Trace Viewer** — запись и воспроизведение тестов с полным контекстом
- **Codegen** — генерация тестов из действий пользователя
- **Визуальные сравнения** — скриншоты и pixel-perfect сравнения
- **Mobile emulation** — тестирование мобильных устройств

---

## Playwright vs Cypress

| Характеристика | Playwright | Cypress |
|---|---|---|
| **Браузеры** | Chromium, Firefox, WebKit | Chromium, Firefox, Edge (ограниченно) |
| **Параллельный запуск** | ✅ Встроенный | ⚠️ Платный (Cypress Dashboard) |
| **Мульти-вкладки** | ✅ Поддерживается | ❌ Ограничено |
| **Мульти-домен** | ✅ Поддерживается | ❌ Не поддерживается |
| **Скорость** | Быстрее (изолированные контексты) | Медленнее (перезапуск между тестами) |
| **Автоожидание** | Встроенное | Встроенное |
| **Debugging** | Trace Viewer, Inspector | Time-travel, Cypress Dashboard |
| **Codegen** | ✅ Встроенный | ❌ Нет |
| **Mobile emulation** | ✅ Встроенный | ⚠️ Ограниченный |
| **Сообщество** | Растущее (Microsoft) | Зрелое (открытый код) |
| **Лицензия** | Apache 2.0 | MIT (бесплатно), Dashboard — платный |

> 💡 **Почему Playwright предпочтительнее в 2026:** Параллельный запуск из коробки, поддержка Safari, мульти-вкладки, мульти-домен, встроенный codegen и trace viewer — всё это бесплатно. Cypress остаётся хорошим выбором для простых проектов, но Playwright выигрывает в масштабируемости.

---

## Установка и настройка

### Базовая установка

```bash
npm install -D @playwright/test
npx playwright install
```

Команда `playwright install` скачивает браузеры (Chromium, Firefox, WebKit).

### Инициализация проекта

```bash
npm init playwright@latest
```

Создаёт:
- `playwright.config.ts` — конфигурация
- `tests/` — директория с тестами
- `tests-examples/` — примеры тестов

### Базовая конфигурация

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",

  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
  },

  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
    {
      name: "firefox",
      use: { ...devices["Desktop Firefox"] },
    },
    {
      name: "webkit",
      use: { ...devices["Desktop Safari"] },
    },
  ],

  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
  },
});
```

### Конфигурация для Next.js

```ts
// playwright.config.ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",

  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },

  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
  ],

  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
});
```

---

## Базовый API: test, page, expect

### Структура теста

```ts
import { test, expect } from "@playwright/test";

test("homepage has correct title", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveTitle(/My App/);
});
```

### test.describe — группировка тестов

```ts
test.describe("Authentication", () => {
  test("login with valid credentials", async ({ page }) => {
    await page.goto("/login");
    await page.fill('[name="email"]', "user@example.com");
    await page.fill('[name="password"]', "password123");
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL("/dashboard");
  });

  test("shows error for invalid credentials", async ({ page }) => {
    await page.goto("/login");
    await page.fill('[name="email"]', "invalid@example.com");
    await page.fill('[name="password"]', "wrong");
    await page.click('button[type="submit"]');
    await expect(page.getByText(/invalid credentials/i)).toBeVisible();
  });
});
```

### test.beforeEach, test.afterEach

```ts
test.describe("User Profile", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/profile");
  });

  test("displays user name", async ({ page }) => {
    await expect(page.getByText("Alice")).toBeVisible();
  });

  test("allows editing email", async ({ page }) => {
    await page.fill('[name="email"]', "new@example.com");
    await page.click('button[type="submit"]');
    await expect(page.getByText(/email updated/i)).toBeVisible();
  });
});
```

---

## Locators — поиск элементов

### getByRole — семантический поиск

```ts
// Кнопки
page.getByRole("button", { name: /submit/i });
page.getByRole("button", { name: "Delete" });

// Заголовки
page.getByRole("heading", { name: /welcome/i });
page.getByRole("heading", { level: 1 });

// Ссылки
page.getByRole("link", { name: /about/i });

// Поля ввода
page.getByRole("textbox", { name: /search/i });
page.getByRole("checkbox", { name: /agree/i });

// Списки
page.getByRole("list");
page.getByRole("listitem");
```

### getByText — поиск по тексту

```ts
// Точный текст
page.getByText("Hello World");

// Регулярное выражение
page.getByText(/hello/i);

// Частичное совпадение
page.getByText("Hello", { exact: false });
```

### getByLabel — поиск по label

```ts
// Связь через htmlFor
page.getByLabel("Email");

// aria-label
page.getByLabel("Search");
```

### getByPlaceholder

```ts
page.getByPlaceholder("Enter your email");
```

### getByTestId — кастомный атрибут

```ts
// В компоненте
<button data-testid="submit-btn">Submit</button>

// В тесте
page.getByTestId("submit-btn");
```

### locator — CSS-селекторы

```ts
// CSS-селектор
page.locator(".btn-primary");
page.locator("#submit-btn");
page.locator("div > button");

// XPath
page.locator("xpath=//button[@type='submit']");

// Текст
page.locator("text=Submit");
```

### Фильтрация и цепочки

```ts
// Фильтрация по тексту
page.getByRole("button").filter({ hasText: /submit/i });

// Фильтрация по наличию дочернего элемента
page.getByRole("listitem").filter({
  has: page.getByText("Alice"),
});

// Цепочки
page.locator("form").getByRole("button");
```

---

## Действия: click, fill, type, press

### click

```ts
// Простой клик
await page.getByRole("button").click();

// Клик с опциями
await page.getByRole("button").click({ force: true }); // Принудительный клик
await page.getByRole("button").click({ position: { x: 10, y: 10 } }); // Координаты
await page.getByRole("button").click({ button: "right" }); // Правый клик
await page.getByRole("button").click({ modifiers: ["Control"] }); // Ctrl+Click
```

### fill — очистка и заполнение

```ts
// Очищает поле и вводит текст
await page.getByLabel("Email").fill("user@example.com");
```

### type — посимвольный ввод

```ts
// Вводит текст посимвольно (как пользователь)
await page.getByLabel("Email").type("user@example.com");

// С задержкой между символами
await page.getByLabel("Email").type("user@example.com", { delay: 100 });
```

### press — нажатие клавиш

```ts
// Нажатие Enter
await page.getByLabel("Search").press("Enter");

// Комбинации клавиш
await page.keyboard.press("Control+A");
await page.keyboard.press("Control+C");
await page.keyboard.press("Control+V");

// Специальные клавиши
await page.keyboard.press("Tab");
await page.keyboard.press("Escape");
await page.keyboard.press("ArrowDown");
```

### check / uncheck — чекбоксы

```ts
// Отметить чекбокс
await page.getByLabel("Agree to terms").check();

// Снять отметку
await page.getByLabel("Agree to terms").uncheck();

// Проверить состояние
await expect(page.getByLabel("Agree to terms")).toBeChecked();
```

### selectOption — выбор из select

```ts
// По значению
await page.getByLabel("Country").selectOption("us");

// По тексту
await page.getByLabel("Country").selectOption({ label: "United States" });

// Несколько значений
await page.getByLabel("Skills").selectOption(["react", "typescript"]);
```

### hover — наведение мыши

```ts
await page.getByRole("button").hover();
```

### dragAndDrop — перетаскивание

```ts
await page.locator("#source").dragTo(page.locator("#target"));
```

### setChecked — универсальный метод для чекбоксов

```ts
// Установить состояние
await page.getByLabel("Agree").setChecked(true);
await page.getByLabel("Agree").setChecked(false);
```

---

## Ожидания: waitForSelector, waitForResponse

### waitForSelector — ожидание элемента

```ts
// Ожидание появления элемента
await page.waitForSelector(".loading", { state: "hidden" });

// Ожидание исчезновения
await page.waitForSelector(".spinner", { state: "detached" });

// Ожидание с таймаутом
await page.waitForSelector(".data", { timeout: 5000 });
```

### waitForResponse — ожидание сетевого ответа

```ts
// Ожидание конкретного запроса
const responsePromise = page.waitForResponse("**/api/users");
await page.click("button.load-users");
const response = await responsePromise;
expect(response.status()).toBe(200);

// Ожидание с предикатом
await page.waitForResponse(
  (response) => response.url().includes("/api/users") && response.status() === 200
);
```

### waitForURL — ожидание навигации

```ts
// Ожидание конкретного URL
await page.waitForURL("/dashboard");

// Ожидание с паттерном
await page.waitForURL("**/dashboard");

// Ожидание с предикатом
await page.waitForURL((url) => url.pathname.includes("/dashboard"));
```

### waitForLoadState — ожидание состояния загрузки

```ts
// Ожидание полной загрузки
await page.waitForLoadState("networkidle");

// Ожидание DOMContentLoaded
await page.waitForLoadState("domcontentloaded");

// Ожидание load
await page.waitForLoadState("load");
```

### Автоматическое ожидание

Playwright автоматически ждёт элементов перед действиями:

```ts
// Не нужно явно ждать — Playwright ждёт автоматически
await page.getByRole("button").click(); // Ждёт, пока кнопка станет видимой и кликабельной
```

---

## Assertions — проверки

### expect — базовые проверки

```ts
// Видимость
await expect(page.getByText("Hello")).toBeVisible();
await expect(page.getByText("Hello")).toBeHidden();

// Наличие в DOM
await expect(page.getByText("Hello")).toBeAttached();
await expect(page.getByText("Hello")).not.toBeAttached();

// Текст
await expect(page.getByRole("heading")).toHaveText("Welcome");
await expect(page.getByRole("heading")).toHaveText(/welcome/i);

// Значение поля
await expect(page.getByLabel("Email")).toHaveValue("user@example.com");
await expect(page.getByLabel("Email")).toHaveValue(/@example\.com/);

// Атрибуты
await expect(page.getByRole("link")).toHaveAttribute("href", "/about");
await expect(page.getByRole("img")).toHaveAttribute("alt", /logo/i);

// CSS-классы
await expect(page.getByRole("button")).toHaveClass(/active/);
await expect(page.getByRole("button")).toHaveClass("btn-primary");

// Стили
await expect(page.getByRole("button")).toHaveCSS("color", "rgb(255, 0, 0)");

// Состояние
await expect(page.getByRole("button")).toBeDisabled();
await expect(page.getByRole("button")).toBeEnabled();
await expect(page.getByRole("checkbox")).toBeChecked();

// Количество элементов
await expect(page.getByRole("listitem")).toHaveCount(5);

// URL страницы
await expect(page).toHaveURL("/dashboard");
await expect(page).toHaveURL(/dashboard/);

// Заголовок страницы
await expect(page).toHaveTitle(/My App/);
```

### Soft assertions — мягкие проверки

```ts
// Мягкие проверки не прерывают тест при падении
await expect.soft(page.getByText("Hello")).toBeVisible();
await expect.soft(page.getByText("World")).toBeVisible();
// Оба проверки выполнятся, даже если первая упадёт
```

### Polling assertions — периодические проверки

```ts
// Проверяет каждые 100мс до таймаута
await expect(async () => {
  const count = await page.getByRole("listitem").count();
  expect(count).toBe(5);
}).toPass({ timeout: 5000 });
```

---

## Тестирование форм

### Базовая форма

```ts
test("submits login form", async ({ page }) => {
  await page.goto("/login");

  await page.getByLabel("Email").fill("user@example.com");
  await page.getByLabel("Password").fill("password123");
  await page.getByRole("button", { name: /login/i }).click();

  await expect(page).toHaveURL("/dashboard");
  await expect(page.getByText(/welcome/i)).toBeVisible();
});
```

### Валидация формы

```ts
test("shows validation error for empty email", async ({ page }) => {
  await page.goto("/register");

  await page.getByRole("button", { name: /register/i }).click();

  await expect(page.getByText(/email is required/i)).toBeVisible();
});

test("shows error for invalid email format", async ({ page }) => {
  await page.goto("/register");

  await page.getByLabel("Email").fill("invalid-email");
  await page.getByRole("button", { name: /register/i }).click();

  await expect(page.getByText(/invalid email format/i)).toBeVisible();
});
```

### Сброс формы

```ts
test("clears form after successful submission", async ({ page }) => {
  await page.goto("/contact");

  await page.getByLabel("Name").fill("Alice");
  await page.getByLabel("Message").fill("Hello");
  await page.getByRole("button", { name: /send/i }).click();

  await expect(page.getByText(/message sent/i)).toBeVisible();
  await expect(page.getByLabel("Name")).toHaveValue("");
  await expect(page.getByLabel("Message")).toHaveValue("");
});
```

### Файловые загрузки

```ts
test("uploads file", async ({ page }) => {
  await page.goto("/upload");

  const fileInput = page.getByLabel("Upload file");
  await fileInput.setInputFiles("path/to/file.pdf");

  await expect(page.getByText("file.pdf")).toBeVisible();

  await page.getByRole("button", { name: /upload/i }).click();

  await expect(page.getByText(/upload successful/i)).toBeVisible();
});
```

---

## Тестирование навигации

### Переход по ссылкам

```ts
test("navigates to about page", async ({ page }) => {
  await page.goto("/");
  await page.getByRole("link", { name: /about/i }).click();

  await expect(page).toHaveURL("/about");
  await expect(page.getByRole("heading", { name: /about us/i })).toBeVisible();
});
```

### Возврат назад

```ts
test("navigates back", async ({ page }) => {
  await page.goto("/products");
  await page.getByRole("link", { name: /product 1/i }).click();
  await page.goBack();

  await expect(page).toHaveURL("/products");
});
```

### Обновление страницы

```ts
test("persists state after reload", async ({ page }) => {
  await page.goto("/cart");
  await page.getByRole("button", { name: /add to cart/i }).click();

  await page.reload();

  await expect(page.getByText("1 item")).toBeVisible();
});
```

---

## Сетевые запросы: intercept, mock

### route — перехват запросов

```ts
test("mocks API response", async ({ page }) => {
  await page.route("**/api/users", (route) => {
    route.fulfill({
      status: 200,
      contentType: "application/json",
      body: JSON.stringify([{ id: 1, name: "Alice" }]),
    });
  });

  await page.goto("/users");

  await expect(page.getByText("Alice")).toBeVisible();
});
```

### Мокирование ошибок

```ts
test("handles API error", async ({ page }) => {
  await page.route("**/api/users", (route) => {
    route.fulfill({
      status: 500,
      body: JSON.stringify({ error: "Server error" }),
    });
  });

  await page.goto("/users");

  await expect(page.getByText(/server error/i)).toBeVisible();
});
```

### Мокирование задержки

```ts
test("shows loading state", async ({ page }) => {
  await page.route("**/api/users", async (route) => {
    await new Promise((resolve) => setTimeout(resolve, 2000));
    route.fulfill({
      status: 200,
      body: JSON.stringify([{ id: 1, name: "Alice" }]),
    });
  });

  await page.goto("/users");

  await expect(page.getByText(/loading/i)).toBeVisible();
  await expect(page.getByText("Alice")).toBeVisible();
});
```

### Аборт запроса

```ts
test("aborts request", async ({ page }) => {
  await page.route("**/api/analytics", (route) => {
    route.abort();
  });

  await page.goto("/");

  // Аналитика не загружается
});
```

### Изменение запроса

```ts
test("modifies request", async ({ page }) => {
  await page.route("**/api/users", (route) => {
    route.continue({
      headers: {
        ...route.request().headers(),
        "X-Custom-Header": "test-value",
      },
    });
  });

  await page.goto("/users");
});
```

---

## Визуальное тестирование: toHaveScreenshot

### Базовые скриншоты

```ts
test("homepage looks correct", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot();
});
```

Playwright создаёт файл `homepage-looks-correct-1-chromium-Darwin.png` в директории `tests/`.

### Скриншоты элементов

```ts
test("button looks correct", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByRole("button", { name: /submit/i })).toHaveScreenshot();
});
```

### Настройки скриншотов

```ts
test("homepage with options", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot({
    maxDiffPixelRatio: 0.01, // Допустимая разница 1%
    threshold: 0.2, // Порог различия пикселей
    animations: "disabled", // Отключить анимации
    mask: [page.getByRole("navigation")], // Замаскировать навигацию
  });
});
```

### Обновление скриншотов

```bash
npx playwright test --update-snapshots
```

### Полностраничные скриншоты

```ts
test("full page screenshot", async ({ page }) => {
  await page.goto("/");
  await page.screenshot({ path: "full-page.png", fullPage: true });
});
```

---

## Page Object Model

### Базовый Page Object

```ts
// pages/LoginPage.ts
import { Page, Locator } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("Email");
    this.passwordInput = page.getByLabel("Password");
    this.submitButton = page.getByRole("button", { name: /login/i });
    this.errorMessage = page.getByText(/invalid credentials/i);
  }

  async goto() {
    await this.page.goto("/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError() {
    await expect(this.errorMessage).toBeVisible();
  }

  async expectRedirectToDashboard() {
    await expect(this.page).toHaveURL("/dashboard");
  }
}
```

### Использование Page Object

```ts
// tests/login.spec.ts
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/LoginPage";

test("login with valid credentials", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("user@example.com", "password123");
  await loginPage.expectRedirectToDashboard();
});

test("shows error for invalid credentials", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("invalid@example.com", "wrong");
  await loginPage.expectError();
});
```

### Page Object с методами-действиями

```ts
// pages/ProductPage.ts
export class ProductPage {
  constructor(private page: Page) {}

  async addToCart(productName: string) {
    const product = this.page.locator(`[data-product="${productName}"]`);
    await product.getByRole("button", { name: /add to cart/i }).click();
  }

  async getCartCount() {
    const count = await this.page.getByTestId("cart-count").textContent();
    return parseInt(count || "0");
  }

  async expectProductInCart(productName: string) {
    await expect(this.page.getByText(productName)).toBeVisible();
  }
}
```

---

## Тестирование аутентификации

### Базовая аутентификация

```ts
test("login and access protected page", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("user@example.com");
  await page.getByLabel("Password").fill("password123");
  await page.getByRole("button", { name: /login/i }).click();

  await page.goto("/profile");
  await expect(page.getByText("Alice")).toBeVisible();
});
```

### Глобальная аутентификация (setup project)

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    {
      name: "setup",
      testMatch: /global-setup\.ts/,
    },
    {
      name: "chromium",
      use: {
        ...devices["Desktop Chrome"],
        storageState: "playwright/.auth/user.json",
      },
      dependencies: ["setup"],
    },
  ],
});
```

```ts
// tests/global-setup.ts
import { test as setup, expect } from "@playwright/test";

const authFile = "playwright/.auth/user.json";

setup("authenticate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("user@example.com");
  await page.getByLabel("Password").fill("password123");
  await page.getByRole("button", { name: /login/i }).click();

  await page.waitForURL("/dashboard");

  await page.context().storageState({ path: authFile });
});
```

### Роли и разные пользователи

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    {
      name: "setup admin",
      testMatch: /admin-setup\.ts/,
    },
    {
      name: "setup user",
      testMatch: /user-setup\.ts/,
    },
    {
      name: "admin tests",
      use: {
        storageState: "playwright/.auth/admin.json",
      },
      dependencies: ["setup admin"],
    },
    {
      name: "user tests",
      use: {
        storageState: "playwright/.auth/user.json",
      },
      dependencies: ["setup user"],
    },
  ],
});
```

---

## Параллельный запуск и конфигурация

### Параллельный запуск тестов

```ts
// playwright.config.ts
export default defineConfig({
  fullyParallel: true, // Все тесты параллельно
  workers: 4, // 4 параллельных воркера
});
```

### Последовательный запуск

```ts
test.describe.configure({ mode: "serial" });

test("first test", async ({ page }) => { ... });
test("second test", async ({ page }) => { ... }); // Ждёт первого
```

### Retry — повторные запуски

```ts
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0, // 2 повтора в CI, 0 локально
});
```

### Timeout — таймауты

```ts
// playwright.config.ts
export default defineConfig({
  timeout: 30000, // Таймаут теста 30 секунд
  expect: {
    timeout: 5000, // Таймаут ожиданий 5 секунд
  },
});
```

### Конфигурация для разных окружений

```ts
// playwright.config.ts
export default defineConfig({
  use: {
    baseURL: process.env.CI ? "https://staging.example.com" : "http://localhost:3000",
  },
});
```

---

## CI-интеграция

### GitHub Actions

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests
on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: npx playwright test

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

### GitLab CI

```yaml
# .gitlab-ci.yml
playwright:
  image: mcr.microsoft.com/playwright:v1.40.0-jammy
  stage: test
  script:
    - npm ci
    - npx playwright test
  artifacts:
    when: always
    paths:
      - playwright-report/
    expire_in: 30 days
```

### Docker

```dockerfile
FROM mcr.microsoft.com/playwright:v1.40.0-jammy

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npx", "playwright", "test"]
```

---

## Трейсы и отладка

### Trace Viewer

```ts
// playwright.config.ts
export default defineConfig({
  use: {
    trace: "on-first-retry", // Запись трейса при первом повторе
  },
});
```

```bash
npx playwright show-report
```

Открывает HTML-отчёт с трейсами, скриншотами и логами.

### Inspector — интерактивная отладка

```bash
npx playwright test --debug
```

Открывает Inspector для пошагового выполнения тестов.

### Codegen — генерация тестов

```bash
npx playwright codegen https://example.com
```

Открывает браузер, записывает действия и генерирует код теста.

### UI Mode — интерактивный режим

```bash
npx playwright test --ui
```

Открывает UI для запуска, отладки и анализа тестов.

---

## Лучшие практики

### 1. Используйте семантические селекторы

```ts
// ❌ CSS-селектор — хрупкий
await page.locator("div > button.btn-primary").click();

// ✅ Семантический селектор
await page.getByRole("button", { name: /submit/i }).click();
```

### 2. Используйте data-testid как последний резерв

```ts
// ✅ Семантический селектор
await page.getByRole("button", { name: /submit/i }).click();

// ⚠️ data-testid — только если нет семантики
await page.getByTestId("submit-btn").click();
```

### 3. Используйте Page Object Model

```ts
// ❌ Логика в тестах
test("login", async ({ page }) => {
  await page.goto("/login");
  await page.fill('[name="email"]', "user@example.com");
  await page.fill('[name="password"]', "password123");
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL("/dashboard");
});

// ✅ Page Object
test("login", async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login("user@example.com", "password123");
  await loginPage.expectRedirectToDashboard();
});
```

### 4. Используйте глобальную аутентификацию

```ts
// ❌ Логин в каждом тесте
test("test 1", async ({ page }) => {
  await page.goto("/login");
  await page.fill('[name="email"]', "user@example.com");
  await page.click('button[type="submit"]');
  // ... тест
});

// ✅ Глобальная аутентификация
test("test 1", async ({ page }) => {
  // Уже аутентифицирован через storageState
  await page.goto("/dashboard");
  // ... тест
});
```

### 5. Используйте expect с auto-waiting

```ts
// ❌ Явное ожидание
await page.waitForSelector(".success");
expect(await page.isVisible(".success")).toBe(true);

// ✅ Auto-waiting
await expect(page.getByText(/success/i)).toBeVisible();
```

### 6. Изолируйте тесты

```ts
// ❌ Тесты зависят друг от друга
test("creates user", async ({ page }) => { ... });
test("edits user", async ({ page }) => { ... }); // Зависит от первого

// ✅ Каждый тест независим
test("creates user", async ({ page }) => { ... });
test("edits user", async ({ page }) => {
  // Создаёт пользователя самостоятельно
  await page.goto("/users/new");
  await page.getByLabel("Name").fill("Alice");
  await page.getByRole("button", { name: /save/i }).click();
  // ... редактирование
});
```

### 7. Используйте мокирование для стабильности

```ts
// ❌ Зависит от реального API
test("loads users", async ({ page }) => {
  await page.goto("/users");
  await expect(page.getByText("Alice")).toBeVisible();
});

// ✅ Мокирование API
test("loads users", async ({ page }) => {
  await page.route("**/api/users", (route) => {
    route.fulfill({
      body: JSON.stringify([{ id: 1, name: "Alice" }]),
    });
  });
  await page.goto("/users");
  await expect(page.getByText("Alice")).toBeVisible();
});
```

---

## Антипаттерны

### 1. Использование CSS-селекторов

```ts
// ❌ Хрупкий селектор
await page.locator("div.container > button.btn").click();

// ✅ Семантический селектор
await page.getByRole("button", { name: /submit/i }).click();
```

### 2. Явные ожидания (waitForTimeout)

```ts
// ❌ Жёсткая задержка
await page.waitForTimeout(2000);
await expect(page.getByText("Loaded")).toBeVisible();

// ✅ Auto-waiting
await expect(page.getByText("Loaded")).toBeVisible();
```

### 3. Зависимость тестов друг от друга

```ts
// ❌ Второй тест зависит от первого
test("creates user", async ({ page }) => { ... });
test("deletes user", async ({ page }) => { ... }); // Зависит от первого

// ✅ Каждый тест независим
test("deletes user", async ({ page }) => {
  // Создаёт пользователя самостоятельно
  await page.goto("/users/new");
  await page.getByLabel("Name").fill("Alice");
  await page.getByRole("button", { name: /save/i }).click();
  // ... удаление
});
```

### 4. Тестирование всего приложения через E2E

```ts
// ❌ E2E для каждой мелочи
test("button has correct color", async ({ page }) => { ... });
test("text is centered", async ({ page }) => { ... });

// ✅ E2E только для критических сценариев
test("user can complete checkout", async ({ page }) => { ... });
```

### 5. Игнорирование параллельного запуска

```ts
// ❌ Последовательный запуск
test.describe.configure({ mode: "serial" });

// ✅ Параллельный запуск
export default defineConfig({
  fullyParallel: true,
});
```

### 6. Отсутствие повторных запусков в CI

```ts
// ❌ Нет retries
export default defineConfig({
  retries: 0,
});

// ✅ Retries в CI
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
});
```
