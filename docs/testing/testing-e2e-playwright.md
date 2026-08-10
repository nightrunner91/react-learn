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

Правильная настройка Playwright определяет, насколько стабильными и быстрыми будут тесты. Конфигурационный файл `playwright.config.ts` — это центр управления: он задаёт браузеры, таймауты, стратегию параллельного запуска и поведение в CI. Потраченное на настройку время окупается стабильностью прогона и удобством отладки.

### Базовая установка

Команды ниже устанавливают пакет и скачивают бинарники браузеров. Бинарники хранятся отдельно от `node_modules`, потому что весят сотни мегабайт и не должны переустанавливаться при каждом `npm install`.

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

Конфигурация определяет поведение тестов на всех уровнях. Ключевые параметры: `fullyParallel` включает параллельное выполнение — без него тесты идут последовательно даже на мощных машинах; `retries` страхует от flaky-тестов в CI; `webServer` автоматически запускает dev-сервер перед прогоном и ждёт его готовности. Параметр `trace: "on-first-retry"` записывает трейс только при повторе — это экономит диск, но даёт полную информацию для отладки упавших тестов.

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

Next.js-проекты отличаются тем, что dev-сервер стартует дольше (компиляция страниц, middleware). Увеличенный `webServer.timeout` предотвращает ложные падения из-за долгого старта. Опция `screenshot: "only-on-failure"` сохраняет скриншоты только для упавших тестов — это помогает при отладке без засорения диска. Для Next.js обычно достаточно одного браузера (Chromium) на этапе разработки, а кроссбраузерные проверки переносят в CI.

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

В основе Playwright лежит три объекта: `test` — описывает тестовый сценарий, `page` — представляет вкладку браузера и предоставляет методы взаимодействия, `expect` — проверяет состояния с автоматическим ожиданием. Каждый тест получает свежий `page` через fixture — это гарантирует изоляцию: тесты не делят состояние, cookies или localStorage. Понимание этого механизма критично: если тесты «протекают» друг в друга, причина почти всегда в неправильном использовании fixtures.

### Структура теста

Функция теста принимает деструктурированный объект `{ page }` — это и есть fixture. Playwright создаёт новый браузерный контекст для каждого теста, поэтому cookies и кеш не переносятся между тестами автоматически.

```ts
import { test, expect } from "@playwright/test";

test("homepage has correct title", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveTitle(/My App/);
});
```

### test.describe — группировка тестов

`test.describe` группирует связанные тесты в логические блоки. Группа — это не только организационная единица: hooks (`beforeEach`, `afterEach`), определённые внутри `describe`, применяются только к тестам этой группы. Вложенные `describe` наследуют hooks от родительских блоков. Это позволяет выстраивать иерархию подготовки: например, внешний `describe` аутентифицирует пользователя, а внутренний — настраивает данные для конкретного сценария.

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

Hooks выполняют код до и после каждого теста. `beforeEach` — место для повторяющейся подготовки: навигация на страницу, заполнение форм, мокирование API. Это устраняет дублирование и делает тесты читаемыми. Важно: hooks выполняются для каждого теста в группе, поэтому тяжёлая подготовка (создание данных через API) может замедлить прогон — в таких случаях лучше использовать глобальную настройку через `storageState` или `test.beforeAll`.

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

Locator — это обещание найти элемент в будущем. Playwright не ищет элемент в момент создания locator'а — он откладывает поиск до момента действия (click, fill, expect). Это называется «ленивым вычислением» и позволяет писать код без явных ожиданий: `await page.getByRole("button").click()` сначала подождёт, пока кнопка появится и станет кликабельной, и только потом кликнет.

Стратегия выбора locator'а влияет на стабильность тестов. Семантические locator'ы (`getByRole`, `getByLabel`) привязаны к доступности — они ломаются только при реальных изменениях UI. CSS-селекторы ломаются при любом рефакторинге вёрстки, даже если визуально ничего не изменилось. Правило: начинайте с семантических locator'ов, `data-testid` — как последний резерв.

### getByRole — семантический поиск

`getByRole` — самый надёжный locator. Он опирается на ARIA-роли, которые браузер вычисляет из HTML-разметки. Если кнопка имеет `role="button"` (явно или неявно через тег `<button>`), `getByRole("button")` найдёт её независимо от классов и структуры DOM. Параметр `name` фильтрует по доступному имени — тексту, который озвучит скринридер. Это делает тесты устойчивыми к рефакторингу и одновременно проверяет доступность.

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

`getByText` ищет элементы по видимому тексту. Он полезен для проверки, что конкретный текст отображается на странице, но менее надёжен, чем `getByRole` — текст может измениться при локализации или копирайтинге. Используйте его для проверки содержимого, а не для взаимодействия с элементами.

```ts
// Точный текст
page.getByText("Hello World");

// Регулярное выражение
page.getByText(/hello/i);

// Частичное совпадение
page.getByText("Hello", { exact: false });
```

### getByLabel — поиск по label

`getByLabel` находит поля формы по тексту их `<label>`. Это идеальный locator для input-элементов: он привязан к семантике формы и одновременно проверяет, что у поля есть доступная подпись. Если label отсутствует или не связан с полем через `htmlFor`, `getByLabel` не найдёт элемент — это сигнал о проблеме с доступностью.

