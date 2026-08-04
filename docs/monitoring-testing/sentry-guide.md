# Sentry: Полное руководство по интеграции

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
8. [Подводные камни и неочевидности](#подводные-камни-и-неочевидности)
9. [Анализ ошибок в Sentry](#анализ-ошибок-в-sentry)
10. [Чек-лист интеграции](#чек-лист-интеграции)

---

## Что такое Sentry и зачем он нужен

Sentry — платформа для мониторинга ошибок и производительности в реальном времени. Она решает четыре задачи:

- **Error Monitoring** — перехват неперехваченных исключений, необработанных промисов и ошибок рендеринга.
- **Performance Monitoring (Tracing)** — измерение скорости загрузки страниц, API-запросов и пользовательских операций.
- **Session Replay** — видео-запись того, что видел пользователь до, во время и после ошибки.
- **Logs** — структурированные логи, коррелированные с ошибками и трейсами.

Главная идея: вы не ждёте, пока пользователь напишет в поддержку. Вы видите ошибку, стектрейс, контекст и запись экрана — ещё до того, как пользователь сам понял, что что-то сломалось.

---

## React (SPA)

### Установка

```bash
npm install @sentry/react --save
```

### Инициализация

Создайте файл `instrument.js` в корне проекта. Этот файл должен быть импортирован **первым** — раньше любого другого кода, включая React.

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

**React 19+** — используйте встроенные хуки ошибок:

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

**React 18 и ниже** — используйте ErrorBoundary:

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

### React Router

Если используете React Router, замените `browserTracingIntegration` на специализированную интеграцию:

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

Для захвата состояния Redux в ошибках:

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
});
```

**Глобальный обработчик ошибок рендеринга** — `app/global-error.tsx`:

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

```javascript
import { createPinia } from "pinia";
import { createSentryPiniaPlugin } from "@sentry/vue";

const pinia = createPinia();
pinia.use(createSentryPiniaPlugin());
```

### Поздняя инициализация

Если приложение создаётся не сразу (например, микрофронтенды):

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

### Установка

```bash
npx sentry@latest init
```

Или через классический wizard:

```bash
npx @sentry/wizard@latest -i nuxt
```

Требования: Nuxt >= 3.7.0 (рекомендуется 3.14.0+).

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

Nuxt, как и Next.js, работает и на клиенте, и на сервере. Серверный Sentry нужно подключать **явно** через флаг `--import`.

**Production:**

```bash
nuxi build
node --import ./.output/server/sentry.server.config.mjs .output/server/index.mjs
```

**Development:**

```bash
# Первый запуск — без флага (сгенерирует конфиг в .nuxt)
nuxt dev

# Повторные запуски:
NODE_OPTIONS='--import ./.nuxt/dev/sentry.server.config.mjs' nuxt dev
```

Если удалите папку `.nuxt` — нужно снова запустить `nuxt dev` без флага, чтобы перегенерировать серверный конфиг.

---

## Source Maps — обязательно

Без source maps стектрейсы в production указывают на минифицированный код — бесполезно.

```bash
npx @sentry/wizard@latest -i sourcemaps
```

Для CI/CD установите переменную окружения:

```bash
SENTRY_AUTH_TOKEN=sntrys_eyJ...
```

Файл `.env.sentry-build-plugin` создаётся автоматически и добавляется в `.gitignore`. Проверьте, что он не попадает в репозиторий.

---

## Правильное логирование ошибок

### captureException — перехваченные ошибки

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

```javascript
Sentry.captureMessage("Пользователь отменил заказ", {
  level: "warning",
  extra: { orderId: "12345" },
});
```

### Структурированные логи

```javascript
Sentry.logger.info("Пользователь вошёл", { userId: "123" });
Sentry.logger.warn("Медленный ответ API", { duration: 5000, endpoint: "/api/orders" });
Sentry.logger.error("Операция не удалась", { reason: "timeout" });
```

### Обогащение контекста

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

```javascript
const result = await Sentry.startSpan(
  { op: "db.query", name: "Получить заказы пользователя" },
  async () => {
    return await db.query("SELECT * FROM orders WHERE user_id = ?", [userId]);
  }
);
```

---

## Подводные камни и неочевидности

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

Включайте для диагностики, но **не забывайте выключить** перед релизом. В production это генерирует лишний шум в консоли.

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

### Alerts

Настройте алерты:
- **Issue Alert** — уведомление при новой ошибке или регрессии
- **Metric Alert** — уведомление при превышении порога (например, > 100 ошибок за 5 минут)

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
- [ ] Тестовая ошибка успешно отправлена и видна в Sentry
- [ ] Алерты настроены
