# Безопасность в React и Next.js

## Содержание

1. [Введение](#введение)
2. [XSS (Cross-Site Scripting)](#xss-cross-site-scripting)
3. [dangerouslySetInnerHTML](#dangerouslysetinnerhtml)
4. [CSRF (Cross-Site Request Forgery)](#csrf-cross-site-request-forgery)
5. [Безопасность Server Actions](#безопасность-server-actions)
6. [Content Security Policy (CSP)](#content-security-policy-csp)
7. [Безопасная работа с секретами](#безопасная-работа-с-секретами)
8. [Аутентификация и авторизация](#аутентификация-и-авторизация)
9. [Безопасность зависимостей](#безопасность-зависимостей)
10. [HTTP-заголовки безопасности](#http-заголовки-безопасности)
11. [Безопасность в Next.js](#безопасность-в-nextjs)
12. [Чек-лист безопасности](#чек-лист-безопасности)
13. [Лучшие практики](#лучшие-практики)
14. [Антипаттерны](#антипаттерны)

---

## Введение

React и Next.js предоставляют защиту от многих атак «из коробки», но разработчик всё равно должен понимать векторы атак и правильно настраивать защиту. Основные угрозы для React/Next-приложений:

- **XSS** — внедрение вредоносного скрипта
- **CSRF** — подделка межсайтовых запросов
- **Инъекции через Server Actions** — выполнение произвольного кода на сервере
- **Утечка секретов** — наличие API-ключей в клиентском бандле
- **Небезопасные зависимости** — уязвимости в npm-пакетах

> ⚠️ **CVE-2025-55182:** В 2025 году была обнаружена критическая уязвимость RCE (Remote Code Execution) в Server Actions React, затрагивающая версии 19.0.0–19.2.2. Всегда обновляйте React и Next.js до последних патч-версий.

---

## XSS (Cross-Site Scripting)

XSS — инъекция вредоносного JavaScript-кода в страницу. React **автоматически экранирует** все значения, вставленные в JSX, что защищает от большинства XSS-атак.

### Как React защищает от XSS

```jsx
function Comment({ text }) {
  // ✅ Безопасно: React экранирует HTML-теги
  return <div>{text}</div>;
}

// Если text = "<script>alert('xss')</script>"
// React отрендерит: <div>&lt;script&gt;alert('xss')&lt;/script&gt;</div>
// Скрипт НЕ выполнится
```

React экранирует:
- Строки в JSX-выражениях `{variable}`
- Значения атрибутов `<div title={userInput}>`
- Дочерний текст компонентов

### Когда XSS всё ещё возможен

#### 1. dangerouslySetInnerHTML (см. ниже)

#### 2. URL-атрибуты

```jsx
// ❌ Уязвимость: javascript: URL
const userUrl = "javascript:alert('xss')";
<a href={userUrl}>Click me</a>

// ✅ Защита: валидация URL
function SafeLink({ href, children }) {
  const isSafe = /^https?:\/\//.test(href);
  if (!isSafe) return <span>{children}</span>;
  return <a href={href}>{children}</a>;
}
```

#### 3. ref через callback

```jsx
// ❌ Потенциальная уязвимость: прямой доступ к DOM
function UserProfile({ bio }) {
  const ref = useRef(null);

  useEffect(() => {
    if (ref.current) {
      ref.current.innerHTML = bio;
    }
  }, [bio]);

  return <div ref={ref} />;
}
```

#### 4. Third-party библиотеки

Библиотеки, которые рендерят HTML напрямую (markdown-парсеры, rich-text editors), могут быть уязвимы, если не санитизируют ввод:

```jsx
import DOMPurify from "dompurify";
import { marked } from "marked";

function MarkdownContent({ content }) {
  // ✅ Санитизация перед рендерингом
  const cleanHtml = DOMPurify.sanitize(marked(content));
  return <div dangerouslySetInnerHTML={{ __html: cleanHtml }} />;
}
```

---

## dangerouslySetInnerHTML

`dangerouslySetInnerHTML` — escape-hatch для рендеринга сырого HTML. Используйте только когда **абсолютно необходимо** и **всегда санитизируйте** ввод.

```jsx
// ❌ ОПАСНО: пользовательский ввод без санитизации
function Comment({ html }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// ✅ Безопасно: санитизация через DOMPurify
import DOMPurify from "dompurify";

function SafeHtml({ html }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p"],
    ALLOWED_ATTR: ["href"],
  });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### Когда оправдано

- Рендеринг контента из CMS (Sanity, Contentful) после санитизации
- Интеграция с legacy-библиотеками (jQuery-плагины, rich-text editors)
- Рендеринг markdown/HTML из доверенного источника

### Когда НЕ оправдано

- Пользовательский контент без санитизации
- Когда можно использовать обычный JSX
- Для стилизации (используйте CSS-in-JS или CSS-модули)

---

## CSRF (Cross-Site Request Forgery)

CSRF — атака, при которой злоумышленник заставляет браузер жертвы выполнить нежелательный запрос к авторизованному сайту.

### Как работает CSRF

1. Пользователь авторизован на `bank.com` (куки сессии в браузере)
2. Пользователь открывает страницу злоумышленника `evil.com`
3. `evil.com` содержит скрытую форму: `<form action="https://bank.com/transfer" method="POST">`
4. Браузер автоматически отправляет куки с `bank.com` вместе с запросом
5. `bank.com` считает запрос легитимным

### Защита в React/Next

#### 1. CSRF-токены (основной метод)

```tsx
// Next.js Route Handler — генерация CSRF-токена
import { cookies } from "next/headers";
import { randomBytes } from "crypto";

export async function GET() {
  const token = randomBytes(32).toString("hex");
  cookies().set("csrf_token", token, {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    path: "/",
  });

  return Response.json({ csrfToken: token });
}

// Клиентский компонент — отправка с токеном
async function submitForm(data: FormData) {
  const res = await fetch("/api/action", {
    method: "POST",
    headers: {
      "X-CSRF-Token": csrfToken,
    },
    body: JSON.stringify(data),
  });
}

// Серверная валидация
export async function POST(req: Request) {
  const token = req.headers.get("X-CSRF-Token");
  const storedToken = cookies().get("csrf_token")?.value;

  if (!token || token !== storedToken) {
    return Response.json({ error: "Invalid CSRF token" }, { status: 403 });
  }

  // обработка запроса
}
```

#### 2. SameSite Cookies

```tsx
// middleware.ts или next.config.js
cookies().set("session_id", value, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
});
```

`SameSite=Strict` запрещает браузеру отправлять куки при cross-site запросах.

#### 3. Проверка Origin/Referer

```tsx
export async function POST(req: Request) {
  const origin = req.headers.get("origin") || req.headers.get("referer");
  const allowedOrigins = ["https://mysite.com"];

  if (!origin || !allowedOrigins.some((o) => origin.startsWith(o))) {
    return Response.json({ error: "Invalid origin" }, { status: 403 });
  }
}
```

---

## Безопасность Server Actions

Server Actions (React 19 / Next.js) выполняются на сервере, но вызываются с клиента. Это создаёт новые векторы атак.

### CVE-2025-55182 — критическая уязвимость

В версиях React 19.0.0–19.2.2 обнаружена уязвимость RCE (Remote Code Execution) в Server Actions. Злоумышленник мог выполнить произвольный код на сервере через специально сформированные запросы.

**Решение:** Обновите React до 19.0.4+ / 19.1.4+ / 19.2.3+ и Next.js до 14.2.35 / 15.2.8 / 16.0.10+.

### Валидация входных данных

Server Actions принимают данные от клиента — **всегда валидируйте**:

```tsx
"use server";

import { z } from "zod";

const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1).max(10000),
  category: z.enum(["tech", "life", "news"]),
});

export async function createPost(formData: FormData) {
  const rawData = {
    title: formData.get("title"),
    content: formData.get("content"),
    category: formData.get("category"),
  };

  const result = createPostSchema.safeParse(rawData);
  if (!result.success) {
    return { error: result.error.flatten() };
  }

  // ✅ Теперь данные валидны
  await db.posts.create(result.data);
  revalidatePath("/posts");
}
```

### Проверка авторизации

```tsx
"use server";

import { auth } from "@/lib/auth";
import { redirect } from "next/navigation";

export async function deletePost(postId: string) {
  const session = await auth();

  if (!session) {
    throw new Error("Unauthorized");
  }

  const post = await db.posts.findById(postId);

  if (post.userId !== session.user.id) {
    throw new Error("Forbidden");
  }

  await db.posts.delete(postId);
  revalidatePath("/posts");
}
```

### Rate Limiting

Server Actions могут быть вызваны многократно — добавьте rate limiting:

```tsx
"use server";

import { rateLimit } from "@/lib/rate-limit";

const limiter = rateLimit({
  interval: 60_000,
  uniqueTokenPerInterval: 50,
});

export async function sendMessage(formData: FormData) {
  try {
    await limiter.check(5, "SEND_MESSAGE");
  } catch {
    return { error: "Too many requests" };
  }

  // обработка сообщения
}
```

---

## Content Security Policy (CSP)

CSP — HTTP-заголовок, который ограничивает источники, откуда браузер может загружать ресурсы (скрипты, стили, изображения). Это вторая линия защиты после XSS-экранирования.

### Настройка CSP в Next.js

```tsx
// next.config.js
const ContentSecurityPolicy = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self' https://api.mysite.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
`;

module.exports = {
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          {
            key: "Content-Security-Policy",
            value: ContentSecurityPolicy.replace(/\s{2,}/g, " ").trim(),
          },
        ],
      },
    ];
  },
};
```

### CSP в middleware (для динамических nonce)

```tsx
// middleware.ts
import { NextResponse } from "next/server";
import { nanoid } from "nanoid";

export function middleware(request: Request) {
  const nonce = nanoid();
  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
    style-src 'self' 'nonce-${nonce}';
  `;

  const response = NextResponse.next();
  response.headers.set("x-nonce", nonce);
  response.headers.set("Content-Security-Policy", csp);

  return response;
}
```

### Report-Only режим

Для тестирования CSP без блокировки:

```tsx
{
  key: "Content-Security-Policy-Report-Only",
  value: csp + " report-uri /api/csp-report";
}
```

Браузер будет отправлять отчёты о нарушениях, но не блокировать ресурсы.

---

## Безопасная работа с секретами

### Секреты на сервере (Next.js)

```tsx
// ✅ Правильно: переменные окружения на сервере
// .env.local (не коммитится в git)
DATABASE_URL=postgresql://...
STRIPE_SECRET_KEY=sk_...
JWT_SECRET=...

// app/api/payments/route.ts
export async function POST(req: Request) {
  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
  // обработка платежа
}
```

### Секреты на клиенте

```tsx
// ❌ ОПАСНО: секрет попадёт в клиентский бандл
// .env.local
NEXT_PUBLIC_API_KEY=sk_...

// components/PaymentForm.tsx
const apiKey = process.env.NEXT_PUBLIC_API_KEY;
// sk_... будет в JavaScript-бандле, доступном всем!

// ✅ Правильно: только публичные ключи
// .env.local
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_...

// Серверный компонент использует секретный ключ
// app/api/create-payment/route.ts
const secretKey = process.env.STRIPE_SECRET_KEY;
```

### Правила для префикса NEXT_PUBLIC_

| Переменная | Доступна на клиенте? | Что хранить |
|---|---|---|
| `DATABASE_URL` | ❌ Нет | URL базы данных |
| `API_SECRET` | ❌ Нет | Секретные ключи API |
| `NEXT_PUBLIC_API_URL` | ✅ Да | Публичные URL |
| `NEXT_PUBLIC_GA_ID` | ✅ Да | ID аналитики |

### Защита .env файлов

```bash
# .gitignore
.env
.env.local
.env.*.local

# НЕ коммитить секреты никогда
```

### Ротация секретов

Используйте сервисы управления секретами для production:
- **Vercel Environment Variables** — для Vercel-деплоя
- **AWS Secrets Manager** / **Azure Key Vault** — для облачных инфраструктур
- **Doppler** / **Infisical** — для управления секретами в команде

---

## Аутентификация и авторизация

### NextAuth.js / Auth.js

```tsx
// lib/auth.ts
import NextAuth from "next-auth";
import GitHub from "next-auth/providers/github";
import Credentials from "next-auth/providers/credentials";

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    GitHub,
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      authorize: async (credentials) => {
        const parsed = loginSchema.safeParse(credentials);
        if (!parsed.success) return null;

        const user = await db.users.findByEmail(parsed.data.email);
        if (!user) return null;

        const isValid = await bcrypt.compare(parsed.data.password, user.passwordHash);
        if (!isValid) return null;

        return { id: user.id, email: user.email, role: user.role };
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.role = token.role;
      return session;
    },
  },
  cookies: {
    sessionToken: {
      name: "session-token",
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax",
      path: "/",
    },
  },
});
```

### Middleware для защиты маршрутов

```tsx
// middleware.ts
import { auth } from "@/lib/auth";
import { NextResponse } from "next/server";

export default auth((req) => {
  const isLoggedIn = !!req.auth;
  const isOnProtectedRoute = req.nextUrl.pathname.startsWith("/dashboard");

  if (isOnProtectedRoute && !isLoggedIn) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
});

export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*"],
};
```

### Проверка авторизации в Server Components

```tsx
// app/dashboard/page.tsx
import { auth } from "@/lib/auth";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const session = await auth();

  if (!session) {
    redirect("/login");
  }

  return <div>Welcome, {session.user.name}</div>;
}
```

### Проверка авторизации в Server Actions

```tsx
"use server";

import { auth } from "@/lib/auth";

export async function updateProfile(formData: FormData) {
  const session = await auth();

  if (!session) {
    throw new Error("Unauthorized");
  }

  const userId = session.user.id;
  // обновление профиля
}
```

---

## Безопасность зависимостей

### Аудит зависимостей

```bash
# npm
npm audit

# pnpm
pnpm audit

# yarn
yarn audit
```

### Автоматический аудит в CI/CD

```yaml
# .github/workflows/security.yml
name: Security Audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm audit --audit-level=high
```

### Инструменты

- **Snyk** — автоматический мониторинг уязвимостей
- **Socket** — анализ supply chain рисков
- **npm audit fix** — автоматическое исправление уязвимостей

### Lock-файлы

Всегда коммитьте `package-lock.json`, `pnpm-lock.yaml` или `yarn.lock` — они гарантируют воспроизводимость сборок и защиту от dependency confusion атак.

---

## HTTP-заголовки безопасности

### Рекомендуемые заголовки

```tsx
// middleware.ts или next.config.js
const securityHeaders = [
  {
    key: "X-Frame-Options",
    value: "DENY",
  },
  {
    key: "X-Content-Type-Options",
    value: "nosniff",
  },
  {
    key: "Referrer-Policy",
    value: "strict-origin-when-cross-origin",
  },
  {
    key: "Permissions-Policy",
    value: "camera=(), microphone=(), geolocation=()",
  },
  {
    key: "X-XSS-Protection",
    value: "1; mode=block",
  },
  {
    key: "Strict-Transport-Security",
    value: "max-age=31536000; includeSubDomains",
  },
];
```

### Применение в Next.js

```tsx
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: securityHeaders,
      },
    ];
  },
};
```

---

## Безопасность в Next.js

### Изоляция серверного и клиентского кода

```tsx
// ✅ Секреты только в Server Components
// app/api/users/route.ts
export async function GET() {
  const users = await db.users.findMany();
  return Response.json(users);
}