```ts
// Связь через htmlFor
page.getByLabel("Email");

// aria-label
page.getByLabel("Search");
```

### getByPlaceholder

`getByPlaceholder` находит поля по атрибуту `placeholder`. Это менее надёжный locator, чем `getByLabel` — placeholder часто меняется при локализации и не заменяет label с точки зрения доступности. Используйте его только если у поля нет label или placeholder — единственный стабильный идентификатор.

```ts
page.getByPlaceholder("Enter your email");
```

### getByTestId — кастомный атрибут

`data-testid` — это явный маркер для тестов. Он не влияет на визуальное отображение и доступность, но создаёт контракта между компонентом и тестом. Используйте его, когда нет семантических locator'ов: сложные составные компоненты, кастомные виджеты, элементы без текста и ролей. Компромисс: `data-testid` требует дисциплины — если разработчики забывают добавлять его, тесты становятся хрупкими.

```ts
// В компоненте
<button data-testid="submit-btn">Submit</button>

// В тесте
page.getByTestId("submit-btn");
```

### locator — CSS-селекторы

CSS-селекторы — самый хрупкий способ поиска элементов. Они ломаются при любом изменении структуры DOM, классов или имён. Используйте их только для сложных случаев, когда семантические locator'ы не работают: например, выбор конкретного дочернего элемента в списке или стилизованного компонента без доступной семантики. XPath поддерживает ещё более сложные запросы, но ещё менее читаем.

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

Фильтрация позволяет уточнить locator: `filter({ hasText })` оставляет только элементы с определённым текстом, `filter({ has })` — только элементы, содержащие дочерний locator. Цепочки (`page.locator("form").getByRole("button")`) сужают область поиска до конкретного контейнера. Это мощнее CSS-селекторов, потому что каждый шаг проверяется независимо и даёт понятные ошибки при падении.

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

Действия в Playwright — это не просто команды, а операции с встроенными ожиданиями. Перед каждым действием Playwright проверяет, что элемент видим, стабилен (не анимируется), не перекрыт другим элементом и включён. Это устраняет класс ошибок «element is not clickable», но требует понимания: если действие зависает, значит элемент не проходит одну из этих проверок.

### click

`click` — базовое действие для взаимодействия с кнопками, ссылками и другими кликабельными элементами. Опция `force: true` отключает проверки видимости и перекрытия — используйте её только для отладки, в продакшн-тестах это маскирует реальные проблемы. `position` позволяет кликнуть в конкретную точку элемента — полезно для canvas или кастомных виджетов.

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

`fill` очищает поле и вводит новый текст за одну операцию. Это предпочтительный способ заполнения форм: он быстрый и предсказуемый. `fill` не триггерит посимвольные события `input`, поэтому если компонент реагирует на каждый символ (например, валидация в реальном времени), используйте `type`.

```ts
// Очищает поле и вводит текст
await page.getByLabel("Email").fill("user@example.com");
```

### type — посимвольный ввод

`type` имитирует реальный пользовательский ввод: каждое символ генерирует события `keydown`, `keypress`, `input`, `keyup`. Это необходимо для тестирования компонентов с автодополнением, масками ввода или валидацией на лету. Компромисс: `type` медленнее `fill`, поэтому используйте его только там, где посимвольное поведение критично.

```ts
// Вводит текст посимвольно (как пользователь)
await page.getByLabel("Email").type("user@example.com");

// С задержкой между символами
await page.getByLabel("Email").type("user@example.com", { delay: 100 });
```

### press — нажатие клавиш

`press` нажимает клавишу или комбинацию клавиш. Он полезен для тестирования клавиатурной навигации, горячих клавиш и отправки форм через Enter. `page.keyboard.press` работает на уровне страницы, а не элемента — это позволяет тестировать глобальные шорткаты.

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

`check` и `uncheck` устанавливают состояние чекбокса или радиокнопки. В отличие от `click`, они проверяют текущее состояние и не выполняют действие, если элемент уже в нужном состоянии. Это делает тесты более устойчивыми: не нужно проверять состояние перед изменением.

```ts
// Отметить чекбокс
await page.getByLabel("Agree to terms").check();

// Снять отметку
await page.getByLabel("Agree to terms").uncheck();

// Проверить состояние
await expect(page.getByLabel("Agree to terms")).toBeChecked();
```

### selectOption — выбор из select

`selectOption` выбирает опцию в `<select>`. Он принимает значение (`value`), текст (`label`) или массив для мульти-селекта. Это специализированный метод для нативных select-элементов — для кастомных dropdown-компонентов используйте `click` на триггере и `click` на опции.

```ts
// По значению
await page.getByLabel("Country").selectOption("us");

// По тексту
await page.getByLabel("Country").selectOption({ label: "United States" });

// Несколько значений
await page.getByLabel("Skills").selectOption(["react", "typescript"]);
```

### hover — наведение мыши

