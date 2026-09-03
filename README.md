# Подготовка к интервью на Frontend-разработчика 2026 (React/Next.js)

Этот репозиторий — не набор ответов для зубрёжки, а структурированный путь от фундамента к продвинутым темам. Каждый раздел содержит вопросы, на которые ты должен уметь уверенно ответить. Рядом — ссылки на подробные статьи в `/docs`, где тема раскрыта глубоко и с примерами.

## Как пользоваться

1. Иди по разделам сверху вниз — они расположены от базы к продвинутому.
2. Открой статью, прочитай, разберись с примерами.
3. Пройдись по чеклисту вопросов — если на каждый можешь дать чёткий ответ без подглядывания, иди дальше.
4. Если застрял — вернись к статье, перечитай проблемные места.
5. Не перескакивай через разделы: каждая тема опирается на предыдущие.

## Методология обучения

Чтобы извлечь из максимум из этого репозитория, используй активные техники обучения, а не пассивное чтение.

**Active Recall**

Закрой статью и ответь на вопрос чеклиста вслух. Если не можешь — вернись к материалу, найди пробел и повтори.

**Feynman Technique**

Объясни тему простыми словами так, будто собеседник — джун. Если где-то запинаешься, значит, тема ещё не усвоена.

**Spaced Repetition**

Повторяй сложные темы через 1, 3 и 7 дней. Не полагайся на однократное прочтение.

**Мок-интервью**

Отвечай на вопросы с таймером, записывай себя или проси друга/коллегу провести собеседование. Теория закрепляется только через устную практику.

**Практика**

Решай 1–2 задачи на LeetCode/CodeWars в день (easy/medium) и веди pet-проект на Next.js + TypeScript.

**Трекер прогресса**

После каждого сеанса обучения отметь каждый пройденный пункт чеклиста (здесь либо в любом удобном месте, хоть на бумаге):

🟢 Знаю уверенно
🟡 Знаю частично
🔴 Не знаю / плохо понимаю

Цель — довести все пункты до 🟢, не переходя к следующему разделу, пока в текущем нет минимум 80% зелёных.

**Рекомендуемый распорядок**

- 30-45 мин — теория: 3–5 новых вопросов из чеклиста + сверка со статьёй.
- 10-15 мин — быстрое (блиц) повторение недавних тем (вчерашних и недельных).
- 1 раз в неделю — мок-интервью: 30 минут устных ответов.

---

## 1. JavaScript Core

Фундамент всего. Без глубокого понимания JavaScript любые вопросы про React, замыкания, `this` или асинхронность превращаются в угадывание. Это спрашивают на каждом собеседовании — от Junior до Senior.

- [ ] Как JavaScript хранит данные: примитивы по значению, объекты по ссылке — и почему `arr1 === arr2` ведёт себя неожиданно
- [ ] Как работает неявное преобразование типов (type coercion) и почему `[] + {}` даёт неожиданный результат
- [ ] Что такое область видимости (scope) и как работает hoisting
- [ ] Чем отличаются `var`, `let` и `const` на уровне движка (Temporal Dead Zone, hoisting)
- [ ] Что такое замыкание (closure) и почему оно критически важно для хуков React
- [ ] Как работает `this` — чем отличается от лексического scope, и как `bind`/`call`/`apply` меняют контекст
- [ ] Что такое прототипное наследование и цепочка прототипов (`__proto__`, `prototype`)
- [ ] Как устроен Event Loop: call stack, task queue, microtask queue — и в каком порядке они выполняются
- [ ] Что такое Promise и какие у него состояния
- [ ] В чём разница между `Promise.all`, `Promise.race`, `Promise.allSettled` и `Promise.any`
- [ ] Как работает `async/await` под капотом
- [ ] Как обработать ошибку в асинхронном коде
- [ ] Как отменить асинхронную операцию
- [ ] Чем ES Modules отличаются от CommonJS
- [ ] Как работают `import`/`export` и `require`/`module.exports`
- [ ] Что такое циклические зависимости и как с ними бороться
- [ ] Что такое tree shaking и side effects
- [ ] Чем `Map` отличается от обычного объекта и когда его использовать
- [ ] Зачем нужны `WeakMap` и `WeakSet`
- [ ] Как работают итераторы и генераторы
- [ ] Что такое Proxy и для чего его применяют
- [ ] Зачем нужен Reflect
- [ ] Какие ограничения у Proxy
- [ ] Что такое чистая функция и иммутабельность
- [ ] Как работают `map`, `filter`, `reduce`, `find`, `some`, `every`
- [ ] Чем `debounce` отличается от `throttle`
- [ ] Как JavaScript управляет памятью: стек, куча, сборка мусора
- [ ] Какие типичные утечки памяти встречаются во фронтенде
- [ ] Зачем нужны `WeakMap`/`WeakSet` и как работают `WeakRef`/`FinalizationRegistry`

