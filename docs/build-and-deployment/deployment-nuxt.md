# Деплой Nuxt 3

## Содержание

1. [Что такое Nitro](#что-такое-nitro)
2. [`nuxt build` vs `nuxt generate`](#nuxt-build-vs-nuxt-generate)
3. [SSR-деплой](#ssr-деплой)
4. [SSG и Prerender](#ssg-и-prerender)
5. [ISR через `routeRules`](#isr-через-routerules)
6. [Edge-пресеты](#edge-пресеты)
7. [Runtime config и environment variables](#runtime-config-и-environment-variables)
8. [Server routes и API](#server-routes-и-api)
9. [Деплой на популярные платформы](#деплой-на-популярные-платформы)
10. [Image optimization](#image-optimization)
11. [Чеклист перед деплоем](#чеклист-перед-деплоем)
12. [Заключение](#заключение)

---

## Что такое Nitro

Nuxt 3 построен поверх **Nitro** — универсального серверного движка. Nitro отвечает за:

- Рендеринг страниц на сервере.
- Генерацию статических файлов.
- Подготовку serverless-функций для разных платформ.
- Работу с server routes в `server/api/`.
- Edge-деплой на CDN-узлы.

```
┌─────────────────────────────────────────────────────────────┐
│                         Nuxt 3                               │
├─────────────────────────────────────────────────────────────┤
│                         Nitro                                │
├─────────────────┬─────────────────┬─────────────────────────┤
│   SSR / SSG     │  Server Routes  │    Edge Presets         │
│   рендеринг     │  server/api/    │    Cloudflare, Vercel   │
└─────────────────┴─────────────────┴─────────────────────────┘
```

Nitro позволяет одной командой собрать приложение для Node.js, Cloudflare Workers, Vercel, Netlify, Deno Deploy и многих других сред.

---

## `nuxt build` vs `nuxt generate`

| Команда | Что делает | Требует сервер | Подходит для |
|---------|-----------|----------------|--------------|
| `nuxt build` | Собирает SSR/ISR приложение | Да | Full-stack приложения |
| `nuxt generate` | Генерирует статические HTML | Нет | Блоги, лендинги, документация |

### `nuxt build`

```bash
nuxt build
```

Создаёт папку `.output/` с:

- `public/` — статические файлы.
- `server/` — серверный код.
- `nitro.json` — манифест Nitro.

Запуск production:

```bash
node .output/server/index.mjs
```

### `nuxt generate`

```bash
nuxt generate
```

Создаёт папку `.output/public/` со статическими файлами. Можно загрузить на любой статический хостинг.

### Гибридный режим

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
    '/blog/**': { prerender: true },
    '/dashboard/**': { ssr: false },
    '/api/**': { cors: true },
  },
})
```

---

## SSR-деплой

### Базовая конфигурация

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: true, // По умолчанию
})
```

### Что происходит при сборке

```
┌─────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ nuxt build  │ ───► │  Nitro готовит  │ ───► │  .output/       │
│             │      │  SSR-сервер     │      │  server/index.mjs│
└─────────────┘      └─────────────────┘      └─────────────────┘
```

### Запуск в production

```bash
node .output/server/index.mjs
```

### Docker

```dockerfile
FROM node:20-alpine
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

ENV NUXT_HOST=0.0.0.0
ENV NUXT_PORT=3000

EXPOSE 3000

CMD ["node", ".output/server/index.mjs"]
```

### Presets для SSR

Nitro автоматически определяет preset по окружению, но можно задать явно:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'node-server', // По умолчанию для self-hosted
  },
})
```

---

## SSG и Prerender

### Глобальная генерация

```bash
nuxt generate
```

### Частичный prerender

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
    '/about': { prerender: true },
    '/blog/**': { prerender: true },
  },
})
```

### Динамические маршруты

Для динамических маршрутов нужно указать `prerender` и задать параметры:

```vue
<!-- pages/blog/[slug].vue -->
<script setup>
const route = useRoute()
const { data: post } = await useFetch(`/api/posts/${route.params.slug}`)
</script>
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    prerender: {
      routes: ['/blog/first-post', '/blog/second-post'],
    },
  },
})
```

### Или через `useAsyncData` с `prerender`

```ts
// server/api/posts.get.ts
export default defineEventHandler(() => {
  return [
    { slug: 'first-post', title: 'First Post' },
    { slug: 'second-post', title: 'Second Post' },
  ]
})
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    prerender: {
      routes: ['/'],
    },
  },
})
```

---

## ISR через `routeRules`

Nuxt 3 поддерживает ISR через `routeRules` в `nuxt.config.ts`.

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': { isr: 60 },
    '/products/**': { isr: 300 },
  },
})
```

### On-Demand Revalidation

```ts
// server/api/revalidate.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)

  await $fetch(`/api/__nuft__/isr?path=${body.path}`, {
    method: 'POST',
    baseURL: useRuntimeConfig().public.siteUrl,
  })

  return { revalidated: true }
})
```

> ISR в Nuxt реализован через Nitro и зависит от preset. На Vercel и Netlify работает из коробки, на self-hosted требует настройки хранилища.

---

## Edge-пресеты

Nitro умеет собирать приложение для Edge Runtime.

### Cloudflare Workers

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
  },
})
```

```bash
nuxt build
# Папка dist/ готова для Cloudflare Pages
```

### Vercel Edge

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'vercel-edge',
  },
})
```

### Netlify Edge

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'netlify-edge',
  },
})
```

