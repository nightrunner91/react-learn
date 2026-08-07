# Мокирование и изоляция тестов

## Содержание

1. [Зачем мокировать](#зачем-мокировать)
2. [Stub, Spy, Mock — в чём разница](#stub-spy-mock--в-чём-разница)
3. [vi.fn() — создание mock-функций](#vifn--создание-mock-функций)
4. [vi.spyOn() — шпионаж за методами](#vispyon--шпионаж-за-методами)
5. [vi.mock() — мокирование модулей](#vimock--мокирование-модулей)
6. [vi.hoisted() — переменные для vi.mock](#vihoisted--переменные-для-vimock)
7. [MSW — мокирование API на сетевом уровне](#msw--мокирование-api-на-сетевом-уровне)
8. [MSW — настройка и структура](#msw--настройка-и-структура)
9. [Переопределение handlers в тестах](#переопределение-handlers-в-тестах)
10. [Мокирование fetch](#мокирование-fetch)
11. [Мокирование localStorage](#мокирование-localstorage)
12. [Мокирование таймеров](#мокирование-таймеров)
13. [Мокирование модулей браузера](#мокирование-модулей-браузера)
14. [Фабрики тестовых данных](#фабрики-тестовых-данных)
15. [Жизненный цикл моков: clear, reset, restore](#жизненный-цикл-моков-clear-reset-restore)
16. [Лучшие практики](#лучшие-практики)
17. [Антипаттерны](#антипаттерны)

---

## Зачем мокировать

Тесты должны быть **быстрыми**, **стабильными** и **изолированными**. Реальные зависимости нарушают эти принципы:

| Зависимость | Проблема | Решение |
|---|---|---|
| API (fetch, axios) | Медленные, нестабильные, требуют сеть | MSW или vi.mock |
| База данных | Требует реальную БД, медленная | vi.mock репозитория |
| localStorage | Сохраняется между тестами | Очистка или vi.spyOn |
| setTimeout/setInterval | Тесты ждут реальное время | vi.useFakeTimers() |
| Date.now() | Зависит от текущего времени | vi.setSystemTime() |
| console.log | Засоряет вывод тестов | vi.spyOn(console, "log") |
| Сторонние библиотеки | Могут менять поведение | vi.mock модуля |

### Что мокировать, что не мокировать

| Мокировать | Не мокировать |
|---|---|
| Внешние API (fetch, axios) | Утилиты вашего проекта |
| Базу данных | React-хуки (useState, useEffect) |
| Файловую систему | Компоненты (если не нужно) |
| Сетевые запросы | Бизнес-логику |
| Таймеры (setTimeout) | Чистые функции |

> 💡 **Правило:** Мокируйте только **внешние зависимости** — то, что выходит за рамки вашего кода. Не мокируйте внутренние модули, если они не являются внешними зависимостями.

---

## Stub, Spy, Mock — в чём разница

Эти термины часто путают. Вот чёткое различие:

### Stub — заглушка

Заменяет функцию, возвращает заранее определённое значение. Не отслеживает вызовы.

```ts
// Stub — просто возвращает значение
const stub = () => 42;
expect(stub()).toBe(42);
```

### Spy — шпион

Оборачивает реальную функцию, отслеживает вызовы, но сохраняет оригинальное поведение.

```ts
const obj = { greet: (name: string) => `Hello, ${name}` };
const spy = vi.spyOn(obj, "greet");

obj.greet("Alice");

expect(spy).toHaveBeenCalledWith("Alice"); // Отслеживает вызов
expect(obj.greet("Bob")).toBe("Hello, Bob"); // Сохраняет поведение
```

### Mock — полноценная замена

Заменяет функцию, отслеживает вызовы, позволяет управлять поведением.

```ts
const mock = vi.fn();
mock("hello");

expect(mock).toHaveBeenCalledWith("hello"); // Отслеживает вызов
mock.mockReturnValue("world");
expect(mock()).toBe("world"); // Управляемое поведение
```

### Сравнительная таблица

| Характеристика | Stub | Spy | Mock |
|---|---|---|---|
| **Заменяет функцию** | ✅ | ✅ (оборачивает) | ✅ |
| **Отслеживает вызовы** | ❌ | ✅ | ✅ |
| **Сохраняет поведение** | ❌ | ✅ | ❌ (по умолчанию) |
| **Управляемое поведение** | ✅ (фиксированное) | ✅ (через mockReturnValue) | ✅ |
| **Восстановление оригинала** | ❌ | ✅ (mockRestore) | ❌ |

> 💡 В Vitest/Jest `vi.fn()` создаёт **mock**, `vi.spyOn()` создаёт **spy** (который можно превратить в mock через `mockReturnValue`). Термин "stub" используется редко — обычно это просто `vi.fn().mockReturnValue(...)`.

---

## vi.fn() — создание mock-функций

### Базовое использование

```ts
const mockFn = vi.fn();

mockFn("hello");
mockFn("world");

expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith("hello");
expect(mockFn).toHaveBeenLastCalledWith("world");
expect(mockFn).toHaveBeenNthCalledWith(1, "hello");
expect(mockFn).toHaveBeenNthCalledWith(2, "world");
```

### С реализацией

```ts
// Mock с возвращаемым значением
const mockAdd = vi.fn((a: number, b: number) => a + b);
expect(mockAdd(2, 3)).toBe(5);

// Mock с фиксированным возвращаемым значением
const mockFn = vi.fn().mockReturnValue(42);
expect(mockFn()).toBe(42);
expect(mockFn("anything")).toBe(42); // Игнорирует аргументы
```

### Последовательные возвращаемые значения

```ts
const mockFn = vi.fn()
  .mockReturnValueOnce("first")
  .mockReturnValueOnce("second")
  .mockReturnValue("default");

expect(mockFn()).toBe("first");
expect(mockFn()).toBe("second");
expect(mockFn()).toBe("default");
expect(mockFn()).toBe("default"); // Все последующие — "default"
```

### Async mock

```ts
//Resolved value
const mockFetch = vi.fn().mockResolvedValue({ data: [1, 2, 3] });
const result = await mockFetch();
expect(result.data).toEqual([1, 2, 3]);

// Rejected value
const mockFetch = vi.fn().mockRejectedValue(new Error("Network error"));
await expect(mockFetch()).rejects.toThrow("Network error");

// Последовательные async-значения
const mockFn = vi.fn()
  .mockResolvedValueOnce("first")
  .mockResolvedValueOnce("second");
```

### Проверка вызовов

```ts
const mockFn = vi.fn();

mockFn("a", "b");
mockFn("c");

// Количество вызовов
expect(mockFn).toHaveBeenCalledTimes(2);

// Аргументы вызовов
expect(mockFn).toHaveBeenCalledWith("a", "b");
expect(mockFn).toHaveBeenLastCalledWith("c");
expect(mockFn).toHaveBeenNthCalledWith(1, "a", "b");

// Все вызовы
expect(mockFn.mock.calls).toEqual([["a", "b"], ["c"]]);

// Возвращаемые значения
const mockFn2 = vi.fn().mockReturnValue("result");
mockFn2();
expect(mockFn2.mock.results).toEqual([{ type: "return", value: "result" }]);
```

---

## vi.spyOn() — шпионаж за методами

### Базовое использование

```ts
const consoleSpy = vi.spyOn(console, "log");

console.log("hello");

expect(consoleSpy).toHaveBeenCalledWith("hello");

consoleSpy.mockRestore(); // Восстановить оригинальный console.log
```

### Переопределение реализации

```ts
const user = {
  getName: () => "Alice",
};

const spy = vi.spyOn(user, "getName");

// Шпион сохраняет оригинальное поведение
expect(user.getName()).toBe("Alice");
expect(spy).toHaveBeenCalledTimes(1);

// Переопределение
spy.mockReturnValue("Bob");
expect(user.getName()).toBe("Bob");

// Восстановление
spy.mockRestore();
expect(user.getName()).toBe("Alice");
```

### Spy на модуле

```ts
import * as api from "./api";

it("calls api.fetchUsers", async () => {
  const spy = vi.spyOn(api, "fetchUsers").mockResolvedValue([{ id: 1 }]);
  
  const users = await api.fetchUsers();
  
  expect(spy).toHaveBeenCalled();
  expect(users).toEqual([{ id: 1 }]);
  
  spy.mockRestore();
});
```

### Spy на методах DOM

```ts
const scrollSpy = vi.spyOn(HTMLElement.prototype, "scrollTo");

window.scrollTo(0, 100);

expect(scrollSpy).toHaveBeenCalledWith(0, 100);

scrollSpy.mockRestore();
```

---

## vi.mock() — мокирование модулей

### Полное мокирование

```ts
// Полностью заменяет модуль
vi.mock("./api", () => ({
  fetchUsers: vi.fn().mockResolvedValue([{ id: 1, name: "Alice" }]),
  deleteUser: vi.fn().mockResolvedValue(true),
}));

// Использование
import { fetchUsers } from "./api";

it("loads users", async () => {
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

### Частичное мокирование (с сохранением оригинала)

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
// Автоматически мокирует все экспорты
vi.mock("./api");

// Все функции становятся vi.fn() с undefined
import { fetchUsers } from "./api";

it("fetchUsers is mocked", () => {
  expect(vi.mocked(fetchUsers)).toBeDefined();
  expect(fetchUsers()).toBeUndefined(); // По умолчанию возвращает undefined
});
```

### vi.mock() с типами

```ts
vi.mock("./api", () => ({
  fetchUsers: vi.fn<() => Promise<User[]>>().mockResolvedValue([
    { id: 1, name: "Alice" },
  ]),
}));
```

### vi.mocked() — типизация моков

```ts
import { fetchUsers } from "./api";

vi.mock("./api");

it("uses typed mock", async () => {
  const mockFetch = vi.mocked(fetchUsers);
  mockFetch.mockResolvedValue([{ id: 1, name: "Alice" }]);
  
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

---

## vi.hoisted() — переменные для vi.mock

### Проблема

`vi.mock()` вызывается **до** импортов (hoisting), поэтому обычные переменные недоступны внутри factory-функции:

```ts
// ❌ ОШИБКА — mockFn не доступен в vi.mock
const mockFn = vi.fn();

vi.mock("./api", () => ({
  fetchUsers: mockFn, // ❌ mockFn is not defined
}));
```

### Решение: vi.hoisted()

```ts
// ✅ vi.hoisted() поднимает переменные вместе с vi.mock
const { mockFetchUsers, mockDeleteUser } = vi.hoisted(() => ({
  mockFetchUsers: vi.fn(),
  mockDeleteUser: vi.fn(),
}));

vi.mock("./api", () => ({
  fetchUsers: mockFetchUsers,
  deleteUser: mockDeleteUser,
}));

// Использование в тестах
import { fetchUsers } from "./api";

it("fetches users", async () => {
  mockFetchUsers.mockResolvedValue([{ id: 1 }]);
  
  const users = await fetchUsers();
  
  expect(users).toEqual([{ id: 1 }]);
  expect(mockFetchUsers).toHaveBeenCalled();
});
```

### vi.hoisted() с классами

```ts
const { MockIntersectionObserver } = vi.hoisted(() => {
  class MockIntersectionObserver {
    observe = vi.fn();
    unobserve = vi.fn();
    disconnect = vi.fn();
  }
  return { MockIntersectionObserver };
});

vi.mock("./IntersectionObserver", () => ({
  IntersectionObserver: MockIntersectionObserver,
}));
```

---

## MSW — мокирование API на сетевом уровне

### Что такое MSW

**MSW (Mock Service Worker)** — библиотека для мокирования HTTP-запросов на уровне Service Worker. Код компонента не знает, что работает с моком — он думает, что общается с реальным API.

### Преимущества MSW

| Преимущество | Описание |
|---|---|
| **Работает с любой библиотекой** | fetch, axios, react-query, swr — неважно |
| **Один handler для тестов и разработки** | Можно использовать в Storybook, моках для фронтенда |
| **Точечное переопределение** | Можно переопределить handler для конкретного теста |
| **Реалистичное мокирование** | Перехватывает запросы на сетевом уровне, не в коде |
| **Поддержка REST и GraphQL** | Из коробки |

### Установка

```bash
npm install -D msw
```

### Инициализация

```bash
npx msw init public/ --save
```

Это создаст `public/mockServiceWorker.js` — Service Worker, который перехватывает запросы.

---

## MSW — настройка и структура

### Базовая структура

```
src/
  test/
    mocks/
      handlers.ts    — базовые handlers
      server.ts      — setupServer для Node.js
      browser.ts     — setupWorker для браузера (опционально)
    setup.ts         — глобальная настройка тестов
```

### Handlers

```ts
// src/test/mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  // GET /api/users
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "Alice" },
      { id: 2, name: "Bob" },
    ]);
  }),

  // GET /api/users/:id
  http.get("/api/users/:id", ({ params }) => {
    const { id } = params;
    return HttpResponse.json({ id, name: "Alice" });
  }),

  // POST /api/users
  http.post("/api/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(
      { id: 3, ...body },
      { status: 201 }
    );
  }),

  // DELETE /api/users/:id
  http.delete("/api/users/:id", ({ params }) => {
    return new HttpResponse(null, { status: 204 });
  }),
];
```

### Server (для Node.js / тестов)

```ts
// src/test/mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

### Setup-файл

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach, beforeAll, afterAll } from "vitest";
import { server } from "./mocks/server";

// Запуск сервера перед всеми тестами
beforeAll(() => server.listen({ onUnhandledRequest: "error" }));

// Сброс handlers после каждого теста
afterEach(() => {
  server.resetHandlers();
  cleanup();
});

// Остановка сервера после всех тестов
afterAll(() => server.close());
```

### onUnhandledRequest

```ts
server.listen({
  onUnhandledRequest: "error", // Ошибка на необработанные запросы
  // onUnhandledRequest: "warn", // Предупреждение
  // onUnhandledRequest: "bypass", // Игнорировать
});
```

> 💡 **Рекомендация:** Используйте `"error"` в тестах — это гарантирует, что все запросы покрыты handlers. Если тест делает запрос без handler — вы сразу узнаете.

---

## Переопределение handlers в тестах

### server.use() — точечное переопределение

```ts
import { http, HttpResponse } from "msw";
import { server } from "../test/mocks/server";

it("handles server error", async () => {
  // Переопределяет handler только для этого теста
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json(
        { error: "Server error" },
        { status: 500 }
      );
    })
  );

  render(<UserList />);

  expect(await screen.findByText(/server error/i)).toBeInTheDocument();
});

it("handles empty list", async () => {
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json([]);
    })
  );

  render(<UserList />);

  expect(await screen.findByText(/no users found/i)).toBeInTheDocument();
});
```

### Как работает server.use()

```
1. beforeAll: server.listen() — запускает сервер с базовыми handlers
2. Тест 1: server.use() — добавляет переопределение
3. afterEach: server.resetHandlers() — убирает переопределения
4. Тест 2: базовые handlers снова активны
```

### Переопределение с задержкой

```ts
it("shows loading state", async () => {
  server.use(
    http.get("/api/users", async () => {
      await new Promise((resolve) => setTimeout(resolve, 1000));
      return HttpResponse.json([{ id: 1, name: "Alice" }]);
    })
  );

  render(<UserList />);

  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

### Переопределение для конкретного URL

```ts
server.use(
  http.get("/api/users/:id", ({ params }) => {
    if (params.id === "999") {
      return HttpResponse.json({ error: "Not found" }, { status: 404 });
    }
    return HttpResponse.json({ id: params.id, name: "Alice" });
  })
);
```

---

## Мокирование fetch

### Без MSW — через vi.mock

```ts
// Глобальный mock fetch
global.fetch = vi.fn();

beforeEach(() => {
  vi.mocked(global.fetch).mockReset();
});

it("fetches data", async () => {
  vi.mocked(global.fetch).mockResolvedValueOnce({
    ok: true,
    json: async () => [{ id: 1, name: "Alice" }],
  });

  const users = await fetchUsers();

  expect(global.fetch).toHaveBeenCalledWith("/api/users");
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

### С обработкой ошибок

```ts
it("handles network error", async () => {
  vi.mocked(global.fetch).mockRejectedValueOnce(new Error("Network error"));

  await expect(fetchUsers()).rejects.toThrow("Network error");
});

it("handles API error", async () => {
  vi.mocked(global.fetch).mockResolvedValueOnce({
    ok: false,
    status: 500,
    json: async () => ({ error: "Server error" }),
  });

  await expect(fetchUsers()).rejects.toThrow("Server error");
});
```

---

## Мокирование localStorage

### Очистка между тестами

```ts
beforeEach(() => {
  localStorage.clear();
});

it("persists data to localStorage", () => {
  render(<SettingsPage />);
  
  fireEvent.click(screen.getByLabelText(/dark mode/i));
  
  expect(localStorage.getItem("theme")).toBe("dark");
});
```

### Mock localStorage

```ts
const localStorageMock = (() => {
  let store: Record<string, string> = {};
  return {
    getItem: vi.fn((key: string) => store[key] ?? null),
    setItem: vi.fn((key: string, value: string) => {
      store[key] = value;
    }),
    removeItem: vi.fn((key: string) => {
      delete store[key];
    }),
    clear: vi.fn(() => {
      store = {};
    }),
  };
})();

Object.defineProperty(window, "localStorage", {
  value: localStorageMock,
});
```

---

## Мокирование таймеров

### vi.useFakeTimers()

```ts
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

### vi.setSystemTime()

```ts
it("formats relative date", () => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date("2026-01-15T12:00:00Z"));

  expect(formatRelativeDate(new Date("2026-01-15T10:00:00Z"))).toBe("2 hours ago");

  vi.useRealTimers();
});
```

### advanceTimersByTimeAsync — для async/await

```ts
it("loads data after delay", async () => {
  vi.useFakeTimers();

  const promise = loadData();

  await vi.advanceTimersByTimeAsync(3000);

  const data = await promise;
  expect(data).toEqual([1, 2, 3]);

  vi.useRealTimers();
});
```

---

## Мокирование модулей браузера

### IntersectionObserver

```ts
const mockIntersectionObserver = vi.fn();
mockIntersectionObserver.mockReturnValue({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
});

window.IntersectionObserver = mockIntersectionObserver;
```

### matchMedia

```ts
Object.defineProperty(window, "matchMedia", {
  writable: true,
  value: vi.fn().mockImplementation((query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});
```

### window.scrollTo

```ts
window.scrollTo = vi.fn();

it("scrolls to top", () => {
  scrollToTop();
  expect(window.scrollTo).toHaveBeenCalledWith(0, 0);
});
```

---

## Фабрики тестовых данных

### Зачем нужны фабрики

```ts
// ❌ Дублирование данных в тестах
it("creates user", () => {
  const user = { id: 1, name: "Alice", email: "alice@example.com", role: "user" };
  // ...
});

it("displays user", () => {
  const user = { id: 2, name: "Bob", email: "bob@example.com", role: "admin" };
  // ...
});

// ✅ Фабрика — переиспользование и гибкость
const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  ...overrides,
});

it("creates user", () => {
  const user = createUser({ name: "Alice" });
  // ...
});
```

### Базовая фабрика

```ts
// src/test/factories.ts
export const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  createdAt: new Date("2026-01-01"),
  ...overrides,
});

export const createProduct = (overrides?: Partial<Product>): Product => ({
  id: crypto.randomUUID(),
  name: "Test Product",
  price: 29.99,
  category: "electronics",
  inStock: true,
  ...overrides,
});
```

### Использование фабрик

```ts
import { createUser } from "../test/factories";

it("displays user name", () => {
  const user = createUser({ name: "Alice" });
  render(<UserProfile user={user} />);
  expect(screen.getByText("Alice")).toBeInTheDocument();
});

it("shows admin badge for admin users", () => {
  const admin = createUser({ role: "admin" });
  render(<UserProfile user={admin} />);
  expect(screen.getByText(/admin/i)).toBeInTheDocument();
});

it("hides admin badge for regular users", () => {
  const user = createUser({ role: "user" });
  render(<UserProfile user={user} />);
  expect(screen.queryByText(/admin/i)).not.toBeInTheDocument();
});
```

### Фабрика с последовательностью

```ts
let userIdCounter = 0;

export const createUser = (overrides?: Partial<User>): User => {
  userIdCounter++;
  return {
    id: userIdCounter,
    name: `User ${userIdCounter}`,
    email: `user${userIdCounter}@example.com`,
    role: "user",
    ...overrides,
  };
};

// Использование
const user1 = createUser(); // { id: 1, name: "User 1", ... }
const user2 = createUser(); // { id: 2, name: "User 2", ... }
const admin = createUser({ role: "admin" }); // { id: 3, name: "User 3", role: "admin" }
```

### Фабрика для массивов

```ts
export const createUsers = (count: number, overrides?: Partial<User>): User[] => {
  return Array.from({ length: count }, () => createUser(overrides));
};

// Использование
const users = createUsers(5); // 5 пользователей
const admins = createUsers(3, { role: "admin" }); // 3 админа
```

---

## Жизненный цикл моков: clear, reset, restore

### vi.clearAllMocks()

Очищает **историю вызовов** всех моков, но сохраняет реализацию.

```ts
const mockFn = vi.fn().mockReturnValue(42);
mockFn("hello");

expect(mockFn).toHaveBeenCalledTimes(1);

vi.clearAllMocks();

expect(mockFn).toHaveBeenCalledTimes(0); // История очищена
expect(mockFn()).toBe(42); // Реализация сохранена
```

### vi.resetAllMocks()

Очищает историю **и убирает реализацию** (моки возвращают `undefined`).

```ts
const mockFn = vi.fn().mockReturnValue(42);
mockFn("hello");

vi.resetAllMocks();

expect(mockFn).toHaveBeenCalledTimes(0); // История очищена
expect(mockFn()).toBeUndefined(); // Реализация убрана
```

### vi.restoreAllMocks()

Очищает историю, убирает реализацию **и восстанавливает оригиналы** (для `vi.spyOn`).

```ts
const obj = { greet: () => "hello" };
const spy = vi.spyOn(obj, "greet").mockReturnValue("mocked");

expect(obj.greet()).toBe("mocked");

vi.restoreAllMocks();

expect(obj.greet()).toBe("hello"); // Оригинальная реализация восстановлена
```

### Сравнительная таблица

| Метод | История вызовов | Реализация | Оригинал (spyOn) |
|---|---|---|---|
| `mockClear()` / `clearAllMocks()` | Очищает | Сохраняет | Сохраняет |
| `mockReset()` / `resetAllMocks()` | Очищает | Убирает (→ undefined) | Сохраняет |
| `mockRestore()` / `restoreAllMocks()` | Очищает | Убирает | Восстанавливает |

### Когда использовать

```ts
afterEach(() => {
  // Обычно достаточно clearAllMocks
  vi.clearAllMocks();
});

// restoreAllMocks — если используете spyOn
afterEach(() => {
  vi.restoreAllMocks();
});
```

---

## Лучшие практики

### 1. Используйте MSW для API

```ts
// ✅ MSW — реалистичное мокирование
server.use(
  http.get("/api/users", () => HttpResponse.json([{ id: 1 }]))
);

// ❌ Ручной mock fetch — хрупкий
global.fetch = vi.fn().mockResolvedValue({
  ok: true,
  json: async () => [{ id: 1 }],
});
```

### 2. Переопределяйте handlers точечно

```ts
// ✅ Точечное переопределение
it("handles error", async () => {
  server.use(
    http.get("/api/users", () => HttpResponse.json({ error: "Error" }, { status: 500 }))
  );
  // ...
});

// ❌ Глобальное изменение handlers
beforeEach(() => {
  handlers[0] = http.get("/api/users", () => ...); // Хрупко
});
```

### 3. Используйте фабрики для данных

```ts
// ✅ Фабрика — гибкость и переиспользование
const user = createUser({ name: "Alice" });

// ❌ Хардкод в каждом тесте
const user = { id: 1, name: "Alice", email: "alice@example.com", role: "user" };
```

### 4. Очищайте моки между тестами

```ts
afterEach(() => {
  vi.clearAllMocks();
  server.resetHandlers();
});
```

### 5. Мокируйте только внешние зависимости

```ts
// ✅ Мокируем API (внешняя зависимость)
server.use(http.get("/api/users", () => ...));

// ❌ Мокируем утилиты проекта (внутренняя зависимость)
vi.mock("./utils/format", () => ({ format: vi.fn() }));
```

### 6. Используйте vi.hoisted() для переменных в vi.mock

```ts
// ✅ vi.hoisted() — переменные доступны в vi.mock
const { mockFn } = vi.hoisted(() => ({ mockFn: vi.fn() }));
vi.mock("./api", () => ({ fetchUsers: mockFn }));

// ❌ Ошибка — переменная не доступна
const mockFn = vi.fn();
vi.mock("./api", () => ({ fetchUsers: mockFn })); // ❌
```

### 7. Типизируйте моки

```ts
// ✅ vi.mocked() — типизация
import { fetchUsers } from "./api";
vi.mock("./api");

const mockFetch = vi.mocked(fetchUsers);
mockFetch.mockResolvedValue([{ id: 1 }]);

// ❌ Без типизации — нет autocomplete
(fetchUsers as any).mockResolvedValue([{ id: 1 }]);
```

---

## Антипаттерны

### 1. Избыточное мокирование

```ts
// ❌ Мокирует всё — тест проверяет моки, а не код
vi.mock("react", () => ({ useState: vi.fn() }));
vi.mock("./utils", () => ({ format: vi.fn() }));
vi.mock("./hooks", () => ({ useAuth: vi.fn() }));

// ✅ Мокирует только внешние зависимости
server.use(http.get("/api/data", () => HttpResponse.json({ data: [] })));
```

### 2. Мокирование внутренних модулей без необходимости

```ts
// ❌ Мокирует утилиту проекта
vi.mock("./utils/format", () => ({ formatPrice: vi.fn().mockReturnValue("$10") }));

// ✅ Использует реальную утилиту
import { formatPrice } from "./utils/format";
expect(formatPrice(10)).toBe("$10.00");
```

### 3. Забытый mockRestore

```ts
// ❌ Не восстанавливает оригинал
const spy = vi.spyOn(console, "log");
// ... тест
// console.log остаётся замоканым для следующих тестов

// ✅ Восстанавливает оригинал
const spy = vi.spyOn(console, "log");
// ... тест
spy.mockRestore();

// ✅ Или в afterEach
afterEach(() => {
  vi.restoreAllMocks();
});
```

### 4. Зависимость от порядка тестов

```ts
// ❌ Второй тест зависит от первого
it("creates user", () => {
  mockCreateUser({ id: 1 });
  expect(getUsers()).toHaveLength(1);
});

it("finds user", () => {
  // ❌ Пользователь создан в предыдущем тесте
  expect(findUser(1)).not.toBeNull();
});

// ✅ Каждый тест независим
it("finds user", () => {
  mockCreateUser({ id: 1 });
  expect(findUser(1)).not.toBeNull();
});
```

### 5. Хардкод данных вместо фабрик

```ts
// ❌ Дублирование данных
it("test 1", () => {
  const user = { id: 1, name: "Alice", email: "alice@example.com" };
});
it("test 2", () => {
  const user = { id: 2, name: "Bob", email: "bob@example.com" };
});

// ✅ Фабрика
it("test 1", () => {
  const user = createUser({ name: "Alice" });
});
it("test 2", () => {
  const user = createUser({ name: "Bob" });
});
```

### 6. Игнорирование onUnhandledRequest

```ts
// ❌ Необработанные запросы игнорируются — сложно отловить ошибки
server.listen();

// ✅ Ошибка на необработанные запросы
server.listen({ onUnhandledRequest: "error" });
```

### 7. Мокирование без очистки

```ts
// ❌ Моки накапливаются между тестами
beforeAll(() => {
  vi.mock("./api");
});

// ✅ Очистка в afterEach
afterEach(() => {
  vi.clearAllMocks();
  server.resetHandlers();
});
```