// ❌ НЕ экспортируйте секретные данные в клиентские компоненты
// app/page.tsx — Server Component
import { db } from "@/lib/db";

export default async function Page() {
  const users = await db.users.findMany();
  return <ClientComponent users={users} />;
}

// components/ClientComponent.tsx — Client Component
"use client";
export default function ClientComponent({ users }) {
  // ✅ users — это данные, не секреты
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### Валидация параметров маршрутов

```tsx
// app/users/[id]/page.tsx
import { z } from "zod";

const paramsSchema = z.object({
  id: z.string().uuid(),
});

export default async function UserPage({ params }: { params: { id: string } }) {
  const parsed = paramsSchema.safeParse(params);

  if (!parsed.success) {
    return <div>Invalid user ID</div>;
  }

  const user = await db.users.findById(parsed.data.id);
  return <div>{user.name}</div>;
}
```

### Защита от Open Redirect

```tsx
// ❌ Уязвимость: redirect на любой URL
const callbackUrl = searchParams.get("callbackUrl");
redirect(callbackUrl);

// ✅ Защита: валидация URL
const callbackUrl = searchParams.get("callbackUrl");
const isInternal = callbackUrl?.startsWith("/");

if (isInternal) {
  redirect(callbackUrl);
} else {
  redirect("/");
}
```

---

## Чек-лист безопасности

### Перед деплоем

- [ ] Обновлены React, Next.js и все зависимости
- [ ] Проверено `npm audit` / `pnpm audit`
- [ ] Секреты не в клиентском бандле (проверить через `NEXT_PUBLIC_`)
- [ ] Настроены HTTP-заголовки безопасности
- [ ] Настроен CSP (хотя бы в report-only режиме)
- [ ] Server Actions валидируют входные данные
- [ ] Проверка авторизации во всех защищённых маршрутах и действиях
- [ ] CSRF-защита для критических операций
- [ ] Rate limiting для публичных API
- [ ] `.env` файлы в `.gitignore`
- [ ] Cookie с флагами `httpOnly`, `secure`, `sameSite`

### Регулярно

- [ ] Аудит зависимостей (еженедельно/ежемесячно)
- [ ] Ротация секретов (ежеквартально)
- [ ] Проверка CSP-отчётов
- [ ] Penetration testing (для критических приложений)

---

## Лучшие практики

### 1. Принцип наименьших привилегий

Каждый компонент, API-эндпоинт и Server Action должны иметь минимально необходимые права:

```tsx
// ✅ Проверка прав на уровне действия
export async function deleteUser(userId: string) {
  const session = await auth();
  if (session.user.role !== "admin") {
    throw new Error("Forbidden");
  }
  await db.users.delete(userId);
}
```

### 2. Глубокая защита (Defense in Depth)

Не полагайтесь на одну линию защиты:

```tsx
// 1. Middleware — проверка авторизации
// 2. Server Component — проверка прав
// 3. Server Action — валидация данных
// 4. Database — уникальные constraints

// Все слои работают вместе
```

### 3. Безопасность по умолчанию

```tsx
// ✅ Компонент безопасен по умолчанию
function ExternalLink({ href, children }) {
  return (
    <a href={href} rel="noopener noreferrer" target="_blank">
      {children}
    </a>
  );
}

// rel="noopener noreferrer" предотвращает tabnabbing-атаку
```

---

## Антипаттерны

### 1. Хранение секретов в клиентском коде

```tsx
// ❌ Секрет в клиентском компоненте
"use client";
const API_KEY = "sk_live_...";
fetch(`https://api.service.com/data?key=${API_KEY}`);

// ✅ Серверный прокси
// app/api/proxy/route.ts
export async function GET() {
  const res = await fetch("https://api.service.com/data", {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` },
  });
  return Response.json(await res.json());
}
```

### 2. Игнорирование валидации

```tsx
// ❌ Доверие клиентским данным
"use server";
export async function createUser(formData: FormData) {
  const name = formData.get("name");
  const email = formData.get("email");
  await db.users.create({ name, email });
}

// ✅ Валидация через Zod
"use server";
const schema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
});

export async function createUser(formData: FormData) {
  const data = schema.parse({
    name: formData.get("name"),
    email: formData.get("email"),
  });
  await db.users.create(data);
}
```

### 3. Хранение токенов в localStorage

```tsx
// ❌ Уязвимо к XSS
localStorage.setItem("token", jwt);

// ✅ HttpOnly cookie (устанавливается сервером)
cookies().set("session", jwt, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
});
```

### 4. Отсутствие rate limiting

```tsx
// ❌ Неограниченные запросы
export async function POST(req: Request) {
  const data = await req.json();
  await sendEmail(data.email);
  return Response.json({ ok: true });
}

// ✅ Rate limiting
export async function POST(req: Request) {
  await limiter.check(3, "SEND_EMAIL");
  const data = await req.json();
  await sendEmail(data.email);
  return Response.json({ ok: true });
}
```
