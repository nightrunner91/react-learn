# Next.js как монолит: прямое обращение к базе данных без отдельного API

Next.js App Router позволяет строить полноценные приложения, где серверные компоненты и Server Actions обращаются к базе данных напрямую — без отдельного REST/GraphQL API-слоя. Этот документ детально разбирает подход, его жизнеспособность, подводные камни и необходимые навыки.

## Содержание

- [Что такое монолитный подход в Next.js](#что-такое-монолитный-подход-в-nextjs)
- [Жизнеспособность в коммерческой разработке](#жизнеспособность-в-коммерческой-разработке)
- [Архитектура прямого обращения к БД](#архитектура-прямого-обращения-к-бд)
- [Плюсы подхода](#плюсы-подхода)
- [Минусы подхода](#минусы-подхода)
- [Подводные камни для frontend-разработчика](#подводные-камни-для-frontend-разработчика)
- [Базы данных в монолитном Next.js](#базы-данных-в-монолитном-nextjs)
- [ORM и инструменты работы с БД](#orm-и-инструменты-работы-с-бд)
- [Необходимый уровень знаний для коммерческой разработки](#необходимый-уровень-знаний-для-коммерческой-разработки)
- [Навыки для решения бизнес-задач](#навыки-для-решения-бизнес-задач)
- [Паттерны и примеры](#паттерны-и-примеры)
- [Когда стоит, а когда не стоит](#когда-стоит-а-когда-не-стоит)
- [Сравнение с традиционной архитектурой](#сравнение-с-традиционной-архитектурой)

## Что такое монолитный подход в Next.js

В традиционной веб-разработке архитектура разделена:

```
┌─────────────┐     HTTP/REST     ┌─────────────┐     SQL/ORM     ┌─────────────┐
│  Frontend   │ ───────────────→  │   Backend   │ ──────────────→ │     БД      │
│  (React)    │ ←───────────────  │  (Node.js)  │ ←────────────── │ (PostgreSQL)│
└─────────────┘     JSON          └─────────────┘    данные       └─────────────┘
     3 слоя: клиент → API-сервер → база данных
```

В монолитном Next.js API-слой исчезает. Серверные компоненты и Server Actions обращаются к БД напрямую:

```
┌───────────────────────────────────────────────────────────┐
│                    Next.js (монолит)                       │
│                                                           │
│  ┌──────────────────┐   ┌──────────────────┐             │
│  │ Server Component  │   │  Server Action   │             │
│  │ (async/await)    │──→│  ('use server')  │             │
│  └────────┬─────────┘   └────────┬─────────┘             │
│           │                       │                       │
│           └───────────┬───────────┘                       │
│                       ▼                                   │
│            ┌──────────────────┐                           │
│            │   ORM / Query    │                           │
│            │   Builder        │                           │
│            └────────┬─────────┘                           │
│                     │                                     │
├─────────────────────┼─────────────────────────────────────┤
│                     ▼                                     │
│            ┌──────────────────┐                           │
│            │   База данных    │                           │
│            └──────────────────┘                           │
└───────────────────────────────────────────────────────────┘
         + клиентские компоненты в браузере
```

**Ключевая идея:** код чтения и записи данных живёт в том же проекте, что и UI. Нет отдельного бэкенд-сервиса, нет REST-эндпоинтов, нет контрактов между фронтендом и бэкендом.

### Как это работает технически

```tsx
// app/users/page.tsx — Server Component
import { db } from '@/lib/db' // Prisma client

// Прямой запрос к БД из компонента
export default async function UsersPage() {
  const users = await db.user.findMany({
    include: { posts: true }
  })

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          {user.name} — {user.posts.length} постов
        </li>
      ))}
    </ul>
  )
}
```

```tsx
// app/actions/create-user.ts — Server Action
'use server'

import { db } from '@/lib/db'
import { revalidatePath } from 'next/cache'

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string
  const email = formData.get('email') as string

  await db.user.create({
    data: { name, email }
  })

  revalidatePath('/users')
}
```

```tsx
// app/users/create-form.tsx — Client Component
'use client'

import { createUser } from '@/app/actions/create-user'

export function CreateUserForm() {
  return (
    <form action={createUser}>
      <input name="name" required />
      <input name="email" type="email" required />
      <button type="submit">Создать</button>
    </form>
  )
}
```

Всё это работает потому, что Next.js — это **полноценный Node.js-сервер**. Серверные компоненты выполняются на сервере, и у них есть прямой доступ к любым серверным ресурсам, включая подключение к БД.

## Жизнеспособность в коммерческой разработке

### Текущий статус (2025–2026)

Этот подход **полностью жизнеспособен** и уже не является экспериментом. Вот доказательства:

| Факт | Значение |
|------|----------|
| **Vercel** | Официально продвигает этот подход как «Full-stack React» |
| **Prisma** | 40k+ звёзд на GitHub, используется в продакшене тысячами компаний |
| **Next.js + Prisma** | Самый популярный стартер на GitHub для fullstack-приложений |
| **T3 Stack** (Next.js + tRPC + Prisma + TypeScript) | Массово используется в стартапах и средних проектах |
| **Supabase + Next.js** | Официальная интеграция, используется в коммерческих проектах |

### Где это используется в продакшене

**Стартапы и MVP** — основной сценарий. Одна команда (или даже один разработчик) закрывает и фронтенд, и бэкенд. Скорость разработки критична, отдельный бэкенд-сервис — overkill.

**Средний бизнес** — внутренние инструменты, дашборды, CRM, админ-панели. Команда 2–5 fullstack-разработчиков на Next.js.

**Крупные компании** — используют гибридный подход. Next.js как BFF (Backend for Frontend), который агрегирует данные из микросервисов, но часть данных читает напрямую из БД для производительности.

### Когда это общепринято

```
Общепринято и распространено:
  ✅ Стартапы и MVP
  ✅ Внутренние инструменты
  ✅ Контентные сайты с динамикой
  ✅ SaaS-приложения малого и среднего масштаба
  ✅ Прототипы и proof-of-concept

Требует осторожности:
  ⚠️ Высоконагруженные системы (>10k RPS)
  ⚠️ Микросервисная архитектура
  ⚠️ Команды с чётким разделением frontend/backend
  ⚠️ Системы с жёсткими требованиями к безопасности

Не рекомендуется:
  ❌ Мобильные приложения (нужен отдельный API)
  ❌ Публичные API для третьих сторон
  ❌ Системы с десятками микросервисов
```

### Реальная статистика

По данным State of JS 2024 и опросам разработчиков:

- **~60%** Next.js-проектов используют прямое обращение к БД через ORM
- **~25%** используют Next.js как BFF перед существующим API
- **~15%** используют чистый клиентский SPA с отдельным API

## Архитектура прямого обращения к БД

### Структура проекта

```
my-app/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx          ← Server Component, читает БД
│   │   └── register/page.tsx
│   ├── (dashboard)/
│   │   ├── page.tsx                ← Server Component, агрегирует данные
│   │   ├── users/page.tsx          ← Server Component, список из БД
│   │   └── settings/page.tsx
│   ├── api/
│   │   └── webhooks/
│   │       └── stripe/route.ts     ← Route Handler для внешних событий
│   ├── actions/
│   │   ├── auth.ts                 ← Server Actions (login, register)
│   │   ├── users.ts                ← Server Actions (CRUD)
│   │   └── billing.ts
│   └── layout.tsx
├── lib/
│   ├── db.ts                       ← Prisma client (singleton)
│   ├── auth.ts                     ← Auth logic (NextAuth/Lucia)
│   ├── validators.ts               ← Zod schemas
│   └── utils.ts
├── prisma/
│   ├── schema.prisma               ← Схема БД
│   └── migrations/                 ← Миграции
├── components/
│   ├── ui/                         ← Переиспользуемые UI-компоненты
│   └── forms/                      ← Формы с Server Actions
└── middleware.ts                    ← Auth middleware
```

### Поток данных

```
┌──────────────────────────────────────────────────────────────────┐
│                         СЕРВЕР                                    │
│                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     │
│  │   Middleware  │────→│    Layout    │────→│     Page     │     │
│  │  (auth check) │     │  (auth data) │     │  (DB query)  │     │
│  └──────────────┘     └──────────────┘     └──────┬───────┘     │
│                                                     │             │
│                              ┌──────────────────────┤             │
│                              ▼                      ▼             │
│                     ┌──────────────┐     ┌──────────────┐        │
│                     │ Server       │     │   Server     │        │
│                     │ Component    │     │   Action     │        │
│                     │ (read DB)    │     │  (write DB)  │        │
│                     └──────┬───────┘     └──────┬───────┘        │
│                            │                     │                │
│                            └──────────┬──────────┘                │
│                                       ▼                           │
│                            ┌──────────────────┐                   │
│                            │   Prisma / ORM   │                   │
│                            └────────┬─────────┘                   │
│                                     │                             │
├─────────────────────────────────────┼─────────────────────────────┤
│                                     ▼                             │
│                            ┌──────────────────┐                   │
│                            │   PostgreSQL     │                   │
│                            └──────────────────┘                   │
│                                                                   │
├───────────────────────────────────────────────────────────────────┤
│                         КЛИЕНТ (Браузер)                          │
│                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     │
│  │  Client       │     │  Client       │     │  Client       │     │
│  │  Component    │     │  Component    │     │  Component    │     │
│  │  (forms)      │────→│  (state)      │     │  (UI)         │     │
│  └──────────────┘     └──────────────┘     └──────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### Подключение к БД (singleton-паттерн)

```tsx
// lib/db.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

// В dev-режиме Hot Module Reload создаёт новые экземпляры,
// поэтому используем globalThis для singleton
export const db = globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = db
}
```

## Плюсы подхода

### 1. Скорость разработки

```
Традиционный подход:
  1. Дизайним API-контракт (OpenAPI/Swagger)
  2. Бэкенд пишет эндпоинты
  3. Фронтенд ждёт, пока бэкенд будет готов
  4. Фронтенд пишет fetch/axios-обёртки
  5. Интеграционное тестирование
  6. Исправление несоответствий

Монолитный Next.js:
  1. Пишешь Server Component / Server Action
  2. Готово
```

Нет контрактов, нет сериализации/десериализации, нет версионирования API. Данные — это просто JavaScript-объекты.

### 2. End-to-end типобезопасность

```tsx
// prisma/schema.prisma
model User {
  id    String @id @default(cuid())
  name  String
  email String @unique
  posts Post[]
}

// Server Component — типобезопасность от БД до UI
export default async function UsersPage() {
  const users = await db.user.findMany({ include: { posts: true } })
  //    ^? User[] (автоматический тип из Prisma)

  return users.map(user => (
    <div key={user.id}>
      {/* user.name — string, user.posts — Post[] — всё типизировано */}
      {user.name} — {user.posts.length}
    </div>
  ))
}
```

Типы генерируются из схемы БД и проходят через весь стек без ручного описания интерфейсов.

### 3. Меньше кода и Boilerplate

```
Традиционный подход (REST API):

  Бэкенд:
    - Route handler
    - Controller
    - Service layer
    - Repository / DAO
    - DTO mapping
    - Validation
    - Serialization

  Фронтенд:
    - API client (axios/fetch)
    - Type definitions (дублирование!)
    - Error handling для HTTP
    - Loading states
    - Caching layer

Монолитный Next.js:

    - Server Component / Server Action
    - Prisma query (валидация через Zod, опционально)
```

### 4. Нет проблем с синхронизацией данных

В традиционной архитектуре:
- API вернул данные в формате A, а фронтенд ожидает формат B
- Изменили поле в БД — нужно обновить DTO, API-ответ, фронтенд-типы
- Версионирование API (v1, v2, ...)

В монолите:
- Изменил схему БД → Prisma сгенерировал новые типы → TypeScript покажет ошибки в компонентах
- Один источник истины — схема БД

### 5. Производительность

```
Традиционный подход:
  Клиент → HTTP → API-сервер → БД → API-сервер → HTTP → Клиент
  (3 сетевых хопа + сериализация/десериализация на каждом)

Монолитный Next.js:
  Сервер → БД → Сервер → (сериализованный HTML/RSC payload) → Клиент
  (1 сетевой хоп, нет промежуточной сериализации)
```

Серверные компоненты рендерятся на сервере. Данные из БД попадают прямо в React-дерево без промежуточных HTTP-запросов.

### 6. Упрощённый деплой

Один проект — один деплой. Нет необходимости координировать деплой API и фронтенда.

```
Традиционно:
  deploy backend → deploy frontend → проверить совместимость

Монолит:
  deploy next-app → готово
```

## Минусы подхода

### 1. Привязка к одному фреймворку

```
Проблема:
  Весь бизнес-логика в Next.js → нельзя переиспользовать с:
    - Мобильным приложением (React Native, Flutter, Swift)
    - Другим фронтендом (Vue, Svelte)
    - Внешними интеграциями
    - Микросервисами
```

Если появится мобильное приложение — придётся выносить логику в отдельный API.

### 2. Масштабируемость

```
Монолит:
  ┌──────────────────────────────────────┐
  │         Один Next.js сервер          │
  │  (UI + бизнес-логика + работа с БД)  │
  │  Масштабируется только целиком       │
  └──────────────────────────────────────┘

Микросервисы:
  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │  Auth   │  │  Users  │  │ Billing │
  │ Service │  │ Service │  │ Service │
  │ (scale) │  │ (scale) │  │ (scale) │
  └─────────┘  └─────────┘  └─────────┘
  Каждый сервис масштабируется независимо
```

Монолит масштабируется только целиком. Если нагрузка растёт только на одну часть приложения — всё равно нужно масштабировать весь сервер.

### 3. Безопасность — зона повышенной ответственности

```tsx
// ОПАСНО: нет валидации, SQL injection через Prisma (невозможно),
// но XSS и логика — реальны

// Плохо — нет проверки прав
export default async function AdminPage() {
  // Этот компонент доступен всем!
  const users = await db.user.findMany()
  return <UsersList users={users} />
}

// Хорошо — проверка в middleware + в компоненте
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'

export default async function AdminPage() {
  const session = await auth()
  if (!session || session.user.role !== 'admin') {
    redirect('/')
  }

  const users = await db.user.findMany()
  return <UsersList users={users} />
}
```

В монолите **нет отдельного API-слоя с централизованной авторизацией**. Каждый Server Component и Server Action сам отвечает за проверку прав. Легко забыть проверку — и данные утекут.

### 4. Сложность тестирования

```
Традиционный API:
  - API легко тестировать: отправил HTTP-запрос → получил JSON-ответ
  - Можно тестировать фронтенд и бэкенд отдельно
  - Mock API — стандартная практика

Монолит:
  - Server Components harder to test (нужен контекст рендеринга)
  - Server Actions требуют эмуляции серверного окружения
  - Нужно мокировать БД (in-memory SQLite или mock Prisma)
  - E2E-тесты становятся основными
```

### 5. Раздувание бандла и сложности с зависимостями

```tsx
// Проблема: серверные зависимости могут случайно попасть в клиентский бандл

// lib/heavy-server-stuff.ts
import { PrismaClient } from '@prisma/client'  // 2MB+
import { generatePDF } from 'pdfkit'             // серверная библиотека

export const db = new PrismaClient()

// Если случайно импортировать это в 'use client' компонент:
'use client'
import { db } from '@/lib/heavy-server-stuff'  // ← ОШИБКА
// Prisma и pdfkit попадут в клиентский бандл
```

Next.js предупреждает о таких случаях, но не всегда может их предотвратить.

### 6. Нет чёткого разделения ответственности

```
В команде из 10 разработчиков:
  - 5 frontend
  - 5 backend

Традиционно: каждый работает в своей зоне.
Монолит: все работают в одном коде, конфликты при merge,
         разные уровни экспертизы, разный стиль кода.
```

### 7. Ограничения серверless-окружения

```
Vercel (serverless):
  - Cold start: каждое подключение к БД = задержка
  - Connection pool: лимит подключений к БД
  - Таймауты: длительные запросы могут быть оборваны
  - Регион: серверless функции могут быть далеко от БД
```

## Подводные камни для frontend-разработчика

### 1. Понимание модели данных

```
Frontend-разработчик привык к плоским JSON-объектам:
  { id: 1, name: "John", posts: [{ title: "Hello" }] }

В БД данные нормализованы:
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  users   │────→│  posts   │────→│ comments │
  │          │     │          │     │          │
  │ id       │     │ id       │     │ id       │
  │ name     │     │ user_id  │     │ post_id  │
  │ email    │     │ title    │     │ body     │
  └──────────┘     │ body     │     │ user_id  │
                   └──────────┘     └──────────┘

Проблема: N+1 запросы
  SELECT * FROM posts;           -- 1 запрос
  SELECT * FROM users WHERE id=? -- N запросов (по одному на каждый пост)
```

**Решение:** изучать `include` / `select` в Prisma, понимать JOIN-ы, использовать `withRelated`.

### 2. Миграции и схема БД

```
Frontend-разработчик может не знать:
  - Что такое миграции и зачем они нужны
  - Как безопасно изменить структуру БД в продакшене
  - Что такое rollback миграции
  - Чем отличается `prisma migrate dev` от `prisma migrate deploy`
  - Как работать с existing database

Опасность:
  - Случайное удаление данных при миграции
  - Потеря данных при `prisma migrate reset`
  - Неправильные типы данных (String vs Int vs UUID)
  - Отсутствие индексов → медленные запросы на больших данных
```

### 3. Транзакции

```tsx
// Проблема: деньги списались, но заказ не создался
// Без транзакции — данные в несогласованном состоянии

// Плохо — два отдельных запроса
await db.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } })
await db.order.create({ data: { userId: 1, total: 100 } })
// Если второй запрос упадёт — деньги спишутся, но заказ не создастся

// Хорошо — транзакция
await db.$transaction(async (tx) => {
  await tx.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } })
  await tx.order.create({ data: { userId: 1, total: 100 } })
})
// Если что-то упадёт — всё откатится
```

### 4. Индексы и производительность запросов

```
Frontend-разработчик пишет:
  const users = await db.user.findMany({
    where: { email: { contains: 'gmail' } }
  })

На 10 пользователях — работает мгновенно.
На 1 000 000 пользователях — запрос виснет на 10+ секунд.

Причина: нет индекса на поле email.
Решение: понимать EXPLAIN ANALYZE, знать про B-tree индексы,
         понимать разницу между Seq Scan и Index Scan.
```

### 5. Connection pooling

```
Проблема serverless (Vercel):
  Каждая serverless function instance = новое подключение к БД
  При 100 одновременных запросов = 100 подключений
  PostgreSQL по умолчанию поддерживает ~100 подключений

Решения:
  - PgBouncer / Supavisor (connection pooler)
  - Prisma Accelerate (управляемый pool)
  - Neon / PlanetScale (встроенный pool)
  - Настройка max connections в PrismaClient
```

```tsx
// lib/db.ts — правильная настройка для serverless
export const db = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  // Ограничение подключений
  // По умолчанию Prisma создаёт пул из ~4 соединений
})
```

### 6. Безопасность данных

```
Frontend-разработчик привык, что API фильтрует данные.
В монолите нужно САМОМУ следить за:

  1. Авторизация — кто может читать/писать данные
  2. Валидация — входные данные должны быть проверены
  3. Sanitization — защита от XSS при выводе
  4. Rate limiting — защита от брутфорса
  5. SQL injection — ORM защищает, но raw queries нет
  6. Чувствительные данные — пароли, токены, PII

// Плохо — возвращаем всё, включая хеш пароля
const user = await db.user.findUnique({ where: { id: userId } })
return user // { id, name, email, passwordHash, ... }

// Хорошо — выбираем только нужные поля
const user = await db.user.findUnique({
  where: { id: userId },
  select: { id: true, name: true, email: true }
})
```

### 7. Ошибки и их обработка

```tsx
// Frontend-разработчик привык к HTTP-статусам: 404, 500, 422
// В монолите — ошибки БД, которые нужно правильно обрабатывать

import { Prisma } from '@prisma/client'

try {
  await db.user.create({ data: { email: 'existing@email.com' } })
} catch (error) {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    if (error.code === 'P2002') {
      // Unique constraint violation — email уже занят
      return { error: 'Email уже используется' }
    }
    if (error.code === 'P2025') {
      // Record not found
      return { error: 'Запись не найдена' }
    }
  }
  // Неизвестная ошибка
  return { error: 'Внутренняя ошибка сервера' }
}
```

### 8. Кеширование и revalidation

```tsx
// Проблема: данные закэшировались и не обновляются

// Серверные компоненты кэшируются по умолчанию в Next.js
// После мутации через Server Action нужно явно инвалидировать кэш

import { revalidatePath, revalidateTag } from 'next/cache'

export async function updateUser(formData: FormData) {
  await db.user.update({ ... })

  // Обязательно! Иначе пользователи увидят старые данные
  revalidatePath('/users')       // инвалидировать страницу
  revalidateTag('users')         // или по тегу
}
```

### 9. Race conditions

```tsx
// Проблема: два одновременных запроса
// Пользователь дважды нажал "Купить"

// Плохо — read-modify-write без блокировки
const product = await db.product.findUnique({ where: { id: productId } })
if (product.stock > 0) {
  await db.product.update({
    where: { id: productId },
    data: { stock: product.stock - 1 }  // race condition!
  })
}

// Хорошо — атомарная операция
await db.product.updateMany({
  where: { id: productId, stock: { gt: 0 } },
  data: { stock: { decrement: 1 } }
})

// Или с транзакцией и блокировкой
await db.$transaction(async (tx) => {
  const product = await tx.product.findUniqueOrThrow({
    where: { id: productId },
    // ... для некоторых БД можно использовать FOR UPDATE
  })
  if (product.stock > 0) {
    await tx.product.update({
      where: { id: productId },
      data: { stock: { decrement: 1 } }
    })
  }
})
```

## Базы данных в монолитном Next.js

### Реляционные БД (основной выбор)

| БД | Использование | Особенности |
|----|---------------|-------------|
| **PostgreSQL** | Самый популярный выбор | JSONB, полнотекстовый поиск, расширения (PostGIS), лучшая поддержка в экосистеме |
| **MySQL** | Второй по популярности | Простота, широкий хостинг, хорошо знаком многим |
| **SQLite** | Разработка, малые проекты, edge | Встроена в Turso, libSQL, работает на edge |
| **CockroachDB** | Распределённые системы | PostgreSQL-совместимая, глобальная репликация |

### NoSQL БД

| БД | Использование | Особенности |
|----|---------------|-------------|
| **MongoDB** | Гибкая схема, документы | Prisma поддерживает, но менее удобно чем реляционные |
| **Redis** | Кэш, сессии, очереди | Используется как дополнение к основной БД |
| **DynamoDB** | AWS-экосистема | Serverless-friendly, но сложный API |

### Serverless-ориентированные БД

| БД | Особенности |
|----|-------------|
| **Neon** | Serverless PostgreSQL, branching, autoscaling |
| **PlanetScale** | Serverless MySQL, branching, schema changes без downtime |
| **Turso** | Edge SQLite (libSQL), низкая задержка |
| **Supabase** | PostgreSQL + Auth + Realtime + Storage (BaaS) |
| **Convex** | Realtime БД с транзакциями, TypeScript-first |
| **Upstash** | Serverless Redis + Kafka + Vector |

### Рекомендуемый стек для старта

```
Для большинства проектов:
  PostgreSQL + Prisma + Next.js

Для serverless-first:
  Neon (PostgreSQL) + Prisma + Next.js на Vercel

Для edge-first:
  Turso (SQLite) + drizzle-orm + Next.js

Для быстрого MVP:
  Supabase + Next.js (Auth + DB + Storage из коробки)
```

### Почему PostgreSQL — стандарт выбора

```
1. Лучшая поддержка в ORM (Prisma, Drizzle, TypeORM)
2. JSONB — хранит JSON прямо в реляционной БД (гибкость NoSQL)
3. Полнотекстовый поиск (tsvector) — не нужен Elasticsearch для начала
4. PostGIS — геоданные
5. Массивы, enum, составные типы
6. Massive community и документация
7. Бесплатные тарифы: Supabase, Neon, Railway
```

## ORM и инструменты работы с БД

### Prisma (самый популярный)

```tsx
// prisma/schema.prisma
model User {
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
}

// Использование
const users = await db.user.findMany({
  where: { posts: { some: { published: true } } },
  include: { posts: { where: { published: true } } },
  orderBy: { createdAt: 'desc' },
  take: 10,
})
```

**Плюсы:** отличная типобезопасность, миграции, GUI (Prisma Studio), большое сообщество.
**Минусы:** runtime overhead, не поддерживает все фичи БД, медленнее raw SQL.

### Drizzle ORM (набирающая популярность)

```tsx
// lib/schema.ts
import { pgTable, serial, text, boolean } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
})

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  published: boolean('published').default(false),
  authorId: serial('author_id').references(() => users.id),
})

