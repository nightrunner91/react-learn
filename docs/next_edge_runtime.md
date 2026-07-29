# Edge Runtime в Next.js

## Что такое Edge Runtime

Edge Runtime — это среда выполнения JavaScript, оптимизированная для низкой задержки (latency) и быстрого запуска. В отличие от традиционного Node.js сервера, Edge функции выполняются на "edge" — ближе к пользователю, на CDN-серверах по всему миру.

### Ключевые отличия от Node.js

**Node.js Runtime:**
- Полноценный серверный runtime
- Доступ ко всей файловой системе
- Поддержка всех Node.js API
- Запускается на одном или нескольких серверах
- Время запуска: ~100-500ms
- Холодный старт: заметная задержка

**Edge Runtime:**
- Легковесный runtime на основе V8 Isolate
- Ограниченный доступ к API (Web API только)
- Нет файловой системы
- Запускается на CDN по всему миру
- Время запуска: ~5-50ms
- Холодный старт: минимальный

## Как работает Edge Runtime

### Архитектура

```
┌─────────────────────────────────────────────────────┐
│                    Пользователь                      │
│                  (Москва, Россия)                    │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────┐
        │   Edge Location (CDN)    │
        │   Москва / Санкт-Петербург│
        │                          │
        │  ┌────────────────────┐  │
        │  │  Edge Function     │  │
        │  │  (V8 Isolate)      │  │
        │  │  - Быстрый запуск  │  │
        │  │  - Минимум памяти  │  │
        │  └────────────────────┘  │
        └──────────────────────────┘
```

### V8 Isolate

Edge Runtime использует V8 Isolates — легковесные изолированные среды выполнения JavaScript:

- **Быстрый запуск** — не нужно загружать весь Node.js runtime
- **Минимум памяти** — каждый isolate занимает ~1-5MB (vs ~50-100MB для Node.js)
- **Параллелизм** — тысячи isolates на одном сервере
- **Безопасность** — изолированы друг от друга

## Когда использовать Edge Runtime

### Подходит для

**1. Middleware**
```tsx
// middleware.ts
export const config = {
  matcher: ['/((?!api|_next/static|favicon.ico).*)'],
}

export function middleware(request) {
  // Геолокация
  const country = request.geo?.country || 'US'
  
  // A/B тестирование
  const variant = Math.random() > 0.5 ? 'A' : 'B'
  
  // Аутентификация
  const token = request.cookies.get('token')
  if (!token) {
    return NextResponse.redirect('/login')
  }
  
  return NextResponse.next()
}
```

**2. API Routes с низкой задержкой**
```tsx
// app/api/geo/route.ts
export const runtime = 'edge'

export async function GET(request: Request) {
  const country = request.geo?.country
  const city = request.geo?.city
  
  return Response.json({ country, city })
}
```

**3. Простые страницы**
```tsx
// app/page.tsx
export const runtime = 'edge'

export default function Page() {
  return <h1>Быстрая страница</h1>
}
```

**4. A/B тестирование**
```tsx
export const runtime = 'edge'

export async function GET(request) {
  const bucket = Math.random() > 0.5 ? 'control' : 'variant'
  
  return new Response(JSON.stringify({ bucket }), {
    headers: {
      'Content-Type': 'application/json',
      'Set-Cookie': `ab-test=${bucket}; Path=/`,
    },
  })
}
```

**5. Геолокация и локализация**
```tsx
export const runtime = 'edge'

export async function GET(request) {
  const country = request.geo?.country
  const locale = country === 'RU' ? 'ru' : 'en'
  
  const content = await getLocalizedContent(locale)
  
  return Response.json(content)
}
```

### НЕ подходит для

**1. Тяжелые вычисления**
```tsx
// Плохо — Edge не для этого
export const runtime = 'edge'

export async function POST(request) {
  const data = await request.json()
  
  // Тяжелая обработка изображений
  const processed = await sharp(data.image)
    .resize(1000, 1000)
    .toBuffer()
  
  return Response.json({ processed })
}
```

