# Безопасность Vue и Nuxt: специфика фреймворка

## Содержание

1. [Как Vue защищает от XSS по умолчанию](#как-vue-защищает-от-xss-по-умолчанию)
2. [v-html: двустороннее лезвие](#v-html-двустороннее-лезвие)
3. [Динамические атрибуты и URL](#динамические-атрибуты-и-url)
4. [Refs и императивный DOM](#refs-и-императивный-dom)
5. [Composition API и безопасность](#composition-api-и-безопасность)
6. [Nuxt 3: SSR и безопасность](#nuxt-3-ssr-и-безопасность)
7. [Nuxt Security Module](#nuxt-security-module)
8. [Серверные маршруты Nitro](#серверные-маршруты-nitro)
9. [runtimeConfig и секреты](#runtimeconfig-и-секреты)
10. [CSRF в Nuxt](#csrf-в-nuxt)
11. [CSP в Nuxt](#csp-в-nuxt)
12. [Чек-лист для Vue/Nuxt](#чек-лист-для-vuenuxt)
13. [Терминология](#терминология)

---

## Как Vue защищает от XSS по умолчанию

Vue, как и React, автоматически экранирует текстовые интерполяции `{{ }}`. Это означает, что любые HTML-теги в строке будут отображены как текст, а не выполнены.

```vue
<template>
  <div>{{ userInput }}</div>
</template>

<script setup>
const userInput = "<script>alert('xss')<\/script>";
</script>
```

Результат в DOM:

```html
<div>&lt;script&gt;alert('xss')&lt;/script&gt;</div>
```

Vue использует `textContent` для вставки текста, что безопасно по умолчанию.

### Атрибуты

Динамические атрибуты тоже экранируются:

```vue
<template>
  <div :title="userTitle">hover me</div>
</template>
```

Но Vue **не проверяет семантику URL**. Если `userTitle` содержит `javascript:alert(1)`, Vue экранирует кавычки, но `javascript:` протокол останется валидным для браузера при использовании в `href`.

---

## v-html: двустороннее лезвие

`v-html` — это прямой аналог `dangerouslySetInnerHTML` в React. Он вставляет HTML как есть, без экранирования.

```vue
<template>
  <!-- ❌ Опасно без санитизации -->
  <div v-html="userHtml"></div>
</template>
```

### Правильное использование

```vue
<script setup>
import { computed } from 'vue';
import DOMPurify from 'dompurify';

const props = defineProps(['rawHtml']);

const safeHtml = computed(() =>
  DOMPurify.sanitize(props.rawHtml, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p'],
    ALLOWED_ATTR: ['href'],
  })
);
</script>

<template>
  <div v-html="safeHtml"></div>
</template>
```

### Предупреждение в документации

Vue официально предупреждает: `v-html` можно использовать только для доверенного контента. Никогда не используйте его для пользовательского ввода без санитизации.

---

## Динамические атрибуты и URL

Vue не валидирует протоколы URL. Это распространённый вектор:

```vue
<template>
  <!-- ❌ Опасно -->
  <a :href="userUrl">link</a>
</template>
```

Если `userUrl = "javascript:alert(document.cookie)"`, при клике выполнится JavaScript.

### Безопасная ссылка

```vue
<script setup>
const props = defineProps(['href']);

const isSafe = computed(() => {
  return props.href.startsWith('/') ||
         /^https?:\/\//.test(props.href);
});
</script>

<template>
  <a v-if="isSafe" :href="href" rel="noopener noreferrer" target="_blank">
    <slot />
  </a>
  <span v-else>
    <slot />
  </span>
</template>
```

### target="_blank" и rel="noopener"

Всегда используйте `rel="noopener noreferrer"` для внешних ссылок. Это защищает от tabnabbing — атаки, при которой открытая вкладка может изменить `window.opener.location` родительской страницы.

---

## Refs и императивный DOM

Когда вы получаете прямой доступ к DOM-элементу через `ref`, вы обходите защиту Vue.

```vue
<script setup>
import { ref, onMounted } from 'vue';

const el = ref(null);

onMounted(() => {
  // ❌ XSS, если content содержит вредоносный HTML
  el.value.innerHTML = props.content;
});
</script>

<template>
  <div ref="el"></div>
</template>
```

Безопасная альтернатива:

```vue
<script setup>
import { ref, onMounted } from 'vue';
import DOMPurify from 'dompurify';

const el = ref(null);

onMounted(() => {
  el.value.innerHTML = DOMPurify.sanitize(props.content);
});
</script>
```

Лучше всего избегать прямой манипуляции `innerHTML` вообще.

---

## Composition API и безопасность

Composition API не меняет принципов безопасности, но делает некоторые паттерны удобнее.

### Хранение токенов

```ts
// composables/useAuth.ts
import { ref } from 'vue';

const accessToken = ref<string | null>(null);

export function useAuth() {
  const setToken = (token: string) => {
    accessToken.value = token;
  };

  return { accessToken, setToken };
}
```

Токен в памяти защищён от XSS лучше, чем в `localStorage`, но теряется при обновлении страницы.

### Fetch с авторизацией

```ts
export async function fetchWithAuth(url: string, options: RequestInit = {}) {
  const { accessToken } = useAuth();

  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: accessToken.value ? `Bearer ${accessToken.value}` : '',
    },
  });
}
```

---

## Nuxt 3: SSR и безопасность

Nuxt рендерит приложение на сервере. Это добавляет удобства, но и новые риски.

### Universal rendering

В Nuxt 3 компоненты по умолчанию рендерятся на сервере, затем гидратируются на клиенте. Vue-интерполяции экранируются и на сервере, и на клиенте. Но `v-html` в SSR-контексте особенно опасен: вредоносный код попадает в исходный HTML и выполняется мгновенно, до гидратации.

### Правильная обработка пользовательского ввода в SSR

```vue
<script setup>
import DOMPurify from 'dompurify';

const { data: article } = await useFetch('/api/article/1');

const safeContent = computed(() =>
  DOMPurify.sanitize(article.value?.content || '')
);
</script>

<template>
  <article v-html="safeContent"></article>
</template>
```

### Опасность useFetch с пользовательским URL

```vue
<script setup>
// ❌ Опасно: URL из query-параметра без валидации
const { data } = await useFetch(() => `/api/${useRoute().params.slug}`);
</script>
```

Параметры маршрута должны валидироваться на сервере.

---

## Nuxt Security Module

`nuxt-security` — официальный модуль безопасности для Nuxt. Он помогает настроить:

- CSP;
- security-заголовки;
- CORS;
- rate limiting;
- защиту от XSS;
- subresource integrity (SRI);
- и многое другое.

### Базовая настройка

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    headers: {
      contentSecurityPolicy: {
        'default-src': ["'self'"],
        'script-src': ["'self'", "'nonce-{{nonce}}'", "'strict-dynamic'"],
        'style-src': ["'self'", "'nonce-{{nonce}}'"],
        'img-src': ["'self'", 'data:', 'https:'],
        'connect-src': ["'self'"],
        'frame-ancestors': ["'none'"],
      },
      crossOriginEmbedderPolicy: 'require-corp',
      xFrameOptions: 'DENY',
    },
    rateLimiter: {
      tokensPerInterval: 100,
      interval: 'hour',
    },
  },
});
```

Модуль автоматически генерирует nonce для каждого запроса и встраивает его в скрипты.

---

## Серверные маршруты Nitro

Nuxt использует Nitro для серверных маршрутов. Это аналог Route Handlers в Next.js.

### Пример с валидацией

```ts
// server/api/posts.post.ts
import { z } from 'zod';

const schema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1).max(10000),
});

export default defineEventHandler(async (event) => {
  const session = await getUserSession(event);

  if (!session) {
    throw createError({ statusCode: 401, statusMessage: 'Unauthorized' });
  }

  const body = await readBody(event);
  const result = schema.safeParse(body);

  if (!result.success) {
    throw createError({ statusCode: 400, statusMessage: 'Invalid input' });
  }

  await createPost(session.user.id, result.data);

  return { ok: true };
});
```

### Проверка прав

```ts
// server/api/posts/[id].delete.ts
export default defineEventHandler(async (event) => {
  const session = await getUserSession(event);
  const id = getRouterParam(event, 'id');

  const post = await getPost(id);

  if (post.authorId !== session.user.id) {
    throw createError({ statusCode: 403, statusMessage: 'Forbidden' });
  }

  await deletePost(id);
  return { ok: true };
});
```

---

## runtimeConfig и секреты

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: process.env.API_SECRET,
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE,
    },
  },
});
```

```vue
<script setup>
const config = useRuntimeConfig();

// Только на сервере
console.log(config.apiSecret);

// Доступно на клиенте и сервере
console.log(config.public.apiBase);
</script>
```

**Правило:** секреты всегда в `runtimeConfig` без `public`, публичные значения — в `runtimeConfig.public`.

---

## CSRF в Nuxt

Nuxt не имеет встроенной CSRF-защиты из коробки, но её легко добавить.

### SameSite cookies

```ts
// server/utils/session.ts
export async function setUserSession(event, user) {
  await setCookie(event, 'session', JSON.stringify(user), {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7,
  });
}
```

### CSRF-токены

Можно использовать `h3-cors` или встроенные middleware для проверки Origin и CSRF-токенов.

```ts
// server/middleware/csrf.ts
export default defineEventHandler((event) => {
  if (event.method === 'POST') {
    const origin = getHeader(event, 'origin');
    const allowed = ['https://mysite.com'];

    if (!origin || !allowed.includes(origin)) {
      throw createError({ statusCode: 403 });
    }
  }
});
```

---

## CSP в Nuxt

Через `nuxt-security`:

```ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    headers: {
      contentSecurityPolicy: {
        'default-src': ["'self'"],
        'script-src': ["'self'", "'nonce-{{nonce}}'"],
        'style-src': ["'self'", "'nonce-{{nonce}}'"],
      },
    },
  },
});
```

Без модуля — через Nuxt middleware:

```ts
// server/middleware/csp.ts
export default defineEventHandler((event) => {
  appendResponseHeader(
    event,
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self'; style-src 'self';"
  );
});
```

---

## Чек-лист для Vue/Nuxt

### Vue

- [ ] Использовать `{{ }}` вместо `v-html` для пользовательского текста.
- [ ] Санитизировать HTML перед использованием в `v-html`.
- [ ] Валидировать URL в `:href` и `:src`.
- [ ] Использовать `rel="noopener noreferrer"` для внешних ссылок.
- [ ] Избегать `innerHTML` через refs.

### Nuxt

- [ ] Установить `nuxt-security` и настроить CSP.
- [ ] Использовать `runtimeConfig` для секретов.
- [ ] Валидировать входные данные в server routes.
- [ ] Проверять авторизацию в server routes и middleware.
- [ ] Настроить HttpOnly, Secure, SameSite cookies.
- [ ] Передавать клиенту только необходимые данные.

---

## Терминология

| Термин | Значение |
|--------|----------|
| **v-html** | Директива Vue для вставки сырого HTML |
| **ref** | Ссылка на DOM-элемент в Vue |
| **Composition API** | API Vue для организации логики компонентов |
| **Nitro** | Серверный движок Nuxt |
| **runtimeConfig** | Конфигурация Nuxt для серверных и публичных значений |
| **nuxt-security** | Модуль безопасности для Nuxt |
| **Tabnabbing** | Атака через window.opener у открытой вкладки |
| **Universal rendering** | SSR + CSR в одном приложении |

---

## Что важно понимать

Vue и Nuxt предоставляют хорошую базовую защиту от XSS через автоматическое экранирование, но у них есть мощные инструменты (`v-html`, refs, server routes), которые требуют осознанного подхода.

Для SOC2 и CASA важно показать, что команда:

- понимает различие между клиентом и сервером в Nuxt;
- использует `runtimeConfig` для секретов;
- валидирует данные в Nitro;
- настраивает CSP и security-заголовки через `nuxt-security`.