// Использование — SQL-like синтаксис
import { eq, and } from 'drizzle-orm'

const result = await db
  .select()
  .from(users)
  .leftJoin(posts, eq(users.id, posts.authorId))
  .where(eq(users.name, 'John'))
```

**Плюсы:** легковесная, SQL-like API, edge-compatible, нет runtime overhead.
**Минусы:** менее зрелая, меньше инструментов, миграции менее удобные.

### Kysely (type-safe query builder)

```tsx
// Строитель запросов с типобезопасностью, без ORM-магии
const users = await db
  .selectFrom('users')
  .innerJoin('posts', 'posts.author_id', 'users.id')
  .select(['users.name', 'posts.title'])
  .where('posts.published', '=', true)
  .execute()
```

### Сравнение ORM

| Характеристика | Prisma | Drizzle | Kysely |
|----------------|--------|---------|--------|
| Типобезопасность | Отличная | Отличная | Отличная |
| Синтаксис | Декларативный | SQL-like | SQL-like |
| Миграции | Встроенные | Встроенные | Нет (использует внешние) |
| Edge-compatible | Ограниченно | Да | Да |
| Производительность | Средняя | Высокая | Высокая |
| Зрелость | Высокая | Средняя | Средняя |
| Community | Огромное | Растущее | Нишевое |

## Необходимый уровень знаний для коммерческой разработки

### Минимальный уровень (Junior Fullstack)

```
Базы данных:
  ✅ Понимание реляционной модели (таблицы, связи, ключи)
  ✅ SQL на уровне SELECT, INSERT, UPDATE, DELETE, JOIN
  ✅ Понимание индексов (зачем нужны, когда создавать)
  ✅ Нормализация (1NF, 2NF, 3NF — хотя бы концептуально)
  ✅ Умение спроектировать простую схему (users, posts, comments)