`hover` перемещает курсор на элемент, триггеря события `mouseenter` и `mouseover`. Это необходимо для тестирования tooltip'ов, dropdown-меню и других hover-эффектов. В headless-режиме hover может работать иначе, чем в headed — если тест падает только в CI, проверьте, не зависит ли он от реального курсора мыши.

```ts
await page.getByRole("button").hover();
```

### dragAndDrop — перетаскивание

`dragTo` имитирует drag-and-drop: зажимает элемент, перемещает в целевую позицию и отпускает. Это работает для нативного HTML5 drag-and-drop и для кастомных реализаций на mouse events. Компромисс: drag-and-drop тесты часто flaky из-за гонок между анимацией и действиями — добавляйте явные ожидания состояния после перетаскивания.

```ts
await page.locator("#source").dragTo(page.locator("#target"));
```

### setChecked — универсальный метод для чекбоксов

`setChecked` — альтернатива `check`/`uncheck`, принимающая булево значение. Это удобно, когда состояние определяется переменной: `await element.setChecked(isEnabled)` читается лучше, чем условный `if/else` с `check` и `uncheck`.

```ts
// Установить состояние
await page.getByLabel("Agree").setChecked(true);
await page.getByLabel("Agree").setChecked(false);
```

---

## Ожидания: waitForSelector, waitForResponse

Ожидания — самая частая причина flaky-тестов. Playwright решает эту проблему автоожиданием: действия (`click`, `fill`) автоматически ждут, пока элемент станет готов. Но для сложных сценариев — ожидание сетевого ответа, навигации или скрытия элемента — нужны явные `waitFor*` методы. Ключевой принцип: никогда не используйте фиксированные задержки (`waitForTimeout`), всегда ждите конкретного состояния.

### waitForSelector — ожидание элемента

`waitForSelector` ждёт изменения состояния элемента в DOM. Параметры `state`: `"visible"` (по умолчанию) — элемент есть в DOM и видим, `"hidden"` — элемент есть, но невидим, `"attached"` — элемент есть в DOM, `"detached"` — элемент удалён из DOM. Используйте `"hidden"` для ожидания завершения загрузки, `"detached"` для ожидания удаления элемента.

```ts
// Ожидание появления элемента
await page.waitForSelector(".loading", { state: "hidden" });

// Ожидание исчезновения
await page.waitForSelector(".spinner", { state: "detached" });

// Ожидание с таймаутом
await page.waitForSelector(".data", { timeout: 5000 });
```

### waitForResponse — ожидание сетевого ответа

`waitForResponse` ждёт конкретного HTTP-ответа. Это критично для тестирования асинхронных операций: отправка формы, загрузка данных, удаление. Паттерн: создаёте promise до действия, выполняете действие, ждёте promise. Это надёжнее, чем ждать появления текста, потому что проверяет именно сетевой результат, а не побочный эффект рендера.

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

`waitForURL` ждёт изменения URL страницы. Это полезно после клика по ссылке или отправки формы, когда нужно убедиться, что навигация произошла. Предикат-функция позволяет проверять сложные условия: например, что URL содержит определённые query-параметры.

```ts
// Ожидание конкретного URL
await page.waitForURL("/dashboard");

// Ожидание с паттерном
await page.waitForURL("**/dashboard");

// Ожидание с предикатом
await page.waitForURL((url) => url.pathname.includes("/dashboard"));
```

### waitForLoadState — ожидание состояния загрузки

`waitForLoadState` ждёт одного из трёх состояний: `"load"` — событие `load` (все ресурсы загружены), `"domcontentloaded"` — DOM построен, `"networkidle"` — нет активных сетевых запросов в течение 500мс. `"networkidle"` полезен для SPA, где данные загружаются асинхронно, но может замедлить тесты, если на странице есть постоянные фоновые запросы (аналитика, websockets).

```ts
// Ожидание полной загрузки
await page.waitForLoadState("networkidle");

// Ожидание DOMContentLoaded
await page.waitForLoadState("domcontentloaded");

// Ожидание load
await page.waitForLoadState("load");
```

### Автоматическое ожидание

Playwright автоматически ждёт элементов перед действиями: проверяет, что элемент присоединён к DOM, видим, стабилен (не анимируется), не перекрыт другим элементом, и включён. Это означает, что явные `waitForSelector` нужны редко — только для сложных сценариев вроде ожидания сетевого ответа или скрытия элемента. Если действие зависает, значит элемент не проходит одну из проверок автоожидания — ищите причину в UI, а не в тесте.

```ts
// Не нужно явно ждать — Playwright ждёт автоматически
await page.getByRole("button").click(); // Ждёт, пока кнопка станет видимой и кликабельной
```

---

## Assertions — проверки

Assertions в Playwright — это не просто проверки, а retry-циклы. `await expect(locator).toBeVisible()` не падает мгновенно, если элемент невидим: оно повторяет проверку каждые 100мс до таймаута (по умолчанию 5 секунд). Это называется «auto-retrying assertions» и устраняет необходимость явных ожиданий перед проверками. Компромисс: если тест падает с таймаутом assertion, это не значит, что элемент не появился — это значит, что он не появился за 5 секунд. Увеличение таймаута маскирует проблему, а не решает её.

