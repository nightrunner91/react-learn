# Мониторинг React и Next.js через Sentry

## Содержание

1. [Что такое Sentry](#что-такое-sentry)
2. [Зачем нужен мониторинг](#зачем-нужен-мониторинг)
3. [Установка и настройка](#установка-и-настройка)
4. [Перехват ошибок в React](#перехват-ошибок-в-react)
5. [Перехват ошибок в Next.js](#перехват-ошибок-в-nextjs)
6. [Performance Monitoring](#performance-monitoring)
7. [Release Health](#release-health)
8. [Source Maps](#source-maps)
9. [Custom Context и Breadcrumbs](#custom-context-и-breadcrumbs)
10. [User Feedback](#user-feedback)
11. [Alerts и интеграции](#alerts-и-интеграции)
12. [Лучшие практики](#лучшие-практики)
13. [Антипаттерны](#антипаттерны)

---

## Что такое Sentry

**Sentry** — платформа мониторинга ошибок и производительности в реальном времени. Она отслеживает:

- **Exceptions** — неперехваченные ошибки JavaScript
- **Performance issues** — медленные загрузки страниц, API-запросы, операции
- **Releases** — какие ошибки появились в новых версиях
- **User impact** — сколько пользователей затронуто

Sentry можно развернуть:
- **Sentry SaaS** — облачная версия (sentry.io)
- **Self-hosted** — на собственной инфраструктуре (Docker)

---

## Зачем нужен мониторинг

### Проблема: «у меня работает»

Разработчик тестирует на своём компьютере, в своём браузере, с хорошими данными. Пользователи сталкиваются с:

- Ошибками на конкретных устройствах/браузерах
- Проблемами производительности на слабых сетях
- Ошибками в edge-case сценариях
- Регрессиями после деплоя

Без мониторинга вы узнаёте о проблемах из жалоб пользователей — часто слишком поздно.

### Что даёт Sentry

- **Мгновенное уведомление** об ошибках в production
- **Полный стек-трейс** с source maps
- **Контекст** — браузер, ОС, устройство, версия приложения
- **Хлебные крошки** — действия пользователя перед ошибкой
- **Release tracking** — какая версия сломалась
- **Performance metrics** — Web Vitals, время загрузки

---

## Установка и настройка

### React (Vite / CRA)

```bash
npm install @sentry/react
```

```jsx
// main.tsx
import * as Sentry from "@sentry/react";
import { BrowserTracing } from "@sentry/tracing";

Sentry.init({
  dsn: "https://examplePublicKey@o0.ingest.sentry.io/0",
  integrations: [
    new BrowserTracing({
      tracePropagationTargets: ["localhost", /^https:\/\/yourserver\.io\/api/],
    }),
    new Sentry.Replay({
      maskAllText: false,
      blockAllMedia: false,
    }),
  ],

  // Performance
  tracesSampleRate: 1.0,

  // Session Replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,

  // Environment
  environment: process.env.NODE_ENV,

  // Release
  release: `my-app@${process.env.APP_VERSION}`,
});

// Глобальный Error Boundary
const SentryErrorBoundary = Sentry.ErrorBoundary;

function App() {
  return (
    <SentryErrorBoundary fallback={<ErrorFallback />}>
      <Router />
    </SentryErrorBoundary>
  );
}
```

### Next.js (App Router)

```bash
npm install @sentry/nextjs
```

```bash
npx @sentry/wizard@latest -i nextjs
```

Wizard автоматически создаст конфигурационные файлы:

```js
// sentry.client.config.js
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.replayIntegration(),
  ],
});
```

```js
// sentry.server.config.js
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 1.0,
});
```

```js
// sentry.edge.config.js
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 1.0,
});
```

```js
// next.config.js
const { withSentryConfig } = require("@sentry/nextjs");

module.exports = withSentryConfig(
  {
    // ваша конфигурация Next.js
  },
  {
    org: "my-org",
    project: "my-app",
    silent: !process.env.CI,
    widenClientFileUpload: true,
    hideSourceMaps: true,
    disableLogger: true,
  }
);
```

---

## Перехват ошибок в React

### Error Boundary + Sentry

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

// Или через Sentry.ErrorBoundary
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

---

## Перехват ошибок в Next.js

### Server Components

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

### API Routes / Route Handlers

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

### Server Actions

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

### Error.tsx (App Router)

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

### Global Error (App Router)

```tsx
// app/global-error.tsx
"use client";

import * as Sentry from "@sentry/nextjs";
import { useEffect } from "react";

export default function GlobalError({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <html>
      <body>
        <h2>Critical error</h2>
        <button onClick={reset}>Try again</button>
      </body>
    </html>
  );
}
```

---

## Performance Monitoring

### Автоматическое отслеживание

Sentry автоматически отслеживает:

- **Page loads** — время загрузки страниц
- **API calls** — длительность HTTP-запросов
- **Component renders** — время рендеринга компонентов
- **Web Vitals** — LCP, FID, CLS, FCP, TTFB

```jsx
// Автоматически отслеживается после инициализации
Sentry.init({
  tracesSampleRate: 1.0,
});
```

### Custom transactions

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

```tsx
// Middleware для отслеживания всех запросов
import * as Sentry from "@sentry/nextjs";

export default Sentry.wrapMiddleware(async (request) => {
  const response = await NextResponse.next();
  return response;
});
```

---

## Release Health

### Tracking releases

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

Sentry показывает:

- **Crash-free sessions** — процент сессий без крашей
- **Crash-free users** — процент пользователей без крашей
- **Issues per release** — сколько ошибок в каждом релизе
- **Regression detection** — ошибки, которые вернулись

---

## Source Maps

Source maps позволяют Sentry показывать оригинальный код (TypeScript, JSX) вместо транспилированного JavaScript.

### Загрузка source maps

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

`@sentry/nextjs` автоматически загружает source maps при сборке:

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

## Custom Context и Breadcrumbs

### User context

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

Sentry автоматически записывает хлебные крошки:

- HTTP-запросы
- Console.log
- DOM-события (клики)
- Navigation

Можно добавлять кастомные:

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

## User Feedback

### Форма обратной связи

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

### Интеграции

Sentry интегрируется с:

- **Slack / Discord** — уведомления в каналы
- **GitHub / GitLab** — привязка к коммитам и PR
- **Jira / Linear** — создание тикетов
- **PagerDuty** — эскалация критических ошибок
- **Webhooks** — кастомные интеграции

---

## Лучшие практики

### 1. Разделяйте окружения

```jsx
Sentry.init({
  environment: process.env.NODE_ENV,
  dsn: process.env.NODE_ENV === "production"
    ? process.env.SENTRY_DSN_PROD
    : process.env.SENTRY_DSN_DEV,
});
```

### 2. Игнорируйте шум

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

```jsx
Sentry.init({
  tracesSampleRate: process.env.NODE_ENV === "production" ? 0.2 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

### 4. Не логируйте секреты

```jsx
// ❌ Опасно
Sentry.captureException(error, {
  extra: { password: user.password },
});

// ✅ Безопасно
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

### 1. Отправка каждой ошибки

```jsx
// ❌ Слишком много шума
try {
  await api.getData();
} catch (error) {
  Sentry.captureException(error);
}

// ✅ Только критические ошибки
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

Без release tracking вы не знаете, в какой версии появилась ошибка. Всегда указывайте `release` при инициализации.

### 3. Игнорирование performance monitoring

```jsx
// ❌ Только ошибки
Sentry.init({
  dsn: "...",
});

// ✅ Ошибки + производительность
Sentry.init({
  dsn: "...",
  tracesSampleRate: 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

### 4. Отсутствие error boundaries

```jsx
// ❌ Ошибка в одном компоненте ломает всё приложение
function App() {
  return <Router />;
}

// ✅ Error Boundary оборачивает критические части
function App() {
  return (
    <Sentry.ErrorBoundary fallback={<ErrorFallback />}>
      <Router />
    </Sentry.ErrorBoundary>
  );
}
```
