# Глубокое погружение в Route Handlers и Server Actions

Route Handlers и Server Actions — два механизма серверной логики в Next.js App Router. Route Handlers дают полный контроль над HTTP-запросом/ответом (как традиционный REST API), а Server Actions абстрагируют эту сложность, позволяя вызывать серверные функции напрямую из компонентов. Этот документ разбирает оба подхода, их внутреннее устройство и ключевые различия.

## Содержание

- [Что такое route.ts — простое объяснение](#что-такое-routets--простое-объяснение)
- [Маппинг файлов на URL](#маппинг-файлов-на-url)
- [NextResponse.json() — что это и зачем](#nextresponsejson--что-это-и-зачем)
- [fetch — универсальный HTTP клиент](#fetch--универсальный-http-клиент)
- [BFF паттерн — адаптация кривого API](#bff-паттерн--адаптация-кривого-api)
- [Route Handlers vs Server Actions — когда что использовать](#route-handlers-vs-server-actions--когда-что-использовать)
- [Архитектура серверной логики](#архитектура-серверной-логики)
- [Route Handlers](#route-handlers)
- [HTTP методы и обработчики](#http-методы-и-обработчики)
- [Request и Response объекты](#request-и-response-объекты)
- [Кэширование в Route Handlers](#кэширование-в-route-handlers)
- [Динамические сегменты](#динамические-сегменты)
- [CORS и заголовки](#cors-и-заголовки)
- [Streaming и Web Streams API](#streaming-и-web-streams-api)
- [Server Actions](#server-actions)
- [Директива "use server"](#директива-use-server)
- [Вызов Server Actions](#вызов-server-actions)
- [useFormState и useFormStatus](#useformstate-и-useformstatus)
- [Revalidation и мутации](#revalidation-и-мутации)
- [Оптимистичные обновления](#оптимистичные-обновления)
- [Server Actions под капотом](#server-actions-под-капотом)
- [Route Handlers vs Server Actions](#route-handlers-vs-server-actions)
- [Таблица сравнения](#таблица-сравнения)
- [Когда что использовать](#когда-что-использовать)
- [Практические паттерны](#практические-паттерны)
- [Best Practices](#best-practices)
- [Антипаттерны](#антипаттерны)
- [Сравнение с Nuxt.js](#сравнение-с-nuxtjs)

## Что такое route.ts — простое объяснение

### route.ts ≠ автоматический вызов на странице

**Важно понимать:** `route.ts` и `page.tsx` — это **разные** вещи, которые не вызывают друг друга автоматически.

```
❌ НЕПРАВИЛЬНОЕ ПОНИМАНИЕ:
app/posts/route.ts с GET() → автоматически вызывается при заходе на /posts
app/posts/page.tsx → рендерит страницу

✅ ПРАВИЛЬНОЕ ПОНИМАНИЕ:
app/api/posts/route.ts → API endpoint на /api/posts (бэкенд)
app/posts/page.tsx → UI страница на /posts (фронтенд)
```

**Правило:** Ты **не можешь** иметь оба файла на одном уровне (`app/posts/route.ts` + `app/posts/page.tsx`) — Next.js выдаст ошибку.

### Стандартная структура

```
app/
├── api/
│   └── posts/
│       └── route.ts     → API endpoint на /api/posts
└── posts/
    └── page.tsx         → UI страница на /posts
```

### Как фронтенд вызывает бэкенд

```tsx
// app/posts/page.tsx (фронтенд)
export default async function PostsPage() {
  // Сам вызываешь API через fetch
  const res = await fetch('http://localhost:3000/api/posts')
  const posts = await res.json()
  
  return (
    <ul>
      {posts.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  )
}
```

## Маппинг файлов на URL

### Путь к файлу = путь URL

```
app/api/posts/route.ts       →  /api/posts
app/api/users/route.ts       →  /api/users
app/api/posts/[id]/route.ts  →  /api/posts/1, /api/posts/2
app/api/docs/[...slug]/route.ts → /api/docs/a, /api/docs/a/b
```

### HTTP методы определяются именами функций

```ts
// app/api/posts/route.ts
export async function GET() { }    // GET /api/posts
export async function POST() { }   // POST /api/posts
export async function PUT() { }    // PUT /api/posts
export async function DELETE() { } // DELETE /api/posts
```

**Важно:** В одном файле **не может** быть двух функций с одинаковым именем:

```ts
// ❌ Ошибка: duplicate export
export async function GET() { }
export async function GET() { }

// ✅ Правильно: одна функция на метод
export async function GET() { }
export async function POST() { }
```

### Динамические параметры

```ts
// app/api/posts/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const id = params.id  // "1", "2", и т.д.
  return NextResponse.json({ id })
}
```

## NextResponse.json() — что это и зачем

### Что происходит внутри

`NextResponse.json()` превращает JavaScript объект в правильный HTTP ответ:

```ts
import { NextResponse } from 'next/server'

export async function GET() {
  const data = { id: 1, title: 'Hello' }
  
  // Превращает объект в HTTP ответ с:
  // - Status: 200
  // - Content-Type: application/json
  // - Body: {"id":1,"title":"Hello"}
  return NextResponse.json(data)
}
```

### Почему нельзя просто вернуть объект

```ts
// ❌ Не сработает
return { id: 1 }

// ❌ Тоже не сработает
return JSON.stringify({ id: 1 })

// ✅ Правильно
return NextResponse.json({ id: 1 })
```

### Дополнительные опции

```ts
// Кастомный статус
return NextResponse.json({ error: 'Not found' }, { status: 404 })

// Заголовки
return NextResponse.json(data, {
  status: 201,
  headers: { 'X-Custom-Header': 'value' }
})

// Cookies
const response = NextResponse.json({ ok: true })
response.cookies.set('token', 'abc123', {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
})
return response
```

### Другие типы ответов

```ts
// Текст
return new Response('Hello', {
  headers: { 'Content-Type': 'text/plain' }
})

// HTML
return new Response('<h1>Hello</h1>', {
  headers: { 'Content-Type': 'text/html' }
})

// Пустой ответ
return new Response(null, { status: 204 })
```

## fetch — универсальный HTTP клиент

### Базовое использование

`fetch` — встроенная функция браузера для HTTP запросов. По умолчанию использует GET:

```ts
// GET запрос (по умолчанию)
fetch('/api/posts')

// POST запрос
fetch('/api/posts', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title: 'New Post' })
})
```

### Все HTTP методы

```ts
// GET — получить данные
fetch('/api/posts')

// POST — создать
fetch('/api/posts', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title: 'New Post' })
})

// PUT — обновить полностью
fetch('/api/posts/1', {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title: 'Updated' })
})

// PATCH — обновить частично
fetch('/api/posts/1', {
  method: 'PATCH',
  body: JSON.stringify({ title: 'Updated' })
})

// DELETE — удалить
fetch('/api/posts/1', {
  method: 'DELETE'
})
```

### Использование в компонентах

```tsx
// app/posts/page.tsx
export default async function PostsPage() {
  // GET запрос
  const res = await fetch('http://localhost:3000/api/posts')
  const posts = await res.json()
  
  return (
    <div>
      <h1>Posts</h1>
      <ul>
        {posts.map(p => <li key={p.id}>{p.title}</li>)}
      </ul>
      
      {/* Для POST/PUT/DELETE нужен клиентский компонент */}
      <ClientActions />
    </div>
  )
}
```

```tsx
// app/posts/ClientActions.tsx
'use client'

export default function ClientActions() {
  // POST - создать
  const createPost = async () => {
    await fetch('/api/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: 'New Post' })
    })
  }

  // PUT - обновить
  const updatePost = async (id: number) => {
    await fetch(`/api/posts/${id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: 'Updated' })
    })
  }

  // DELETE - удалить
  const deletePost = async (id: number) => {
    await fetch(`/api/posts/${id}`, {
      method: 'DELETE'
    })
  }

  return (
    <div>
      <button onClick={createPost}>Create</button>
      <button onClick={() => updatePost(1)}>Update</button>
      <button onClick={() => deletePost(1)}>Delete</button>
    </div>
  )
}
```

## BFF паттерн — адаптация кривого API

### Что такое BFF

**BFF (Backend for Frontend)** — прослойка между фронтом и бэком, которая адаптирует кривой API под нормальный.

### Пример: кривой REST API

```
POST /api/action?action=get_posts
POST /api/action?action=create_post&title=...
POST /api/action?action=delete_post&id=...
```

### Адаптация через Route Handlers

```ts
// app/api/posts/route.ts
export async function GET() {
  // Адаптируем кривой бэк
  const res = await fetch('http://backend.com/api/action?action=get_posts')
  const data = await res.json()
  return NextResponse.json(data)
}

export async function POST(request: Request) {
  const body = await request.json()
  const res = await fetch(
    `http://backend.com/api/action?action=create_post&title=${body.title}`,
    { method: 'POST' }
  )
  return NextResponse.json(await res.json())
}
```

```ts
// app/api/posts/[id]/route.ts
export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  const res = await fetch(
    `http://backend.com/api/action?action=delete_post&id=${params.id}`,
    { method: 'POST' }
  )
  return NextResponse.json(await res.json())
}
```

### Что можешь делать в BFF

- Менять URL и методы
- Трансформировать данные
- Объединять несколько запросов в один
- Добавлять кэширование, авторизацию, логирование
- Адаптировать кривой API под нормальный

### Пример: нормализация данных

```ts
// app/api/posts/route.ts
export async function GET() {
  // Кривой бэк: два запроса вместо одного
  const [postsRes, usersRes] = await Promise.all([
    fetch('http://backend.com/api/posts'),
    fetch('http://backend.com/api/users')
  ])
  
  const posts = await postsRes.json()
  const users = await usersRes.json()
  
  // Нормализуем данные для фронта
  const normalized = posts.map(post => ({
    id: post.id,
    title: post.name,  // бэк возвращает "name" вместо "title"
    author: users.find(u => u.id === post.userId)?.name
  }))
  
  return NextResponse.json(normalized)
}
```

## Route Handlers vs Server Actions — когда что использовать

### Простое сравнение

**Route Handlers** — для GET запросов, внешних API, webhook'ов, когда нужен полный контроль

**Server Actions** — для форм и мутаций, когда нужно просто отправить данные на сервер

### Сравнение кода

**Route Handler (старый способ):**

```tsx
// app/posts/ClientActions.tsx
'use client'

export default function ClientActions() {
  const createPost = async () => {
    await fetch('/api/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: 'New Post' })
    })
  }

  return <button onClick={createPost}>Create</button>
}
```

**Server Action (новый способ):**

```tsx
// app/posts/ClientActions.tsx
'use client'
import { createPost } from './actions'

export default function ClientActions() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button>Create</button>
    </form>
  )
}
```

```ts
// app/posts/actions.ts
'use server'

export async function createPost(formData: FormData) {
  const title = formData.get('title')
  // Работаем с БД напрямую
  await db.query('INSERT INTO posts ...', { title })
}
```

### Почему не только Route Handlers?

- Server Actions проще для форм
- Нет нужды писать `fetch`, `JSON.stringify`, заголовки
- Автоматическая сериализация
- Работают даже без JavaScript в браузере (прогрессивное улучшение)

### Итого

- **GET данные** → Route Handler или напрямую в `page.tsx`
- **Мутации (POST/PUT/DELETE)** → Server Actions
- **Внешние API/webhook'и** → Route Handlers
- **Кривой API** → Route Handlers как BFF прослойка

## Архитектура серверной логики

### Два подхода к серверной логике

```
┌─────────────────────────────────────────────────────────────────┐
│                        Next.js App Router                        │
├─────────────────────────────────┬───────────────────────────────┤
│       Route Handlers            │       Server Actions          │
├─────────────────────────────────┼───────────────────────────────┤
│  app/api/users/route.ts         │  app/actions.ts               │
│                                 │  (или inline в компоненте)    │
│  ┌───────────────────────────┐  │  ┌───────────────────────────┐│
│  │ export async function     │  │  │ 'use server'              ││
│  │   GET(request: Request)   │  │  │ export async function     ││
│  │     { ... }               │  │  │   createPost(formData)    ││
│  │                           │  │  │   { ... }                 ││
│  │ export async function     │  │  │                           ││
│  │   POST(request: Request)  │  │  │                           ││
│  │     { ... }               │  │  │                           ││
│  └───────────────────────────┘  │  └───────────────────────────┘│
│                                 │                               │
│  Полный HTTP-контроль           │  Абстракция над HTTP          │
│  REST API стиль                 │  RPC-стиль                    │
│  GET/POST/PUT/PATCH/DELETE      │  Только POST (мутации)        │
│                                 │                               │
└─────────────────────────────────┴───────────────────────────────┘
```

### Поток выполнения

```
Route Handler:
Browser → HTTP Request → Next.js Router → route.ts → Response → Browser

Server Action:
Browser → Form Submit / Action Call → Hidden POST → Next.js Action Runtime → 
Server Function → Re-render / Redirect → Browser
```

## Route Handlers

### Что такое Route Handlers

Route Handlers — это функции, которые обрабатывают HTTP-запросы для определённого URL-маршрута. Они экспортируются из файла `route.ts` (или `route.js`) внутри директории `app`.

```
app/
├── api/
│   ├── route.ts              → /api
│   ├── users/
│   │   └── route.ts          → /api/users
│   ├── posts/
│   │   ├── route.ts          → /api/posts
│   │   └── [id]/
│   │       └── route.ts      → /api/posts/:id
│   └── webhooks/
│       └── stripe/
│           └── route.ts      → /api/webhooks/stripe
```

### Базовый Route Handler

```ts
// app/api/hello/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  return NextResponse.json({ message: 'Hello, World!' })
}
```

### Типизация

```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'

const UserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().int().positive().optional(),
})

type User = z.infer<typeof UserSchema>

export async function GET(request: NextRequest) {
  const users: User[] = await db.users.findMany()
  return NextResponse.json(users)
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const validated: User = UserSchema.parse(body)
    
    const user = await db.users.create({ data: validated })
    return NextResponse.json(user, { status: 201 })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { errors: error.errors },
        { status: 400 }
      )
    }
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    )
  }
}
```

## HTTP методы и обработчики

### Поддерживаемые методы

Route Handlers поддерживают все стандартные HTTP-методы:

```ts
// app/api/resource/route.ts
import { NextRequest, NextResponse } from 'next/server'

// GET — получение ресурса
export async function GET(request: NextRequest) {
  const data = await fetchData()
  return NextResponse.json(data)
}

// POST — создание ресурса
export async function POST(request: NextRequest) {
  const body = await request.json()
  const created = await createData(body)
  return NextResponse.json(created, { status: 201 })
}

// PUT — полная замена ресурса
export async function PUT(request: NextRequest) {
  const body = await request.json()
  const updated = await updateData(body)
  return NextResponse.json(updated)
}

// PATCH — частичное обновление
export async function PATCH(request: NextRequest) {
  const body = await request.json()
  const patched = await patchData(body)
  return NextResponse.json(patched)
}

// DELETE — удаление ресурса
export async function DELETE(request: NextRequest) {
  await deleteData()
  return new NextResponse(null, { status: 204 })
}

// HEAD — метаданные (без тела)
export async function HEAD(request: NextRequest) {
  return new NextResponse(null, {
    headers: { 'X-Custom-Header': 'value' },
  })
}

// OPTIONS — информация о доступных методах (CORS preflight)
export async function OPTIONS(request: NextRequest) {
  return new NextResponse(null, {
    status: 204,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    },
  })
}
```

### Обработка ошибок по методам

```ts
export async function POST(request: NextRequest) {
  try {
    const data = await request.json()
    const result = await process(data)
    return NextResponse.json(result)
  } catch (error) {
    // Логирование ошибки
    console.error('POST error:', error)
    
    // Возврат структурированной ошибки
    return NextResponse.json(
      {
        error: {
          code: 'PROCESSING_FAILED',
          message: error instanceof Error ? error.message : 'Unknown error',
        },
      },
      { status: 500 }
    )
  }
}
```

## Request и Response объекты

### NextRequest (расширенный Request)

`NextRequest` расширяет стандартный Web `Request` дополнительными методами:

```ts
import { NextRequest } from 'next/server'

export async function GET(request: NextRequest) {
  // URL и путь
  const url = request.url                    // Полный URL
  const nextUrl = request.nextUrl            // URL-объект Next.js
  const pathname = nextUrl.pathname          // /api/users
  const searchParams = nextUrl.searchParams  // URLSearchParams
  
  // Query-параметры
  const page = searchParams.get('page')      // ?page=2
  const limit = searchParams.get('limit')    // ?limit=10
  
  // Заголовки
  const contentType = request.headers.get('content-type')
  const auth = request.headers.get('authorization')
  const userAgent = request.headers.get('user-agent')
  
  // Cookies
  const session = request.cookies.get('session')?.value
  const allCookies = request.cookies.getAll()
  
  // Geo (если доступен)
  const geo = request.geo
  const country = geo?.country
  const city = geo?.city
  
  // IP (если доступен)
  const ip = request.ip
  
  // Body (для POST/PUT/PATCH)
  const body = await request.json()          // JSON
  const text = await request.text()          // Текст
  const buffer = await request.arrayBuffer() // ArrayBuffer
  const formData = await request.formData()  // FormData
  const blob = await request.blob()          // Blob
  
  // Method
  const method = request.method              // 'GET', 'POST', etc.
  
  return NextResponse.json({ ok: true })
}
```

### NextResponse (расширенный Response)

```ts
import { NextResponse } from 'next/server'

// JSON-ответ
return NextResponse.json({ data: 'value' })
return NextResponse.json({ data: 'value' }, { status: 201 })

// Текстовый ответ
return new NextResponse('Plain text', {
  headers: { 'Content-Type': 'text/plain' },
})

// HTML-ответ
return new NextResponse('<h1>Hello</h1>', {
  headers: { 'Content-Type': 'text/html' },
})

// Редирект
return NextResponse.redirect('https://example.com')
return NextResponse.redirect(new URL('/login', request.url))

// Редирект с кодом
return NextResponse.redirect('https://example.com', 301)

// Пустой ответ с кодом
return new NextResponse(null, { status: 204 })

// Установка заголовков
const response = NextResponse.json({ data: 'value' })
response.headers.set('X-Custom-Header', 'value')
response.headers.set('Cache-Control', 's-maxage=60')
return response

// Установка cookies
const response = NextResponse.json({ ok: true })
response.cookies.set('token', 'abc123', {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',
  maxAge: 60 * 60 * 24, // 1 день
  path: '/',
})
return response

// Удаление cookies
const response = NextResponse.json({ ok: true })
response.cookies.delete('token')
return response
```

### Работа с Query-параметрами

```ts
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  
  // Один параметр
  const page = searchParams.get('page') || '1'
  const limit = searchParams.get('limit') || '10'
  
  // Массив параметров (?tag=a&tag=b)
  const tags = searchParams.getAll('tag')
  
  // Проверка наличия
  const hasFilter = searchParams.has('filter')
  
  // Итерация
  searchParams.forEach((value, key) => {
    console.log(key, value)
  })
  
  const data = await fetchData({
    page: parseInt(page),
    limit: parseInt(limit),
    tags,
  })
  
  return NextResponse.json(data)
}
```

### Работа с FormData (multipart/form-data)

```ts
export async function POST(request: NextRequest) {
  const formData = await request.formData()
  
  // Текстовые поля
  const name = formData.get('name') as string
  const email = formData.get('email') as string
  
  // Файлы
  const avatar = formData.get('avatar') as File
  if (avatar) {
    const bytes = await avatar.arrayBuffer()
    const buffer = Buffer.from(bytes)
    // Сохранение файла
    await saveFile(buffer, avatar.name)
  }
  
  // Множественные файлы
  const documents = formData.getAll('documents') as File[]
  for (const doc of documents) {
    await saveFile(doc)
  }
  
  return NextResponse.json({ success: true })
}
```

## Кэширование в Route Handlers

### Статические Route Handlers (кэширование)

Если Route Handler не использует `Request` объект и не имеет динамических данных, он кэшируется:

```ts
// КЭШИРУЕТСЯ — статический ответ
// Вызывается один раз при билде, результат кэшируется
export async function GET() {
  const data = await fetch('https://api.example.com/data')
  return NextResponse.json(data)
}
```

### Динамические Route Handlers

Использование `Request` делает handler динамическим:

```ts
// НЕ КЭШИРУЕТСЯ — динамический ответ
// Вызывается при каждом запросе
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  const page = searchParams.get('page')
  
  const data = await fetchData(page)
  return NextResponse.json(data)
}
```

### Конфигурация кэширования

```ts
// app/api/data/route.ts

// Динамический рендеринг при каждом запросе
export const dynamic = 'force-dynamic'

// Или: статический рендеринг при билде
// export const dynamic = 'force-static'

// Периодическая ревалидация (ISR)
export const revalidate = 60 // ревалидация каждые 60 секунд

export async function GET() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 }, // Кэшировать на 60 секунд
  })
  return NextResponse.json(await data.json())
}
```

### Runtime конфигурация

```ts
// Node.js runtime (по умолчанию)
export const runtime = 'nodejs'

// Edge runtime (для низкой задержки)
export const runtime = 'edge'

export async function GET() {
  // В edge runtime нет доступа к Node.js API
  // (fs, crypto, и т.д.)
  const data = await fetch('https://api.example.com/data')
  return NextResponse.json(await data.json())
}
```

## Динамические сегменты

### Параметры маршрута

```ts
// app/api/posts/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'

// Один параметр
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const post = await db.posts.findUnique({
    where: { id: params.id },
  })
  
  if (!post) {
    return NextResponse.json(
      { error: 'Post not found' },
      { status: 404 }
    )
  }
  
  return NextResponse.json(post)
}

export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const body = await request.json()
  const post = await db.posts.update({
    where: { id: params.id },
    data: body,
  })
  return NextResponse.json(post)
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  await db.posts.delete({
    where: { id: params.id },
  })
  return new NextResponse(null, { status: 204 })
}
```

### Вложенные динамические сегменты

```ts
// app/api/users/[userId]/posts/[postId]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: { userId: string; postId: string } }
) {
  const post = await db.posts.findFirst({
    where: {
      id: params.postId,
      authorId: params.userId,
    },
  })
  return NextResponse.json(post)
}
```

### Catch-all сегменты

```ts
// app/api/docs/[...slug]/route.ts
// Совпадает с: /api/docs/a, /api/docs/a/b, /api/docs/a/b/c
export async function GET(
  request: NextRequest,
  { params }: { params: { slug: string[] } }
) {
  const path = params.slug.join('/')
  const doc = await getDoc(path)
  return NextResponse.json(doc)
}

// app/api/docs/[[...slug]]/route.ts
// Optional catch-all: совпадает и с /api/docs (без slug)
export async function GET(
  request: NextRequest,
  { params }: { params: { slug?: string[] } }
) {
  const path = params.slug?.join('/') || 'index'
  const doc = await getDoc(path)
  return NextResponse.json(doc)
}
```

## CORS и заголовки

### Настройка CORS

```ts
// app/api/external/route.ts
import { NextRequest, NextResponse } from 'next/server'

const corsHeaders = {
  'Access-Control-Allow-Origin': 'https://example.com',
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
  'Access-Control-Max-Age': '86400', // 24 часа
}

// Preflight запрос
export async function OPTIONS() {
  return new NextResponse(null, {
    status: 204,
    headers: corsHeaders,
  })
}

// Основной запрос
export async function GET() {
  const data = await fetchData()
  return NextResponse.json(data, { headers: corsHeaders })
}
```

### Динамический CORS (по домену)

```ts
const allowedOrigins = ['https://app.com', 'https://admin.com']

function getCorsHeaders(origin: string | null) {
  if (origin && allowedOrigins.includes(origin)) {
    return {
      'Access-Control-Allow-Origin': origin,
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    }
  }
  return {}
}

export async function OPTIONS(request: NextRequest) {
  const origin = request.headers.get('origin')
  return new NextResponse(null, {
    status: 204,
    headers: getCorsHeaders(origin),
  })
}

export async function GET(request: NextRequest) {
  const origin = request.headers.get('origin')
  const data = await fetchData()
  return NextResponse.json(data, { headers: getCorsHeaders(origin) })
}
```

### Аутентификация через заголовки

```ts
import { NextRequest, NextResponse } from 'next/server'
import { verify } from 'jsonwebtoken'

export async function GET(request: NextRequest) {
  const authHeader = request.headers.get('authorization')
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    )
  }
  
  const token = authHeader.split(' ')[1]
  
  try {
    const payload = verify(token, process.env.JWT_SECRET!)
    const user = await db.users.findUnique({
      where: { id: payload.sub },
    })
    
    if (!user) {
      return NextResponse.json(
        { error: 'User not found' },
        { status: 404 }
      )
    }
    
    const data = await getProtectedData(user.id)
    return NextResponse.json(data)
  } catch {
    return NextResponse.json(
      { error: 'Invalid token' },
      { status: 401 }
    )
  }
}
```

## Streaming и Web Streams API

### Streaming JSON

```ts
export async function GET() {
  const encoder = new TextEncoder()
  
  const stream = new ReadableStream({
    async start(controller) {
      controller.enqueue(encoder.encode('[\n'))
      
      const items = await db.items.findMany()
      
      for (let i = 0; i < items.length; i++) {
        const item = JSON.stringify(items[i])
        const comma = i < items.length - 1 ? ',' : ''
        controller.enqueue(encoder.encode(item + comma + '\n'))
      }
      
      controller.enqueue(encoder.encode(']'))
      controller.close()
    },
  })
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'application/json',
      'Transfer-Encoding': 'chunked',
    },
  })
}
```

### Streaming из базы данных

```ts
export async function GET() {
  const cursor = db.items.findMany({ cursor: { batchSize: 100 } })
  
  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder()
      controller.enqueue(encoder.encode('['))
      
      let first = true
      for await (const item of cursor) {
        if (!first) controller.enqueue(encoder.encode(','))
        controller.enqueue(encoder.encode(JSON.stringify(item)))
        first = false
      }
      
      controller.enqueue(encoder.encode(']'))
      controller.close()
    },
  })
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'application/json',
      'Transfer-Encoding': 'chunked',
    },
  })
}
```

### Server-Sent Events (SSE)

```ts
export async function GET() {
  const encoder = new TextEncoder()
  let intervalId: NodeJS.Timeout
  
  const stream = new ReadableStream({
    start(controller) {
      // Отправка данных каждые 1 секунду
      intervalId = setInterval(() => {
        const data = JSON.stringify({
          time: new Date().toISOString(),
          value: Math.random(),
        })
        controller.enqueue(
          encoder.encode(`data: ${data}\n\n`)
        )
      }, 1000)
      
      // Закрытие через 30 секунд
      setTimeout(() => {
        clearInterval(intervalId)
        controller.close()
      }, 30000)
    },
    cancel() {
      clearInterval(intervalId)
    },
  })
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  })
}
```

## Server Actions

### Что такое Server Actions

Server Actions — это асинхронные функции, которые выполняются на сервере, но вызываются напрямую из клиентских компонентов. Они абстрагируют HTTP-слой, позволяя писать серверный код как обычную функцию.

```
┌─────────────────────────────────────────────────────────────┐
│                    Server Action Flow                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client Component                                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ <form action={createPost}>                           │   │
│  │   <input name="title" />                             │   │
│  │   <button>Submit</button>                            │   │
│  │ </form>                                              │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          │ Hidden POST request               │
│                          │ (автоматическая сериализация)     │
│                          ▼                                   │
│  Server                                                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 'use server'                                         │   │
│  │ export async function createPost(formData: FormData) │   │
│  │   const title = formData.get('title')                │   │
│  │   await db.posts.create({ data: { title } })         │   │
│  │   revalidatePath('/posts')                           │   │
│  │ }                                                    │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          │ Response (re-render)              │
│                          ▼                                   │
│  Browser обновляет UI                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Создание Server Actions

