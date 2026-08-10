# Тестирование в Next.js

## Содержание

1. [Особенности тестирования Next.js](#особенности-тестирования-nextjs)
2. [Конфигурация Vitest для Next.js](#конфигурация-vitest-для-nextjs)
3. [Тестирование клиентских компонентов](#тестирование-клиентских-компонентов)
4. [Тестирование Server Components](#тестирование-server-components)
5. [Тестирование Server Actions](#тестирование-server-actions)
6. [Тестирование Route Handlers](#тестирование-route-handlers)
7. [Тестирование Middleware](#тестирование-middleware)
8. [Мокирование next/navigation](#мокирование-nextnavigation)
9. [Мокирование next/headers](#мокирование-nextheaders)
10. [Тестирование generateStaticParams и генерации](#тестирование-generatestaticparams-и-генерации)
11. [Тестирование revalidatePath и revalidateTag](#тестирование-revalidatepath-и-revalidatetag)
12. [Интеграция с MSW](#интеграция-с-msw)
13. [Тестирование App Router vs Pages Router](#тестирование-app-router-vs-pages-router)
14. [Тестирование с базами данных](#тестирование-с-базами-данных)
15. [Тестирование аутентификации](#тестирование-аутентификации)
16. [Тестирование метаданных](#тестирование-метаданных)
17. [Лучшие практики](#лучшие-практики)
18. [Антипаттерны](#антипаттерны)

---

## Особенности тестирования Next.js

Next.js добавляет слои абстракции поверх React, которые требуют особого подхода к тестированию:

| Компонент Next.js | Особенность тестирования |
|---|---|
| **Server Components** | Рендерятся на сервере, нельзя использовать Testing Library напрямую |
| **Server Actions** | Async-функции с `"use server"`, тестируются как unit-функции |
| **Route Handlers** | API-эндпоинты, тестируются как HTTP-запросы |
| **Middleware** | Выполняется на Edge, требует мокирования Request/Response |
| **next/navigation** | Хуки `useRouter`, `usePathname` нужно мокировать |
| **next/headers** | `cookies()`, `headers()` — серверные API, требуют мокирования |
| **generateStaticParams** | Функции генерации, тестируются как unit |
| **revalidatePath / revalidateTag** | Побочные эффекты, проверяются через моки |

### Что НЕ нужно тестировать

- **Фреймворк Next.js** — он уже протестирован командой Vercel
- **Внутренний механизм рендеринга** — RSC, гидратация, кэширование
- **Конфигурацию Next.js** — `next.config.js` не нуждается в тестах

### Что НУЖНО тестировать

- **Бизнес-логику** в Server Components и Server Actions
- **Валидацию данных** в Route Handlers
- **Маршрутизацию** в Middleware
- **Поведение** клиентских компонентов
- **Побочные эффекты** (revalidatePath, redirect)

---

## Конфигурация Vitest для Next.js

Vitest — это тестовый фреймворк, который работает поверх Vite. Для Next.js его нужно настроить так, чтобы он понимал JSX, алиасы путей и браузерное окружение. Главное отличие от Jest — скорость: Vitest использует нативный ESM и не требует транспиляции через Babel.

### Базовая конфигурация

Базовая конфигурация задаёт окружение `jsdom` (эмуляция браузера), глобальные `describe`/`it`/`expect` и алиас `@` для абсолютных импортов. Без `jsdom` тесты React-компонентов не будут работать, потому что не будет объекта `document`.

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

### Setup-файл

Setup-файл выполняется перед каждым тестом. Здесь подключаются matchers из `@testing-library/jest-dom` (например, `toBeInTheDocument`), вызывается `cleanup()` для удаления DOM между тестами и сбрасываются моки. Без `cleanup` DOM-элементы из предыдущих тестов будут просачиваться в следующие и вызывать ложные срабатывания.

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

### Мокирование Next.js API

Next.js предоставляет собственные API (`next/navigation`, `next/link`, `next/image`), которые не существуют в тестовой среде `jsdom`. Если их не замокать, тесты упадут с ошибкой модуля. Важно мокировать их именно в setup-файле, чтобы моки были доступны во всех тестах без повторения.

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach, vi } from "vitest";

// Мокирование next/navigation
vi.mock("next/navigation", () => ({
  useRouter: vi.fn(() => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn(),
    refresh: vi.fn(),
  })),
  usePathname: vi.fn(() => "/"),
  useSearchParams: vi.fn(() => new URLSearchParams()),
  useParams: vi.fn(() => ({})),
}));

// Мокирование next/link
vi.mock("next/link", () => ({
  default: ({ children, href }: { children: React.ReactNode; href: string }) =>
    `<a href="${href}">${children}</a>`,
}));

// Мокирование next/image
vi.mock("next/image", () => ({
  default: (props: React.ImgHTMLAttributes<HTMLImageElement>) =>
    `<img ${Object.entries(props).map(([k, v]) => `${k}="${v}"`).join(" ")} />`,
}));

afterEach(() => {
  cleanup();
  vi.clearAllMocks();
  vi.restoreAllMocks();
});
```

---

## Тестирование клиентских компонентов

Клиентские компоненты (с `"use client"`) тестируются как обычные React-компоненты. Это самый простой случай: рендерим через Testing Library, взаимодействуем через `userEvent`, проверяем результат.

Главный принцип — тестировать поведение, а не реализацию. Не проверяйте состояние `useState` напрямую, вместо этого проверяйте, что отображается на экране после действий пользователя.

### Базовый компонент

Компонент `SearchBar` использует хук `useRouter` из Next.js для навигации. В тесте мы должны замокать `next/navigation`, потому что в `jsdom` нет маршрутизатора Next.js.

```tsx
// src/app/components/SearchBar.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";

export function SearchBar() {
  const [query, setQuery] = useState("");
  const router = useRouter();

  const handleSearch = (e: React.FormEvent) => {
    e.preventDefault();
    router.push(`/search?q=${encodeURIComponent(query)}`);
  };

  return (
    <form onSubmit={handleSearch}>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <button type="submit">Search</button>
    </form>
  );
}
```

### Тест

```tsx
// src/app/components/SearchBar.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { SearchBar } from "./SearchBar";
import { useRouter } from "next/navigation";

vi.mock("next/navigation");

describe("SearchBar", () => {
  it("navigates to search page on submit", async () => {
    const user = userEvent.setup();
    const mockPush = vi.fn();
    vi.mocked(useRouter).mockReturnValue({
      push: mockPush,
      replace: vi.fn(),
      back: vi.fn(),
      prefetch: vi.fn(),
      refresh: vi.fn(),
    });

    render(<SearchBar />);

    await user.type(screen.getByPlaceholderText(/search/i), "react");
    await user.click(screen.getByRole("button", { name: /search/i }));

    expect(mockPush).toHaveBeenCalledWith("/search?q=react");
  });
});
```

---

## Тестирование Server Components

Server Components выполняются на сервере. Их нельзя рендерить через Testing Library напрямую. Вместо этого — вызываем как async-функцию и проверяем результат.

Ключевое отличие от клиентских компонентов: Server Component — это по сути async-функция, которая возвращает JSX. Мы можем вызвать её напрямую в тесте, а полученный JSX уже отрендерить через Testing Library. Это позволяет тестировать бизнес-логику (запросы к БД, условный рендеринг) без запуска сервера.

### Server Component

Компонент `UsersPage` — async-функция. Она не имеет доступа к хукам React (useState, useEffect), но может выполнять async-операции напрямую в теле компонента.

```tsx
// src/app/users/page.tsx
import { db } from "@/lib/db";

export default async function UsersPage() {
  const users = await db.users.findMany();

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Тест

```tsx
// src/app/users/page.test.tsx
import { render, screen } from "@testing-library/react";
import UsersPage from "./page";

vi.mock("@/lib/db", () => ({
  db: {
    users: {
      findMany: vi.fn(),
    },
  },
}));

import { db } from "@/lib/db";

describe("UsersPage", () => {
  it("displays list of users", async () => {
    vi.mocked(db.users.findMany).mockResolvedValue([
      { id: 1, name: "Alice" },
      { id: 2, name: "Bob" },
    ]);

    const page = await UsersPage();
    render(page);

    expect(screen.getByText("Alice")).toBeInTheDocument();
    expect(screen.getByText("Bob")).toBeInTheDocument();
  });

  it("shows empty state", async () => {
    vi.mocked(db.users.findMany).mockResolvedValue([]);

    const page = await UsersPage();
    render(page);

    expect(screen.queryByRole("listitem")).not.toBeInTheDocument();
  });
});
```

### Server Component с параметрами

Динамические маршруты в Next.js получают `params` как пропс. В тесте мы передаём их вручную. Также здесь важно тестировать ветку с `notFound()` — она выбрасывает специальное исключение, которое Next.js превращает в 404-страницу.

```tsx
// src/app/users/[id]/page.tsx
import { db } from "@/lib/db";
import { notFound } from "next/navigation";

export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await db.users.findById(params.id);

  if (!user) {
    notFound();
  }

  return <div>{user.name}</div>;
}
```

### Тест

```tsx
// src/app/users/[id]/page.test.tsx
import { render, screen } from "@testing-library/react";
import UserPage from "./page";

vi.mock("@/lib/db", () => ({
  db: {
    users: {
      findById: vi.fn(),
    },
  },
}));

vi.mock("next/navigation", () => ({
  notFound: vi.fn(() => {
    throw new Error("NEXT_NOT_FOUND");
  }),
}));

import { db } from "@/lib/db";
import { notFound } from "next/navigation";

describe("UserPage", () => {
  it("displays user by id", async () => {
    vi.mocked(db.users.findById).mockResolvedValue({ id: "1", name: "Alice" });

    const page = await UserPage({ params: { id: "1" } });
    render(page);

    expect(screen.getByText("Alice")).toBeInTheDocument();
    expect(db.users.findById).toHaveBeenCalledWith("1");
  });

  it("calls notFound for unknown user", async () => {
    vi.mocked(db.users.findById).mockResolvedValue(null);

    await expect(UserPage({ params: { id: "999" } })).rejects.toThrow("NEXT_NOT_FOUND");
    expect(notFound).toHaveBeenCalled();
  });
});
```

---

## Тестирование Server Actions

Server Actions — это async-функции с директивой `"use server"`. Тестируются как обычные unit-функции.

Server Actions — это мутации данных в Next.js. Они принимают `FormData` или обычные аргументы и выполняют операции на сервере: запись в БД, валидацию, ревалидацию кэша. Тестировать их нужно как unit-функции: вызываем напрямую, проверяем результат и побочные эффекты. Не нужно запускать HTTP-сервер — это замедлит тесты и добавит ненужной сложности.

### Server Action

Функция `createPost` принимает `FormData`, валидирует через Zod, создаёт пост в БД и ревалидирует кэш. В тесте мокаем БД и `revalidatePath`, затем проверяем оба сценария: успешное создание и ошибку валидации.

```ts
// src/app/actions/createPost.ts
"use server";

import { db } from "@/lib/db";
import { revalidatePath } from "next/cache";
import { z } from "zod";

const createPostSchema = z.object({
  title: z.string().min(1, "Title is required"),
  content: z.string().min(1, "Content is required"),
});

export async function createPost(formData: FormData) {
  const data = createPostSchema.parse({
    title: formData.get("title"),
    content: formData.get("content"),
  });

  const post = await db.posts.create({
    data: {
      title: data.title,
      content: data.content,
    },
  });

  revalidatePath("/posts");

  return { success: true, post };
}
```

### Тест

```ts
// src/app/actions/createPost.test.ts
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

import { db } from "@/lib/db";
import { revalidatePath } from "next/cache";

describe("createPost", () => {
  it("creates a post and revalidates path", async () => {
    vi.mocked(db.posts.create).mockResolvedValue({
      id: 1,
      title: "Test Post",
      content: "Content",
    });

    const formData = new FormData();
    formData.append("title", "Test Post");
    formData.append("content", "Content");

    const result = await createPost(formData);

    expect(result.success).toBe(true);
    expect(result.post.title).toBe("Test Post");
    expect(db.posts.create).toHaveBeenCalledWith({
      data: { title: "Test Post", content: "Content" },
    });
    expect(revalidatePath).toHaveBeenCalledWith("/posts");
  });

  it("throws error for invalid data", async () => {
    const formData = new FormData();
    formData.append("title", "");
    formData.append("content", "");

    await expect(createPost(formData)).rejects.toThrow();
    expect(db.posts.create).not.toHaveBeenCalled();
  });
});
```

### Server Action с авторизацией

Большинство Server Actions требуют проверки аутентификации. В тесте мокаем функцию `auth()` и проверяем обе ветки: авторизованный пользователь и неавторизованный. Это важно, потому что отсутствие проверки авторизации — частая уязвимость.

```ts
// src/app/actions/updateProfile.ts
"use server";

import { auth } from "@/lib/auth";
import { db } from "@/lib/db";

export async function updateProfile(formData: FormData) {
  const session = await auth();

  if (!session) {
    return { error: "Unauthorized" };
  }

  const name = formData.get("name") as string;

  await db.users.update({
    where: { id: session.userId },
    data: { name },
  });

  return { success: true };
}
```

### Тест

```ts
// src/app/actions/updateProfile.test.ts
import { updateProfile } from "./updateProfile";

vi.mock("@/lib/auth", () => ({
  auth: vi.fn(),
}));

vi.mock("@/lib/db", () => ({
  db: {
    users: {
      update: vi.fn(),
    },
  },
}));

import { auth } from "@/lib/auth";
import { db } from "@/lib/db";

describe("updateProfile", () => {
  it("updates profile for authenticated user", async () => {
    vi.mocked(auth).mockResolvedValue({ userId: "1" } as any);
    vi.mocked(db.users.update).mockResolvedValue({ id: "1", name: "Alice" });

    const formData = new FormData();
    formData.append("name", "Alice");

    const result = await updateProfile(formData);

    expect(result.success).toBe(true);
    expect(db.users.update).toHaveBeenCalledWith({
      where: { id: "1" },
      data: { name: "Alice" },
    });
  });

  it("returns error for unauthenticated user", async () => {
    vi.mocked(auth).mockResolvedValue(null);

    const formData = new FormData();
    formData.append("name", "Alice");

    const result = await updateProfile(formData);

    expect(result.error).toBe("Unauthorized");
    expect(db.users.update).not.toHaveBeenCalled();
  });
});
```

---

## Тестирование Route Handlers

Route Handlers — это API-эндпоинты в Next.js App Router.

Route Handlers экспортируют функции `GET`, `POST`, `PUT`, `DELETE` и т.д. Каждая получает `NextRequest` и возвращает `NextResponse`. Тестировать их можно двумя способами: (1) вызвать функцию напрямую с моком Request — быстро, но менее реалистично; (2) запустить реальный сервер через `fetch` — медленно, но проверяет весь HTTP-стек. Для unit-тестов подходит первый вариант.

### Route Handler

Обработчик `users/route.ts` экспортирует `GET` и `POST`. `GET` не принимает аргументов, `POST` получает `NextRequest`. В тесте конструируем `Request` вручную — это стандартный Web API, не требует Next.js-сервера.

```ts
// src/app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";
import { db } from "@/lib/db";

export async function GET() {
  const users = await db.users.findMany();
  return NextResponse.json(users);
}

export async function POST(request: NextRequest) {
  const body = await request.json();

  if (!body.name) {
    return NextResponse.json({ error: "Name is required" }, { status: 400 });
  }

  const user = await db.users.create({ data: { name: body.name } });
  return NextResponse.json(user, { status: 201 });
}
```

### Тест

```ts
// src/app/api/users/route.test.ts
import { GET, POST } from "./route";

vi.mock("@/lib/db", () => ({
  db: {
    users: {
      findMany: vi.fn(),
      create: vi.fn(),
    },
  },
}));

import { db } from "@/lib/db";

describe("Users API", () => {
  describe("GET", () => {
    it("returns list of users", async () => {
      vi.mocked(db.users.findMany).mockResolvedValue([
        { id: 1, name: "Alice" },
        { id: 2, name: "Bob" },
      ]);

      const response = await GET();
      const data = await response.json();

      expect(response.status).toBe(200);
      expect(data).toEqual([
        { id: 1, name: "Alice" },
        { id: 2, name: "Bob" },
      ]);
    });
  });

  describe("POST", () => {
    it("creates a user", async () => {
      vi.mocked(db.users.create).mockResolvedValue({ id: 1, name: "Alice" });

      const request = new Request("http://localhost:3000/api/users", {
        method: "POST",
        body: JSON.stringify({ name: "Alice" }),
      });

      const response = await POST(request as any);
      const data = await response.json();

      expect(response.status).toBe(201);
      expect(data.name).toBe("Alice");
    });

    it("returns 400 for missing name", async () => {
      const request = new Request("http://localhost:3000/api/users", {
        method: "POST",
        body: JSON.stringify({}),
      });

      const response = await POST(request as any);
      const data = await response.json();

      expect(response.status).toBe(400);
      expect(data.error).toBe("Name is required");
    });
  });
});
```

---

## Тестирование Middleware

Middleware выполняется на Edge Runtime. Тестируется через Request/Response.

Middleware в Next.js перехватывает запросы до того, как они достигнут страницы. Типичные задачи: проверка авторизации, редиректы, i18n. Тестировать middleware просто — это чистая функция, которая принимает `NextRequest` и возвращает `NextResponse`. Не нужно мокировать ничего, кроме самого Request.

### Middleware

Функция `middleware` проверяет cookie `token` и редиректит на `/login`, если его нет. В тесте создаём `NextRequest` с нужным URL и cookies, затем проверяем статус и заголовок `location`.

```ts
// src/middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("token")?.value;

  if (!token && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*"],
};
```

### Тест

```ts
// src/middleware.test.ts
import { middleware } from "./middleware";
import { NextRequest } from "next/server";

describe("middleware", () => {
  it("redirects to login for unauthenticated users", () => {
    const request = new NextRequest("http://localhost:3000/dashboard");

    const response = middleware(request);

    expect(response.status).toBe(307);
    expect(response.headers.get("location")).toBe("http://localhost:3000/login");
  });

  it("allows authenticated users to access dashboard", () => {
    const request = new NextRequest("http://localhost:3000/dashboard");
    request.cookies.set("token", "valid-token");

    const response = middleware(request);

    expect(response.status).toBe(200);
  });

  it("allows access to public pages", () => {
    const request = new NextRequest("http://localhost:3000/about");

    const response = middleware(request);

    expect(response.status).toBe(200);
  });
});
```

---

## Мокирование next/navigation

Модуль `next/navigation` содержит хуки для клиентских компонентов и функции для серверных. Все они зависят от контекста Next.js, которого нет в тестовой среде. Поэтому каждый хук нужно мокировать через `vi.mock` и настраивать возвращаемое значение через `vi.mocked().mockReturnValue()`.

### useRouter

`useRouter` даёт доступ к навигации: `push`, `replace`, `back`, `prefetch`, `refresh`. В тесте мокаем его и проверяем, что компонент вызывает нужный метод с правильным URL.

```ts
import { useRouter } from "next/navigation";

vi.mock("next/navigation", () => ({
  useRouter: vi.fn(() => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn(),
    refresh: vi.fn(),
  })),
}));

// Использование в тесте
it("navigates on submit", async () => {
  const mockPush = vi.fn();
  vi.mocked(useRouter).mockReturnValue({
    push: mockPush,
    replace: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn(),
    refresh: vi.fn(),
  });

  render(<MyComponent />);

  // ... действия

  expect(mockPush).toHaveBeenCalledWith("/dashboard");
});
```

### usePathname

`usePathname` возвращает текущий путь. Используется для подсветки активного пункта меню, условного рендеринга. В тесте мокаем нужное значение и проверяем, что компонент корректно реагирует на изменение пути.

```ts
import { usePathname } from "next/navigation";

vi.mock("next/navigation", () => ({
  usePathname: vi.fn(() => "/"),
}));

// Использование в тесте
it("highlights current page in nav", () => {
  vi.mocked(usePathname).mockReturnValue("/about");

  render(<Navigation />);

  expect(screen.getByRole("link", { name: /about/i })).toHaveAttribute("aria-current", "page");
});
```

### useSearchParams

`useSearchParams` возвращает объект `URLSearchParams` с параметрами из URL. Используется для фильтрации, поиска, сортировки. В тесте создаём `URLSearchParams` вручную и передаём в мок.

```ts
import { useSearchParams } from "next/navigation";

vi.mock("next/navigation", () => ({
  useSearchParams: vi.fn(() => new URLSearchParams()),
}));

// Использование в тесте
it("filters by search query", () => {
  vi.mocked(useSearchParams).mockReturnValue(new URLSearchParams("q=react"));

  render(<SearchResults />);

  expect(screen.getByText(/results for "react"/i)).toBeInTheDocument();
});
```

### useParams

`useParams` возвращает параметры динамического маршрута (например, `{ id: "42" }` для `/users/[id]`). В тесте мокаем нужные параметры и проверяем, что компонент загружает правильные данные.

```ts
import { useParams } from "next/navigation";

vi.mock("next/navigation", () => ({
  useParams: vi.fn(() => ({ id: "1" })),
}));

// Использование в тесте
it("loads user by id from params", () => {
  vi.mocked(useParams).mockReturnValue({ id: "42" });

  render(<UserPage />);

  expect(screen.getByText("User 42")).toBeInTheDocument();
});
```

### redirect и notFound

`redirect` и `notFound` — серверные функции, которые прерывают выполнение, выбрасывая специальное исключение. В тесте мокаем их так, чтобы они выбрасывали ошибку с известным сообщением, затем проверяем через `rejects.toThrow`.

```ts
vi.mock("next/navigation", () => ({
  redirect: vi.fn((url: string) => {
    throw new Error(`REDIRECT: ${url}`);
  }),
  notFound: vi.fn(() => {
    throw new Error("NEXT_NOT_FOUND");
  }),
}));
```

---

## Мокирование next/headers

Модуль `next/headers` предоставляет серверные API для чтения cookies и HTTP-заголовков запроса. Эти функции работают только в контексте серверного компонента или Server Action. В тестах их нужно мокировать, чтобы эмулировать разные сценарии: авторизованный/неавторизованный пользователь, наличие/отсутствие токена.

### cookies

`cookies()` возвращает объект для чтения/записи cookies. В тесте мокаем метод `get`, чтобы возвращать нужное значение для конкретных cookie-имён.

```ts
vi.mock("next/headers", () => ({
  cookies: vi.fn(() => ({
    get: vi.fn((name: string) => {
      if (name === "token") return { value: "mock-token" };
      return undefined;
    }),
    set: vi.fn(),
    delete: vi.fn(),
  })),
}));
```

### headers

`headers()` возвращает объект для чтения HTTP-заголовков. Используется для получения токенов авторизации, информации о локали, пользовательском агенте. Мокаем аналогично `cookies`.

```ts
vi.mock("next/headers", () => ({
  headers: vi.fn(() => ({
    get: vi.fn((name: string) => {
      if (name === "authorization") return "Bearer mock-token";
      return null;
    }),
  })),
}));
```

---

## Тестирование generateStaticParams и генерации

`generateStaticParams` используется для статической генерации динамических маршрутов. Она вызывается во время сборки и возвращает массив параметров. Тестируется как обычная async-функция — мокаем БД, проверяем, что возвращаются все нужные параметры.

### generateStaticParams

Функция запрашивает все посты из БД и формирует массив `{ id }`. Тест проверяет, что для каждого поста создаётся правильный параметр.

```tsx
// src/app/posts/[id]/page.tsx
import { db } from "@/lib/db";

export async function generateStaticParams() {
  const posts = await db.posts.findMany({ select: { id: true } });
  return posts.map((post) => ({ id: post.id.toString() }));
}

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await db.posts.findById(params.id);
  return <article>{post.title}</article>;
}
```

### Тест

```ts
// src/app/posts/[id]/page.test.ts
import { generateStaticParams } from "./page";

vi.mock("@/lib/db", () => ({
  db: {
    posts: {
      findMany: vi.fn(),
      findById: vi.fn(),
    },
  },
}));

import { db } from "@/lib/db";

describe("generateStaticParams", () => {
  it("generates params for all posts", async () => {
    vi.mocked(db.posts.findMany).mockResolvedValue([
      { id: 1 },
      { id: 2 },
      { id: 3 },
    ]);

    const params = await generateStaticParams();

    expect(params).toEqual([
      { id: "1" },
      { id: "2" },
      { id: "3" },
    ]);
  });
});
```

---

## Тестирование revalidatePath и revalidateTag

После мутации данных нужно инвалидировать кэш, чтобы пользователи увидели актуальные данные. `revalidatePath` инвалидирует кэш конкретного пути, `revalidateTag` — по тегу. В тестах проверяем, что эти функции вызываются с правильными аргументами после выполнения действия.

### Компонент с revalidatePath

Функция `deletePost` удаляет пост и ревалидирует путь `/posts` и корневой layout. В тесте проверяем оба вызова `revalidatePath` — это гарантирует, что кэш будет корректно инвалидирован.

```ts
// src/app/actions/deletePost.ts
"use server";

import { db } from "@/lib/db";
import { revalidatePath } from "next/cache";

export async function deletePost(postId: string) {
  await db.posts.delete({ where: { id: postId } });
  revalidatePath("/posts");
  revalidatePath("/", "layout");
}
```

### Тест

```ts
// src/app/actions/deletePost.test.ts
import { deletePost } from "./deletePost";

vi.mock("@/lib/db", () => ({
  db: {
    posts: {
      delete: vi.fn(),
    },
  },
}));

vi.mock("next/cache", () => ({
  revalidatePath: vi.fn(),
  revalidateTag: vi.fn(),
}));

import { db } from "@/lib/db";
import { revalidatePath } from "next/cache";

describe("deletePost", () => {
  it("deletes post and revalidates paths", async () => {
    vi.mocked(db.posts.delete).mockResolvedValue({ id: "1" });

    await deletePost("1");

    expect(db.posts.delete).toHaveBeenCalledWith({ where: { id: "1" } });
    expect(revalidatePath).toHaveBeenCalledWith("/posts");
    expect(revalidatePath).toHaveBeenCalledWith("/", "layout");
  });
});
```

---

## Интеграция с MSW

MSW (Mock Service Worker) перехватывает HTTP-запросы на уровне сети и возвращает мокированные ответы. В отличие от `vi.mock(global.fetch)`, MSW тестирует реальный код отправки запросов — включая заголовки, тело, обработку ошибок. Это более реалистичный подход, но он требует настройки.

### Настройка MSW для Next.js

Настройка состоит из трёх частей: handlers (описание моков), server (настройка MSW для Node.js) и setup-файл (запуск/остановка сервера). Handlers можно переопределять в отдельных тестах через `server.use()`.

```ts
// src/test/mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "Alice" },
      { id: 2, name: "Bob" },
    ]);
  }),

  http.post("/api/posts", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 1, ...body }, { status: 201 });
  }),
];
```

```ts
// src/test/mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach, beforeAll, afterAll } from "vitest";
import { server } from "./mocks/server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => {
  server.resetHandlers();
  cleanup();
  vi.clearAllMocks();
});
afterAll(() => server.close());
```

### Использование в тестах

```tsx
// src/app/users/page.test.tsx
import { render, screen } from "@testing-library/react";
import UsersPage from "./page";

describe("UsersPage", () => {
  it("displays users from API", async () => {
    const page = await UsersPage();
    render(page);

    expect(await screen.findByText("Alice")).toBeInTheDocument();
    expect(screen.getByText("Bob")).toBeInTheDocument();
  });
});
```

---

## Тестирование App Router vs Pages Router

В Next.js есть две модели маршрутизации: App Router (новый, рекомендуется) и Pages Router (legacy). Подходы к тестированию отличаются.

### App Router (рекомендуется)

В App Router данные загружаются напрямую в Server Component через async/await. Тестирование сводится к вызову компонента как функции и проверке результата.

### Pages Router (legacy)

В Pages Router используется `getServerSideProps` или `getStaticProps` для загрузки данных. Эти функции возвращают объект `{ props }`, который передаётся в компонент. Тестируем `getServerSideProps` как unit-функцию.

```tsx
// src/app/users/page.tsx
export default async function UsersPage() {
  const users = await fetchUsers();
  return <UserList users={users} />;
}
```

### Pages Router (legacy)

```tsx
// src/pages/users.tsx
import { GetServerSideProps } from "next";

export const getServerSideProps: GetServerSideProps = async () => {
  const users = await fetchUsers();
  return { props: { users } };
};

export default function UsersPage({ users }) {
  return <UserList users={users} />;
}
```

### Тест getServerSideProps

`getServerSideProps` получает контекст запроса (query, params, req, res). В тесте передаём пустой объект `as any`, если контекст не используется. Проверяем, что функция возвращает правильную структуру `{ props: { ... } }`.

```ts
// src/pages/users.test.ts
import { getServerSideProps } from "./users";

vi.mock("@/lib/api", () => ({
  fetchUsers: vi.fn(),
}));

import { fetchUsers } from "@/lib/api";

describe("getServerSideProps", () => {
  it("returns users as props", async () => {
    vi.mocked(fetchUsers).mockResolvedValue([{ id: 1, name: "Alice" }]);

    const result = await getServerSideProps({} as any);

    expect(result).toEqual({
      props: { users: [{ id: 1, name: "Alice" }] },
    });
  });
});
```

---

## Тестирование с базами данных

В тестах не подключаемся к реальной БД — это медленно и ненадёжно. Вместо этого мокаем клиент БД. Структура мока зависит от ORM.

### Мокирование Prisma

Prisma имеет цепочечный API (`prisma.user.findMany()`). Мокаем каждый метод как `vi.fn()`. В тестах настраиваем возвращаемые значения через `mockResolvedValue`.

### Мокирование Drizzle

Drizzle использует цепочки вызовов (`db.select().from().where()`). Каждый метод в цепочке возвращает объект с следующим методом. В моке нужно воспроизвести эту цепочку.

```ts
vi.mock("@/lib/prisma", () => ({
  prisma: {
    user: {
      findMany: vi.fn(),
      findById: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
    },
  },
}));
```

### Мокирование Drizzle

```ts
vi.mock("@/lib/db", () => ({
  db: {
    select: vi.fn(() => ({
      from: vi.fn(() => ({
        where: vi.fn(() => Promise.resolve([])),
      })),
    })),
    insert: vi.fn(() => ({
      values: vi.fn(() => Promise.resolve({})),
    })),
  },
}));
```

---

## Тестирование аутентификации

Аутентификация — критическая часть приложения. В тестах мокаем функцию получения сессии и проверяем обе ветки: авторизованный и неавторизованный пользователь. Это защищает от регрессий в логике доступа.

### Мокирование NextAuth

NextAuth предоставляет `getServerSession` для серверных компонентов и `useSession` для клиентских. В тестах Server Components мокаем `getServerSession`, в тестах клиентских компонентов — `useSession`.

```ts
vi.mock("next-auth", () => ({
  getServerSession: vi.fn(),
}));

vi.mock("@/lib/auth", () => ({
  auth: vi.fn(),
}));

import { getServerSession } from "next-auth";
import { auth } from "@/lib/auth";

// Использование в тесте
it("returns data for authenticated user", async () => {
  vi.mocked(auth).mockResolvedValue({ user: { id: "1" } } as any);

  const result = await getProtectedData();

  expect(result).toBeDefined();
});
```

---

## Тестирование метаданных

`generateMetadata` — async-функция, которая возвращает объект метаданных (title, description, og:image и т.д.). Используется для SEO и социальных сетей. Тестируется как обычная async-функция: мокаем источник данных, вызываем `generateMetadata`, проверяем результат.

### generateMetadata

Функция получает `params` и загружает данные из БД для формирования метаданных. В тесте проверяем, что title и description корректно формируются из данных поста.

```tsx
// src/app/posts/[id]/page.tsx
import { db } from "@/lib/db";
import { Metadata } from "next";

export async function generateMetadata({ params }: { params: { id: string } }): Promise<Metadata> {
  const post = await db.posts.findById(params.id);
  return {
    title: post.title,
    description: post.excerpt,
  };
}
```

### Тест

```ts
// src/app/posts/[id]/page.test.ts
import { generateMetadata } from "./page";

vi.mock("@/lib/db", () => ({
  db: {
    posts: {
      findById: vi.fn(),
    },
  },
}));

import { db } from "@/lib/db";

describe("generateMetadata", () => {
  it("returns metadata for post", async () => {
    vi.mocked(db.posts.findById).mockResolvedValue({
      title: "Test Post",
      excerpt: "This is a test post",
    });

    const metadata = await generateMetadata({ params: { id: "1" } });

    expect(metadata).toEqual({
      title: "Test Post",
      description: "This is a test post",
    });
  });
});
```

---

## Лучшие практики

### 1. Тестируйте бизнес-логику, не фреймворк

```ts
// ❌ Тестирует Next.js
it("renders server component", async () => {
  const html = await renderToString(<Page />);
  expect(html).toContain("div");
});

// ✅ Тестирует бизнес-логику
it("fetches and displays users", async () => {
  vi.mocked(db.users.findMany).mockResolvedValue([{ id: 1, name: "Alice" }]);
  const page = await UsersPage();
  render(page);
  expect(screen.getByText("Alice")).toBeInTheDocument();
});
```

### 2. Мокируйте next/navigation в setup-файле

```ts
// src/test/setup.ts
vi.mock("next/navigation", () => ({
  useRouter: vi.fn(() => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn(),
    refresh: vi.fn(),
  })),
  usePathname: vi.fn(() => "/"),
  useSearchParams: vi.fn(() => new URLSearchParams()),
}));
```

### 3. Используйте MSW для API-запросов

```ts
// ✅ MSW — реалистичное мокирование
server.use(
  http.get("/api/users", () => HttpResponse.json([{ id: 1 }]))
);

// ❌ Ручной mock fetch
global.fetch = vi.fn().mockResolvedValue({
  ok: true,
  json: async () => [{ id: 1 }],
});
```

### 4. Тестируйте Server Actions как unit-функции

```ts
// ✅ Server Action — обычная async-функция
const result = await createPost(formData);
expect(result.success).toBe(true);

// ❌ Пытается тестировать через HTTP
const response = await fetch("/api/posts", { method: "POST" });
```

### 5. Проверяйте побочные эффекты

```ts
// ✅ Проверяет revalidatePath
await deletePost("1");
expect(revalidatePath).toHaveBeenCalledWith("/posts");

// ❌ Игнорирует побочные эффекты
await deletePost("1");
expect(db.posts.delete).toHaveBeenCalled();
```

---

## Антипаттерны

### 1. Тестирование фреймворка

```ts
// ❌ Тестирует Next.js
it("calls next/navigation redirect", async () => {
  await myAction();
  expect(redirect).toHaveBeenCalledWith("/login");
});

// ✅ Тестирует поведение
it("redirects unauthenticated users", async () => {
  vi.mocked(auth).mockResolvedValue(null);
  await expect(myAction()).rejects.toThrow("REDIRECT: /login");
});
```

### 2. Избыточное мокирование

```ts
// ❌ Мокирует всё
vi.mock("next/navigation");
vi.mock("next/headers");
vi.mock("next/cache");
vi.mock("next-auth");

// ✅ Мокирует только то, что нужно для конкретного теста
vi.mock("@/lib/auth", () => ({
  auth: vi.fn().mockResolvedValue(null),
}));
```

### 3. Игнорирование типов

```ts
// ❌ Нет типизации моков
vi.mock("next/navigation", () => ({
  useRouter: () => ({}),
}));

// ✅ Типизация моков
vi.mock("next/navigation", () => ({
  useRouter: vi.fn((): AppRouterInstance => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn(),
    refresh: vi.fn(),
  })),
}));
```

### 4. Тестирование Route Handlers через HTTP

```ts
// ❌ Запускает реальный сервер
const response = await fetch("http://localhost:3000/api/users");

// ✅ Вызывает handler напрямую
const response = await GET();
```

### 5. Забытые моки в setup-файле

```ts
// ❌ next/navigation не замокан — ошибки в тестах
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
// Забыли vi.mock("next/navigation")

// ✅ Все Next.js API замоканые в setup
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
vi.mock("next/navigation", () => ({ ... }));
vi.mock("next/link", () => ({ ... }));
vi.mock("next/image", () => ({ ... }));
```