ORM:
  ✅ Prisma: CRUD операции, связи (one-to-many, many-to-many)
  ✅ Понимание include/select для eager loading
  ✅ Базовые миграции (migrate dev, migrate deploy)

Безопасность:
  ✅ Валидация входных данных (Zod)
  ✅ Авторизация (проверка прав в каждом Server Component/Action)
  ✅ Хранение секретов (env variables, не в коде)

Деплой:
  ✅ Настройка DATABASE_URL
  ✅ Запуск миграций при деплое
  ✅ Понимание connection pooling
```

### Средний уровень (Middle Fullstack)

```
Базы данных:
  ✅ Транзакции и их гарантии (ACID)
  ✅ Индексы: составные, уникальные, частичные
  ✅ EXPLAIN ANALYZE — чтение планов запросов
  ✅ Оптимизация медленных запросов
  ✅ Connection pooling (PgBouncer, Prisma Accelerate)
  ✅ Бэкапы и восстановление

Архитектура:
  ✅ Service layer — вынос бизнес-логики из компонентов
  ✅ Паттерн Repository для абстракции БД
  ✅ Валидация через Zod на границах
  ✅ Обработка ошибок БД (constraint violations, deadlock)
  ✅ Кеширование (Redis, in-memory)
  ✅ Rate limiting

