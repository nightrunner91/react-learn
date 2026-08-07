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

### Базовая установка

```bash
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

- `@testing-library/react` — основная библиотека (render, screen, fireEvent)
- `@testing-library/jest-dom` — дополнительные matcher'ы (toBeInTheDocument, toHaveTextContent)
- `@testing-library/user-event` — более реалистичные пользовательские события (рекомендуется вместо fireEvent)

### Настройка Vitest

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

```tsx
it("debug example", () => {
  render(<MyComponent />);
  
  screen.debug(); // Выводит весь DOM
  screen.debug(screen.getByRole("button")); // Выводит конкретный элемент
});
```

---

## screen — поиск элементов

`screen` — это объект, предоставляющий все query-методы для поиска элементов в DOM.

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

```tsx
// ✅ Элемент отсутствует
expect(screen.queryByText("Error")).not.toBeInTheDocument();

// ❌ getByText выбросит ошибку, если элемент не найден
expect(screen.getByText("Error")).not.toBeInTheDocument(); // FAIL — ошибка до expect
```

### findBy* — ожидание появления

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
