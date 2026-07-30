# Кэширование данных в Next.js: полное руководство

Next.js расширяет нативный `fetch` API, добавляя многоуровневую систему кэширования. Понимание этой системы — ключ к производительности приложений. Этот документ разбирает каждый уровень кэша, их взаимодействие и стратегии управления.

> Этот документ — детальное руководство по кэшированию. Базовые концепты Data Fetching описаны в [Глубокое погружение в Data Fetching](./next_data_fetching.md).

## Содержание

- [4 уровня кэша в Next.js](#4-уровня-кэша-в-nextjs)
- [Data Cache](#data-cache)
  - [Где физически хранится](#где-физически-хранится)
  - [Пошаговый процесс](#пошаговый-процесс)
  - [Сценарии поведения](#сценарии-поведения)
- [Request Memoization](#request-memoization)
- [Full Route Cache](#full-route-cache)
- [Router Cache](#router-cache)
- [Взаимодействие всех уровней кэша](#взаимодействие-всех-уровней-кэша)
- [Полный цикл кэширования: таймлайн](#полный-цикл-кэширования-таймлайн)
- [Ре-валидация кэша](#ре-валидация-кэша)
  - [Time-based revalidation](#time-based-revalidation)
  - [On-demand revalidation](#on-demand-revalidation)
  - [revalidatePath vs revalidateTag](#revalidatepath-vs-revalidatetag)
- [Кэширование vs Хранение](#кэширование-vs-хранение)
- [Глобальная настройка кэширования](#глобальная-настройка-кэширования)
- [Сравнение с Nuxt.js](#сравнение-с-nuxtjs)
- [Best Practices](#best-practices)
- [Антипаттерны](#антипаттерны)

## 4 уровня кэша в Next.js

В Next.js есть **4 уровня кэша**, каждый со своим расположением и временем жизни:

```
┌─────────────────────────────────────────────────────────────────┐
│  1. REQUEST MEMOIZATION (Сервер, один render)                   │
│     └── Где: Память Node.js процесса                           │
│     └── Время жизни: Один render страницы                      │
│     └── Что: Дедупликация одинаковых fetch внутри одного render │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  2. DATA CACHE (Сервер, persistent)                             │
│     └── Где: Диск сервера (.next/cache/fetch-cache)            │
│     └── Время жизни: Permanent (пока не инвалидирован)         │
│     └── Что: Результаты fetch запросов                         │
│     └── Доступ: Только сервер (Node.js)                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  3. FULL ROUTE CACHE (Сервер, persistent)                       │
│     └── Где: Диск сервера (.next/server/app)                   │
│     └── Время жизни: Permanent (для статических страниц)       │
│     └── Что: Готовый HTML + RSC Payload                        │
│     └── Доступ: Только сервер (Node.js)                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  4. ROUTER CACHE (Клиент, браузер)                              │
│     └── Где: Память браузера (RAM)                             │
│     └── Время жизни: Сессия (до F5 или router.refresh())       │
│     └── Что: RSC Payload посещённых маршрутов                  │
│     └── Доступ: Только клиент (браузер)                        │
└─────────────────────────────────────────────────────────────────┘
```

### Сводная таблица

| Кэш | Где | Что | Duration | Как управлять |
|-----|-----|-----|----------|---------------|
| **Data Cache** | Сервер | fetch responses | Permanent / revalidate | `cache`, `next.revalidate`, `next.tags` |
| **Full Route Cache** | Сервер | HTML + RSC | Permanent | `dynamic`, `revalidate` |
| **Router Cache** | Клиент | RSC payload | Session | `router.refresh()` |
| **Request Memoization** | Сервер | Дедупликация fetch | Один render | Автоматически |

## Data Cache

Data Cache — серверный кэш результатов `fetch` запросов. Это основной механизм, который позволяет избегать повторных HTTP-запросов к API.

### Где физически хранится

```
.next/cache/fetch-cache/
├── hash-of-url-1.json
├── hash-of-url-2.json
└── hash-of-url-3.json
```

Каждый файл содержит:
```json
{
  "data": { "products": [...] },
  "timestamp": 1234567890
}
```

### Пошаговый процесс

**Что происходит при первом запросе:**

```
1. Пользователь → GET /products
   ↓
2. Next.js получает запрос
   ↓
3. Выполняет page.tsx на сервере
   ↓
4. Встречает: await fetch('https://api.example.com/products')
   ↓
5. Проверяет Data Cache:
   ┌──────────────────────────────┐
   │ .next/cache/fetch-cache/     │
   │ └── hash-of-url.json         │ ← Файл не найден
   └──────────────────────────────┘
   ↓
6. Кэш пуст → делает реальный HTTP запрос к API
   ↓
7. Получает response: { products: [...] }
   ↓
8. Сохраняет в Data Cache:
   ┌──────────────────────────────┐
   │ .next/cache/fetch-cache/     │
   │ └── hash-of-url.json         │ ← Создан файл с данными
   │     {                         │
   │       "data": {...},          │
   │       "timestamp": 1234567890 │
   │     }                         │
   └──────────────────────────────┘
   ↓
9. Рендерит HTML с данными
   ↓
10. Отправляет HTML клиенту
```

**Что происходит при повторном запросе (кэш найден):**

```
1. Пользователь → GET /products
   ↓
2. Next.js получает запрос
   ↓
3. Выполняет page.tsx на сервере
   ↓
4. Встречает: await fetch('https://api.example.com/products')
   ↓
5. Проверяет Data Cache:
   ┌──────────────────────────────┐
   │ .next/cache/fetch-cache/     │
   │ └── hash-of-url.json         │ ← Файл найден!
   └──────────────────────────────┘
   ↓
6. Проверяет срок действия:
   - Если cache: 'force-cache' → используем кэш (бессрочно)
   - Если next: { revalidate: 60 } → проверяем timestamp
     - Если прошло < 60 сек → используем кэш
     - Если прошло >= 60 сек → делаем новый запрос (фоновая ревалидация)
   ↓
7. Кэш валиден → читаем данные из файла
   ↓
8. НЕ делаем HTTP запрос к API (экономия ресурсов)
   ↓
9. Рендерит HTML с кэшированными данными
   ↓
10. Отправляет HTML клиенту
```

### Сценарии поведения

**Сценарий 1: Data Cache (force-cache)**

```tsx
const data = await fetch('https://api.example.com/data').then(r => r.json())
```

```
Первый запрос:
  1. Проверка Data Cache → не найден
  2. Реальный HTTP запрос к API
  3. Сохранение в Data Cache (навсегда)
  4. Возврат данных

Второй запрос (через день):
  1. Проверка Data Cache → найден
  2. Чтение из файла .next/cache/fetch-cache/hash.json
  3. Возврат кэшированных данных (HTTP запрос НЕ делается)
  
Результат: мгновенный ответ, нет нагрузки на API
```

**Сценарий 2: Data Cache с revalidate**

```tsx
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
}).then(r => r.json())
```

```
Первый запрос (T=0):
  1. Проверка Data Cache → не найден
  2. Реальный HTTP запрос к API
  3. Сохранение в Data Cache с timestamp
  4. Возврат данных

Второй запрос (T=30сек):
  1. Проверка Data Cache → найден
  2. Проверка timestamp: прошло 30сек < 60сек → валиден
  3. Возврат кэшированных данных

Третий запрос (T=61сек):
  1. Проверка Data Cache → найден
  2. Проверка timestamp: прошло 61сек >= 60сек → устарел
  3. Возврат кэшированных данных (пользователь видит старые)
  4. Фоновая ревалидация:
     - Реальный HTTP запрос к API
     - Обновление Data Cache
     - Следующий запрос получит новые данные
     
Результат: пользователь всегда видит данные (старые или новые),
           нет задержки на запрос к API
```

**Сценарий 3: no-store (без кэша)**

```tsx
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store'
}).then(r => r.json())
```

```
Каждый запрос:
  1. Проверка Data Cache → пропускается (no-store)
  2. Реальный HTTP запрос к API
  3. Возврат данных (кэш НЕ сохраняется)
  
Результат: всегда актуальные данные, нагрузка на API
```

## Request Memoization

В React-приложениях компоненты часто композируются из независимых частей. Каждый компонент может запрашивать данные, которые ему нужны, не зная о соседях:

```tsx
// app/dashboard/page.tsx
import { UserStats } from './UserStats'
import { UserAvatar } from './UserAvatar'

export default async function Dashboard({ params }: { params: { id: string } }) {
  return (
    <div>
      <UserStats userId={params.id} />
      <UserAvatar userId={params.id} />
    </div>
  )
}

// app/dashboard/UserStats.tsx
async function getUser(id: string) {
  return fetch(`https://api.example.com/users/${id}`).then(r => r.json())
}

export async function UserStats({ userId }: { userId: string }) {
  const user = await getUser(userId)
  return <div>Постов: {user.postsCount}</div>
}

// app/dashboard/UserAvatar.tsx
async function getUser(id: string) {
  return fetch(`https://api.example.com/users/${id}`).then(r => r.json())
}

export async function UserAvatar({ userId }: { userId: string }) {
  const user = await getUser(userId)
  return <img src={user.avatarUrl} />
}
```

**Без memoization:** 2 HTTP-запроса к одному URL (каждый компонент делает свой запрос).

**С memoization:** 1 HTTP-запрос. Next.js видит одинаковый URL и возвращает тот же Promise.

### Почему это не ошибка программиста

Передача данных через props потребовала бы изменения архитектуры:

```tsx
// Альтернатива без memoization — родитель загружает данные
export default async function Dashboard({ params }: { params: { id: string } }) {
  const user = await getUser(params.id) // Родитель загружает
  
  return (
    <div>
      <UserStats user={user} /> {/* Передаём через props */}
      <UserAvatar user={user} /> {/* Передаём через props */}
    </div>
  )
}
```

**Проблемы этого подхода:**
- Родитель должен знать, какие данные нужны детям
- Компоненты теряют самостоятельность (не переиспользовать в другом месте)
- Нарушается принцип "каждый компонент загружает свои данные"

### Реальные сценарии

**Сценарий 1: Layout и Page**

```tsx
// app/layout.tsx
export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser() // Запрос #1 (для навигации)
  
  return (
    <html>
      <body>
        <Header user={user} />
        {children}
      </body>
    </html>
  )
}

// app/profile/page.tsx
export default async function ProfilePage() {
  const user = await getCurrentUser() // Запрос #2 (тот же URL)
  
  return <Profile user={user} />
}
```

Layout не знает, какая страница внутри него. Страница не знает, что layout уже загрузил пользователя. Memoization предотвращает дублирование без изменения архитектуры.

**Сценарий 2: Вложенные компоненты**

```tsx
// app/product/page.tsx
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id)
  
  return (
    <div>
      <ProductHeader product={product} />
      <ProductDetails productId={params.id} />
    </div>
  )
}

// app/product/ProductDetails.tsx
export async function ProductDetails({ productId }: { productId: string }) {
  const product = await getProduct(productId) // Тот же URL
  
  return (
    <div>
      <p>{product.description}</p>
      <ProductReviews productId={productId} />
    </div>
  )
}
```

`ProductDetails` может использоваться на других страницах без `product` в props. Компонент самозависимый.

### Когда это действительно ошибка

```tsx
// ❌ Дублирование в одном компоненте — бессмысленно
export default async function Page() {
  const user1 = await fetch('/api/me').then(r => r.json())
  const user2 = await fetch('/api/me').then(r => r.json())
  
  return <div>{user1.name}</div>
}
```

Memoization спасает от этого, но лучше просто не писать так.

### Как это работает

```
Рендер страницы:
  UserStats: getUser('1')  →  fetch('.../users/1')  →  Promise A
  UserAvatar: getUser('1') →  Проверка memoization  →  Возврат Promise A (тот же)

Результат: только ОДИН HTTP запрос, оба компонента получают данные
```

**Где хранится:** В памяти Node.js процесса, только на время одного render.

> **Важно:** Request Memoization работает только для одинаковых `fetch` вызовов с одинаковыми URL и опциями. Если URL отличается — это разные запросы.

## Full Route Cache

Для статических страниц Next.js кэширует **весь HTML + RSC Payload**:

```
┌─────────────────────────────────────────────────────────────┐
│  .next/server/app/                                           │
│  ├── products/                                               │
│  │   ├── page_client-reference-manifest.js                  │
│  │   └── page.html  ← Готовый HTML страницы                 │
│  │   └── page.rsc   ← RSC Payload (сериализованный tree)    │
│  └── blog/                                                   │
│      └── page.html                                           │
│      └── page.rsc                                            │
└─────────────────────────────────────────────────────────────┘
```

**Что происходит:**

```
1. Build time или первый запрос (для ISR)
   ↓
2. Next.js рендерит страницу полностью
   ↓
3. Сохраняет:
   - HTML → .next/server/app/products/page.html
   - RSC Payload → .next/server/app/products/page.rsc
   ↓
4. При следующем запросе:
   - Читает готовый HTML с диска
   - НЕ выполняет page.tsx заново
   - НЕ делает fetch запросы
   - Отправляет готовый HTML
```

**Когда используется:**
- Статические страницы (нет `cookies()`, `headers()`, `searchParams`)
- ISR страницы (после revalidation)

**Когда НЕ используется:**
- Динамические страницы (есть `cookies()`, `headers()`)
- `fetch` с `cache: 'no-store'`
- `export const dynamic = 'force-dynamic'`

```tsx
// Отключить Full Route Cache для конкретной страницы
export const dynamic = 'force-dynamic'

export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store'
  }).then(r => r.json())
  
  return <div>{data.value}</div>
}
```

## Router Cache

Router Cache — **клиентский кэш** RSC Payload в браузере. Включён **по умолчанию**, не требует настройки.

### Что кэшируется

При навигации по сайту Next.js сохраняет RSC Payload (сериализованный React tree) каждого посещённого маршрута в памяти браузера:

```
┌─────────────────────────────────────────────────────────────┐
│  Браузер (Память)                                            │
│                                                              │
│  Router Cache:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  / → RSC Payload (сериализованный React tree)          │ │
│  │  /products → RSC Payload                               │ │
│  │  /products/123 → RSC Payload                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Время жизни: до F5 или router.refresh()                    │
└─────────────────────────────────────────────────────────────┘
```

### Включён по умолчанию

Router Cache работает **автоматически** для всех навигаций через `<Link>` или `router.push()`. Никакой конфигурации не требуется.

### Время жизни

| Событие | Что происходит |
|---------|----------------|
| **F5 / Ctrl+R** | Router Cache очищается полностью |
| **router.refresh()** | Router Cache очищается, данные перезагружаются |
| **Закрытие вкладки** | Кэш теряется (это память браузера, не диск) |
| **Переход по прямой ссылке** | Кэш не используется (новый entry point) |

### Когда Router Cache используется

**Сценарий 1: Возврат на посещённую страницу**

```
1. Пользователь на /products
   ↓
2. Кликает на /products/123 → загружается, сохраняется в Router Cache
   ↓
3. Возвращается назад на /products/123
   ↓
4. Router Cache: найден → мгновенная навигация (нет запроса к серверу)
```

**Сценарий 2: Навигация через `<Link>` с prefetch**

```tsx
import Link from 'next/link'

export function Navigation() {
  return (
    <Link href="/products">Товары</Link>
  )
}
```

Когда ссылка появляется в viewport, Next.js **автоматически prefetch** RSC Payload и сохраняет в Router Cache. При клике — мгновенная навигация.

### Статические vs динамические страницы

**Статические страницы** (кэшируются в Full Route Cache):

```
1. Пользователь кликает на /about
   ↓
2. Router Cache: не найден
   ↓
3. Запрос к серверу → Full Route Cache: найден
   ↓
4. Сервер возвращает готовый RSC Payload (без выполнения page.tsx)
   ↓
5. Сохраняется в Router Cache
```

**Динамические страницы** (с `cookies()`, `headers()`, `no-store`):

```
1. Пользователь кликает на /profile
   ↓
2. Router Cache: не найден
   ↓
3. Запрос к серверу → Full Route Cache: пропущен
   ↓
4. Сервер выполняет page.tsx, делает fetch с no-store
   ↓
5. Возвращает RSC Payload
   ↓
6. Сохраняется в Router Cache (но только для этой сессии)
```

> **Важно:** Router Cache работает одинаково для статических и динамических страниц. Разница в том, что динамические страницы не кэшируются на сервере (Full Route Cache), но всё равно кэшируются в браузере после первого посещения.

### Prefetching и Router Cache

**App Router (Next.js 13+):**

Next.js автоматически prefetch ссылки, которые появляются в viewport (в production mode):

```tsx
// Когда ссылка появляется на экране, Next.js prefetch RSC Payload
<Link href="/products/123">Товар 123</Link>

// При клике — мгновенная навигация из Router Cache
```

**Как это работает:**

```
1. Пользователь скроллит страницу
   ↓
2. Ссылка /products/123 появляется в viewport
   ↓
3. Next.js автоматически prefetch RSC Payload
   ↓
4. Сохраняется в Router Cache
   ↓
5. При клике — мгновенная навигация (нет запроса к серверу)
```

**Pages Router (старое поведение):**

В Pages Router prefetch происходит **только при наведении мыши** или фокусе на ссылку:

```tsx
// Pages Router: prefetch только при hover/focus
<Link href="/products/123">Товар 123</Link>
```

> **RSC Payload** — это сериализованное представление React Server Components tree, которое сервер отправляет клиенту. Содержит структуру компонентов, их props и результаты серверных fetch запросов. Клиент использует RSC Payload для гидратации и рендеринга без повторного выполнения серверного кода.

**Отключить prefetch:**

```tsx
<Link href="/products/123" prefetch={false}>
  Товар 123
</Link>
```

> **Важно:** Prefetch в viewport работает только в production mode. В development mode prefetch происходит только при наведении, чтобы избежать лишней нагрузки на сервер при разработке.

### Как очистить Router Cache

Router Cache нужно очищать, когда:
- Данные изменились через мутацию (Server Actions)
- Данные обновляются в реальном времени
- Важно видеть актуальное состояние (админ-панели, дашборды)

Router Cache НЕ мешает, когда:
- Контент статический и редко меняется
- Пользователь сам обновляет страницу (F5)
- Данные персонализированы и не кэшируются на сервере

**Способ 1: router.refresh()**

```tsx
'use client'

import { useRouter } from 'next/navigation'

export function RefreshButton() {
  const router = useRouter()

  return (
    <button onClick={() => {
      router.refresh()
    }}>
      Обновить данные
    </button>
  )
}
```

**Способ 2: После мутаций (Server Actions)**

```tsx
'use server'

import { revalidatePath } from 'next/cache'

export async function updateItem(formData: FormData) {
  await db.item.update({ /* ... */ })
  revalidatePath('/items')
}

// На клиенте
'use client'
import { useRouter } from 'next/navigation'

export function UpdateForm() {
  const router = useRouter()
  
  return (
    <form action={async (formData) => {
      await updateItem(formData)
      router.refresh() // Очищает Router Cache
    }}>
      {/* ... */}
    </form>
  )
}
```

**Способ 3: router.push() с replace**

```tsx
'use client'

import { useRouter } from 'next/navigation'

export function DeleteButton() {
  const router = useRouter()

  const handleDelete = async () => {
    await deleteItem()
    router.push('/items') // Переход на страницу (обновит Router Cache)
  }

  return <button onClick={handleDelete}>Удалить</button>
}
```

### Когда Router Cache может мешать

**Проблема: Данные не обновляются после мутации**

```tsx
// ❌ Плохо: после мутации пользователь видит старые данные
'use server'

export async function updateItem(formData: FormData) {
  await db.item.update({ /* ... */ })
  revalidatePath('/items') // Очищает серверные кэши
  // Но Router Cache в браузере остаётся!
}

// ✅ Хорошо: очищаем Router Cache на клиенте
'use client'
import { useRouter } from 'next/navigation'

export function UpdateForm() {
  const router = useRouter()
  
  return (
    <form action={async (formData) => {
      await updateItem(formData)
      router.refresh() // Очищает Router Cache
    }}>
      {/* ... */}
    </form>
  )
}
```

**Проблема: Пользователь видит устаревшие данные**

Если пользователь долго находится на сайте, Router Cache может содержать устаревшие данные. Решение:

```tsx
'use client'

import { useEffect } from 'react'
import { useRouter } from 'next/navigation'

export function AutoRefresh() {
  const router = useRouter()

  useEffect(() => {
    // Обновляем данные каждые 5 минут
    const interval = setInterval(() => {
      router.refresh()
    }, 5 * 60 * 1000)

    return () => clearInterval(interval)
  }, [router])

  return null
}
```

### Router Cache и searchParams

Router Cache **не кэширует** разные searchParams как разные страницы:

```tsx
// app/products/page.tsx
export default function ProductsPage({
  searchParams
}: {
  searchParams: { category?: string }
}) {
  // /products?category=electronics
  // /products?category=clothing
  
  return <ProductList category={searchParams.category} />
}
```

**Что происходит:**

```
1. Пользователь на /products?category=electronics
   ↓
2. Переходит на /products?category=clothing
   ↓
3. Router Cache: /products → найден (тот же маршрут)
   ↓
4. Но searchParams изменились → сервер всё равно вызывается
   ↓
5. Router Cache обновляется для новых searchParams
```

> **Важно:** Router Cache кэширует по маршруту, но при изменении searchParams сервер всё равно вызывается для получения новых данных.

### Отличие от Full Route Cache

| Аспект | Router Cache | Full Route Cache |
|--------|--------------|------------------|
| **Где** | Браузер (клиент) | Сервер (диск) |
| **Что** | RSC Payload | HTML + RSC Payload |
| **Когда** | При навигации | При build / первом запросе |
| **Время жизни** | Сессия (до F5) | Permanent (для статических) |
| **Для кого** | Один пользователь | Все пользователи |
| **Как очистить** | `router.refresh()` | `revalidatePath()` |

### Сценарии использования

**Когда Router Cache полезен:**
- Частая навигация между страницами (dashboard, admin panel)
- Возврат на ранее посещённые страницы (кнопка "назад")
- Prefetching ссылок для мгновенной навигации

**Когда Router Cache может мешать:**
- Данные часто меняются (real-time приложения)
- После мутаций (нужен `router.refresh()`)
- Персонализированный контент (нужен `no-store` на сервере)

## Взаимодействие всех уровней кэша

```
┌─────────────────────────────────────────────────────────────────┐
│                        ЗАПРОС К СЕРВЕРУ                          │
│                                                                  │
│  1. Router Cache (Клиент)                                       │
│     └── Если найден → используется (нет запроса к серверу)      │
│     └── Если не найден → запрос к серверу                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        СЕРВЕР                                    │
│                                                                  │
│  2. Full Route Cache                                            │
│     └── Если найден → возвращается готовый HTML                 │
│     └── Если не найден → выполняется page.tsx                   │
│                                                                  │
│  3. Request Memoization (внутри одного render)                  │
│     └── Дедупликация одинаковых fetch                           │
│                                                                  │
│  4. Data Cache                                                  │
│     └── Если найден и валиден → используются кэшированные данные│
│     └── Если не найден или устарел → реальный HTTP запрос       │
│     └── Результат сохраняется в Data Cache                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        ОТВЕТ КЛИЕНТУ                             │
│                                                                  │
│  5. Сохранение в Router Cache                                   │
│     └── RSC Payload сохраняется для будущих навигаций           │
└─────────────────────────────────────────────────────────────────┘
```

## Полный цикл кэширования: таймлайн

```tsx
// app/products/page.tsx
export const revalidate = 3600

export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products', {
    next: { 
      revalidate: 3600,
      tags: ['products']
    }
  }).then(r => r.json())

  return <ProductList products={products} />
}
```

**Таймлайн:**

```
T=0:     Первый запрос
         ↓
         Full Route Cache: пуст → выполняется page.tsx
         Data Cache: пуст → реальный fetch к API
         ↓
         Результат:
         - Full Route Cache: сохранён HTML + RSC
         - Data Cache: сохранён response от API
         - Router Cache: сохранён RSC Payload
         
T=1мин:  Второй запрос (тот же пользователь)
         ↓
         Router Cache: найден → используется (нет запроса к серверу)
         
T=5мин:  Новый пользователь → GET /products
         ↓
         Full Route Cache: найден → возвращается готовый HTML
         (page.tsx НЕ выполняется, fetch НЕ делается)
         
T=1час:  Прошёл revalidate=3600
         ↓
         Full Route Cache: устарел
         ↓
         Запрос → выполняется page.tsx
         ↓
         Data Cache: устарел (прошло 3600 сек)
         ↓
         Реальный fetch к API → новые данные
         ↓
         Обновление:
         - Full Route Cache: обновлён
         - Data Cache: обновлён
         
T=1час+1мин:  Server Action: revalidateTag('products')
         ↓
         Data Cache: инвалидирован (независимо от TTL)
         Full Route Cache: инвалидирован
         ↓
         Следующий запрос → реальный fetch → новые данные
```

## Ре-валидация кэша

Ре-валидация — процесс обновления кэшированных данных до истечения их срока жизни. Next.js предоставляет два механизма: по таймеру (автоматически) и по событию (вручную).

### Time-based revalidation

**Что это:** Автоматическое обновление кэша через заданный интервал времени (в секундах). Когда кэш устаревает, Next.js возвращает старые данные пользователю, а в фоне делает новый запрос и обновляет кэш. Следующий пользователь получит уже свежие данные.

**Когда применять:**
- Данные меняются предсказуемо и не критично к задержке (каталог товаров, список статей, курсы валют)
- Нет возможности отслеживать изменения на стороне источника данных
- Хочется снизить нагрузку на API, жертвуя актуальностью на секунды/минуты

**Как применять:**

```tsx
// Вариант 1: через опцию fetch
export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products', {
    next: { revalidate: 60 }
  }).then(r => r.json())

  return <ProductList products={products} />
}

// Вариант 2: через segment config (применяется ко всем fetch на странице)
export const revalidate = 60

export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products').then(r => r.json())
  return <ProductList products={products} />
}
```

```
T=0:     Первый запрос → реальный fetch → кэш сохранён
T=30:    Второй запрос → кэш валиден (30 < 60) → возврат из кэша
T=61:    Третий запрос → кэш устарел (61 >= 60):
         1. Пользователь получает СТАРЫЕ данные из кэша (мгновенно)
         2. В фоне → реальный fetch к API → обновление кэша
         3. Следующий запрос получит НОВЫЕ данные
```

> **Важно:** Time-based revalidation работает по принципу **stale-while-revalidate** — пользователь никогда не ждёт запрос к API, но может получить слегка устаревшие данные.

### On-demand revalidation

**Что это:** Ручная инвалидация кэша по событию. Вы сами решаете, когда данные устарели — например, после создания, обновления или удаления записи в БД. Кэш удаляется немедленно, и следующий запрос получит свежие данные.

**Когда применять:**
- Данные меняются через пользовательские действия (CRUD-операции в админке)
- Актуальность данных критична (цены, наличие, статусы заказов)
- Есть webhook или событие от CMS/источника данных
- Нужно обновить кэш сразу после публикации статьи/товара

**Как применять:**

```tsx
// app/actions/revalidate.ts
'use server'

import { revalidatePath } from 'next/cache'
import { revalidateTag } from 'next/cache'

export async function revalidateProducts() {
  revalidatePath('/products')
}

export async function revalidateProductsByTag() {
  revalidateTag('products')
}
```

#### revalidatePath

**Что это:** Инвалидирует Data Cache и Full Route Cache для конкретного URL-пути. Все кэшированные данные и отрендеренный HTML для этого пути удаляются.

**Когда применять:**
- Простая структура данных — одна страница = один источник данных
- CRUD-операции, где чётко известно, какая страница устарела
- Нужно обновить конкретный маршрут после мутации

**Как применять:**

```tsx
// app/actions/update-product.ts
'use server'

import { revalidatePath } from 'next/cache'

export async function updateProduct(formData: FormData) {
  const id = formData.get('id') as string
  const name = formData.get('name') as string

  await db.product.update({
    where: { id },
    data: { name },
  })

  revalidatePath('/products')
  revalidatePath(`/products/${id}`)
}
```

```
revalidatePath('/products')       → инвалидирует /products
revalidatePath('/products/123')   → инвалидирует /products/123
revalidatePath('/', 'layout')     → инвалидирует корневой layout и все вложенные страницы
```

> **Важно:** `revalidatePath` инвалидирует Full Route Cache и Data Cache для указанного пути, но **не очищает Router Cache** в браузере пользователя. Для этого нужен `router.refresh()` на клиенте.

#### revalidateTag

**Что это:** Инвалидирует Data Cache для всех `fetch`-запросов, помеченных указанным тегом. Работает независимо от того, на каких страницах используются эти запросы — один тег может объединять данные с разных маршрутов.

**Когда применять:**
- Одни и те же данные используются на нескольких страницах (товары на главной, в каталоге, в рекомендациях)
- Сложные зависимости — один тег объединяет несколько fetch с разных источников
- Нужна гранулярная инвалидация по логическим группам данных, а не по URL
- Данные обновляются через webhook (CMS прислал событие «статья опубликована»)

**Как применять:**

```tsx
// app/products/page.tsx — помечаем кэш тегом
export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products', {
    next: { tags: ['products'] }
  }).then(r => r.json())

  return <ProductList products={products} />
}

// app/home/page.tsx — те же данные, тот же тег
export default async function HomePage() {
  const featured = await fetch('https://api.example.com/products?featured=true', {
    next: { tags: ['products'] }
  }).then(r => r.json())

  return <FeaturedProducts products={featured} />
}

// app/actions/update-product.ts — инвалидируем по тегу
'use server'

import { revalidateTag } from 'next/cache'

export async function updateProduct(formData: FormData) {
  await db.product.update({ /* ... */ })

  revalidateTag('products')
}
```

```
revalidateTag('products')  → инвалидирует ВСЕ fetch с тегом 'products'
                             независимо от страницы: /products, /home, /recommendations
```

> **Совет:** Один fetch может иметь несколько тегов: `next: { tags: ['products', 'homepage'] }`. Это позволяет инвалидировать данные разными событиями.

### revalidatePath vs revalidateTag

| Аспект | revalidatePath | revalidateTag |
|--------|---------------|---------------|
| **Что инвалидирует** | Конкретные пути URL | Все fetch с указанным тегом |
| **Гранулярность** | По маршруту | По логической группе данных |
| **Связь с источником** | Привязана к URL страницы | Привязана к данным, а не к странице |
| **Масштаб** | Один или несколько путей | Все страницы, где используется тег |
| **Использование** | Простые CRUD | Сложные зависимости данных |
| **Пример** | `revalidatePath('/products')` | `revalidateTag('products')` |

```tsx
// revalidatePath — инвалидирует все страницы по пути
revalidatePath('/products')
revalidatePath('/products/123')

// revalidateTag — инвалидирует все fetch с тегом
// Независимо от того, на какой странице они используются
revalidateTag('products')
```

**Когда что выбрать:**

```
Данные живут на одной странице?
  → revalidatePath('/products')

Одни данные используются на 3+ страницах?
  → revalidateTag('products')

Нужна точечная инвалидация конкретного товара?
  → revalidatePath('/products/123')

Webhook от CMS говорит «обнови всё связанное с товарами»?
  → revalidateTag('products')
```

## Кэширование vs Хранение

В Next.js есть два уровня работы с данными:

### Data Cache (кэширование данных)

```
┌─────────────────────────────────────────────────────────┐
│                   Data Cache                              │
│                                                           │
│  fetch(url, { cache: 'force-cache' })                    │
│  fetch(url, { next: { revalidate: 60 } })                │
│                                                           │
│  Где: Сервер (persistent across requests)                 │
│  Что: Результат fetch запросов                            │
│  Когда: При build и runtime                               │
│  Как работает: Кэширует response от fetch                  │
└─────────────────────────────────────────────────────────┘
```

### Request Memoization (мемоизация запросов)

```tsx
async function getUser(id: string) {
  return fetch(`https://api.example.com/users/${id}`).then(r => r.json())
}

export default async function Page() {
  const [user1, user2] = await Promise.all([
    getUser('1'),
    getUser('1'),
  ])

  return <div>{user1.name}</div>
}
```

### Full Route Cache

```
┌─────────────────────────────────────────────────────────┐
│                  Full Route Cache                         │
│                                                           │
│  Кэширует: HTML + RSC Payload целиком                    │
│  Где: Сервер                                              │
│  Когда: При build (статические страницы)                  │
│  Как отключить:                                           │
│    - cookies() / headers() в серверном компоненте         │
│    - fetch с cache: 'no-store'                            │
│    - export const dynamic = 'force-dynamic'               │
└─────────────────────────────────────────────────────────┘
```

### Router Cache (клиентский)

```
┌─────────────────────────────────────────────────────────┐
│                  Router Cache (Браузер)                   │
│                                                           │
│  Кэширует: RSC Payload visited routes                    │
│  Где: Клиент (браузер)                                   │
│  Когда: При навигации                                     │
│  Duration: Сессия (до обновления страницы)                │
│  Как очистить: router.refresh()                          │
└─────────────────────────────────────────────────────────┘
```

## Глобальная настройка кэширования

```tsx
// next.config.js
module.exports = {
  experimental: {
    // В Next.js 15+ fetch по умолчанию не кэшируется при build
    // Для кэширования нужно явно указывать cache: 'force-cache'
    // или next: { revalidate: ... }
  }
}
```

> **Важно:** В Next.js 15+ поведение по умолчанию изменилось. `fetch` больше **не кэшируется** по умолчанию при `build`. Для кэширования нужно явно указывать `cache: 'force-cache'` или `next: { revalidate: ... }`.

## Сравнение с Nuxt.js

> **Nuxt:** В Nuxt кэширование работает иначе. `useFetch` автоматически кэширует данные в **payload** (передаётся с SSR). На клиенте данные восстанавливаются из payload без повторного запроса. Для HTTP-кэширования используется `useFetch(url, { $fetch: { headers: { 'Cache-Control': '...' } } })` или Nitro caching rules в `nuxt.config.ts`. Нет встроенного Data Cache как в Next.js — кэширование делегируется CDN/браузеру или серверным middleware.

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
    '/api/data': { swr: true },
    '/blog/**': { isr: 3600 },
    '/static/**': { static: true },
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

> **Nuxt:** В Nuxt нет прямого аналога `revalidatePath` / `revalidateTag`. Для обновления данных используется: 1) `refreshNuxtData()` — принудительно перезагружает данные на клиенте; 2) `clearNuxtData()` — очищает кэш; 3) Nitro caching rules (`routeRules` в `nuxt.config.ts`) для серверного кэширования с `swr`, `static`, `isr`. ISR в Nuxt настраивается через `routeRules: { '/blog/**': { isr: 3600 } }`.

## Best Practices

### 1. Выбирай стратегию кэширования осознанно

```tsx
// Статические данные — кэшируем навсегда
const features = await fetch('https://api.example.com/features', {
  cache: 'force-cache',
}).then(r => r.json())

// Динамические данные — без кэша
const user = await fetch('https://api.example.com/me', {
  cache: 'no-store',
  headers: { Authorization: `Bearer ${token}` },
}).then(r => r.json())

// Редко меняющиеся данные — с revalidate
const products = await fetch('https://api.example.com/products', {
  next: { revalidate: 3600 },
}).then(r => r.json())
```

### 2. Используй теги для гранулярной инвалидации

```tsx
// Страница товаров
const products = await fetch('/api/products', {
  next: { tags: ['products'] }
}).then(r => r.json())

// Страница рекомендаций (использует те же данные)
const recommendations = await fetch('/api/products?featured=true', {
  next: { tags: ['products'] }
}).then(r => r.json())

// При обновлении — инвалидируем все сразу
revalidateTag('products')
```

### 3. Разделяй статический и динамический контент

```tsx
import { Suspense } from 'react'

export default async function HomePage() {
  return (
    <div>
      {/* Статический — кэшируется */}
      <HeroSection />

      {/* Динамический — без кэша */}
      <Suspense fallback={<FeedSkeleton />}>
        <UserFeed />
      </Suspense>
    </div>
  )
}
```

### 4. Очищай Router Cache после мутаций

```tsx
'use client'

import { useRouter } from 'next/navigation'

export function DeleteButton({ id }: { id: string }) {
  const router = useRouter()

  const handleDelete = async () => {
    await deleteItem(id)
    router.refresh()
  }

  return <button onClick={handleDelete}>Удалить</button>
}
```

## Антипаттерны

### 1. revalidate: 0

```tsx
// ❌ Бессмысленно — кэш сразу невалиден
const data = await fetch('/api/data', {
  next: { revalidate: 0 }
}).then(r => r.json())

// ✅ Используй no-store для динамических данных
const data = await fetch('/api/data', {
  cache: 'no-store'
}).then(r => r.json())

// ✅ Или revalidate с разумным TTL
const data = await fetch('/api/data', {
  next: { revalidate: 60 }
}).then(r => r.json())
```

### 2. Забывать про Router Cache

```tsx
// ❌ После мутации данные не обновятся из-за Router Cache
export async function updateItem(formData: FormData) {
  await db.item.update({ /* ... */ })
  revalidatePath('/items')
  // Пользователь может не увидеть изменения!
}

// ✅ Добавь router.refresh() на клиенте
'use client'
import { useRouter } from 'next/navigation'

export function UpdateForm() {
  const router = useRouter()
  
  return (
    <form action={async (formData) => {
      await updateItem(formData)
      router.refresh()
    }}>
      {/* ... */}
    </form>
  )
}
```

### 3. Кэшировать данные пользователя

```tsx
// ❌ Данные пользователя в кэше — другие пользователи увидят чужие данные
export default async function ProfilePage() {
  const user = await fetch('https://api.example.com/me').then(r => r.json())
  return <div>{user.name}</div>
}

// ✅ Всегда no-store для данных конкретного пользователя
export default async function ProfilePage() {
  const token = cookies().get('auth-token')?.value
  const user = await fetch('https://api.example.com/me', {
    cache: 'no-store',
    headers: { Authorization: `Bearer ${token}` },
  }).then(r => r.json())
  return <div>{user.name}</div>
}
```

### 4. Один тег для всех данных

```tsx
// ❌ Слишком широкий тег — инвалидирует всё
revalidateTag('data')

// ✅ Гранулярные теги
revalidateTag('products')
revalidateTag('categories')
revalidateTag('user-profile')
```