Безопасность:
  ✅ Row-level security (RLS)
  ✅ Защита от CSRF
  ✅ Rate limiting на Server Actions
  ✅ Аудит и логирование
```

### Продвинутый уровень (Senior Fullstack)

```
Базы данных:
  ✅ Проектирование сложных схем (наследование, полиморфные связи)
  ✅ Партиционирование таблиц
  ✅ Репликация и шардирование
  ✅ Distributed transactions (Saga pattern)
  ✅ Event sourcing и CQRS
  ✅ Realtime (WebSocket, database triggers)

Масштабирование:
  ✅ Read replicas для разделения чтения/записи
  ✅ Кэширование на нескольких уровнях
  ✅ Background jobs (Bull, Inngest)
  ✅ Очереди сообщений
  ✅ Мониторинг и алертинг

Архитектура:
  ✅ Модульная архитектура (domain-driven design)
  ✅ Multi-tenancy
  ✅ Feature flags
  ✅ A/B тестирование
```

## Навыки для решения бизнес-задач

### 1. Проектирование модели данных

```
Бизнес-задача: «Интернет-магазин с корзиной, заказами и оплатой»

Нужно спроектировать:
  ┌──────────┐    ┌──────────────┐    ┌──────────────┐
  │  users   │    │   orders     │    │   payments   │
  │──────────│    │──────────────│    │──────────────│
  │ id       │←───│ user_id      │    │ order_id     │
  │ name     │    │ total        │←───│ amount       │
  │ email    │    │ status       │    │ status       │
  └──────────┘    │ created_at   │    │ provider     │
       │          └──────┬───────┘    │ paid_at      │
       │                 │            └──────────────┘
       ▼                 ▼
  ┌──────────┐    ┌──────────────┐
  │  carts   │    │ order_items  │    ┌──────────────┐
  │──────────│    │──────────────│    │  products    │
  │ user_id  │    │ order_id     │←───│──────────────│
  │ product_ │    │ product_id   │    │ id           │
  │  id      │    │ quantity     │    │ name         │
  │ quantity │    │ price        │    │ price        │
  └──────────┘    └──────────────┘    │ stock        │
                                      └──────────────┘