[Типы данных и приведение типов](./docs/javascript/js-data-types.md) · [Область видимости и hoisting](./docs/javascript/js-scope-hoisting.md) · [Замыкания](./docs/javascript/js-closures.md) · [Контекст выполнения и `this`](./docs/javascript/js-this.md) · [Прототипы и классы](./docs/javascript/js-prototypes-classes.md) · [Event Loop](./docs/javascript/js-event-loop.md) · [Асинхронность: Promise, async/await](./docs/javascript/js-async.md) · [Модули](./docs/javascript/js-modules.md) · [Коллекции, итераторы, генераторы](./docs/javascript/js-collections-iterators.md) · [Proxy и Reflect](./docs/javascript/js-proxy-reflect.md) · [Функциональное программирование](./docs/javascript/js-functional.md) · [Память и производительность](./docs/javascript/js-memory.md)

---

## 2. Алгоритмическая сложность и Big O

На собеседованиях в 2026 году всё чаще спрашивают не только React, но и алгоритмы. Тебе не нужно решать олимпиадные задачи — но понимать, почему один подход быстрее другого при росте данных, обязательно.

- [ ] Что такое Big O и почему описывает худший случай
- [ ] Чем `O(1)` отличается от `O(n)`, `O(log n)` и `O(n²)` — и как это выглядит в коде
- [ ] Почему поиск в `Map`/`Set` — `O(1)`, а в массиве — `O(n)`
- [ ] Что такое бинарный поиск и почему его сложность `O(log n)`
- [ ] Как оценить сложность вложенных циклов и рекурсивных функций
- [ ] Какие структуры данных чаще встречаются во фронтенде (Map, Set, очередь, стек) и где они применяются

[Алгоритмическая сложность и Big O](./docs/algorithms/big-o-notation.md)

---

## 3. TypeScript

TypeScript стал стандартом в React-разработке. На собеседовании спрашивают не «знаешь ли ты TS», а насколько глубоко ты понимаешь дженерики, utility-типы и умеешь типизировать хуки и компоненты.

- [ ] Чем `interface` отличается от `type` — и когда что использовать
- [ ] Что такое дженерики (generics) и как писать типобезопасные функции и компоненты
- [ ] Как работают constraints (`extends`) и default types в дженериках
- [ ] Что такое utility-типы (`Partial`, `Required`, `Pick`, `Omit`, `Record`, `ReturnType`, `Parameters`) — и как они реализованы под капотом
- [ ] Что такое conditional types и как работает `infer`
- [ ] Что такое type guards и как сужать типы (`is`, `in`, discriminated unions)
- [ ] Как типизировать пропсы, события и хуки в React-компонентах
- [ ] Как типизировать кастомные хуки с дженериками и async-паттерны (AbortController, fetch-машина)
- [ ] Чем enum отличается от union types — и почему `as const` часто лучше enum
- [ ] Что такое декораторы и где они применяются (NestJS, MobX)
- [ ] Как работает mapped types и как создать свой utility-тип