### expect — базовые проверки

Большинство assertions проверяют состояние элемента: видимость, текст, значение, атрибуты, стили. Все они auto-retrying. `toBeVisible` проверяет, что элемент есть в DOM и видим (не `display: none`, не `visibility: hidden`, не нулевых размеров). `toBeAttached` проверяет только наличие в DOM, независимо от видимости. `toHaveText` принимает строку или regex — regex полезен для частичных совпадений.

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

Обычные assertions прерывают тест при первом падении. Soft assertions (`expect.soft`) продолжают выполнение, собирая все ошибки. Это полезно, когда нужно проверить несколько независимых аспектов UI за один проход — например, что все элементы формы валидны. Но soft assertions усложняют отладку: тест может пройти через множество невалидных состояний перед падением. Используйте их осознанно, не для всех проверок.

```ts
// Мягкие проверки не прерывают тест при падении
await expect.soft(page.getByText("Hello")).toBeVisible();
await expect.soft(page.getByText("World")).toBeVisible();
// Оба проверки выполнятся, даже если первая упадёт
```

### Polling assertions — периодические проверки

`toPass` выполняет assertion-функцию повторно до тех пор, пока она не пройдёт или не истечёт таймаут. Это полезно для проверки условий, которые не выражаются через locator: количество элементов, сложные вычисления, состояния нескольких элементов. Компромисс: polling assertions сложнее отлаживать, потому что ошибка может быть не в условии, а в том, что функция бросает исключение до `expect`.

```ts
// Проверяет каждые 100мс до таймаута
await expect(async () => {
  const count = await page.getByRole("listitem").count();
  expect(count).toBe(5);
}).toPass({ timeout: 5000 });
```

---

## Тестирование форм

Формы — критическая точка E2E-тестов, потому что они объединяют взаимодействие пользователя, валидацию и сетевые запросы. Стратегия: тестируйте формы как пользователь, а не как разработчик. Заполняйте поля через `fill`, отправляйте через кнопку (не через `page.keyboard.press("Enter")` — это не всегда работает), проверяйте результат через URL или появление текста. Для валидации тестируйте не только успешные сценарии, но и граничные случаи: пустые поля, невалидный формат, максимальная длина.

### Базовая форма

Типичный сценарий: навигация на страницу, заполнение полей, отправка, проверка результата. Результат проверяйте через навигацию (`toHaveURL`) или появление текста подтверждения — это надёжнее, чем проверять состояние формы.

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

Валидация может быть клиентской (HTML5 `required`, `pattern`) или серверной (ответ API с ошибками). Клиентская валидация проверяется мгновенно, серверная требует ожидания сетевого ответа. Для серверной валидации используйте `waitForResponse` или ждите появления текста ошибки — не используйте фиксированные задержки.

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

После успешной отправки формы поля обычно очищаются. Проверка этого поведения важна: если форма не сбрасывается, пользователь может случайно отправить данные дважды. Проверяйте значение полей через `toHaveValue("")` после отправки.

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

`setInputFiles` устанавливает файлы для `<input type="file">`. Playwright не открывает реальный file picker — он напрямую устанавливает файлы в input. Это быстрее и надёжнее, но не тестирует UX file picker'а. Для тестирования drag-and-drop загрузок используйте `page.locator("#dropzone").dispatchEvent("drop", { dataTransfer })`.

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

Навигация — это переход между страницами или состояниями приложения. В SPA навигация часто виртуальная (меняется URL без перезагрузки), в MPA — реальная (перезагрузка страницы). Playwright работает одинаково с обоими случаями, но важно понимать разницу: в SPA после клика по ссылке DOM не пересоздаётся, в MPA — пересоздаётся полностью. Это влияет на то, как быстро появляются элементы после навигации.

### Переход по ссылкам

После клика по ссылке проверяйте не только URL, но и содержимое новой страницы. URL может измениться до того, как контент загрузится, поэтому `toHaveURL` без проверки контента даёт ложную уверенность.

```ts
test("navigates to about page", async ({ page }) => {
  await page.goto("/");
  await page.getByRole("link", { name: /about/i }).click();

  await expect(page).toHaveURL("/about");
  await expect(page.getByRole("heading", { name: /about us/i })).toBeVisible();
});
```

### Возврат назад

`goBack` и `goForward` имитируют кнопки браузера. Это полезно для тестирования, что состояние сохраняется при навигации: например, фильтры на странице списка должны восстановиться после возврата. В SPA `goBack` может не работать, если роутер не интегрирован с history API — в таких случаях используйте ссылку «Назад» в UI.

```ts
test("navigates back", async ({ page }) => {
  await page.goto("/products");
  await page.getByRole("link", { name: /product 1/i }).click();
  await page.goBack();

  await expect(page).toHaveURL("/products");
});
```

### Обновление страницы

`reload` перезагружает страницу. Это тестирует персистентность состояния: данные в localStorage, IndexedDB или URL должны восстановиться. Если состояние хранится только в памяти (React state, Redux), оно потеряется при перезагрузке — и тест это обнаружит.

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