#### Inline в компоненте

```tsx
// app/posts/create/page.tsx
import { revalidatePath } from 'next/cache'

export default function CreatePostPage() {
  async function createPost(formData: FormData) {
    'use server'
    
    const title = formData.get('title') as string
    const content = formData.get('content') as string
    
    await db.posts.create({
      data: { title, content },
    })
    
    revalidatePath('/posts')
  }
  
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" />
      <textarea name="content" placeholder="Content" />
      <button type="submit">Create Post</button>
    </form>
  )
}
```

#### Отдельный файл

```ts
// app/actions/posts.ts
'use server'

import { revalidatePath } from 'next/cache'
import { z } from 'zod'

const CreatePostSchema = z.object({
  title: z.string().min(3).max(100),
  content: z.string().min(10),
})

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string
  
  // Валидация
  const validated = CreatePostSchema.parse({ title, content })
  
  // Создание
  const post = await db.posts.create({
    data: validated,
  })
  
  // Ревайлидация кэша
  revalidatePath('/posts')
  revalidatePath('/', 'layout')
  
  return post
}

export async function deletePost(postId: string) {
  await db.posts.delete({ where: { id: postId } })
  revalidatePath('/posts')
}

export async function updatePost(postId: string, formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string
  
  await db.posts.update({
    where: { id: postId },
    data: { title, content },
  })
  
  revalidatePath(`/posts/${postId}`)
}
```