[TypeScript Generics](./docs/typescript/typescript-generics.md) · [Utility Types](./docs/typescript/typescript-utility-types.md) · [TypeGuard](./docs/typescript/typescript-typeguard.md) · [Хуки и async-паттерны](./docs/typescript/typescript-hooks-async.md) · [`infer`](./docs/typescript/typescript-infer.md) · [Enums](./docs/typescript/typescript-enums.md) · [Decorators](./docs/typescript/typescript-decorators.md) · [TypeScript в React](./docs/typescript/typescript-react.md) · [Return Types](./docs/typescript/typescript-function-return-types.md)

---

## 4. Основы React

Ядро библиотеки. Здесь всё: от JSX и компонентов до хуков и жизненного цикла. Это спрашивают всегда — без этого нет смысла идти дальше.

- [ ] Что такое React и чем он отличается от Vue и Angular (библиотека vs фреймворк)
- [ ] Что такое JSX и во что он компилируется (`React.createElement` / `jsx-runtime`)
- [ ] Чем функциональные компоненты отличаются от классовых — и почему классы считаются устаревшими
- [ ] Что такое пропсы и состояние, и в чём между ними разница
- [ ] Как работает условный рендеринг (тернарный оператор, `&&`, nullish)
- [ ] Что такое фрагменты (`<>...</>`) и зачем нужен проп `key` у `<React.Fragment>`
- [ ] Что такое ключи (keys) в списках и почему нельзя использовать индекс массива
- [ ] Чем `useEffect` отличается от `useLayoutEffect` — и когда что применять
- [ ] Как работает `useState` — и почему обновление объекта требует spread (`{ ...prev, field: value }`)
- [ ] Что такое `useReducer` и когда его предпочесть перед `useState`
- [ ] Что такое `useRef` — для доступа к DOM и хранения значений без ререндера
- [ ] Что такое пользовательские хуки (custom hooks) и как они помогают переиспользовать логику
- [ ] Что такое управляемые (controlled) и неуправляемые (uncontrolled) компоненты
- [ ] Что такое prop drilling и как его уменьшить (Context, Zustand, композиция)
- [ ] Как работает система синтетических событий (SyntheticEvent) и делегирование событий в React
- [ ] Что такое `defaultProps` и `propTypes` — и почему TypeScript их заменил

[Хуки React](./docs/react/react_hooks.md) · [useEffect](./docs/react/react_useeffect.md) · [useContext](./docs/react/react_usecontext.md) · [use()](./docs/react/react_use.md) · [useImperativeHandle](./docs/react/react_use_imperative_handle.md) · [defaultProps/propTypes](./docs/react/react_defaultprops_proptypes.md) · [Error Boundary](./docs/react/react_errorboundary.md) · [Синтетические события](./docs/react/react_synthetic_events.md) · [Порядок рендеринга](./docs/react/react_rendering_order.md)

---

## 5. Внутреннее устройство React

Понимание того, что происходит «под капотом» — то, что отличает Middle от Junior. Fiber, согласование, конкурентный режим, мемоизация — всё это объясняет, *почему* React работает именно так. Современные API React 19+ вроде Actions, `useActionState`, `useOptimistic`, `use()` и ref как проп дополняют эту картину, поэтому они рассматриваются здесь же.

