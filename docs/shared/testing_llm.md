# Тестирование React / Next.js и использование LLM

## Содержание

1. [Современный стек тестирования](#современный-стек-тестирования)
2. [Модульное тестирование (Unit)](#модульное-тестирование-unit)
3. [Интеграционное тестирование](#интеграционное-тестирование)
4. [Компонентное тестирование](#компонентное-тестирование)
5. [E2E тестирование](#e2e-тестирование)
6. [Тестирование хуков](#тестирование-хуков)
7. [Мокирование API](#мокирование-api)
8. [Тестирование Server Components](#тестирование-server-components)
9. [Тестирование Server Actions](#тестирование-server-actions)
10. [Визуальное тестирование](#визуальное-тестирование)
11. [LLM для тестирования](#llm-для-тестирования)
12. [Генерация тестов через LLM](#генерация-тестов-через-llm)
13. [Тестирование с LLM-ассистентом](#тестирование-с-llm-ассистентом)
14. [Лучшие практики](#лучшие-практики)
15. [Антипаттерны](#антипаттерны)

---

## Современный стек тестирования

В 2026 году рекомендуемый стек:

| Уровень | Инструмент | Назначение |
|---|---|---|
| Раннер | **Vitest** | Модульные/интеграционные тесты (нативный для Vite) |
| Компоненты | **Testing Library** | Тестирование с точки зрения пользователя |
| E2E | **Playwright** | Сквозное тестирование в браузере |
| Моки | **MSW** | Мокирование API на сетевом уровне |
| Визуальные | **Chromatic / Percy** | Snapshot-тестирование UI |
| Покрытие | **c8 / istanbul** | Code coverage |

### Установка

```bash
# Vitest + Testing Library
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom

# Playwright
npm install -D @playwright/test
npx playwright install

# MSW
npm install -D msw
```

### Конфигурация Vitest

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
    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html"],
    },
  },
});
```

```ts
// src/test/setup.ts
import "@testing-library/jest-dom";
```

---

## Модульное тестирование (Unit)

### Тестирование утилит

```ts
// utils/format.ts
export function formatCurrency(amount: number, currency = "USD"): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency,
  }).format(amount);
}

// utils/format.test.ts
import { describe, it, expect } from "vitest";
import { formatCurrency } from "./format";

describe("formatCurrency", () => {
  it("formats USD by default", () => {
    expect(formatCurrency(1234.56)).toBe("$1,234.56");
  });

  it("formats EUR when specified", () => {
    expect(formatCurrency(1234.56, "EUR")).toBe("€1,234.56");
  });

  it("handles zero", () => {
    expect(formatCurrency(0)).toBe("$0.00");
  });

  it("handles negative values", () => {
    expect(formatCurrency(-50)).toBe("-$50.00");
  });
});
```

### Тестирование бизнес-логики

```ts
// features/cart/cartLogic.ts
export type CartItem = { id: string; name: string; price: number; quantity: number };

export function addToCart(items: CartItem[], newItem: Omit<CartItem, "quantity">): CartItem[] {
  const existing = items.find((item) => item.id === newItem.id);

  if (existing) {
    return items.map((item) =>
      item.id === newItem.id ? { ...item, quantity: item.quantity + 1 } : item
    );
  }

  return [...items, { ...newItem, quantity: 1 }];
}

export function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// features/cart/cartLogic.test.ts
import { describe, it, expect } from "vitest";
import { addToCart, calculateTotal } from "./cartLogic";

describe("cartLogic", () => {
  describe("addToCart", () => {
    it("adds new item to empty cart", () => {
      const result = addToCart([], { id: "1", name: "Widget", price: 10 });
      expect(result).toEqual([{ id: "1", name: "Widget", price: 10, quantity: 1 }]);
    });

    it("increments quantity for existing item", () => {
      const items = [{ id: "1", name: "Widget", price: 10, quantity: 2 }];
      const result = addToCart(items, { id: "1", name: "Widget", price: 10 });
      expect(result[0].quantity).toBe(3);
    });

    it("does not mutate original array", () => {
      const items = [{ id: "1", name: "Widget", price: 10, quantity: 1 }];
      const result = addToCart(items, { id: "2", name: "Gadget", price: 20 });
      expect(items).toHaveLength(1);
      expect(result).toHaveLength(2);
    });
  });

  describe("calculateTotal", () => {
    it("returns 0 for empty cart", () => {
      expect(calculateTotal([])).toBe(0);
    });

    it("calculates total correctly", () => {
      const items = [
        { id: "1", name: "Widget", price: 10, quantity: 2 },
        { id: "2", name: "Gadget", price: 25, quantity: 1 },
      ];
      expect(calculateTotal(items)).toBe(45);
    });
  });
});
```

---

## Интеграционное тестирование

### Тестирование хуков с API

```tsx
// hooks/useUsers.test.tsx
import { renderHook, waitFor } from "@testing-library/react";
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";
import { afterAll, afterEach, beforeAll, describe, it, expect } from "vitest";
import { useUsers } from "./useUsers";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const server = setupServer(
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: "1", name: "Alice" },
      { id: "2", name: "Bob" },
    ]);
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}

describe("useUsers", () => {
  it("fetches and returns users", async () => {
    const { result } = renderHook(() => useUsers(), { wrapper: createWrapper() });

    expect(result.current.isLoading).toBe(true);

    await waitFor(() => expect(result.current.isSuccess).toBe(true));

    expect(result.current.data).toEqual([
      { id: "1", name: "Alice" },
      { id: "2", name: "Bob" },
    ]);
  });

  it("handles API error", async () => {
    server.use(http.get("/api/users", () => HttpResponse.json({ error: "Server error" }, { status: 500 })));

    const { result } = renderHook(() => useUsers(), { wrapper: createWrapper() });

    await waitFor(() => expect(result.current.isError).toBe(true));
    expect(result.current.error).toBeDefined();
  });
});
```

---

## Компонентное тестирование

### Базовое тестирование компонента

```tsx
// components/Counter.tsx
import { useState } from "react";

export function Counter({ initialCount = 0 }) {
  const [count, setCount] = useState(initialCount);

  return (
    <div>
      <p data-testid="count">Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={() => setCount(c => c - 1)}>Decrement</button>
    </div>
  );
}

// components/Counter.test.tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { describe, it, expect } from "vitest";
import { Counter } from "./Counter";

describe("Counter", () => {
  it("renders with initial count", () => {
    render(<Counter initialCount={5} />);
    expect(screen.getByTestId("count")).toHaveTextContent("Count: 5");
  });

  it("increments count on button click", () => {
    render(<Counter />);
    fireEvent.click(screen.getByText("Increment"));
    expect(screen.getByTestId("count")).toHaveTextContent("Count: 1");
  });

  it("decrements count on button click", () => {
    render(<Counter initialCount={3} />);
    fireEvent.click(screen.getByText("Decrement"));
    expect(screen.getByTestId("count")).toHaveTextContent("Count: 2");
  });

  it("calls onChange callback", () => {
    const handleChange = vi.fn();
    render(<Counter onChange={handleChange} />);
    fireEvent.click(screen.getByText("Increment"));
    expect(handleChange).toHaveBeenCalledWith(1);
  });
});
```

### Тестирование форм

```tsx
// components/LoginForm.tsx
export function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  const handleSubmit = (e) => {
    e.preventDefault();
    onSubmit({ email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />

      <label htmlFor="password">Password</label>
      <input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />

      <button type="submit">Login</button>
    </form>
  );
}

// components/LoginForm.test.tsx
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { describe, it, expect, vi } from "vitest";
import { LoginForm } from "./LoginForm";

describe("LoginForm", () => {
  it("submits form with email and password", async () => {
    const handleSubmit = vi.fn();
    render(<LoginForm onSubmit={handleSubmit} />);

    fireEvent.change(screen.getByLabelText("Email"), { target: { value: "test@example.com" } });
    fireEvent.change(screen.getByLabelText("Password"), { target: { value: "password123" } });
    fireEvent.click(screen.getByText("Login"));

    await waitFor(() => {
      expect(handleSubmit).toHaveBeenCalledWith({
        email: "test@example.com",
        password: "password123",
      });
    });
  });

  it("shows validation error for empty email", () => {
    render(<LoginForm onSubmit={vi.fn()} />);
    fireEvent.click(screen.getByText("Login"));
    expect(screen.getByText("Email is required")).toBeInTheDocument();
  });
});
```

### Тестирование асинхронных компонентов

```tsx
import { render, screen, waitFor } from "@testing-library/react";
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";
import { UserProfile } from "./UserProfile";

const server = setupServer();

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe("UserProfile", () => {
  it("displays user data after loading", async () => {
    server.use(
      http.get("/api/user/1", () => {
        return HttpResponse.json({ name: "Alice", email: "alice@example.com" });
      })
    );

    render(<UserProfile userId="1" />);

    expect(screen.getByText("Loading...")).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.getByText("Alice")).toBeInTheDocument();
      expect(screen.getByText("alice@example.com")).toBeInTheDocument();
    });
  });

  it("displays error on API failure", async () => {
    server.use(
      http.get("/api/user/1", () => {
        return HttpResponse.json({ error: "Not found" }, { status: 404 });
      })
    );

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText("Failed to load user")).toBeInTheDocument();
    });
  });
});
```

---

## E2E тестирование

### Playwright

```ts
// e2e/auth.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Authentication", () => {
  test("user can login with valid credentials", async ({ page }) => {
    await page.goto("/login");

    await page.fill('input[name="email"]', "test@example.com");
    await page.fill('input[name="password"]', "password123");
    await page.click('button[type="submit"]');

    await expect(page).toHaveURL("/dashboard");
    await expect(page.locator("h1")).toHaveText("Welcome, Test User");
  });

  test("user sees error with invalid credentials", async ({ page }) => {
    await page.goto("/login");

    await page.fill('input[name="email"]', "wrong@example.com");
    await page.fill('input[name="password"]', "wrong");
    await page.click('button[type="submit"]');

    await expect(page.locator('[data-testid="error"]')).toHaveText("Invalid credentials");
  });

  test("user can logout", async ({ page }) => {
    await page.goto("/dashboard");
    await page.click('button:has-text("Logout")');
    await expect(page).toHaveURL("/login");
  });
});
```

### Тестирование форм в Playwright

```ts
test("checkout flow", async ({ page }) => {
  await page.goto("/cart");

  await page.click('button:has-text("Checkout")');

  await page.fill('input[name="name"]', "John Doe");
  await page.fill('input[name="address"]', "123 Main St");
  await page.fill('input[name="card"]', "4242424242424242");

  await page.click('button:has-text("Place Order")');

  await expect(page.locator('[data-testid="success"]')).toBeVisible();
  await expect(page).toHaveURL(/\/order\/.+/);
});
```

---

## Тестирование хуков

### renderHook

```tsx
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

