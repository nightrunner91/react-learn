# Методы рендеринга в Next.js

Next.js поддерживает несколько стратегий рендеринга, каждая из которых подходит для определённых сценариев.

---

## 1. CSR — Client-Side Rendering (Рендеринг на клиенте)

**Что это:** Страница рендерится полностью в браузере с помощью JavaScript.

**Когда использовать:**
- Дашборды, приватные страницы
- Интерактивные приложения без SEO-требований
- Контент, не требующий индексации

**Как включить:**
```tsx
'use client'

import { useState, useEffect } from 'react'

export default function Page() {
  const [data, setData] = useState(null)

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData)
  }, [])

  return <div>{data?.title}</div>
}
```

**Плюсы:**
- Мгновенная навигация после первой загрузки
- Снижает нагрузку на сервер

**Минусы:**
- Плохо для SEO (поисковики видят пустую страницу)
- Долгая первая загрузка ([FCP](#fcp), [LCP](#lcp))
- Зависимость от JavaScript

---

## 2. SSR — Server-Side Rendering (Рендеринг на сервере)

**Что это:** Страница генерируется на сервере при каждом запросе.

**Когда использовать:**
- Динамический контент, который часто меняется
- Персонализированные страницы
- Данные, которые должны быть актуальными при каждом запросе

**Как включить:**
```tsx
// app/page.tsx
export const dynamic = 'force-dynamic' // Отключает кэширование

export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store'
  })
  const posts = await data.json()

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

**Плюсы:**
- SEO-friendly (полный HTML при первом запросе)
- Всегда актуальные данные
- Быстрая первая отрисовка

**Минусы:**
- Нагрузка на сервер при каждом запросе
- Медленнее, чем статические страницы
- Требует серверной инфраструктуры

---

## 3. SSG — Static Site Generation (Статическая генерация)

**Что это:** Страницы генерируются один раз при сборке проекта и раздаются как статические файлы.

**Когда использовать:**
- Блоги, документация
- Маркетинговые страницы
- Контент, который редко меняется

**Как включить:**
```tsx
// app/posts/page.tsx
export const dynamic = 'force-static' // По умолчанию в Next.js

export default async function Page() {
  const posts = await getAllPosts() // Выполняется при сборке

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

**Для динамических маршрутов:**
```tsx
// app/posts/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json())
  
  return posts.map(post => ({
    slug: post.slug
  }))
}

export default async function Page({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug)
  return <article>{post.content}</article>
}
```

**Плюсы:**
- Максимальная производительность (CDN)
- Отличное SEO
- Низкая нагрузка на сервер

**Минусы:**
- Долгая сборка при большом количестве страниц
- Контент устаревает до следующей сборки

---

## 4. ISR — Incremental Static Regeneration (Инкрементальная статическая регенерация)

**Что это:** Комбинация SSG и SSR. Страницы генерируются статически, но могут обновляться в фоне с заданной периодичностью.

**Когда использовать:**
- Контент, который обновляется периодически (раз в минуту/час/день)
- Большие сайты, где полная пересборка нецелесообразна
- E-commerce (цены, наличие товаров)

**Как включить:**
```tsx
// app/page.tsx
export const revalidate = 60 // Обновлять каждые 60 секунд

export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 } // Или через опции fetch
  })

  return <div>{data.title}</div>
}
```

**On-Demand Revalidation (по запросу):**

Механизм принудительного обновления кэша ISR-страницы по событию, без ожидания истечения `revalidate`.

**Как это работает:**
1. Страница сгенерирована статически и лежит в кэше
2. Происходит событие (например, добавлена новая статья в CMS)
3. Вызывается API-эндпоинт, который очищает кэш для конкретной страницы
4. Следующий запрос к странице генерирует её заново и кладёт свежую версию в кэш

**revalidatePath** — очищает кэш конкретного пути:
```tsx
// app/api/revalidate/route.ts
import { revalidatePath } from 'next/cache'

export async function POST(request: Request) {
  const { path } = await request.json()
  
  revalidatePath(path) // Очищает кэш для /posts/my-article
  
  return Response.json({ revalidated: true, now: Date.now() })
}
```

**revalidateTag** — очищает кэш по тегу (более гибкий подход):
```tsx
// app/posts/page.tsx
export default async function Page() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] } // Помечаем данные тегом
  }).then(res => res.json())
  
  return <PostsList posts={posts} />
}

// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache'

export async function POST(request: Request) {
  revalidateTag('posts') // Очищает ВСЕ данные с тегом 'posts'
  
  return Response.json({ revalidated: true })
}
```

**Практический сценарий (CMS + Webhook):**
```tsx
// app/api/webhook/route.ts
import { revalidateTag } from 'next/cache'

export async function POST(request: Request) {
  const secret = request.headers.get('x-webhook-secret')
  
  if (secret !== process.env.WEBHOOK_SECRET) {
    return Response.json({ error: 'Invalid secret' }, { status: 401 })
  }
  
  const { event, data } = await request.json()
  
  if (event === 'post.updated') {
    revalidateTag(`post-${data.id}`) // Обновляем конкретный пост
    revalidatePath('/') // Обновляем главную (список постов)
  }
  
  return Response.json({ revalidated: true })
}
```

**Разница между Path и Tag:**

| Подход | Когда использовать |
|--------|-------------------|
| `revalidatePath` | Знаешь точный URL страницы |
| `revalidateTag` | Нужно обновить несколько страниц/компонентов с одними данными |

**Пример с тегами:**
```tsx
// app/posts/page.tsx — использует тег 'posts'
const posts = await fetchPosts({ next: { tags: ['posts'] } })

// app/posts/[slug]/page.tsx — использует тег конкретного поста
const post = await fetchPost(slug, { next: { tags: [`post-${slug}`] } })

// app/api/revalidate/route.ts
revalidateTag('posts') // Обновит список постов
revalidateTag('post-my-article') // Обновит только конкретный пост
```

**Поведение stale-while-revalidate:**
1. Запрос к странице с невалидным кэшем → stale-while-revalidate
2. Пользователь получает старую версию (мгновенно)
3. В фоне генерируется новая версия
4. Следующие запросы получают свежую версию

Это означает, что **первый пользователь после ревалидации может увидеть устаревший контент**, но последующие — уже актуальный.

**Плюсы:**
- Производительность SSG + актуальность SSR
- Не нужна полная пересборка
- Гибкость в выборе частоты обновления

**Минусы:**
- Пользователь может увидеть устаревший контент (до revalidate)
- Сложнее в настройке, чем чистый SSG

---

## 5. RSC — React Server Components (Серверные компоненты React)

**Что это:** Компоненты, которые выполняются исключительно на сервере. Не отправляются в бандл клиента.

**Когда использовать:**
- Получение данных напрямую из компонента (БД, API, файловая система)
- Работа с конфиденциальными данными (токены, ключи API, секреты)
- Тяжёлые зависимости (не нужно тащить в клиентский бандл)
- Статический контент (блоги, документация, маркетинговые страницы)
- Компоненты, которые не требуют интерактивности (карточки, списки, навигация)
- Композиция с клиентскими компонентами (обёртка для передачи данных)
- Оптимизация размера бандла (большие библиотеки остаются на сервере)
- Прямой доступ к backend-ресурсам без API-слоя
- По умолчанию все компоненты в App Router — серверные

**Как использовать:**
```tsx
// app/page.tsx — Серверный компонент (по умолчанию)
import { db } from '@/lib/db' // Прямой доступ к БД

export default async function Page() {
  const users = await db.user.findMany() // Async/await прямо в компоненте
  
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}
```

**Клиентский компонент (когда нужен интерактив):**
```tsx
// components/Counter.tsx
'use client'

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)
  
  return <button onClick={() => setCount(count + 1)}>{count}</button>
}
```

**Плюсы:**
- Нулевой размер бандла для серверных компонентов
- Прямой доступ к backend-ресурсам
- Автоматический code splitting
- Улучшенная безопасность (секреты не попадают в клиент)

**Минусы:**
- Нет доступа к хукам (useState, useEffect)
- Нет доступа к браузерным API (window, localStorage)
- Нет обработчиков событий

**Практическое соотношение RSC и Client Components:**

В реальном Next.js проекте **большинство компонентов — серверные (RSC)**. Типичное соотношение: **70-80% RSC, 20-30% Client Components**.

**Почему так:**
- Страницы, layouts, компоненты данных — всё это RSC по умолчанию
- Клиентские компоненты нужны только там, где есть интерактивность (формы, модальные окна, слайдеры, дропдауны)
- RSC лучше для производительности (меньше JavaScript на клиенте)

**Типичные Client Components:**
- Формы с валидацией
- Модальные окна, дропдауны, табы
- Слайдеры, карусели
- Компоненты с локальным состоянием (счётчики, toggle)
- Обработчики событий (onClick, onChange, onSubmit)
- Использование хуков (useState, useEffect, useRef)

**Типичные RSC:**
- Страницы (app/**/page.tsx)
- Layouts (app/**/layout.tsx)
- Компоненты получения данных
- Статические компоненты (карточки, списки, навигация)
- Обёртки для клиентских компонентов

**Терминология:**
- **RSC** — React Server Components (серверные компоненты)
- **Client Components** — клиентские компоненты (официальное название)
- Термин **RCC** (React Client Components) **не используется официально**, но иногда встречается в сообществах. Правильно говорить просто "Client Components" или "клиентские компоненты".

---

## 6. Streaming SSR (Потоковый серверный рендеринг)

**Что это:** Разновидность SSR, где HTML отправляется частями (чанками) по мере готовности компонентов.

**Когда использовать:**
- Страницы с тяжёлыми компонентами
- Когда нужно показать контент как можно быстрее
- Использование Suspense для graceful degradation

**Как включить:**
```tsx
// app/page.tsx
import { Suspense } from 'react'

export default function Page() {
  return (
    <div>
      <h1>Главная страница</h1>
      
      <Suspense fallback={<div>Загрузка...</div>}>
        <SlowComponent /> {/* Рендерится позже */}
      </Suspense>
    </div>
  )
}

async function SlowComponent() {
  const data = await fetchSlowData() // Блокирующий запрос
  return <div>{data.content}</div>
}
```

**Плюсы:**
- Быстрый Time to First Byte (TTFB)
- Прогрессивная загрузка контента
- Лучший пользовательский опыт

**Минусы:**
- Сложнее в отладке
- Требует поддержки Suspense на клиенте

---

## Сравнительная таблица

| Метод | SEO | Скорость | Актуальность | Сложность |
|-------|-----|----------|--------------|-----------|
| CSR   | Плохо | Медленная [FCP](#fcp) | Всегда | Низкая |
| SSR   | Отлично | Быстрая | Всегда | Средняя |
| SSG   | Отлично | Максимальная | При сборке | Низкая |
| ISR   | Отлично | Максимальная | Периодически | Средняя |
| RSC   | Отлично | Максимальная | Зависит от данных | Средняя |
| Streaming | Отлично | Быстрая TTFB | Всегда | Высокая |

---

## Рекомендации по выбору

- **Блог/документация** → SSG
- **E-commerce** → ISR + RSC
- **Дашборд** → CSR + RSC
- **Новости/соцсеть** → SSR или ISR
- **Лендинг** → SSG
- **Админка** → CSR

---

## Сноски

<a id="fcp"></a>
**FCP (First Contentful Paint)** — время до первого отображения любого контента на экране (текст, изображение, SVG). Показывает, насколько быстро пользователь видит, что страница "загружается".

<a id="lcp"></a>
**LCP (Largest Contentful Paint)** — время до отрисовки самого крупного видимого элемента (обычно главное изображение или заголовок). Ключевая метрика воспринимаемой скорости загрузки.

---

## Полезные ссылки

- [Next.js Rendering Documentation](https://nextjs.org/docs/app/building-your-application/rendering)
- [React Server Components](https://react.dev/reference/rsc/server-components)
- [Next.js Caching](https://nextjs.org/docs/app/building-your-application/caching)