```

**Навык:** умение перевести бизнес-требования в схему БД.

### 2. Валидация и бизнес-правила

```tsx
// Бизнес-правило: скидка 10% при заказе от 1000р
// Бизнес-правило: бесплатная доставка от 3000р
// Бизнес-правило: нельзя заказать больше, чем на складе

import { z } from 'zod'

const orderSchema = z.object({
  items: z.array(z.object({
    productId: z.string(),
    quantity: z.number().int().positive(),
  })).min(1),
})

export async function createOrder(input: unknown) {
  const { items } = orderSchema.parse(input)

  return await db.$transaction(async (tx) => {
    // Проверка наличия и цен
    const products = await tx.product.findMany({
      where: { id: { in: items.map(i => i.productId) } }
    })

    // Проверка stock
    for (const item of items) {
      const product = products.find(p => p.id === item.productId)
      if (!product || product.stock < item.quantity) {
        throw new Error(`Недостаточно товара: ${product?.name}`)
      }
    }

    // Расчёт суммы
    const total = items.reduce((sum, item) => {
      const product = products.find(p => p.id === item.productId)!
      return sum + product.price * item.quantity
    }, 0)

    // Скидки
    const discount = total >= 1000 ? total * 0.1 : 0
    const shipping = total >= 3000 ? 0 : 300

    // Создание заказа
    const order = await tx.order.create({
      data: {
        userId: session.user.id,
        total: total - discount + shipping,
        discount,
        shipping,
        items: {
          create: items.map(item => ({
            productId: item.productId,
            quantity: item.quantity,
            price: products.find(p => p.id === item.productId)!.price,
          }))
        }
      }
    })

    // Уменьшение stock
    for (const item of items) {
      await tx.product.update({
        where: { id: item.productId },
        data: { stock: { decrement: item.quantity } }
      })
    }

    return order
  })
}
```

### 3. Аутентификация и авторизация

```
Необходимые навыки:
  ✅ NextAuth.js / Auth.js / Lucia — выбор и настройка
  ✅ OAuth 2.0 (Google, GitHub, etc.)
  ✅ JWT vs Session-based auth
  ✅ Role-based access control (RBAC)
  ✅ Middleware для защиты маршрутов
  ✅ Server-side session validation