**2. Работа с файловой системой**
```tsx
// Плохо — нет fs в Edge
export const runtime = 'edge'

export async function GET() {
  const file = await fs.readFile('./data.json') // Ошибка!
  return Response.json(JSON.parse(file))
}
```

**3. Большие зависимости**
```tsx
// Плохо — большие бандлы не поддерживаются
export const runtime = 'edge'

import { PrismaClient } from '@prisma/client' // Слишком большой

export async function GET() {
  const prisma = new PrismaClient()
  const users = await prisma.user.findMany()
  return Response.json(users)
}
```

**4. Долгие операции**
```tsx
// Плохо — Edge имеет лимиты на время выполнения
export const runtime = 'edge'

export async function POST() {
  // Генерация видео может занять минуты
  await generateVideo() // Timeout!
  
  return Response.json({ done: true })
}
```

## Ограничения Edge Runtime

### Поддерживаемые API

**Web API (поддерживаются):**
- `fetch` — HTTP запросы
- `Request`, `Response` — работа с HTTP
- `URL`, `URLSearchParams` — парсинг URL
- `Headers` — HTTP заголовки
- `crypto` — Web Crypto API
- `TextEncoder`, `TextDecoder` — работа с текстом
- `setTimeout`, `setInterval` — таймеры (с ограничениями)

**Node.js API (НЕ поддерживаются):**
- `fs` — файловая система
- `child_process` — дочерние процессы
- `cluster` — кластеризация
- `net`, `tls` — сетевые сокеты
- `process` — частично (только `process.env`)

### Лимиты

**Vercel Edge Functions:**
- Размер бандла: до 4MB (сжатый)
- Время выполнения: до 30 секунд
- Память: до 128MB
- Количество запросов: зависит от плана

**Cloudflare Workers:**
- Размер скрипта: до 1MB (free), до 5MB (paid)
- Время выполнения: до 30 секунд (free), до 50ms CPU (paid)
- Память: до 128MB

## Конфигурация

### Route Handlers

```tsx
// app/api/data/route.ts
export const runtime = 'edge' // 'nodejs' | 'edge'

export async function GET(request: Request) {
  return Response.json({ data: 'Hello from Edge' })
}
```

### Pages

```tsx
// app/page.tsx
export const runtime = 'edge'

export default function Page() {
  return <div>Edge Page</div>
}
```

### Middleware

```tsx
// middleware.ts
export const config = {
  matcher: ['/((?!api|_next/static|favicon.ico).*)'],
}

export function middleware(request) {
  return NextResponse.next()
}
```

## Практические примеры

### Аутентификация через Middleware

```tsx
// middleware.ts
import { NextResponse } from 'next/server'
import { jwtVerify } from 'jose'

export async function middleware(request) {
  const token = request.cookies.get('auth-token')?.value
  
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  
  try {
    const secret = new TextEncoder().encode(process.env.JWT_SECRET!)
    await jwtVerify(token, secret)
    
    return NextResponse.next()
  } catch (error) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
}

export const config = {
  matcher: ['/dashboard/:path*', '/profile/:path*'],
}
```

### Геолокация и редирект

```tsx
// middleware.ts
import { NextResponse } from 'next/server'

export function middleware(request) {
  const country = request.geo?.country || 'US'
  
  // Редирект для конкретных стран
  if (country === 'RU') {
    return NextResponse.redirect(new URL('/ru', request.url))
  }
  
  if (country === 'CN') {
    return NextResponse.redirect(new URL('/zh', request.url))
  }
  
  return NextResponse.next()
}
```

### Feature Flags

```tsx
// middleware.ts
import { NextResponse } from 'next/server'

export function middleware(request) {
  const userId = request.cookies.get('user-id')?.value
  
  // Простая логика feature flags
  const enableNewUI = userId && parseInt(userId) % 10 < 3 // 30% пользователей
  
  const response = NextResponse.next()
  response.headers.set('x-feature-new-ui', enableNewUI ? 'true' : 'false')
  
  return response
}
```

### Rate Limiting

