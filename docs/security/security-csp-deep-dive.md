# Content Security Policy: настройка, эволюция и практика

## Содержание

1. [Что такое CSP и зачем она нужна](#что-такое-csp-и-зачем-она-нужна)
2. [Как CSP работает под капотом](#как-csp-работает-под-капотом)
3. [Основные директивы CSP](#основные-директивы-csp)
4. [Inline-скрипты и стили: проблема и решения](#inline-скрипты-и-стили-проблема-и-решения)
5. [Nonce-based CSP](#nonce-based-csp)
6. [Hash-based CSP](#hash-based-csp)
7. [strict-dynamic](#strict-dynamic)
8. [CSP в Next.js](#csp-в-nextjs)
9. [CSP в Nuxt](#csp-в-nuxt)
10. [Report-Only режим и мониторинг](#report-only-режим-и-мониторинг)
11. [Типичные ошибки при внедрении](#типичные-ошибки-при-внедрении)
12. [Чек-лист внедрения CSP](#чек-лист-внедрения-csp)
13. [Терминология CSP](#терминология-csp)

---

## Что такое CSP и зачем она нужна

Content Security Policy (CSP) — это HTTP-заголовок, который сообщает браузеру, откуда разрешено загружать различные ресурсы: скрипты, стили, изображения, шрифты, iframe, соединения.

Если XSS-уязвимость всё же есть, CSP может предотвратить выполнение вредоносного кода, потому что браузер откажется загружать скрипты с неразрешённых источников.

Аналогия: представьте ресторан, где повара готовят только из продуктов из проверенного списка поставщиков. Если кто-то подбросит мешок с продуктами с улицы, повар его просто не примет — он не из списка.

Без CSP браузер загружает любой ресурс, который встречает в HTML. С CSP он сначала проверяет «список поставщиков».

---

## Как CSP работает под капотом

Когда браузер получает HTML, он начинает парсить страницу и загружать ресурсы. Увидев CSP-заголовок, браузер сравнивает каждый ресурс с политикой:

- `<script src="https://evil.com/xss.js">` — блокируется, если `evil.com` не в `script-src`.
- `<img src="https://tracker.com/pixel.gif">` — блокируется, если `tracker.com` не в `img-src`.
- Inline-обработчик `onclick="alert(1)"` — блокируется, если нет `'unsafe-inline'`.

Если ресурс не проходит проверку, браузер:

- блокирует его загрузку или выполнение;
- пишет ошибку в консоль;
- (опционально) отправляет отчёт на указанный URL.

---

## Основные директивы CSP

| Директива | Что контролирует |
|-----------|------------------|
| `default-src` | Значение по умолчанию для всех директив, которые не указаны явно |
| `script-src` | Источники JavaScript |
| `style-src` | Источники CSS |
| `img-src` | Источники изображений |
| `font-src` | Источники шрифтов |
| `connect-src` | URL для fetch, XHR, WebSocket, EventSource |
| `media-src` | Аудио и видео |
| `object-src` | Flash, Java-апплеты, PDF-объекты (рекомендуется `'none'`) |
| `frame-src` | URL для iframe и frame |
| `frame-ancestors` | Кто может встраивать ваш сайт в iframe (защита от clickjacking) |
| `base-uri` | URL для `<base>` тега |
| `form-action` | URL, куда можно отправлять формы |
| `upgrade-insecure-requests` | Принудительно использовать HTTPS для всех ресурсов |
| `block-all-mixed-content` | Блокировать смешанный HTTP-контент |

### Пример базовой политики

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
```

### Специальные значения

| Значение | Значение |
|----------|----------|
| `'self'` | Текущий источник (scheme + host + port) |
| `'none'` | Ничего не разрешено |
| `'unsafe-inline'` | Разрешить inline-скрипты/стили |
| `'unsafe-eval'` | Разрешить eval() и new Function() |
| `'unsafe-hashes'` | Разрешить конкретные inline-обработчики по хэшу |
| `'nonce-...'` | Разрешить элементы с указанным nonce |
| `'sha256-...'` | Разрешить элементы с указанным хэшем |
| `'strict-dynamic'` | Доверять скриптам, загруженным доверенным корневым скриптом |

---

## Inline-скрипты и стили: проблема и решения

Inline-скрипты — главная головная боль CSP. Если разрешить `'unsafe-inline'`, политика существенно ослабевает, потому что XSS часто использует именно inline-обработчики.

Но многие фреймворки и библиотеки генерируют inline-скрипты:

- inline-стили CSS-in-JS;
- скрипты аналитики;
- runtime-скрипты Next.js и Nuxt;
- Google Tag Manager.

Решения:

1. **Nonce** — случайный токен для каждого запроса.
2. **Hash** — хэш конкретного inline-блока.
3. **Вынести скрипты во внешние файлы** — если возможно.

---

## Nonce-based CSP

Nonce (number used once) — это случайная строка, генерируемая сервером для каждого запроса. Разрешённые inline-скрипты и стили получают атрибут `nonce="..."`.

```http
Content-Security-Policy: script-src 'self' 'nonce-abc123';
```

```html
<script nonce="abc123">
  console.log('этот скрипт выполнится');
</script>

<script>
  console.log('этот скрипт заблокирован');
</script>
```

### Важно

- Nonce должен быть криптографически случайным и длинным (минимум 128 бит).
- Nonce должен меняться при каждом запросе.
- Nonce нельзя предсказать — иначе злоумышленник сможет его использовать.

### Пример в Next.js middleware

```ts
// middleware.ts
import { NextResponse } from 'next/server';
import { randomBytes } from 'crypto';

export function middleware(request: Request) {
  const nonce = randomBytes(16).toString('base64');
  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' data: https:;
    connect-src 'self';
    frame-ancestors 'none';
  `;

  const response = NextResponse.next();
  response.headers.set('Content-Security-Policy', csp.replace(/\s{2,}/g, ' ').trim());
  response.headers.set('x-nonce', nonce);
  return response;
}
```

А затем в компоненте получить nonce через `headers()` и передать в скрипты.

---

## Hash-based CSP

Вместо nonce можно разрешить конкретный inline-скрипт по его SHA-хэшу.

```http
Content-Security-Policy: script-src 'self' 'sha256-abc123...';
```

```html
<script>console.log('hello');</script>
```

Браузер вычисляет хэш содержимого скрипта и сравнивает с политикой.

Минусы:

- Любое изменение скрипта требует обновления хэша.
- Не подходит для динамического контента.
- Не работает для inline-обработчиков `onclick` (только через `'unsafe-hashes'`).

Плюсы:

- Не требует серверного состояния.
- Хорошо подходит для статических inline-скриптов.

---

## strict-dynamic

`'strict-dynamic'` меняет логику `script-src`: если разрешён корневой скрипт (по nonce или хэшу), все скрипты, которые он загружает динамически, тоже считаются доверенными.

```http
Content-Security-Policy: script-src 'nonce-abc123' 'strict-dynamic';
```

```html
<script nonce="abc123" src="https://cdn.example.com/app.js"></script>
```

Если `app.js` динамически создаёт `<script src="https://cdn.example.com/chunk.js">`, браузер разрешит его загрузку, даже если `cdn.example.com` не указан явно.

Это полезно для бандлеров и динамических импортов, но требует, чтобы корневой скрипт был защищён nonce.

---

## CSP в Next.js

Next.js позволяет настраивать CSP через `next.config.js` или middleware.

### Через next.config.js

```js
// next.config.js
const csp = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
`;

module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: csp.replace(/\s{2,}/g, ' ').trim(),
          },
        ],
      },
    ];
  },
};
```

### Через middleware с nonce

Более безопасный вариант — middleware, который генерирует nonce для каждого запроса. Next.js 15+ имеет встроенную поддержку nonce в документации.

### Особенности App Router

- Server Components рендерятся на сервере, поэтому nonce можно встраивать прямо в runtime-скрипты.
- Client Components получают nonce через props или context.
- Встроенные inline-скрипты Next.js (например, для предзагрузки) требуют `'unsafe-inline'` или nonce.

---

## CSP в Nuxt

Nuxt предлагает модуль `nuxt-security`, который значительно упрощает настройку CSP.

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
        'connect-src': ["'self'", 'https://api.example.com'],
        'frame-ancestors': ["'none'"],
      },
    },
  },
});
```

Модуль автоматически генерирует nonce и встраивает его в скрипты и стили.

---

## Report-Only режим и мониторинг

CSP может сломать приложение, если политика слишком строгая. Чтобы избежать этого, используйте `Content-Security-Policy-Report-Only`:

```http
Content-Security-Policy-Report-Only:
  default-src 'self';
  script-src 'self';
  report-uri /api/csp-report;
```

Браузер будет:

- загружать ресурсы как обычно;
- отправлять отчёты о нарушениях на `/api/csp-report`;
- не блокировать ресурсы.

Это позволяет собрать статистику, увидеть, какие ресурсы реально используются, и только потем переключиться на enforcing-режим.

### Формат отчёта

```json
{
  "csp-report": {
    "document-uri": "https://example.com/page",
    "violated-directive": "script-src",
    "blocked-uri": "https://evil.com/script.js",
    "original-policy": "default-src 'self'; script-src 'self';"
  }
}
```

---

## Типичные ошибки при внедрении

### 1. `'unsafe-inline'` в `script-src`

Разрешает любые inline-скрипты, что сводит на нет защиту от XSS.

### 2. Слишком широкие домены

```http
script-src 'self' https:;
```

Разрешение любого HTTPS — почти то же самое, что отсутствие CSP.

### 3. `*` в `connect-src`

```http
connect-src *;
```

Позволяет любые исходящие соединения, включая эксфильтрацию данных.

### 4. Игнорирование `frame-ancestors`

Без `frame-ancestors 'none'` сайт можно встроить в iframe и использовать для clickjacking.

### 5. Неправильный nonce

Использование предсказуемого nonce, например, основанного на timestamp, позволяет злоумышленнику его угадать.

---

## Чек-лист внедрения CSP

### Подготовка

- [ ] Составить инвентарь всех источников ресурсов (скрипты, стили, шрифты, API, изображения).
- [ ] Определить, какие inline-скрипты и стили нужны.
- [ ] Решить: nonce, hash или вынести во внешние файлы.

### Внедрение

- [ ] Начать с `Content-Security-Policy-Report-Only`.
- [ ] Настроить endpoint для сбора отчётов.
- [ ] Собрать отчёты в течение 1–2 недель.
- [ ] Исправить нарушения: добавить недостающие источники или изменить код.
- [ ] Переключиться на `Content-Security-Policy`.

### Поддержка

- [ ] Регулярно проверять логи нарушений.
- [ ] Обновлять политику при добавлении новых интеграций.
- [ ] Проверять CSP на staging и production.

---

## Терминология CSP

| Термин | Значение |
|--------|----------|
| **CSP** | Content Security Policy — политика загрузки ресурсов |
| **Directive** | Правило CSP, например `script-src` |
| **Source expression** | Значение директивы, например `'self'` или `https://cdn.com` |
| **Nonce** | Одноразовый случайный токен для разрешения inline-ресурсов |
| **Hash** | SHA-хэш конкретного inline-ресурса |
| **strict-dynamic** | Директива, доверяющая скриптам, загруженным доверенным корнем |
| **Report-Only** | Режим, в котором CSP только отчитывается, но не блокирует |
| **Violation report** | Отчёт браузера о нарушении CSP |
| **frame-ancestors** | Директива, контролирующая встраивание в iframe |
| **Mixed content** | Смешанный HTTP/HTTPS контент на HTTPS-странице |

---

## Что важно понимать

CSP — это не замена экранированию и санитизации, а вторая линия обороны. Даже если в приложении есть XSS, правильно настроенная CSP может предотвратить выполнение вредоносного кода.

Идеальная CSP — это политика без `'unsafe-inline'` и `'unsafe-eval'` с явным перечислением источников. На практике к этому приходят постепенно, через Report-Only режим и nonce-based подход.

Для SOC2 и CASA наличие CSP, настроенной и мониторящей нарушения, является важным evidence защиты клиентских данных.
