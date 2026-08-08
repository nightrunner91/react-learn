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

Простейший snapshot-тест рендерит компонент и сохраняет его DOM-дерево в файл. При повторном запуске Vitest сравнивает новый рендер с сохранённым. Если структура изменилась — тест падает и показывает diff. Это защищает от непреднамеренных изменений разметки, например, когда вы случайно удалили класс или изменили вложенность элементов.

```tsx
import { render } from "@testing-library/react";
import { Button } from "./Button";

it("renders button correctly", () => {
  const { container } = render(<Button variant="primary">Click me</Button>);
  expect(container).toMatchSnapshot();
});
```

### Что создаёт Vitest

При первом запуске Vitest создаёт файл снапшота в директории `__snapshots__` рядом с тестом. Этот файл — текстовое представление DOM-дерева, которое легко читать и ревьюить в git. Важно коммитить эти файлы в репозиторий: они и есть ваш baseline. Если файл удалить, тест просто пересоздаст его при следующем запуске — но это означает, что вы потеряли защиту от регрессий.

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

Inline-снапшоты хранятся прямо в тестовом файле, а не в отдельном `.snap` файле. Это удобно для коротких снапшотов — не нужно переключаться между файлами при ревью. Vitest сам вставляет и обновляет содержимое inline-снапшота при запуске с флагом `-u`.

Не злоупотребляйте inline-снапшотами: длинные snapshot'ы загромождают тест и мешают читать логику. Если snapshot занимает больше 5-10 строк, лучше вынести его в отдельный файл.

```tsx
it("renders button with inline snapshot", () => {
  const { container } = render(<Button>Click</Button>);
  expect(container.innerHTML).toMatchInlineSnapshot(`
    "<button class="btn btn--primary">Click</button>"
  `);
});
```

### Обновление snapshots

Когда компонент намеренно изменился, snapshot нужно обновить. Но никогда не делайте это автоматически, не посмотрев diff. Snapshots — это контракт: если вы обновили их слепо, вы можете пропустить баг. Всегда проверяйте `git diff __snapshots__/` перед обновлением. В CI обновлять snapshots нельзя — только локально.

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

Playwright имеет встроенную поддержку визуального тестирования через `toHaveScreenshot()`. В отличие от snapshot-тестов, которые сравнивают DOM, Playwright делает реальный скриншот рендеринга браузера и сравнивает его пиксельно. Это значит, что он ловит проблемы, которые не видны в DOM: неправильные отступы, цвета, шрифты. При первом запуске скриншот сохраняется как baseline, при последующих — сравнивается с ним.

```ts
import { test, expect } from "@playwright/test";

test("button looks correct", async ({ page }) => {
  await page.goto("/storybook/button");
  await expect(page.getByRole("button")).toHaveScreenshot();
});
```

Playwright создаёт файл `button-looks-correct-1-chromium-Darwin.png` в директории тестов.

### Скриншоты элементов

Иногда нужно проверить не всю страницу, а конкретный элемент. Это снижает хрупкость теста: изменения в других частях страницы не сломают его. Скриншот элемента также проще ревьюить — вы видите ровно то, что тестируете. Используйте `toHaveScreenshot()` на locator'е, а не на всей странице, когда проверяете отдельный компонент.

```ts
test("button component", async ({ page }) => {
  await page.goto("/components/button");
  const button = page.getByRole("button", { name: /submit/i });
  await expect(button).toHaveScreenshot();
});
```

### Настройки сравнения

Пиксельное сравнение редко бывает идеальным: антиалиасинг шрифтов, субпиксельный рендеринг и различия в окружении могут давать мелкие расхождения. Параметры `threshold` и `maxDiffPixelRatio` позволяют задать допустимый порог. `mask` — критически важная опция: она закрашивает указанные элементы серым прямоугольником, исключая их из сравнения. Используйте маски для всего динамического: время, аватары, счётчики.

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