### Вызов Server Actions

#### Из формы

```tsx
import { createPost } from '@/app/actions/posts'

export default function CreatePostForm() {
  return (
    <form action={createPost}>
      <input name="title" />
      <textarea name="content" />
      <button type="submit">Submit</button>
    </form>
  )
}
```

#### Из обработчика события

```tsx
'use client'

import { createPost } from '@/app/actions/posts'

export function PostButton() {
  async function handleClick() {
    const formData = new FormData()
    formData.set('title', 'My Post')
    formData.set('content', 'Content here')
    
    const result = await createPost(formData)
    console.log('Created:', result)
  }
  
  return <button onClick={handleClick}>Create Post</button>
}
```

#### С аргументами (bind)

```tsx
import { deletePost } from '@/app/actions/posts'

export function PostList({ posts }: { posts: Post[] }) {
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>
          <span>{post.title}</span>
          <form action={deletePost.bind(null, post.id)}>
            <button type="submit">Delete</button>
          </form>
        </li>
      ))}
    </ul>
  )
}
```

## useFormState и useFormStatus

### useFormState (React 19+)

Хук для управления состоянием формы при использовании Server Actions:

```tsx
'use client'

import { useFormState } from 'react-dom'
import { createPost } from '@/app/actions/posts'

// Начальное состояние
const initialState = {
  error: null as string | null,
  success: false,
}

export default function CreatePostForm() {
  const [state, formAction] = useFormState(createPostWithState, initialState)
  
  return (
    <form action={formAction}>
      <input name="title" />
      <textarea name="content" />
      
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Post created!</p>}
      
      <button type="submit">Create</button>
    </form>
  )
}

// Action с состоянием
async function createPostWithState(
  prevState: typeof initialState,
  formData: FormData
) {
  'use server'
  
  const title = formData.get('title') as string
  const content = formData.get('content') as string
  
  if (title.length < 3) {
    return { error: 'Title too short', success: false }
  }
  
  try {
    await db.posts.create({ data: { title, content } })
    revalidatePath('/posts')
    return { error: null, success: true }
  } catch {
    return { error: 'Failed to create post', success: false }
  }
}
```