describe("useCounter", () => {
  it("increments count", () => {
    const { result } = renderHook(() => useCounter());

    expect(result.current.count).toBe(0);

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it("accepts initial value", () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it("resets count", () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
      result.current.increment();
    });

    expect(result.current.count).toBe(2);

    act(() => {
      result.current.reset();
    });

    expect(result.current.count).toBe(0);
  });
});
```

---

## Мокирование API

### MSW (Mock Service Worker)

```ts
// src/test/mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: "1", name: "Alice" },
      { id: "2", name: "Bob" },
    ]);
  }),

  http.post("/api/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: "3", ...body }, { status: 201 });
  }),

  http.delete("/api/users/:id", ({ params }) => {
    return HttpResponse.json({ success: true });
  }),
];

// src/test/mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);

// src/test/setup.ts
import { server } from "./mocks/server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Переопределение handlers

```ts
import { http, HttpResponse } from "msw";
import { server } from "./mocks/server";

it("handles server error", async () => {
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json({ error: "Server error" }, { status: 500 });
    })
  );

  render(<UserList />);

  await waitFor(() => {
    expect(screen.getByText("Failed to load users")).toBeInTheDocument();
  });
});
```

---

## Тестирование Server Components

```tsx
// app/users/page.tsx
import { db } from "@/lib/db";
import { UserList } from "./UserList";

export default async function UsersPage() {
  const users = await db.users.findMany();
  return <UserList users={users} />;
}

// app/users/page.test.tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import UsersPage from "./page";

vi.mock("@/lib/db", () => ({
  db: {
    users: {
      findMany: vi.fn(),
    },
  },
}));

describe("UsersPage", () => {
  it("renders users from database", async () => {
    const { db } = await import("@/lib/db");
    vi.mocked(db.users.findMany).mockResolvedValue([
      { id: "1", name: "Alice" },
      { id: "2", name: "Bob" },
    ]);

    const page = await UsersPage();
    render(page);

    expect(screen.getByText("Alice")).toBeInTheDocument();
    expect(screen.getByText("Bob")).toBeInTheDocument();
  });
});
```