- [ ] Что такое Virtual DOM и как работает алгоритм согласования (reconciliation)
- [ ] Что такое Fiber и какую проблему он решил (прерываемый рендеринг вместо синхронного)
- [ ] Из каких трёх деревьев состоит Fiber (current, workInProgress, committed) и как они переключаются
- [ ] Что такое две фазы рендеринга: Render phase (чистая, прерываемая) и Commit phase (синхронная)
- [ ] В каком порядке React рендерит компоненты (сверху вниз) и вызывает эффекты (снизу вверх)
- [ ] Что такое конкурентный React — и какие приоритеты обновлений существуют (Sync, InputContinuous, Default, Transition, Idle)
- [ ] Как работают `useTransition` и `useDeferredValue` — и чем они отличаются
- [ ] Что такое автоматическое батчирование в React 18+ и как его отключить (`flushSync`)
- [ ] Что такое `useMemo`, `useCallback` и `React.memo` — и когда мемоизация НЕ нужна
- [ ] Что такое React Compiler и как он автоматизирует мемоизацию
- [ ] Что такое `<Suspense>` и как он управляет состоянием загрузки декларативно
- [ ] Как хук `use()` разворачивает Promise и читает Context прямо в рендере
- [ ] Что такое HOC (компоненты высшего порядка) и заменили ли их хуки
- [ ] Что такое Error Boundary — что он ловит, а что нет, и почему хуки не могут его заменить
- [ ] Что такое Actions (Действия) и как работает `<form action={...}>`
- [ ] Как работает `useActionState` — управление состоянием, pending и ошибкой действия
- [ ] Что такое `useOptimistic` и как делать оптимистичные обновления UI
- [ ] Чем `use()` отличается от `useContext()` и `useEffect` + `useState`
- [ ] Что такое Owner Stack (`captureOwnerStack`) и как он работает
- [ ] Что такое `<Activity />`, `useEffectEvent`, `cacheSignal`, Performance Tracks
- [ ] Что такое Partial Pre-rendering и как он работает

[Fiber](./docs/react/react_fiber.md) · [Suspense](./docs/react/react_suspense.md) · [useTransition и useDeferredValue](./docs/react/react_concurrent_hooks.md) · [Мемоизация](./docs/react/react_memorization.md) · [HOC](./docs/react/react_hoc.md) · [Error Boundary](./docs/react/react_errorboundary.md) · [Порядок рендеринга](./docs/react/react_rendering_order.md) · [Хук use()](./docs/react/react_use.md)

---

## 6. Хранение данных и управление состоянием

Когда данных становится больше, чем может вместить один компонент, нужны инструменты для глобального состояния и клиентского хранения. Разберись, когда что применять.

- [ ] Что такое localStorage, sessionStorage и Cookies — в чём разница, ограничения, безопасность
- [ ] Что такое IndexedDB и когда его стоит использовать вместо Web Storage
- [ ] Что такое prop drilling и как Context API решает эту проблему
- [ ] Как работает Context API — `createContext`, `Provider`, `useContext` / `use()`
- [ ] Когда Context API — плохой выбор (высокочастотные обновления) и что использовать вместо него
- [ ] Что такое Zustand — и чем он отличается от Redux (нет провайдеров, минимум бойлерплейта)
- [ ] Что такое TanStack Query — и зачем он нужен (кэширование, фоновое обновление, дедупликация)
- [ ] Когда использовать Zustand (клиентское/UI-состояние), а когда TanStack Query (серверное/асинхронное состояние)
- [ ] Что такое Redux Toolkit и RTK Query — и актуален ли Redux в 2026 году
- [ ] Что такое `useReducer` + `useContext` как «мини-Redux»

[Клиентское хранение](./docs/state-management/client-storage.md) · [Zustand](./docs/state-management/zustand.md) · [TanStack Query](./docs/state-management/tanstack_query.md) · [useContext](./docs/react/react_usecontext.md)

---

## 7. API и коммуникация

Фронтенд не живёт в вакууме — он постоянно общается с бэкендом. Нужно понимать, какие протоколы и подходы существуют, и когда какой применять.