```

```tsx
// middleware.ts — защита маршрутов
import { auth } from '@/lib/auth'
import { NextResponse } from 'next/server'

export default auth((req) => {
  const isLoggedIn = !!req.auth
  const isOnLoginPage = req.nextUrl.pathname === '/login'
  const isAdminRoute = req.nextUrl.pathname.startsWith('/admin')

  if (!isLoggedIn && !isOnLoginPage) {
    return NextResponse.redirect(new URL('/login', req.url))
  }

  if (isAdminRoute && req.auth?.user?.role !== 'admin') {
    return NextResponse.redirect(new URL('/', req.url))
  }

  return NextResponse.next()
})

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

### 4. Обработка файлов и загрузка

```tsx
// Бизнес-задача: загрузка аватарок, документов, изображений
// Нужен S3-compatible storage (AWS S3, Cloudflare R2, Supabase Storage)

import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'

const s3 = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
})

export async function uploadAvatar(formData: FormData) {
  const file = formData.get('avatar') as File
  const buffer = Buffer.from(await file.arrayBuffer())

  const key = `avatars/${crypto.randomUUID()}-${file.name}`

  await s3.send(new PutObjectCommand({
    Bucket: process.env.AWS_BUCKET!,
    Key: key,
    Body: buffer,
    ContentType: file.type,
  }))

  await db.user.update({
    where: { id: session.user.id },
    data: { avatarUrl: `${process.env.CDN_URL}/${key}` }
  })
}
```

