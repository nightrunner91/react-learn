# Деплой Next.js

## Содержание

1. [Что происходит при `next build`](#что-происходит-при-next-build)
2. [Режимы вывода](#режимы-вывода)
3. [Деплой статического сайта](#деплой-статического-сайта)
4. [Деплой SSR и ISR](#деплой-ssr-и-isr)
5. [Edge Runtime](#edge-runtime)
6. [Image Optimization](#image-optimization)
7. [Environment variables](#environment-variables)
8. [Деплой на Vercel](#деплой-на-vercel)
9. [Деплой в Docker](#деплой-в-docker)
10. [Деплой на других платформах](#деплой-на-других-платформах)
11. [Чеклист перед деплоем](#чеклист-перед-деплоем)
12. [Заключение](#заключение)

---

## Что происходит при `next build`

Команда `next build` превращает исходный код в production-артефакты. Процесс состоит из нескольких этапов:

1. **Компиляция.** TypeScript, JSX, CSS превращаются в оптимизированный JS.
2. **Сборка бандлов.** Клиентский и серверный бандлы разделяются.
3. **Генерация статических страниц.** Страницы с `dynamic = 'force-static'` или без динамических данных рендерятся в HTML.
4. **Подготовка serverless-функций.** Динамические маршруты, API Routes, Route Handlers упаковываются в функции.
5. **Оптимизация изображений.** Подготавливается инфраструктура для `next/image`.
6. **Создание `.next/`.** Все артефакты попадают в эту папку.

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   next build    │ ───► │  Компиляция и   │ ───► │  Артефакты в   │
│                 │      │  оптимизация    │      │  .next/         │
└─────────────────┘      └─────────────────┘      └─────────────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
    │  Статические │      │  Serverless │      │  Клиентский │
    │  HTML-файлы  │      │  функции    │      │  JS-бандл   │
    │  (.html)     │      │  для SSR    │      │  (.js)      │
    └─────────────┘      └─────────────┘      └─────────────┘
```

---

## Режимы вывода

Next.js поддерживает три режима вывода, которые определяют, как приложение можно запускать в production.

### По умолчанию

```js
// next.config.js
module.exports = {}
```

- Статические страницы раздаются как файлы.
- Динамические страницы и API Routes выполняются как serverless-функции.
- Подходит для Vercel, Netlify, Railway, Docker.

### `output: 'standalone'`

```js
// next.config.js
module.exports = {
  output: 'standalone',
}
```

- Создаёт минимальный production-сервер с `server.js` в корне `.next/standalone/`.
- Включает только те зависимости, которые нужны для работы приложения.
- Идеально для Docker и self-hosted деплоя.

```dockerfile
# Dockerfile с standalone
FROM node:20-alpine
WORKDIR /app

COPY .next/standalone ./
COPY .next/static ./.next/static
COPY public ./public

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

### `output: 'export'`

```js
// next.config.js
module.exports = {
  output: 'export',
}
```

- Всё приложение превращается в статические HTML, CSS, JS файлы.
- Папка `out/` готова к загрузке на любой статический хостинг.
- Не работают: SSR, ISR, API Routes, Middleware, `next/image` с оптимизацией.

**Когда использовать:** блоги, лендинги, документация.

---

## Деплой статического сайта

### Настройка

```js
// next.config.js
module.exports = {
  output: 'export',
  images: {
    unoptimized: true, // Обязательно для static export
  },
}
```

### Динамические маршруты

Для каждого динамического маршрута нужна `generateStaticParams`:

```tsx
// app/posts/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json())

  return posts.map(post => ({
    slug: post.slug,
  }))
}

export default async function Page({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug)
  return <article>{post.content}</article>
}
```

### Куда деплоить

- **Vercel** — автоматически распознаёт static export.
- **Netlify** — drag-and-drop папку `out/`.
- **GitHub Pages** — через GitHub Actions.
- **Cloudflare Pages** — загрузка папки `out/`.
- **AWS S3 + CloudFront** — классическая схема.

---

## Деплой SSR и ISR

### SSR

Для принудительного SSR на странице:

```tsx
// app/page.tsx
export const dynamic = 'force-dynamic'

export default async function Page() {
  const response = await fetch('https://api.example.com/data', {
    cache: 'no-store',
  })
  const data = await response.json()

  return <div>{data.title}</div>
}
```

### ISR

```tsx
// app/page.tsx
export const revalidate = 60

export default async function Page() {
  const response = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 },
  })
  const data = await response.json()

  return <div>{data.title}</div>
}
```

### On-Demand Revalidation

```tsx
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache'

export async function POST(request: Request) {
  const { path, tag } = await request.json()

  if (path) revalidatePath(path)
  if (tag) revalidateTag(tag)

  return Response.json({ revalidated: true })
}
```

### Где работает ISR

| Платформа | Поддержка ISR | Примечание |
|-----------|---------------|------------|
| Vercel | Да | Работает из коробки |
| Netlify | Да | Через Next.js Runtime |
| Cloudflare Pages | Частично | Через `@cloudflare/next-on-pages` |
| Self-hosted | Нужна настройка | Требуется сервер с поддержкой фоновых задач |
| Static Export | Нет | ISR невозможен без сервера |

---

## Edge Runtime

Edge Runtime позволяет запускать страницы и API-роуты на CDN-узлах.

```tsx
// app/api/geo/route.ts
export const runtime = 'edge'

export async function GET(request: Request) {
  const country = request.geo?.country
  const city = request.geo?.city

  return Response.json({ country, city })
}
```

### Middleware

Middleware в Next.js всегда выполняется на Edge:

```tsx
// middleware.ts
import { NextResponse } from 'next/server'

export function middleware(request: Request) {
  const token = request.cookies.get('token')

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/dashboard/:path*'],
}
```

### Ограничения Edge Runtime

- Нет Node.js API: `fs`, `child_process`, `net`.
- Нет больших зависимостей вроде Prisma.
- Ограничения по памяти и времени выполнения.

---

## Image Optimization

`next/image` по умолчанию оптимизирует изображения на сервере. Это работает только на платформах с серверной частью.

### Для static export

```js
// next.config.js
module.exports = {
  output: 'export',
  images: {
    unoptimized: true,
  },
}
```

### Custom loader

```js
// next.config.js
module.exports = {
  images: {
    loader: 'custom',
    loaderFile: './app/image-loader.ts',
  },
}
```

```ts
// app/image-loader.ts
export default function myImageLoader({ src, width, quality }: {
  src: string
  width: number
  quality?: number
}) {
  return `https://cdn.example.com/${src}?w=${width}&q=${quality || 75}`
}
```

### Популярные loaders

| Loader | Сервис |
|--------|--------|
| `default` | Сервер Next.js |
| `imgix` | Imgix |
| `cloudinary` | Cloudinary |
| `akamai` | Akamai Image Manager |
| `custom` | Любой CDN |

---

## Environment variables

### Build-time vs Runtime

| Префикс | Когда доступна | Где используется |
|---------|---------------|------------------|
| Без префикса | Только на сервере | Серверные компоненты, API |
| `NEXT_PUBLIC_` | Во время сборки и в браузере | Клиентские компоненты |

### Пример

```env
# .env.local
DATABASE_URL=postgresql://...
API_SECRET=sk-...
NEXT_PUBLIC_API_URL=https://api.example.com
```

### Важно

- `NEXT_PUBLIC_` переменные встраиваются в клиентский бандл при сборке.
- Их нельзя менять без пересборки.
- Секреты никогда не должны иметь префикс `NEXT_PUBLIC_`.

### Runtime config для self-hosted

```js
// next.config.js
module.exports = {
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },
}
```

Но для современных self-hosted деплоев лучше использовать `process.env` напрямую на сервере.

---

## Деплой на Vercel

Vercel — оптимальная платформа для Next.js, потому что создатели Next.js и Vercel — одна компания.

### Преимущества

- Zero-config для Next.js.
- Автоматическая поддержка SSR, ISR, Edge.
- Image Optimization из коробки.
- Preview Deployments для каждого PR.
- Автоматический HTTPS, CDN, аналитика.

### Что происходит при деплое

1. Подключение репозитория к Vercel.
2. Vercel определяет Next.js автоматически.
3. Выполняется `next build`.
4. Статические файлы раздаются через Edge Network.
5. Serverless-функции создаются для SSR/API Routes.
6. Каждый PR получает свой preview URL.

### Настройка через dashboard

- Environment Variables.
- Build Command: `next build`.
- Output Directory: `.next` (по умолчанию).
- Install Command: `npm install` или `npm ci`.

### Vercel CLI

```bash
npm i -g vercel
vercel
vercel --prod
```

---

## Деплой в Docker

Docker даёт полный контроль над окружением и работает на любой платформе.

### Multi-stage Dockerfile

```dockerfile
# Этап 1: Зависимости
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Этап 2: Сборка
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Этап 3: Production
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000

CMD ["node", "server.js"]
```

### Важные настройки

```js
// next.config.js
module.exports = {
  output: 'standalone',
}
```

### Сборка и запуск

```bash
docker build -t my-next-app .
docker run -p 3000:3000 my-next-app
```

---

## Деплой на других платформах

### Netlify

```bash
npm install -g netlify-cli
netlify deploy --build --prod
```

- Поддерживает Next.js Runtime.
- ISR работает через on-demand revalidation.
- Edge Functions поддерживаются.

### Railway

1. Подключить репозиторий.
2. Добавить переменные окружения.
3. Railway автоматически запускает `npm start`.

Важно: убедитесь, что в `package.json` есть скрипт `start`:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  }
}
```

### Cloudflare Pages

```bash
npm install @cloudflare/next-on-pages
npx @cloudflare/next-on-pages
```

- Требуется адаптер, потому что Cloudflare использует Workers runtime.
- Ограничения по Node.js API.
- Отличная глобальная сеть.

### Self-hosted на VPS

```bash
npm run build
npm start
```

Лучше использовать менеджер процессов:

```bash
npm install -g pm2
pm2 start npm --name "next-app" -- start
```

---

## Чеклист перед деплоем

- [ ] `npm run build` проходит локально без ошибок.
- [ ] Все environment variables настроены на платформе.
- [ ] Секреты не имеют префикса `NEXT_PUBLIC_`.
- [ ] `next/image` настроен под выбранную платформу.
- [ ] ISR настроен и протестирован, если используется.
- [ ] Middleware работает в Edge Runtime.
- [ ] API Routes защищены и валидируют входные данные.
- [ ] Добавлен health check endpoint.
- [ ] Настроен мониторинг ошибок и производительности.
- [ ] Есть план отката (rollback).

---

## Заключение

Деплой Next.js зависит от выбранной стратегии рендеринга и платформы.

**Ключевые выводы:**

1. `output: 'export'` — для чистого SSG на любом статическом хостинге.
2. `output: 'standalone'` — для Docker и self-hosted.
3. Vercel даёт максимальную поддержку всех фич Next.js.
4. ISR требует платформу с serverless-функциями или сервер.
5. `next/image` нуждается в сервере или custom loader.
6. Edge Runtime — для middleware и простых API, не для тяжёлой логики.
7. Docker — универсальный способ деплоя, но требует больше настройки.

На собеседовании часто спрашивают про разницу между `output: 'export'` и `output: 'standalone'`, про ISR, про `NEXT_PUBLIC_` и про оптимизацию изображений. Будьте готовы объяснить каждый пункт на примере.