### Популярные presets

| Preset | Платформа |
|--------|-----------|
| `node-server` | Self-hosted, Docker, Railway |
| `vercel` | Vercel (SSR) |
| `vercel-edge` | Vercel Edge Functions |
| `netlify` | Netlify (SSR) |
| `netlify-edge` | Netlify Edge Functions |
| `cloudflare-pages` | Cloudflare Pages |
| `cloudflare-module` | Cloudflare Workers |
| `deno-deploy` | Deno Deploy |

---

## Runtime config и environment variables

### `runtimeConfig`

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '', // Серверная часть
    public: {
      apiBase: '/api', // Доступно на клиенте
    },
  },
})
```

### Переменные окружения

```env
# .env
NUXT_API_SECRET=secret-key
NUXT_PUBLIC_API_BASE=https://api.example.com
```

Nuxt автоматически маппит `NUXT_*` переменные в `runtimeConfig`.

### Использование в коде

```vue
<script setup>
const config = useRuntimeConfig()

// На сервере
console.log(config.apiSecret)

// На клиенте
console.log(config.public.apiBase)
</script>
```

### Важно

- `runtimeConfig.public` доступен и на клиенте, и на сервере.
- `runtimeConfig` без `public` — только на сервере.
- В отличие от `NEXT_PUBLIC_`, здесь переменные можно менять без пересборки, если preset это поддерживает.

---

## Server routes и API

Server routes в Nuxt — это обработчики на базе h3, которые компилируются Nitro.

```ts
// server/api/users.get.ts
export default defineEventHandler(() => {
  return [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
  ]
})
```

```ts
// server/api/users.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)

  if (!body.name) {
    throw createError({
      statusCode: 400,
      statusMessage: 'Name is required',
    })
  }

  return { id: 1, name: body.name }
})
```

### Middleware на сервере

```ts
// server/middleware/auth.ts
export default defineEventHandler((event) => {
  const token = getHeader(event, 'authorization')

  if (!token) {
    throw createError({ statusCode: 401 })
  }
})
```

### Что работает в server routes

- Работа с БД.
- Вызов внешних API.
- Чтение/запись файлов (в Node.js preset).
- Edge-compatible код (для edge presets).

---

## Деплой на популярные платформы

### Vercel

1. Импортировать репозиторий.
2. Vercel автоматически определяет Nuxt.
3. Настроить environment variables.
4. Деплой.

```bash
npx nuxi@latest init my-app
npx vercel
```

### Netlify

```bash
npx netlify-cli deploy --build --prod
```

Или через UI: задать Build command `npm run build` и Publish directory `dist`.

### Railway

```bash
railway login
railway init
railway up
```

Railway автоматически определит Node.js приложение и запустит `npm start`.

### Cloudflare Pages

```bash
npx nuxi build --preset cloudflare-pages
```

Затем загрузить папку `dist/` в Cloudflare Pages.

### Self-hosted VPS

```bash
nuxt build
node .output/server/index.mjs
```

Рекомендуется использовать процесс-менеджер:

```bash
npm install -g pm2
pm2 start .output/server/index.mjs --name nuxt-app
```

---

## Image optimization

### Nuxt Image

```bash
npm install @nuxt/image
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/image'],
  image: {
    provider: 'ipx', // По умолчанию для self-hosted
  },
})
```

```vue
<template>
  <NuxtImg src="/photo.jpg" width="800" height="600" />
</template>
```

### Провайдеры

| Провайдер | Когда использовать |
|-----------|-------------------|
| `ipx` | Self-hosted, Docker |
| `cloudflare` | Cloudflare Pages |
| `cloudinary` | Cloudinary |
| `imgix` | Imgix |
| `vercel` | Vercel |

### Static export

При `nuxt generate` изображения оптимизируются на этапе сборки.

---

## Чеклист перед деплоем

- [ ] `nuxt build` проходит локально без ошибок.
- [ ] `nuxt generate` протестирован, если используется SSG.
- [ ] Все environment variables добавлены в `runtimeConfig`.
- [ ] Server routes валидируют входные данные.
- [ ] ISR настроен и протестирован.
- [ ] Выбран правильный Nitro preset.
- [ ] `site.url` настроен для SEO и метаданных.
- [ ] Image provider соответствует платформе.
- [ ] Добавлен health check.
- [ ] Есть план отката.

---

## Заключение

Nuxt 3 благодаря Nitro предлагает очень гибкий деплой: от статического экспорта до Edge Runtime.

**Ключевые выводы:**

1. `nuxt build` — для SSR/ISR, требует сервер.
2. `nuxt generate` — для чистого SSG, можно хостить где угодно.
3. `routeRules` — центральное место для управления SSG/SSR/ISR.
4. Nitro preset определяет, куда можно задеплоить приложение.
5. `runtimeConfig` — правильный способ работы с переменными окружения.
6. Edge-presets работают на Cloudflare, Vercel, Netlify.
7. `@nuxt/image` упрощает оптимизацию изображений, но требует настройки провайдера.

На собеседовании важно понимать разницу между `nuxt build` и `nuxt generate`, уметь настраивать `routeRules` и объяснять, зачем нужен Nitro.