### useFormStatus

Хук для получения статуса отправки формы:

```tsx
'use client'

import { useFormStatus } from 'react-dom'

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus()
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  )
}

export default function CreatePostForm() {
  return (
    <form action={createPost}>
      <input name="title" />
      <SubmitButton /> {/* Внутри <form> */}
    </form>
  )
}
```

### Комбинированное использование

```tsx
'use client'

import { useFormState, useFormStatus } from 'react-dom'
import { createPost } from '@/app/actions/posts'

function SubmitButton() {
  const { pending } = useFormStatus()
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Creating...' : 'Create Post'}
    </button>
  )
}

export default function CreatePostForm() {
  const [state, formAction] = useFormState(createPostWithState, {
    error: null,
    success: false,
  })
  
  return (
    <form action={formAction}>
      <input name="title" disabled={state.pending} />
      <textarea name="content" disabled={state.pending} />
      
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Created!</p>}
      
      <SubmitButton />
    </form>
  )
}
```

## Revalidation и мутации

### revalidatePath

```ts
import { revalidatePath } from 'next/cache'

export async function updatePost(formData: FormData) {
  'use server'
  
  await db.posts.update({ ... })
  
  // Ревайлидация конкретного пути
  revalidatePath('/posts/123')
  
  // Ревайлидация всех путей под паттерном
  revalidatePath('/posts')
  
  // Ревайлидация всего сайта
  revalidatePath('/', 'layout')
}
```