Мокирование сетевых запросов — один из самых мощных и одновременно опасных инструментов E2E-тестирования. С одной стороны, моки делают тесты быстрыми, стабильными и независимыми от бэкенда. С другой — они создают «параллельную реальность», где тест проходит, но реальное приложение не работает. Стратегия: мокайте только то, что не можете контролировать (сторонние API, платёжные системы), и используйте реальные endpoints для собственного бэкенда. Если мокаете всё — вы тестируете моки, а не приложение.

### route — перехват запросов

`page.route` перехватывает сетевые запросы и позволяет изменить ответ. Это основа мокирования: вы контролируете, что получит приложение от сервера. Route устанавливается до навигации или действия, которое триггерит запрос. Важно: route применяется ко всем запросам,_matching_ паттерн, поэтому будьте конкретны в URL.

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

Тестирование обработки ошибок критично, но сложно в реальных условиях. Мокирование позволяет симулировать любой HTTP-статус: 400 (bad request), 401 (unauthorized), 403 (forbidden), 404 (not found), 500 (server error). Это проверяет, что приложение показывает понятные сообщения пользователю и не оставляет его в неопределённом состоянии.

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

Реальные API отвечают с задержкой от 50мс до нескольких секунд. Тесты без задержек не проверяют loading-состояния и могут маскировать race conditions. Добавление искусственной задержки в мок проверяет, что приложение показывает спиннеры, скелетоны или прогресс-бары, и корректно обрабатывает медленные ответы.

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

`route.abort()` симулирует сетевую ошибку: запрос не достигает сервера. Это тестирует обработку offline-режима, таймаутов и сетевых сбоев. Полезно для проверки, что приложение не падает при потере соединения и показывает понятные сообщения.

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

`route.continue()` модифицирует запрос перед отправкой: добавляет заголовки, меняет метод или тело. Это полезно для тестирования аутентификации (добавление токена), A/B-тестирования (передача вариантов) и проверки, что приложение отправляет корректные заголовки. В отличие от `fulfill`, `continue` позволяет запросу достичь реального сервера.

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

Визуальное тестирование сравнивает скриншоты pixel-by-pixel и находит различия, невидимые глазу. Это мощный инструмент для регрессии UI, но он же — источник ложных срабатываний. Любое изменение рендеринга (шрифт, антиалиасинг, субпиксельный рендеринг) ломает скриншот. Стратегия: используйте визуальные тесты для стабильных компонентов (кнопки, иконки, layout), но не для динамического контента (даты, аватары, тексты). Маскируйте изменяемые области через `mask`.

### Базовые скриншоты

`toHaveScreenshot()` делает скриншот страницы и сравнивает с эталоном. При первом запуске скриншот сохраняется как baseline, при последующих — сравнивается с ним. Разница хранится в директории `test-results`.

```ts
test("homepage looks correct", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot();
});
```

Playwright создаёт файл `homepage-looks-correct-1-chromium-Darwin.png` в директории `tests/`.

### Скриншоты элементов

Скриншот конкретного элемента точнее, чем всей страницы: он не ломается при изменении других частей UI. Используйте его для компонентов, которые должны выглядеть одинаково независимо от контекста: кнопки, карточки, модальные окна.

```ts
test("button looks correct", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByRole("button", { name: /submit/i })).toHaveScreenshot();
});
```

### Настройки скриншотов

`maxDiffPixelRatio` задаёт допустимый процент различий — 0.01 означает, что 1% пикселей могут отличаться без падения теста. Это критично для кроссбраузерного тестирования: рендеринг в Chrome и Safari отличается на субпиксельном уровне. `threshold` определяет чувствительность к различиям цвета. `mask` скрывает элементы (навигация, футер, даты), которые меняются между прогонами. `animations: "disabled"` отключает анимации, чтобы они не влияли на скриншот.

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

Команда `--update-snapshots` перезаписывает baseline-скриншоты. Используйте её только после визуальной проверки: если тест упал из-за реального бага, обновление скриншота закрепит баг как эталон. Всегда проверяйте diff перед обновлением.

```bash
npx playwright test --update-snapshots
```

### Полностраничные скриншоты

`page.screenshot({ fullPage: true })` делает скриншот всей страницы, включая прокручиваемую область. Это полезно для документации или ручного сравнения, но не для автоматических тестов: полностраничные скриншоты слишком чувствительны к изменениям контента. Используйте их для лендингов с фиксированным контентом.

```ts
test("full page screenshot", async ({ page }) => {
  await page.goto("/");
  await page.screenshot({ path: "full-page.png", fullPage: true });
});
```

---

## Page Object Model

Page Object Model (POM) — паттерн организации тестов, где каждая страница представлена классом с locator'ами и методами. Это решает три проблемы: дублирование (один login-сценарий используется в десятках тестов), читаемость (тест описывает бизнес-логику, а не CSS-селекторы) и поддерживаемость (изменение UI требует правки только в одном месте). Компромисс: POM добавляет абстракцию — для маленьких проектов (5-10 тестов) он может быть избыточен. Применяйте POM, когда тестов больше 20 или когда несколько тестов используют одну страницу.

### Базовый Page Object

