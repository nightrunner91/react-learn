# Unit-тестирование с Vitest

## Содержание

1. [Что такое Vitest](#что-такое-vitest)
2. [Установка и настройка](#установка-и-настройка)
3. [Базовый API: describe, it, expect](#базовый-api-describe-it-expect)
4. [Matchers — проверки значений](#matchers--проверки-значений)
5. [Асинхронные тесты](#асинхронные-тесты)
6. [Мокирование: vi.fn, vi.spyOn, vi.mock](#мокирование-vifn-vispyon-vimock)
7. [Фейковые таймеры](#фейковые-таймеры)
8. [Snapshot-тестирование](#snapshot-тестирование)
9. [Параметризованные тесты](#параметризованные-тесты)
10. [Тестирование хуков](#тестирование-хуков)
11. [Покрытие кода](#покрытие-кода)
12. [Конфигурация для React и Next.js](#конфигурация-для-react-и-nextjs)
13. [Vitest vs Jest](#vitest-vs-jest)
14. [Лучшие практики](#лучшие-практики)

---

## Что такое Vitest

**Vitest** — это фреймворк для тестирования, созданный командой Vite. Он предоставляет API, совместимый с Jest, но работает нативно с Vite — использует те же трансформации, конфигурацию и систему модулей.

### Почему Vitest, а не Jest

| Характеристика | Vitest | Jest |
|---|---|---|
| **Интеграция с Vite** | Нативная — одна конфигурация | Требует отдельную настройку (babel, ts-jest) |
| **ESM** | Поддержка из коробки | Частичная, требует экспериментальных флагов |
| **TypeScript** | Работает без дополнительной настройки | Требует ts-jest или babel-jest |
| **Скорость** | Быстрый (использует Vite-кэш) | Средний |
| **Watch mode** | Нативный HMR-режим | File-based watch |
| **Snapshot** | Совместим с Jest snapshots | Jest snapshots |
| **API** | Совместим с Jest | Оригинальный API |
| **Coverage** | Встроенный (v8, istanbul) | Встроенный (v8, babel) |

> 💡 **Правило:** Если проект на Vite — используйте Vitest. Если проект на Webpack — Jest остаётся хорошим выбором. Миграция с Jest на Vitest обычно безболезненна благодаря совместимости API.

---

## Установка и настройка

### Базовая установка

```bash
npm install -D vitest
```

### Конфигурация

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["**/*.{test,spec}.{ts,tsx}"],
    exclude: ["node_modules", "dist"],
  },
});
```

### Для React-проекта

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
    css: true,
    include: ["src/**/*.{test,spec}.{ts,tsx}"],
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

### Расширенная конфигурация

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
    css: true,

    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html"],
      include: ["src/**/*.{ts,tsx}"],
      exclude: [
        "src/test/**",
        "src/**/*.d.ts",
        "src/**/*.stories.tsx",
      ],
      thresholds: {
        statements: 80,
        branches: 70,
        functions: 85,
        lines: 80,
      },
    },

    alias: {
      "@": "/src",
    },
  },
  resolve: {
    alias: {
      "@": "/src",
    },
  },
});
```

### Скрипты в package.json

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage",
    "test:watch": "vitest watch"
  }
}
```

---

## Базовый API: describe, it, expect

### describe — группировка тестов

```ts
describe("calculateDiscount", () => {
  describe("for premium users", () => {
    it("returns 10% discount", () => {
      expect(calculateDiscount(100, true)).toBe(10);
    });
  });

  describe("for regular users", () => {
    it("returns no discount", () => {
      expect(calculateDiscount(100, false)).toBe(0);
    });
  });
});
```

### it / test — определение теста

```ts
it("returns sum of two numbers", () => {
  expect(add(2, 3)).toBe(5);
});

// test — алиас для it
test("returns sum of two numbers", () => {
  expect(add(2, 3)).toBe(5);
});
```

### expect — проверки

```ts
expect(value).toBe(42);           // Строгое равенство (===)
expect(value).toEqual({ a: 1 });  // Глубокое равенство
expect(value).toBeTruthy();       // truthy-значение
expect(value).toBeDefined();      // !== undefined
expect(value).toBeNull();         // === null
expect(value).toBeUndefined();    // === undefined
```

### Хуки жизненного цикла

```ts
describe("UserRepository", () => {
  let repo: UserRepository;

  beforeAll(async () => {
    // Выполняется один раз перед всеми тестами
    await initDatabase();
  });

  afterAll(async () => {
    // Выполняется один раз после всех тестов
    await closeDatabase();
  });

  beforeEach(() => {
    // Выполняется перед каждым тестом
    repo = new UserRepository();
  });

  afterEach(() => {
    // Выполняется после каждого теста
    vi.clearAllMocks();
  });

  it("creates user", () => {
    const user = repo.create({ name: "Alice" });
    expect(user.name).toBe("Alice");
  });
});
```

### Порядок выполнения

```
beforeAll
  beforeEach
    it("test 1")
  afterEach
  beforeEach
    it("test 2")
  afterEach
afterAll
```

---

## Matchers — проверки значений

### Равенство

```ts
// Примитивы
expect(42).toBe(42);
expect("hello").toBe("hello");
expect(true).toBe(true);

// null, undefined
expect(null).toBeNull();
expect(undefined).toBeUndefined();
expect(42).not.toBeUndefined();

// Объекты и массивы (глубокое сравнение)
expect({ a: 1, b: 2 }).toEqual({ a: 1, b: 2 });
expect([1, 2, 3]).toEqual([1, 2, 3]);

// Ссылочное равенство
const obj = { a: 1 };
expect(obj).toBe(obj);
expect(obj).not.toBe({ a: 1 }); // Другой объект — другая ссылка
```

### Числа

```ts
expect(42).toBeGreaterThan(40);
expect(42).toBeGreaterThanOrEqual(42);
expect(42).toBeLessThan(50);
expect(42).toBeLessThanOrEqual(42);
expect(0.1 + 0.2).toBeCloseTo(0.3, 5); // Для float-сравнений
expect(NaN).toBeNaN();
```

### Строки

```ts
expect("hello world").toContain("world");
expect("hello world").toMatch(/hello/);
expect("hello").toHaveLength(5);
expect("Hello World").toMatch(/^hello$/i);
```

### Массивы

```ts
expect([1, 2, 3]).toContain(2);
expect([1, 2, 3]).toContainEqual({ id: 1 });
expect([1, 2, 3]).toHaveLength(3);
expect([1, 2, 3]).toEqual(expect.arrayContaining([1, 2]));
```

### Объекты

```ts
expect({ a: 1, b: 2, c: 3 }).toHaveProperty("a");
expect({ a: 1, b: 2, c: 3 }).toHaveProperty("a", 1);
expect({ a: { b: 1 } }).toHaveProperty("a.b", 1);

expect({ a: 1, b: 2 }).toEqual(
  expect.objectContaining({ a: 1 }) // Может содержать дополнительные поля
);
```

### Boolean и типы

```ts
expect(true).toBeTruthy();
expect(false).toBeFalsy();
expect(1).toBeTruthy();
expect(0).toBeFalsy();
expect("").toBeFalsy();
expect(null).toBeFalsy();
expect(undefined).toBeFalsy();

expect("hello").toBeInstanceOf(String);
expect([1, 2]).toBeInstanceOf(Array);
expect(new Date()).toBeInstanceOf(Date);
```

### Исключения

```ts
expect(() => {
  throw new Error("boom");
}).toThrow("boom");

expect(() => {
  throw new TypeError("wrong type");
}).toThrow(TypeError);

expect(() => {
  throw new Error("boom");
}).toThrow(/boom/);

// Асинхронные исключения
await expect(asyncFunction()).rejects.toThrow("error message");
await expect(asyncFunction()).resolves.toBe(42);
```

### DOM (с @testing-library/jest-dom)

```ts
expect(element).toBeInTheDocument();
expect(element).toBeVisible();
expect(element).toBeEmpty();
expect(element).toBeDisabled();
expect(element).toBeEnabled();
expect(element).toBeChecked();
expect(element).toBeRequired();
expect(element).toBeValid();
expect(element).toBeInvalid();

expect(element).toHaveTextContent("Hello");
expect(element).toHaveTextContent(/Hello/);
expect(element).toHaveAttribute("href", "/about");
expect(element).toHaveClass("active");
expect(element).toHaveStyle("color: red");
expect(element).toHaveFocus();
expect(element).toHaveValue("hello");
expect(element).toHaveFormValues({ name: "Alice" });

expect(element).toHaveAccessibleName("Search");
expect(element).toHaveAccessibleDescription("Find products");
expect(element).toHaveRole("button");
```

---

## Асинхронные тесты

### async/await

```ts
it("fetches user data", async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe("Alice");
});
```

### Промисы

```ts
it("fetches user data", () => {
  return fetchUser(1).then((user) => {
    expect(user.name).toBe("Alice");
  });
});
```

### resolves / rejects

```ts
it("resolves with user data", async () => {
  await expect(fetchUser(1)).resolves.toEqual({
    id: 1,
    name: "Alice",
  });
});

it("rejects with error for unknown user", async () => {
  await expect(fetchUser(999)).rejects.toThrow("User not found");
});
```

### done-колбэк (редко используется)

```ts
it("subscribes to events", (done) => {
  const callback = (data: string) => {
    expect(data).toBe("hello");
    done();
  };
  subscribe(callback);
});
```

> 💡 **Рекомендация:** Используйте async/await вместо done-колбэка. Это чище и понятнее.

### Таймауты

```ts
it("takes long to process", async () => {
  const result = await slowOperation();
  expect(result).toBe("done");
}, 10000); // Таймаут 10 секунд для этого теста
```

---

## Мокирование: vi.fn, vi.spyOn, vi.mock

### vi.fn() — создание mock-функции

```ts
const mockFn = vi.fn();

mockFn("hello");
mockFn("world");

expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith("hello");
expect(mockFn).toHaveBeenLastCalledWith("world");
```

### vi.fn() с реализацией

```ts
// Mock с возвращаемым значением
const mockAdd = vi.fn((a: number, b: number) => a + b);
expect(mockAdd(2, 3)).toBe(5);

// Mock с последовательными возвращаемыми значениями
const mockFn = vi.fn()
  .mockReturnValueOnce("first")
  .mockReturnValueOnce("second")
  .mockReturnValue("default");

expect(mockFn()).toBe("first");
expect(mockFn()).toBe("second");
expect(mockFn()).toBe("default");
expect(mockFn()).toBe("default");

// Mock с async-реализацией
const mockFetch = vi.fn().mockResolvedValue({ data: [1, 2, 3] });
const result = await mockFetch();
expect(result.data).toEqual([1, 2, 3]);

// Mock с отклонением
const mockFetch = vi.fn().mockRejectedValue(new Error("Network error"));
await expect(mockFetch()).rejects.toThrow("Network error");
```

### vi.spyOn() — шпионаж за методом

```ts
const consoleSpy = vi.spyOn(console, "log");

doSomething();

expect(consoleSpy).toHaveBeenCalledWith("Processing...");

consoleSpy.mockRestore(); // Восстановить оригинальный метод
```

### vi.spyOn() для переопределения

```ts
const user = {
  getName: () => "Alice",
};

const spy = vi.spyOn(user, "getName").mockReturnValue("Bob");

expect(user.getName()).toBe("Bob");

spy.mockRestore();
expect(user.getName()).toBe("Alice");
```

### vi.mock() — мокирование модулей

```ts
// Полное мокирование модуля
vi.mock("./api", () => ({
  fetchUsers: vi.fn().mockResolvedValue([{ id: 1, name: "Alice" }]),
  deleteUser: vi.fn().mockResolvedValue(true),
}));

// Использование в тесте
import { fetchUsers } from "./api";

it("loads users", async () => {
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

### vi.mock() с factory-функцией (частичный mock)

```ts
vi.mock("./api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("./api")>();
  return {
    ...actual,
    fetchUsers: vi.fn().mockResolvedValue([{ id: 1, name: "Mocked" }]),
    // deleteUser остаётся оригинальным
  };
});
```

### Автоматическое мокирование

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    setupFiles: ["./src/test/setup.ts"],
  },
});

// src/test/setup.ts — моки по умолчанию
vi.mock("./api", () => ({
  fetchUsers: vi.fn().mockResolvedValue([]),
}));
```

### vi.hoisted() — переменные для vi.mock

```ts
// vi.mock вызывается до импортов, поэтому обычные переменные недоступны
// vi.hoisted() решает эту проблему

const { mockFetchUsers, mockDeleteUser } = vi.hoisted(() => ({
  mockFetchUsers: vi.fn(),
  mockDeleteUser: vi.fn(),
}));

vi.mock("./api", () => ({
  fetchUsers: mockFetchUsers,
  deleteUser: mockDeleteUser,
}));

it("fetches users", async () => {
  mockFetchUsers.mockResolvedValue([{ id: 1 }]);
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1 }]);
});
```

### Очистка моков

```ts
afterEach(() => {
  vi.clearAllMocks();     // Очищает историю вызовов всех моков
  vi.restoreAllMocks();   // Восстанавливает оригинальные реализации
});

// Или для конкретного мока:
mockFn.mockClear();       // Очищает историю вызовов
mockFn.mockReset();       // Очищает + убирает реализацию
mockFn.mockRestore();     // Reset + восстанавливает оригинал (только для spyOn)
```

### Разница между clear, reset, restore

| Метод | История вызовов | Реализация | Оригинал (spyOn) |
|---|---|---|---|
| `mockClear()` | Очищает | Сохраняет | Сохраняет |
| `mockReset()` | Очищает | Убирает (→ undefined) | Сохраняет |
| `mockRestore()` | Очищает | Убирает | Восстанавливает |

---

## Фейковые таймеры

### Зачем нужны

Тесты не должны ждать реального времени. Фейковые таймеры позволяют контролировать `setTimeout`, `setInterval`, `Date` и другие временные API.

### Базовое использование

```ts
import { vi, it, expect } from "vitest";

it("shows notification after delay", () => {
  vi.useFakeTimers();

  const callback = vi.fn();
  setTimeout(callback, 5000);

  expect(callback).not.toHaveBeenCalled();

  vi.advanceTimersByTime(5000);

  expect(callback).toHaveBeenCalledTimes(1);

  vi.useRealTimers();
});
```

### advanceTimersByTime

```ts
it("polls every 10 seconds", () => {
  vi.useFakeTimers();

  const poll = vi.fn();
  setInterval(poll, 10000);

  vi.advanceTimersByTime(30000); // Продвигаем на 30 секунд

  expect(poll).toHaveBeenCalledTimes(3);

  vi.useRealTimers();
});
```

### runAllTimers

```ts
it("executes all pending timers", () => {
  vi.useFakeTimers();

  setTimeout(() => console.log("1"), 100);
  setTimeout(() => console.log("2"), 500);
  setTimeout(() => console.log("3"), 1000);

  vi.runAllTimers(); // Выполняет все таймеры

  vi.useRealTimers();
});
```

### runOnlyPendingTimers

```ts
it("executes only pending timers", () => {
  vi.useFakeTimers();

  setTimeout(() => {
    console.log("1");
    setTimeout(() => console.log("2"), 100); // Рекурсивный таймер
  }, 100);

  vi.runOnlyPendingTimers(); // Выполнит только первый, не рекурсивный

  vi.useRealTimers();
});
```

### setSystemTime

```ts
it("formats date correctly", () => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date("2026-01-15T12:00:00Z"));

  expect(formatRelativeDate(new Date())).toBe("today");

  vi.useRealTimers();
});
```

### Использование с async/await

```ts
it("shows loading state", async () => {
  vi.useFakeTimers();

  const { result } = renderHook(() => useDelayedData());

  expect(result.current.isLoading).toBe(true);

  await vi.advanceTimersByTimeAsync(3000);

  expect(result.current.isLoading).toBe(false);
  expect(result.current.data).toEqual([1, 2, 3]);

  vi.useRealTimers();
});
```

---

## Snapshot-тестирование

### Базовые snapshot-тесты

```ts
it("renders button correctly", () => {
  const { container } = render(<Button variant="primary">Click me</Button>);
  expect(container).toMatchSnapshot();
});
```

Vitest создаёт файл `__snapshots__/button.test.ts.snap` с сериализованным DOM:

```
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

```ts
it("renders button with inline snapshot", () => {
  const { container } = render(<Button>Click</Button>);
  expect(container.innerHTML).toMatchInlineSnapshot(`
    "<button class="btn btn--primary">Click</button>"
  `);
});
```

### Когда использовать snapshot

| Сценарий | Snapshot? |
|---|---|
| Иконки, SVG | ✅ Да — стабильные, редко меняются |
| Базовые UI-компоненты (Button, Input) | ✅ Да — если не меняются часто |
| Сложные компоненты с бизнес-логикой | ❌ Нет — snapshot будет ломаться при каждом рефакторе |
| Компоненты с динамическими данными | ❌ Нет — snapshot зависит от данных |

### Обновление snapshot

```bash
vitest --update    # Обновить все snapshots
vitest -u          # Короткая форма
```

> ⚠️ **Внимание:** Обновляйте snapshot только после проверки diff'а. Слепое обновление скрывает нежелательные изменения.

---

## Параметризованные тесты

### each — тесты с разными данными

```ts
it.each([
  [1, 1, 2],
  [2, 3, 5],
  [10, 20, 30],
  [-1, 1, 0],
])("add(%i, %i) returns %i", (a, b, expected) => {
  expect(add(a, b)).toBe(expected);
});
```

### each с объектами

```ts
it.each([
  { input: "hello", expected: "Hello" },
  { input: "WORLD", expected: "World" },
  { input: "fOo", expected: "Foo" },
])("capitalize($input) returns $expected", ({ input, expected }) => {
  expect(capitalize(input)).toBe(expected);
});
```

### each для edge cases

```ts
it.each([
  { input: "", expected: 0, description: "empty string" },
  { input: "a", expected: 1, description: "single char" },
  { input: "hello world", expected: 2, description: "two words" },
  { input: "  spaces  ", expected: 1, description: "with spaces" },
  { input: "a,b,c", expected: 3, description: "comma separated" },
])("countWords: $description", ({ input, expected }) => {
  expect(countWords(input)).toBe(expected);
});
```

### Тестирование типов

```ts
it.each([
  { value: null, expected: false },
  { value: undefined, expected: false },
  { value: "", expected: false },
  { value: 0, expected: false },
  { value: "hello", expected: true },
  { value: 42, expected: true },
  { value: [], expected: true },
  { value: {}, expected: true },
])("isTruthy($value) returns $expected", ({ value, expected }) => {
  expect(isTruthy(value)).toBe(expected);
});
```

---

## Тестирование хуков

### renderHook

```ts
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

### Хуки с провайдерами

```ts
import { renderHook } from "@testing-library/react";
import { ThemeProvider } from "./ThemeProvider";

it("useTheme returns current theme", () => {
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <ThemeProvider initialTheme="dark">{children}</ThemeProvider>
  );

  const { result } = renderHook(() => useTheme(), { wrapper });

  expect(result.current.theme).toBe("dark");
});
```

### Тестирование асинхронных хуков

```ts
it("useFetch loads data", async () => {
  server.use(
    http.get("/api/user/1", () => HttpResponse.json({ id: 1, name: "Alice" }))
  );

  const { result } = renderHook(() => useFetch("/api/user/1"));

  expect(result.current.isLoading).toBe(true);

  await waitFor(() => {
    expect(result.current.isLoading).toBe(false);
  });

  expect(result.current.data).toEqual({ id: 1, name: "Alice" });
});
```

### Тестирование хуков с эффектами

```ts
it("useLocalStorage persists value", () => {
  const { result } = renderHook(() => useLocalStorage("key", "default"));

  expect(result.current[0]).toBe("default");

  act(() => {
    result.current[1]("new value");
  });

  expect(result.current[0]).toBe("new value");
  expect(localStorage.getItem("key")).toBe('"new value"');
});
```

---

## Покрытие кода

### Установка coverage-провайдера

```bash
npm install -D @vitest/coverage-v8
```

### Конфигурация

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html", "lcov"],
      reportsDirectory: "./coverage",

      include: ["src/**/*.{ts,tsx}"],
      exclude: [
        "src/test/**",
        "src/**/*.d.ts",
        "src/**/*.stories.tsx",
        "src/main.tsx",
        "src/vite-env.d.ts",
      ],

      thresholds: {
        "**/*": {
          statements: 80,
          branches: 70,
          functions: 85,
          lines: 80,
        },
        "src/utils/**": {
          statements: 90,
          branches: 80,
          functions: 90,
          lines: 90,
        },
      },
    },
  },
});
```

### Запуск с покрытием

```bash
vitest run --coverage
```

### Анализ отчёта

```
% Coverage report from v8
-----------|---------|----------|---------|---------|
File       | % Stmts | % Branch | % Funcs | % Lines |
-----------|---------|----------|---------|---------|
All files  |   82.60 |    71.42 |   86.36 |   82.60 |
 utils/    |   95.00 |    90.00 |  100.00 |   95.00 |
  format   |  100.00 |   100.00 |  100.00 |  100.00 |
  validate |   90.00 |    80.00 |  100.00 |   90.00 |
 hooks/    |   75.00 |    60.00 |   80.00 |   75.00 |
  useAuth  |   80.00 |    66.66 |   85.71 |   80.00 |
  useCart  |   70.00 |    53.33 |   75.00 |   70.00 |
-----------|---------|----------|---------|---------|
```

---

## Конфигурация для React и Next.js

### React + Vite

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
    css: true,
  },
});
```

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach } from "vitest";

afterEach(() => {
  cleanup();
});
```