Когда визуальные изменения ожидаемы (например, вы поменяли дизайн), baseline нужно обновить. Флаг `--update-snapshots` перезаписывает все скриншоты. Как и с DOM-snapshots, никогда не делайте этого не глядя: используйте `git diff` для проверки изменений. В CI этот флаг использовать нельзя — CI должен только сравнивать.

```bash
npx playwright test --update-snapshots
```

### Полностраничные скриншоты

`page.screenshot()` делает скриншот всей страницы, включая прокручиваемую область. Это отличается от `toHaveScreenshot()`, который по умолчанию снимает только viewport. Полностраничные скриншоты полезны для архивации состояния страницы или для отладки, но для регрессионных тестов они слишком хрупкие — слишком много контента может измениться. Предпочитайте скриншоты конкретных компонентов.

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

Любой контент, который меняется от запуска к запуску, сделает визуальный тест нестабильным (flaky). Типичные источники нестабильности: timestamps, аватары пользователей, случайные ID, рекламные баннеры, контент, подгружаемый из API. Маскирование — это компромисс: вы жертвуете проверкой части страницы ради стабильности тестов. Если динамического контента слишком много, возможно, стоит сузить область тестирования до конкретного компонента.

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

Chromatic работает в связке со Storybook: он берёт ваши stories, рендерит их в облаке и сравнивает скриншоты. Пакет `chromatic` — это CLI, который отправляет stories на сервер. Бесплатный план подходит для open-source проектов, для коммерческих нужно покупать лицензию.

```bash
```

### Настройка GitHub Actions

Chromatic интегрируется в CI через официальный GitHub Action. Параметр `fetch-depth: 0` обязателен — Chromatic анализирует git-историю, чтобы определить, какие stories изменились, и тестировать только их (Turbo Snap). Без полного history оптимизация не работает, и тесты будут медленнее. Токен проекта храните в GitHub Secrets, не коммитьте его.

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

Chromatic автоматически находит все stories в проекте и делает их скриншоты. Вам не нужно писать отдельные тесты — достаточно иметь stories. При каждом PR Chromatic покажет в GitHub, какие компоненты изменились визуально, и позволит принять или отклонить изменения через UI. Это особенно удобно для дизайн-ревью: дизайнер может aprobarить изменения, не запуская проект локально.

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

Percy от BrowserStack — альтернатива Chromatic. Главное отличие: Percy поддерживает не только Storybook, но и любые страницы, включая Playwright, Cypress, Puppeteer. Если вам нужно тестировать не только компоненты, но и целые страницы в разных браузерах, Percy может быть удобнее. Бесплатный план ограничен 1200 скриншотами в месяц.

```bash
```

### Использование с Playwright

Percy оборачивает скриншоты в свой API: `percySnapshot` отправляет снимок на сервер Percy, где он сравнивается с baseline. Параметр `widths` позволяет сделать снимки на разных ширинах экрана за один вызов — это заменяет необходимость писать отдельные тесты для каждого breakpoint. Percy рендерит каждый snapshot в реальном браузере, что даёт более достоверные результаты, чем локальное сравнение.

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

Percy в CI работает через `percy exec`, который оборачивает ваш тестовый раннер. Все вызовы `percySnapshot` внутри тестов автоматически отправляются на сервер. Токен `PERCY_TOKEN` храните в секретах CI. Percy поддерживает параллельное выполнение тестов и автоматически агрегирует результаты.

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

Story — это изолированное состояние компонента. Каждая story описывает один вариант: с разными props, состояниями или контекстом. Storybook рендерит stories независимо друг от друга, что делает визуальные тесты стабильнее: на компонент не влияют другие части приложения. Типизация через `Meta<typeof Component>` и `StoryObj` даёт автодополнение и проверку типов для args.

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

