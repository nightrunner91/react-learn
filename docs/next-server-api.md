# Next.js Server API - Полное руководство

## next/headers

### headers()

**Назначение:** Чтение HTTP заголовков входящего запроса в Server Components.

**Импорт:**
```tsx
import { headers } from 'next/headers'
```

**Пример использования:**
```tsx
import { headers } from 'next/headers'

export default async function Page() {
  const headersList = await headers()
  const userAgent = headersList.get('user-agent')
  const authorization = headersList.get('authorization')
  
  return <div>User Agent: {userAgent}</div>
}
```

**Важно:**
- Асинхронная функция (начиная с Next.js 15)
- Возвращает **read-only** объект Headers
- Нельзя изменять заголовки (нет методов `set` или `delete`)
- Использование делает маршрут динамическим (отключает статическую генерацию)

**Доступные методы:**
- `get(name)` - получить значение заголовка
- `has(name)` - проверить наличие заголовка
- `entries()` - итератор всех пар ключ/значение
- `keys()` - итератор ключей
- `values()` - итератор значений
- `forEach()` - выполнить функцию для каждой пары

---

### cookies()

**Назначение:** Чтение и запись HTTP cookies в Server Components, Server Functions и Route Handlers.

**Импорт:**
```tsx
import { cookies } from 'next/headers'
```

**Примеры:**

#### Чтение cookie:
```tsx
import { cookies } from 'next/headers'

export default async function Page() {
  const cookieStore = await cookies()
  const theme = cookieStore.get('theme')
  
  return <div>Current theme: {theme?.value}</div>
}
```

#### Запись cookie (только в Server Functions или Route Handlers):
```tsx
'use server'

import { cookies } from 'next/headers'

export async function setTheme(theme: string) {
  const cookieStore = await cookies()
  cookieStore.set('theme', theme, { 
    secure: true,
    httpOnly: true,
    maxAge: 60 * 60 * 24 * 30 // 30 дней
  })
}
```

#### Удаление cookie:
```tsx
'use server'

import { cookies } from 'next/headers'

export async function deleteTheme() {
  const cookieStore = await cookies()
  cookieStore.delete('theme')
}
```

**Важно:**
- Асинхронная функция (начиная с Next.js 15)
- В Server Components можно только **читать** cookies
- В Server Functions и Route Handlers можно **читать и писать**
- Использование делает маршрут динамическим

**Доступные методы:**
- `get(name)` - получить cookie по имени
- `getAll()` - получить все cookies
- `has(name)` - проверить наличие cookie
- `set(name, value, options)` - установить cookie
- `delete(name)` - удалить cookie

**Опции при установке cookie:**
- `expires` - дата истечения
- `maxAge` - время жизни в секундах
- `domain` - домен
- `path` - путь (по умолчанию `/`)
- `secure` - только HTTPS
- `httpOnly` - недоступна для JavaScript
- `sameSite` - контроль кросс-доменных запросов (`lax`, `strict`, `none`)

---

### draftMode()

**Назначение:** Управление режимом черновика для предпросмотра неопубликованного контента из CMS.

**Импорт:**
```tsx
import { draftMode } from 'next/headers'
```

**Примеры:**

#### Проверка статуса:
```tsx
import { draftMode } from 'next/headers'

export default async function Page() {
  const { isEnabled } = await draftMode()
  
  return (
    <div>
      Draft Mode is {isEnabled ? 'Enabled' : 'Disabled'}
    </div>
  )
}
```

#### Включение режима (в Route Handler):
```tsx
import { draftMode } from 'next/headers'

export async function GET() {
  const draft = await draftMode()
  draft.enable()
  return new Response('Draft mode enabled')
}
```

#### Выключение режима:
```tsx
import { draftMode } from 'next/headers'

export async function GET() {
  const draft = await draftMode()
  draft.disable()
  return new Response('Draft mode disabled')
}
```

**Важно:**
- Асинхронная функция (начиная с Next.js 15)
- Работает через cookie `__prerender_bypass`
- При включении обходит кэширование
- Новое значение cookie генерируется при каждом `next build`

---

## next/navigation

### redirect()

**Назначение:** Перенаправление пользователя на другой URL.

**Импорт:**
```tsx
import { redirect } from 'next/navigation'
```

