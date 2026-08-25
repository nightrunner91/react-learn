# HTTP-заголовки безопасности: полный гид для фронтендера

## Содержание

1. [Почему заголовки — это важно](#почему-заголовки--это-важно)
2. [Content-Security-Policy](#content-security-policy)
3. [Strict-Transport-Security (HSTS)](#strict-transport-security-hsts)
4. [X-Frame-Options](#x-frame-options)
5. [X-Content-Type-Options](#x-content-type-options)
6. [Referrer-Policy](#referrer-policy)
7. [Permissions-Policy](#permissions-policy)
8. [Cross-Origin-Opener-Policy (COOP)](#cross-origin-opener-policy-coop)
9. [Cross-Origin-Embedder-Policy (COEP)](#cross-origin-embedder-policy-coep)
10. [Cross-Origin-Resource-Policy (CORP)](#cross-origin-resource-policy-corp)
11. [Clear-Site-Data](#clear-site-data)
12. [Cache-Control для чувствительных страниц](#cache-control-для-чувствительных-страниц)
13. [Как настроить в Next.js и Nuxt](#как-настроить-в-nextjs-и-nuxt)
14. [Чек-лист заголовков](#чек-лист-заголовков)
15. [Терминология](#терминология)

---

## Почему заголовки — это важно

HTTP-заголовки безопасности — это инструкции, которые сервер отправляет браузеру вместе с ответом. Они не требуют изменений в бизнес-логике приложения, но эффективно закрывают целые классы атак.

Аналогия: если HTML-страница — это содержимое письма, то security-заголовки — это конверт с пометками: «Не вскрывать», «Только лично в руки», «Вернуть отправителю при нарушении». Браузер читает эти пометки и ведёт себя соответственно.

---

## Content-Security-Policy

Подробно разобран в отдельной статье [CSP: глубокое погружение](./security-csp-deep-dive.md).

Кратко: CSP говорит браузеру, откуда разрешено загружать скрипты, стили, изображения и другие ресурсы. Это вторая линия обороны от XSS.

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'
```

---

## Strict-Transport-Security (HSTS)

HSTS указывает браузеру, что сайт должен загружаться только по HTTPS. Даже если пользователь введёт `http://example.com`, браузер автоматически перенаправит на HTTPS.

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- `max-age` — время в секундах, на которое браузер запоминает политику.
- `includeSubDomains` — распространяет политику на поддомены.
- `preload` — позволяет включить сайт в preload-список браузеров.

### Предостережение

Перед включением HSTS убедитесь, что весь сайт работает по HTTPS. Иначе пользователи не смогут зайти.

---

## X-Frame-Options

Запрещает встраивание страницы в `<iframe>`. Защищает от clickjacking — атаки, при которой злоумышленник накладывает прозрачный iframe с вашим сайтом на свою страницу.

```http
X-Frame-Options: DENY
```

Возможные значения:

- `DENY` — запретить встраивание полностью.
- `SAMEORIGIN` — разрешить встраивание только с того же источника.

Современная альтернатива — директива CSP `frame-ancestors`:

```http
Content-Security-Policy: frame-ancestors 'none'
```

---

## X-Content-Type-Options

Запрещает браузеру угадывать MIME-тип файла (mime sniffing). Без этого заголовка браузер может интерпретировать, например, загруженный `.txt` как JavaScript.

```http
X-Content-Type-Options: nosniff
```

---

## Referrer-Policy

Контролирует, какой URL отправляется в заголовке `Referer` при переходе на другие сайты.

```http
Referrer-Policy: strict-origin-when-cross-origin
```

| Значение | Поведение |
|----------|-----------|
| `no-referrer` | Не отправлять Referer вообще |
| `strict-origin-when-cross-origin` | На своём сайте — полный URL, на чужом — только origin |
| `origin` | Всегда отправлять только origin |
| `unsafe-url` | Всегда отправлять полный URL (небезопасно) |

Рекомендуется `strict-origin-when-cross-origin` — баланс приватности и функциональности.

---

## Permissions-Policy

Контролирует, какие браузерные API могут использоваться на странице.

```http
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
```

Если ваше приложение не использует камеру — заблокируйте её. Это уменьшает поверхность атаки и защищает приватность пользователей.

---

## Cross-Origin-Opener-Policy (COOP)

COOP изолирует окно/вкладку от других источников. Защищает от атак через `window.opener`, таких как tabnabbing.

```http
Cross-Origin-Opener-Policy: same-origin
```

| Значение | Поведение |
|----------|-----------|
| `same-origin` | Только окна с того же origin могут взаимодействовать |
| `same-origin-allow-popups` | Разрешить взаимодействие с попапами того же origin |
| `unsafe-none` | Нет изоляции (по умолчанию) |

---

## Cross-Origin-Embedder-Policy (COEP)

COEP требует, чтобы встроенные ресурсы (iframe, изображения, скрипты) разрешали cross-origin embedding через CORP или CORS.

```http
Cross-Origin-Embedder-Policy: require-corp
```

Вместе с COOP включает так называемый **cross-origin isolated** режим, который необходим для некоторых мощных API (SharedArrayBuffer, performance.measureUserAgentSpecificMemory и др.).

---

## Cross-Origin-Resource-Policy (CORP)

CORP указывает, кто может встраивать ваши ресурсы на других сайтах.

```http
Cross-Origin-Resource-Policy: same-origin
```

| Значение | Поведение |
|----------|-----------|
| `same-origin` | Только ваш origin |
| `same-site` | Тот же site (включая поддомены) |
| `cross-origin` | Любой origin |

---

## Clear-Site-Data

Позволяет серверу приказать браузеру очистить cookie, storage, cache. Полезен при logout.

```http
Clear-Site-Data: "cookies", "storage", "cache"
```

При выходе из системы это гарантирует, что браузер удалит сессионные данные.

---

## Cache-Control для чувствительных страниц

Не security-заголовок в узком смысле, но важен для защиты данных.

```http
Cache-Control: no-store, no-cache, must-revalidate, proxy-revalidate
Pragma: no-cache
Expires: 0
```

Используйте для страниц с личными данными, чтобы браузер и прокси не кэшировали их.

---

## Как настроить в Next.js и Nuxt

### Next.js

```js
// next.config.js
const securityHeaders = [
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains' },
  { key: 'Cross-Origin-Opener-Policy', value: 'same-origin' },
];

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```

### Nuxt

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    headers: {
      xFrameOptions: 'DENY',
      xContentTypeOptions: 'nosniff',
      referrerPolicy: 'strict-origin-when-cross-origin',
      permissionsPolicy: 'camera=(), microphone=(), geolocation=()',
      strictTransportSecurity: {
        maxAge: 31536000,
        includeSubdomains: true,
      },
      crossOriginOpenerPolicy: 'same-origin',
    },
  },
});
```

---

## Чек-лист заголовков

- [ ] `Content-Security-Policy` настроена и протестирована.
- [ ] `Strict-Transport-Security` включён для HTTPS-only сайтов.
- [ ] `X-Frame-Options` или `frame-ancestors 'none'` защищают от clickjacking.
- [ ] `X-Content-Type-Options: nosniff` установлен.
- [ ] `Referrer-Policy` ограничивает утечку URL.
- [ ] `Permissions-Policy` блокирует ненужные API.
- [ ] `Cross-Origin-Opener-Policy` включён.
- [ ] Для чувствительных страниц `Cache-Control: no-store`.
- [ ] При logout используется `Clear-Site-Data`.

---

## Терминология

| Термин | Значение |
|--------|----------|
| **HSTS** | HTTP Strict Transport Security — принудительный HTTPS |
| **CSP** | Content Security Policy |
| **COOP** | Cross-Origin-Opener-Policy — изоляция окон |
| **COEP** | Cross-Origin-Embedder-Policy — требования к встраиваемым ресурсам |
| **CORP** | Cross-Origin-Resource-Policy — политика встраивания ресурсов |
| **Clickjacking** | Атака через прозрачный iframe |
| **Tabnabbing** | Атака через window.opener открытой вкладки |
| **Mime sniffing** | Определение MIME-типа браузером по содержимому |
| **Cross-origin isolated** | Режим с COOP + COEP для мощных API |

---

## Что важно понимать

Security-заголовки — это дешёвый и эффективный способ защиты. Они не заменяют безопасный код, но добавляют несколько независимых слоёв защиты. Для SOC2 и CASA наличие и документирование этих заголовков является стандартным evidence.

Фронтенд-разработчик должен знать, какой заголовок за что отвечает, и уметь настраивать их в своём фреймворке.