### revalidateTag

```ts
export async function GET() {
  const data = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] },
  })
  return NextResponse.json(await data.json())
}

// Server Action для ревайлидации
export async function invalidatePosts() {
  'use server'
  revalidateTag('posts')
}
```

### revalidateQueryKey (TanStack Query)

```ts
'use client'

import { useQueryClient } from '@tanstack/react-query'
import { useFormState } from 'react-dom'

export function PostForm() {
  const queryClient = useQueryClient()
  
  async function createPostWithInvalidate(
    prevState: any,
    formData: FormData
  ) {
    'use server'
    const result = await createPost(formData)
    return result
  }
  
  const [state, formAction] = useFormState(createPostWithInvalidate, {})
  
  // После успешной мутации — инвалидация кэша
  useEffect(() => {
    if (state.success) {
      queryClient.invalidateQueries({ queryKey: ['posts'] })
    }
  }, [state.success, queryClient])
  
  return <form action={formAction}>...</form>
}
```

## Оптимистичные обновления

### useOptimistic

```tsx
'use client'

import { useOptimistic } from 'react'
import { addTodo } from '@/app/actions/todos'

type Todo = { id: string; text: string; completed: boolean }

export function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo: string) => [
      ...state,
      { id: 'temp', text: newTodo, completed: false },
    ]
  )
  
  async function handleSubmit(formData: FormData) {
    const text = formData.get('text') as string
    addOptimistic(text) // Мгновенное обновление UI
    await addTodo(formData) // Серверная мутация
  }
  
  return (
    <>
      <form action={handleSubmit}>
        <input name="text" />
        <button>Add</button>
      </form>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </>
  )
}
```