---

## Тестирование Server Actions

```tsx
// app/actions/createPost.ts
"use server";

import { db } from "@/lib/db";
import { revalidatePath } from "next/cache";

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string;

  if (!title || title.length < 3) {
    return { error: "Title must be at least 3 characters" };
  }

  await db.posts.create({ title });
  revalidatePath("/posts");
  return { success: true };
}

// app/actions/createPost.test.ts
import { describe, it, expect, vi } from "vitest";
import { createPost } from "./createPost";

vi.mock("@/lib/db", () => ({
  db: {
    posts: {
      create: vi.fn(),
    },
  },
}));

vi.mock("next/cache", () => ({
  revalidatePath: vi.fn(),
}));

describe("createPost", () => {
  it("creates post with valid title", async () => {
    const formData = new FormData();
    formData.append("title", "My Post");

    const result = await createPost(formData);

    expect(result).toEqual({ success: true });
    expect(vi.mocked(db.posts.create)).toHaveBeenCalledWith({ title: "My Post" });
  });

  it("returns error for short title", async () => {
    const formData = new FormData();
    formData.append("title", "ab");

    const result = await createPost(formData);

    expect(result).toEqual({ error: "Title must be at least 3 characters" });
    expect(db.posts.create).not.toHaveBeenCalled();
  });
});
```

