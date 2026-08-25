# Безопасность Next.js: Server Components, Server Actions и заголовки

## Содержание

1. [Почему Next.js меняет подход к безопасности](#почему-nextjs-меняет-подход-к-безопасности)
2. [Разделение серверного и клиентского кода](#разделение-серверного-и-клиентского-кода)
3. [Server Components и утечка данных](#server-components-и-утечка-данных)
4. [Server Actions: валидация и авторизация](#server-actions-валидация-и-авторизация)
5. [Middleware Next.js](#middleware-nextjs)
6. [Route Handlers](#route-handlers)
7. [CSP в Next.js](#csp-в-nextjs)
8. [Заголовки безопасности](#заголовки-безопасности)
9. [Защита от Open Redirect](#защита-от-open-redirect)
10. [App Router vs Pages Router](#app-router-vs-pages-router)
11. [Чек-лист безопасности Next.js](#чек-лист-безопасности-nextjs)
12. [Терминология](#терминология)

---

## Почему Next.js меняет подход к безопасности

Next.js — это full-stack фреймворк. В одном проекте у вас есть:

- Server Components (рендерятся на сервере);
- Client Components (исполняются в браузере);
- Route Handlers (API endpoints);
- Server Actions (серверные функции, вызываемые с клиента);
- Middleware (выполняется до рендеринга).

Это означает, что фронтенд-разработчик Next.js должен думать не только о браузере, но и о серверной части: валидации, авторизации, секретах, заголовках.

Аналогия: раньше фронтенд был витриной магазина, а бэкенд — складом. В Next.js фронтенд-разработчик часто работает и за кассой, и на складе, и у дверей. Значит, он должен знать правила безопасности для всех зон.

---

## Разделение серверного и клиентского кода

### Server Components

- Выполняются только на сервере.
- Могут обращаться к базе данных, секретам, файловой системе.
- Их код не попадает в клиентский бандл.
- По умолчанию все компоненты в App Router — серверные.

### Client Components

- Помечаются директивой `"use client"`.
- Выполняются в браузере.
- Имеют доступ к DOM, localStorage, браузерным API.
- Не должны содержать секреты.

### Граница доверия

```tsx
// ✅ Правильно: секреты только в Server Component
import { db } from '@/lib/db';

export default async function Page() {
  const data = await db.query('SELECT * FROM users');
  return <ClientList data={data} />;
}
```

```tsx
// ⚠️ Осторожно: объект data полностью попадает в клиентский JS,
// даже если рендерится только часть полей
'use client';

export default function ClientList({ data }) {
  return <ul>{data.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

Если `data` содержит `email`, `phone`, `passwordHash`, пользователь увидит их в DevTools.

---

## Server Components и утечка данных

### Пример утечки

```tsx
// app/users/page.tsx
import { db } from '@/lib/db';

export default async function UsersPage() {
  const users = await db.users.findMany();

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

Если `findMany` возвращает все поля, они будут в RSC Payload, даже если рендерится только `name`. Передавайте клиенту только нужные поля.

### Исправление

```tsx
const users = await db.users.findMany({
  select: { id: true, name: true },
});
```

---

## Server Actions: валидация и авторизация

Server Actions — это функции, которые выполняются на сервере, но вызываются с клиента. Они удобны, но создают иллюзию безопасности.

### Главное правило

**Никогда не доверяйте клиенту.** Любой может отправить HTTP-запрос на endpoint Server Action, минуя ваш UI.

### Валидация

```tsx
'use server';

import { z } from 'zod';
import { auth } from '@/lib/auth';

const schema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1).max(10000),
});

export async function createPost(formData: FormData) {
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  const raw = {
    title: formData.get('title'),
    content: formData.get('content'),
  };

  const result = schema.safeParse(raw);
  if (!result.success) {
    return { error: result.error.flatten() };
  }

  await db.posts.create({
    ...result.data,
    authorId: session.user.id,
  });
}
```

### Авторизация

```tsx
'use server';

export async function deletePost(postId: string) {
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  const post = await db.posts.findById(postId);
  if (!post || post.authorId !== session.user.id) {
    throw new Error('Forbidden');
  }

  await db.posts.delete(postId);
}
```

### Rate limiting

```tsx
'use server';

import { rateLimit } from '@/lib/rate-limit';

const limiter = rateLimit({
  interval: 60_000,
  uniqueTokenPerInterval: 50,
});

export async function sendMessage(formData: FormData) {
  try {
    await limiter.check(5, 'SEND_MESSAGE');
  } catch {
    return { error: 'Too many requests' };
  }

  // ...
}
```

### CVE-2025-55182

В 2025 году в Server Actions React была обнаружена критическая уязвимость RCE. Затронуты версии React 19.0.0–19.2.2. Решение — обновить React и Next.js до исправленных версий.

Этот инцидент показывает, что Server Actions — это серверный код, который должен обновляться и мониториться так же, как любой другой сервер.

---

## Middleware Next.js

Middleware выполняется на Edge Runtime перед рендерингом страницы. Это хорошее место для:

- перенаправления неавторизованных пользователей;
- установки security-заголовков;
- проверки геолокации;
- A/B-тестирования.

### Пример: защита маршрутов

```ts
// middleware.ts
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

export default auth((req) => {
  const isLoggedIn = !!req.auth;
  const isProtected = req.nextUrl.pathname.startsWith('/dashboard');

  if (isProtected && !isLoggedIn) {
    return NextResponse.redirect(new URL('/login', req.url));
  }
});

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*'],
};
```

### Ограничения

- Middleware не заменяет проверку авторизации в Server Actions и Route Handlers.
- Edge Runtime имеет ограниченный API (например, нет прямого доступа к некоторым Node.js модулям).

---

## Route Handlers

Route Handlers (`route.ts`) — это API endpoints в App Router. Они заменяют API Routes из Pages Router.

### Пример безопасного Route Handler

```ts
// app/api/users/route.ts
import { auth } from '@/lib/auth';
import { z } from 'zod';

const querySchema = z.object({
  page: z.coerce.number().min(1).default(1),
  limit: z.coerce.number().max(100).default(20),
});

export async function GET(request: Request) {
  const session = await auth();
  if (!session) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { searchParams } = new URL(request.url);
  const result = querySchema.safeParse({
    page: searchParams.get('page'),
    limit: searchParams.get('limit'),
  });

  if (!result.success) {
    return Response.json({ error: 'Invalid query' }, { status: 400 });
  }

  const users = await db.users.paginate(result.data);
  return Response.json(users);
}
```

---

## CSP в Next.js

### Через next.config.js

```js
// next.config.js
const csp = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
`;

module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: csp.replace(/\s{2,}/g, ' ').trim(),
          },
        ],
      },
    ];
  },
};
```

### Через middleware с nonce

```ts
// middleware.ts
import { NextResponse } from 'next/server';
import { randomBytes } from 'crypto';

export function middleware(request: Request) {
  const nonce = randomBytes(16).toString('base64');
  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' data: https:;
    connect-src 'self';
    frame-ancestors 'none';
  `;

  const response = NextResponse.next();
  response.headers.set('Content-Security-Policy', csp.replace(/\s{2,}/g, ' ').trim());
  response.headers.set('x-nonce', nonce);
  return response;
}
```

### Nonce в компоненте

```tsx
// app/layout.tsx
import { headers } from 'next/headers';

export default function RootLayout({ children }) {
  const nonce = headers().get('x-nonce');

  return (
    <html lang="en">
      <body>
        <script nonce={nonce} dangerouslySetInnerHTML={{ __html: '...' }} />
        {children}
      </body>
    </html>
  );
}
```

---

## Заголовки безопасности

```js
// next.config.js
const securityHeaders = [
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains' },
];

module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: securityHeaders,
      },
    ];
  },
};
```

---

## Защита от Open Redirect

```tsx
// ❌ Уязвимо
const callbackUrl = searchParams.get('callbackUrl');
redirect(callbackUrl);

// ✅ Безопасно
const callbackUrl = searchParams.get('callbackUrl');
const isInternal = callbackUrl?.startsWith('/');
redirect(isInternal ? callbackUrl : '/');
```

Для более надёжной проверки используйте белый список разрешённых URL.

---

## App Router vs Pages Router

| Аспект | App Router | Pages Router |
|--------|-----------|--------------|
| Server Components | По умолчанию | Нет |
| Server Actions | Да | Нет |
| Route Handlers | `route.ts` | API Routes |
| Middleware | Да | Да |
| Data fetching | `async` компоненты | `getServerSideProps`, `getStaticProps` |

В Pages Router `getServerSideProps` и `getStaticProps` сериализуют данные в HTML. Если в них попадает пользовательский ввод, это может привести к stored XSS.

---

## Чек-лист безопасности Next.js

### Архитектура

- [ ] Хранить секреты только в Server Components / Route Handlers / Server Actions.
- [ ] Не использовать `NEXT_PUBLIC_` для секретов.
- [ ] Передавать в Client Components только необходимые данные.
- [ ] Использовать серверный прокси для внешних API с секретами.

### Server Actions и Route Handlers

- [ ] Валидировать все входные данные с Zod.
- [ ] Проверять авторизацию на каждом уровне.
- [ ] Проверять владение ресурсом.
- [ ] Добавлять rate limiting.
- [ ] Не доверять параметрам URL.

### Защита в глубину

- [ ] Настроить CSP.
- [ ] Установить security-заголовки.
- [ ] Настроить HttpOnly, Secure, SameSite cookies.
- [ ] Защитить от Open Redirect.
- [ ] Обновлять React, Next.js и зависимости.

---

## Терминология

| Термин | Значение |
|--------|----------|
| **Server Component** | Компонент React, рендерящийся только на сервере |
| **Client Component** | Компонент React, исполняющийся в браузере |
| **Server Action** | Серверная функция, вызываемая с клиента |
| **Route Handler** | API endpoint в App Router |
| **Middleware** | Код, выполняемый перед рендерингом |
| **RSC Payload** | Данные, передаваемые Server Components клиенту |
| **Open Redirect** | Перенаправление на произвольный URL |
| **Edge Runtime** | Среда выполнения middleware в Next.js |

---

## Что важно понимать

Next.js размывает границу между фронтендом и бэкендом. Это даёт мощь, но и ответственность. Фронтенд-разработчик Next.js должен уметь:

- разделять серверный и клиентский код;
- валидировать и авторизовать Server Actions;
- настраивать CSP и security-заголовки;
- защищать от Open Redirect и утечки данных.

Для SOC2 и CASA это означает, что security-контроли должны охватывать и Server Components, и Client Components, и middleware.