### Next.js

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import { resolve } from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    include: ["src/**/*.{test,spec}.{ts,tsx}"],
    alias: {
      "@": resolve(__dirname, "./src"),
    },
  },
});
```

### Мокирование Next.js API

```ts
// src/test/setup.ts
vi.mock("next/navigation", () => ({
  useRouter: vi.fn(() => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn(),
  })),
  usePathname: vi.fn(() => "/"),
  useSearchParams: vi.fn(() => new URLSearchParams()),
}));

vi.mock("next/link", () => ({
  default: ({ children, href }: { children: React.ReactNode; href: string }) =>
    `<a href="${href}">${children}</a>`,
}));
```

---

## Vitest vs Jest

| Характеристика | Vitest | Jest |
|---|---|---|
| **Совместимость API** | Совместим с Jest | Оригинальный API |
| **ESM** | Нативная поддержка | Экспериментальная |
| **TypeScript** | Из коробки | Требует ts-jest |
| **Vite-проекты** | Нативная интеграция | Отдельная настройка |
| **Watch mode** | HMR-based | File watcher |
| **Coverage** | v8, istanbul | v8, babel |
| **Snapshot** | Совместимы | Оригинальные |
| **Workspace** | Встроенный | Нет |
| **Browser mode** | Встроенный (Playwright) | Нет (jest-environment-jsdom) |
| **Миграция с Jest** | Минимальная | — |

### Когда выбрать Jest

- Проект на Webpack (без Vite)
- Legacy-проект с уже настроенным Jest
- Нужна специфичная Jest-экосистема (jest-circus, jest-environment-node)

### Когда выбрать Vitest

- Проект на Vite
- Новый проект
- Нужна нативная ESM-поддержка
- Нужен TypeScript без дополнительной настройки
- Нужен browser mode

---

## Лучшие практики

### 1. Используйте `globals: true` для чистого API

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
  },
});

// Теперь не нужно импортировать describe, it, expect
// describe("...", () => { it("...", () => { expect(...) }) })
```

