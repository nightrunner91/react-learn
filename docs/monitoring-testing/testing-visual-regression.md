# Визуальное и регрессионное тестирование

## Содержание

1. [Что такое визуальное тестирование](#что-такое-визуальное-тестирование)
2. [Snapshot-тестирование](#snapshot-тестирование)
3. [Визуальное регрессионное тестирование](#визуальное-регрессионное-тестирование)
4. [Playwright для визуального тестирования](#playwright-для-визуального-тестирования)
5. [Chromatic](#chromatic)
6. [Percy](#percy)
7. [Storybook для визуального тестирования](#storybook-для-визуального-тестирования)
8. [Тестирование адаптивности](#тестирование-адаптивности)
9. [Тестирование тем](#тестирование-тем)
10. [Тестирование анимаций](#тестирование-анимаций)
11. [Тестирование шрифтов и иконок](#тестирование-шрифтов-и-иконок)
12. [CI/CD для визуальных тестов](#cicd-для-визуальных-тестов)
13. [Лучшие практики](#лучшие-практики)
14. [Антипаттерны](#антипаттерны)

---

## Что такое визуальное тестирование

Визуальное тестирование проверяет, что компоненты выглядят правильно визуально. Обычные тесты проверяют логику, но не вид. Компонент может работать правильно, но выглядеть сломанным.

### Виды визуального тестирования

| Вид | Что проверяет | Инструменты |
|---|---|---|
| **Snapshot-тестирование** | Сериализованный DOM | Jest, Vitest |
| **Визуальное регрессионное** | Пиксельные differences | Playwright, Chromatic, Percy |
| **Storybook** | Компоненты в изоляции | Storybook + addons |
| **Кросс-браузерное** | Внешний вид в разных браузерах | Playwright, BrowserStack |

### Когда использовать визуальное тестирование

**Использовать:**
- UI-библиотеки (Button, Input, Card)
- Дизайн-системы
- Компоненты с сложной стилизацией
- Критичные страницы (landing, checkout)

**Не использовать:**
- Компоненты с динамическим контентом (списки пользователей)
- Часто меняющиеся компоненты
- Компоненты с анимациями (если не тестируете анимации)

---

## Snapshot-тестирование

### Что такое snapshot

Snapshot — это сериализованное представление компонента (обычно DOM). Тест сравнивает текущий snapshot с сохранённым.

### Базовый snapshot-тест

```tsx
import { render } from "@testing-library/react";
import { Button } from "./Button";

it("renders button correctly", () => {
  const { container } = render(<Button variant="primary">Click me</Button>);
  expect(container).toMatchSnapshot();
});
```

### Что создаёт Vitest

Vitest создаёт файл `__snapshots__/Button.test.tsx.snap`:

```javascript
// Vitest Snapshot 1, https://vitest.dev/guide/snapshot.html

exports[`renders button correctly 1`] = `
<div>
  <button
    class="btn btn--primary"
  >
    Click me
  </button>
</div>
`;
```

### Inline snapshots

```tsx
it("renders button with inline snapshot", () => {
  const { container } = render(<Button>Click</Button>);
  expect(container.innerHTML).toMatchInlineSnapshot(`
    "<button class="btn btn--primary">Click</button>"
  `);
});
```

### Обновление snapshots

```bash
vitest --update    # Обновить все snapshots
vitest -u          # Короткая форма
```

### Когда использовать snapshot

| Сценарий | Snapshot? | Почему |
|---|---|---|
| Иконки, SVG | ✅ Да | Стабильные, редко меняются |
| Базовые UI-компоненты | ✅ Да | Если не меняются часто |
| Сложные компоненты | ❌ Нет | Snapshot ломается при рефакторе |
| Динамический контент | ❌ Нет | Snapshot зависит от данных |

### Ограничения snapshot-тестов

- **Хрупкие** — ломаются при любом изменении разметки
- **Не проверяют поведение** — только структуру
- **Требуют ревью** — нельзя слепо обновлять
- **Не кросс-браузерные** — только DOM, не рендеринг

---

## Визуальное регрессионное тестирование

### Что такое визуальная регрессия

Визуальная регрессия — это непреднамеренное изменение внешнего вида компонента. Например, кнопка стала другого цвета или сместилась на 2 пикселя.

### Как работает визуальное сравнение

1. Делается скриншот компонента (baseline)
2. При следующем запуске делается новый скриншот
3. Скриншоты сравниваются пиксельно
4. Если разница превышает порог — тест падает

### Инструменты

| Инструмент | Тип | Особенности |
|---|---|---|
| **Playwright** | Локальный | Встроенный `toHaveScreenshot()` |
| **Chromatic** | Облачный | Интеграция со Storybook, CI |
| **Percy** | Облачный | Поддержка множества браузеров |
| **BackstopJS** | Локальный | Open-source, гибкий |

---

## Playwright для визуального тестирования

### Базовое использование

```ts
import { test, expect } from "@playwright/test";

test("button looks correct", async ({ page }) => {
  await page.goto("/storybook/button");
  await expect(page.getByRole("button")).toHaveScreenshot();
});
```

Playwright создаёт файл `button-looks-correct-1-chromium-Darwin.png` в директории тестов.

### Скриншоты элементов

```ts
test("button component", async ({ page }) => {
  await page.goto("/components/button");
  const button = page.getByRole("button", { name: /submit/i });
  await expect(button).toHaveScreenshot();
});
```

### Настройки сравнения

```ts
test("homepage with options", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot({
    maxDiffPixelRatio: 0.01, // Допустимая разница 1%
    threshold: 0.2, // Порог различия пикселей (0-1)
    animations: "disabled", // Отключить анимации
    mask: [
      page.getByRole("navigation"), // Замаскировать навигацию
      page.locator(".timestamp"), // Замаскировать время
    ],
  });
});
```

### Параметры сравнения

| Параметр | Описание | Значение по умолчанию |
|---|---|---|
| `threshold` | Порог различия пикселей (0-1) | 0.2 |
| `maxDiffPixels` | Максимальное количество разных пикселей | Нет лимита |
| `maxDiffPixelRatio` | Максимальная доля разных пикселей (0-1) | Нет лимита |
| `animations` | Обработка анимаций | `"allow"` |
| `mask` | Элементы для маскировки (серые прямоугольники) | `[]` |
| `clip` | Область для скриншота `{x, y, width, height}` | Вся страница |

### Обновление скриншотов

```bash
npx playwright test --update-snapshots
```

### Полностраничные скриншоты

```ts
test("full page screenshot", async ({ page }) => {
  await page.goto("/");
  await page.screenshot({
    path: "full-page.png",
    fullPage: true,
  });
});
```

### Маскирование динамического контента

```ts
test("page with dynamic content", async ({ page }) => {
  await page.goto("/dashboard");
  
  await expect(page).toHaveScreenshot({
    mask: [
      page.locator(".timestamp"), // Время
      page.locator(".user-avatar"), // Аватар пользователя
      page.locator("[data-dynamic]"), // Динамический контент
    ],
  });
});
```

---

## Chromatic

### Что такое Chromatic

Chromatic — облачный сервис для визуального тестирования, созданный командой Storybook. Автоматически делает скриншоты компонентов в разных браузерах и сравнивает их.

### Установка

```bash
npm install -D chromatic
```

### Настройка GitHub Actions

```yaml
# .github/workflows/chromatic.yml
name: Chromatic
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Важно для Chromatic
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - run: npm ci
      
      - uses: chromaui/action@v1
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

### Использование со Storybook

```tsx
// Button.stories.tsx
import { Button } from "./Button";

export default {
  component: Button,
};

export const Primary = {
  args: {
    variant: "primary",
    children: "Click me",
  },
};

export const Secondary = {
  args: {
    variant: "secondary",
    children: "Cancel",
  },
};
```

Chromatic автоматически делает скриншоты всех stories и сравнивает их.

### Преимущества Chromatic

- **Интеграция со Storybook** — работает из коробки
- **Кросс-браузерное тестирование** — Chrome, Firefox, Safari
- **UI Review** — визуальный интерфейс для ревью изменений
- **CI/CD интеграция** — GitHub Actions, GitLab CI
- **Turbo Snap** — оптимизация для монорепо

---

## Percy

### Что такое Percy

Percy — облачный сервис визуального тестирования от BrowserStack. Поддерживает множество фреймворков и браузеров.

### Установка

```bash
npm install -D @percy/cli @percy/playwright
```

### Использование с Playwright

```ts
import { test, expect } from "@playwright/test";
import percySnapshot from "@percy/playwright";

test("homepage visual test", async ({ page }) => {
  await page.goto("/");
  await percySnapshot(page, "Homepage");
});

test("button component", async ({ page }) => {
  await page.goto("/components/button");
  await percySnapshot(page, "Button Component", {
    widths: [375, 768, 1280], // Разные ширины
  });
});
```

### CI/CD интеграция

```yaml
# .github/workflows/percy.yml
name: Percy
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  percy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - run: npm ci
      
      - run: npx playwright install --with-deps
      
      - run: npx percy exec -- npx playwright test
        env:
          PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
```

---

## Storybook для визуального тестирования

### Что такое Storybook

Storybook — инструмент для разработки UI-компонентов в изоляции. Позволяет создавать stories для каждого состояния компонента.

### Базовая story

```tsx
// Button.stories.tsx
import { Button } from "./Button";
import type { Meta, StoryObj } from "@storybook/react";

const meta: Meta<typeof Button> = {
  component: Button,
  title: "Components/Button",
  tags: ["autodocs"],
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: {
    variant: "primary",
    children: "Click me",
  },
};

export const Secondary: Story = {
  args: {
    variant: "secondary",
    children: "Cancel",
  },
};

export const Disabled: Story = {
  args: {
    variant: "primary",
    children: "Disabled",
    disabled: true,
  },
};

export const Loading: Story = {
  args: {
    variant: "primary",
    children: "Loading...",
    loading: true,
  },
};
```

### Interaction tests

```tsx
import { within, userEvent, expect } from "@storybook/test";

export const WithInteraction: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const button = canvas.getByRole("button");
    
    await userEvent.click(button);
    
    await expect(canvas.getByText("Clicked!")).toBeInTheDocument();
  },
};
```

### Visual tests с Chromatic

```tsx
// Button.stories.tsx
export const Primary: Story = {
  args: {
    variant: "primary",
    children: "Click me",
  },
  parameters: {
    chromatic: {
      viewports: [320, 768, 1024], // Тестирование на разных ширинах
    },
  },
};
```

---

## Тестирование адаптивности

### Playwright — разные viewports

```ts
import { test, expect, devices } from "@playwright/test";

test.describe("Responsive design", () => {
  test("mobile view", async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto("/");
    await expect(page).toHaveScreenshot("mobile.png");
  });

  test("tablet view", async ({ page }) => {
    await page.setViewportSize({ width: 768, height: 1024 });
    await page.goto("/");
    await expect(page).toHaveScreenshot("tablet.png");
  });

  test("desktop view", async ({ page }) => {
    await page.setViewportSize({ width: 1280, height: 800 });
    await page.goto("/");
    await expect(page).toHaveScreenshot("desktop.png");
  });
});
```

### Playwright — эмуляция устройств

```ts
test("iPhone 12", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot("iphone-12.png");
});

test.describe("Mobile devices", () => {
  test.use({
    ...devices["iPhone 12"],
  });

  test("iPhone 12 layout", async ({ page }) => {
    await page.goto("/");
    await expect(page).toHaveScreenshot();
  });
});
```

### Множественные viewports в одном тесте

```ts
test("responsive component", async ({ page }) => {
  const viewports = [
    { width: 375, height: 667, name: "mobile" },
    { width: 768, height: 1024, name: "tablet" },
    { width: 1280, height: 800, name: "desktop" },
  ];

  for (const viewport of viewports) {
    await page.setViewportSize({ width: viewport.width, height: viewport.height });
    await page.goto("/components/card");
    await expect(page.getByRole("article")).toHaveScreenshot(`card-${viewport.name}.png`);
  }
});
```

---

## Тестирование тем

### Тестирование dark/light темы

```ts
test("light theme", async ({ page }) => {
  await page.goto("/?theme=light");
  await expect(page).toHaveScreenshot("light-theme.png");
});

test("dark theme", async ({ page }) => {
  await page.goto("/?theme=dark");
  await expect(page).toHaveScreenshot("dark-theme.png");
});
```

### Эмуляция prefers-color-scheme

```ts
test("dark mode (system preference)", async ({ page }) => {
  await page.emulateMedia({ colorScheme: "dark" });
  await page.goto("/");
  await expect(page).toHaveScreenshot("dark-mode.png");
});
```

### Тестирование всех тем

```ts
const themes = ["light", "dark", "high-contrast"];

for (const theme of themes) {
  test(`${theme} theme`, async ({ page }) => {
    await page.goto(`/?theme=${theme}`);
    await expect(page).toHaveScreenshot(`theme-${theme}.png`);
  });
}
```

---

## Тестирование анимаций

### Отключение анимаций

```ts
test("component without animations", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot({
    animations: "disabled", // Отключить анимации
  });
});
```

### Тестирование состояний анимации

```ts
test("hover state", async ({ page }) => {
  await page.goto("/components/button");
  const button = page.getByRole("button");
  
  await button.hover();
  
  await expect(button).toHaveScreenshot("button-hover.png");
});
```

### Тестирование переходов

```ts
test("modal animation", async ({ page }) => {
  await page.goto("/modal-demo");
  
  // До открытия
  await expect(page).toHaveScreenshot("modal-closed.png");
  
  // Открыть модальное окно
  await page.getByRole("button", { name: /open modal/i }).click();
  await page.waitForTimeout(500); // Ждём анимацию
  
  // После открытия
  await expect(page).toHaveScreenshot("modal-opened.png");
});
```

---

## Тестирование шрифтов и иконок

### Тестирование загрузки шрифтов

```ts
test("fonts loaded correctly", async ({ page }) => {
  await page.goto("/");
  
  // Ждём загрузки шрифтов
  await page.waitForFunction(() => document.fonts.ready);
  
  await expect(page).toHaveScreenshot("with-fonts.png");
});
```

### Тестирование SVG-иконок

```tsx
import { render } from "@testing-library/react";
import { Icon } from "./Icon";

it("renders icon correctly", () => {
  const { container } = render(<Icon name="check" />);
  expect(container).toMatchSnapshot();
});
```

---

## CI/CD для визуальных тестов

### GitHub Actions — Playwright

```yaml
name: Visual Tests
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  visual-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - run: npm ci
      
      - run: npx playwright install --with-deps
      
      - run: npx playwright test visual/
      
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

### Сравнение в CI

```yaml
- name: Compare screenshots
  run: |
    npx playwright test visual/
    # Если есть различия, тест упадёт
```

### Хранение baseline в репозитории

```
tests/
  visual/
    homepage.spec.ts
    __screenshots__/
      homepage-1-chromium-Darwin.png
      homepage-1-firefox-Darwin.png
```

---

## Лучшие практики

### 1. Используйте маски для динамического контента

```ts
await expect(page).toHaveScreenshot({
  mask: [
    page.locator(".timestamp"),
    page.locator(".user-avatar"),
  ],
});
```

### 2. Отключайте анимации

```ts
await expect(page).toHaveScreenshot({
  animations: "disabled",
});
```

### 3. Тестируйте на разных viewports

```ts
const viewports = [
  { width: 375, name: "mobile" },
  { width: 768, name: "tablet" },
  { width: 1280, name: "desktop" },
];
```

### 4. Используйте Storybook для изоляции

```tsx
// Тестируйте компоненты в изоляции, а не на реальных страницах
export const Primary = {
  args: { variant: "primary", children: "Click me" },
};
```

### 5. Ревьюте изменения перед обновлением snapshots

```bash
# Посмотреть diff перед обновлением
git diff __snapshots__/

# Обновить только если изменения ожидаемы
vitest -u
```

### 6. Используйте стабильные селекторы

```ts
// ✅ Семантический селектор
await expect(page.getByRole("button")).toHaveScreenshot();

// ❌ Хрупкий селектор
await expect(page.locator("div > button")).toHaveScreenshot();
```

---

## Антипаттерны

### 1. Snapshot-тестирование всего

```tsx
// ❌ Snapshot ломается при каждом изменении
expect(container).toMatchSnapshot();

// ✅ Snapshot только для стабильных компонентов
expect(screen.getByRole("button")).toMatchSnapshot();
```

### 2. Слепое обновление snapshots

```bash
# ❌ Обновление без ревью
vitest -u

# ✅ Ревью diff перед обновлением
git diff __snapshots__/
vitest -u
```

### 3. Игнорирование динамического контента

```ts
// ❌ Скриншот с динамическим временем
await expect(page).toHaveScreenshot();

// ✅ Маскирование динамического контента
await expect(page).toHaveScreenshot({
  mask: [page.locator(".timestamp")],
});
```

### 4. Тестирование анимаций без отключения

```ts
// ❌ Анимации делают тесты нестабильными
await expect(page).toHaveScreenshot();

// ✅ Отключение анимаций
await expect(page).toHaveScreenshot({
  animations: "disabled",
});
```

### 5. Использование хрупких селекторов

```ts
// ❌ CSS-селектор
await expect(page.locator("div > button")).toHaveScreenshot();

// ✅ Семантический селектор
await expect(page.getByRole("button")).toHaveScreenshot();
```

### 6. Тестирование слишком больших областей

```ts
// ❌ Вся страница — много динамического контента
await expect(page).toHaveScreenshot();

// ✅ Конкретный компонент
await expect(page.getByRole("main")).toHaveScreenshot();
```