---

## Визуальное тестирование

### Chromatic (Storybook)

```bash
npm install -D chromatic @storybook/react
```

```tsx
// components/Button.stories.tsx
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

```bash
npx chromatic --project-token=xxx
```

### Percy

```bash
npm install -D @percy/cli @percy/playwright
```

```ts
// e2e/visual.spec.ts
import { test, expect } from "@playwright/test";
import percySnapshot from "@percy/playwright";

test("homepage visual test", async ({ page }) => {
  await page.goto("/");
  await percySnapshot(page, "Homepage");
});

test("dashboard visual test", async ({ page }) => {
  await page.goto("/dashboard");
  await percySnapshot(page, "Dashboard");
});
```

---

## LLM для тестирования

### Генерация тестов через ChatGPT / Claude

LLM могут генерировать тесты по описанию компонента или функции:

**Промпт:**
```
Напиши тесты для функции addToCart на Vitest.
Функция принимает массив CartItem и новый элемент.
Если элемент уже есть — увеличивает quantity.
Если нет — добавляет с quantity: 1.
Не мутирует исходный массив.

type CartItem = { id: string; name: string; price: number; quantity: number };

export function addToCart(items: CartItem[], newItem: Omit<CartItem, "quantity">): CartItem[] {
  // реализация
}
```

**Результат (генерируется LLM):**
```ts
import { describe, it, expect } from "vitest";
import { addToCart } from "./cart";

describe("addToCart", () => {
  it("adds new item with quantity 1", () => {
    const result = addToCart([], { id: "1", name: "Widget", price: 10 });
    expect(result).toEqual([{ id: "1", name: "Widget", price: 10, quantity: 1 }]);
  });

  it("increments quantity for existing item", () => {
    const items = [{ id: "1", name: "Widget", price: 10, quantity: 2 }];
    const result = addToCart(items, { id: "1", name: "Widget", price: 10 });
    expect(result[0].quantity).toBe(3);
  });

  it("does not mutate original array", () => {
    const items = [{ id: "1", name: "Widget", price: 10, quantity: 1 }];
    addToCart(items, { id: "2", name: "Gadget", price: 20 });
    expect(items).toHaveLength(1);
  });

  it("handles multiple different items", () => {
    const items = [{ id: "1", name: "Widget", price: 10, quantity: 1 }];
    const result = addToCart(items, { id: "2", name: "Gadget", price: 20 });
    expect(result).toHaveLength(2);
  });
});
```

### Генерация edge cases

**Промпт:**
```
Какие edge cases нужно протестировать для функции calculateTotal(items: CartItem[]): number?
```

**LLM предложит:**
- Пустой массив → 0
- Один элемент
- Несколько элементов с разными ценами
- Количество = 0
- Отрицательные цены
- Очень большие числа (переполнение)
- NaN / undefined в price

---

## Тестирование с LLM-ассистентом

### Copilot / Cursor для написания тестов

В IDE с AI-ассистентом:

```ts
// Наведите курсор на функцию и нажмите Cmd+I (Cursor) или Ctrl+I
// Промпт: "Generate unit tests for this function"

export function debounce<T extends (...args: any[]) => void>(fn: T, delay: number): T {
  let timeoutId: ReturnType<typeof setTimeout>;
  return ((...args: any[]) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  }) as T;
}
```

AI сгенерирует:

```ts
import { describe, it, expect, vi } from "vitest";
import { debounce } from "./debounce";

