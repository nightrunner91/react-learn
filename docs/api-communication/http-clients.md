# HTTP-клиенты: fetch, axios, ky и другие способы работы с API

## Содержание

1. [Зачем отдельный HTTP-клиент](#зачем-отдельный-http-клиент)
2. [fetch — нативный API браузера](#fetch--нативный-api-браузера)
3. [axios — классическая библиотека](#axios--классическая-библиотека)
4. [ky — лёгкий обёрточный клиент](#ky--лёгкий-обёрточный-клиент)
5. [ofetch — универсальный клиент от экосистемы Nuxt](#ofetch--универсальный-клиент-от-экосистемы-nuxt)
6. [Сравнение библиотек](#сравнение-библиотек)
7. [Когда что использовать](#когда-что-использовать)
8. [Конфигурация через переменные окружения](#конфигурация-через-переменные-окружения)
9. [Паттерн централизованного API-клиента](#паттерн-централизованного-api-клиента)
10. [Лучшие практики](#лучшие-практики)
11. [Антипаттерны](#антипаттерны)

---

## Зачем отдельный HTTP-клиент

В любом фронтенд-приложении рано или поздно появляется слой, отвечающий за общение с сервером. На первых порах можно обходиться голым `fetch`, но по мере роста проекта требуется:

- **Базовый URL** — не повторять `https://api.example.com` в каждом запросе.
- **Заголовки по умолчанию** — `Content-Type`, `Authorization`, `Accept-Language`.
- **Обработка ошибок** — единообразное превращение HTTP-ответов в понятные исключения.
- **Interceptors** — логирование, токены, request ID, retry.
- **Отмена запросов** — `AbortController` или встроенные механизмы библиотеки.
- **Типизация** — удобная работа с TypeScript.

Хороший HTTP-клиент решает эти задачи без самописной обёртки над `fetch`.

---

## fetch — нативный API браузера

`fetch` — встроенный в браузер и Node.js 18+ API для HTTP-запросов. Он низкоуровневый, стандартизированный и не требует установки.

### Базовый пример

```js
const response = await fetch('/api/users');

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}

const users = await response.json();
```

### Преимущества

- Встроен в браузер и современный Node.js.
- Поддерживает `AbortController` для отмены.
- Работает со Streams и FormData.
- Нет лишних килобайтов в бандле.

### Недостатки

- Не бросает исключение на HTTP-ошибках (`4xx`, `5xx`) — нужно проверять `response.ok` вручную.
- Нет встроенных interceptors.
- Нет автоматического преобразования тела запроса в JSON.
- Более многословный API.
- Отсутствие встроенного retry и timeout.

### fetch с POST и JSON

```js
const response = await fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ name: 'Alice', email: 'alice@example.com' }),
});

if (!response.ok) {
  const error = await response.json();
  throw new Error(error.message);
}

const user = await response.json();
```

### fetch с AbortController

```js
const controller = new AbortController();

const response = await fetch('/api/users', {
  signal: controller.signal,
});

// Отмена через 5 секунд
setTimeout(() => controller.abort(), 5000);
```

---

## axios — классическая библиотека

`axios` — самая популярная HTTP-библиотека для JavaScript. Она работает и в браузере, и в Node.js, имеет удобный API и богатую экосистему.

### Базовый пример

```js
import axios from 'axios';

const { data: users } = await axios.get('/api/users');
```

### Преимущества

- Автоматическая сериализация JSON.
- Бросает исключение на `status >= 400`.
- Interceptors для запросов и ответов.
- Поддержка timeout из коробки.
- Поддержка transformRequest / transformResponse.
- Старые, но знакомые CancelToken и современный AbortController.
- Отличная поддержка старых браузеров (XMLHttpRequest под капотом).

### Недостатки

- Размер бандла (~13 KB gzipped).
- Некоторые фичи устарели (CancelToken).
- В 2024–2026 годах сообщество активно обсуждает, стоит ли использовать axios в новых проектах, потому что `fetch` покрывает большинство сценариев.

### axios instance

```js
const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

const { data: user } = await api.get('/users/123');
```

### Interceptors

```js
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### Отмена запроса в axios

```js
const controller = new AbortController();

const { data } = await axios.get('/api/users', {
  signal: controller.signal,
});

controller.abort();
```

---

## ky — лёгкий обёрточный клиент

`ky` — это небольшая обёртка над `fetch` от создателя `got`. Сохраняет всё хорошее от `fetch`, но добавляет удобство `axios`.

### Базовый пример

```js
import ky from 'ky';

const users = await ky.get('/api/users').json();
```

### Преимущества

- Маленький размер (~4 KB gzipped).
- Бросает исключение на HTTP-ошибках.
- Автоматический `.json()` через метод `.json()`.
- Удобный API для retry, timeout, hooks.
- Поддержка `AbortController`.
- Написан на `fetch`, поэтому использует нативные возможности браузера.

### Недостатки

- Нет поддержки старых браузеров без fetch.
- Экосистема плагинов меньше, чем у axios.
- Некоторые привычные фичи axios (например, transformResponse) отсутствуют.

### ky instance

```js
const api = ky.create({
  prefixUrl: 'https://api.example.com',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
  retry: {
    limit: 3,
    methods: ['get'],
    statusCodes: [408, 413, 429, 500, 502, 503, 504],
  },
});

const user = await api.get('users/123').json();
```

### Hooks в ky

```js
const api = ky.create({
  hooks: {
    beforeRequest: [
      (request) => {
        request.headers.set('Authorization', `Bearer ${getToken()}`);
      },
    ],
    afterResponse: [
      async (request, options, response) => {
        if (!response.ok) {
          console.error('API error:', response.status);
        }
        return response;
      },
    ],
  },
});
```

### Отмена в ky

```js
const controller = new AbortController();

const users = await ky.get('/api/users', {
  signal: controller.signal,
}).json();
```

---

## ofetch — универсальный клиент от экосистемы Nuxt

`ofetch` (ранее `$fetch` в Nuxt 2/3) — HTTP-клиент от UnJS, используемый в Nuxt 3. Он основан на `fetch` и заточен под SSR/CSR изоморфные запросы.

### Базовый пример

```js
import { ofetch } from 'ofetch';

const users = await ofetch('/api/users');
```

### Преимущества

- Работает одинаково в браузере и на сервере (SSR).
- Автоматический парсинг JSON.
- Встроенный retry.
- Удобная обработка ошибок через `onRequest`, `onResponse`, `onRequestError`, `onResponseError`.
- Поддержка `baseURL`.

### Недостатки

- Тесно связан с экосистемой UnJS/Nuxt.
- Меньше документации и примеров вне Nuxt.
- Не всегда нужен в React/Next.js проектах.

### ofetch instance

```js
const api = ofetch.create({
  baseURL: 'https://api.example.com',
  retry: 3,
  headers: {
    Accept: 'application/json',
  },
  async onRequest({ options }) {
    options.headers.set('Authorization', `Bearer ${getToken()}`);
  },
  async onResponseError({ response }) {
    if (response.status === 401) {
      await refreshToken();
    }
  },
});

const user = await api('/users/123');
```

---

## Сравнение библиотек

| Критерий | fetch | axios | ky | ofetch |
|----------|-------|-------|-----|--------|
| Размер | 0 KB | ~13 KB | ~4 KB | ~5 KB |
| База | Нативный API | XMLHttpRequest | fetch | fetch |
| JSON | Ручной `.json()` | Авто | Метод `.json()` | Авто |
| Ошибки HTTP | Не бросает | Бросает | Бросает | Бросает |
| Interceptors | Нет | Да | Hooks | Hooks |
| Retry | Нет | Нет | Да | Да |
| Timeout | Нет | Да | Да | Да |
| AbortController | Да | Да | Да | Да |
| Старые браузеры | Зависит от браузера | Да | Нет | Нет |
| TypeScript | Средне | Хорошо | Хорошо | Хорошо |
| SSR | Да | Да | Да | Отлично |

---

## Когда что использовать

### Выбирай fetch, если:

- Проект маленький и запросов мало.
- Важен размер бандла (0 KB).
- Не нужны interceptors и retry.
- Вы готовы писать небольшую обёртку самостоятельно.

### Выбирай axios, если:

- Команда уже знакома с axios.
- Нужна поддержка старых браузеров.
- Используются сложные interceptors.
- Проект большой и миграция на fetch-обёртку нецелесообразна.

### Выбирай ky, если:

- Хочется лёгкую альтернативу axios.
- Важен размер бандла.
- Целевые браузеры поддерживают fetch.
- Нужен встроенный retry.

### Выбирай ofetch, если:

- Проект на Nuxt 3.
- Нужны изоморфные запросы (SSR + CSR).
- Хочется минимальной конфигурации.

---

## Конфигурация через переменные окружения

Адреса API не должны быть захардкожены в коде. Для разных окружений — development, staging, production — используются разные `baseURL`. Хранить их принято в переменных окружения.

### Префиксы для популярных фреймворков

| Фреймворк / сборщик | Префикс для клиентских переменных | Пример |
|---------------------|-----------------------------------|--------|
| Vite | `VITE_` | `VITE_API_URL` |
| Next.js | `NEXT_PUBLIC_` | `NEXT_PUBLIC_API_URL` |
| Nuxt | `NUXT_PUBLIC_` | `NUXT_PUBLIC_API_BASE` |
| Create React App | `REACT_APP_` | `REACT_APP_API_URL` |
| Vanilla webpack | настраивается вручную | `API_URL` |

### Пример `.env`

```bash
# .env.local
VITE_API_URL=https://api.example.com
```

### Использование в API-клиенте

```ts
import ky from 'ky';

export const api = ky.create({
  prefixUrl: import.meta.env.VITE_API_URL,
  timeout: 10000,
});
```

### Важное правило: публичные vs секретные переменные

Переменные с публичными префиксами (`NEXT_PUBLIC_`, `VITE_`, `NUXT_PUBLIC_`, `REACT_APP_`) встраиваются в клиентский бандл на этапе сборки. Любой пользователь может увидеть их значение в DevTools.

```bash
# ✅ Можно использовать на клиенте
NEXT_PUBLIC_API_URL=https://api.example.com

# ❌ Никогда не делайте так
NEXT_PUBLIC_STRIPE_SECRET_KEY=sk_live_abc123
```

Секретные ключи должны оставаться на сервере. Подробнее о безопасности env-переменных — в разделе [Управление секретами](../security/security-secrets-management.md).

### `.env` не должен попадать в git

```gitignore
# .gitignore
.env
.env.local
.env.*.local
```

Если `.env` случайно попал в репозиторий, считайте все значения скомпрометированными — отзовите и пересоздайте ключи.

---

## Паттерн централизованного API-клиента

Независимо от библиотеки, рекомендуется создавать единый экземпляр клиента для всего приложения.

### На ky

```ts
import ky from 'ky';

export const api = ky.create({
  prefixUrl: import.meta.env.VITE_API_URL,
  timeout: 10000,
  retry: {
    limit: 2,
    methods: ['get'],
    statusCodes: [408, 429, 500, 502, 503, 504],
  },
  hooks: {
    beforeRequest: [
      (request) => {
        const token = localStorage.getItem('accessToken');
        if (token) {
          request.headers.set('Authorization', `Bearer ${token}`);
        }
        request.headers.set('X-Request-ID', crypto.randomUUID());
      },
    ],
    afterResponse: [
      async (request, options, response) => {
        if (!response.ok) {
          const body = await response.json().catch(() => ({}));
          throw new ApiError(response.status, body.message || 'API error');
        }
        return response;
      },
    ],
  },
});

class ApiError extends Error {
  constructor(
    public status: number,
    public message: string
  ) {
    super(message);
  }
}
```

### Использование в компонентах

```ts
import { api } from '@/api/client';

async function loadUser(id: string) {
  return api.get(`users/${id}`).json<User>();
}
```

---

## Лучшие практики

### 1. Создавай один экземпляр клиента

Не импортируй `axios` или `ky` напрямую в каждом файле. Используй настроенный инстанс.

```ts
// ❌ Плохо
import axios from 'axios';
await axios.get('/api/users');

// ✅ Хорошо
import { api } from '@/api/client';
await api.get('users');
```

### 2. Централизуй обработку ошибок

Превращайте HTTP-ошибки в понятные исключения на уровне клиента, а не в каждом компоненте.

### 3. Используй timeout

Бесконечно висящий запрос хуже упавшего.

```ts
const api = ky.create({
  timeout: 10000,
});
```

### 4. Логируй запросы в dev-режиме

```ts
api.interceptors.request.use((config) => {
  if (import.meta.env.DEV) {
    console.log('→', config.method?.toUpperCase(), config.url);
  }
  return config;
});
```

### 5. Добавляй request ID

Уникальный ID на каждый запрос упрощает отладку на бэкенде.

```ts
request.headers.set('X-Request-ID', crypto.randomUUID());
```

### 6. Не храни токены в небезопасных местах

Используй `HttpOnly` cookies там, где возможно. Если используешь `localStorage`, понимай риски XSS.

---

## Антипаттерны

### 1. Дублирование baseURL

```ts
// ❌ Плохо
await fetch('https://api.example.com/users');
await fetch('https://api.example.com/posts');

// ✅ Хорошо
await api.get('users');
await api.get('posts');
```

### 2. Игнорирование HTTP-ошибок с fetch

```ts
// ❌ Плохо
const response = await fetch('/api/users');
const data = await response.json(); // Может быть 500

// ✅ Хорошо
if (!response.ok) throw new Error(...);
```

### 3. Хардкод статусов по всему проекту

```ts
// ❌ Плохо
if (error.response.status === 401) { ... }

// ✅ Хорошо
if (error.status === HttpStatus.Unauthorized) { ... }
```

### 4. Перехват всех ошибок в interceptor

```ts
// ❌ Плохо: тихо проглатываем ошибки
api.interceptors.response.use(
  (res) => res,
  () => null
);
```

### 5. Разный клиент в разных частях приложения

Использование axios в одних модулях, fetch в других и ky в третьих усложняет поддержку.

---

## Итог

- **`fetch`** — база, но требует обёртки для серьёзных проектов.
- **`axios`** — проверенный выбор с богатым API, но тяжелее.
- **`ky`** — лёгкая и современная альтернатива axios.
- **`ofetch`** — лучший выбор для Nuxt/SSR.

Для Middle+ разработчика важно понимать не только синтаксис библиотек, но и когда какую выбирать, как централизовать клиент и как обрабатывать ошибки единообразно.