Page Object инкапсулирует locator'ы как свойства и действия как методы. Конструктор принимает `page` — это позволяет создавать объект для каждого теста с чистым контекстом. Методы возвращают `Promise<void>` или `Promise<Locator>`, чтобы тесты могли цепочить вызовы.

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

Тесты с Page Object читаются как спецификация: `loginPage.goto()`, `loginPage.login()`, `loginPage.expectRedirectToDashboard()`. Это делает тесты понятными для не-разработчиков (QA, продукт-менеджеров) и устойчивыми к изменениям UI — если структура HTML изменится, правится только Page Object, а не каждый тест.

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

Page Object может не только выполнять действия, но и возвращать данные (`getCartCount`) или проверять состояния (`expectProductInCart`). Это создаёт единый интерфейс для работы со страницей: тест не знает о внутренней структуре DOM, он оперирует бизнес-понятиями. Методы-проверки внутри Page Object используют `expect` — это централизует логику проверок и делает её переиспользуемой.

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

Аутентификация — критический сценарий, но логиниться в каждом тесте неэффективно: это замедляет прогон и создаёт точки отказа. Playwright решает эту проблему через `storageState` — сохранение cookies и localStorage между тестами. Стратегия: один setup-тест логинится и сохраняет состояние, остальные тесты используют его. Это сокращает время прогона в 3-5 раз для проектов с множеством защищённых страниц.

### Базовая аутентификация

Простейший подход: логин в каждом тесте. Это работает для малых проектов, но не масштабируется. Если у вас 100 тестов, защищённых аутентификацией, и логин занимает 2 секунды, вы теряете 200 секунд только на аутентификацию.

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

Setup project выполняется один раз перед всеми тестами. Он логинится, сохраняет cookies и localStorage в JSON-файл, остальные тесты загружают этот файл через `storageState`. Это создаёт «аутентифицированный контекст» для всех тестов. Важно: setup project должен быть в `dependencies` у основных проектов, чтобы Playwright выполнил его первым. Файл `playwright/.auth/user.json` добавьте в `.gitignore` — он содержит сессионные данные.

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

Разные роли (admin, user, moderator) требуют разных setup project'ов. Каждый project логинится под своим пользователем и сохраняет состояние в отдельный файл. Тесты группируются по project'ам через `dependencies` — admin-тесты используют admin-state, user-тесты используют user-state. Это изолирует роли и предотвращает случайное использование admin-прав в user-тестах.

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

Параллельный запуск — одно из ключевых преимуществ Playwright. Каждый тест выполняется в изолированном браузерном контексте (отдельные cookies, localStorage), поэтому тесты не влияют друг на друга. Это позволяет безопасно запускать тесты параллельно на одном компьютере или распределённо в CI. Компромисс: параллельный запуск требует, чтобы тесты были независимы — если тесты делят состояние (база данных, файлы), параллелизация приведёт к flaky-результатам.

### Параллельный запуск тестов

`fullyParallel: true` включает параллельное выполнение всех тестов. `workers` задаёт количество параллельных процессов — по умолчанию Playwright использует половину ядер CPU. В CI лучше явно задать `workers: 1` для стабильности или увеличить, если runner мощный.

```ts
// playwright.config.ts
export default defineConfig({
  fullyParallel: true, // Все тесты параллельно
  workers: 4, // 4 параллельных воркера
});
```

### Последовательный запуск

`test.describe.configure({ mode: "serial" })` заставляет тесты в группе выполняться последовательно. Это необходимо только если тесты делят состояние и не могут быть изолированы. Компромисс: serial-тесты медленнее и хрупче — если первый тест падает, остальные не выполнятся. Избегайте serial-режима, создавая данные в каждом тесте независимо.

```ts
test.describe.configure({ mode: "serial" });

test("first test", async ({ page }) => { ... });
test("second test", async ({ page }) => { ... }); // Ждёт первого
```

### Retry — повторные запуски

`retries` задаёт количество повторных запусков при падении. В CI рекомендуется `retries: 2` — это маскирует flaky-тесты (временные проблемы с сетью, рендерингом), но не скрывает реальные баги. Локально `retries: 0` — если тест падает, вы хотите узнать об этом сразу. Важно: если тест падает при retries: 2, но проходит при retries: 3, это не стабильный тест — это flaky-тест, который нужно исправить.

```ts
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0, // 2 повтора в CI, 0 локально
});
```

### Timeout — таймауты

`timeout` — максимальное время выполнения одного теста (по умолчанию 30 секунд). `expect.timeout` — максимальное время auto-retrying assertions (по умолчанию 5 секунд). Если тест превышает `timeout`, он падает с ошибкой «Test timeout exceeded». Если assertion превышает `expect.timeout`, он падает с «Timeout 5000ms exceeded». Увеличение таймаутов маскирует проблемы, а не решает их — ищите причину медленных тестов (долгие запросы, тяжёлый рендеринг).

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

Использование `process.env.CI` позволяет задать разные `baseURL` для локальной разработки и CI. В CI тесты работают против staging-окружения, локально — против `localhost`. Это требует, чтобы staging был стабилен и содержал тестовые данные. Альтернатива: запускать dev-сервер в CI через `webServer` — это медленнее, но изолированнее.

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