- [ ] Что такое REST и какие у него ограничения (stateless, ресурсы, HTTP-методы)
- [ ] Что такое over-fetching и under-fetching — и как GraphQL решает эти проблемы
- [ ] Чем GraphQL отличается от REST — и в каких случаях что лучше выбрать
- [ ] Что такое WebSocket и чем он отличается от HTTP и SSE (Server-Sent Events)
- [ ] Как организовать WebSocket-соединение в React (хук `useWebSocket`, переподключения)
- [ ] Что такое паттерн BFF (Backend for Frontend) и зачем он нужен
- [ ] Как обрабатывать ошибки API и какие коды ответов HTTP нужно знать

[REST](./docs/api-communication/rest.md) · [GraphQL vs REST](./docs/api-communication/graphql-vs-rest.md) · [WebSocket](./docs/api-communication/websocket_react_next.md) · [BFF](./docs/architecture/backend_for_frontend.md)

---

## 8. Архитектура и паттерны проектирования

На уровне Middle+ спрашивают не «как написать компонент», а «как организовать приложение». Паттерны, принципы, структура файлов — всё это показывает, что ты мыслишь как инженер, а не как кодер.

- [ ] Что такое композиция компонентов и почему она важнее наследования в React
- [ ] Что такое Compound Components, Render Props и Container/Presentational паттерны
- [ ] Что такое SOLID — и как каждый принцип применяется в React-компонентах
- [ ] Что такое DRY, KISS, YAGNI — и как они проявляются во фронтенде
- [ ] Что такое MVC, MVP и MVVM — и как React/Vue соотносятся с этими паттернами
- [ ] Какие паттерны GoF встречаются во фронтенде (Абстрактная фабрика, Декоратор, Синглтон, Наблюдатель)
- [ ] Как организовать структуру файлов в React/Next.js проекте (feature-based, atomic design)
- [ ] Что такое паттерн BFF и когда его стоит применять

[Паттерны проектирования](./docs/architecture/design_patterns.md) · [SOLID](./docs/architecture/solid-frontend.md) · [DRY/KISS/YAGNI](./docs/architecture/dry-kiss-yagni-frontend.md) · [MVC/MVVM](./docs/architecture/mvc-mvvm-article.md) · [GoF паттерны](./docs/architecture/gof_patterns_frontend.md) · [BFF](./docs/architecture/backend_for_frontend.md)

---

## 9. Next.js

Next.js — основной full-stack фреймворк для React. Серверные компоненты, файловая маршрутизация, кэширование, Server Actions — всё это нужно знать.

- [ ] Что такое App Router и чем он отличается от Pages Router
- [ ] Как работает файловая маршрутизация: динамические сегменты, Route Groups, параллельные и перехватывающие роуты
- [ ] Что такое серверные компоненты (Server Components) и чем они отличаются от клиентских (`"use client"`)
- [ ] Как правильно компоновать серверные и клиентские компоненты (правило: серверный нельзя рендерить внутри клиентского, но можно передать через `children`)
- [ ] Как работает загрузка данных в App Router (`async/await` в серверных компонентах, без `useEffect`)
- [ ] Что такое Route Handlers (`route.ts`) и когда их использовать вместо Server Actions
- [ ] Что такое Server Actions (`"use server"`) и как они работают под капотом
- [ ] Какие 4 уровня кэша есть в Next.js (Data Cache, Request Memoization, Full Route Cache, Router Cache)
- [ ] Как работает `revalidatePath` и `revalidateTag` — и чем time-based revalidation отличается от on-demand
- [ ] Что такое RSC Payload и как серверные компоненты отправляют данные клиенту
- [ ] Что такое Edge Runtime и чем он отличается от Node.js (V8 Isolate, ограничения API)
- [ ] Как Next.js оптимизирует навигацию (`<Link>`, prefetching) и изображения (`<Image>`)
- [ ] Что такое Next.js как монолит с прямым обращением к БД — плюсы, минусы, подводные камни
- [ ] Как работает `next/headers` и `next/navigation` в серверных компонентах