**Пример:**
```tsx
import { redirect } from 'next/navigation'

export default async function Page({ params }) {
  const { id } = await params
  const data = await fetchData(id)
  
  if (!data) {
    redirect('/login')
  }
  
  return <div>{data.title}</div>
}
```

**Параметры:**
```tsx
redirect(path: string, type?: 'replace' | 'push')
```

**Важно:**
- Бросает ошибку `NEXT_REDIRECT` - вызывайте **вне** блока `try/catch`
- В Server Actions использует 303 redirect
- В остальных случаях использует 307 redirect
- Поддерживает абсолютные URL для внешних ссылок
- В Client Components работает только при рендеринге, не в обработчиках событий

**Альтернатива:** `permanentRedirect()` для 308 redirect

---

### notFound()

**Назначение:** Отображение страницы 404 (not-found).

**Импорт:**
```tsx
import { notFound } from 'next/navigation'
```

**Пример:**
```tsx
import { notFound } from 'next/navigation'

export default async function Page({ params }) {
  const { id } = await params
  const user = await fetchUser(id)
  
  if (!user) {
    notFound()
  }
  
  return <div>{user.name}</div>
}
```

**Важно:**
- Бросает ошибку `NEXT_HTTP_ERROR_FALLBACK;404`
- Автоматически добавляет `<meta name="robots" content="noindex" />`
- Рендерит файл `not-found.tsx` в текущем сегменте маршрута
- Не требует `return` благодаря TypeScript типу `never`

---

### useRouter()

**Назначение:** Программная навигация в Client Components.

**Импорт:**
```tsx
import { useRouter } from 'next/navigation'
```

**Пример:**
```tsx
'use client'

import { useRouter } from 'next/navigation'

export default function Page() {
  const router = useRouter()
  
  return (
    <button onClick={() => router.push('/dashboard')}>
      Go to Dashboard
    </button>
  )
}
```

**Доступные методы:**
- `push(href, options)` - навигация с добавлением в историю
- `replace(href, options)` - навигация без добавления в историю
- `refresh()` - обновление текущей страницы
- `prefetch(href)` - предварительная загрузка маршрута
- `back()` - назад
- `forward()` - вперед

**Важно:**
- Работает только в Client Components
- Для навигации предпочитайте `<Link>` компонент
- Опция `scroll: false` отключает прокрутку вверх

---

## next/cache

### revalidatePath()

**Назначение:** Инвалидация кэшированных данных для конкретного пути.

**Импорт:**
```tsx
import { revalidatePath } from 'next/cache'
```

**Примеры:**

#### Инвалидация конкретной страницы:
```tsx
import { revalidatePath } from 'next/cache'

revalidatePath('/blog/post-1')
```

#### Инвалидация всех страниц с динамическим сегментом:
```tsx
import { revalidatePath } from 'next/cache'

revalidatePath('/blog/[slug]', 'page')
```

#### Инвалидация layout:
```tsx
import { revalidatePath } from 'next/cache'

revalidatePath('/blog/[slug]', 'layout')
```

#### Использование в Server Action:
```tsx
'use server'

import { revalidatePath } from 'next/cache'

export async function updatePost() {
  await savePost()
  revalidatePath('/blog')
}
```

**Параметры:**
```tsx
revalidatePath(path: string, type?: 'page' | 'layout')
```

**Важно:**
- Работает только в Server Functions и Route Handlers
- При использовании с rewrites указывайте destination path
- Инвалидация происходит при следующем посещении пути

---

### revalidateTag()

**Назначение:** Инвалидация кэшированных данных по тегам.

**Импорт:**
```tsx
import { revalidateTag } from 'next/cache'
```

**Пример:**

#### Добавление тега к fetch:
```tsx
const posts = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] }
})
```

#### Инвалидация по тегу:
```tsx
'use server'

import { revalidateTag } from 'next/cache'

export async function addPost() {
  await savePost()
  revalidateTag('posts', 'max')
}
```

**Параметры:**
```tsx
revalidateTag(tag: string, profile?: string | { expire?: number })
```

**Важно:**
- Рекомендуется использовать `profile: 'max'` для stale-while-revalidate
- Инвалидирует данные во всех страницах, использующих этот тег
- Работает только в Server Functions и Route Handlers

---

## Клиентские хуки

### usePathname()

