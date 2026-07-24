# Глубокое погружение в роутинг Next.js: App Router

Next.js App Router — это революционный подход к маршрутизации, основанный на файловой системе и React Server Components. В отличие от традиционных роутеров, здесь структура файлов напрямую определяет URL-маршруты вашего приложения.

## Содержание

- [Основы App Router](#основы-app-router)
- [Базовая маршрутизация](#базовая-маршрутизация)
- [Динамические сегменты](#динамические-сегменты)
- [Route Groups](#route-groups)
- [Параллельные роуты](#параллельные-роуты)
- [Перехватывающие роуты](#перехватывающие-роуты)
- [Специальные файлы](#специальные-файлы)
- [Layouts и Templates](#layouts-и-templates)
- [Loading и Error UI](#loading-и-error-ui)
- [Route Handlers](#route-handlers)
- [Навигация](#навигация)
- [Middleware](#middleware)
- [Best Practices](#best-practices)

## Основы App Router

### Что такое App Router

App Router — это система маршрутизации в Next.js 13+, построенная на React Server Components. Ключевые особенности:

- **Файловая система как API**: структура папок определяет маршруты
- **Серверные компоненты по умолчанию**: все компоненты серверные, если не указано иное
- **Вложенные layouts**: каждый сегмент может иметь свой layout
- **Стриминг**: постепенная загрузка контента
- **Предварительная выборка**: автоматическая prefetching навигации

### Структура директории app

```
my-app/
├── app/
│   ├── layout.tsx          # Корневой layout (обязателен)
│   ├── page.tsx            # Главная страница (/)
│   ├── loading.tsx         # Глобальный loading UI
│   ├── error.tsx           # Глобальный error boundary
│   ├── not-found.tsx       # Глобальная 404 страница
│   ├── blog/
│   │   ├── layout.tsx      # Layout для /blog/*
│   │   ├── page.tsx        # Список постов (/blog)
│   │   ├── [slug]/
│   │   │   └── page.tsx    # Пост блога (/blog/my-post)
│   │   └── category/
│   │       └── [name]/
│   │           └── page.tsx  # Категория (/blog/category/tech)
│   └── api/
│       └── users/
│           └── route.ts    # API endpoint (/api/users)
```

## Базовая маршрутизация

### Создание страниц

Каждая папка в `app` представляет сегмент URL. Чтобы сделать сегмент доступным, создайте файл `page.tsx`:

```tsx
// app/page.tsx — Главная страница (/)
export default function HomePage() {
  return <h1>Добро пожаловать</h1>
}

// app/about/page.tsx — Страница "О нас" (/about)
export default function AboutPage() {
  return <h1>О компании</h1>
}

// app/blog/page.tsx — Список постов (/blog)
export default async function BlogPage() {
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

### Вложенные маршруты

Вложенность папок создает вложенные URL:

```
app/
├── shop/
│   ├── page.tsx              # /shop
│   ├── cart/
│   │   └── page.tsx          # /shop/cart
│   └── checkout/
│       └── page.tsx          # /shop/checkout
```

**Важно**: папка без `page.tsx` технически не создает маршрут, но **не рекомендуется** размещать в `/app` общие компоненты и утилиты. Это антипаттерн, который нарушает семантику директории и может привести к проблемам с импортами. Вместо этого выносите переиспользуемый код в отдельные директории (`components/`, `lib/`, `hooks/`) вне `/app`. Используйте colocation (подпапки `components/` внутри роута) только для компонентов, специфичных для конкретного роута.

## Динамические сегменты

### Базовые динамические роуты

Используйте квадратные скобки для динамических сегментов:

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPost({
  params
}: {
  params: { slug: string }
}) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(r => r.json())
  
  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  )
}
```

**Примеры URL:**
- `/blog/hello-world` → `params.slug = 'hello-world'`
- `/blog/nextjs-routing` → `params.slug = 'nextjs-routing'`

### Множественные динамические сегменты

```tsx
// app/shop/[category]/[productId]/page.tsx
export default function ProductPage({
  params
}: {
  params: { category: string; productId: string }
}) {
  return (
    <div>
      Категория: {params.category}
      Товар: {params.productId}
    </div>
  )
}
```

**Примеры:**
- `/shop/electronics/laptop-123` → `{ category: 'electronics', productId: 'laptop-123' }`

### Catch-All роуты

Для захвата нескольких сегментов используйте `[...slug]`:

```tsx
// app/docs/[...slug]/page.tsx
export default function DocsPage({
  params
}: {
  params: { slug: string[] }
}) {
  return (
    <div>
      Путь: {params.slug.join(' → ')}
    </div>
  )
}
```

**Примеры:**
- `/docs/a` → `slug = ['a']`
- `/docs/a/b` → `slug = ['a', 'b']`
- `/docs/a/b/c` → `slug = ['a', 'b', 'c']`

### Optional Catch-All

Двойные скобки `[[...slug]]` делают сегмент опциональным:

```tsx
// app/shop/[[...categories]]/page.tsx
export default function ShopPage({
  params
}: {
  params: { categories?: string[] }
}) {
  if (!params.categories) {
    return <div>Все товары</div>
  }
  
  return (
    <div>
      Фильтр: {params.categories.join(' / ')}
    </div>
  )
}
```

**Примеры:**
- `/shop` → `categories = undefined`
- `/shop/electronics` → `categories = ['electronics']`
- `/shop/electronics/laptops` → `categories = ['electronics', 'laptops']`

### Приоритет маршрутов

При конфликте маршрутов Next.js использует приоритет:

1. **Статические маршруты** (highest priority)
2. **Динамические маршруты** (`[param]`)
3. **Catch-all маршруты** (`[...slug]`)

```
app/
├── shop/
│   ├── new/
│   │   └── page.tsx          # /shop/new (статический)
│   ├── [category]/
│   │   └── page.tsx          # /shop/electronics (динамический)
│   └── [...slug]/
│       └── page.tsx          # /shop/a/b/c (catch-all)
```

## Route Groups

Route Groups позволяют организовывать код без влияния на URL. Используйте круглые скобки:

```
app/
├── (marketing)/
│   ├── about/
│   │   └── page.tsx          # /about
│   ├── pricing/
│   │   └── page.tsx          # /pricing
│   └── layout.tsx            # Layout для marketing страниц
├── (dashboard)/
│   ├── dashboard/
│   │   └── page.tsx          # /dashboard
│   ├── settings/
│   │   └── page.tsx          # /settings
│   └── layout.tsx            # Layout для dashboard
└── layout.tsx                # Корневой layout
```

### Практическое применение

**Разные layouts для разных секций:**

```tsx
// app/(marketing)/layout.tsx
export default function MarketingLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <div>
      <MarketingHeader />
      <main>{children}</main>
      <MarketingFooter />
    </div>
  )
}

// app/(dashboard)/layout.tsx
export default function DashboardLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <div className="flex">
      <DashboardSidebar />
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

**Организация по функциональности:**

```
app/
├── (auth)/
│   ├── login/
│   │   └── page.tsx
│   └── register/
│       └── page.tsx
├── (shop)/
│   ├── products/
│   │   └── page.tsx
│   └── cart/
│       └── page.tsx
└── (admin)/
    ├── users/
    │   └── page.tsx
    └── analytics/
        └── page.tsx
```

## Параллельные роуты

Параллельные роуты позволяют рендерить несколько страниц одновременно в одном layout. Используйте `@` для определения слотов:

```
app/
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── @stats/
│   │   └── page.tsx
│   ├── @activity/
│   │   └── page.tsx
│   └── @notifications/
│       └── page.tsx
```

### Реализация параллельных роутов

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  stats,
  activity,
  notifications
}: {
  children: React.ReactNode
  stats: React.ReactNode
  activity: React.ReactNode
  notifications: React.ReactNode
}) {
  return (
    <div>
      <h1>Dashboard</h1>
      
      {/* Основной контент */}
      {children}
      
      {/* Параллельные слоты */}
      <div className="grid grid-cols-3 gap-4">
        <section>{stats}</section>
        <section>{activity}</section>
        <section>{notifications}</section>
      </div>
    </div>
  )
}

// app/dashboard/page.tsx
export default function DashboardPage() {
  return <div>Основной контент dashboard</div>
}

// app/dashboard/@stats/page.tsx
export default async function StatsSlot() {
  const stats = await fetch('/api/stats').then(r => r.json())
  
  return (
    <div>
      <h2>Статистика</h2>
      <p>Пользователей: {stats.users}</p>
      <p>Заказов: {stats.orders}</p>
    </div>
  )
}

// app/dashboard/@activity/page.tsx
export default async function ActivitySlot() {
  const activity = await fetch('/api/activity').then(r => r.json())
  
  return (
    <div>
      <h2>Последняя активность</h2>
      <ul>
        {activity.map(item => (
          <li key={item.id}>{item.description}</li>
        ))}
      </ul>
    </div>
  )
}
```

### Преимущества параллельных роутов

- **Независимая загрузка**: каждый слот загружается асинхронно
- **Изолированные ошибки**: ошибка в одном слоте не ломает другие
- **Стриминг**: каждый слот стримится независимо
- **Условный рендеринг**: можно показывать/скрывать слоты

### Обработка ошибок в слотах

```tsx
// app/dashboard/@stats/error.tsx
'use client'

export default function StatsError({
  error,
  reset
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div>
      <p>Ошибка загрузки статистики</p>
      <button onClick={reset}>Попробовать снова</button>
    </div>
  )
}
```

### Важное уточнение о параллельных роутах

**Параллельные роуты — это не отдельные маршруты.** Слоты `@stats`, `@activity` и т.д. **не создают URL** и не доступны для прямого перехода через браузерную строку.

```
app/
├── dashboard/
│   ├── page.tsx              # /dashboard ← Единственный доступный URL
│   ├── @stats/
│   │   └── page.tsx          # НЕ создает /dashboard/stats
│   └── @activity/
│       └── page.tsx          # НЕ создает /dashboard/activity
```

Параллельные роуты — это **дробление одной страницы на независимые секции**, а не система маршрутизации. Все слоты рендерятся вместе на одном URL через layout.

**Когда использовать:**
- Dashboard с независимыми виджетами
- Страница с секциями, которые могут падать/загружаться независимо
- Нужна изоляция ошибок между частями страницы

**Когда НЕ использовать:**
- Нужны отдельные URL для секций → используйте обычные вложенные роуты
- Компоненты зависят друг от друга
- Простая страница с одним контентом

**Сравнение:**

| Аспект | Параллельные роуты (`@`) | Обычные вложенные роуты |
|--------|--------------------------|-------------------------|
| Создают URL | ❌ Нет | ✅ Да |
| Прямой переход | ❌ Нельзя | ✅ Можно |
| Назначение | Секции одной страницы | Отдельные страницы |
| Изоляция | Независимые loading/error | Общие |

## Перехватывающие роуты

Перехватывающие роуты позволяют показать маршрут в другом контексте. Например, открыть модальное окно при клике, но полную страницу при прямом переходе по URL.

### Синтаксис перехвата

```
(.)      — перехват на том же уровне
(..)     — перехват на уровень выше
(..)(..) — перехват на два уровня выше
(...)    — перехват от корня app
```

### Пример: Модальное окно для фото

```
app/
├── feed/
│   └── page.tsx              # Лента постов
├── photo/[id]/
│   ├── page.tsx              # Полная страница фото
│   └── (.)photo/[id]/
│       └── page.tsx          # Модальное окно (перехват)
```

**Реализация:**

```tsx
// app/feed/page.tsx
import Link from 'next/link'

export default function FeedPage() {
  const photos = [
    { id: '1', url: '/photos/1.jpg' },
    { id: '2', url: '/photos/2.jpg' }
  ]
  
  return (
    <div>
      {photos.map(photo => (
        <Link key={photo.id} href={`/photo/${photo.id}`}>
          <img src={photo.url} alt={`Photo ${photo.id}`} />
        </Link>
      ))}
    </div>
  )
}

// app/photo/[id]/page.tsx
export default async function PhotoPage({
  params
}: {
  params: { id: string }
}) {
  const photo = await getPhoto(params.id)
  
  return (
    <div>
      <h1>Фото {params.id}</h1>
      <img src={photo.fullUrl} alt={photo.title} />
      <p>{photo.description}</p>
    </div>
  )
}

// app/feed/(.)photo/[id]/page.tsx
import Modal from '@/components/Modal'

export default async function PhotoModal({
  params
}: {
  params: { id: string }
}) {
  const photo = await getPhoto(params.id)
  
  return (
    <Modal>
      <img src={photo.fullUrl} alt={photo.title} />
      <p>{photo.description}</p>
    </Modal>
  )
}
```

### Как это работает

1. **Клик в ленте**: Next.js перехватывает навигацию и показывает модальное окно из `feed/(.)photo/[id]/page.tsx`
2. **Прямой переход по URL**: показывается полная страница из `photo/[id]/page.tsx`
3. **Обновление страницы**: если модалка открыта, при обновлении показывается полная страница

### Практические сценарии

- **Модальные окна**: предпросмотр контента без перехода
- **Мастеры**: многошаговые формы с сохранением контекста
- **Комментарии**: открытие комментариев в sidebar
- **Детали**: просмотр деталей элемента списка

### Почему при обновлении показывается полная страница

Поведение "модалка при клике, полная страница при обновлении" — это **осознанный дизайн**, а не ограничение:

**1. Шаринг ссылок**
```
1. Пользователь в ленте кликает на фото → модалка
2. Копирует URL: example.com/photo/123
3. Отправляет другу
4. Друг открывает → ожидает полную страницу (не модалку без контекста)
```

**2. Прямой переход по URL**
- Пользователь перешел на `/photo/123` напрямую
- Нет контекста ленты для модалки
- Логично показать полную страницу

**3. SEO и доступность**
- Поисковики индексируют полный URL
- Скринридеры ожидают полноценный контент
- Модалка без контекста — плохой UX

**4. Состояние теряется при обновлении**
- JavaScript состояние исчезает
- React компоненты пересоздаются
- Нет способа "запомнить", что была открыта модалка

**Когда сохранять модалку при обновлении:**
- Модалка содержит форму с данными (регистрация, оплата)
- Пользователь может потерять введенные данные
- Модалка = основной workflow, а не превью

**Когда НЕ сохранять:**
- Модалка = превью/быстрый просмотр
- Полная страница дает больше ценности
- Простой UX важнее сложного состояния

## Специальные файлы

### page.tsx

Основной файл страницы. Делает маршрут доступным:

```tsx
// app/blog/page.tsx
export default async function BlogPage() {
  const posts = await getPosts()
  
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

### layout.tsx

Общий layout для сегмента. Не пересоздается при навигации:

```tsx
// app/dashboard/layout.tsx
import { Sidebar } from '@/components/Sidebar'

export default function DashboardLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <div className="flex">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

**Важно**: layout должен принимать `children` как prop.

### template.tsx

Похож на layout, но пересоздается при каждой навигации:

```tsx
// app/dashboard/template.tsx
export default function DashboardTemplate({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <div className="animate-fade-in">
      {children}
    </div>
  )
}
```

**Когда использовать template:**
- Анимации входа/выхода
- Сброс состояния при навигации
- Fetch данных при каждом переходе

### loading.tsx

Loading UI для Suspense fallback:

```tsx
// app/blog/loading.tsx
export default function BlogLoading() {
  return (
    <div>
      <div className="skeleton" />
      <div className="skeleton" />
      <div className="skeleton" />
    </div>
  )
}
```

Автоматически оборачивает `page.tsx` и вложенные layouts в Suspense:

```tsx
// Эквивалентно:
<Suspense fallback={<BlogLoading />}>
  <BlogPage />
</Suspense>
```

### error.tsx и not-found.tsx

Оба файла используют **одинаковую иерархию поиска** снизу вверх по структуре папок, но служат разным целям.

#### Иерархия поиска

Next.js ищет `error.tsx` и `not-found.tsx` снизу вверх:

```
app/
├── error.tsx                  # Глобальный error (fallback)
├── not-found.tsx              # Глобальный not-found (fallback)
├── blog/
│   ├── error.tsx              # Для /blog/*
│   ├── not-found.tsx          # Для /blog/*
│   └── [slug]/
│       └── page.tsx           # Ошибка/notFound() → ищет в blog/, потом в app/
└── shop/
    ├── error.tsx              # Для /shop/*
    ├── not-found.tsx          # Для /shop/*
    └── [id]/
        └── page.tsx           # Ошибка/notFound() → ищет в shop/, потом в app/
```

**Логика поиска:**
1. Ищет файл в текущем сегменте
2. Если не нашел — идет в родительский сегмент
3. Если дошел до корня — показывает глобальный файл из `app/`

#### Различия между error.tsx и not-found.tsx

| Аспект | not-found.tsx | error.tsx |
|--------|---------------|-----------|
| **Назначение** | Страница 404 (ресурс не найден) | Обработка ошибок (что-то сломалось) |
| **Вызов** | Явный через `notFound()` | Автоматический при ошибке |
| **Клиентский компонент** | ❌ Нет | ✅ Да (`'use client'`) |
| **Props** | Нет | `error`, `reset` |
| **Обязательность** | Глобальный обязателен | Глобальный **критически важен** |
| **Локальные нужны** | Редко | **Чаще**, чем для not-found |

#### error.tsx — обработка ошибок

Error boundary для обработки ошибок. Должен быть клиентским компонентом:

```tsx
// app/blog/error.tsx
'use client'

export default function BlogError({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div>
      <h2>Ошибка загрузки поста</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Попробовать снова</button>
      <Link href="/blog">Вернуться к списку постов</Link>
    </div>
  )
}
```

**Важно**: `error.tsx` должен быть клиентским компонентом (`'use client'`).

#### not-found.tsx — страница 404

Страница 404 для сегмента. Вызывается программно через `notFound()`:

```tsx
// app/blog/not-found.tsx
export default function BlogNotFound() {
  return (
    <div>
      <h2>Пост не найден</h2>
      <p>Запрашиваемый пост не существует</p>
    </div>
  )
}

// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation'

export default async function BlogPost({
  params
}: {
  params: { slug: string }
}) {
  const post = await getPost(params.slug)
  
  if (!post) {
    notFound() // Вызовет not-found.tsx
  }
  
  return <article>{post.content}</article>
}
```

#### Best Practice

**Глобальные файлы** — обязательны в `app/`:

```tsx
// app/not-found.tsx
import Link from 'next/link'

export default function GlobalNotFound() {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h1 className="text-6xl font-bold">404</h1>
      <p className="text-xl mt-4">Страница не найдена</p>
      <Link href="/" className="mt-8 px-6 py-3 bg-blue-500 text-white rounded">
        На главную
      </Link>
    </div>
  )
}

// app/error.tsx
'use client'

export default function GlobalError({
  error,
  reset
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h1 className="text-4xl font-bold">Что-то пошло не так</h1>
      <p className="text-xl mt-4">{error.message}</p>
      <button onClick={reset} className="mt-8 px-6 py-3 bg-blue-500 text-white rounded">
        Попробовать снова
      </button>
    </div>
  )
}
```

**Локальные файлы** — создавай когда нужна специфика:

```tsx
// app/blog/not-found.tsx — есть рекомендации
export default function BlogNotFound() {
  return (
    <div>
      <h1>Пост не найден</h1>
      <p>Возможно, он был удален или перемещен</p>
      <div className="mt-8">
        <h2>Популярные посты:</h2>
        <PopularPosts />
      </div>
      <Link href="/blog">Все посты</Link>
    </div>
  )
}

// app/blog/error.tsx — специфичная логика восстановления
'use client'

export default function BlogError({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>Ошибка загрузки поста</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Попробовать снова</button>
      <Link href="/blog">Вернуться к списку постов</Link>
    </div>
  )
}

// app/shop/not-found.tsx — есть похожие товары
export default function ShopNotFound() {
  return (
    <div>
      <h1>Товар не найден</h1>
      <p>Этот товар больше не доступен</p>
      <div className="mt-8">
        <h2>Похожие товары:</h2>
        <SimilarProducts />
      </div>
      <Link href="/shop">Вернуться в магазин</Link>
    </div>
  )
}

// app/shop/error.tsx — другая логика
'use client'

export default function ShopError({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>Ошибка загрузки товара</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Перезагрузить</button>
      <Link href="/shop">Вернуться в каталог</Link>
    </div>
  )
}
```

#### Когда создавать локальные файлы

**Создавай локальный `not-found.tsx`, если:**
- ✅ Можно предложить релевантные альтернативы (похожие товары, популярные посты)
- ✅ Есть специфичный поиск по разделу
- ✅ Нужен контекстный текст (не просто "404")
- ✅ Можно восстановить пользователя с полезными ссылками

**Создавай локальный `error.tsx`, если:**
- ✅ Нужна специфичная кнопка retry (перезагрузить виджет, пост, товар)
- ✅ Есть контекстные ссылки (вернуться к списку, в каталог)
- ✅ Разная логика восстановления для разных секций
- ✅ Нужен разный UI для разных типов ошибок

**Не создавай локальные файлы, если:**
- ❌ Дизайн одинаковый для всех страниц
- ❌ Нет специфичной логики
- ❌ Просто хочешь "на всякий случай"
- ❌ Глобальные файлы достаточно хороши

**Правило:** локальные файлы должны **помогать пользователю**, а не просто говорить "не найдено" или "ошибка". Если не можешь предложить ценность — используй глобальные.

**Важное отличие:** локальные `error.tsx` нужны **чаще**, чем локальные `not-found.tsx`, потому что разные секции приложения требуют разной логики восстановления.

## Layouts и Templates

### Вложенные layouts

Layouts поддерживают вложенность. Каждый layout получает `children` от вложенных страниц:

```
app/
├── layout.tsx                # Корневой layout
├── shop/
│   ├── layout.tsx            # Layout для /shop/*
│   ├── page.tsx              # /shop
│   └── [category]/
│       ├── layout.tsx        # Layout для /shop/[category]/*
│       ├── page.tsx          # /shop/[category]
│       └── [productId]/
│           └── page.tsx      # /shop/[category]/[productId]
```

**Реализация:**

```tsx
// app/layout.tsx
export default function RootLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ru">
      <body>
        <Header />
        {children}
        <Footer />
      </body>
    </html>
  )
}

// app/shop/layout.tsx
export default function ShopLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <div>
      <ShopNav />
      <main>{children}</main>
    </div>
  )
}

// app/shop/[category]/layout.tsx
export default function CategoryLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <div className="flex">
      <CategorySidebar />
      <div className="flex-1">{children}</div>
    </div>
  )
}
```

### Разница между Layout и Template

**Layout:**
- Сохраняет состояние при навигации
- Не пересоздается
- Лучше для общего UI (sidebar, header)

**Template:**
- Пересоздается при каждой навигации
- Сбрасывает состояние
- Лучше для анимаций и fetch

```tsx
// app/dashboard/layout.tsx — сохраняется
export default function DashboardLayout({ children }) {
  const [sidebarOpen, setSidebarOpen] = useState(true)
  
  return (
    <div>
      <Sidebar isOpen={sidebarOpen} />
      {children}
    </div>
  )
}

// app/dashboard/template.tsx — пересоздается
export default function DashboardTemplate({ children }) {
  useEffect(() => {
    console.log('Template mounted') // Вызовется при каждом переходе
  }, [])
  
  return <div className="animate-fade-in">{children}</div>
}
```

## Loading и Error UI

### Глобальные loading и error

```tsx
// app/loading.tsx
export default function GlobalLoading() {
  return <div className="spinner">Загрузка...</div>
}

// app/error.tsx
'use client'

export default function GlobalError({
  error,
  reset
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div>
      <h1>Ошибка приложения</h1>
      <button onClick={reset}>Перезагрузить</button>
    </div>
  )
}
```

### Вложенные loading и error

Каждый сегмент может иметь свои loading и error:

```
app/
├── loading.tsx               # Глобальный loading
├── error.tsx                 # Глобальный error
├── blog/
│   ├── loading.tsx           # Loading для /blog
│   ├── error.tsx             # Error для /blog
│   └── [slug]/
│       ├── loading.tsx       # Loading для /blog/[slug]
│       └── error.tsx         # Error для /blog/[slug]
```

### Обработка ошибок в Server Components

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPost({
  params
}: {
  params: { slug: string }
}) {
  try {
    const post = await fetchPost(params.slug)
    return <article>{post.content}</article>
  } catch (error) {
    // Ошибка автоматически попадет в error.tsx
    throw error
  }
}
```

## Route Handlers

Route Handlers позволяют создавать API endpoints:

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET() {
  const users = await db.users.findMany()
  return NextResponse.json(users)
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  const user = await db.users.create({ data: body })
  return NextResponse.json(user, { status: 201 })
}
```

### Динамические Route Handlers

```tsx
// app/api/users/[id]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const user = await db.users.findUnique({
    where: { id: params.id }
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
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const body = await request.json()
  const user = await db.users.update({
    where: { id: params.id },
    data: body
  })
  
  return NextResponse.json(user)
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  await db.users.delete({
    where: { id: params.id }
  })
  
  return new NextResponse(null, { status: 204 })
}
```

### Поддерживаемые HTTP методы

- `GET`
- `POST`
- `PUT`
- `PATCH`
- `DELETE`
- `HEAD`
- `OPTIONS`

## Навигация

### Компонент Link

Используйте `next/link` для клиентской навигации:

```tsx
import Link from 'next/link'

export default function Navigation() {
  return (
    <nav>
      <Link href="/">Главная</Link>
      <Link href="/about">О нас</Link>
      <Link href="/blog" prefetch={false}>
        Блог (без prefetch)
      </Link>
    </nav>
  )
}
```

### Динамические ссылки

```tsx
<Link href={`/blog/${post.slug}`}>
  {post.title}
</Link>

// С query параметрами
<Link href={{
  pathname: '/shop',
  query: { category: 'electronics', sort: 'price' }
}}>
  Электроника
</Link>
```

### Программная навигация

```tsx
'use client'

import { useRouter } from 'next/navigation'

export default function ClientComponent() {
  const router = useRouter()
  
  return (
    <button onClick={() => {
      router.push('/dashboard')
      // router.replace('/login') — заменить текущий маршрут
      // router.back() — назад
      // router.refresh() — обновить текущий маршрут
    }}>
      Перейти в dashboard
    </button>
  )
}
```

### Redirect в Server Components

```tsx
import { redirect } from 'next/navigation'

export default async function ProtectedPage() {
  const user = await getUser()
  
  if (!user) {
    redirect('/login')
  }
  
  return <div>Защищенная страница</div>
}
```

## Middleware

Middleware выполняется перед каждым запросом:

```tsx
// middleware.ts (в корне проекта)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token')
  
  // Защита routes
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  
  // Добавление headers
  const response = NextResponse.next()
  response.headers.set('x-custom-header', 'value')
  
  return response
}

export const config = {
  matcher: [
    '/dashboard/:path*',  // Все routes в /dashboard
    '/api/protected/:path*'
  ]
}
```

### Практические примеры middleware

**Аутентификация:**

```tsx
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { verifyToken } from '@/lib/auth'

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value
  
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  
  try {
    const user = await verifyToken(token)
    
    // Передаем информацию о пользователе в headers
    const response = NextResponse.next()
    response.headers.set('x-user-id', user.id)
    
    return response
  } catch (error) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
}

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*']
}
```

**Геолокация:**

```tsx
export function middleware(request: NextRequest) {
  const country = request.geo?.country || 'US'
  const url = request.nextUrl.clone()
  
  // Редирект на локализованную версию
  if (country === 'RU' && !url.pathname.startsWith('/ru')) {
    url.pathname = `/ru${url.pathname}`
    return NextResponse.redirect(url)
  }
  
  return NextResponse.next()
}
```

## Best Practices

### 1. Организация структуры

```
app/
├── (marketing)/          # Публичные страницы
│   ├── about/
│   ├── pricing/
│   └── blog/
├── (app)/                # Приложение
│   ├── dashboard/
│   ├── settings/
│   └── profile/
├── (auth)/               # Аутентификация
│   ├── login/
│   └── register/
└── api/                  # API endpoints
    ├── auth/
    └── users/
```

### 2. Colocation файлов

Размещайте компоненты и утилиты рядом с местом использования:

```
app/
├── blog/
│   ├── page.tsx
│   ├── components/
│   │   ├── BlogCard.tsx
│   │   └── BlogList.tsx
│   ├── utils/
│   │   └── format-date.ts
│   └── [slug]/
│       └── page.tsx
```

### 3. Использование loading states

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'
import { StatsCard, StatsCardSkeleton } from './components/StatsCard'

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      
      <Suspense fallback={<StatsCardSkeleton />}>
        <StatsCard />
      </Suspense>
    </div>
  )
}
```

### 4. Обработка ошибок

```tsx
// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation'

export default async function BlogPost({
  params
}: {
  params: { slug: string }
}) {
  const post = await getPost(params.slug)
  
  if (!post) {
    notFound()
  }
  
  if (post.isDraft) {
    return (
      <div>
        <p>Этот пост еще не опубликован</p>
        <Link href="/blog">Вернуться к списку</Link>
      </div>
    )
  }
  
  return <article>{post.content}</article>
}
```

### 5. Оптимизация навигации

```tsx
// Используйте prefetch для важных ссылок
<Link href="/dashboard" prefetch={true}>
  Dashboard
</Link>

// Отключите prefetch для редко используемых
<Link href="/admin" prefetch={false}>
  Admin
</Link>
```

### 6. Типизация params и searchParams

```tsx
// app/blog/[slug]/page.tsx
type PageProps = {
  params: { slug: string }
  searchParams: { [key: string]: string | string[] | undefined }
}

export default async function BlogPost({
  params,
  searchParams
}: PageProps) {
  const { slug } = params
  const { view = 'default' } = searchParams
  
  // ...
}
```

### 7. Избегание client components

```tsx
// Плохо: весь компонент клиентский
'use client'

export default function BlogPage() {
  const [posts, setPosts] = useState([])
  
  useEffect(() => {
    fetch('/api/posts')
      .then(r => r.json())
      .then(setPosts)
  }, [])
  
  return <ul>{posts.map(...)}</ul>
}

// Хорошо: серверный компонент
export default async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts')
    .then(r => r.json())
  
  return <ul>{posts.map(...)}</ul>
}
```

### 8. Использование generateMetadata

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next'

export async function generateMetadata({
  params
}: {
  params: { slug: string }
}): Promise<Metadata> {
  const post = await getPost(params.slug)
  
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      images: [post.coverImage]
    }
  }
}

export default async function BlogPost({ params }) {
  // ...
}
```

## Заключение

App Router в Next.js предоставляет мощную и гибкую систему маршрутизации. Ключевые преимущества:

- **Интуитивная структура**: файлы и папки = маршруты
- **Серверные компоненты**: лучшая производительность из коробки
- **Вложенные layouts**: переиспользование UI без перерендеринга
- **Параллельные роуты**: независимая загрузка секций
- **Перехватывающие роуты**: сложные UX паттерны
- **Автоматическая оптимизация**: prefetching, стриминг, code splitting

Изучайте документацию и экспериментируйте с различными паттернами для создания оптимальной архитектуры вашего приложения.