E2E-тесты в CI — это страховка от регрессии перед мерджем. Ключевые требования: тесты должны запускаться автоматически на каждый PR, отчёты должны сохраняться как artifacts (для отладки), а падение тестов должно блокировать мердж. Компромисс: E2E-тесты в CI медленнее локальных (меньше ресурсов, сетевые задержки), поэтому оптимизируйте конфигурацию: используйте `retries: 2`, параллельный запуск и кэш браузеров.

### GitHub Actions

GitHub Actions — популярный CI для GitHub-репозиториев. Workflow запускает тесты на каждый push и PR. `upload-artifact` сохраняет HTML-отчёт Playwright — его можно скачать и изучить упавшие тесты. `retention-days` задаёт срок хранения artifacts.

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

GitLab CI использует Docker-образ с предустановленными браузерами Playwright. Это ускоряет прогон — не нужно скачивать браузеры при каждом запуске. `artifacts` сохраняют отчёт, `expire_in` задаёт срок хранения.

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

Docker-образ с Playwright гарантирует идентичное окружение на всех машинах. Это устраняет проблемы «у меня работает». Образ `mcr.microsoft.com/playwright` содержит все зависимости браузеров (библиотеки для Linux). Используйте его как base для CI или локального запуска через `docker run`.

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

Отладка E2E-тестов сложнее, чем unit-тестов: вы не можете поставить breakpoint и пошагово выполнить код, потому что тест работает с браузером. Playwright решает эту проблему через трейсы — записи всех действий, скриншотов, DOM-снимков и сетевых запросов. Трейс можно воспроизвести как видео и увидеть, что происходило в каждом шаге. Это незаменимо для анализа flaky-тестов и багов, которые не воспроизводятся локально.

### Trace Viewer

`trace: "on-first-retry"` записывает трейс только при первом повторе упавшего теста. Это экономит диск и время, но даёт полную информацию для отладки. `show-report` открывает HTML-отчёт с трейсами, скриншотами и логами — вы можете кликнуть на каждый шаг и увидеть состояние DOM, консоли и сети.

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

`--debug` открывает Inspector — графический интерфейс для пошагового выполнения тестов. Вы можете ставить breakpoints, инспектировать DOM, выполнять команды в консоли. Это полезно для отладки сложных сценариев, где трейс недостаточно информативен.

```bash
npx playwright test --debug
```

Открывает Inspector для пошагового выполнения тестов.

### Codegen — генерация тестов

`codegen` записывает ваши действия в браузере и генерирует код теста. Это полезно для быстрого старта: вы кликаете по UI, а Playwright создаёт код с семантическими locator'ами. Компромисс: сгенерированный код не идеален — он не использует Page Object, не оптимизирован для читаемости. Используйте codegen как черновик, а затем рефакторите код.

```bash
npx playwright codegen https://example.com
```

Открывает браузер, записывает действия и генерирует код теста.

### UI Mode — интерактивный режим

`--ui` открывает интерактивный интерфейс для запуска, отладки и анализа тестов. В отличие от Inspector, UI Mode показывает все тесты, позволяет запускать их по одному или группами, видеть таймлайн действий и скриншоты. Это лучший инструмент для разработки новых тестов и анализа flaky-поведения.

```bash
npx playwright test --ui
```

Открывает UI для запуска, отладки и анализа тестов.

---

## Лучшие практики

Лучшие практики E2E-тестирования сформированы на основе опыта тысяч проектов. Их цель — сделать тесты стабильными, читаемыми и поддерживаемыми. Стабильность означает, что тесты не падают из-за временных проблем (сеть, рендеринг). Читаемость означает, что тест описывает бизнес-логику, а не технические детали. Поддерживаемость означает, что изменение UI требует минимальных правок в тестах.

### 1. Используйте семантические селекторы

Семантические селекторы (`getByRole`, `getByLabel`) привязаны к доступности, а не к визуальной реализации. Они ломаются только при реальных изменениях UI, а не при рефакторинге CSS. Это делает тесты устойчивыми и одновременно проверяет, что приложение доступно для скринридеров.

```ts
// ❌ CSS-селектор — хрупкий
await page.locator("div > button.btn-primary").click();

// ✅ Семантический селектор
await page.getByRole("button", { name: /submit/i }).click();
```

### 2. Используйте data-testid как последний резерв

`data-testid` — это явный контракт между компонентом и тестом. Он не влияет на визуальное отображение и доступность, но создаёт зависимость: если разработчик удалит `data-testid`, тест упадёт. Используйте его только когда нет семантических селекторов: кастомные компоненты, сложные виджеты, элементы без текста.

```ts
// ✅ Семантический селектор
await page.getByRole("button", { name: /submit/i }).click();

// ⚠️ data-testid — только если нет семантики
await page.getByTestId("submit-btn").click();
```

### 3. Используйте Page Object Model

Page Object Model централизует логику взаимодействия со страницей. Без POM каждый тест содержит CSS-селекторы и действия — изменение UI требует правки во всех тестах. С POM правится только один класс, а тесты остаются неизменными. POM также делает тесты читаемыми: `loginPage.login()` понятнее, чем последовательность `fill` и `click`.

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