[Роутинг](./docs/next/next_routing.md) · [Data Fetching](./docs/next/next_data_fetching.md) · [Серверные и клиентские компоненты](./docs/next/next_server_client_composition.md) · [Route Handlers и Server Actions](./docs/next/next_route_handlers_server_actions.md) · [Кэширование](./docs/next/next_caching.md) · [Оптимизация](./docs/next/next_optimization.md) · [RSC Payload](./docs/next/next_rsc_payload.md) · [Edge Runtime](./docs/next/next_edge_runtime.md) · [Монолит и БД](./docs/next/next_monolith_direct_db.md) · [Server API](./docs/next/next-server-api.md)

---

## 10. Производительность

Быстрый сайт — это не бонус, а требование. Core Web Vitals влияют на SEO и конверсию. Нужно понимать, что замедляет приложение и как это измерить.

- [ ] Что такое Core Web Vitals: TTFB, FCP, LCP, CLS, INP — и какие нормальные значения
- [ ] Какие методы рендеринга существуют (CSR, SSR, SSG, ISR) — и когда какой применять
- [ ] Что такое гидратация и какие у неё проблемы (большой JS-бандл блокирует интерактивность)
- [ ] Что такое Selective Hydration и Partial Hydration в Next.js
- [ ] Как устроен браузер: процессы (Browser Process, Renderer, GPU), потоки, Site Isolation
- [ ] Что такое Web Workers и когда их нужно использовать (тяжёлые вычисления вне main thread)
- [ ] Что такое виртуализация списков и когда она необходима (react-window, react-virtuoso)
- [ ] Как оптимизировать размер бандла: code splitting, tree shaking, динамические импорты
- [ ] Что такое React Profiler и как находить узкие места производительности

[Методы рендеринга](./docs/performance/rendering-methods.md) · [Гидратация](./docs/performance/hydration.md) · [Метрики производительности](./docs/performance/web-performance-metrics.md) · [Архитектура браузера](./docs/performance/browser-architecture.md) · [Web Workers](./docs/performance/web_workers.md) · [Виртуализация списков](./docs/performance/list_virtualization.md)

---

## 11. Безопасность

Фронтенд — точка входа для большинства атак. XSS, CSRF, утечки секретов — всё это реальные угрозы, которые спрашивают на собеседованиях. Для компаний, готовящихся к SOC2 и CASA, важно не только писать безопасный код, но и понимать процессы: управление зависимостями, access reviews, change management, incident response.

- [ ] Что такое XSS (Cross-Site Scripting) — какие виды существуют (reflected, stored, DOM-based, mutation, blind)
- [ ] Как React, Vue, Nuxt и Next.js защищают от XSS по умолчанию и где их защита заканчивается
- [ ] Чем экранирование отличается от санитизации и валидации
- [ ] Что такое `dangerouslySetInnerHTML` и `v-html` и почему они опасны
- [ ] Что такое CSRF (Cross-Site Request Forgery) — механика атаки и почему она работает даже с JSON API
- [ ] Как SameSite cookies защищают от CSRF и чем отличаются `Strict`, `Lax` и `None`
- [ ] Что такое Content Security Policy (CSP), её основные директивы и режим Report-Only
- [ ] Какие HTTP security-заголовки нужно знать: HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, COOP, COEP, CORP
- [ ] Чем аутентификация отличается от авторизации
- [ ] Сессии через cookies vs JWT: плюсы, минусы, риски
- [ ] Access token и refresh token: зачем нужна пара и что такое token rotation
- [ ] OAuth 2.0, OpenID Connect и PKCE: для чего нужны и как работают
- [ ] Где хранить токены: cookies, localStorage, memory — trade-offs
- [ ] Почему нельзя хранить секреты (API-ключи, приватные ключи) в клиентском коде
- [ ] Как `NEXT_PUBLIC_` и `runtimeConfig.public` влияют на утечку секретов
- [ ] Какие особенности безопасности Server Actions в Next.js и как их правильно валидировать
- [ ] Как Next.js разделяет Server Components и Client Components и почему это важно для безопасности
- [ ] Что такое supply chain attack и известные инциденты (ua-parser-js, colors, event-stream)
- [ ] Как работает `npm audit`, зачем нужны lock-файлы
- [ ] Что такое SOC2 и CASA и почему они важны для фронтенда
- [ ] Что такое access review и change management
- [ ] Как фронтенд-разработчик участвует в incident response
- [ ] Что такое Secure SDLC и threat modeling

