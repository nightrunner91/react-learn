# React Suspense — Глубокое погружение

## Содержание

1. [Что такое Suspense](#что-такое-suspense)
2. [Какую проблему решает Suspense](#какую-проблему-решает-suspense)
3. [Механика работы](#механика-работы)
4. [API и синтаксис](#api-и-синтаксис)
5. [Сценарии использования](#сценарии-использования)
6. [Suspense и хук `use()`](#suspense-и-хук-use)
7. [Suspense в SSR и потоковая передача](#suspense-в-ssr-и-потоковая-передача)
8. [Suspense и Error Boundaries](#suspense-и-error-boundaries)
9. [Suspense в React 19.1 и 19.2](#suspense-в-react-191-и-192)
10. [Лучшие практики](#лучшие-практики)
11. [История и эволюция](#история-и-эволюция)
12. [Антипаттерны](#антипаттерны)

---

## Что такое Suspense

`<Suspense>` — это компонент React, который позволяет **декларативно управлять состоянием загрузки** для всего поддерева компонентов. Вместо того чтобы каждый компонент самостоятельно обрабатывал состояния `loading`, `error` и `success` через `useState` + `useEffect`, Suspense переводит эту обязанность на инфраструктурный уровень React.

```jsx
import { Suspense } from "react";

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <UserDashboard />
    </Suspense>
  );
}
```

Когда `UserDashboard` (или любой компонент внутри него) «приостанавливается», React отображает `fallback` — `<LoadingSpinner />` — вместо приостановленного поддерева. Когда данные готовы, React автоматически отображает настоящее содержимое.

---

## Какую проблему решает Suspense

До Suspense каждый компонент, загружающий данные, вынужден был реализовывать конечный автомат из трёх состояний вручную:

```jsx
// ❌ Традиционный подход: гонка состояний и императивное управление
function UserProfile({ id }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetchUser(id)
      .then(data => { if (!cancelled) { setUser(data); setLoading(false); } })
      .catch(err => { if (!cancelled) { setError(err); setLoading(false); } });
    return () => { cancelled = true; };
  }, [id]);

  if (loading) return <Spinner />;
  if (error) return <ErrorPage error={error} />;
  return <Profile user={user} />;
}
```

**Проблемы этого подхода:**

1. **Бойлерплейт.** Каждый компонент дублирует логику `loading`/`error`/`success`.
2. **Waterfall-запросы.** Родитель ждёт свои данные → рендерит дочерний компонент → дочерний ждёт свои данные. Каскадная задержка.
3. **Гонка состояний.** При быстрой смене `id` старый ответ может прийти позже нового — нужен флаг `cancelled`.
4. **Несогласованность UI.** Один компонент показал спиннер, другой — данные, третий — ошибку. Нет единой точки управления загрузкой.
5. **Сложность SSR.** Традиционный подход не позволяет начать отправку HTML в браузер до завершения всех запросов.

**Suspense решает эти проблемы:**

| Проблема | Решение Suspense |
|---|---|
| Бойлерплейт | Не нужно вручную управлять `loading`/`error` в каждом компоненте |
| Waterfall | Компоненты могут начать загрузку данных до рендера (render-as-you-fetch) |
| Гонка состояний | React гарантирует консистентность: при смене пропсов предыдущий рендер отбрасывается |
| Несогласованность | Единый `fallback` для всего поддерева |
| SSR | Потоковая передача: HTML отправляется прогрессивно по мере готовности |

---

## Механика работы

### Принцип выбрасывания промиса (Throw-a-Promise)

Suspense основан на механизме, который можно назвать «выбрасывание промиса во время рендера»:

1. Во время рендера компонент обнаруживает, что данные ещё не готовы.
2. Компонент **выбрасывает (throw) промис** загрузки.
3. React **ловит** этот throw через внутренний механизм в ближайшей границе `<Suspense>`.
4. React отображает `fallback` вместо поддерева.
5. Когда промис разрешается, React **повторно рендерит** приостановленное поддерево.
6. Если промис отклоняется — ошибка «всплывает» до ближайшей `<ErrorBoundary>`.

```
┌─────────────────────────────────────────────────┐
│ <Suspense fallback={<Fallback />}>               │
│                                                   │
│   ┌───────────────────────────┐                   │
│   │ <UserDashboard>            │                   │
│   │   ┌─────────────────────┐ │                   │
│   │   │ <UserProfile>       │ │                   │
│   │   │   const user =      │ │                   │
│   │   │   use(userPromise); │ │ ← throw Promise  │
│   │   │   // ↑ «Приостановка»│ │                   │
│   │   └─────────────────────┘ │                   │
│   └───────────────────────────┘                   │
│                                                   │
│   → Пока Promise не resolved:                     │
│     отображается <Fallback />                     │
│   → После resolve:                                │
│     <UserDashboard /> рендерится заново            │
└─────────────────────────────────────────────────┘
```

### Несколько границ Suspense

Suspense можно вкладывать для детализированного контроля загрузки:

```jsx
<Suspense fallback={<PageSkeleton />}>
  <Header />                          {/* Мгновенно */}

  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />                       {/* Приостановлен → отдельный скелетон */}
  </Suspense>

  <Suspense fallback={<ContentSkeleton />}>
    <MainContent />                   {/* Приостановлен → отдельный скелетон */}
  </Suspense>
</Suspense>
```

Каждая граница `<Suspense>` действует независимо:
- React не ждёт разрешения *всех* Suspense-границ — каждая разрешается по мере готовности своих данных.
- Внешняя граница отображает `fallback`, только если внутренние тоже не готовы — иначе показывает уже разрешённые внутренние границы.

### Конкурентный рендеринг и прерывания

В React 18+ Suspense работает с конкурентным рендерером (Fiber). Когда компонент приостанавливается:

1. React **не отбрасывает** уже готовый рендер родительских компонентов.
2. Соседние Suspense-границы продолжают обрабатываться параллельно.
3. Если появляется более приоритетное обновление (например, ввод пользователя), React может **прервать** ожидание Suspense.
4. При возобновлении React продолжает с того же места.

Это контрастирует с React 16–17, где приостановка отбрасывала всё незавершённое дерево (синхронный режим legacy).

---

## API и синтаксис

### `<Suspense>`

```tsx
interface SuspenseProps {
  children: ReactNode;
  fallback: ReactNode;
}
```

- **`children`** — поддерево, которое может приостановиться.
- **`fallback`** — что показывать во время ожидания. Может быть любым React-узлом: `<Spinner />`, скелетон, текст.

### `lazy()`

```jsx
const Dashboard = lazy(() => import("./Dashboard"));
// Dashboard — это компонент, обёрнутый в lazy.
// При первом рендере выбрасывает промис загрузки модуля.
```

### `use()` (React 19)

Хук `use()` — основной способ «приостановить» компонент в React 19:

```jsx
import { use } from "react";

function UserProfile({ userPromise }) {
  const user = use(userPromise); // Приостанавливает компонент до разрешения userPromise
  return <h1>{user.name}</h1>;
}
```

В отличие от всех остальных хуков, `use()` можно вызывать **условно** (внутри `if`, циклов, return-early):

```jsx
function Profile({ userPromise, isAdmin }) {
  if (isAdmin) {
    const admin = use(adminPromise); // ✅ Условный вызов
    return <AdminPanel admin={admin} />;
  }
  const user = use(userPromise);
  return <UserPanel user={user} />;
}
```

`use()` также работает с контекстом, заменяя `useContext`:

```jsx
const theme = use(ThemeContext); // React 19: вместо useContext(ThemeContext)
```

---

## Сценарии использования

### 1. Загрузка данных (Data Fetching)

Классический паттерн: обернуть компонент, загружающий данные, в `<Suspense>`:

```jsx
function PostPage({ postId }) {
  // Промис создаётся ДО рендера — ключевой принцип render-as-you-fetch
  const commentsPromise = fetchComments(postId);

  return (
    <Suspense fallback={<PostSkeleton />}>
      <Post postId={postId} />
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments commentsPromise={commentsPromise} />
      </Suspense>
    </Suspense>
  );
}

function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);
  return (
    <ul>
      {comments.map(c => <li key={c.id}>{c.text}</li>)}
    </ul>
  );
}
```

Обратите внимание: `commentsPromise` создаётся **до** рендера `<Comments />`. Это позволяет запустить загрузку данных сразу, не дожидаясь рендера родительского `<Post />` — устранение waterfall.

### 2. Разделение кода (Code Splitting)

```jsx
import { lazy, Suspense } from "react";

const Editor = lazy(() => import("./Editor"));
const Chart = lazy(() => import("./Chart"));

function Dashboard() {
  return (
    <main>
      <h1>Dashboard</h1>
      <Suspense fallback={<div>Loading editor…</div>}>
        <Editor />
      </Suspense>
      <Suspense fallback={<div>Loading chart…</div>}>
        <Chart />
      </Suspense>
    </main>
  );
}
```

`lazy()` разделяет бандл на отдельные чанки, которые загружаются только при первом рендере компонента. Без `<Suspense>` `lazy()` выбросит необработанный промис и вызовет ошибку.

### 3. Асинхронные контексты и ресурсы

Паттерн, в котором данные загружаются на уровне роута и передаются через контекст как промис:

```jsx
const UserContext = createContext(null);

function UserRoute({ userId, children }) {
  const userPromise = useMemo(() => fetchUser(userId), [userId]);
  return (
    <UserContext value={userPromise}>
      <Suspense fallback={<UserSkeleton />}>
        {children}
      </Suspense>
    </UserContext>
  );
}

function UserName() {
  const userPromise = use(UserContext);
  const user = use(userPromise);
  return <span>{user.name}</span>;
}
```

### 4. Вложенные Suspense для независимых секций

```jsx
function ProductPage({ productId }) {
  return (
    <div>
      <Suspense fallback={<ProductHeroSkeleton />}>
        <ProductHero productId={productId} />
      </Suspense>

      <div style={{ display: "flex" }}>
        <Suspense fallback={<ReviewsSkeleton />}>
          <Reviews productId={productId} />
        </Suspense>
        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations productId={productId} />
        </Suspense>
      </div>
    </div>
  );
}
```

Каждая секция появляется независимо — пользователь видит контент прогрессивно, без ожидания самой медленной части.

### 5. `useTransition` + Suspense для навигации

При переходе между страницами `useTransition` позволяет сохранить предыдущий UI видимым, пока загружается новый:

```jsx
function App() {
  const [page, setPage] = useState("/");
  const [isPending, startTransition] = useTransition();

  function navigate(url) {
    startTransition(() => setPage(url));
  }

  return (
    <div>
      <nav>
        <a onClick={() => navigate("/about")}>About</a>
        <a onClick={() => navigate("/contact")}>Contact</a>
      </nav>
      {isPending && <GlobalSpinner />}
      <Suspense fallback={<PageSkeleton />}>
        <PageContent page={page} />
      </Suspense>
    </div>
  );
}
```

Во время перехода:
- Старая страница продолжает отображаться (не заменяется fallback'ом сразу).
- React ждёт заданное время (по умолчанию ~400 мс), прежде чем показать `fallback`.
- Если новый контент загружается быстро — старый UI заменяется без мигания спиннера.

---

## Suspense в SSR и потоковая передача

Одно из самых мощных применений Suspense — **потоковый серверный рендеринг** (Streaming SSR).

### Как это работает

Традиционный SSR ждёт загрузки **всех** данных перед отправкой первого байта HTML (All-or-Nothing):

```
Традиционный SSR:
[Запрос] → [Ждём все данные] → [Рендерим всё] → [Отправляем HTML]
                                      ↑── Time to First Byte: медленный
```

Потоковый SSR с Suspense отправляет HTML прогрессивно:

```
Потоковый SSR:
[Запрос] → [Рендерим статику] → [Отправляем HTML] → [Поток: Suspense fallback]
                                                        → [Поток: данные готовы → inline-скрипт замены]
                                      ↑── TTFB: быстрый
```

### Пример в Next.js App Router

```jsx
export default async function Page() {
  return (
    <main>
      <Header />
      <Suspense fallback={<Skeleton variant="hero" />}>
        <HeroBanner />         {/* Тяжёлый запрос к API */}
      </Suspense>
      <Suspense fallback={<Skeleton variant="reviews" />}>
        <Reviews />            {/* Ещё один тяжёлый запрос */}
      </Suspense>
    </main>
  );
}
```

HTML-поток выглядит так (*упрощённо*):

```html
<!-- Отправляется немедленно -->
<header>...</header>

<!-- Временно: заглушка для HeroBanner -->
<div hidden id="S:1"><!-- Скелетон --></div>

<!-- Временно: заглушка для Reviews -->
<div hidden id="S:2"><!-- Скелетон --></div>

<!-- ...позже, когда Promise разрешился: -->
<script>$RC("S:1", function() { /* замена скелетона на контент */ })</script>
<div hidden id="S:1"><!-- Настоящий контент --></div>

<script>$RC("S:2", function() { /* замена скелетона на контент */ })</script>
<div hidden id="S:2"><!-- Настоящий контент --></div>
```

Этот подход даёт:
- **Быстрый TTFB** — браузер начинает получать и парсить HTML немедленно.
- **Прогрессивный рендеринг** — контент появляется по мере готовности, а не одним куском.
- **Устойчивость** — медленный микросервис не блокирует всю страницу.

---

## Suspense и Error Boundaries

Suspense и Error Boundaries работают в тандеме:

- **Suspense** обрабатывает **pending** состояние (промис ещё не resolved).
- **Error Boundary** обрабатывает **rejected** состояние (промис отклонился).

```jsx
<ErrorBoundary fallback={<ErrorPage />}>
  <Suspense fallback={<LoadingPage />}>
    <DataComponent />
  </Suspense>
</ErrorBoundary>
```

Порядок важен: `ErrorBoundary` снаружи, `Suspense` внутри. Если поменять местами — ошибка из отклонённого промиса не будет поймана.

### Интеграция с react-error-boundary

```jsx
import { ErrorBoundary } from "react-error-boundary";
import { Suspense } from "react";

function DataSection() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <Suspense fallback={<Skeleton />}>
        <AsyncData />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### Совместная работа с `use()` (React 19)

`use()` унифицирует обработку: если переданный промис rejected — это автоматически становится ошибкой рендера, которую ловит ближайшая `ErrorBoundary`. Больше не нужен `try/catch` для `use(promise)`.

---

## Suspense в React 19.1 и 19.2

### React 19.1 — улучшенный Suspense

- Поддержка границ Suspense на всех фазах: клиент, сервер, гидратация.
- Улучшена обработка вложенных Suspense-границ в конкурентном режиме.

### React 19.2 — батчирование границ Suspense в SSR

До React 19.2 серверный потоковый рендеринг отправлял каждую Suspense-границу **независимо**, как только её данные становились доступны. Это приводило к тому, что контент «появлялся по частям» — сначала одна секция, через 50 мс другая.

React 19.2 вводит **батчирование**: границы Suspense, разрешающиеся в пределах короткого временного окна, группируются и отправляются вместе. Это приводит поведение серверного рендеринга в соответствие с клиентским, где границы Suspense всегда показываются атомарно.

```jsx
// React 19.2: границы, разрешившиеся в пределах одного «тика»,
// показываются одновременно, а не по отдельности.
<Suspense fallback={<Skeleton1 />}>
  <WidgetA />
</Suspense>
<Suspense fallback={<Skeleton2 />}>
  <WidgetB />
</Suspense>
// Если WidgetA и WidgetB разрешаются с разницей < 50 мс —
// оба появляются одновременно.
```

---

## Лучшие практики

### 1. Размещайте границы Suspense осмысленно

Не оборачивайте всё приложение в один `<Suspense>` — это вернёт проблему «всё или ничего».

```jsx
// ❌ Плохо: весь контент ждёт самую медленную часть
<Suspense fallback={<BigSpinner />}>
  <App />
</Suspense>

// ✅ Хорошо: независимые секции со своими скелетонами
<App>
  <Suspense fallback={<HeaderSkeleton />}>
    <Header />
  </Suspense>
  <Suspense fallback={<ContentSkeleton />}>
    <Content />
  </Suspense>
</App>
```

### 2. Создавайте промисы до рендера (render-as-you-fetch)

```jsx
// ❌ Плохо: waterfall — fetch внутри useEffect в компоненте
function Component() {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  if (!data) throw fetchData(); // ...

// ✅ Хорошо: промис создан до рендера
const dataPromise = fetchData();
function Component({ dataPromise }) {
  const data = use(dataPromise);
  // ...
}
```

### 3. Не используйте Suspense для обработчиков событий

Suspense работает только на фазе **рендера**. Обработчики событий не приостанавливаются:

```jsx
function handleClick() {
  const data = use(somePromise); // ❌ Ошибка: use() нельзя в обработчиках
}
```

Для асинхронных действий в обработчиках используйте `useTransition` + `useActionState`.

### 4. `<ErrorBoundary>` снаружи, `<Suspense>` внутри

```jsx
<ErrorBoundary fallback={<ErrorPage />}>
  <Suspense fallback={<Loading />}>
    <AsyncComponent />
  </Suspense>
</ErrorBoundary>
```

### 5. Используйте `useDeferredValue` для отложенных обновлений внутри Suspense

```jsx
function SearchResults({ query }) {
  return (
    <Suspense fallback={<ResultSkeleton />}>
      <ResultsList query={query} />
    </Suspense>
  );
}
```

Если `query` меняется слишком часто (ввод с клавиатуры), используйте `useDeferredValue`, чтобы не создавать каскад приостановок:

```jsx
const deferredQuery = useDeferredValue(query);
<Suspense fallback={<ResultSkeleton />}>
  <ResultsList query={deferredQuery} />
</Suspense>
```

### 6. Избегайте приостановки уже видимого контента

Когда пользователь уже видит контент, внезапное появление `fallback` (spinner поверх данных) — плохой UX. Используйте `useTransition`:

```jsx
const [isPending, startTransition] = useTransition();

function handleTabChange(tab) {
  startTransition(() => setActiveTab(tab));
}
// isPending === true → можно показать индикатор, но старый контент не скрывается
```

---

## История и эволюция

| Версия React | Возможности Suspense |
|---|---|
| **16.6** (2018) | Первое появление: `React.lazy()` + `<Suspense>` только для code splitting. Не поддерживал data fetching. Синхронный рендер. |
| **16.8** (2019) | Появление хуков. Suspense для data fetching — экспериментальный, за флагом `unstable_ConcurrentMode`. |
| **18.0** (2022) | Suspense для data fetching — стабильный. Конкурентный рендеринг. Потоковый SSR. `useTransition`, `useDeferredValue`. |
| **19.0** (2024) | Хук `use()` — нативный способ приостановки. Suspense + Actions + `useOptimistic`. |
| **19.1** (Март 2025) | Улучшенный Suspense на всех фазах. Owner Stack для отладки. |
| **19.2** (Октябрь 2025) | Батчирование Suspense-границ в SSR. Частичный предварительный рендеринг. `useEffectEvent`. `cacheSignal`. |

---

## Антипаттерны

### 1. Suspense без fallback

```jsx
// ❌ Паника в консоли: промис не пойман
<Suspense>
  <LazyComponent />
</Suspense>
```

`fallback` — обязательный проп.

### 2. Suspense вокруг всего приложения

```jsx
// ❌ Любой компонент с приостановкой скрывает ВСЁ приложение
<Suspense fallback={<FullPageSpinner />}>
  <App />
</Suspense>
```

Решение: размещайте границы вокруг конкретных поддеревьев.

### 3. Приостановка в компонентах без внешнего Suspense

```jsx
function App() {
  return <LazyComponent />; // ❌ Uncaught Promise — нужен <Suspense> выше по дереву
}
```

### 4. Создание промиса во время рендера

```tsx
function BadComponent() {
  const promise = fetchUser(id); // ❌ Новый промис на каждом рендере → бесконечные приостановки
  const user = use(promise);
}
```

Решение: мемоизировать промис или создавать его вне компонента:

```tsx
const userPromise = fetchUser(id);

function GoodComponent({ userPromise }) {
  const user = use(userPromise); // ✅ Стабильная ссылка
}
```

### 5. Suspense внутри Error Boundary (порядок наоборот)

```jsx
// ❌ Ошибка из отклонённого промиса не будет поймана
<Suspense fallback={<Loading />}>
  <ErrorBoundary fallback={<Error />}>
    <DataComponent />
  </ErrorBoundary>
</Suspense>
```

Правильный порядок:

```jsx
// ✅ ErrorBoundary снаружи
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Loading />}>
    <DataComponent />
  </Suspense>
</ErrorBoundary>
```

---

## Итог

Suspense — это фундаментальный переход от **императивного** управления асинхронными состояниями к **декларативному**. Вместо того чтобы каждый компонент вручную обрабатывал `loading`/`error`/`success`, Suspense позволяет:

- **Декларативно** описывать состояние загрузки (`fallback`) один раз для целого поддерева.
- **Прогрессивно** показывать контент по мере готовности (вложенные Suspense).
- **Эффективно** загружать данные без waterfall (render-as-you-fetch).
- **Потоково** отправлять HTML с сервера (Streaming SSR).
- **Консистентно** обрабатывать ошибки через пару Suspense + Error Boundary.

В сочетании с хуком `use()` (React 19), конкурентным рендерингом и потоковым SSR, Suspense является краеугольным камнем современной архитектуры React-приложений.