## Server Actions под капотом

### Как это работает

```
1. Компиляция:
   'use server' → Next.js создаёт server reference
   Функция сериализуется и регистрируется в runtime

2. Клиентский вызов:
   Action → fetch POST на специальный endpoint
   Тело: { actionId, args (сериализованные) }

3. Серверная обработка:
   Next.js получает POST → находит action по ID →
   Десериализует аргументы → выполняет функцию →
   Сериализует результат → возвращает клиенту

4. Клиентский ответ:
   Получает результат → обновляет React state →
   Re-render компонента
```

### Сериализация аргументов

Server Actions поддерживают сериализацию:
- Примитивы: string, number, boolean, null, undefined
- Объекты и массивы (JSON-сериализуемые)
- Date, RegExp, Map, Set, BigInt
- FormData, File, Blob
- React элементы (jsx)

НЕ поддерживаются:
- Функции
- Class instances (кроме встроенных)
- Symbol
- Circular references

## Route Handlers vs Server Actions

### Ключевые различия

| Аспект | Route Handlers | Server Actions |
|--------|----------------|----------------|
| **Назначение** | REST API endpoints | Мутации данных из UI |
| **HTTP-контроль** | Полный (методы, заголовки, статусы) | Абстрагирован (только POST) |
| **Вызов** | HTTP-запрос (fetch, axios) | Прямой вызов функции |
| **Типизация** | Ручная (Request/Response) | Автоматическая (функция) |
| **Кэширование** | Встроенное (static/dynamic) | Нет (всегда динамические) |
| **Streaming** | Поддерживается | Не поддерживается |
| **CORS** | Ручная настройка | Не нужно (тот же origin) |
| **Формат данных** | JSON, FormData, любой | FormData или аргументы |
| **Внешние вызовы** | Да (из других сервисов) | Нет (только из приложения) |
| **Webhooks** | Да | Нет |
| **Оптимистичные обновления** | Вручную (через state) | useOptimistic |
| **Revalidation** | Вручную (revalidatePath/Tag) | Автоматическая (revalidatePath) |
| **Progressive Enhancement** | Нет | Да (работает без JS) |
| **Runtime** | Node.js или Edge | Только Node.js |
| **Сложность** | Выше (нужно обрабатывать HTTP) | Ниже (как обычная функция) |