[Безопасность веб-приложений](./docs/security/security.md) · [XSS](./docs/security/security-xss-deep-dive.md) · [CSRF](./docs/security/security-csrf-deep-dive.md) · [CSP](./docs/security/security-csp-deep-dive.md) · [HTTP-заголовки](./docs/security/security-http-headers.md) · [Аутентификация и авторизация](./docs/security/security-authn-authz.md) · [Секреты](./docs/security/security-secrets-management.md) · [Next.js](./docs/security/security-nextjs.md) · [Vue/Nuxt](./docs/security/security-vue-nuxt.md) · [Зависимости](./docs/security/security-dependency-supply-chain.md) · [SOC2 и CASA](./docs/security/security-soc2-casa-workflows.md)

---

## 12. Тестирование

Тесты — это не опция, а часть профессиональной разработки. Нужно понимать пирамиду тестирования, уметь писать unit-тесты, тестировать компоненты и E2E.

- [ ] Что такое пирамида тестирования и Testing Trophy — и чем они отличаются
- [ ] Что такое FIRST-принципы и паттерн AAA (Arrange-Act-Assert)
- [ ] Что такое TDD (Red-Green-Refactor) и когда его стоит применять
- [ ] Что такое Vitest и чем он отличается от Jest
- [ ] Как тестировать React-компоненты с React Testing Library — `render`, `screen`, queries, `fireEvent` vs `userEvent`
- [ ] Что такое мокирование: Stub, Spy, Mock — и чем `vi.fn()`, `vi.spyOn()`, `vi.mock()` отличаются
- [ ] Что такое MSW (Mock Service Worker) и зачем мокировать API на сетевом уровне
- [ ] Что такое Playwright и чем он отличается от Cypress
- [ ] Как тестировать Next.js: клиентские компоненты, Server Components, Server Actions, Route Handlers
- [ ] Что такое визуальное регрессионное тестирование (Chromatic, Percy, `toHaveScreenshot`)
- [ ] Что такое Lighthouse CI и как тестировать производительность в CI/CD
- [ ] Что такое Sentry и как интегрировать мониторинг ошибок в React/Next.js
- [ ] Что такое Spec-Driven Development и чем он отличается от классического TDD
- [ ] Как тестировать Vue-компоненты с Vue Test Utils
- [ ] Как тестировать Nuxt 3: компоненты, composables, server routes, middleware

[Основы тестирования](./docs/testing/testing-fundamentals.md) · [Vitest](./docs/testing/testing-vitest.md) · [React-компоненты](./docs/testing/testing-react-components.md) · [Vue-компоненты](./docs/testing/testing-vue-components.md) · [Мокирование](./docs/testing/testing-mocking.md) · [Playwright](./docs/testing/testing-e2e-playwright.md) · [Тестирование Next.js](./docs/testing/testing-nextjs.md) · [Тестирование Nuxt](./docs/testing/testing-nuxt.md) · [Визуальное тестирование](./docs/testing/testing-visual-regression.md) · [Тестирование производительности](./docs/testing/testing-performance.md) · [Sentry](./docs/testing/sentry-guide.md) · [Spec-Driven Development](./docs/testing/spec-driven-development.md)

---

## 13. Сборка, CI/CD и деплой

Понимание того, что происходит между `npm run build` и production, показывает, что ты видишь картину целиком. Всё чаще спрашивают на Middle+ позициях.