### 5. Фоновые задачи

```
Бизнес-задачи, требующие фоновой обработки:
  - Отправка email (регистрация, восстановление пароля)
  - Обработка платежей (webhooks от Stripe)
  - Генерация отчётов
  - Очистка старых данных
  - Resize изображений
  - Отправка push-уведомлений

Решения:
  - Inngest (serverless-friendly, TypeScript-first)
  - BullMQ + Redis (классический подход)
  - Trigger.dev (облачный)
  - Cron jobs (Vercel Cron, node-cron)
```

```tsx
// Inngest — пример фоновой задачи
import { Inngest } from 'inngest'

const inngest = new Inngest({ id: 'my-app' })

export const sendWelcomeEmail = inngest.createFunction(
  { id: 'send-welcome-email' },
  { event: 'user/registered' },
  async ({ event, step }) => {
    await step.run('send-email', async () => {
      await resend.emails.send({
        from: 'welcome@myapp.com',
        to: event.data.email,
        subject: 'Добро пожаловать!',
        html: `<h1>Привет, ${event.data.name}!</h1>`,
      })
    })
  }
)
```

### 6. Мониторинг и отладка

```
Необходимые инструменты:
  ✅ Sentry — отслеживание ошибок (frontend + backend)
  ✅ Vercel Analytics / LogDrain — логи серверных функций
  ✅ Prisma Studio — GUI для просмотра данных
  ✅ pgAdmin / DBeaver — прямой доступ к БД
  ✅ APM (Application Performance Monitoring) — замеры скорости запросов

Что нужно мониторить:
  - Медленные запросы к БД (>100ms)
  - Ошибки Prisma (constraint violations, connection errors)
  - Утечки подключений к БД
  - Использование памяти serverless functions
  - Ошибки авторизации
```

## Паттерны и примеры

### Service Layer (вынос логики из компонентов)

```tsx
// ❌ Плохо — бизнес-логика в компоненте
export default async function ProductsPage() {
  const products = await db.product.findMany()
  const filtered = products.filter(p => p.price > 100 && p.stock > 0)
  const sorted = filtered.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
  // ... компонент загромождён логикой
}

// ✅ Хорошо — service layer
// lib/services/products.ts
export async function getAvailableProducts() {
  return db.product.findMany({
    where: { price: { gt: 100 }, stock: { gt: 0 } },
    orderBy: { createdAt: 'desc' },
  })
}

// app/products/page.tsx — чистый компонент
import { getAvailableProducts } from '@/lib/services/products'

export default async function ProductsPage() {
  const products = await getAvailableProducts()
  return <ProductsList products={products} />
}
```

### Оптимистичные обновления

```tsx
'use client'

import { useOptimistic } from 'react'
import { toggleTodo } from '@/app/actions/todos'

export function TodoItem({ todo }: { todo: Todo }) {
  const [optimisticTodo, setOptimistic] = useOptimistic(
    todo,
    (state, newCompleted: boolean) => ({ ...state, completed: newCompleted })
  )

  async function handleToggle() {
    const newCompleted = !optimisticTodo.completed
    setOptimistic(newCompleted)
    await toggleTodo(todo.id, newCompleted)
  }

  return (
    <label>
      <input
        type="checkbox"
        checked={optimisticTodo.completed}
        onChange={handleToggle}
      />
      {optimisticTodo.title}
    </label>
  )
}
```

### Пагинация

```tsx
// app/posts/page.tsx
import { db } from '@/lib/db'
import { Pagination } from '@/components/pagination'

const PAGE_SIZE = 20

export default async function PostsPage({
  searchParams
}: {
  searchParams: { page?: string }
}) {
  const page = Number(searchParams.page ?? 1)

  const [posts, totalCount] = await Promise.all([
    db.post.findMany({
      skip: (page - 1) * PAGE_SIZE,
      take: PAGE_SIZE,
      orderBy: { createdAt: 'desc' },
      include: { author: { select: { name: true } } },
    }),
    db.post.count(),
  ])

  const totalPages = Math.ceil(totalCount / PAGE_SIZE)

  return (
    <>
      <PostsList posts={posts} />
      <Pagination currentPage={page} totalPages={totalPages} />
    </>
  )
}
```

### Cursor-based пагинация (для больших таблиц)