Interaction tests проверяют поведение компонента: клики, ввод текста, навигацию. Функция `play` выполняется автоматически при открытии story в Storybook и при прогоне тестов через `@storybook/test-runner`. Это позволяет тестировать интерактивные состояния (hover, dropdown, модальные окна) без отдельного e2e-фреймворка. Однако для сложных сценариев с несколькими страницами лучше использовать Playwright.

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

Chromatic позволяет задать параметры визуального тестирования прямо в story через `parameters.chromatic`. Параметр `viewports` делает скриншоты на разных ширинах — это заменяет необходимость писать отдельные тесты для каждого breakpoint. Можно также задать задержку перед скриншотом (`delay`), если компонент загружает данные или анимируется.

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

Адаптивный дизайн — один из главных источников визуальных багов. Компонент может выглядеть хорошо на десктопе, но ломаться на мобильном. Тестирование на разных viewports ловит эти проблемы автоматически. Выберите 3-4 характерных ширины (mobile, tablet, desktop, wide) и делайте скриншоты на каждой. Не тестируйте все возможные ширины — достаточно ключевых breakpoint'ов.

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

Playwright содержит встроенную базу устройств (devices) с реальными параметрами: viewport, user agent, device pixel ratio, поддержка touch. Эмуляция устройства даёт более реалистичные результаты, чем просто изменение viewport, потому что учитывает все характеристики. Используйте `test.use()` для применения конфигурации устройства ко всем тестам в describe-блоке.

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

Цикл по viewports в одном тесте удобнее, чем отдельные тесты: все скриншоты генерируются в одном контексте, и легче сравнивать результаты. Но есть trade-off: если один viewport упадёт, остальные не выполнятся. Для критичных компонентов лучше использовать отдельные тесты, для рутинной проверки — цикл.

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

Переключение тем — частый источник багов: цвета могут сливаться, контрастность падать, иконки становиться невидимыми. Визуальные тесты на каждую тему гарантируют, что оба варианта выглядят корректно. Если тема переключается через URL-параметр, просто навигируйте на нужную страницу. Если через CSS-класс или data-атрибут на `<html>`, используйте `page.evaluate()` для его установки.

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

Некоторые приложения используют системные настройки пользователя через `prefers-color-scheme` media query. Playwright может эмулировать эти настройки через `page.emulateMedia()`. Это проверяет, что компонент корректно реагирует на системную тему, даже если в приложении нет ручного переключателя.

```ts
test("dark mode (system preference)", async ({ page }) => {
  await page.emulateMedia({ colorScheme: "dark" });
  await page.goto("/");
  await expect(page).toHaveScreenshot("dark-mode.png");
});
```

### Тестирование всех тем

Если в приложении больше двух тем (например, light, dark, high-contrast), удобно параметризовать тесты. Цикл по темам сокращает дублирование кода. Убедитесь, что каждая тема имеет свой baseline-скриншот — они не должны совпадать.

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

Анимации — главный враг стабильных визуальных тестов. Каждый запуск может захватить кадр в другой момент, что приведёт к flaky-тестам. Playwright может отключить CSS-анимации и transitions опцией `animations: "disabled"`. Это замораживает элемент в конечном состоянии. Если вам нужно проверить именно анимацию, используйте другой подход (см. ниже).

```ts
test("component without animations", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot({
    animations: "disabled", // Отключить анимации
  });
});
```

### Тестирование состояний анимации

Иногда нужно проверить визуальное состояние после hover, focus или другого взаимодействия. Проблема в том, что hover-эффекты могут включать transitions, и скриншот может быть сделан в промежуточном состоянии. Решение: либо отключить анимации, либо добавить явную задержку через `waitForTimeout`, чтобы transition завершился. Задержка — это хак, но для визуальных тестов это допустимо.

```ts
test("hover state", async ({ page }) => {
  await page.goto("/components/button");
  const button = page.getByRole("button");
  
  await button.hover();
  
  await expect(button).toHaveScreenshot("button-hover.png");
});
```