### Архитектурные различия

```
Route Handler:
┌─────────────┐     HTTP      ┌─────────────┐
│   Client    │ ────────────→ │   Route     │
│  (Browser)  │ ←──────────── │  Handler    │
└─────────────┘   Response    └─────────────┘
                       ↑
               Полное управление
               методом, заголовками,
               статусом, кэшем

Server Action:
┌─────────────┐   RPC Call   ┌─────────────┐
│   Client    │ ────────────→ │   Server    │
│  Component  │ ←──────────── │   Action    │
└─────────────┘   Result      └─────────────┘
                       ↑
               Простой вызов функции
               без HTTP-деталей
```

## Когда что использовать

### Используйте Route Handlers когда:

1. **Публичный REST API**
```ts
// app/api/v1/users/route.ts
export async function GET() {
  // API для внешних клиентов
  return NextResponse.json(await getUsers())
}
```

2. **Webhooks**
```ts
// app/api/webhooks/stripe/route.ts
export async function POST(request: NextRequest) {
  const signature = request.headers.get('stripe-signature')
  const body = await request.text()
  
  const event = stripe.webhooks.constructEvent(
    body,
    signature!,
    process.env.STRIPE_WEBHOOK_SECRET!
  )
  
  await handleWebhookEvent(event)
  return new NextResponse(null, { status: 200 })
}
```

3. **Кастомная логика кэширования**
```ts
export const revalidate = 3600 // Кэшировать на час
export async function GET() {
  return NextResponse.json(await getExpensiveData())
}
```

4. **Streaming ответы**
```ts
export async function GET() {
  const stream = createReadStream()
  return new Response(stream)
}
```

5. **CORS для внешних доменов**
```ts
export async function GET(request: NextRequest) {
  const origin = request.headers.get('origin')
  return NextResponse.json(data, {
    headers: { 'Access-Control-Allow-Origin': origin },
  })
}
```

### Используйте Server Actions когда:

1. **Формы и мутации**
```tsx
<form action={createPost}>
  <input name="title" />
  <button>Submit</button>
</form>
```

2. **Прогрессивное улучшение**
```tsx
// Работает даже без JavaScript
<form action={addToCart}>
  <button name="productId" value={product.id}>
    Add to Cart
  </button>
</form>
```

3. **Простые CRUD операции**
```tsx
async function deleteItem(id: string) {
  'use server'
  await db.items.delete({ where: { id } })
  revalidatePath('/items')
}
```

4. **Оптимистичные обновления**
```tsx
const [optimistic, addOptimistic] = useOptimistic(items)
async function addItem(formData: FormData) {
  addOptimistic(newItem)
  await createItem(formData)
}
```

5. **Валидация на сервере**
```tsx
async function submitForm(prevState: any, formData: FormData) {
  'use server'
  const validated = Schema.parse(Object.fromEntries(formData))
  await saveData(validated)
  return { success: true }
}
```

## Практические паттерны

### Паттерн: CRUD API

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server'

// GET /api/posts
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  const page = parseInt(searchParams.get('page') || '1')
  const limit = parseInt(searchParams.get('limit') || '10')
  
  const posts = await db.posts.findMany({
    skip: (page - 1) * limit,
    take: limit,
    orderBy: { createdAt: 'desc' },
  })
  
  const total = await db.posts.count()
  
  return NextResponse.json({
    data: posts,
    meta: { page, limit, total, pages: Math.ceil(total / limit) },
  })
}

// POST /api/posts
export async function POST(request: NextRequest) {
  const body = await request.json()
  const post = await db.posts.create({ data: body })
  return NextResponse.json(post, { status: 201 })
}

// app/api/posts/[id]/route.ts
// GET /api/posts/:id
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const post = await db.posts.findUnique({ where: { id: params.id } })
  if (!post) return NextResponse.json({ error: 'Not found' }, { status: 404 })
  return NextResponse.json(post)
}

// PUT /api/posts/:id
export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const body = await request.json()
  const post = await db.posts.update({ where: { id: params.id }, data: body })
  return NextResponse.json(post)
}

// DELETE /api/posts/:id
export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  await db.posts.delete({ where: { id: params.id } })
  return new NextResponse(null, { status: 204 })
}
```

### Паттерн: Аутентификация через middleware

```ts
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token')?.value
  
  if (!token && request.nextUrl.pathname.startsWith('/api/protected')) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
  
  return NextResponse.next()
}

export const config = {
  matcher: '/api/protected/:path*',
}
```

### Паттерн: Rate Limiting

```ts
// lib/rateLimit.ts
const rateLimitMap = new Map<string, { count: number; resetTime: number }>()

export function rateLimit(ip: string, limit = 100, window = 60000) {
  const now = Date.now()
  const record = rateLimitMap.get(ip)
  
  if (!record || now > record.resetTime) {
    rateLimitMap.set(ip, { count: 1, resetTime: now + window })
    return true
  }
  
  if (record.count >= limit) {
    return false
  }
  
  record.count++
  return true
}