**Назначение:** Получение текущего пути URL в Client Components.

```tsx
'use client'

import { usePathname } from 'next/navigation'

export default function Page() {
  const pathname = usePathname()
  return <div>Current path: {pathname}</div>
}
```

---

### useSearchParams()

**Назначение:** Получение параметров поиска (query string) в Client Components.

```tsx
'use client'

import { useSearchParams } from 'next/navigation'

export default function Page() {
  const searchParams = useSearchParams()
  const query = searchParams.get('q')
  
  return <div>Search: {query}</div>
}
```

**Важно:** Вызывает клиентский рендеринг до ближайшего Suspense boundary

---

## Сводная таблица

| API | Импорт | Где работает | Назначение |
|-----|--------|--------------|------------|
| `headers()` | `next/headers` | Server Components | Чтение HTTP заголовков |
| `cookies()` | `next/headers` | Server Components, Server Functions | Работа с cookies |
| `draftMode()` | `next/headers` | Server Components, Route Handlers | Управление режимом черновика |
| `redirect()` | `next/navigation` | Server/Client Components, Server Actions | Перенаправление |
| `notFound()` | `next/navigation` | Server/Client Components | Отображение 404 |
| `useRouter()` | `next/navigation` | Client Components | Программная навигация |
| `usePathname()` | `next/navigation` | Client Components | Текущий путь |
| `useSearchParams()` | `next/navigation` | Client Components | Параметры поиска |
| `revalidatePath()` | `next/cache` | Server Functions, Route Handlers | Инвалидация кэша по пути |
| `revalidateTag()` | `next/cache` | Server Functions, Route Handlers | Инвалидация кэша по тегам |

---

## Best Practices

### 1. Работа с cookies

```tsx
// ✅ Правильно: чтение в Server Component
export default async function Page() {
  const cookieStore = await cookies()
  const theme = cookieStore.get('theme')
  return <div>{theme?.value}</div>
}

// ✅ Правильно: запись в Server Action
'use server'
export async function updateTheme(theme: string) {
  const cookieStore = await cookies()
  cookieStore.set('theme', theme)
}

// ❌ Неправильно: попытка записи в Server Component
export default async function Page() {
  const cookieStore = await cookies()
  cookieStore.set('theme', 'dark') // Ошибка!
}
```

### 2. Обработка ошибок с redirect

```tsx
// ✅ Правильно: redirect вне try/catch
export default async function Page({ params }) {
  const { id } = await params
  
  try {
    const data = await fetchData(id)
    if (!data) redirect('/not-found')
    return <div>{data.title}</div>
  } catch (error) {
    // Обработка других ошибок
    console.error(error)
    return <div>Error occurred</div>
  }
}

// ❌ Неправильно: redirect внутри try/catch
export default async function Page({ params }) {
  const { id } = await params
  
  try {
    const data = await fetchData(id)
    if (!data) redirect('/not-found') // Будет пойман!
  } catch (error) {
    // redirect бросает ошибку, которая будет поймана здесь
  }
}
```

### 3. Комбинирование revalidatePath и revalidateTag

```tsx
'use server'

import { revalidatePath, revalidateTag } from 'next/cache'

export async function updatePost(id: string) {
  await savePost(id)
  
  // Инвалидация конкретной страницы
  revalidatePath(`/blog/${id}`)
  
  // Инвалидация всех страниц с постами
  revalidateTag('posts', 'max')
}
```

### 4. Использование headers для авторизации

```tsx
import { headers } from 'next/headers'

export default async function Page() {
  const headersList = await headers()
  const authorization = headersList.get('authorization')
  
  const res = await fetch('https://api.example.com/user', {
    headers: { authorization }
  })
  
  const user = await res.json()
  return <div>{user.name}</div>
}
```

---

## Динамический рендеринг

Использование следующих API автоматически делает маршрут динамическим:
- `headers()`
- `cookies()`
- `draftMode()`
- `searchParams` в page components

Это означает, что страница будет рендериться при каждом запросе, а не кэшироваться статически.

---

## Дополнительные ресурсы

- [Официальная документация Next.js](https://nextjs.org/docs)
- [Server Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Caching](https://nextjs.org/docs/app/getting-started/caching)
- [Server Actions](https://nextjs.org/docs/app/getting-started/mutating-data)