```tsx
// middleware.ts
import { NextResponse } from 'next/server'

const rateLimit = new Map<string, { count: number; resetAt: number }>()

export function middleware(request) {
  const ip = request.ip || 'unknown'
  const now = Date.now()
  
  let record = rateLimit.get(ip)
  
  if (!record || now > record.resetAt) {
    record = { count: 0, resetAt: now + 60000 } // 1 минута
    rateLimit.set(ip, record)
  }
  
  record.count++
  
  if (record.count > 100) {
    return new NextResponse('Too Many Requests', { status: 429 })
  }
  
  return NextResponse.next()
}
```

### A/B тестирование

```tsx
// middleware.ts
import { NextResponse } from 'next/server'

export function middleware(request) {
  let bucket = request.cookies.get('ab-bucket')?.value
  
  if (!bucket) {
    bucket = Math.random() > 0.5 ? 'control' : 'variant'
    
    const response = NextResponse.next()
    response.cookies.set('ab-bucket', bucket, {
      maxAge: 60 * 60 * 24 * 30, // 30 дней
    })
    
    return response
  }
  
  return NextResponse.next()
}
```

## Edge vs Serverless

### Edge Functions

**Плюсы:**
- Мгновенный запуск (~5ms)
- Глобальное распределение
- Низкая задержка
- Масштабирование до тысяч запросов

**Минусы:**
- Ограниченный runtime
- Лимиты на размер и время
- Нет доступа к Node.js API

### Serverless Functions (Node.js)

**Плюсы:**
- Полный Node.js runtime
- Нет ограничений на зависимости
- Доступ ко всем API

**Минусы:**
- Холодный старт (~100-500ms)
- Задержка от региона
- Ограниченное количество инстансов

### Когда что использовать

**Edge Runtime:**
- Middleware
- Простые API endpoints
- Геолокация
- A/B тестирование
- Аутентификация
- Редиректы

**Serverless (Node.js):**
- Сложная бизнес-логика
- Работа с БД
- Обработка файлов
- Тяжелые вычисления
- Интеграции с внешними сервисами

## Оптимизация Edge функций

### Минимизируйте размер бандла

```tsx
// Плохо
import { PrismaClient } from '@prisma/client'

// Хорошо — используйте HTTP клиент
export const runtime = 'edge'

export async function GET() {
  const users = await fetch('https://api.example.com/users').then(r => r.json())
  return Response.json(users)
}
```

### Используйте кэширование

```tsx
export const runtime = 'edge'

export async function GET() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 3600 }, // Кэш на 1 час
  })
  
  return Response.json(await data.json())
}
```

### Избегайте больших зависимостей

```tsx
// Вместо moment.js
import { format } from 'date-fns'

// Вместо lodash
import { debounce } from 'lodash-es'
```

## Deployment

### Vercel

Edge функции автоматически деплоятся на Vercel Edge Network:

```bash
vercel deploy
```

### Cloudflare Workers

```bash
npm install @cloudflare/next-on-pages
npx @cloudflare/next-on-pages
```

### Self-hosted

Для self-hosted deployments используйте адаптеры:
- `@edge-runtime/next-server`
- Cloudflare Workers
- Deno Deploy

## Лучшие практики

1. **Используйте Edge для middleware** — минимальная задержка
2. **Минимизируйте зависимости** — меньше размер бандла
3. **Кэшируйте ответы** — используйте `Cache-Control`
4. **Обрабатывайте ошибки** — Edge функции могут падать
5. **Тестируйте локально** — `next dev` поддерживает Edge runtime
6. **Мониторьте метрики** — следите за временем выполнения
7. **Не используйте для сложной логики** — Edge не для этого

## Заключение

Edge Runtime — мощный инструмент для оптимизации задержки и масштабирования. Используйте его для middleware, простых API, геолокации и A/B тестирования. Для сложной бизнес-логики и работы с БД оставляйте Node.js runtime.

Помните: Edge — это не замена Node.js, а дополнение для специфических задач, где важна низкая задержка и глобальное распределение.
