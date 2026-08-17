# Тестирование в Nuxt 3

## Содержание

1. [Особенности тестирования Nuxt](#особенности-тестирования-nuxt)
2. [Конфигурация Vitest для Nuxt](#конфигурация-vitest-для-nuxt)
3. [Тестирование компонентов](#тестирование-компонентов)
4. [Тестирование composables](#тестирование-composables)
5. [Тестирование server API routes](#тестирование-server-api-routes)
6. [Тестирование middleware](#тестирование-middleware)
7. [Тестирование plugins](#тестирование-plugins)
8. [Тестирование useAsyncData / useFetch](#тестирование-useasyncdata--usefetch)
9. [Мокирование Nuxt API](#мокирование-nuxt-api)
10. [Тестирование NuxtLink](#тестирование-nuxtlink)
11. [Лучшие практики](#лучшие-практики)
12. [Антипаттерны](#антипаттерны)

---

## Особенности тестирования Nuxt

Nuxt 3 — это метафреймворк поверх Vue 3. Он добавляет слои абстракции, которые требуют особого подхода к тестированию:

| Компонент Nuxt | Особенность тестирования |
|---|---|
| **Auto-imports** | Composables и компоненты импортируются автоматически — в тестах это нужно учитывать |
| **Server routes** | API-эндпоинты в `server/api/` — тестируются через `registerEndpoint` или HTTP-запросы |
| **Middleware** | Код в `middleware/` — чистые функции `Request → Response` |
| **Plugins** | Регистрируются при старте приложения — нужна интеграционная среда |
| **useAsyncData / useFetch** | Работают с контекстом Nuxt — требуют `mountSuspended` |
| **SSR** | Компоненты рендерятся на сервере — нужно дожидаться завершения асинхронных операций |
| **Nuxt runtime config** | `useRuntimeConfig()` читает переменные окружения — нужно мокировать |

### Что НЕ нужно тестировать

- **Фреймворк Nuxt** — он уже протестирован командой Nuxt
- **Внутренний механизм рендеринга** — гидратация, автоимпорты, Nitro
- **Конфигурацию Nuxt** — `nuxt.config.ts` не нуждается в unit-тестах

### Что НУЖНО тестировать

- **Бизнес-логику** в composables и server routes
- **Валидацию данных** в API-эндпоинтах
- **Поведение** клиентских компонентов
- **Побочные эффекты** middleware и plugins
- **Интеграцию** между composables и UI

---

## Конфигурация Vitest для Nuxt

Для Nuxt используется официальный модуль `@nuxt/test-utils/module`. Он автоматически настраивает Vitest для работы с Nuxt: auto-imports, `.vue` файлы, runtime config, серверную часть.

### Установка

```bash
npm install -D @nuxt/test-utils vitest @vue/test-utils
```

### Базовая конфигурация

```ts
// vitest.config.ts
import { defineVitestConfig } from "@nuxt/test-utils/config";

export default defineVitestConfig({
  test: {
    environment: "nuxt",
    globals: true,
  },
});
```

> 💡 `environment: "nuxt"` — специальное окружение от `@nuxt/test-utils`, которое эмулирует Nuxt-рантайм. Это отличается от обычного `jsdom` для Vue.

### Конфигурация с явным rootDir

```ts
// vitest.config.ts
import { defineVitestConfig } from "@nuxt/test-utils/config";

export default defineVitestConfig({
  test: {
    environment: "nuxt",
    environmentOptions: {
      nuxt: {
        rootDir: ".",
        domEnvironment: "happy-dom",
      },
    },
    globals: true,
  },
});
```

### Скрипты в package.json

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:unit": "vitest run --testNamePattern unit"
  }
}
```

---

## Тестирование компонентов

Клиентские компоненты в Nuxt тестируются почти так же, как обычные Vue-компоненты. Но если компонент использует `useAsyncData`, `useFetch` или другие Nuxt-composables, нужен `mountSuspended`.

### Базовый компонент

```vue
<!-- SearchBar.vue -->
<template>
  <form @submit.prevent="handleSubmit">
    <input v-model="query" placeholder="Search..." />
    <button type="submit">Search</button>
  </form>
</template>

<script setup>
import { ref } from "vue";

const query = ref("");
const emit = defineEmits(["search"]);

function handleSubmit() {
  emit("search", query.value);
}
</script>
```

### Тест

```ts
import { mount } from "@vue/test-utils";
import { describe, it, expect } from "vitest";
import SearchBar from "./SearchBar.vue";

describe("SearchBar", () => {
  it("emits search event on submit", async () => {
    const wrapper = mount(SearchBar);

    await wrapper.find("input").setValue("nuxt");
    await wrapper.find("form").trigger("submit");

    expect(wrapper.emitted("search")).toHaveLength(1);
    expect(wrapper.emitted("search")[0]).toEqual(["nuxt"]);
  });
});
```

### mountSuspended для SSR-компонентов

```vue
<!-- UserProfile.vue -->
<template>
  <div>
    <h1>{{ user?.name }}</h1>
    <p>{{ user?.email }}</p>
  </div>
</template>

<script setup>
const { data: user } = await useFetch("/api/user/1");
</script>
```

```ts
import { mountSuspended, registerEndpoint } from "@nuxt/test-utils";
import UserProfile from "./UserProfile.vue";

describe("UserProfile", () => {
  registerEndpoint("/api/user/1", () => ({
    id: 1,
    name: "Alice",
    email: "alice@example.com",
  }));

  it("renders user data", async () => {
    const wrapper = await mountSuspended(UserProfile);

    expect(wrapper.text()).toContain("Alice");
    expect(wrapper.text()).toContain("alice@example.com");
  });
});
```

> 💡 `mountSuspended` ждёт завершения всех асинхронных операций в компоненте (например, `useFetch`) перед возвратом wrapper. Это критично для SSR-компонентов.

---

## Тестирование composables

Composables — сердце бизнес-логики Nuxt-приложений. Их можно тестировать двумя способами: напрямую через `mountSuspended` в тестовом компоненте или с помощью `mockNuxtImport`.

### Через тестовый компонент

```ts
// composables/useCounter.js
export function useCounter() {
  const count = ref(0);
  const increment = () => count.value++;
  return { count, increment };
}
```

```ts
import { mountSuspended } from "@nuxt/test-utils";
import { defineComponent } from "vue";
import { useCounter } from "./composables/useCounter";

it("useCounter increments", async () => {
  const TestComponent = defineComponent({
    setup() {
      return useCounter();
    },
    template: "<button @click=\"increment\">{{ count }}</button>",
  });

  const wrapper = await mountSuspended(TestComponent);

  expect(wrapper.text()).toBe("0");

  await wrapper.find("button").trigger("click");

  expect(wrapper.text()).toBe("1");
});
```

### Мокирование Nuxt-composables

```ts
import { mockNuxtImport } from "@nuxt/test-utils";

mockNuxtImport("useRuntimeConfig", () => {
  return () => ({
    public: {
      apiBaseUrl: "https://api.example.com",
    },
  });
});

it("uses runtime config", () => {
  const config = useRuntimeConfig();
  expect(config.public.apiBaseUrl).toBe("https://api.example.com");
});
```

---

## Тестирование server API routes

Server routes в Nuxt — это обработчики на базе h3. Их можно тестировать через `registerEndpoint` или напрямую вызывая обработчик.

### Server route

```ts
// server/api/users.get.ts
import { defineEventHandler } from "h3";

export default defineEventHandler(async () => {
  return [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" },
  ];
});
```

### Тест через registerEndpoint

```ts
import { registerEndpoint } from "@nuxt/test-utils";
import { $fetch } from "ofetch";

describe("Users API", () => {
  registerEndpoint("/api/users", defineEventHandler(() => [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" },
  ]));

  it("returns list of users", async () => {
    const users = await $fetch("/api/users");

    expect(users).toHaveLength(2);
    expect(users[0].name).toBe("Alice");
  });
});
```

### Тест обработчика напрямую

```ts
// server/api/users.post.ts
import { defineEventHandler, readBody } from "h3";

export default defineEventHandler(async (event) => {
  const body = await readBody(event);

  if (!body.name) {
    throw createError({ statusCode: 400, statusMessage: "Name is required" });
  }

  return { id: 1, name: body.name };
});
```

```ts
import usersPost from "./server/api/users.post";
import { createEvent } from "h3";

it("creates user with valid data", async () => {
  const event = createEvent({});
  event.node.req.body = { name: "Alice" };

  const result = await usersPost(event);

  expect(result).toEqual({ id: 1, name: "Alice" });
});
```

> ⚠️ Прямой вызов обработчика быстрее, но менее реалистичен: вы не проверяете маршрутизацию, middleware и парсинг тела запроса.

---

## Тестирование middleware

Middleware в Nuxt — это функции, которые выполняются перед рендерингом страницы. Они принимают `to` и `from` маршруты и могут делать `navigateTo` или `abortNavigation`.

```ts
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to) => {
  const auth = useAuth();

  if (!auth.isAuthenticated && to.path !== "/login") {
    return navigateTo("/login");
  }
});
```

### Тест

```ts
import authMiddleware from "./middleware/auth";
import { mockNuxtImport } from "@nuxt/test-utils";

mockNuxtImport("useAuth", () => {
  return () => ({
    isAuthenticated: false,
  });
});

it("redirects unauthenticated users to login", () => {
  const to = { path: "/dashboard" };
  const from = { path: "/" };

  const result = authMiddleware(to, from);

  expect(result).toEqual("/login");
});
```

> 💡 `navigateTo` в тестовом окружении возвращает строку URL или объект маршрута. Это позволяет проверить логику редиректа без реального перехода.

---

## Тестирование plugins

Plugins в Nuxt регистрируются при старте приложения и добавляют глобальные свойства, provide-значения или выполняют side-эффекты.

```ts
// plugins/toast.client.ts
export default defineNuxtPlugin(() => {
  return {
    provide: {
      toast: {
        success: (message: string) => console.log(`Success: ${message}`),
        error: (message: string) => console.error(`Error: ${message}`),
      },
    },
  };
});
```

### Тест

```ts
import toastPlugin from "./plugins/toast.client";

it("provides toast methods", () => {
  const nuxtApp = useNuxtApp();
  const pluginResult = toastPlugin(nuxtApp);

  expect(pluginResult.provide.toast.success).toBeTypeOf("function");
  expect(pluginResult.provide.toast.error).toBeTypeOf("function");
});
```

> 💡 Для сложных plugins часто проще протестировать их через интеграционный тест с `mountSuspended`, чем мокировать весь `nuxtApp`.

---

## Тестирование useAsyncData / useFetch

`useAsyncData` и `useFetch` — основные инструменты загрузки данных в Nuxt. Они зависят от Nuxt-контекста, поэтому для их тестирования используется `mountSuspended` + `registerEndpoint`.

```vue
<!-- PostsPage.vue -->
<template>
  <div>
    <div v-if="pending">Loading...</div>
    <ul v-else>
      <li v-for="post in posts" :key="post.id">{{ post.title }}</li>
    </ul>
  </div>
</template>

<script setup>
const { data: posts, pending } = await useFetch("/api/posts");
</script>
```

```ts
import { mountSuspended, registerEndpoint } from "@nuxt/test-utils";
import PostsPage from "./PostsPage.vue";

describe("PostsPage", () => {
  registerEndpoint("/api/posts", () => [
    { id: 1, title: "First post" },
    { id: 2, title: "Second post" },
  ]);

  it("renders posts after loading", async () => {
    const wrapper = await mountSuspended(PostsPage);

    expect(wrapper.text()).toContain("First post");
    expect(wrapper.text()).toContain("Second post");
    expect(wrapper.text()).not.toContain("Loading...");
  });
});
```

### Ошибка при загрузке

```ts
it("shows error message", async () => {
  registerEndpoint("/api/posts", () => {
    throw createError({ statusCode: 500, statusMessage: "Server error" });
  });

  const wrapper = await mountSuspended(PostsPage);

  expect(wrapper.text()).toContain("Error");
});
```

---

## Мокирование Nuxt API

Nuxt предоставляет множество встроенных composables, которые не существуют вне Nuxt-рантайма. В тестах их нужно мокировать.

### useRoute

```ts
import { mockNuxtImport } from "@nuxt/test-utils";

mockNuxtImport("useRoute", () => {
  return () => ({
    path: "/products",
    params: { id: "1" },
    query: { page: "1" },
  });
});

it("reads route params", () => {
  const route = useRoute();
  expect(route.params.id).toBe("1");
});
```

### useRuntimeConfig

```ts
mockNuxtImport("useRuntimeConfig", () => {
  return () => ({
    public: {
      apiUrl: "https://api.example.com",
      featureFlag: true,
    },
    private: {
      secretKey: "test-secret",
    },
  });
});
```

### navigateTo

```ts
vi.mock("#app/composables/router", () => ({
  navigateTo: vi.fn((to) => to),
}));
```

### useRouter

```ts
mockNuxtImport("useRouter", () => {
  return () => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
  });
});
```

---

## Тестирование NuxtLink

`<NuxtLink>` — компонент Nuxt для навигации. В тестах его можно заменить на обычный `<a>` через stubs, чтобы не зависеть от роутера.

```ts
it("renders link", () => {
  const wrapper = mount(Navigation, {
    global: {
      stubs: {
        NuxtLink: {
          props: ["to"],
          template: "<a :href=\"to\"><slot /></a>",
        },
      },
    },
  });

  const link = wrapper.find("a");
  expect(link.attributes("href")).toBe("/about");
});
```

Или глобально в `vitest.config.ts`:

```ts
// vitest.config.ts
export default defineVitestConfig({
  test: {
    environment: "nuxt",
    environmentOptions: {
      nuxt: {
        mocks: {
          NuxtLink: true,
        },
      },
    },
  },
});
```

---

## Лучшие практики

### 1. Используйте mountSuspended для компонентов с useFetch

```ts
// ✅ Правильно
const wrapper = await mountSuspended(PostsPage);

// ❌ Обычный mount может не дождаться данных
const wrapper = mount(PostsPage);
```

### 2. Мокируйте API через registerEndpoint

```ts
// ✅ Реалистично и стабильно
registerEndpoint("/api/users", () => [{ id: 1, name: "Alice" }]);

// ❌ Мокирование useFetch хрупко
vi.mock("useFetch", () => ({ useFetch: vi.fn() }));
```

### 3. Тестируйте server routes как функции

```ts
// ✅ Быстро и изолированно
const result = await handler(event);
expect(result).toEqual({ ... });

// ❌ Запуск реального сервера медленно
const result = await fetch("http://localhost:3000/api/users");
```

### 4. Мокируйте runtime config

```ts
mockNuxtImport("useRuntimeConfig", () => {
  return () => ({ public: { apiUrl: "http://localhost:3000" } });
});
```

### 5. Очищайте моки между тестами

```ts
afterEach(() => {
  vi.clearAllMocks();
  vi.restoreAllMocks();
});
```

---

## Антипаттерны

### 1. Тестирование фреймворка

```ts
// ❌ Тестирует Nuxt
it("auto-imports work", () => {
  expect(useState).toBeDefined();
});

// ✅ Тестирует бизнес-логику
it("increments counter", async () => {
  const wrapper = await mountSuspended(Counter);
  await wrapper.find("button").trigger("click");
  expect(wrapper.text()).toContain("1");
});
```

### 2. Избыточное мокирование

```ts
// ❌ Мокирует всё
mockNuxtImport("useRoute", ...);
mockNuxtImport("useRouter", ...);
mockNuxtImport("useRuntimeConfig", ...);
mockNuxtImport("useHead", ...);

// ✅ Мокирует только то, что нужно
mockNuxtImport("useRuntimeConfig", () => ({ public: { apiUrl: "..." } }));
```

### 3. Игнорирование асинхронности SSR

```ts
// ❌ Не дожидается загрузки данных
it("renders posts", () => {
  const wrapper = mount(PostsPage);
  expect(wrapper.text()).toContain("First post");
});

// ✅ Использует mountSuspended
it("renders posts", async () => {
  const wrapper = await mountSuspended(PostsPage);
  expect(wrapper.text()).toContain("First post");
});
```

### 4. Тестирование server routes через HTTP

```ts
// ❌ Медленно и ненадёжно
const res = await fetch("http://localhost:3000/api/users");

// ✅ Быстро и изолированно
registerEndpoint("/api/users", () => [...]);
const users = await $fetch("/api/users");
```

### 5. Хардкод тестовых данных

```ts
// ❌ Дублирование
const user = { id: 1, name: "Alice", email: "alice@example.com", role: "user" };

// ✅ Фабрика
const createUser = (overrides) => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  ...overrides,
});
```
