# Глубокое погружение в Data Fetching в Next.js

Загрузка данных в Next.js App Router кардинально отличается от традиционных React-приложений. Серверные компоненты позволяют загружать данные напрямую в компоненте через `async/await`, без `useEffect`, `useState` и клиентских хуков. Этот документ разбирает все подходы, паттерны и подводные камни.

## Содержание

- [Архитектура данных в Next.js](#архитектура-данных-в-nextjs)
- [Data Fetching на сервере](#data-fetching-на-сервере)
- [Кэширование данных](#кэширование-данных)
- [Параллельная загрузка данных](#параллельная-загрузка-данных)
- [Последовательная загрузка данных](#последовательная-загрузка-данных)
- [Data Fetching на клиенте](#data-fetching-на-клиенте)
- [Server Actions и мутации](#server-actions-и-мутации)
- [Route Handlers](#route-handlers)
- [Streaming и Suspense](#streaming-и-suspense)
- [Практические паттерны](#практические-паттерны)
- [Best Practices](#best-practices)
- [Антипаттерны](#антипаттерны)
- [Сравнение с Nuxt.js](#сравнение-с-nuxtjs)

## Архитектура данных в Next.js

### Серверные vs Клиентские компоненты

В Next.js App Router все компоненты — **серверные по умолчанию**. Это фундаментальное отличие от Pages Router и традиционного React.

```
┌─────────────────────────────────────────────────────┐
│                    Сервер (Node.js)                  │
│                                                      │
│  ┌──────────────────┐   ┌──────────────────┐        │
│  │ Server Component  │   │ Server Component  │        │
│  │ (async/await)    │──→│ (fetch, DB query) │        │
│  └──────────────────┘   └──────────────────┘        │
│           │                                          │
│           │ RSC Payload (сериализованный React tree) │
│           ▼                                          │
├─────────────────────────────────────────────────────┤
│                    Клиент (Браузер)                   │
│                                                      │
│  ┌──────────────────┐   ┌──────────────────┐        │
│  │ Client Component  │   │ Client Component  │        │
│  │ ('use client')   │   │ (useState, etc)  │        │
│  └──────────────────┘   └──────────────────┘        │
└─────────────────────────────────────────────────────┘
```

### Где можно загружать данные

| Место | Где выполняется | Когда использовать |
|-------|-----------------|-------------------|
| Server Component | Сервер при рендере | Основная загрузка данных |
| Client Component | Браузер | Интерактивность, пользовательские события |
| Route Handler | Сервер по запросу | API endpoints, webhooks |
| Server Action | Сервер при мутации | Form submissions, изменения данных |
| Middleware | Edge при запросе | Аутентификация, редиректы |

> **Nuxt:** В Nuxt используется другой подход — **Composition API**. Вместо Server Components есть `useFetch` и `useAsyncData`, которые работают и на сервере (SSR), и на клиенте (гидратация). Данные загружаются в `setup()` и автоматически сериализуются в HTML, затем восстанавливаются на клиенте без повторного запроса. Нет разделения на `'use client'` и серверные компоненты — все компоненты универсальны.

## Data Fetching на сервере

### Базовый async/await в Server Components

Серверные компоненты могут быть `async` функциями. Это позволяет использовать `await` напрямую:

```tsx
// app/posts/page.tsx — Server Component (по умолчанию)
export default async function PostsPage() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json())

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

**Что происходит под капотом:**

1. Next.js получает запрос на `/posts`
2. Выполняет `PostsPage` на сервере
3. `fetch` делает запрос к API
4. Компонент рендерится в HTML
5. HTML отправляется клиенту
6. Клиент получает готовый HTML без JavaScript для этого компонента

> **Nuxt:** Аналогичный результат достигается через `useFetch` / `useAsyncData` в `<script setup>`. Данные загружаются на сервере при SSR, сериализуются в `<script>` тег (payload), и на клиенте восстанавливаются из payload без повторного запроса. Синтаксис: `const { data } = await useFetch('/api/posts')`.

### Fetch с параметрами

```tsx
// app/posts/[category]/page.tsx
export default async function CategoryPosts({
  params
}: {
  params: { category: string }
}) {
  const posts = await fetch(
    `https://api.example.com/posts?category=${params.category}`
  ).then(r => r.json())

  return (
    <div>
      <h1>Категория: {params.category}</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

### Fetch с searchParams

```tsx
// app/search/page.tsx
export default async function SearchPage({
  searchParams
}: {
  searchParams: { q?: string; page?: string }
}) {
  const query = searchParams.q || ''
  const page = Number(searchParams.page) || 1

  const results = query
    ? await fetch(`https://api.example.com/search?q=${query}&page=${page}`)
        .then(r => r.json())
    : []

  return (
    <div>
      <h1>Результаты поиска: {query}</h1>
      {results.map(item => (
        <div key={item.id}>{item.title}</div>
      ))}
    </div>
  )
}
```

### Загрузка из базы данных

Серверные компоненты имеют прямой доступ к базе данных:

```tsx
// app/users/page.tsx
import { db } from '@/lib/db'

export default async function UsersPage() {
  const users = await db.user.findMany({
    orderBy: { createdAt: 'desc' },
    take: 20,
  })

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name} — {user.email}</li>
      ))}
    </ul>
  )
}
```

### Обработка ошибок при fetch

```tsx
// app/products/page.tsx
export default async function ProductsPage() {
  const res = await fetch('https://api.example.com/products')

  if (!res.ok) {
    // Эта ошибка будет перехвачена error.tsx
    throw new Error('Не удалось загрузить товары')
  }

  const products = await res.json()

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  )
}
```

### notFound() для отсутствующих данных

```tsx
// app/products/[id]/page.tsx
import { notFound } from 'next/navigation'

export default async function ProductPage({
  params
}: {
  params: { id: string }
}) {
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(r => r.json())

  if (!product) {
    notFound()
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </div>
  )
}
```

### Можно ли обойтись без REST API?

Да, в большинстве случаев можно. Серверные компоненты имеют прямой доступ к БД, что позволяет строить **монолитные приложения** без отдельного API-слоя.

**Когда можно обойтись без API:**
- Монолитное приложение (блог, дашборд, e-commerce, SaaS)
- Нет мобильного приложения или другого клиента
- Нет необходимости в публичном API
- Нет внешних webhook'ов

**Когда API всё-таки нужен:**
- Мобильное приложение (React Native, Flutter)
- Другой фронтенд (Vue, React на другом домене)
- Webhooks от внешних сервисов (Stripe, GitHub)
- Публичный API для третьих сторон
- Real-time (WebSocket, Server-Sent Events)
- Микросервисная архитектура

**Рекомендуемая стратегия:** начни с Server Components + Server Actions. Добавь Route Handlers, когда появится конкретная причина.

**Паттерн: data access layer** — выноси логику работы с БД в переиспользуемые функции:

```tsx
// lib/data/posts.ts
import { db } from '@/lib/db'

export async function getPosts() {
  return db.post.findMany({ orderBy: { createdAt: 'desc' } })
}

export async function getPost(id: string) {
  return db.post.findUnique({ where: { id } })
}

// app/posts/page.tsx — используем напрямую
import { getPosts } from '@/lib/data/posts'
export default async function PostsPage() {
  const posts = await getPosts()
  return <ul>...</ul>
}

// app/api/posts/route.ts — когда понадобится API
import { getPosts } from '@/lib/data/posts'
export async function GET() {
  return Response.json(await getPosts())
}
```

Так ты начинаешь без API, но когда он понадобится — выносишь ту же логику в Route Handler за минуты.

## Кэширование данных

Next.js расширяет нативный `fetch` API, добавляя многоуровневую систему кэширования. Это одно из ключевых отличий от обычного React.

> **Nuxt:** В Nuxt кэширование работает иначе. `useFetch` автоматически кэширует данные в **payload** (передаётся с SSR). На клиенте данные восстанавливаются из payload без повторного запроса. Для HTTP-кэширования используется `useFetch(url, { $fetch: { headers: { 'Cache-Control': '...' } } })` или Nitro caching rules в `nuxt.config.ts`. Нет встроенного Data Cache как в Next.js — кэширование делегируется CDN/браузеру или серверным middleware.

### Основные опции кэширования

| Опция | Поведение | Когда использовать |
|-------|-----------|-------------------|
| По умолчанию | `force-cache` — кэшируется навсегда | Статические данные |
| `cache: 'no-store'` | Без кэша | Динамические данные, данные пользователя |
| `next: { revalidate: N }` | Кэш с TTL | Данные, которые меняются редко |
| `next: { tags: [...] }` | Кэш с тегами для on-demand revalidation | Данные с зависимостями |

```tsx
// Кэшируется (по умолчанию)
const data = await fetch('https://api.example.com/data').then(r => r.json())

// Без кэширования — запрос выполняется при каждом рендере
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store'
}).then(r => r.json())

// Кэш обновляется каждые 60 секунд
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
}).then(r => r.json())
```

> **Важно:** В Next.js 15+ поведение по умолчанию изменилось. `fetch` больше **не кэшируется** по умолчанию при `build`. Для кэширования нужно явно указывать `cache: 'force-cache'` или `next: { revalidate: ... }`.

### 4 уровня кэша

В Next.js **4 уровня кэша**, каждый со своим расположением и временем жизни:

| Кэш | Где | Что | Duration | Как управлять |
|-----|-----|-----|----------|---------------|
| **Request Memoization** | Сервер | Дедупликация fetch | Один render | Автоматически |
| **Data Cache** | Сервер (диск) | fetch responses | Permanent / revalidate | `cache`, `next.revalidate`, `next.tags` |
| **Full Route Cache** | Сервер (диск) | HTML + RSC | Permanent | `dynamic`, `revalidate` |
| **Router Cache** | Клиент (браузер) | RSC payload | Session | `router.refresh()` |

Подробный разбор каждого уровня кэша, их взаимодействие и полные примеры — в отдельной статье: **[Кэширование данных в Next.js: полное руководство](./next_caching.md)**.

## Параллельная загрузка данных

### Promise.all для параллельных запросов

Когда данные не зависят друг от друга, загружайте их параллельно:

```tsx
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const [users, posts, stats] = await Promise.all([
    fetch('https://api.example.com/users').then(r => r.json()),
    fetch('https://api.example.com/posts').then(r => r.json()),
    fetch('https://api.example.com/stats').then(r => r.json()),
  ])

  return (
    <div>
      <StatsPanel stats={stats} />
      <UsersList users={users} />
      <PostsList posts={posts} />
    </div>
  )
}
```

### Параллельная загрузка с отдельными функциями

```tsx
// lib/data.ts
async function getUsers() {
  return fetch('https://api.example.com/users').then(r => r.json())
}

async function getPosts() {
  return fetch('https://api.example.com/posts').then(r => r.json())
}

async function getStats() {
  return fetch('https://api.example.com/stats').then(r => r.json())
}

// app/dashboard/page.tsx
import { getUsers, getPosts, getStats } from '@/lib/data'

export default async function DashboardPage() {
  const [users, posts, stats] = await Promise.all([
    getUsers(),
    getPosts(),
    getStats(),
  ])

  return (
    <div>
      <StatsPanel stats={stats} />
      <UsersList users={users} />
      <PostsList posts={posts} />
    </div>
  )
}
```

### Почему это важно

```
Последовательная загрузка (❌ медленно):
  getUsers()  ────────→ 200ms
  getPosts()            ────────→ 200ms
  getStats()                      ────────→ 200ms
  Итого: 600ms

Параллельная загрузка (✅ быстро):
  getUsers()  ────────→ 200ms
  getPosts()  ────────→ 200ms
  getStats()  ────────→ 200ms
  Итого: 200ms
```

## Последовательная загрузка данных

Когда данные зависят друг от друга, загружайте последовательно:

```tsx
// app/user/[id]/page.tsx
export default async function UserPage({
  params
}: {
  params: { id: string }
}) {
  // 1. Сначала загружаем пользователя
  const user = await fetch(`https://api.example.com/users/${params.id}`)
    .then(r => r.json())

  // 2. Затем загружаем посты пользователя (зависит от user)
  const posts = await fetch(`https://api.example.com/users/${user.id}/posts`)
    .then(r => r.json())

  // 3. Затем загружаем комментарии к постам (зависит от posts)
  const comments = await Promise.all(
    posts.map(post =>
      fetch(`https://api.example.com/posts/${post.id}/comments`)
        .then(r => r.json())
    )
  )

  return (
    <div>
      <h1>{user.name}</h1>
      {posts.map((post, i) => (
        <div key={post.id}>
          <h2>{post.title}</h2>
          <ul>
            {comments[i].map(comment => (
              <li key={comment.id}>{comment.text}</li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  )
}
```

### Оптимизация: Waterfall vs Pipeline

```
Waterfall (❌ каждый запрос ждёт предыдущий):
  user ──→ posts ──→ comments
  200ms   200ms     200ms = 600ms

Pipeline (✅ минимизируем зависимости):
  user ──→ posts ──→ comments
  200ms   │  200ms   │  200ms = 600ms
          │          │
  Но если posts и comments можно параллельно:
  user ──→ [posts, comments]
  200ms     200ms = 400ms
```

## Data Fetching на клиенте

### Когда использовать клиентский fetch

Клиентская загрузка данных нужна, когда:
- Данные зависят от действий пользователя
- Нужна интерактивность (пагинация, фильтрация, поиск)
- Данные обновляются в реальном времени

### Базовый useEffect + fetch

```tsx
'use client'

import { useState, useEffect } from 'react'

export default function ClientDataComponent() {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    fetch('https://api.example.com/data')
      .then(r => r.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false))
  }, [])

  if (loading) return <div>Загрузка...</div>
  if (error) return <div>Ошибка</div>

  return <div>{JSON.stringify(data)}</div>
}
```

### Клиентский fetch с зависимостями

```tsx
'use client'

import { useState, useEffect } from 'react'

export default function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([])
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    if (!query) return

    setLoading(true)
    fetch(`https://api.example.com/search?q=${query}`)
      .then(r => r.json())
      .then(setResults)
      .finally(() => setLoading(false))
  }, [query])

  return (
    <div>
      {loading ? <Spinner /> : (
        <ul>
          {results.map(item => (
            <li key={item.id}>{item.title}</li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

### Клиентский fetch с abort

```tsx
'use client'

import { useState, useEffect } from 'react'

export default function SearchComponent() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])

  useEffect(() => {
    if (!query) {
      setResults([])
      return
    }

    const controller = new AbortController()

    fetch(`https://api.example.com/search?q=${query}`, {
      signal: controller.signal
    })
      .then(r => r.json())
      .then(setResults)
      .catch(err => {
        if (err.name !== 'AbortError') throw err
      })

    return () => controller.abort()
  }, [query])

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Поиск..."
      />
      <ul>
        {results.map(item => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

### Использование SWR на клиенте

```tsx
'use client'

import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

export default function UserList() {
  const { data, error, isLoading, mutate } = useSWR('/api/users', fetcher)

  if (isLoading) return <div>Загрузка...</div>
  if (error) return <div>Ошибка</div>

  return (
    <div>
      <button onClick={() => mutate()}>Обновить</button>
      <ul>
        {data.map((user: any) => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  )
}
```

## Server Actions и мутации

> **Nuxt:** В Nuxt нет прямого аналога Server Actions. Мутации выполняются через: 1) **Form composable** (`useForm` из `@vueuse/core`) + API вызовы к `/server/api`; 2) **Server routes** (`/server/api/**`) для обработки POST/PUT/DELETE; 3) **Nitro event handlers** для сложной логики. Форма отправляется через обычный `<form>` с `@submit.prevent` и `$fetch` вызовом. Нет встроенной интеграции с формами как в Next.js `useActionState`.

### Базовая Server Action

```tsx
// app/actions/create-post.ts
'use server'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string

  await db.post.create({
    data: { title, content },
  })
}
```

### Использование в форме

```tsx
// app/posts/new/page.tsx
import { createPost } from '@/app/actions/create-post'

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Заголовок" />
      <textarea name="content" placeholder="Содержание" />
      <button type="submit">Создать пост</button>
    </form>
  )
}
```

### Server Action с revalidation

```tsx
// app/actions/create-post.ts
'use server'

import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string

  const post = await db.post.create({
    data: { title, content },
  })

  revalidatePath('/posts')
  redirect(`/posts/${post.id}`)
}
```

### Server Action с валидацией

```tsx
'use server'

import { z } from 'zod'

const PostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
})

export async function createPost(formData: FormData) {
  const rawData = {
    title: formData.get('title') as string,
    content: formData.get('content') as string,
  }

  const validated = PostSchema.parse(rawData)

  const post = await db.post.create({
    data: validated,
  })

  return post
}
```

### Server Action с обработкой ошибок

```tsx
'use server'

export type ActionState = {
  error?: string
  success?: boolean
}

export async function createPost(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  try {
    const title = formData.get('title') as string
    const content = formData.get('content') as string

    if (!title || title.length < 3) {
      return { error: 'Заголовок слишком короткий' }
    }

    await db.post.create({ data: { title, content } })

    return { success: true }
  } catch (error) {
    return { error: 'Произошла ошибка при создании поста' }
  }
}
```

### Использование useActionState (React 19)

```tsx
'use client'

import { useActionState } from 'react'
import { createPost, type ActionState } from '@/app/actions/create-post'

export default function NewPostForm() {
  const [state, action, isPending] = useActionState<ActionState, FormData>(
    createPost,
    {}
  )

  return (
    <form action={action}>
      <input name="title" placeholder="Заголовок" />
      <textarea name="content" placeholder="Содержание" />

      {state.error && <p className="text-red-500">{state.error}</p>}
      {state.success && <p className="text-green-500">Пост создан!</p>}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Создание...' : 'Создать'}
      </button>
    </form>
  )
}
```

## Route Handlers

> **Nuxt:** Аналог Route Handlers — **Server Routes** в директории `/server/api/`. Файл `/server/api/users.get.ts` создаёт endpoint `GET /api/users`. Поддерживаются все HTTP-методы через суффиксы: `.get.ts`, `.post.ts`, `.put.ts`, `.delete.ts`. Внутри используется `defineEventHandler` и `readBody(event)` для парсинга тела запроса. В отличие от Next.js, Nuxt server routes работают на **Nitro** (универсальный HTTP-сервер), что позволяет деплоить на больше платформ (Cloudflare Workers, Deno, Node.js).

### Базовый Route Handler

```tsx
// app/api/users/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  const users = await db.user.findMany()
  return NextResponse.json(users)
}

export async function POST(request: Request) {
  const body = await request.json()
  const user = await db.user.create({ data: body })
  return NextResponse.json(user, { status: 201 })
}
```

### Route Handler с параметрами

```tsx
// app/api/users/[id]/route.ts
import { NextResponse } from 'next/server'

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await db.user.findUnique({
    where: { id: params.id },
  })

  if (!user) {
    return NextResponse.json(
      { error: 'User not found' },
      { status: 404 }
    )
  }

  return NextResponse.json(user)
}

export async function PUT(
  request: Request,
  { params }: { params: { id: string } }
) {
  const body = await request.json()
  const user = await db.user.update({
    where: { id: params.id },
    data: body,
  })

  return NextResponse.json(user)
}

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  await db.user.delete({
    where: { id: params.id },
  })

  return new NextResponse(null, { status: 204 })
}
```

### Route Handler с кэшированием

```tsx
// app/api/products/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  const products = await fetch('https://api.example.com/products', {
    next: { revalidate: 60 },
  }).then(r => r.json())

  return NextResponse.json(products)
}
```

### Route Handler с query параметрами

```tsx
// app/api/search/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams
  const query = searchParams.get('q') || ''
  const page = Number(searchParams.get('page')) || 1
  const limit = Number(searchParams.get('limit')) || 20

  const results = await search(query, page, limit)

  return NextResponse.json({
    results,
    page,
    limit,
    total: results.total,
  })
}
```

## Streaming и Suspense

> **Nuxt:** В Nuxt нет React Suspense. Вместо этого используется **`<Suspense>` из Vue 3** (экспериментальный в Vue 3.3+, стабилен в 3.5+) или **`<DevOnly>`** для условного рендеринга. Для гранулярной загрузки данных используется `useAsyncData` с `lazy: true` — данные загружаются асинхронно без блокировки рендера. Стриминг HTML работает через SSR, но без progressive rendering как в React. Для loading states используется `useFetch` + `status` / `pending` ref.

### Базовый Suspense для streaming

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'

export default async function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>

      <Suspense fallback={<StatsSkeleton />}>
        <StatsWidget />
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <ChartWidget />
      </Suspense>

      <Suspense fallback={<ActivitySkeleton />}>
        <ActivityFeed />
      </Suspense>
    </div>
  )
}

async function StatsWidget() {
  const stats = await fetch('https://api.example.com/stats').then(r => r.json())
  return <StatsPanel stats={stats} />
}

async function ChartWidget() {
  const data = await fetch('https://api.example.com/chart').then(r => r.json())
  return <Chart data={data} />
}

async function ActivityFeed() {
  const activities = await fetch('https://api.example.com/activity').then(r => r.json())
  return <ActivityList activities={activities} />
}
```

### Как работает streaming

```
Время →

0ms     ┌─ HTML с fallback (skeleton) отправлен клиенту
        │
100ms   ├─ StatsWidget загрузился → заменили skeleton на контент
        │
200ms   ├─ ChartWidget загрузился → заменили skeleton на контент
        │
500ms   └─ ActivityFeed загрузился → заменили skeleton на контент

Пользователь видит контент постепенно, а не ждёт полной загрузки.
```

### loading.tsx как автоматический Suspense

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div>
      <div className="skeleton h-8 w-48" />
      <div className="skeleton h-64 w-full mt-4" />
    </div>
  )
}

// app/dashboard/page.tsx
// loading.tsx автоматически оборачивает page.tsx в Suspense
export default async function DashboardPage() {
  const data = await fetchSlowData()
  return <Dashboard data={data} />
}
```

### Вложенный Suspense для гранулярного streaming

```tsx
// app/shop/page.tsx
import { Suspense } from 'react'

export default async function ShopPage() {
  return (
    <div>
      <Suspense fallback={<CategorySkeleton />}>
        <CategoryList />
      </Suspense>

      <Suspense fallback={<ProductsSkeleton />}>
        <ProductGrid />
      </Suspense>
    </div>
  )
}

async function CategoryList() {
  const categories = await fetch('https://api.example.com/categories', {
    next: { revalidate: 3600 },
  }).then(r => r.json())

  return (
    <nav>
      {categories.map(cat => (
        <a key={cat.id} href={`/shop/${cat.slug}`}>{cat.name}</a>
      ))}
    </nav>
  )
}

async function ProductGrid() {
  const products = await fetch('https://api.example.com/products', {
    cache: 'no-store',
  }).then(r => r.json())

  return (
    <div className="grid grid-cols-3">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  )
}
```

## Практические паттерны

### Паттерн 1: Статическая генерация с ISR

```tsx
// app/blog/page.tsx
export const revalidate = 3600 // Перегенерация каждый час

export default async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 },
  }).then(r => r.json())

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

### Паттерн 2: Динамическая страница с данными пользователя

```tsx
// app/profile/page.tsx
import { cookies } from 'next/headers'

export const dynamic = 'force-dynamic'

export default async function ProfilePage() {
  const token = cookies().get('auth-token')?.value

  if (!token) {
    redirect('/login')
  }

  const user = await fetch('https://api.example.com/me', {
    headers: { Authorization: `Bearer ${token}` },
    cache: 'no-store',
  }).then(r => r.json())

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

### Паттерн 3: Гибридный подход (сервер + клиент)

```tsx
// app/products/page.tsx — Серверная загрузка списка
import { Suspense } from 'react'
import { ProductFilters } from './components/ProductFilters'

export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products', {
    next: { revalidate: 60 },
  }).then(r => r.json())

  return (
    <div>
      <ProductFilters />
      <Suspense fallback={<ProductsSkeleton />}>
        <ProductList initialProducts={products} />
      </Suspense>
    </div>
  )
}

// app/products/components/ProductFilters.tsx — Клиентская фильтрация
'use client'

import { useState } from 'react'

export function ProductFilters() {
  const [category, setCategory] = useState('all')
  const [sort, setSort] = useState('name')

  return (
    <div>
      <select value={category} onChange={e => setCategory(e.target.value)}>
        <option value="all">Все</option>
        <option value="electronics">Электроника</option>
      </select>
      <select value={sort} onChange={e => setSort(e.target.value)}>
        <option value="name">По имени</option>
        <option value="price">По цене</option>
      </select>
    </div>
  )
}
```

### Паттерн 4: Prefetching при наведении

```tsx
// app/components/ProductLink.tsx
'use client'

import Link from 'next/link'
import { useRouter } from 'next/navigation'

export function ProductLink({ id, name }: { id: string; name: string }) {
  const router = useRouter()

  const handleMouseEnter = () => {
    router.prefetch(`/products/${id}`)
  }

  return (
    <Link
      href={`/products/${id}`}
      onMouseEnter={handleMouseEnter}
    >
      {name}
    </Link>
  )
}
```

### Паттерн 5: Оптимистичные обновления через Server Actions

```tsx
// app/actions/toggle-todo.ts
'use server'

import { revalidateTag } from 'next/cache'

export async function toggleTodo(todoId: string) {
  const todo = await db.todo.findUnique({ where: { id: todoId } })

  await db.todo.update({
    where: { id: todoId },
    data: { done: !todo.done },
  })

  revalidateTag('todos')
}

// app/todos/page.tsx
import { Suspense } from 'react'
import { TodoList } from './TodoList'

export default async function TodosPage() {
  return (
    <div>
      <h1>Мои задачи</h1>
      <Suspense fallback={<div>Загрузка...</div>}>
        <TodoList />
      </Suspense>
    </div>
  )
}

// app/todos/TodoList.tsx
import { toggleTodo } from '@/app/actions/toggle-todo'
import { db } from '@/lib/db'

export async function TodoList() {
  const todos = await db.todo.findMany()

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <form>
            <input
              type="checkbox"
              defaultChecked={todo.done}
              onChange={async () => {
                'use server'
                await toggleTodo(todo.id)
              }}
            />
            <span className={todo.done ? 'line-through' : ''}>
              {todo.title}
            </span>
          </form>
        </li>
      ))}
    </ul>
  )
}
```

### Паттерн 6: Обработка ошибок с retry

```tsx
// lib/fetch-with-retry.ts
export async function fetchWithRetry(
  url: string,
  options?: RequestInit,
  retries = 3
): Promise<Response> {
  for (let i = 0; i < retries; i++) {
    try {
      const res = await fetch(url, options)
      if (res.ok) return res
      if (res.status >= 500 && i < retries - 1) {
        await new Promise(r => setTimeout(r, 1000 * (i + 1)))
        continue
      }
      return res
    } catch (error) {
      if (i === retries - 1) throw error
      await new Promise(r => setTimeout(r, 1000 * (i + 1)))
    }
  }
  throw new Error('Max retries exceeded')
}

// app/data/page.tsx
import { fetchWithRetry } from '@/lib/fetch-with-retry'

export default async function DataPage() {
  const data = await fetchWithRetry('https://api.example.com/unstable-data')
    .then(r => r.json())

  return <div>{JSON.stringify(data)}</div>
}
```

## Best Practices

### 1. Приоритет серверных компонентов

```tsx
// ❌ Плохо: клиентский компонент для загрузки данных
'use client'

export default function PostsPage() {
  const [posts, setPosts] = useState([])

  useEffect(() => {
    fetch('/api/posts').then(r => r.json()).then(setPosts)
  }, [])

  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}

// ✅ Хорошо: серверный компонент
export default async function PostsPage() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json())

  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

### 2. Колocation данных и компонентов

```
app/
├── products/
│   ├── page.tsx              # Серверный: загрузка списка
│   ├── [id]/
│   │   └── page.tsx          # Серверный: загрузка товара
│   ├── components/
│   │   ├── ProductCard.tsx   # Клиентский: интерактивность
│   │   └── ProductGrid.tsx   # Серверный: layout
│   └── actions/
│       └── add-to-cart.ts    # Server Action
```

### 3. Использование loading states

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>

      <Suspense fallback={<StatsSkeleton />}>
        <StatsSection />
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <ChartSection />
      </Suspense>
    </div>
  )
}
```

### 4. Типизация данных

```tsx
// types/post.ts
export interface Post {
  id: string
  title: string
  content: string
  author: {
    id: string
    name: string
  }
  createdAt: string
}

// lib/posts.ts
export async function getPosts(): Promise<Post[]> {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 },
  })

  if (!res.ok) throw new Error('Failed to fetch posts')

  return res.json()
}

// app/posts/page.tsx
import { getPosts } from '@/lib/posts'
import type { Post } from '@/types/post'

export default async function PostsPage() {
  const posts: Post[] = await getPosts()

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

### 5. Разделение статических и динамических данных

```tsx
// app/page.tsx
import { Suspense } from 'react'

export default async function HomePage() {
  return (
    <div>
      {/* Статический контент — кэшируется */}
      <HeroSection />

      {/* Динамический контент — не кэшируется */}
      <Suspense fallback={<FeedSkeleton />}>
        <UserFeed />
      </Suspense>
    </div>
  )
}

async function HeroSection() {
  // Статические данные — кэшируются на build
  const features = await fetch('https://api.example.com/features', {
    cache: 'force-cache',
  }).then(r => r.json())

  return <Hero features={features} />
}

async function UserFeed() {
  // Динамические данные — без кэша
  const feed = await fetch('https://api.example.com/feed', {
    cache: 'no-store',
    headers: { Authorization: `Bearer ${getToken()}` },
  }).then(r => r.json())

  return <Feed items={feed} />
}
```

### 6. Использование generateMetadata для SEO

```tsx
// app/products/[id]/page.tsx
import type { Metadata } from 'next'

export async function generateMetadata({
  params,
}: {
  params: { id: string }
}): Promise<Metadata> {
  const product = await getProduct(params.id)

  return {
    title: product.name,
    description: product.description,
    openGraph: {
      images: [product.image],
    },
  }
}

export default async function ProductPage({
  params,
}: {
  params: { id: string }
}) {
  const product = await getProduct(params.id)

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </div>
  )
}

// Функция для загрузки данных — мемоизируется
async function getProduct(id: string) {
  return fetch(`https://api.example.com/products/${id}`, {
    next: { revalidate: 60 },
  }).then(r => r.json())
}
```

### 7. Обработка ошибок на разных уровнях

```tsx
// app/error.tsx — глобальный error boundary
'use client'

export default function GlobalError({
  error,
  reset,
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div>
      <h1>Что-то пошло не так</h1>
      <button onClick={reset}>Попробовать снова</button>
    </div>
  )
}

// app/products/error.tsx — специфичный для секции
'use client'

export default function ProductsError({
  error,
  reset,
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div>
      <h2>Ошибка загрузки товаров</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Повторить</button>
    </div>
  )
}
```

## Антипаттерны

### 1. Клиентский fetch для статических данных

```tsx
// ❌ Плохо: лишний JavaScript на клиенте
'use client'

export default function AboutPage() {
  const [content, setContent] = useState(null)

  useEffect(() => {
    fetch('/api/about').then(r => r.json()).then(setContent)
  }, [])

  return <div>{content?.text}</div>
}

// ✅ Хорошо: серверный компонент, нет JS на клиенте
export default async function AboutPage() {
  const content = await fetch('https://api.example.com/about').then(r => r.json())
  return <div>{content.text}</div>
}
```

### 2. Waterfall запросов без необходимости

```tsx
// ❌ Плохо: последовательные независимые запросы
export default async function Page() {
  const users = await fetch('/api/users').then(r => r.json())    // 200ms
  const posts = await fetch('/api/posts').then(r => r.json())    // 200ms
  const stats = await fetch('/api/stats').then(r => r.json())    // 200ms
  // Итого: 600ms

  return <div>...</div>
}

// ✅ Хорошо: параллельные запросы
export default async function Page() {
  const [users, posts, stats] = await Promise.all([
    fetch('/api/users').then(r => r.json()),  // ┐
    fetch('/api/posts').then(r => r.json()),   // ├─ 200ms
    fetch('/api/stats').then(r => r.json()),   // ┘
  ])
  // Итого: 200ms

  return <div>...</div>
}
```

### 3. Дублирование fetch запросов

```tsx
// ❌ Плохо: один и тот же fetch в разных компонентах
async function Header() {
  const user = await fetch('/api/me').then(r => r.json())
  return <div>{user.name}</div>
}

async function Sidebar() {
  const user = await fetch('/api/me').then(r => r.json())
  return <div>{user.email}</div>
}

// ✅ Хорошо: один fetch, данные передаются через props
export default async function Page() {
  const user = await fetch('/api/me').then(r => r.json())

  return (
    <div>
      <Header user={user} />
      <Sidebar user={user} />
    </div>
  )
}
```

> **Примечание:** Next.js автоматически мемоизирует одинаковые `fetch` запросы в пределах одного render. Но явная передача данных через props всё равно предпочтительнее для читаемости.

### 4. Отсутствие обработки ошибок

```tsx
// ❌ Плохо: нет обработки ошибок
export default async function Page() {
  const data = await fetch('https://api.example.com/data').then(r => r.json())
  return <div>{data.value}</div>
}

// ✅ Хорошо: обработка ошибок
export default async function Page() {
  let data
  try {
    const res = await fetch('https://api.example.com/data')
    if (!res.ok) throw new Error('Failed to fetch')
    data = await res.json()
  } catch (error) {
    // Ошибка попадёт в error.tsx
    throw error
  }

  return <div>{data.value}</div>
}
```

### 5. Использование 'use client' без необходимости

```tsx
// ❌ Плохо: весь компонент клиентский из-за одной кнопки
'use client'

export default async function ProductPage({ params }) {
  const product = await fetch(`/api/products/${params.id}`).then(r => r.json())
  const [quantity, setQuantity] = useState(1)

  return (
    <div>
      <h1>{product.name}</h1>
      <button onClick={() => setQuantity(q => q + 1)}>+</button>
    </div>
  )
}

// ✅ Хорошо: серверный компонент + клиентский подкомпонент
export default async function ProductPage({ params }) {
  const product = await fetch(`/api/products/${params.id}`).then(r => r.json())

  return (
    <div>
      <h1>{product.name}</h1>
      <AddToCartButton productId={product.id} />
    </div>
  )
}

// components/AddToCartButton.tsx
'use client'

import { useState } from 'react'

export function AddToCartButton({ productId }: { productId: string }) {
  const [quantity, setQuantity] = useState(1)

  return (
    <button onClick={() => setQuantity(q => q + 1)}>+</button>
  )
}
```

### 6. Неправильное использование revalidate

```tsx
// ❌ Плохо: revalidate: 0 — кэш сразу невалиден
const data = await fetch('/api/data', {
  next: { revalidate: 0 }  // Бессмысленно
}).then(r => r.json())

// ✅ Хорошо: revalidate для редко меняющихся данных
const data = await fetch('/api/data', {
  next: { revalidate: 3600 }  // Обновление раз в час
}).then(r => r.json())

// ✅ Хорошо: no-store для динамических данных
const data = await fetch('/api/data', {
  cache: 'no-store'
}).then(r => r.json())
```

### 7. Загрузка данных на клиенте через Route Handler

```tsx
// ❌ Плохо: лишнее звено — Route Handler для простого fetch
// app/api/posts/route.ts
export async function GET() {
  const posts = await fetch('https://external-api.com/posts').then(r => r.json())
  return NextResponse.json(posts)
}

// app/posts/page.tsx — клиентский компонент
'use client'
useEffect(() => {
  fetch('/api/posts').then(r => r.json()).then(setPosts)
}, [])

// ✅ Хорошо: прямой fetch из серверного компонента
export default async function PostsPage() {
  const posts = await fetch('https://external-api.com/posts').then(r => r.json())
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

---

## Сравнение с Nuxt.js

### Архитектурные различия

| Аспект | Next.js (App Router) | Nuxt 3 |
|--------|---------------------|--------|
| **Модель компонентов** | Server Components + `'use client'` | Универсальные компоненты (SSR + гидратация) |
| **Загрузка данных на сервере** | `async` компоненты + `await fetch()` | `useFetch()` / `useAsyncData()` в `<script setup>` |
| **Загрузка данных на клиенте** | Client Components + `useEffect` / SWR | Те же `useFetch()` / `useAsyncData()` (работают везде) |
| **Мутации** | Server Actions (`'use server'`) | Server Routes (`/server/api/**`) + `$fetch` |
| **API endpoints** | Route Handlers (`app/api/**/route.ts`) | Server Routes (`/server/api/**.get.ts`) |
| **Кэширование данных** | Data Cache (серверный, persistent) | Payload cache (передаётся с SSR) + Nitro caching |
| **Клиентский кэш** | Router Cache (RSC payload в браузере) | Нет отдельного клиентского кэша |
| **Streaming** | React Suspense + `loading.tsx` | Vue `<Suspense>` (экспериментальный) + `lazy: true` |
| **Runtime** | Node.js / Edge (Vercel) | Nitro (Node.js, Deno, Cloudflare Workers, Bun) |
| **Revalidation** | `revalidatePath`, `revalidateTag` | `refreshNuxtData()`, `clearNuxtData()`, `routeRules.isr` |

### Синтаксис: загрузка данных

**Next.js:**

```tsx
// app/posts/page.tsx — Server Component
export default async function PostsPage() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json())

  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}
```

**Nuxt:**

```vue
<!-- pages/posts/index.vue -->
<script setup>
const { data: posts } = await useFetch('https://api.example.com/posts')
</script>

<template>
  <ul>
    <li v-for="post in posts" :key="post.id">{{ post.title }}</li>
  </ul>
</template>
```

**Ключевые отличия:**
- Next.js использует `async` функции и `await` напрямую в компоненте
- Nuxt использует Composition API (`useFetch`, `useAsyncData`) в `<script setup>`
- В Next.js компонент **либо** серверный, **либо** клиентский (`'use client'`)
- В Nuxt компонент **универсальный** — работает на сервере (SSR) и на клиенте (гидратация)

### Синтаксис: мутации

**Next.js (Server Actions):**

```tsx
// app/actions/create-post.ts
'use server'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  await db.post.create({ data: { title } })
  revalidatePath('/posts')
}

// app/posts/new/page.tsx
import { createPost } from '@/app/actions/create-post'

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Создать</button>
    </form>
  )
}
```

**Nuxt (Server Routes + $fetch):**

```ts
// server/api/posts.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  return await db.post.create({ data: body })
})

// pages/posts/new.vue
<script setup>
const title = ref('')

async function handleSubmit() {
  await $fetch('/api/posts', {
    method: 'POST',
    body: { title: title.value }
  })
  await refreshNuxtData('posts') // Обновить кэш
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <input v-model="title" />
    <button type="submit">Создать</button>
  </form>
</template>
```

**Ключевые отличия:**
- Next.js Server Actions — **типобезопасные функции**, вызываемые напрямую из компонентов
- Nuxt — **HTTP-вызовы** к server routes через `$fetch`
- Next.js имеет встроенную интеграцию с формами (`<form action={...}>`)
- Nuxt использует стандартные формы с `@submit.prevent` и JavaScript-обработчиками

### Кэширование

**Next.js:**

```tsx
// Кэш по умолчанию (force-cache)
const data = await fetch('/api/data').then(r => r.json())

// Без кэша
const data = await fetch('/api/data', { cache: 'no-store' }).then(r => r.json())

// Кэш с revalidation
const data = await fetch('/api/data', { next: { revalidate: 60 } }).then(r => r.json())

// On-demand revalidation
revalidatePath('/data')
revalidateTag('data')
```

**Nuxt:**

```ts
// useFetch автоматически кэширует в payload (SSR → клиент)
const { data } = await useFetch('/api/data')

// Без кэша (каждый раз новый запрос)
const { data } = await useFetch('/api/data', { 
  $fetch: { headers: { 'Cache-Control': 'no-store' } } 
})

// Серверное кэширование через routeRules (nuxt.config.ts)
export default defineNuxtConfig({
  routeRules: {
    '/api/data': { swr: true },              // Stale-while-revalidate
    '/blog/**': { isr: 3600 },               // Incremental Static Regeneration
    '/static/**': { static: true },          // Полностью статический
  }
})

// Обновление данных на клиенте
await refreshNuxtData('data-key')
```

**Ключевые отличия:**
- Next.js имеет **Data Cache** — серверный кэш fetch-запросов с granular control
- Nuxt кэширует данные в **payload** (передаётся с SSR) и делегирует HTTP-кэширование Nitro
- Next.js: `revalidatePath` / `revalidateTag` для on-demand revalidation
- Nuxt: `refreshNuxtData()` для клиентского обновления, `routeRules.isr` для серверного

### Streaming и Loading States

**Next.js:**

```tsx
import { Suspense } from 'react'

export default function Page() {
  return (
    <div>
      <Suspense fallback={<Skeleton />}>
        <SlowComponent />
      </Suspense>
    </div>
  )
}

// loading.tsx — автоматический Suspense fallback
export default function Loading() {
  return <Skeleton />
}
```

**Nuxt:**

```vue
<script setup>
// lazy: true — не блокирует рендер, данные загружаются асинхронно
const { data, pending } = await useFetch('/api/data', { lazy: true })
</script>

<template>
  <div>
    <div v-if="pending">Загрузка...</div>
    <div v-else>{{ data }}</div>
  </div>
</template>
```

**Ключевые отличия:**
- Next.js использует **React Suspense** для granular streaming (прогрессивная загрузка секций)
- Nuxt использует **`lazy: true`** в `useFetch` — данные загружаются без блокировки, но без progressive HTML streaming
- Next.js: `loading.tsx` автоматически оборачивает страницу в Suspense
- Nuxt: `pending` ref для ручного управления loading state
- Next.js поддерживает **progressive HTML streaming** (сервер отправляет HTML по частям)
- Nuxt: SSR отдаёт полный HTML после загрузки всех данных (нет progressive streaming из коробки)

### Обработка ошибок

**Next.js:**

```tsx
// app/error.tsx — Error Boundary
'use client'
export default function Error({ error, reset }) {
  return <button onClick={reset}>Попробовать снова</button>
}

// app/not-found.tsx — 404 страница
export default function NotFound() {
  return <div>Страница не найдена</div>
}

// В серверном компоненте
import { notFound } from 'next/navigation'
if (!data) notFound()
```

**Nuxt:**

```vue
<!-- error.vue — глобальная страница ошибок -->
<script setup>
const props = defineProps({ error: Object })
const handleError = () => clearError({ redirect: '/' })
</script>

<template>
  <div>
    <h1>{{ error.statusCode }}</h1>
    <button @click="handleError">Исправить</button>
  </div>
</template>

<!-- В useFetch -->
<script setup>
const { data, error } = await useFetch('/api/data')
if (error.value) {
  // Обработка ошибки
}
</script>
```

**Ключевые отличия:**
- Next.js: `error.tsx` — React Error Boundary (клиентский компонент)
- Nuxt: `error.vue` — глобальная страница ошибок (универсальный компонент)
- Next.js: `notFound()` — программный вызов 404
- Nuxt: `createError({ statusCode: 404 })` — создание ошибки
- Next.js: `reset()` в error boundary для retry
- Nuxt: `clearError()` для сброса ошибки

### Когда что использовать

| Сценарий | Next.js | Nuxt |
|----------|---------|------|
| **Статический контент** | `async` компонент + `force-cache` | `useFetch` + `routeRules: { static: true }` |
| **Динамические данные** | `cache: 'no-store'` | `useFetch` + `$fetch: { headers: { 'Cache-Control': 'no-store' } }` |
| **Данные с TTL** | `next: { revalidate: 60 }` | `routeRules: { isr: 60 }` |
| **Мутации** | Server Actions | Server Routes + `$fetch` |
| **Real-time данные** | Client Component + WebSocket | `useFetch` + WebSocket / SSE |
| **Form submissions** | `<form action={serverAction}>` | `<form @submit.prevent="handler">` + `$fetch` |
| **On-demand revalidation** | `revalidatePath` / `revalidateTag` | `refreshNuxtData()` |

### Миграция между фреймворками

**Из Next.js в Nuxt:**
- `async` Server Components → `useFetch` / `useAsyncData` в `<script setup>`
- `'use client'` компоненты → универсальные компоненты (нет разделения)
- Server Actions → Server Routes (`/server/api/**`) + `$fetch`
- Route Handlers → Server Routes (`.get.ts`, `.post.ts`)
- `revalidatePath` → `refreshNuxtData()` + `routeRules.isr`
- React Suspense → `lazy: true` в `useFetch` + `pending` ref

**Из Nuxt в Next.js:**
- `useFetch` → `async` Server Components + `await fetch()`
- `<script setup>` → `async function Component() { ... }`
- Server Routes → Route Handlers (`app/api/**/route.ts`)
- `$fetch` → Server Actions (для мутаций) или `fetch` (для GET)
- `routeRules.isr` → `next: { revalidate: N }` в fetch
- `refreshNuxtData()` → `revalidatePath` / `revalidateTag`
