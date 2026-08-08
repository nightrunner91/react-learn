# Sentry: Полное руководство по интеграции и мониторингу

> Как правильно установить Sentry, не наступить на грабли и научиться анализировать ошибки так, чтобы это реально помогало, а не просто занимало место в дашборде.

---

## Содержание

1. [Что такое Sentry и зачем он нужен](#что-такое-sentry-и-зачем-он-нужен)
2. [React (SPA)](#react-spa)
3. [Next.js](#nextjs)
4. [Vue 3](#vue-3)
5. [Nuxt 3](#nuxt-3)
6. [Source Maps — обязательно](#source-maps--обязательно)
7. [Правильное логирование ошибок](#правильное-логирование-ошибок)
8. [Performance Monitoring](#performance-monitoring)
9. [Custom Context и Breadcrumbs](#custom-context-и-breadcrumbs)
10. [Release Health](#release-health)
11. [User Feedback](#user-feedback)
12. [Alerts и интеграции](#alerts-и-интеграции)
13. [Подводные камни и неочевидности](#подводные-камни-и-неочевидности)
14. [Анализ ошибок в Sentry](#анализ-ошибок-в-sentry)
15. [Лучшие практики](#лучшие-практики)
16. [Антипаттерны](#антипаттерны)
17. [Чек-лист интеграции](#чек-лист-интеграции)

---

## Что такое Sentry и зачем он нужен

Sentry — платформа для мониторинга ошибок и производительности в реальном времени. Она решает четыре задачи:

- **Error Monitoring** — перехват неперехваченных исключений, необработанных промисов и ошибок рендеринга.
- **Performance Monitoring (Tracing)** — измерение скорости загрузки страниц, API-запросов и пользовательских операций.
- **Session Replay** — видео-запись того, что видел пользователь до, во время и после ошибки.
- **Logs** — структурированные логи, коррелированные с ошибками и трейсами.

Главная идея: вы не ждёте, пока пользователь напишет в поддержку. Вы видите ошибку, стектрейс, контекст и запись экрана — ещё до того, как пользователь сам понял, что что-то сломалось.

Sentry можно развернуть:
- **Sentry SaaS** — облачная версия (sentry.io)
- **Self-hosted** — на собственной инфраструктуре (Docker)

### Проблема: «у меня работает»

Разработчик тестирует на своём компьютере, в своём браузере, с хорошими данными. Пользователи сталкиваются с:

- Ошибками на конкретных устройствах/браузерах
- Проблемами производительности на слабых сетях
- Ошибками в edge-case сценариях
- Регрессиями после деплоя

Без мониторинга вы узнаёте о проблемах из жалоб пользователей — часто слишком поздно.

---

## React (SPA)

React — single-page application: весь код загружается один раз, и дальше всё работает в браузере. Это значит, что ошибки могут возникнуть в любой момент — при рендере, в обработчике событий, в эффекте. Sentry перехватывает их глобально и отправляет на сервер.

### Установка

SDK `@sentry/react` — обёртка над `@sentry/browser` с React-специфичными интеграциями: ErrorBoundary, хуки ошибок React 19, интеграция с React Router.

```bash
npm install @sentry/react --save
```

### Инициализация

Конфигурация Sentry должна быть выполнена **до** любого другого кода. Если Sentry инициализируется после импорта React или компонентов — ошибки из этих модулей не будут перехвачены. Поэтому `instrument.js` импортируется первой строкой в entry point.

Параметры конфигурации:
- **`tracesSampleRate`** — доля транзакций для performance monitoring. `1.0` = 100%, в production ставьте `0.1` или ниже, иначе квота закончится за часы.
- **`replaysSessionSampleRate`** — доля записываемых сессий. `0.1` означает, что пишется каждая десятая.
- **`replaysOnErrorSampleRate`** — доля сессий с ошибками, которые записываются. `1.0` гарантирует, что при ошибке replay будет всегда.
- **`tracePropagationTargets`** — URL-ы, куда передавать `sentry-trace` заголовок для distributed tracing. Без этого бэкенд не увидит связь между фронтенд- и бэкенд-спанами.

```javascript
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: "https://<key>@o<orgId>.ingest.sentry.io/<projectId>",

  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration(),
    Sentry.feedbackIntegration({ colorScheme: "system" }),
  ],

  enableLogs: true,

  tracesSampleRate: 1.0,
  tracePropagationTargets: [/^\//, /^https:\/\/yourserver\.io\/api/],

  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,

  environment: process.env.NODE_ENV,
  release: `my-app@${process.env.APP_VERSION}`,
});
```

Подключите в точке входа:

```javascript
// instrument.js — ПЕРВЫЙ импорт!
import "./instrument";
import App from "./App";
import { createRoot } from "react-dom/client";

const root = createRoot(document.getElementById("app"));
root.render(<App />);
```

### Обработка ошибок рендеринга

Ошибки рендеринга — самый коварный тип ошибок в React. Если компонент бросает ошибку во время рендера, React может размонтировать всё дерево. Без ErrorBoundary пользователь увидит белый экран.

**React 19+** — появились встроенные хуки `onUncaughtError`, `onCaughtError`, `onRecoverableError`. Они заменяют ErrorBoundary для большинства случаев и дают доступ к `componentStack`. `Sentry.reactErrorHandler()` — обёртка, которая автоматически отправляет ошибку в Sentry и вызывает React-обработчик.

```javascript
import * as Sentry from "@sentry/react";

const root = createRoot(document.getElementById("app"), {
  onUncaughtError: Sentry.reactErrorHandler((error, errorInfo) => {
    console.warn("Uncaught error", error, errorInfo.componentStack);
  }),
  onCaughtError: Sentry.reactErrorHandler(),
  onRecoverableError: Sentry.reactErrorHandler(),
});
```

**React 18 и ниже** — хуков `onUncaughtError` нет, поэтому нужен ErrorBoundary. `Sentry.ErrorBoundary` — готовый компонент, который перехватывает ошибки рендера дочерних компонентов, отправляет их в Sentry и показывает fallback-UI. Важно оборачивать не всё приложение целиком, а логические части — так ошибка в одном виджете не убьёт всю страницу.

```javascript
import * as Sentry from "@sentry/react";

function App() {
  return (
    <Sentry.ErrorBoundary fallback={<p>Произошла ошибка</p>}>
      <YourAppContent />
    </Sentry.ErrorBoundary>
  );
}
```

### Кастомный Error Boundary

`Sentry.ErrorBoundary` покрывает 90% случаев, но иногда нужен кастомный класс-компонент. Причины: логирование в собственную систему, отправка аналитики, разный fallback для разных типов ошибок. Класс-компонент обязателен — `componentDidCatch` работает только в классах.

```jsx
import * as Sentry from "@sentry/react";
import { Component } from "react";

class ErrorBoundary extends Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    Sentry.captureException(error, {
      extra: errorInfo,
    });
  }

  render() {
    if (this.state.hasError) {
      return <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}
```

Или через `Sentry.ErrorBoundary` с кастомным fallback:

```jsx
<Sentry.ErrorBoundary
  fallback={({ error, componentStack, resetError }) => (
    <div>
      <h2>Error: {error?.message}</h2>
      <button onClick={resetError}>Try again</button>
    </div>
  )}
  onError={(error, componentStack) => {
    console.error("Caught error:", error, componentStack);
  }}
>
  <App />
</Sentry.ErrorBoundary>
```

### Перехват в обработчиках событий

Ошибки в `onClick`, `onSubmit` и других обработчиках React ловит через `error bubbling` — они всплывают до ErrorBoundary. Но если вы обрабатываете ошибку в `try/catch` внутри обработчика, она не дойдёт до Boundary. В этом случае вызывайте `captureException` вручную. Добавляйте `tags` и `extra` — это поможет фильтровать ошибки в дашборде и понимать контекст без воспроизведения.

```jsx
function DeleteButton({ postId }) {
  const handleClick = async () => {
    try {
      await api.deletePost(postId);
    } catch (error) {
      Sentry.captureException(error, {
        tags: { action: "delete_post" },
        extra: { postId },
      });
      toast.error("Failed to delete post");
    }
  };

  return <button onClick={handleClick}>Delete</button>;
}
```

### Перехват в хуках

Кастомные хуки — хорошее место для централизованного логирования. Вместо того чтобы добавлять `captureException` в каждый `useEffect` и каждый `fetch`, создайте обёртку. `Sentry.startTransaction` создаёт спан для performance monitoring — вы увидите в дашборде, сколько времени занимает каждая операция. Не забудьте `transaction.finish()` в `finally`, иначе спан никогда не закроется и будет висеть в трассировке.

```jsx
function useSafeFetch() {
  const fetchWithSentry = async (url) => {
    const transaction = Sentry.startTransaction({ name: `fetch ${url}` });

    try {
      const res = await fetch(url);
      if (!res.ok) {
        throw new Error(`HTTP ${res.status}`);
      }
      return await res.json();
    } catch (error) {
      Sentry.captureException(error, {
        tags: { url },
      });
      throw error;
    } finally {
      transaction.finish();
    }
  };

  return fetchWithSentry;
}
```

### React Router

Без интеграции с роутером Sentry видит все навигации как одну страницу — `/`. Это бесполезно: вы не поймёте, на каком маршруте произошла ошибка. `reactRouterV7BrowserTracingIntegration` автоматически отслеживает переходы между страницами и создаёт отдельные транзакции для каждого маршрута. Передавайте хуки роутера — Sentry использует их, чтобы подписаться на изменения `location`.

```javascript
import * as Sentry from "@sentry/react";
import { useLocation, useNavigationType, createRoutesFromChildren, matchRoutes } from "react-router";

Sentry.init({
  dsn: "https://<key>@o<orgId>.ingest.sentry.io/<projectId>",
  integrations: [
    Sentry.reactRouterV7BrowserTracingIntegration({
      useEffect: React.useEffect,
      useLocation,
      useNavigationType,
      createRoutesFromChildren,
      matchRoutes,
    }),
  ],
});
```

### Redux

Состояние Redux — ценный контекст при отладке. Когда ошибка произошла, вы хотите знать, какой был `state` — какие данные загружены, какой пользователь авторизован, что в корзине. `createReduxEnhancer` автоматически прикрепляет состояние store к каждому событию в Sentry. По умолчанию отправляется весь state — если там есть чувствительные данные, настройте `stateTransformer` для фильтрации.

```javascript
import { createStore, compose } from "redux";
import * as Sentry from "@sentry/react";

const sentryEnhancer = Sentry.createReduxEnhancer();
const store = createStore(rootReducer, compose(sentryEnhancer));
```

---

## Next.js

### Установка через wizard

```bash
npx @sentry/wizard@latest -i nextjs
```

Wizard создаст все необходимые файлы. Если предпочитаете ручную настройку — см. [документацию](https://docs.sentry.io/platforms/javascript/guides/nextjs/manual-setup.md).

### Архитектура: три среды выполнения

Next.js работает в трёх разных средах, и для каждой нужен свой конфиг. Это главный концептуальный момент, который нельзя игнорировать.

**Клиент** — `instrumentation-client.ts`:

```typescript
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: "https://<key>@o<orgId>.ingest.sentry.io/<projectId>",
  tracesSampleRate: process.env.NODE_ENV === "development" ? 1.0 : 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  enableLogs: true,
  integrations: [Sentry.replayIntegration()],
});
```

**Сервер (Node.js)** — `sentry.server.config.ts`:

```typescript
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: "https://<key>@o<orgId>.ingest.sentry.io/<projectId>",
  tracesSampleRate: process.env.NODE_ENV === "development" ? 1.0 : 0.1,
  enableLogs: true,
});
```

**Edge Runtime** — `sentry.edge.config.ts`:

```typescript
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: "https://<key>@o<orgId>.ingest.sentry.io/<projectId>",
  tracesSampleRate: process.env.NODE_ENV === "development" ? 1.0 : 0.1,
  enableLogs: true,
});
```

**Регистрация серверных конфигов** — `instrumentation.ts`:

```typescript
import * as Sentry from "@sentry/nextjs";

export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    await import("./sentry.server.config");
  }
  if (process.env.NEXT_RUNTIME === "edge") {
    await import("./sentry.edge.config");
  }
}

export const onRequestError = Sentry.captureRequestError;
```

**Обёртка next.config**:

```typescript
import { withSentryConfig } from "@sentry/nextjs";

export default withSentryConfig(nextConfig, {
  org: "<your-org-slug>",
  project: "<your-project-slug>",
  authToken: process.env.SENTRY_AUTH_TOKEN,
  tunnelRoute: "/sentry-tunnel",
  silent: !process.env.CI,
  widenClientFileUpload: true,
  hideSourceMaps: true,
});
```

### Перехват ошибок в Next.js

Next.js App Router разделяет код на Server Components и Client Components. Ошибки в Server Components происходят на сервере — пользователь их не видит, но страница может вернуть 500 или пустой HTML. В Client Components — аналогично React SPA, ошибка ломает рендер. Разные среды требуют разных подходов к обработке.

**Server Components:**

Server Components выполняются на сервере при каждом запросе. Если `fetchUserData()` бросит ошибку, Next.js покажет `error.tsx` или вернёт 500. `Sentry.captureException` здесь нужен, чтобы увидеть ошибку в дашборде — без него вы узнаете о проблеме только из логов сервера.

```tsx
// app/page.tsx
import * as Sentry from "@sentry/nextjs";

export default async function Page() {
  try {
    const data = await fetchUserData();
    return <UserProfile data={data} />;
  } catch (error) {
    Sentry.captureException(error);
    return <div>Failed to load user data</div>;
  }
}
```

**API Routes / Route Handlers:**

Route Handlers — это серверный код, аналог backend API. Ошибки здесь возвращают HTTP-ответы клиенту, но без `captureException` вы не увидите стектрейс в Sentry. Важно: не возвращайте детали ошибки клиенту — только общее сообщение. Детали уходят в Sentry через `extra`.

```tsx
// app/api/users/route.ts
import * as Sentry from "@sentry/nextjs";

export async function GET() {
  try {
    const users = await db.users.findMany();
    return Response.json(users);
  } catch (error) {
    Sentry.captureException(error);
    return Response.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

**Server Actions:**

Server Actions — функции, которые выполняются на сервере, но вызываются из клиентских компонентов. Ошибки в них всплывают до клиента через RPC, но без `captureException` на сервере вы потеряете контекст. Если action делает `throw`, клиент получит общую ошибку — детали смотрите в Sentry.

```tsx
"use server";

import * as Sentry from "@sentry/nextjs";

export async function createPost(formData: FormData) {
  try {
    const title = formData.get("title") as string;
    await db.posts.create({ title });
  } catch (error) {
    Sentry.captureException(error, {
      tags: { action: "create_post" },
    });
    throw error;
  }
}
```

**Error.tsx (App Router):**

`error.tsx` — специальный файл Next.js, который автоматически рендерится при ошибке в дочерних компонентах маршрута. Это аналог ErrorBoundary, но встроенный в роутинг. `"use client"` обязателен — компонент использует хуки. `Sentry.captureException` в `useEffect` отправляет ошибку при первом рендере error-компонента.

```tsx
// app/error.tsx
"use client";

import * as Sentry from "@sentry/nextjs";
import { useEffect } from "react";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

**Глобальный обработчик ошибок рендеринга** — `app/global-error.tsx`:

`global-error.tsx` перехватывает ошибки, которые не поймал `error.tsx` — например, ошибки в корневом `layout.tsx`. Это последний рубеж: без него ошибка в layout приведёт к белому экрану. В отличие от `error.tsx`, здесь нужно вернуть полный HTML (`<html>`, `<body>`), потому что layout тоже сломан.

```tsx
"use client";

import * as Sentry from "@sentry/nextjs";
import { useEffect } from "react";

export default function GlobalError({
  error,
}: {
  error: Error & { digest?: string };
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <html>
      <body>
        <h1>Something went wrong!</h1>
      </body>
    </html>
  );
}
```

---

## Vue 3

Vue и React отличаются в обработке ошибок. Vue имеет встроенный `app.config.errorHandler`, который перехватывает ошибки из компонентов, хуков жизненного цикла и директив. Sentry использует этот механизм — поэтому `app` обязателен в `Sentry.init`. Без него Sentry не знает, к какому приложению привязать обработчик.

### Установка

```bash
npm install @sentry/vue --save
```

### Инициализация в `main.js`

Ключевое отличие от React: в Vue нужно передать инстанс приложения в `Sentry.init`. Без этого Vue-интеграция не будет работать.

```javascript
import { createApp } from "vue";
import { createRouter } from "vue-router";
import * as Sentry from "@sentry/vue";

const app = createApp({
  // ...
});

const router = createRouter({
  // ...
});

Sentry.init({
  app, // Обязательно!
  dsn: "https://<key>@o<orgId>.ingest.sentry.io/<projectId>",

  integrations: [
    Sentry.browserTracingIntegration({ router }),
    Sentry.replayIntegration(),
    Sentry.feedbackIntegration({ colorScheme: "system" }),
  ],

  enableLogs: true,

  tracesSampleRate: 1.0,
  tracePropagationTargets: ["localhost", /^https:\/\/yourserver\.io\/api/],

  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});

app.use(router);
app.mount("#app");
```

### Интеграция с Pinia

Pinia — аналог Redux для Vue. `createSentryPiniaPlugin` автоматически добавляет состояние store к ошибкам в Sentry. Это полезно для отладки: вы видите, какие данные были в store в момент ошибки. Плагин подписывается на все actions и мутации, поэтому не нужно вручную добавлять `captureException` в каждый store.

```javascript
import { createPinia } from "pinia";
import { createSentryPiniaPlugin } from "@sentry/vue";

const pinia = createPinia();
pinia.use(createSentryPiniaPlugin());
```

### Поздняя инициализация

Обычный сценарий: микрофронтенды, shell-приложения, или когда Vue-инстанс создаётся динамически. Проблема в том, что Sentry нужно инициализировать до появления `app`, но Vue-интеграция требует `app`. Решение — инициализировать Sentry без Vue-интеграции, а потом добавить её через `addIntegration`. Фильтр `integrations.filter` убирает автоматическую Vue-интеграцию, чтобы не было конфликта.

```javascript
import * as Sentry from "@sentry/vue";

Sentry.init({
  dsn: "...",
  integrations: (integrations) =>
    integrations.filter((integration) => integration.name !== "Vue"),
});

// Позже, когда app создан:
const app = createApp({ template: "<div>hello</div>" });
Sentry.addIntegration(Sentry.vueIntegration({ app }));
```

---

## Nuxt 3

Nuxt 3 — метафреймворк на базе Vue 3, как Next.js для React. Главное отличие от чистого Vue: код выполняется и на сервере (SSR), и на клиенте. Клиентский Sentry подключается автоматически через Nuxt-модуль, но серверный нужно запускать явно — через `--import` при старте Node.js.

### Установка

```bash
npx sentry@latest init
```

Или через классический wizard:

```bash
npx @sentry/wizard@latest -i nuxt
```

Требования: Nuxt >= 3.7.0 (рекомендуется 3.14.0+). Версии ниже 3.14.0 имеют проблемы с совместимостью `ofetch` и `@vercel/nft` — Sentry может не загрузиться на сервере.

Для Nuxt < 3.14.0 добавьте overrides в `package.json`:

```json
{
  "overrides": {
    "ofetch": "^1.4.0",
    "@vercel/nft": "^0.27.4"
  }
}
```

### Запуск — критически важно

Nuxt, как и Next.js, работает и на клиенте, и на сервере. Серверный Sentry нужно подключать **явно** через флаг `--import`. Без этого флага Node.js не загрузит Sentry-конфиг, и ошибки SSR не будут отправлены. Это самая частая ошибка при деплое Nuxt с Sentry — всё работает локально, но в production ошибки с сервера молча теряются.

**Production:**

Production-сборка генерирует серверный конфиг в `.output/server/sentry.server.config.mjs`. Флаг `--import` загружает его до старта приложения.

```bash
nuxi build
node --import ./.output/server/sentry.server.config.mjs .output/server/index.mjs
```

**Development:**

В dev-режиме конфиг генерируется в `.nuxt/dev/`. Первый запуск `nuxt dev` без флага создаёт этот конфиг. Повторные запуски уже могут использовать `--import`. Если удалите `.nuxt` — конфиг исчезнет, и нужно снова запустить `nuxt dev` без флага для регенерации.

```bash
# Первый запуск — без флага (сгенерирует конфиг в .nuxt)
nuxt dev

# Повторные запуски:
NODE_OPTIONS='--import ./.nuxt/dev/sentry.server.config.mjs' nuxt dev
```

Если удалите папку `.nuxt` — нужно снова запустить `nuxt dev` без флага, чтобы перегенерировать серверный конфиг.

---

## Source Maps — обязательно

Source maps связывают минифицированный production-код с исходниками. Без них стектрейс в Sentry указывает на строку 1 колонку 12345 — бесполезно. С source maps вы видите `src/components/UserProfile.tsx:42`.

**Важно:** source maps не должны быть доступны пользователям — они раскрывают исходный код. Загружайте их напрямую в Sentry при сборке, а не публикуйте на CDN. `hideSourceMaps: true` в Next.js-конфиге удаляет их из публичной сборки.

```bash
npx @sentry/wizard@latest -i sourcemaps
```

Для CI/CD установите переменную окружения:

```bash
SENTRY_AUTH_TOKEN=sntrys_eyJ...
```

Файл `.env.sentry-build-plugin` создаётся автоматически и добавляется в `.gitignore`. Проверьте, что он не попадает в репозиторий.

### Загрузка source maps

Загрузка source maps — часть CI/CD-пайплайна. `sentry-cli` — универсальный инструмент, работает с любым бандлером. Webpack-плагин удобнее, если проект на webpack — он загружает maps автоматически после сборки.

```bash
# Через sentry-cli
sentry-cli releases files my-app@1.0.0 upload-sourcemaps ./dist --url-prefix "~/static"

# Или через webpack plugin
const { SentryWebpackPlugin } = require("@sentry/webpack-plugin");

module.exports = {
  plugins: [
    new SentryWebpackPlugin({
      org: "my-org",
      project: "my-app",
      include: "./dist",
    }),
  ],
};
```

### Next.js (автоматически)

`@sentry/nextjs` автоматически загружает source maps при сборке — не нужно добавлять `sentry-cli` в CI. `widenClientFileUpload: true` разрешает загрузку файлов больше стандартного лимита (актуально для больших бандлов). `hideSourceMaps: true` удаляет `.map` файлы из публичной директории `.next/static` — они загружаются напрямую в Sentry, а не доступны через браузер.

```js
// next.config.js
module.exports = withSentryConfig(
  {
    // ваша конфигурация
  },
  {
    widenClientFileUpload: true,
    hideSourceMaps: true,
  }
);
```

---

## Правильное логирование ошибок

Sentry предоставляет три уровня логирования: `captureException` для исключений, `captureMessage` для событий без исключения, и `Sentry.logger` для структурированных логов. Выбор зависит от ситуации:

- **captureException** — когда есть объект `Error` с стектрейсом. Показывает полный стек, контекст, breadcrumbs.
- **captureMessage** — когда события важны, но не являются ошибками (пользователь отменил заказ, достиг лимита). Можно задать `level`: `info`, `warning`, `error`, `fatal`.
- **Sentry.logger** — для бизнес-логов, которые коррелируются с трейсами. Полезно для отладки производительности.

### captureException — перехваченные ошибки

`captureException` принимает объект ошибки и опциональный контекст. `tags` — для фильтрации в дашборде (например, все ошибки checkout). `extra` — произвольные данные для отладки (orderId, retryCount). Не кладите в `extra` чувствительные данные — пароли, токены, PII.

```javascript
try {
  await fetchData();
} catch (error) {
  Sentry.captureException(error, {
    tags: { feature: "checkout" },
    extra: { orderId: "12345", retryCount: 3 },
  });
}
```

### captureMessage — логирование без исключения

Используйте `captureMessage`, когда нужно зафиксировать событие, но исключения нет. Например, пользователь достиг лимита, отменил заказ, или бизнес-правило сработало. `level: "warning"` покажет событие как предупреждение, а не ошибку — не будет триггерить алерты по критичности.

```javascript
Sentry.captureMessage("Пользователь отменил заказ", {
  level: "warning",
  extra: { orderId: "12345" },
});
```

### Структурированные логи

`Sentry.logger` — это не `console.log`. Это структурированные логи, которые Sentry коррелирует с ошибками и трейсами. Когда вы видите ошибку в Sentry, рядом будут логи с тем же `trace_id` — вы увидите последовательность событий, приведших к ошибке. Требует `enableLogs: true` в `Sentry.init`.

```javascript
Sentry.logger.info("Пользователь вошёл", { userId: "123" });
Sentry.logger.warn("Медленный ответ API", { duration: 5000, endpoint: "/api/orders" });
Sentry.logger.error("Операция не удалась", { reason: "timeout" });
```

### Обогащение контекста

Контекст делает ошибки полезными. Без `setUser` вы видите анонимную ошибку — не знаете, кого затронуло. Без `setTag` не можете фильтровать по фичам, окружениям, версиям. Без `setContext` не видите состояние приложения в момент ошибки. Вызывайте эти методы при изменении состояния — после логина, смены страницы, обновления данных.

```javascript
// Пользователь
Sentry.setUser({ id: "42", email: "user@example.com" });

// Теги — для фильтрации в дашборде
Sentry.setTag("environment", "staging");
Sentry.setTag("app_version", "2.1.0");

// Произвольный контекст
Sentry.setContext("cart", { items: 3, total: 1500 });
```

### Пользовательские спаны для трейсинга

Автоматический трейсинг измеряет HTTP-запросы и загрузку страниц. Но бизнес-операции (checkout, поиск, загрузка данных) не видны без кастомных спанов. `Sentry.startSpan` создаёт спан внутри текущей транзакции — вы увидите его в Performance-дашборде. `op` — тип операции (db.query, http.client), `name` — описание для UI.

```javascript
const result = await Sentry.startSpan(
  { op: "db.query", name: "Получить заказы пользователя" },
  async () => {
    return await db.query("SELECT * FROM orders WHERE user_id = ?", [userId]);
  }
);
```

---

## Performance Monitoring

Performance monitoring показывает, **как быстро** работает приложение, а не **что** сломалось. Ошибки — это бинарно: работает/не работает. Производительность — спектр: страница грузится 200ms или 5 секунд. Sentry измеряет Web Vitals (LCP, FID, CLS), HTTP-запросы, рендеринг компонентов. Это помогает находить узкие места: медленный API, тяжёлый компонент, проблемы с сетью.

### Автоматическое отслеживание

Sentry автоматически отслеживает:

- **Page loads** — время загрузки страниц (navigation timing)
- **API calls** — длительность HTTP-запросов (fetch, XHR)
- **Component renders** — время рендеринга компонентов (React, Vue)
- **Web Vitals** — LCP ( Largest Contentful Paint), FID (First Input Delay), CLS (Cumulative Layout Shift), FCP (First Contentful Paint), TTFB (Time to First Byte)

`tracesSampleRate: 1.0` отправляет 100% транзакций — подходит для разработки, но в production быстро сожрёт квоту. Ставьте `0.1` или ниже.

```jsx
Sentry.init({
  tracesSampleRate: 1.0,
});
```

### Custom transactions

Транзакция — это группа связанных спанов. Автоматический трейсинг создаёт транзакции для навигации и HTTP-запросов. Но бизнес-операции (checkout, оплата, поиск) требуют кастомных транзакций. `startTransaction` создаёт новую транзакцию, `startChild` добавляет спан внутрь. Не забудьте `finish()` — иначе транзакция не закроется и не отправится.

```jsx
function CheckoutForm() {
  const handleSubmit = async () => {
    const transaction = Sentry.startTransaction({ name: "checkout" });
    const span = transaction.startChild({ op: "payment", description: "Process payment" });

    try {
      await processPayment();
      span.setStatus("ok");
    } catch (error) {
      span.setStatus("internal_error");
      Sentry.captureException(error);
    } finally {
      span.finish();
      transaction.finish();
    }
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

### Performance в Next.js

Middleware в Next.js выполняется для каждого запроса — идеальное место для измерения времени обработки. `wrapMiddleware` автоматически создаёт транзакцию для каждого запроса к middleware. Это полезно для отслеживания задержек аутентификации, редиректов, локализации.

```tsx
// Middleware для отслеживания всех запросов
import * as Sentry from "@sentry/nextjs";

export default Sentry.wrapMiddleware(async (request) => {
  const response = await NextResponse.next();
  return response;
});
```

---

## Custom Context и Breadcrumbs

Контекст и breadcrumbs превращают абстрактную ошибку в воспроизводимую историю. **Контекст** — это снимок состояния приложения в момент ошибки: кто пользователь, какой язык, что в корзине. **Breadcrumbs** — это хронология событий до ошибки: какие запросы отправлялись, что кликал пользователь, какие логи выводились. Вместе они позволяют воспроизвести баг без связи с пользователем.

### User context

`setUser` прикрепляет информацию о пользователе ко всем последующим событиям. Вызывайте при логине, обновлении профиля, выходе. Если пользователь не авторизован — передайте `null`, чтобы очистить контекст. Не передавайте пароли, токены, PII — только идентификаторы и email.

```jsx
import * as Sentry from "@sentry/react";

function useSentryUser(user) {
  useEffect(() => {
    if (user) {
      Sentry.setUser({
        id: user.id,
        email: user.email,
        username: user.name,
      });
    } else {
      Sentry.setUser(null);
    }
  }, [user]);
}

function App() {
  const { user } = useAuth();
  useSentryUser(user);
  return <Router />;
}
```

### Tags

Теги — это ключ-значение пары для фильтрации и агрегации в дашборде. В отличие от `extra`, теги индексируются — вы можете искать "все ошибки с `feature: checkout`" или "все ошибки в `environment: staging`". Используйте теги для бизнес-метрик: фича, шаг процесса, тип подписки. Не используйте для уникальных значений (orderId, userId) — это раздует индекс.

```jsx
Sentry.setTag("locale", "ru-RU");
Sentry.setTag("subscription", "premium");

// Или при captureException
Sentry.captureException(error, {
  tags: {
    feature: "checkout",
    step: "payment",
  },
});
```

### Extra context

`extra` — произвольные данные для отладки, которые не индексируются. Используйте для деталей, которые нужны только при просмотре конкретной ошибки: orderId, amount, payload. В отличие от тегов, `extra` не подходит для фильтрации — но даёт полный контекст при разборе.

```jsx
Sentry.setExtra("cart_items", cart.length);
Sentry.setExtra("last_api_call", lastApiUrl);

// Или при captureException
Sentry.captureException(error, {
  extra: {
    orderId: order.id,
    amount: order.total,
    paymentMethod: order.paymentMethod,
  },
});
```

### Breadcrumbs

Breadcrumbs — хронология событий до ошибки. Sentry автоматически записывает:

- HTTP-запросы (URL, метод, статус)
- Console.log / console.warn / console.error
- DOM-события (клики, фокус)
- Navigation (переходы между страницами)

Кастомные breadcrumbs добавляйте для бизнес-событий: вход в систему, добавление в корзину, начало checkout. Они помогают понять, **что делал пользователь** до ошибки — без Session Replay.

```jsx
Sentry.addBreadcrumb({
  category: "auth",
  message: "User logged in",
  level: "info",
  data: { userId: user.id },
});

Sentry.addBreadcrumb({
  category: "checkout",
  message: "Added item to cart",
  level: "info",
  data: { productId: item.id, quantity: 2 },
});
```

---

## Release Health

Release tracking связывает ошибки с версиями приложения. Без него вы видите ошибку, но не знаете, в какой версии она появилась и когда была исправлена. С release tracking Sentry показывает: "эта ошибка появилась в 1.2.0, исправлена в 1.3.0, затрагивает 5% пользователей". Это критично для приоритизации: критический баг в старой версии менее важен, чем регрессия в новой.

### Tracking releases

Релиз создаётся при деплое. `sentry-cli releases new` регистрирует версию, `upload-sourcemaps` загружает source maps для этой версии, `finalize` закрывает релиз. Альтернатива — указать `release` в `Sentry.init`, и SDK создаст релиз автоматически при первом событии.

```bash
# Создать релиз
sentry-cli releases new my-app@1.0.0

# Загрузить source maps
sentry-cli releases files my-app@1.0.0 upload-sourcemaps ./dist

# Завершить релиз
sentry-cli releases finalize my-app@1.0.0

# Или через Sentry SDK
Sentry.init({
  release: "my-app@1.0.0",
});
```

### Автоматизация в CI/CD

Ручное создание релизов работает для маленьких команд, но быстро становится рутиной. GitHub Action `getsentry/action-release` автоматизирует процесс: при пуше тега создаёт релиз, загружает source maps, привязывает коммиты. Это даёт полную трассировку: от коммита до ошибки в production.

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags:
      - "v*"

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run build
      - name: Create Sentry release
        uses: getsentry/action-release@v1
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: my-org
          SENTRY_PROJECT: my-app
        with:
          environment: production
          sourcemaps: "./dist"
```

### Health metrics

Health metrics — агрегированные показатели стабильности приложения. Sentry считает их автоматически на основе сессий и событий:

- **Crash-free sessions** — процент сессий без крашей. Цель: > 99.5%. Ниже 99% — серьёзная проблема.
- **Crash-free users** — процент пользователей без крашей. Показывает, насколько ошибка распространена.
- **Issues per release** — сколько уникальных ошибок в каждом релизе. Рост = регрессия.
- **Regression detection** — ошибки, которые были закрыты, но вернулись. Sentry автоматически помечает их как регрессии.

---

## User Feedback

User Feedback — мост между пользователем и разработчиком. Когда пользователь видит ошибку, он может заполнить форму с описанием. Sentry привязывает этот feedback к конкретному событию — вы видите, **что сломалось** и **что думает пользователь**. Это эффективнее, чем скриншоты в поддержке.

### Форма обратной связи

`showReportDialog` — встроенная форма Sentry. Показывает модальное окно с полями для имени, email, описания проблемы. Автоматически привязывается к последнему событию — если ошибка только что произошла, feedback будет привязан к ней.

```jsx
import * as Sentry from "@sentry/react";

function FeedbackButton() {
  const handleClick = () => {
    Sentry.showReportDialog({
      title: "Что-то пошло не так",
      subtitle: "Наши инженеры уже уведомлены",
      labelName: "Имя",
      labelEmail: "Email",
      labelComments: "Опишите проблему",
      labelClose: "Закрыть",
      labelSubmit: "Отправить",
    });
  };

  return <button onClick={handleClick}>Report a bug</button>;
}
```

### Custom feedback widget

Встроенная форма Sentry не всегда подходит — может не соответствовать дизайну, или нужен feedback без привязки к ошибке. Custom widget даёт полный контроль: свой UI, своя логика отправки. `captureMessage` с `level: "info"` отправляет feedback как событие — его видно в Sentry, но не триггерит алерты по ошибкам.

```jsx
function FeedbackWidget() {
  const [isOpen, setIsOpen] = useState(false);
  const [message, setMessage] = useState("");

  const handleSubmit = () => {
    Sentry.captureMessage(message, {
      level: "info",
      tags: { type: "user_feedback" },
    });
    setIsOpen(false);
    setMessage("");
    toast.success("Feedback sent");
  };

  if (!isOpen) {
    return <button onClick={() => setIsOpen(true)}>Feedback</button>;
  }

  return (
    <div className="feedback-widget">
      <textarea
        value={message}
        onChange={(e) => setMessage(e.target.value)}
        placeholder="Describe the issue..."
      />
      <button onClick={handleSubmit}>Send</button>
      <button onClick={() => setIsOpen(false)}>Cancel</button>
    </div>
  );
}
```

---

## Alerts и интеграции

### Alert rules

В Sentry можно настроить алерты:

- **По частоте** — если ошибка произошла > 10 раз за 5 минут
- **По пользователям** — если затронуто > 100 пользователей
- **По критичности** — критические ошибки (fatal, error)
- **По тегам** — определённые фичи или окружения

## Alerts и интеграции

Алерты превращают пассивный мониторинг в активный. Без алертов вы видите ошибки только когда заходите в дашборд. С алертами Sentry сам уведомляет, когда что-то критичное произошло. Но важно настроить правильно — слишком много алертов = alert fatigue, команда перестаёт реагировать.

### Alert rules

В Sentry можно настроить алерты по различным условиям:

- **По частоте** — если ошибка произошла > 10 раз за 5 минут. Полезно для массовых сбоев.
- **По пользователям** — если затронуто > 100 пользователей. Показывает масштаб проблемы.
- **По критичности** — критические ошибки (fatal, error). Для urgent-проблем.
- **По тегам** — определённые фичи или окружения. Например, алерт только для production, но не staging.

### Интеграции

Sentry интегрируется с основными инструментами разработки:

- **Slack / Discord** — уведомления в каналы. Настройте разные каналы для разных команд.
- **GitHub / GitLab** — привязка к коммитам и PR. Sentry автоматически закрывает issues, когда ошибка исправлена в новом релизе.
- **Jira / Linear** — создание тикетов при критических ошибках. Автоматизация экономит время.
- **PagerDuty** — эскалация критических ошибок. Для 24/7 команд.
- **Webhooks** — кастомные интеграции. Можно отправить в любую систему.

---

## Подводные камни и неочевидности

Эти пункты — результат реального опыта. Каждый описывает проблему, на которую можно потратить часы (или дни), если не знать заранее.

### 1. Семплирование и квота

`tracesSampleRate: 1.0` отправляет **100%** транзакций. На реальном трафике это сожрёт всю квоту за день. В production ставьте `0.1` или ниже. Мониторьте usage stats и корректируйте.

Оптимальная стратегия для Replay:
- `replaysSessionSampleRate: 0.1` — 10% всех сессий
- `replaysOnErrorSampleRate: 1.0` — 100% сессий с ошибками

### 2. AdBlock блокирует Sentry

AdBlock режет запросы к `ingest.sentry.io`. Решения:

- **React / Vue:** `tunnel: "/tunnel"` — проксирование через свой сервер (нужна серверная часть для перенаправления)
- **Next.js:** `tunnelRoute: "/sentry-tunnel"` — встроенный прокси в `withSentryConfig`
- **Nuxt:** аналогично через `tunnel`

### 3. Ошибки из DevTools не ловятся

Ошибки, вызванные из консоли браузера (`throw new Error(...)` в DevTools), изолированы (sandboxed) и **не отправляются в Sentry**. Тестируйте через UI-элементы — кнопки, обработчики событий.

### 4. `instrument.js` должен быть первым импортом

Если Sentry инициализируется после другого кода, ошибки из этого кода не будут перехвачены. Это особенно важно в React: `import "./instrument"` должен быть первой строкой в entry point.

### 5. Дублирование ошибок

Не вызывайте `captureException` внутри `try/catch`, если ошибка уже обрабатывается ErrorBoundary или глобальным обработчиком — будет дубль. Проверяйте, кто обрабатывает ошибку, прежде чем отправлять вручную.

### 6. Конфиденциальные данные

По умолчанию Sentry отправляет заголовки, cookies, тела запросов. Для GDPR и безопасности:

```javascript
Sentry.init({
  dataCollection: {
    userInfo: false,
    httpBodies: [],
  },
  beforeSend(event) {
    if (event.request?.headers) {
      delete event.request.headers["authorization"];
    }
    return event;
  },
});
```

### 7. Vue: `app` обязателен в `Sentry.init`

Без передачи инстанса Vue-приложения Sentry не подключит обработчик ошибок Vue. Это самая частая ошибка при интеграции.

### 8. Next.js: три среды — три конфига

Edge Runtime — это не Node.js. Если вы забудете `sentry.edge.config.ts`, ошибки в middleware и edge functions не будут отправлены.

### 9. Nuxt: `--import` при запуске

Без флага `--import` серверный Sentry не загружается. Это неочевидно и легко забыть, особенно в Docker-контейнерах.

### 10. `debug: true` — только для отладки

Включайте для диагностики, но **не забывайте выключать** перед релизом. В production это генерирует лишний шум в консоли.

---

## Анализ ошибок в Sentry

### Issues

Каждая уникальная ошибка группируется в Issue. Обращайте внимание на:

- **First seen / Last seen** — новая ошибка или регрессия
- **Environment** — production / staging / development
- **Release** — в каком релизе появилась (требует настройки release tracking)
- **Tags** — фильтруйте по `browser`, `os`, `device`, кастомным тегам
- **Assignee** — назначайте ответственного

### Traces

- **Web Vitals** (LCP, FID, CLS) — собираются автоматически
- **Custom spans** — для бизнес-операций (checkout, поиск, загрузка данных)
- **Distributed tracing** — сквозная трассировка от фронтенда до бэкенда через `tracePropagationTargets`

### Session Replay

Видео-запись сессии. Самый мощный инструмент для воспроизведения багов:
- Видно, что делал пользователь
- Видно сетевые запросы и консольные логи
- `replaysOnErrorSampleRate: 1.0` гарантирует запись при ошибках

---

## Лучшие практики

Эти рекомендации основаны на опыте команд, которые используют Sentry в production. Каждая практика решает конкретную проблему: шум, безопасность, производительность, стоимость.

### 1. Разделяйте окружения

Разные окружения — разные DSN. Это позволяет фильтровать ошибки по окружению в дашборде и настраивать разные алерты. Production-ошибки триггерят PagerDuty, staging — только Slack. Не используйте один DSN для всех окружений — потеряете контекст.

```jsx
Sentry.init({
  environment: process.env.NODE_ENV,
  dsn: process.env.NODE_ENV === "production"
    ? process.env.SENTRY_DSN_PROD
    : process.env.SENTRY_DSN_DEV,
});
```

### 2. Игнорируйте шум

Не все ошибки都值得 логировать. `ResizeObserver loop limit exceeded` — безвредная ошибка браузера, не влияет на пользователя. `Loading chunk N failed` — пользователь обновил страницу во время деплоя, не критично. `ignoreErrors` убирает их из Sentry полностью. `denyUrls` блокирует ошибки из расширений браузера и `chrome://` — они не ваша проблема.

```jsx
Sentry.init({
  ignoreErrors: [
    "ResizeObserver loop limit exceeded",
    "Non-Error promise rejection captured",
    /Loading chunk \d+ failed/,
  ],
  denyUrls: [
    /extensions\//i,
    /^chrome:\/\//i,
  ],
});
```

### 3. Sample rates для production

`tracesSampleRate: 1.0` отправляет каждую транзакцию. При 1000 запросов/минуту это 1.44 миллиона транзакций/день — квота закончится за часы. В production ставьте `0.1` (10%) или ниже. Для Replay стратегия другая: `replaysSessionSampleRate: 0.1` пишет 10% всех сессий, но `replaysOnErrorSampleRate: 1.0` гарантирует запись при ошибках — вы не пропустите критичные баги.

```jsx
Sentry.init({
  tracesSampleRate: process.env.NODE_ENV === "production" ? 0.2 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

### 4. Не логируйте секреты

Sentry отправляет заголовки, cookies, тела запросов по умолчанию. Это означает, что `Authorization: Bearer ...` может попасть в логи. Для GDPR и безопасности настройте `beforeSend` — хук, который модифицирует событие перед отправкой. Удаляйте чувствительные заголовки, токены, PII. Альтернатива — `dataCollection` для глобального отключения сбора данных.

```jsx
// Опасно
Sentry.captureException(error, {
  extra: { password: user.password },
});

// Безопасно
Sentry.captureException(error, {
  extra: { userId: user.id },
});

// Настройте scrubbing
Sentry.init({
  beforeSend(event) {
    if (event.request?.headers) {
      delete event.request.headers["authorization"];
    }
    return event;
  },
});
```

### 5. Добавляйте контекст к ошибкам

Ошибка без контекста — это "что-то сломалось". С контекстом — "не удалось загрузить заказ #12345 для пользователя в фиче orders". `tags` для фильтрации, `extra` для деталей, `contexts` для структурированных данных. Это экономит часы при отладке: вы сразу видите, какой заказ, какой статус, какой пользователь.

```jsx
async function fetchOrder(orderId) {
  try {
    const res = await fetch(`/api/orders/${orderId}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (error) {
    Sentry.captureException(error, {
      tags: { feature: "orders" },
      extra: { orderId },
      contexts: {
        order: { id: orderId, status: "pending" },
      },
    });
    throw error;
  }
}
```

---

## Антипаттерны

Антипаттерны — это то, что выглядит правильно, но приводит к проблемам: шум, ложная уверенность, пропущенные ошибки. Каждый антипаттерн ниже — реальная ошибка, которую совершают команды при интеграции Sentry.

### 1. Отправка каждой ошибки

Логировать каждую ошибку — путь к alert fatigue. Если `api.getData()` падает из-за временной проблемы сети, вы получите 1000 ошибок в минуту. Фильтруйте по критичности: 5xx — серверная ошибка, логируем. 4xx — клиентская ошибка, не всегда наша проблема. Используйте `toast` для UX, `captureException` только для критичных случаев.

```jsx
// Слишком много шума
try {
  await api.getData();
} catch (error) {
  Sentry.captureException(error);
}

// Только критические ошибки
try {
  await api.getData();
} catch (error) {
  if (error.status >= 500) {
    Sentry.captureException(error);
  }
  toast.error("Failed to load data");
}
```

### 2. Отсутствие release tracking

Без `release` в `Sentry.init` вы не знаете, в какой версии появилась ошибка. Это критично для приоритизации: баг в версии 1.0 (которая уже не используется) менее важен, чем регрессия в 2.0. Всегда указывайте `release` — это строка, которая идентифицирует версию приложения. Обычно это `name@version` из `package.json`.

### 3. Игнорирование performance monitoring

Только ошибки — это половина картины. Вы видите, **что** сломалось, но не **как быстро** работает приложение. Страница может грузиться 10 секунд без единой ошибки — пользователи уйдут, но в Sentry будет чисто. `tracesSampleRate` + `replaysSessionSampleRate` дают полную картину: ошибки + производительность + видео сессий.

```jsx
// Только ошибки
Sentry.init({
  dsn: "...",
});

// Ошибки + производительность
Sentry.init({
  dsn: "...",
  tracesSampleRate: 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

### 4. Отсутствие error boundaries

Без ErrorBoundary ошибка в одном компоненте ломает всё приложение. React размонтирует всё дерево — пользователь видит белый экран. ErrorBoundary оборачивает критические части приложения (роутер, виджеты, формы) и показывает fallback-UI. Ошибка в одном виджете не убьёт всю страницу. Используйте `Sentry.ErrorBoundary` или кастомный класс-компонент.

```jsx
// Ошибка в одном компоненте ломает всё приложение
function App() {
  return <Router />;
}

// Error Boundary оборачивает критические части
function App() {
  return (
    <Sentry.ErrorBoundary fallback={<ErrorFallback />}>
      <Router />
    </Sentry.ErrorBoundary>
  );
}
```

---

## Чек-лист интеграции

- [ ] Установлен правильный SDK (`@sentry/react`, `@sentry/vue`, `@sentry/nextjs`, `@sentry/nuxt`)
- [ ] Инициализация — **первый импорт** в приложении
- [ ] `app` передан в `Sentry.init` (для Vue)
- [ ] Source maps настроены и загружаются
- [ ] `tracesSampleRate` снижен для production (0.1 или ниже)
- [ ] `SENTRY_AUTH_TOKEN` настроен в CI/CD
- [ ] `debug: true` включён для диагностики, выключен перед релизом
- [ ] AdBlock решён через tunnel (если нужно)
- [ ] Конфиденциальные данные фильтруются через `beforeSend` / `dataCollection`
- [ ] ErrorBoundary / GlobalError настроены
- [ ] Серверные конфиги подключены (Next.js — три файла, Nuxt — `--import`)
- [ ] Пользовательский контекст (`setUser`, `setTag`) добавлен
- [ ] Release tracking настроен
- [ ] Алерты настроены
- [ ] Тестовая ошибка успешно отправлена и видна в Sentry
