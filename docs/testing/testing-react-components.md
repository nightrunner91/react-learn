# Тестирование React-компонентов с React Testing Library

## Содержание

1. [Что такое React Testing Library](#что-такое-react-testing-library)
2. [Философия: тестируйте как пользователь](#философия-тестируйте-как-пользователь)
3. [Установка и настройка](#установка-и-настройка)
4. [render — рендеринг компонентов](#render--рендеринг-компонентов)
5. [screen — поиск элементов](#screen--поиск-элементов)
6. [Queries — приоритет и виды](#queries--приоритет-и-виды)
7. [fireEvent vs userEvent](#fireevent-vs-userevent)
8. [Тестирование взаимодействий](#тестирование-взаимодействий)
9. [Тестирование форм](#тестирование-форм)
10. [Тестирование асинхронных компонентов](#тестирование-асинхронных-компонентов)
11. [Тестирование с провайдерами](#тестирование-с-провайдерами)
12. [renderHook — тестирование хуков](#renderhook--тестирование-хуков)
13. [Тестирование условного рендеринга](#тестирование-условного-рендеринга)
14. [Тестирование списков](#тестирование-списков)
15. [Тестирование Error Boundaries](#тестирование-error-boundaries)
16. [Кастомные render-обёртки](#кастомные-render-обёртки)
17. [Лучшие практики](#лучшие-практики)
18. [Антипаттерны](#антипаттерны)

---

## Что такое React Testing Library

**React Testing Library (RTL)** — это библиотека для тестирования React-компонентов, разработанная Kent C. Dodds. Она фокусируется на тестировании с точки зрения **пользователя**, а не внутренней реализации компонента.

### Ключевые отличия от Enzyme (устаревший подход)

| Характеристика | React Testing Library | Enzyme |
|---|---|---|
| **Философия** | Тестировать поведение пользователя | Тестировать внутреннюю реализацию |
| **Доступ к state** | ❌ Нет (и не нужно) | ✅ Есть (shallow rendering) |
| **Доступ к методам** | ❌ Нет | ✅ Есть (instance(), setState()) |
| **Query-методы** | Семантические (getByRole, getByText) | CSS-селекторы (find(), .at()) |
| **Shallow rendering** | ❌ Нет | ✅ Есть |
| **Поддержка React 19** | ✅ Да | ❌ Нет (заброшен) |

> 💡 **Enzyme мёртв.** С выходом React 18+ и функциональных компонентов Enzyme потерял актуальность. React Testing Library — официальный стандарт с 2021 года.

> 📘 Для Vue используется [Vue Test Utils](./testing-vue-components.md) — библиотека с похожей философией, но со своими особенностями доступа к инстансу компонента.

### Почему RTL лучше

1. **Тесты не ломаются при рефакторинге** — если вы переписали компонент на другой хук, но поведение осталось тем же, тесты проходят.
2. **Тесты документируют поведение** — тест описывает, что видит и что может сделать пользователь.
3. **Тесты дают больше уверенности** — если тест проходит, значит компонент работает для пользователя.

---

## Философия: тестируйте как пользователь

### Главный принцип

> Чем больше ваши тесты resemble способы использования вашего программного обеспечения, тем больше уверенности они могут дать вам.
> — Kent C. Dodds

### Что это значит на практике

```tsx
// ❌ Тестирует реализацию — хрупкий тест
it("calls setState when button is clicked", () => {
  const setState = vi.spyOn(React, "useState");
  render(<Counter />);
  fireEvent.click(screen.getByText("Increment"));
  expect(setState).toHaveBeenCalledWith(1);
});

// ✅ Тестирует поведение — устойчивый тест
it("increments count when button is clicked", async () => {
  const user = userEvent.setup();
  render(<Counter />);
  
  await user.click(screen.getByRole("button", { name: /increment/i }));
  
  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

### Что проверяет пользователь

Пользователь не знает о `state`, `props`, `useEffect`. Он видит:
- **Текст** на странице
- **Кнопки** и другие интерактивные элементы
- **Формы** для ввода данных
- **Изменения** после взаимодействия

Тесты должны проверять то же самое.

---

## Установка и настройка

Правильная настройка тестового окружения — фундамент стабильных тестов. RTL требует три компонента: саму библиотеку для рендеринга, matcher'ы для читаемых assertions и библиотеку пользовательских событий. Все три устанавливаются как dev-зависимости, потому что тесты не попадают в production-бандл.

### Базовая установка

```bash
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

- `@testing-library/react` — основная библиотека (render, screen, fireEvent)
- `@testing-library/jest-dom` — дополнительные matcher'ы (toBeInTheDocument, toHaveTextContent)
- `@testing-library/user-event` — более реалистичные пользовательские события (рекомендуется вместо fireEvent)

### Настройка Vitest

Vitest — рекомендуемый тестовый раннер для RTL. Он быстрый, нативно поддерживает TypeScript и ESM, и совместим с Jest API. Ключевая настройка — `environment: "jsdom"`, которая эмулирует DOM в Node.js. Без этого `render()` не сможет работать, потому что в Node.js нет `document`.

`globals: true` позволяет использовать `it`, `expect`, `describe` без импорта — как в Jest. Это удобно, но не обязательно.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
  },
});
```

### Setup-файл

Setup-файл выполняется один раз перед всеми тестами. Здесь регистрируются глобальные matcher'ы и настраивается очистка DOM. `cleanup()` критически важен: без него DOM от предыдущего теста может повлиять на следующий, что приведёт к непредсказуемым результатам и утечкам памяти.

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach } from "vitest";

afterEach(() => {
  cleanup();
});
```

- `@testing-library/jest-dom/vitest` — добавляет matcher'ы в Vitest
- `cleanup()` — очищает DOM после каждого теста (предотвращает утечки)

---

## render — рендеринг компонентов

`render` — главная функция RTL. Она монтирует компонент в виртуальный DOM (jsdom) и возвращает набор утилит для поиска элементов и управления жизненным циклом. Без вызова `render` ни один тест не сможет проверить, что компонент отображается корректно.

Важно понимать, что `render` не создаёт реальный браузерный DOM — он работает в эмулированной среде `jsdom`. Этого достаточно для большинства тестов, но для случаев, когда нужна реальная отрисовка (например, измерение размеров элементов), потребуется `@testing-library/react` с окружением `happy-dom` или интеграция с Playwright.

### Базовый render

```tsx
import { render, screen } from "@testing-library/react";

it("renders a button", () => {
  render(<Button>Click me</Button>);
  
  const button = screen.getByRole("button", { name: /click me/i });
  expect(button).toBeInTheDocument();
});
```

### Что возвращает render

Функция `render` возвращает объект с несколькими полезными свойствами. В повседневных тестах чаще всего используются только `container` (для snapshot-тестов) и query-методы. Остальные свойства — `rerender`, `unmount`, `asFragment` — нужны для специфических сценариев: тестирование обновлений пропсов, проверка эффектов размонтирования и snapshot-тестирование соответственно.

```tsx
const {
  container,      // Корневой DOM-элемент (div)
  baseElement,    // Базовый элемент (document.body)
  debug,          // Функция для вывода DOM в консоль
  rerender,       // Перерендерить с новыми пропсами
  unmount,        // Размонтировать компонент
  asFragment,     // Вернуть DocumentFragment (для snapshot)
  
  // Queries
  getByText,
  getByRole,
  queryByText,
  queryByRole,
  findByText,
  findByRole,
  // ... и другие
} = render(<MyComponent />);
```

### rerender — перерендер с новыми пропсами

`rerender` полезен, когда нужно проверить, как компонент реагирует на изменение пропсов без полного размонтирования. Это единственный способ протестировать поведение при обновлении — например, анимации, побочные эффекты в `useEffect` с зависимостями от пропсов, или мемоизацию.

Не путайте `rerender` с `render` — второй вызов `render` создаст новый корневой элемент, а `rerender` обновит существующий, сохранив внутреннее состояние компонента.

```tsx
it("updates when props change", () => {
  const { rerender } = render(<Greeting name="Alice" />);
  
  expect(screen.getByText("Hello, Alice")).toBeInTheDocument();
  
  rerender(<Greeting name="Bob" />);
  
  expect(screen.getByText("Hello, Bob")).toBeInTheDocument();
  expect(screen.queryByText("Hello, Alice")).not.toBeInTheDocument();
});
```

### debug — отладка DOM

`debug` — ваш главный инструмент отладки. Когда тест падает и вы не понимаете, почему query не находит элемент, вызовите `screen.debug()`. Он выведет текущее состояние DOM в консоль в читаемом формате. Можно передать конкретный элемент, чтобы не засорять вывод.

Не забывайте удалять `debug()` перед коммитом — это отладочный инструмент, а не часть теста.

```tsx
it("debug example", () => {
  render(<MyComponent />);
  
  screen.debug(); // Выводит весь DOM
  screen.debug(screen.getByRole("button")); // Выводит конкретный элемент
});
```

---

## screen — поиск элементов

`screen` — это глобальный объект, предоставляющий все query-методы для поиска элементов в DOM. До появления `screen` (RTL v12) query-методы деструктурировались из результата `render`, что создавало проблемы с областью видимости.

`screen` решает эту проблему: он всегда доступен после вызова `render`, работает в хелпер-функциях и делает тесты более читаемыми. Современный стандарт — использовать именно `screen`, а не деструктуризацию.

### Почему screen лучше деструктуризации

```tsx
// ❌ Деструктуризация из render — нужно передавать в функции
const { getByText } = render(<MyComponent />);

function helper() {
  getByText("Hello"); // ❌ getByText не доступен здесь
}

// ✅ screen — доступен везде
render(<MyComponent />);

function helper() {
  screen.getByText("Hello"); // ✅ Работает
}
```

### Основные методы screen

Разница между `get*`, `query*` и `find*` — ключевой момент для понимания RTL. Выбор неправильного префикса — частая причина хрупких тестов.

**Правило:** `get*` — когда элемент точно должен быть. `query*` — когда проверяете отсутствие. `find*` — когда элемент появится асинхронно.

```tsx
// get* — возвращает элемент или выбрасывает ошибку
screen.getByText("Hello");
screen.getByRole("button");
screen.getByLabelText(/email/i);

// query* — возвращает элемент или null (не выбрасывает ошибку)
screen.queryByText("Hello");
screen.queryByRole("button");

// find* — возвращает Promise (ждёт появления элемента)
await screen.findByText("Hello");
await screen.findByRole("button");
```

---

## Queries — приоритет и виды

Query-методы — это способ найти элемент в DOM. RTL предоставляет несколько типов query, каждый из которых отражает определённый аспект доступности. Приоритет построен по принципу: чем ближе к тому, как пользователь (в том числе с ограниченными возможностями) взаимодействует со страницей — тем лучше.

Использование семантических query (особенно `getByRole`) не только делает тесты устойчивыми к изменениям вёрстки, но и косвенно проверяет доступность вашего приложения. Если `getByRole("button")` не находит кнопку — возможно, у вас проблема с a11y, а не с тестом.

### Приоритет query-методов

Чем ближе к тому, как пользователь находит элемент — тем лучше:

| Приоритет | Метод | Когда использовать |
|---|---|---|
| 1 | **getByRole** | Всегда, когда элемент имеет роль (кнопки, заголовки, ссылки) |
| 2 | **getByLabelText** | Для полей форм (связь с `<label>`) |
| 3 | **getByPlaceholderText** | Для полей без `<label>` (но лучше добавить label) |
| 4 | **getByText** | Для текстового контента (не интерактивные элементы) |
| 5 | **getByDisplayValue** | Для текущих значений полей ввода |
| 6 | **getByAltText** | Для изображений (альтернативный текст) |
| 7 | **getByTitle** | Для элементов с атрибутом `title` |
| 8 | **getByTestId** | Последний резерв (когда нет другого способа) |

### getByRole — самый надёжный

`getByRole` — рекомендуемый query по умолчанию. Роли (button, heading, link, textbox) — это семантическая информация, которую браузеры и скринридеры используют для описания элементов. Они не меняются при смене CSS-классов или структуры DOM, поэтому тесты остаются стабильными.

Параметр `name` позволяет уточнить поиск по доступному имени (текст кнопки, label, aria-label). Регулярные выражения с флагом `i` делают поиск регистронезависимым — это удобно, но не маскируйте реальные ошибки: если текст изменился, тест должен упасть.

```tsx
// Кнопки
screen.getByRole("button", { name: /submit/i });
screen.getByRole("button", { name: "Delete" });

// Заголовки
screen.getByRole("heading", { name: /welcome/i });
screen.getByRole("heading", { level: 1 }); // h1

// Ссылки
screen.getByRole("link", { name: /about/i });

// Поля ввода
screen.getByRole("textbox", { name: /search/i });
screen.getByRole("checkbox", { name: /agree/i });

// Списки
screen.getByRole("list");
screen.getByRole("listitem");

// Диалоги
screen.getByRole("dialog");
screen.getByRole("alertdialog");
```

### getByLabelText — для форм

`getByLabelText` находит элементы формы по их label. Это второй по приоритету query, потому что label — это то, что пользователь реально видит рядом с полем ввода. Связь может быть установлена тремя способами: через `htmlFor`/`id`, через вложенность или через `aria-label`.

Если `getByLabelText` не находит элемент — это сигнал, что у поля может не быть label. Вместо того чтобы переключаться на `getByTestId`, добавьте label в компонент: это улучшит и доступность, и тесты.

```tsx
// Связь через htmlFor
<label htmlFor="email">Email</label>
<input id="email" type="email" />

screen.getByLabelText(/email/i); // Находит input

// Вложенный label
<label>
  Email
  <input type="email" />
</label>

screen.getByLabelText(/email/i); // Тоже находит input

// aria-label
<input aria-label="Search" type="text" />
screen.getByLabelText(/search/i);
```

### getByText — для текста

`getByText` ищет элементы по их текстовому содержимому. Он подходит для проверки текстового контента (заголовки, сообщения, описания), но не для интерактивных элементов — для кнопок и ссылок используйте `getByRole`.

Регулярные выражения — мощный инструмент: `/hello/i` найдёт любой элемент, содержащий "hello" независимо от регистра. Функция-предикат нужна для сложных случаев, когда нужно проверить и текст, и тип элемента одновременно.

```tsx
// Точный текст
screen.getByText("Hello World");

// Регулярное выражение (частичное совпадение)
screen.getByText(/hello/i); // case-insensitive

// Функция (для сложной логики)
screen.getByText((content, element) => {
  return element.tagName.toLowerCase() === "p" && content.startsWith("Hello");
});
```

### queryBy* — проверка отсутствия

`queryBy*` — это единственный правильный способ проверить, что элемент **отсутствует** в DOM. В отличие от `getBy*`, он возвращает `null` вместо выбрасывания ошибки.

Главное правило: `getBy*` внутри `expect(...).not.toBeInTheDocument()` — это антипаттерн. `getBy*` выбросит ошибку до того, как `expect` успеет выполниться. Всегда используйте `queryBy*` для проверки отсутствия.

```tsx
// ✅ Элемент отсутствует
expect(screen.queryByText("Error")).not.toBeInTheDocument();

// ❌ getByText выбросит ошибку, если элемент не найден
expect(screen.getByText("Error")).not.toBeInTheDocument(); // FAIL — ошибка до expect
```

### findBy* — ожидание появления

`findBy*` возвращает Promise и ждёт появления элемента в DOM (по умолчанию до 1 секунды). Это необходимо для тестирования асинхронных операций: загрузки данных, анимаций, отложенного рендеринга.

Под капотом `findBy*` использует `waitFor`, повторяя запрос через короткие интервалы. Если элемент не появляется за таймаут — тест падает. Увеличивайте таймаут только для реально медленных операций; если вам постоянно нужен таймаут больше 3 секунд — скорее всего, проблема в архитектуре компонента.

```tsx
// Ждёт появления элемента (до 1 секунды по умолчанию)
const element = await screen.findByText("Loaded");

// С кастомным таймаутом
const element = await screen.findByText("Loaded", {}, { timeout: 3000 });

// Полезно для асинхронных операций
it("shows data after fetch", async () => {
  render(<DataComponent />);
  
  // Ждём появления данных
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

### findAllBy* — множественные элементы

`getAllBy*` и `findAllBy*` возвращают массив всех подходящих элементов. Используйте их, когда нужно проверить количество элементов (списки, таблицы) или найти все элементы определённого типа.

Если `getAllBy*` не находит ни одного элемента — тест упадёт. Если нужно проверить, что элементов может быть 0 или больше, используйте обёртку: проверьте длину массива через `queryAllBy*`.

```tsx
// Все элементы с ролью
const buttons = screen.getAllByRole("button");
expect(buttons).toHaveLength(3);

// Все элементы с текстом
const items = screen.getAllByText(/item/i);
expect(items).toHaveLength(5);

// Асинхронный поиск всех
const items = await screen.findAllByText(/loading/i);
```

---

## fireEvent vs userEvent

Выбор между `fireEvent` и `userEvent` — один из самых частых вопросов при написании тестов. `fireEvent` — это низкоуровневый инструмент: он просто диспатчит DOM-событие. `userEvent` — высокоуровневый: он симулирует реальное поведение пользователя, вызывая цепочку событий (focus → keydown → keypress → input → keyup → blur) так, как это делает браузер.

**Общее правило:** используйте `userEvent` по умолчанию. `fireEvent` — только когда `userEvent` не поддерживает нужное действие (например, `scroll`, `drag`) или когда вам нужна точная kontrolь над конкретным событием.

### fireEvent — низкоуровневые события

```tsx
import { fireEvent } from "@testing-library/react";

fireEvent.click(button);
fireEvent.change(input, { target: { value: "hello" } });
fireEvent.submit(form);
fireEvent.keyDown(input, { key: "Enter" });
```

**Недостатки:**
- Не симулирует реальное поведение пользователя (нет фокуса, нет промежуточных событий)
- Для `change` нужно вручную указывать `target.value`
- Не触发 `onBlur`, `onFocus` автоматически

### userEvent — реалистичные взаимодействия

```tsx
import userEvent from "@testing-library/user-event";

const user = userEvent.setup();

await user.click(button);
await user.type(input, "hello"); // Посимвольный ввод
await user.clear(input);
await user.tab(); // Переход по Tab
await user.keyboard("{Enter}");
await user.hover(element);
await user.unhover(element);
```

**Преимущества:**
- Симулирует реальное поведение пользователя
- `type()` вызывает `keydown`, `keypress`, `keyup` для каждого символа
- Автоматически управляет фокусом
- Рекомендуется для большинства случаев

### Когда использовать fireEvent

```tsx
// ✅ fireEvent — для простых событий
fireEvent.mouseEnter(element);
fireEvent.mouseLeave(element);
fireEvent.scroll(window);

// ✅ fireEvent — когда не нужен userEvent
fireEvent.dragStart(element);
fireEvent.drop(target);
```

### Сравнение

```tsx
// ❌ fireEvent — неестественный ввод
fireEvent.change(input, { target: { value: "hello" } });
// Не вызывает keydown/keyup, не перемещает курсор

// ✅ userEvent — реалистичный ввод
await user.type(input, "hello");
// Вызывает keydown, keypress, keyup для каждого символа
// Обновляет value, перемещает курсор
```

---

## Тестирование взаимодействий

Тесты взаимодействий проверяют, что компонент корректно реагирует на действия пользователя. Это самая ценная категория тестов: они моделируют реальные сценарии использования и дают максимальную уверенность в работоспособности кода.

Структура любого теста взаимодействия: **arrange** (подготовка — render), **act** (действие — click/type), **assert** (проверка — expect). Следуйте этому паттерну для читаемости.

### Клик по кнопке

```tsx
it("increments counter on button click", async () => {
  const user = userEvent.setup();
  render(<Counter />);
  
  expect(screen.getByText("Count: 0")).toBeInTheDocument();
  
  await user.click(screen.getByRole("button", { name: /increment/i }));
  
  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

### Ввод текста

```tsx
it("updates input value on typing", async () => {
  const user = userEvent.setup();
  render(<SearchInput />);
  
  const input = screen.getByRole("textbox", { name: /search/i });
  
  await user.type(input, "react");
  
  expect(input).toHaveValue("react");
});
```

### Отправка формы

```tsx
it("submits form on Enter", async () => {
  const user = userEvent.setup();
  const handleSubmit = vi.fn();
  
  render(<SearchForm onSubmit={handleSubmit} />);
  
  const input = screen.getByRole("textbox", { name: /search/i });
  
  await user.type(input, "react{enter}");
  
  expect(handleSubmit).toHaveBeenCalledWith("react");
});
```

### Переключение чекбокса

```tsx
it("toggles checkbox", async () => {
  const user = userEvent.setup();
  render(<AgreementCheckbox />);
  
  const checkbox = screen.getByRole("checkbox", { name: /agree/i });
  
  expect(checkbox).not.toBeChecked();
  
  await user.click(checkbox);
  
  expect(checkbox).toBeChecked();
  
  await user.click(checkbox);
  
  expect(checkbox).not.toBeChecked();
});
```

### Выбор из select

```tsx
it("selects option from dropdown", async () => {
  const user = userEvent.setup();
  render(<CountrySelect />);
  
  const select = screen.getByRole("combobox", { name: /country/i });
  
  await user.selectOptions(select, "us");
  
  expect(select).toHaveValue("us");
});
```

### Навигация по Tab

```tsx
it("navigates through form fields with Tab", async () => {
  const user = userEvent.setup();
  render(<LoginForm />);
  
  const emailInput = screen.getByRole("textbox", { name: /email/i });
  const passwordInput = screen.getByRole("textbox", { name: /password/i });
  const submitButton = screen.getByRole("button", { name: /login/i });
  
  await user.tab();
  expect(emailInput).toHaveFocus();
  
  await user.tab();
  expect(passwordInput).toHaveFocus();
  
  await user.tab();
  expect(submitButton).toHaveFocus();
});
```

---

## Тестирование форм

Формы — один из самых важных объектов тестирования, потому что они напрямую связаны с бизнес-логикой: валидация, отправка данных, сброс состояния. Стратегия тестирования форм строится вокруг трёх сценариев: успешная отправка, валидация ошибок и сброс после отправки.

Не мокайте `onChange`-обработчики — это тестирование реализации. Вместо этого вводите данные через `user.type()` и проверяйте результат: значение поля, вызов `onSubmit`, появление ошибок валидации.

### Базовая форма

```tsx
function LoginForm({ onSubmit }: { onSubmit: (data: { email: string; password: string }) => void }) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSubmit({ email, password });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      
      <label htmlFor="password">Password</label>
      <input
        id="password"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      
      <button type="submit">Login</button>
    </form>
  );
}
```

### Тест формы

```tsx
it("submits form with email and password", async () => {
  const user = userEvent.setup();
  const handleSubmit = vi.fn();
  
  render(<LoginForm onSubmit={handleSubmit} />);
  
  await user.type(screen.getByLabelText(/email/i), "user@example.com");
  await user.type(screen.getByLabelText(/password/i), "secret123");
  await user.click(screen.getByRole("button", { name: /login/i }));
  
  expect(handleSubmit).toHaveBeenCalledWith({
    email: "user@example.com",
    password: "secret123",
  });
});
```

### Валидация формы

```tsx
it("shows validation error for empty email", async () => {
  const user = userEvent.setup();
  render(<LoginForm onSubmit={vi.fn()} />);
  
  await user.click(screen.getByRole("button", { name: /login/i }));
  
  expect(screen.getByText(/email is required/i)).toBeInTheDocument();
});
```

### Сброс формы

```tsx
it("clears form after successful submission", async () => {
  const user = userEvent.setup();
  render(<LoginForm onSubmit={vi.fn()} />);
  
  await user.type(screen.getByLabelText(/email/i), "user@example.com");
  await user.click(screen.getByRole("button", { name: /login/i }));
  
  expect(screen.getByLabelText(/email/i)).toHaveValue("");
  expect(screen.getByLabelText(/password/i)).toHaveValue("");
});
```

---

## Тестирование асинхронных компонентов

Асинхронные компоненты — источник большинства проблем в тестировании. React не ждёт завершения асинхронных операций, и если вы попытаетесь проверить результат синхронно — тест упадёт с ошибкой "element not found".

Существует три подхода к тестированию асинхронности:
1. **`findBy*`** — самый простой, ждёт появления элемента автоматически.
2. **`waitFor`** — более гибкий, позволяет ждать произвольное условие.
3. **`waitForElementToBeRemoved`** — ждёт исчезновения элемента (например, спиннера).

Для мокирования сетевых запросов лучше всего использовать **MSW (Mock Service Worker)** — он перехватывает запросы на уровне Service Worker, не требуя изменения кода компонента. Альтернатива — `vi.spyOn` на `fetch`, но это менее реалистично.

### Загрузка данных

```tsx
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  
  useEffect(() => {
    fetchUsers()
      .then(setUsers)
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, []);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Тест с MSW (Mock Service Worker)

```tsx
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";

const server = setupServer(
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "Alice" },
      { id: 2, name: "Bob" },
    ]);
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

it("displays user list after loading", async () => {
  render(<UserList />);
  
  // Сначала показывается loading
  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  
  // Ждём появления данных
  expect(await screen.findByText("Alice")).toBeInTheDocument();
  expect(screen.getByText("Bob")).toBeInTheDocument();
  
  // Loading исчезает
  expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
});
```

### Тест обработки ошибки

```tsx
it("displays error message on fetch failure", async () => {
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json({ error: "Network error" }, { status: 500 });
    })
  );
  
  render(<UserList />);
  
  expect(await screen.findByText(/network error/i)).toBeInTheDocument();
  expect(screen.queryByText("Alice")).not.toBeInTheDocument();
});
```

### waitFor — явное ожидание

```tsx
import { waitFor } from "@testing-library/react";

it("updates UI after async operation", async () => {
  render(<AsyncComponent />);
  
  await waitFor(() => {
    expect(screen.getByText(/success/i)).toBeInTheDocument();
  });
});
```

### waitForElementToBeRemoved — ожидание исчезновения

```tsx
import { waitForElementToBeRemoved } from "@testing-library/react";

it("removes loading indicator after data loads", async () => {
  render(<DataComponent />);
  
  const loading = screen.getByText(/loading/i);
  
  await waitForElementToBeRemoved(loading);
  
  expect(screen.getByText("Data loaded")).toBeInTheDocument();
});
```

---

## Тестирование с провайдерами

Многие компоненты зависят от контекста: тема, локализация, авторизация, роутинг, состояние Redux. Без оборачивания в соответствующие провайдеры такие компоненты не отрендерятся — React выбросит ошибку.

Стратегия: создайте переиспользуемые render-обёртки, которые инкапсулируют все необходимые провайдеры. Это устранит дублирование и обеспечит консистентность тестов. Но не оборачивайте в провайдеры "на всякий случай" — добавляйте только те, которые реально нужны тестируемому компоненту.

### Context Provider

```tsx
const ThemeContext = createContext("light");

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={`btn-${theme}`}>Click me</button>;
}

// Тест с провайдером
it("renders button with theme from context", () => {
  render(
    <ThemeContext value="dark">
      <ThemedButton />
    </ThemeContext>
  );
  
  const button = screen.getByRole("button");
  expect(button).toHaveClass("btn-dark");
});
```

### Кастомная render-функция с провайдером

```tsx
function renderWithTheme(ui: React.ReactElement, theme: string = "light") {
  return render(
    <ThemeContext value={theme}>
      {ui}
    </ThemeContext>
  );
}

it("renders with dark theme", () => {
  renderWithTheme(<ThemedButton />, "dark");
  expect(screen.getByRole("button")).toHaveClass("btn-dark");
});
```

### React Router

```tsx
import { MemoryRouter, Route, Routes } from "react-router-dom";

function renderWithRouter(ui: React.ReactElement, { route = "/" } = {}) {
  return render(
    <MemoryRouter initialEntries={[route]}>
      <Routes>
        <Route path="*" element={ui} />
      </Routes>
    </MemoryRouter>
  );
}

it("navigates to home page", () => {
  renderWithRouter(<HomePage />, { route: "/" });
  expect(screen.getByText("Welcome home")).toBeInTheDocument();
});

it("navigates to about page", () => {
  renderWithRouter(<AboutPage />, { route: "/about" });
  expect(screen.getByText("About us")).toBeInTheDocument();
});
```

### Redux Provider

```tsx
import { configureStore } from "@reduxjs/toolkit";
import { Provider } from "react-redux";

function renderWithRedux(ui: React.ReactElement, store = configureStore({ reducer: {} })) {
  return render(<Provider store={store}>{ui}</Provider>);
}

it("renders with Redux store", () => {
  const store = configureStore({
    reducer: { counter: counterReducer },
    preloadedState: { counter: { value: 42 } },
  });
  
  renderWithRedux(<Counter />, store);
  
  expect(screen.getByText("Count: 42")).toBeInTheDocument();
});
```

### Несколько провайдеров

```tsx
function AllProviders({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider>
      <AuthProvider>
        <RouterProvider>
          <QueryClientProvider client={queryClient}>
            {children}
          </QueryClientProvider>
        </RouterProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

function renderWithAllProviders(ui: React.ReactElement) {
  return render(<AllProviders>{ui}</AllProviders>);
}
```

---

## renderHook — тестирование хуков

`renderHook` позволяет тестировать кастомные хуки изолированно, без рендеринга компонента-обёртки. Это полезно, когда хук содержит сложную логику (таймеры, debouncing, fetch), которую неудобно тестировать через UI.

**Когда использовать `renderHook`:**
- Хук не привязан к конкретному компоненту (переиспользуемый)
- Логика хука сложна и требует детальной проверки промежуточных состояний
- Нужно протестировать хук с разными начальными значениями

**Когда НЕ использовать:**
- Хук простой (один `useState`) — тестируйте его через компонент
- Логика хука тесно связана с UI — тестируйте через компонент

### Базовое использование

```tsx
import { renderHook, act } from "@testing-library/react";

it("useCounter increments", () => {
  const { result } = renderHook(() => useCounter());
  
  expect(result.current.count).toBe(0);
  
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
});
```

### Хуки с начальными значениями

```tsx
it("useCounter starts with initial value", () => {
  const { result } = renderHook(() => useCounter(10));
  
  expect(result.current.count).toBe(10);
});
```

### Хуки с провайдерами

```tsx
it("useTheme returns theme from context", () => {
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <ThemeContext value="dark">
      {children}
    </ThemeContext>
  );
  
  const { result } = renderHook(() => useTheme(), { wrapper });
  
  expect(result.current.theme).toBe("dark");
});
```

### Асинхронные хуки

```tsx
it("useFetch loads data", async () => {
  const { result } = renderHook(() => useFetch("/api/user/1"));
  
  expect(result.current.isLoading).toBe(true);
  
  await waitFor(() => {
    expect(result.current.isLoading).toBe(false);
  });
  
  expect(result.current.data).toEqual({ id: 1, name: "Alice" });
});
```

### rerender хука

```tsx
it("useDebounce updates after delay", async () => {
  const { result, rerender } = renderHook(
    ({ value }) => useDebounce(value, 500),
    { initialProps: { value: "initial" } }
  );
  
  expect(result.current).toBe("initial");
  
  rerender({ value: "updated" });
  
  expect(result.current).toBe("initial"); // Ещё не обновилось
  
  await waitFor(() => {
    expect(result.current).toBe("updated");
  });
});
```

---

## Тестирование условного рендеринга

Условный рендеринг — одна из ключевых возможностей React: компоненты отображаются или скрываются в зависимости от состояния. Тесты должны проверять оба сценария: когда условие `true` и когда `false`.

Главный инструмент — `queryBy*` для проверки отсутствия и `getBy*` для проверки присутствия. Никогда не проверяйте условный рендеринг через `container.innerHTML` или CSS-классы — это детали реализации.

### Условный рендеринг с &&

```tsx
function Alert({ message, isVisible }: { message: string; isVisible: boolean }) {
  return isVisible ? <div role="alert">{message}</div> : null;
}

it("shows alert when isVisible is true", () => {
  render(<Alert message="Success!" isVisible={true} />);
  expect(screen.getByRole("alert")).toHaveTextContent("Success!");
});

it("hides alert when isVisible is false", () => {
  render(<Alert message="Success!" isVisible={false} />);
  expect(screen.queryByRole("alert")).not.toBeInTheDocument();
});
```

### Условный рендеринг с тернарным оператором

```tsx
function Dashboard({ isLoggedIn }: { isLoggedIn: boolean }) {
  return isLoggedIn ? <WelcomeBack /> : <LoginPrompt />;
}

it("shows welcome message for logged-in users", () => {
  render(<Dashboard isLoggedIn={true} />);
  expect(screen.getByText(/welcome back/i)).toBeInTheDocument();
  expect(screen.queryByText(/please log in/i)).not.toBeInTheDocument();
});

it("shows login prompt for anonymous users", () => {
  render(<Dashboard isLoggedIn={false} />);
  expect(screen.getByText(/please log in/i)).toBeInTheDocument();
  expect(screen.queryByText(/welcome back/i)).not.toBeInTheDocument();
});
```

---

## Тестирование списков

Списки — частый паттерн в UI. Тестирование списков сводится к трём сценариям: рендеринг известного набора элементов, обработка пустого списка и динамические изменения (фильтрация, добавление, удаление).

Для проверки количества элементов используйте `getAllByRole("listitem")` с `toHaveLength()`. Для проверки содержимого — `getByText()`. Избегайте обращения к элементам по индексу (`items[0]`) — это хрупкий подход, зависящий от порядка рендеринга.

### Рендеринг списка

```tsx
function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}

it("renders list of users", () => {
  const users = [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" },
    { id: 3, name: "Charlie" },
  ];
  
  render(<UserList users={users} />);
  
  expect(screen.getAllByRole("listitem")).toHaveLength(3);
  expect(screen.getByText("Alice")).toBeInTheDocument();
  expect(screen.getByText("Bob")).toBeInTheDocument();
  expect(screen.getByText("Charlie")).toBeInTheDocument();
});
```

### Пустой список

```tsx
it("shows message for empty list", () => {
  render(<UserList users={[]} />);
  
  expect(screen.queryByRole("listitem")).not.toBeInTheDocument();
  expect(screen.getByText(/no users found/i)).toBeInTheDocument();
});
```

### Фильтрация списка

```tsx
it("filters users by search query", async () => {
  const user = userEvent.setup();
  const users = [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" },
    { id: 3, name: "Charlie" },
  ];
  
  render(<FilterableUserList users={users} />);
  
  const searchInput = screen.getByRole("textbox", { name: /search/i });
  
  await user.type(searchInput, "ali");
  
  expect(screen.getByText("Alice")).toBeInTheDocument();
  expect(screen.queryByText("Bob")).not.toBeInTheDocument();
  expect(screen.queryByText("Charlie")).not.toBeInTheDocument();
});
```

---

## Тестирование Error Boundaries

Error Boundary — это компонент, который перехватывает ошибки JavaScript в дереве компонентов и показывает fallback UI. Без Error Boundary любое исключение в React приведёт к размонтированию всего приложения.

Тестирование Error Boundaries имеет особенность: React намеренно логирует ошибки в консоль при их перехвате. Чтобы тесты не засоряли вывод, используйте `vi.spyOn(console, "error").mockImplementation(() => {})`. Не забывайте восстанавливать консоль через `mockRestore()`, иначе другие тесты могут молча проглотить реальные ошибки.

### Error Boundary компонент

```tsx
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

### Тест Error Boundary

```tsx
function BrokenComponent() {
  throw new Error("Something went wrong");
}

it("shows fallback when child throws error", () => {
  // Подавляем ошибки в консоли
  const consoleSpy = vi.spyOn(console, "error").mockImplementation(() => {});
  
  render(
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <BrokenComponent />
    </ErrorBoundary>
  );
  
  expect(screen.getByText(/something went wrong/i)).toBeInTheDocument();
  
  consoleSpy.mockRestore();
});
```

---

## Кастомные render-обёртки

Когда несколько компонентов зависят от одних и тех же провайдеров (тема, роутер, store), дублирование кода в тестах быстро становится проблемой. Кастомная render-обёртка решает это: вы создаёте функцию, которая автоматически оборачивает компонент во все необходимые провайдеры.

Паттерн: создайте файл `test-utils.ts`, экспортируйте кастомный `render` и импортируйте его вместо `@testing-library/react` во всех тестах. Это также позволяет централизованно менять конфигурацию тестового окружения.

### Переиспользуемая render-функция

```tsx
// test-utils.ts
import { render, RenderOptions } from "@testing-library/react";
import { ThemeProvider } from "./ThemeProvider";
import { AuthProvider } from "./AuthProvider";

function AllProviders({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider>
      <AuthProvider>
        {children}
      </AuthProvider>
    </ThemeProvider>
  );
}

function customRender(ui: React.ReactElement, options?: Omit<RenderOptions, "wrapper">) {
  return render(ui, { wrapper: AllProviders, ...options });
}

export * from "@testing-library/react";
export { customRender as render };
```

### Использование кастомного render

```tsx
// MyComponent.test.tsx
import { render, screen } from "../test-utils";

it("renders with all providers", () => {
  render(<MyComponent />);
  expect(screen.getByText("Hello")).toBeInTheDocument();
});
```

---

## Лучшие практики

Следование лучшим практикам делает тесты устойчивыми к рефакторингу, читаемыми и поддерживаемыми. Главное правило: **тесты должны проверять поведение, а не реализацию**. Если тест ломается при изменении кода, но поведение компонента не изменилось — тест написан неправильно.

Эти практики не догма, а руководство. В некоторых случаях от них можно отступить — но только осознанно и с пониманием trade-offs.

### 1. Используйте getByRole вместо querySelector

```tsx
// ❌ Хрупкий селектор
expect(container.querySelector(".btn-primary")).toBeInTheDocument();

// ✅ Семантический селектор
expect(screen.getByRole("button", { name: /submit/i })).toBeInTheDocument();
```

### 2. Тестируйте поведение, а не реализацию

```tsx
// ❌ Тестирует state
expect(component.state.isOpen).toBe(true);

// ✅ Тестирует видимое поведение
expect(screen.getByRole("dialog")).toBeInTheDocument();
```

### 3. Используйте userEvent вместо fireEvent

```tsx
// ❌ fireEvent — неестественный ввод
fireEvent.change(input, { target: { value: "hello" } });

// ✅ userEvent — реалистичный ввод
await user.type(input, "hello");
```

### 4. Один тест — одна проверка

```tsx
// ❌ Один тест проверяет слишком много
it("handles full user flow", async () => {
  // Логин, навигация, изменение настроек, выход
});

// ✅ Разбито на отдельные тесты
it("shows login form", () => { ... });
it("logs in with valid credentials", async () => { ... });
it("redirects to dashboard after login", async () => { ... });
```

### 5. Используйте factory-функции для тестовых данных

```tsx
const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  ...overrides,
});

it("displays user name", () => {
  const user = createUser({ name: "Alice" });
  render(<UserProfile user={user} />);
  expect(screen.getByText("Alice")).toBeInTheDocument();
});
```

### 6. Очищайте моки между тестами

```tsx
afterEach(() => {
  vi.clearAllMocks();
  vi.restoreAllMocks();
});
```

### 7. Используйте waitFor для асинхронных операций

```tsx
// ❌ Синхронная проверка асинхронного результата
expect(screen.getByText("Loaded")).toBeInTheDocument(); // FAIL

// ✅ Ожидание через waitFor
await waitFor(() => {
  expect(screen.getByText("Loaded")).toBeInTheDocument();
});

// ✅ Или через findBy
expect(await screen.findByText("Loaded")).toBeInTheDocument();
```

---

## Антипаттерны

Антипаттерны — это распространённые ошибки, которые делают тесты хрупкими, медленными и бесполезными. Хрупкий тест — это тест, который падает при рефакторинге, даже если поведение компонента не изменилось. Такие тесты создают ложное чувство уверенности и замедляют разработку.

Запомните: **плохой тест хуже, чем его отсутствие**. Тест, который постоянно падает без причины, разработчики быстро начнут игнорировать или удалять. Лучше иметь меньше тестов, но надёжных.

### 1. Тестирование деталей реализации

```tsx
// ❌ Зависит от внутреннего state
expect(component.state.count).toBe(1);

// ✅ Зависит от видимого поведения
expect(screen.getByText("Count: 1")).toBeInTheDocument();
```

### 2. Использование querySelector

```tsx
// ❌ Хрупкий CSS-селектор
expect(container.querySelector("div > button.btn")).toBeInTheDocument();

// ✅ Семантический query
expect(screen.getByRole("button", { name: /submit/i })).toBeInTheDocument();
```

### 3. Snapshot-тестирование всего

Snapshot-тесты автоматически проверяют, что UI не изменился. Vitest сохраняет текущий рендер и сравнивает будущие с ним. Полезны для стабильных компонентов (иконки, базовые UI-элементы). Вредны для компонентов с бизнес-логикой или динамическими данными — snapshot ломается при каждом изменении и даёт ложную уверенность.

```tsx
// ❌ Snapshot ломается при каждом изменении разметки
expect(container).toMatchSnapshot();

// ✅ Snapshot только для стабильных компонентов
expect(screen.getByRole("button")).toMatchSnapshot();
```

### 4. Зависимость тестов друг от друга

```tsx
// ❌ Второй тест зависит от первого
it("creates user", () => {
  createUser("Alice");
  expect(getUsers()).toHaveLength(1);
});

it("finds user", () => {
  // ❌ Alice создана в предыдущем тесте
  expect(findUser("Alice")).not.toBeNull();
});

// ✅ Каждый тест независим
it("finds user", () => {
  createUser("Alice");
  expect(findUser("Alice")).not.toBeNull();
});
```

### 5. Игнорирование async/await

```tsx
// ❌ Синхронная проверка асинхронного результата
it("loads data", () => {
  render(<DataComponent />);
  expect(screen.getByText("Loaded")).toBeInTheDocument(); // FAIL
});

// ✅ Ожидание через findBy
it("loads data", async () => {
  render(<DataComponent />);
  expect(await screen.findByText("Loaded")).toBeInTheDocument();
});
```

### 6. Избыточное мокирование

```tsx
// ❌ Мокирует всё
vi.mock("react", () => ({ useState: vi.fn() }));
vi.mock("./utils", () => ({ format: vi.fn() }));

// ✅ Мокирует только внешние зависимости
server.use(http.get("/api/data", () => HttpResponse.json({ data: [] })));
```

### 7. Тесты без assertions

```tsx
// ❌ Нет expect — тест всегда проходит
it("renders component", () => {
  render(<MyComponent />);
});

// ✅ Есть проверка
it("renders component", () => {
  render(<MyComponent />);
  expect(screen.getByText("Hello")).toBeInTheDocument();
});
```