- [ ] Что такое бандлер и зачем он нужен (Webpack vs Vite — подходы и отличия)
- [ ] Что такое code splitting и lazy loading (`React.lazy`, `import()`)
- [ ] Что такое tree shaking и как он удаляет мёртвый код
- [ ] Как анализировать размер бандла (webpack-bundle-analyzer, source-map-explorer)
- [ ] Что такое CI/CD и зачем он нужен фронтендеру
- [ ] Как работают GitHub Actions для автоматизации тестов, линтинга и деплоя
- [ ] Что такое монорепозиторий (monorepo) и чем Turborepo отличается от Nx
- [ ] Что такое workspace'ы в npm/pnpm/Yarn и как управлять зависимостями между пакетами
- [ ] Что такое Blue-Green Deployment, Canary Releases и Feature Flags
- [ ] Что такое zero-downtime deployment и rollback strategies
- [ ] Что такое Preview Deployments (деплой каждого PR)
- [ ] Чем CSR, SSR, SSG и ISR отличаются друг от друга и когда какой применять
- [ ] Что такое CDN и как она ускоряет доставку контента
- [ ] Что такое Edge Functions и Edge Runtime — плюсы, минусы и ограничения
- [ ] Какие HTTP-заголовки кэширования нужно знать: `Cache-Control`, `s-maxage`, `stale-while-revalidate`
- [ ] Как задеплоить Next.js: `output: 'export'`, `output: 'standalone'`, Vercel, Docker
- [ ] Что такое `NEXT_PUBLIC_` и почему секреты нельзя выносить на клиент
- [ ] Как задеплоить Nuxt 3: `nuxt build`, `nuxt generate`, Nitro presets, `routeRules`
- [ ] Что такое runtime config в Nuxt и чем он отличается от `NEXT_PUBLIC_`
- [ ] В чём разница между Vercel, Netlify, Railway, Render, Fly.io и Cloudflare
- [ ] Когда выбрать serverless, а когда контейнеры
- [ ] Как организовать SSG + CDN + Edge Functions в одной архитектуре

[Сборка и CI/CD](./docs/build-and-deployment/build-tools-ci-cd.md) · [Стратегии рендеринга](./docs/build-and-deployment/deployment-rendering-strategies.md) · [Деплой Next.js](./docs/build-and-deployment/deployment-nextjs.md) · [Деплой Nuxt 3](./docs/build-and-deployment/deployment-nuxt.md) · [Платформы деплоя](./docs/build-and-deployment/deployment-platforms.md) · [Монорепозитории](./docs/build-and-deployment/monorepos.md) · [Оптимизация бандла](./docs/build-and-deployment/advanced-bundle-optimization.md) · [Стратегии деплоя](./docs/build-and-deployment/deployment-strategies.md)

---

## 14. Платформы: React Native, PWA и Preact

Расширение горизонтов за пределы веба. React Native позволяет писать мобильные приложения на React, PWA — делать веб-приложения, работающие как нативные, а Preact — использовать лёгкий React-совместимый рантайм.

- [ ] Что такое React Native и чем он отличается от React (нативные компоненты вместо DOM)
- [ ] Что такое новая архитектура React Native (Fabric + TurboModules)
- [ ] Чем Expo отличается от Bare CLI — и когда что выбирать
- [ ] Что такое PWA (Progressive Web App) и какие три условия необходимы (HTTPS, Manifest, Service Worker)
- [ ] Что такое Service Workers и какие стратегии кэширования существуют
- [ ] Как работают push-уведомления и Background Sync в PWA
- [ ] Какие ограничения есть у PWA по сравнению с нативными приложениями
- [ ] Что такое Preact и чем он отличается от React (размер бандла, нативные события, отсутствие Concurrent features)
- [ ] Что такое Preact Signals и чем они отличаются от `useState`

[React Native](./docs/platforms/react_native_intro.md) · [PWA](./docs/platforms/pwa.md) · [Preact](./docs/platforms/preact.md)

---