Логин в каждом тесте замедляет прогон и создаёт точки отказа. Глобальная аутентификация через `storageState` выполняет логин один раз и переиспользует сессию. Это сокращает время прогона в 3-5 раз и устраняет flaky-тесты из-за проблем с аутентификацией.

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

`expect` в Playwright — это auto-retrying assertion. Он повторяет проверку до тех пор, пока она не пройдёт или не истечёт таймаут. Это устраняет необходимость явных `waitForSelector` перед проверками. Компромисс: если тест падает с таймаутом, это не значит, что элемент не появился — это значит, что он не появился за 5 секунд. Ищите причину, а не увеличивайте таймаут.

```ts
// ❌ Явное ожидание
await page.waitForSelector(".success");
expect(await page.isVisible(".success")).toBe(true);

// ✅ Auto-waiting
await expect(page.getByText(/success/i)).toBeVisible();
```

### 6. Изолируйте тесты

Каждый тест должен быть независимым: создавать свои данные, выполнять сценарий, проверять результат. Тесты не должны зависеть друг от друга — если один тест падает, остальные должны продолжаться. Это позволяет запускать тесты параллельно и упрощает отладку: если тест падает, вы знаете, что проблема в нём, а не в предыдущем тесте.

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

Зависимость от реального API делает тесты медленными и хрупкими: если API падает, тесты тоже падают. Мокирование API делает тесты быстрыми и предсказуемыми. Компромисс: моки создают «параллельную реальность» — тест проходит, но реальное приложение может не работать. Стратегия: мокайте сторонние API и сложные сценарии (ошибки, задержки), но используйте реальные endpoints для собственного бэкенда.

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

Антипаттерны — это распространённые ошибки, которые делают тесты хрупкими, медленными и сложными в поддержке. Они могут работать в малых проектах, но ломаются при масштабировании. Понимание антипаттернов помогает избежать проблем до того, как они станут критичными.

### 1. Использование CSS-селекторов

CSS-селекторы ломаются при любом изменении структуры DOM или классов. Это делает тесты хрупкими: рефакторинг CSS (который не влияет на UI) ломает тесты. Семантические селекторы (`getByRole`) привязаны к доступности и ломаются только при реальных изменениях UI.

```ts
// ❌ Хрупкий селектор
await page.locator("div.container > button.btn").click();

// ✅ Семантический селектор
await page.getByRole("button", { name: /submit/i }).click();
```

### 2. Явные ожидания (waitForTimeout)

`waitForTimeout` — фиксированная задержка перед действием. Это антипаттерн, потому что он не гарантирует, что элемент появится: если сервер отвечает медленнее, тест падает; если быстрее — тест тратит время впустую. Auto-waiting в Playwright решает эту проблему: действия ждут элемента автоматически, а assertions повторяют проверку до таймаута.

```ts
// ❌ Жёсткая задержка
await page.waitForTimeout(2000);
await expect(page.getByText("Loaded")).toBeVisible();

// ✅ Auto-waiting
await expect(page.getByText("Loaded")).toBeVisible();
```

### 3. Зависимость тестов друг от друга

Если второй тест зависит от первого (например, первый создаёт пользователя, второй его удаляет), это создаёт хрупкость: падение первого теста блокирует второй. Каждый тест должен быть независимым: создавать свои данные, выполнять сценарий, проверять результат. Это позволяет запускать тесты параллельно и упрощает отладку.

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

E2E-тесты медленные и хрупкие. Использование их для проверки каждой мелочи (цвет кнопки, центрирование текста) замедляет прогон и создаёт ложные срабатывания. E2E-тесты должны проверять критические бизнес-сценарии: логин, оплата, регистрация. Для проверки стилей и layout используйте визуальное тестирование или unit-тесты компонентов.

```ts
// ❌ E2E для каждой мелочи
test("button has correct color", async ({ page }) => { ... });
test("text is centered", async ({ page }) => { ... });

// ✅ E2E только для критических сценариев
test("user can complete checkout", async ({ page }) => { ... });
```

### 5. Игнорирование параллельного запуска

Последовательный запуск тестов (`mode: "serial"`) замедляет прогон и создаёт зависимости между тестами. Параллельный запуск (`fullyParallel: true`) ускоряет прогон в 3-5 раз и требует, чтобы тесты были независимы. Если тесты не могут быть параллельными, это сигнал о проблеме с изоляцией — исправьте тесты, а не отключайте параллелизм.

```ts
// ❌ Последовательный запуск
test.describe.configure({ mode: "serial" });

// ✅ Параллельный запуск
export default defineConfig({
  fullyParallel: true,
});
```

### 6. Отсутствие повторных запусков в CI

Flaky-тесты — неизбежная часть E2E-тестирования: сеть нестабильна, рендеринг может варьироваться. Без `retries` каждый flaky-тест блокирует CI. С `retries: 2` flaky-тесты проходят со второй попытки, а реальные баги всё равно падают. Компромисс: если тест требует retries: 3 или больше, это не flaky — это сломанный тест, который нужно исправить.

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