// app/api/route.ts
export async function GET(request: NextRequest) {
  const ip = request.ip || 'unknown'
  
  if (!rateLimit(ip)) {
    return NextResponse.json(
      { error: 'Too many requests' },
      { status: 429 }
    )
  }
  
  return NextResponse.json({ data: 'value' })
}
```

### Паттерн: Server Action с валидацией

```ts
// app/actions/safe-action.ts
'use server'

import { z } from 'zod'

export async function safeAction<TSchema extends z.ZodType, TResult>(
  schema: TSchema,
  data: unknown,
  handler: (validated: z.infer<TSchema>) => Promise<TResult>
): Promise<{ success: true; data: TResult } | { success: false; error: string }> {
  const result = schema.safeParse(data)
  
  if (!result.success) {
    return { success: false, error: result.error.errors[0].message }
  }
  
  try {
    const data = await handler(result.data)
    return { success: true, data }
  } catch {
    return { success: false, error: 'Internal server error' }
  }
}

// Использование
const CreateUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
})

export async function createUser(formData: FormData) {
  'use server'
  
  return safeAction(
    CreateUserSchema,
    {
      name: formData.get('name'),
      email: formData.get('email'),
    },
    async (validated) => {
      const user = await db.users.create({ data: validated })
      revalidatePath('/users')
      return user
    }
  )
}
```

## Best Practices

### Route Handlers

1. **Всегда валидируйте входные данные**
```ts
const body = await request.json()
const validated = Schema.parse(body) // Zod/Yup
```

2. **Используйте правильные HTTP-статусы**
```ts
200 — OK
201 — Created
204 — No Content
400 — Bad Request
401 — Unauthorized
403 — Forbidden
404 — Not Found
429 — Too Many Requests
500 — Internal Server Error
```

3. **Логируйте ошибки**
```ts
catch (error) {
  console.error('API Error:', error)
  return NextResponse.json({ error: 'Internal error' }, { status: 500 })
}
```

4. **Используйте Edge Runtime для простых handlers**
```ts
export const runtime = 'edge'
```

### Server Actions

1. **Всегда ревайлидируйте кэш**
```ts
await db.posts.create({ data })
revalidatePath('/posts') // Обязательно!
```

2. **Обрабатывайте ошибки**
```ts
try {
  await db.posts.create({ data })
} catch (error) {
  return { error: 'Failed to create post' }
}
```

3. **Используйте useFormState для UX**
```tsx
const [state, formAction] = useFormState(action, initialState)
// Показывайте ошибки и статусы
```

4. **Валидируйте на сервере**
```ts
const validated = Schema.parse(Object.fromEntries(formData))
// Никогда не доверяйте клиентским данным
```

## Антипаттерны

### Route Handlers

1. **Не экспонируйте чувствительные данные**
```ts
// ПЛОХО
return NextResponse.json(user) // Включает password hash

// ХОРОШО
const { password, ...safeUser } = user
return NextResponse.json(safeUser)
```

2. **Не забывайте про CORS**
```ts
// ПЛОХО — нет CORS заголовков
return NextResponse.json(data)

// ХОРОШО — явный CORS
return NextResponse.json(data, {
  headers: { 'Access-Control-Allow-Origin': 'https://app.com' }
})
```

### Server Actions

1. **Не вызывайте Actions из других Actions**
```ts
// ПЛОХО
async function action1() {
  'use server'
  await action2() // Избегайте вложенности
}

// ХОРОШО — вынесите общую логику
async function sharedLogic() { ... }
async function action1() { await sharedLogic() }
async function action2() { await sharedLogic() }
```

2. **Не забывайте про revalidatePath**
```ts
// ПЛОХО
await db.posts.create({ data })
// Кэш не обновится, UI устареет

// ХОРОШО
await db.posts.create({ data })
revalidatePath('/posts')
```

## Сравнение с Nuxt.js

### Nuxt.js: Server Routes

```ts
// server/api/posts.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event) // ?page=1
  const posts = await db.posts.findMany({
    skip: (Number(query.page) - 1) * 10,
    take: 10,
  })
  return posts
})

// server/api/posts.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const post = await db.posts.create({ data: body })
  return post
})
```

### Nuxt.js: Server Actions (через Nuxt Utils)

```ts
// composables/usePosts.ts
export const usePosts = () => {
  const createPost = async (data: CreatePostDto) => {
    return await $fetch('/api/posts', {
      method: 'POST',
      body: data,
    })
  }
  
  return { createPost }
}
```

### Ключевые отличия Next.js vs Nuxt.js

| Аспект | Next.js | Nuxt.js |
|--------|---------|---------|
| **Route Handlers** | `route.ts` с HTTP-методами | `server/api/` с HTTP-методами в имени файла |
| **Server Actions** | Встроенные `'use server'` | Нет встроенных (используют `$fetch`) |
| **Мутации** | Server Actions или Route Handlers | Server Routes + `$fetch` |
| **Кэширование** | `revalidatePath`, `revalidateTag` | `useFetch` с кэшированием |
| **Валидация** | Ручная (Zod) | `zod` + `useSafeFetch` |
| **Streaming** | ReadableStream | `useAsyncData` с `transform` |
| **Runtime** | Node.js или Edge | Node.js (Nitro) |
| **Типизация** | Ручная | Автоматическая (auto-imports) |

### Когда что использовать в Next.js

**Route Handlers:**
- Публичные API
- Webhooks
- Кастомная логика кэширования
- Streaming
- Внешние интеграции

**Server Actions:**
- Формы
- Мутации данных
- Прогрессивное улучшение
- Оптимистичные обновления
- Простые CRUD

### Итоговая рекомендация

Используйте **Route Handlers** для всего, что требует полного HTTP-контроля (API, webhooks, streaming). Используйте **Server Actions** для мутаций из UI (формы, кнопки) — они проще, безопаснее и обеспечивают лучший UX.