```tsx
// Offset-пагинация медленная на больших данных:
// SKIP 1000000 — БД должна прочитать и пропустить миллион строк

// Cursor-based — мгновенная пагинация:
export async function getPosts(cursor?: string, limit = 20) {
  return db.post.findMany({
    take: limit + 1, // +1 чтобы узнать есть ли следующая страница
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1,
    }),
    orderBy: { id: 'asc' },
    include: { author: true },
  })
}
```

### Полнотекстовый поиск

```tsx
// PostgreSQL + Prisma
// Полнотекстовый поиск без Elasticsearch

// Через raw SQL (Prisma не поддерживает FTS нативно)
export async function searchPosts(query: string) {
  return db.$queryRaw<Post[]>`
    SELECT id, title, content,
           ts_rank(to_tsvector('russian', title || ' ' || content),
                   plainto_tsquery('russian', ${query})) as rank
    FROM posts
    WHERE to_tsvector('russian', title || ' ' || content)
          @@ plainto_tsquery('russian', ${query})
    ORDER BY rank DESC
    LIMIT 20
  `
}
```

### Multi-tenancy

```tsx
// Бизнес-задача: SaaS, где у каждого клиента свои данные

// Middleware — определяем tenant
export async function getTenantDb(userId: string) {
  const tenant = await db.tenant.findFirst({
    where: { members: { some: { userId } } }
  })

  if (!tenant) throw new Error('Tenant not found')

  // Все запросы фильтруются по tenantId
  return {
    posts: {
      findMany: (args?: any) => db.post.findMany({
        ...args,
        where: { ...args?.where, tenantId: tenant.id }
      }),
    },
    // ...
  }
}
```

## Когда стоит, а когда не стоит

### Стоит использовать монолитный Next.js

```
✅ Стартап / MVP — скорость критична, команда 1-3 человека
✅ Внутренний инструмент — дашборд, админка, CRM
✅ Контентный сайт с динамикой — блог, маркетплейс, каталог
✅ SaaS малого/среднего масштаба
✅ Один разработчик или small fullstack team
✅ Нет мобильного приложения (или оно тоже на React Native + тот же API)
✅ Данные не выходят за рамки одного домена
```

### Не стоит использовать монолитный Next.js

```
❌ Мобильное приложение + веб — нужен отдельный API
❌ Микросервисная архитектура — каждый сервис со своей БД
❌ Публичный API для третьих сторон
❌ Команда с чётким разделением frontend/backend (10+ человек)
❌ Высокие требования к безопасности (банки, медицина) — нужен отдельный security layer
❌ Разные команды отвечают за разные части системы
❌ Необходимость масштабировать части независимо
```

### Гибридный подход (часто оптимальный)

```
┌──────────────────────────────────────────────────────────────┐
│                    Next.js (BFF + часть логики)               │
│                                                               │
│  ┌──────────────────┐   ┌──────────────────┐                │
│  │ Server Component  │   │  Server Action   │                │
│  │ (прямой запрос   │   │  (прямая запись  │                │
│  │  к БД для UI)    │   │   в БД для форм) │                │
│  └────────┬─────────┘   └────────┬─────────┘                │
│           │                       │                           │
│           │    ┌──────────────────┤                           │
│           ▼    ▼                  ▼                           │
│  ┌──────────────────┐   ┌──────────────────┐                │
│  │   Прямой запрос  │   │  HTTP запрос к   │                │
│  │   к БД (быстрые  │   │  внешнему API    │                │
│  │   CRUD операции) │   │  (микросервисы)  │                │
│  └──────────────────┘   └──────────────────┘                │
└──────────────────────────────────────────────────────────────┘
```

## Сравнение с традиционной архитектурой

| Критерий | Монолитный Next.js | Традиционная (Frontend + API) |
|----------|-------------------|-------------------------------|
| Скорость разработки | Очень высокая | Средняя (контракты, координация) |
| Типобезопасность | End-to-end из коробки | Нужен codegen (OpenAPI → TypeScript) |
| Производительность | Выше (нет лишних хопов) | Ниже (сериализация, сеть) |
| Масштабируемость | Ограниченная | Высокая (каждый сервис отдельно) |
| Переиспользование API | Только в рамках Next.js | Любые клиенты (mobile, web, 3rd party) |
| Команда | 1-5 fullstack | Frontend + Backend (разные навыки) |
| Деплой | Один проект | Два+ проекта, координация |
| Тестирование | E2E + интеграционные | Unit + интеграционные для каждого слоя |
| Безопасность | Децентрализованная (каждый компонент) | Централизованная (API gateway) |
| Стоимость | Ниже (меньше инфраструктуры) | Выше (отдельные серверы, API gateway) |
| Vendor lock-in | Средний (Vercel + БД) | Ниже (API абстрагирует) |
| Learning curve | Нужно знать БД + ORM + безопасность | Каждый учит свою область |

## Итого

Монолитный Next.js с прямым обращением к БД — это **зрелый, коммерчески жизнеспособный подход**, который идеально подходит для стартапов, внутренних инструментов и SaaS-приложений. Он не является экспериментом — это основной способ разработки для тысяч компаний.

**Для frontend-разработчика** переход к этому подходу означает необходимость освоить:
1. SQL и реляционные БД (PostgreSQL)
2. ORM (Prisma или Drizzle)
3. Основы безопасности (авторизация, валидация, защита данных)
4. Транзакции и обработку ошибок БД
5. Основы производительности (индексы, connection pooling)

Это не требует становления полноценным backend-разработчиком, но требует понимания основ работы с данными на уровне уверенного middle fullstack-разработчика.