### 2. Структурируйте тестовые файлы рядом с исходными

```
src/
  utils/
    format.ts
    format.test.ts       ← Тест рядом с кодом
  hooks/
    useAuth.ts
    useAuth.test.ts
  components/
    Button.tsx
    Button.test.tsx
```

### 3. Выносите общие утилиты в setup-файлы

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach, vi } from "vitest";

afterEach(() => {
  cleanup();
  vi.clearAllMocks();
  vi.restoreAllMocks();
});
```

### 4. Используйте MSW для мокирования API

```ts
// src/test/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./mocks/handlers";

export const server = setupServer(...handlers);
```

```ts
// src/test/setup.ts
import { server } from "./server";
import { beforeAll, afterEach, afterAll } from "vitest";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### 5. Фабрики для тестовых данных

```ts
// src/test/factories.ts
export const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  ...overrides,
});

export const createProduct = (overrides?: Partial<Product>): Product => ({
  id: crypto.randomUUID(),
  name: "Test Product",
  price: 29.99,
  category: "electronics",
  ...overrides,
});
```

### 6. Тестируйте edge cases параметризованно

```ts
it.each([
  { input: "", expected: false, description: "empty string" },
  { input: "   ", expected: false, description: "whitespace only" },
  { input: "a@b.c", expected: true, description: "valid email" },
  { input: "invalid", expected: false, description: "no @" },
  { input: "@b.c", expected: false, description: "no local part" },
  { input: "a@", expected: false, description: "no domain" },
])("isValidEmail($input) → $expected ($description)", ({ input, expected }) => {
  expect(isValidEmail(input)).toBe(expected);
});
```