describe("debounce", () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it("delays function execution", () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 100);

    debounced();
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(100);
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it("resets timer on subsequent calls", () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 100);

    debounced();
    vi.advanceTimersByTime(50);
    debounced();
    vi.advanceTimersByTime(50);

    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(50);
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it("passes arguments to original function", () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 100);

    debounced("arg1", "arg2");
    vi.advanceTimersByTime(100);

    expect(fn).toHaveBeenCalledWith("arg1", "arg2");
  });
});
```

### Ревью тестов через LLM

**Промпт:**
```
Проанализируй эти тесты и предложи улучшения:

[вставить код тестов]

Критерии:
- Покрытие edge cases
- Читаемость
- Изоляция тестов
- Использование best practices
```

### Генерация тестовых данных

```
Сгенерируй 10 тестовых пользователей для e-commerce приложения.
Каждый должен иметь: id, name, email, address, orderHistory.
orderHistory — массив из 0-5 заказов с продуктами.
Формат: TypeScript.
```

---

## Лучшие практики

### 1. Тестируйте поведение, не реализацию

```tsx
// ❌ Тестирование реализации
expect(component.state.count).toBe(1);
expect(instance.handleClick).toHaveBeenCalled();

// ✅ Тестирование поведения
expect(screen.getByText("Count: 1")).toBeInTheDocument();
```

### 2. Используйте testing-library queries по приоритету

```tsx
// 1. getByRole — самый надёжный
screen.getByRole("button", { name: /submit/i });

// 2. getByLabelText — для форм
screen.getByLabelText("Email");

// 3. getByText — для контента
screen.getByText("Welcome");

// 4. getByTestId — последний вариант
screen.getByTestId("submit-button");
```

### 3. Один assert на тест (в идеале)

```tsx
// ❌ Слишком много проверок в одном тесте
it("handles user flow", () => {
  render(<App />);
  fireEvent.click(screen.getByText("Login"));
  expect(screen.getByText("Welcome")).toBeInTheDocument();
  fireEvent.click(screen.getByText("Settings"));
  expect(screen.getByText("Profile")).toBeInTheDocument();
  fireEvent.click(screen.getByText("Logout"));
  expect(screen.getByText("Login")).toBeInTheDocument();
});

// ✅ Разбейте на отдельные тесты
it("shows welcome after login", () => { ... });
it("navigates to settings", () => { ... });
it("logs out successfully", () => { ... });
```

### 4. Очищайте состояние между тестами

```tsx
afterEach(() => {
  vi.clearAllMocks();
  vi.restoreAllMocks();
});
```

### 5. Используйте factory для тестовых данных

```ts
function createUser(overrides?: Partial<User>): User {
  return {
    id: crypto.randomUUID(),
    name: "Test User",
    email: "test@example.com",
    ...overrides,
  };
}

const user = createUser({ name: "Alice" });
const admin = createUser({ role: "admin" });
```

---

## Антипаттерны

### 1. Snapshot-тестирование всего

```tsx
// ❌ Snapshot ломается при каждом изменении
it("renders correctly", () => {
  const { container } = render(<App />);
  expect(container).toMatchSnapshot();
});

// ✅ Snapshot только для стабильных компонентов
it("renders button correctly", () => {
  const { container } = render(<Button variant="primary">Click</Button>);
  expect(container).toMatchSnapshot();
});
```

### 2. Тестирование деталей реализации

```tsx
// ❌ Зависимость от внутреннего состояния
expect(wrapper.find("input").state("value")).toBe("test");

// ✅ Проверка видимого поведения
expect(screen.getByDisplayValue("test")).toBeInTheDocument();
```

### 3. Игнорирование async/await

```tsx
// ❌ Не ждёт асинхронных операций
it("loads data", () => {
  render(<UserList />);
  expect(screen.getByText("Alice")).toBeInTheDocument(); // FAIL
});

// ✅ Ждёт появления элемента
it("loads data", async () => {
  render(<UserList />);
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

### 4. Мокирование всего

```tsx
// ❌ Излишнее мокирование
vi.mock("react", () => ({ useState: vi.fn() }));

// ✅ Мокирование только внешних зависимостей
vi.mock("@/lib/api", () => ({
  fetchUsers: vi.fn(),
}));
```

### 5. Хрупкие тесты

```tsx
// ❌ Зависимость от DOM-структуры
expect(container.querySelector("div > div > span")).toHaveTextContent("Hello");

// ✅ Зависимость от семантики
expect(screen.getByRole("heading", { name: "Hello" })).toBeInTheDocument();
```