### Тестирование переходов

Тестирование анимаций открытия/закрытия (модальные окна, dropdown, sidebar) — это проверка конечных состояний, а не самого процесса. Вы снимаете скриншот до и после действия, убеждаясь, что компонент перешёл в правильное состояние. `waitForTimeout` здесь необходим, чтобы анимация успела завершиться. Более надёжный способ — ждать появления/исчезновения элемента через `waitForSelector`, а не полагаться на таймер.

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

Веб-шрифты загружаются асинхронно. Если сделать скриншот до загрузки, текст отрисуется fallback-шрифтом, и результат будет отличаться от ожидаемого. `document.fonts.ready` — Promise, который резолвится, когда все шрифты загружены. Всегда ждите его перед визуальными тестами, иначе получите нестабильные результаты.

```ts
test("fonts loaded correctly", async ({ page }) => {
  await page.goto("/");
  
  // Ждём загрузки шрифтов
  await page.waitForFunction(() => document.fonts.ready);
  
  await expect(page).toHaveScreenshot("with-fonts.png");
});
```

### Тестирование SVG-иконок

SVG-иконки — идеальный кандидат для snapshot-тестов: они стабильны, редко меняются, и их DOM-структура важна. Snapshot ловит изменения в путях, размерах и атрибутах. Это особенно полезно для иконочных библиотек, где случайное изменение SVG может быть незаметно при ручном ревью.

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

Визуальные тесты в CI требуют тех же зависимостей, что и обычные e2e-тесты: браузеры, системные библиотеки. `playwright install --with-deps` устанавливает и то, и другое. Артефакты (отчёты и скриншоты) полезно сохранять на 30 дней — это помогает отлаживать упавшие тесты постфактум. Если тест упал из-за визуальной регрессии, Playwright приложит diff-изображение к отчёту.

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

В CI визуальные тесты работают так же, как локально: Playwright сравнивает текущий скриншот с baseline из репозитория. Если есть расхождения — тест падает. Обновлять baseline в CI нельзя (нет флага `--update-snapshots`). Разработчик должен обновить baseline локально и запушить изменения вместе с кодом.

```yaml
- name: Compare screenshots
  run: |
    npx playwright test visual/
    # Если есть различия, тест упадёт
```

### Хранение baseline в репозитории

Baseline-скриншоты хранятся в репозитории рядом с тестами. Имена файлов включают имя теста, индекс скриншота, браузер и ОС — это позволяет иметь разные baseline для разных окружений. Trade-off: репозиторий растёт, особенно если много тестов. Альтернатива — хранить baseline в облаке (Chromatic, Percy), но это добавляет зависимость от внешнего сервиса.

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

Без маскировки динамического контента (время, аватары, счётчики) визуальные тесты будут flaky — падать из-за изменений, которые не являются багами. Маскируйте всё, что меняется от запуска к запуску.

```ts
await expect(page).toHaveScreenshot({
  mask: [
    page.locator(".timestamp"),
    page.locator(".user-avatar"),
  ],
});
```

### 2. Отключайте анимации

CSS-анимации и transitions делают визуальные тесты нестабильными, потому что каждый запуск может захватить кадр в другой момент. Отключение анимаций — обязательная практика для стабильных тестов.

```ts
await expect(page).toHaveScreenshot({
  animations: "disabled",
});
```

### 3. Тестируйте на разных viewports

Адаптивные баги часто пропускаются, потому что разработчик смотрит только на свой монитор. Автоматическое тестирование на 3-4 характерных ширинах ловит проблемы с вёрсткой до того, как они попадут в продакшен.

```ts
const viewports = [
  { width: 375, name: "mobile" },
  { width: 768, name: "tablet" },
  { width: 1280, name: "desktop" },
];
```

### 4. Используйте Storybook для изоляции

Тестирование компонентов на реальных страницах хрупкое: изменения в других частях страницы ломают тест. Storybook изолирует компонент, делая тесты стабильнее и быстрее.

```tsx
// Тестируйте компоненты в изоляции, а не на реальных страницах
export const Primary = {
  args: { variant: "primary", children: "Click me" },
};
```

### 5. Ревьюте изменения перед обновлением snapshots

Слепое обновление snapshots — главная причина, по которой snapshot-тесты теряют доверие. Если вы не смотрите diff, вы не тестируете ничего. Всегда проверяйте `git diff` перед обновлением.

```bash
# Посмотреть diff перед обновлением
git diff __snapshots__/

# Обновить только если изменения ожидаемы
vitest -u
```

### 6. Используйте стабильные селекторы

CSS-селекторы вроде `div > button` ломаются при любом рефакторе разметки. Семантические селекторы (role, text) устойчивы к изменениям структуры DOM. Это правило из e2e-тестирования, но для визуальных тестов оно не менее важно.

```ts
// ✅ Семантический селектор
await expect(page.getByRole("button")).toHaveScreenshot();

// ❌ Хрупкий селектор
await expect(page.locator("div > button")).toHaveScreenshot();
```

---

## Антипаттерны

### 1. Snapshot-тестирование всего

Snapshot всей страницы или большого компонента ломается при любом изменении разметки. Это создаёт шум и заставляет разработчиков слепо обновлять snapshots, теряя смысл тестов. Snapshot'ьте только конкретные, стабильные элементы.

```tsx
// ❌ Snapshot ломается при каждом изменении
expect(container).toMatchSnapshot();

// ✅ Snapshot только для стабильных компонентов
expect(screen.getByRole("button")).toMatchSnapshot();
```

### 2. Слепое обновление snapshots

Команда `vitest -u` без ревью diff — это антипаттерн. Если вы не проверяете, что изменилось, вы можете пропустить реальный баг. Всегда смотрите diff перед обновлением.

```bash
# ❌ Обновление без ревью
vitest -u

# ✅ Ревью diff перед обновлением
git diff __snapshots__/
vitest -u
```

### 3. Игнорирование динамического контента

Если на странице есть время, аватары или другой динамический контент, скриншот будет меняться при каждом запуске. Это делает тест flaky. Всегда маскируйте динамический контент.

```ts
// ❌ Скриншот с динамическим временем
await expect(page).toHaveScreenshot();

// ✅ Маскирование динамического контента
await expect(page).toHaveScreenshot({
  mask: [page.locator(".timestamp")],
});
```

### 4. Тестирование анимаций без отключения

Анимации делают каждый скриншот уникальным, потому что кадр зависит от тайминга. Без отключения анимаций тесты будут flaky. Если нужно проверить анимацию, используйте отдельный подход с фиксированными задержками.

```ts
// ❌ Анимации делают тесты нестабильными
await expect(page).toHaveScreenshot();

// ✅ Отключение анимаций
await expect(page).toHaveScreenshot({
  animations: "disabled",
});
```

### 5. Использование хрупких селекторов

CSS-селекторы вроде `div > button` ломаются при рефакторе разметки. Если селектор не находит элемент, тест падает с ошибкой, не связанной с визуальными изменениями. Используйте семантические селекторы.

```ts
// ❌ CSS-селектор
await expect(page.locator("div > button")).toHaveScreenshot();

// ✅ Семантический селектор
await expect(page.getByRole("button")).toHaveScreenshot();
```

### 6. Тестирование слишком больших областей

Скриншот всей страницы содержит много динамического контента и зависит от множества компонентов. Это делает тест хрупким и сложным для ревью. Тестируйте конкретные компоненты или секции.

```ts
// ❌ Вся страница — много динамического контента
await expect(page).toHaveScreenshot();

// ✅ Конкретный компонент
await expect(page.getByRole("main")).toHaveScreenshot();
```
