# React `use()` — Полное руководство

## Содержание

1. [Что такое `use()`](#что-такое-use)
2. [Условный вызов — ключевое отличие](#условный-вызов--ключевое-отличие)
3. [`use()` с Promise](#use-с-promise)
4. [`use()` с Context](#use-с-context)
5. [Интеграция с Suspense](#интеграция-с-suspense)
6. [Интеграция с Error Boundaries](#интеграция-с-error-boundaries)
7. [Сравнение с другими хуками](#сравнение-с-другими-хуками)
8. [Паттерны и сценарии использования](#паттерны-и-сценарии-использования)
9. [Ограничения и правила](#ограничения-и-правила)
10. [Лучшие практики](#лучшие-практики)
11. [Антипаттерны](#антипаттерны)
12. [История появления](#история-появления)

---

## Что такое `use()`

Хук `use()` — это API, представленный в **React 19.0** (декабрь 2024), который позволяет **читать значение Promise или Context непосредственно на этапе рендера компонента**. Он решает две фундаментальные задачи:

1. **Разворачивание (unwrap) Promise** — компонент получает асинхронные данные прямо в теле функции. Если Promise ещё не разрешён, `use()` «приостанавливает» (suspend) компонент, и React показывает ближайший `<Suspense fallback>`. Когда Promise разрешается, React автоматически продолжает рендер с полученным значением.
2. **Чтение Context** — полная замена `useContext()`, но с возможностью условного вызова и единым синтаксисом для любого контекста.

```jsx
import { use } from "react";

function UserProfile({ userPromise }) {
  const user = use(userPromise); // Приостанавливается, пока Promise не разрешится
  return <h1>Добро пожаловать, {user.name}</h1>;
}
```

Ключевая идея `use()`: **стереть границу между синхронным и асинхронным кодом** в компонентах React. Вы пишете компонент так, как будто данные уже доступны, а инфраструктура React (Suspense, Error Boundaries) берёт на себя заботу об ожидании, ошибках и состояниях загрузки.

До появления `use()` для загрузки данных требовался бойлерплейт из трёх хуков (`useState`, `useEffect`, условный рендеринг) и ручное управление состояниями `loading`, `error` и `data`. Теперь достаточно одного вызова `use(promise)`, обёрнутого в `<Suspense>`.

---

## Условный вызов — ключевое отличие

Единственный хук в React, который **можно вызывать условно** — внутри `if`, `for`, `while`, `switch` и досрочных `return`. Все остальные хуки (`useState`, `useEffect`, `useContext` и т.д.) обязаны вызываться на верхнем уровне компонента в строго одинаковом порядке при каждом рендере.

Причина в том, что `use()` **не управляет состоянием** — он только читает значение из внешнего источника (Promise или Context). Ему не нужно сохранять своё положение в списке хуков между рендерами.

```jsx
function Dashboard({ userPromise, rolePromise, isAdmin }) {
  if (isAdmin) {
    const adminData = use(adminPromise);   // ✅ Условный вызов
    const role = use(rolePromise);         // ✅ Условный вызов
    return <AdminPanel data={adminData} role={role} />;
  }

  const user = use(userPromise);            // ✅ Тоже работает
  return <UserPanel user={user} />;
}
```

Это открывает паттерны, невозможные с обычными хуками:

```jsx
function ProductPage({ productsPromise, featuredPromise }) {
  const featured = use(featuredPromise);

  if (featured.length === 0) {
    return <EmptyState />;  // Ранний выход — use() даже не вызывается для productsPromise
  }

  const products = use(productsPromise);
  return <ProductGrid products={products} featured={featured.slice(0, 3)} />;
}
```

---

## `use()` с Promise

### Механика работы

Когда `use(promise)` вызывается во время рендера, происходит следующее:

| Состояние Promise | Поведение `use()` |
|---|---|
| **pending** (ещё не разрешён) | React «приостанавливает» компонент. Рендер этого поддерева прерывается, ближайший `<Suspense>` показывает fallback. |
| **fulfilled** (успешно разрешён) | Возвращает значение, которым разрешился Promise. Рендер продолжается как обычно. |
| **rejected** (отклонён с ошибкой) | React «выбрасывает» ошибку. Её перехватывает ближайшая `<ErrorBoundary>`. |

Внутренне `use()` использует механизм, схожий с выбрасыванием Promise (throw-promise pattern), на котором построен Suspense с момента его появления. Но программисту не нужно думать об этом — вызов выглядит как обычное синхронное чтение.

### Базовый пример: загрузка данных

```jsx
import { use, Suspense } from "react";

// Промис создаётся ДО рендера — ключевой принцип render-as-you-fetch
const userPromise = fetch("/api/user/42").then(r => r.json());

function App() {
  return (
    <Suspense fallback={<Skeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

function UserProfile({ userPromise }) {
  const user = use(userPromise);
  return (
    <div>
      <Avatar src={user.avatar} />
      <h2>{user.name}</h2>
      <p>{user.bio}</p>
    </div>
  );
}
```

### Параллельная загрузка нескольких ресурсов

В отличие от `await`, где каждый следующий запрос ждёт предыдущий, `use()` не блокирует инициацию других Promise. Компонент приостанавливается, но сами промисы уже в полёте:

```jsx
function ProductPage({ productId }) {
  // Запросы запускаются параллельно, ДО рендера
  const productPromise = fetchProduct(productId);
  const reviewsPromise = fetchReviews(productId);
  const relatedPromise = fetchRelated(productId);

  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductDetail productPromise={productPromise} />
      <Suspense fallback={<ReviewsSkeleton />}>
        <ReviewsSection reviewsPromise={reviewsPromise} />
      </Suspense>
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedProducts relatedPromise={relatedPromise} />
      </Suspense>
    </Suspense>
  );
}
```

### Цепочка зависимых запросов

Если результат одного запроса нужен для другого, используйте паттерн «ленивого» Promise — оборачивайте второй запрос в функцию и создавайте промис только после получения первых данных:

```jsx
function OrderPage({ orderId }) {
  const orderPromise = fetchOrder(orderId);

  return (
    <Suspense fallback={<OrderSkeleton />}>
      <OrderDetail orderPromise={orderPromise} />
    </Suspense>
  );
}

function OrderDetail({ orderPromise }) {
  const order = use(orderPromise);

  // Второй запрос запускается только после получения order.userId
  const userPromise = fetchUser(order.userId);

  const user = use(userPromise);

  return (
    <div>
      <h2>Заказ #{order.id}</h2>
      <p>Клиент: {user.name}</p>
    </div>
  );
}
```

### Стабильная ссылка на Promise

**Критически важно:** Promise, переданный в `use()`, должен быть одним и тем же объектом между рендерами. Если при каждом рендере создавать новый промис, React попадёт в бесконечный цикл приостановки и повторного рендера.

```jsx
// ❌ НЕПРАВИЛЬНО: новый промис на каждом рендере
function Bad({ id }) {
  const data = use(fetch(`/api/item/${id}`).then(r => r.json())); // Бесконечный цикл!
  return <div>{data.name}</div>;
}

// ✅ ПРАВИЛЬНО: кэшируем промис
const cache = new Map();
function getUser(id) {
  if (!cache.has(id)) {
    cache.set(id, fetch(`/api/user/${id}`).then(r => r.json()));
  }
  return cache.get(id);
}

function Good({ id }) {
  const data = use(getUser(id)); // Стабильный промис — нет цикла
  return <div>{data.name}</div>;
}
```

На серверных компонентах React для этого идеально подходит встроенная функция `cache()`:

```jsx
import { cache } from "react";

const getUser = cache((id) => fetch(`/api/user/${id}`).then(r => r.json()));
```

---

## `use()` с Context

`use(context)` полностью заменяет `useContext(context)` с дополнительной возможностью условного вызова:

```jsx
import { use, createContext } from "react";

const ThemeContext = createContext("light");
const LocaleContext = createContext("ru");

function Button({ children }) {
  const theme = use(ThemeContext);   // Вместо useContext(ThemeContext)
  const locale = use(LocaleContext); // Вместо useContext(LocaleContext)
  return (
    <button className={`btn btn--${theme}`}>
      {locale === "ru" ? children : children.toUpperCase()}
    </button>
  );
}
```

### Условное чтение контекста

```jsx
function Header({ variant }) {
  if (variant === "minimal") {
    return <MinimalHeader />; // Контекст вообще не читается
  }

  const theme = use(ThemeContext); // Читается только в этой ветке
  return <FullHeader theme={theme} />;
}
```

### Асинхронный контекст (Promise внутри Context)

Мощный паттерн: положить Promise в контекст на уровне роута и читать его глубоко вложенными компонентами через `use()`:

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

function UserAvatar() {
  const userPromise = use(UserContext); // Читаем Promise из контекста
  const user = use(userPromise);         // Разворачиваем Promise
  return <img src={user.avatar} alt={user.name} />;
}

function UserName() {
  const userPromise = use(UserContext);
  const user = use(userPromise);
  return <span>{user.name}</span>;
}
```

Этот паттерн устраняет пробрасывание пропсов (prop drilling) для асинхронных данных: любой компонент внутри `UserRoute` может получить пользователя через `use(UserContext)`, независимо от глубины вложенности.

---

## Интеграция с Suspense

`use()` и `<Suspense>` — неразрывная пара. Когда `use(promise)` встречает неразрешённый Promise, он приостанавливает рендер текущего компонента. React поднимается вверх по дереву, пока не находит ближайшую границу `<Suspense>`, и показывает её `fallback` вместо приостановленного поддерева.

(Подробнее о Suspense: [React Suspense — Глубокое погружение](./react_suspense.md))

### Базовая связка

```jsx
<Suspense fallback={<LoadingSpinner />}>
  <UserDashboard />
</Suspense>
```

Каждый `use()` внутри `UserDashboard` (и глубже), встретивший pending Promise, приостановит всё поддерево, и пользователь увидит `<LoadingSpinner />`.

### Вложенные границы для независимой загрузки

```jsx
function ArticlePage({ articleId }) {
  return (
    <Layout>
      <Suspense fallback={<ArticleSkeleton />}>
        <ArticleBody articleId={articleId} />
      </Suspense>

      <Suspense fallback={<CommentsSkeleton />}>
        <CommentsSection articleId={articleId} />
      </Suspense>

      <Suspense fallback={<SidebarSkeleton />}>
        <RelatedArticles articleId={articleId} />
      </Suspense>
    </Layout>
  );
}
```

Результат: статья появляется, как только готова, даже если комментарии и боковая панель ещё загружаются. Каждая секция независима.

### Предотвращение waterfall

Без `use()` + Suspense компоненты часто выстраиваются в каскад: родитель загружает данные → рендерит дочерний → дочерний загружает свои данные. С `use()` вы запускаете все Promise заранее (до рендера), и данные загружаются параллельно:

```jsx
// Waterfall исключён: все три запроса стартуют одновременно
function Page({ id }) {
  const aPromise = fetchA(id); // Старт немедленно
  const bPromise = fetchB(id); // Старт немедленно
  const cPromise = fetchC(id); // Старт немедленно

  return (
    <Suspense fallback={<PageSkeleton />}>
      <SectionA promise={aPromise} />
      <SectionB promise={bPromise} />
      <SectionC promise={cPromise} />
    </Suspense>
  );
}
```

---

## Интеграция с Error Boundaries

Если Promise, переданный в `use()`, отклоняется (rejected), React выбрасывает ошибку, которую перехватывает ближайшая `<ErrorBoundary>`. Никакого `try/catch` внутри компонента не требуется.

```jsx
import { ErrorBoundary } from "react-error-boundary";

function App() {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <Suspense fallback={<Loading />}>
        <UserProfile userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Рекомендуемая в React 19 трёхслойная архитектура для любого асинхронного компонента:

```
<ErrorBoundary fallback={<ErrorUI />}>       ← Ловит rejected Promise
  <Suspense fallback={<LoadingUI />}>        ← Показывает загрузку для pending Promise
    <AsyncComponent />                       ← use(promise) внутри
  </Suspense>
</ErrorBoundary>
```

---

## Сравнение с другими хуками

### `use(promise)` vs `useEffect` + `useState`

| | `useEffect` + `useState` | `use(promise)` + `<Suspense>` |
|---|---|---|
| Код компонента | Три состояния: loading, error, data (5–15 строк бойлерплейта) | Одно состояние: data (1 строка) |
| Waterfall | Часто случается | Можно полностью избежать (render-as-you-fetch) |
| Гонка состояний | Нужен флаг `cancelled` / `AbortController` | React гарантирует консистентность |
| Состояние загрузки | Каждый компонент сам решает | Декларативно через `<Suspense>` |
| Обработка ошибок | `catch` в каждом `useEffect` | Единая `<ErrorBoundary>` на всё поддерево |

```jsx
// ❌ Традиционный подход (~12 строк бойлерплейта)
function OldProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  useEffect(() => {
    let cancelled = false;
    fetchUser(userId)
      .then(d => { if (!cancelled) { setUser(d); setLoading(false); } })
      .catch(e => { if (!cancelled) { setError(e); setLoading(false); } });
    return () => { cancelled = true; };
  }, [userId]);
  if (loading) return <Spinner />;
  if (error) return <ErrorPage error={error} />;
  return <h1>{user.name}</h1>;
}

// ✅ Подход с use() (~3 строки)
function NewProfile({ userPromise }) {
  const user = use(userPromise);
  return <h1>{user.name}</h1>;
}
```

### `use(context)` vs `useContext(context)`

| | `useContext()` | `use(context)` |
|---|---|---|
| Условный вызов | ❌ Запрещён | ✅ Разрешён |
| В циклах | ❌ Запрещён | ✅ Разрешён |
| После return | ❌ Запрещён | ✅ Разрешён |
| Синтаксис | `useContext(MyContext)` | `use(MyContext)` |
| Совместимость | React 16.8+ | React 19.0+ |

Функционально идентичны для прямого чтения контекста. `use()` следует предпочитать в новых проектах на React 19.

### `use()` vs `useMemo()` для кэширования Promise

`useMemo` может кэшировать Promise, но **не умеет** его разворачивать. `use()` разворачивает, но сам по себе **не кэширует**:

```jsx
// useMemo кэширует создание Promise, но не разворачивает
const promise = useMemo(() => fetchUser(id), [id]);
// Всё равно нужен useEffect для чтения...

// use() разворачивает Promise, но не кэширует его создание
const data = use(fetchUser(id)); // fetchUser вызывается на КАЖДОМ рендере!
```

Правильная комбинация: `useMemo` (или `cache()`) для стабилизации Promise + `use()` для разворачивания:

```jsx
const promise = useMemo(() => fetchUser(id), [id]);
const data = use(promise);
```

---

## Паттерны и сценарии использования

### 1. Классический render-as-you-fetch

Промис создаётся **до** рендера — в обработчике события, при инициализации роута или в родительском компоненте:

```jsx
function ProductRoute({ productId }) {
  const productPromise = fetchProduct(productId); // Старт до рендера ProductPage

  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductPage productPromise={productPromise} />
    </Suspense>
  );
}
```

### 2. Асинхронный контекст роута

```jsx
const DataContext = createContext(null);

function App() {
  const userPromise = useMemo(() => fetchCurrentUser(), []);
  const configPromise = useMemo(() => fetchAppConfig(), []);

  return (
    <DataContext value={{ userPromise, configPromise }}>
      <Suspense fallback={<AppSkeleton />}>
        <MainLayout />
      </Suspense>
    </DataContext>
  );
}

function Header() {
  const { configPromise } = use(DataContext);
  const config = use(configPromise);
  return <header style={{ background: config.theme.primary }}>...</header>;
}

function Sidebar() {
  const { userPromise } = use(DataContext);
  const user = use(userPromise);
  return <nav>{user.name}</nav>;
}
```

### 3. Условная загрузка по ролям

```jsx
function DocumentViewer({ userPromise, documentId }) {
  const user = use(userPromise);

  if (user.role === "admin") {
    const fullDocument = use(fetchFullDocument(documentId));
    return <AdminDocumentView data={fullDocument} />;
  }

  if (user.role === "editor") {
    const editData = use(fetchEditorData(documentId));
    return <EditorView data={editData} />;
  }

  const publicData = use(fetchPublicDocument(documentId));
  return <PublicView data={publicData} />;
}
```

### 4. Ленивая инициализация через `cache()`

```jsx
import { cache } from "react";

const getPost = cache(async (id) => {
  const res = await fetch(`/api/posts/${id}`);
  return res.json();
});

const getComments = cache(async (postId) => {
  const res = await fetch(`/api/posts/${postId}/comments`);
  return res.json();
});

function PostPage({ postId }) {
  return (
    <Suspense fallback={<PostSkeleton />}>
      <Post postId={postId} />
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={postId} />
      </Suspense>
    </Suspense>
  );
}

function Post({ postId }) {
  const post = use(getPost(postId)); // Дедуплицируется через cache()
  return <article><h1>{post.title}</h1><p>{post.body}</p></article>;
}

function Comments({ postId }) {
  const comments = use(getComments(postId));
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}
```

### 5. Чтение нескольких контекстов разного типа

```jsx
function SmartButton({ children, onClick }) {
  const theme = use(ThemeContext);
  const locale = use(LocaleContext);
  const permissions = use(PermissionsContext);
  const flags = use(FeatureFlagsContext);

  const label = locale === "ru" ? children : children.toUpperCase();
  const disabled = !permissions.includes("click");

  return (
    <button
      className={`btn btn--${theme}`}
      disabled={disabled || !flags.enableButtons}
      onClick={onClick}
    >
      {label}
    </button>
  );
}
```

---

## Ограничения и правила

### Где можно вызывать

| Контекст вызова | Разрешён? |
|---|---|
| Тело функционального компонента | ✅ Да |
| Тело пользовательского хука | ✅ Да |
| Внутри `if` / `for` / `while` | ✅ Да |
| Внутри `switch` / после `return` | ✅ Да |
| Обработчики событий (`onClick`, `onSubmit`) | ❌ Нет |
| `useEffect` / `useLayoutEffect` | ❌ Нет |
| Колбэки (`setTimeout`, `setInterval`) | ❌ Нет |
| Классовые компоненты | ❌ Нет |

### Ключевые правила

1. **Promise должен быть стабильным между рендерами.** Один и тот же объект Promise должен передаваться в `use()` при каждом рендере. Новый промис на каждом рендере приводит к бесконечному циклу: приостановка → разрешение → повторный рендер с новым промисом → снова приостановка.

2. **Promise должен создаваться до рендера.** Не создавайте промис прямо в аргументе `use()`:
   ```jsx
   const data = use(fetch(url)); // ❌ Новый промис на каждом рендере
   ```

3. **`use()` сам по себе не вызывает повторный рендер.** В отличие от `useState`, разрешение Promise не приводит к автоматическому ре-рендеру компонента. React использует внутренний механизм Suspense для продолжения прерванного рендера.

4. **Обязательно наличие `<Suspense>` в дереве.** Если ни одного `<Suspense>` нет выше компонента с `use(pendingPromise)`, React выбросит необработанный Promise и вы получите ошибку в консоли или белый экран.

5. **Не работает с серверными компонентами React.** На серверных компонентах используйте нативный `async/await`:
   ```jsx
   // ✅ Серверный компонент
   async function ServerComponent({ id }) {
     const data = await db.query(...);
     return <div>{data}</div>;
   }

   // ❌ use() — только в клиентских компонентах
   "use client";
   function ClientComponent({ promise }) {
     const data = use(promise);
     return <div>{data}</div>;
   }
   ```

---

## Лучшие практики

### 1. Render-as-you-fetch: создавайте Promise до рендера

```jsx
// ✅ Promise создаётся ДО вызова компонента
function Page({ id }) {
  const promise = useMemo(() => fetchData(id), [id]);
  return (
    <Suspense fallback={<Skeleton />}>
      <Content promise={promise} />
    </Suspense>
  );
}
```

### 2. Используйте `cache()` для дедупликации

```jsx
import { cache } from "react";

const getUser = cache((id) => fetch(`/api/user/${id}`).then(r => r.json()));

// Вызывается из нескольких мест — запрос выполнится только один раз
function ComponentA({ id }) { const user = use(getUser(id)); ... }
function ComponentB({ id }) { const user = use(getUser(id)); ... }
```

### 3. Трёхслойная обёртка: ErrorBoundary → Suspense → Компонент

```jsx
<ErrorBoundary fallback={<ErrorPage />}>
  <Suspense fallback={<LoadingPage />}>
    <AsyncComponent promise={dataPromise} />
  </Suspense>
</ErrorBoundary>
```

### 4. Дробите Suspense для независимой загрузки секций

Не заворачивайте всю страницу в один `<Suspense>` — оборачивайте независимые секции по отдельности, чтобы пользователь видел контент по мере готовности.

### 5. Передавайте Promise через пропсы, а не создавайте внутри

```jsx
// ✅ Пропс
function Child({ promise }) { const data = use(promise); ... }

// ❌ Создание внутри
function Child({ id }) {
  const data = use(fetchData(id)); // Проблема со стабильностью ссылки
}
```

### 6. Для простого чтения контекста предпочитайте `use()` вместо `useContext()`

Начиная с React 19, `use(ThemeContext)` — более современный и гибкий синтаксис, не имеющий недостатков по сравнению с `useContext(ThemeContext)`.

---

## Антипаттерны

### ❌ Создание Promise прямо в аргументе `use()`

```jsx
function Bad({ id }) {
  const data = use(fetch(`/api/${id}`).then(r => r.json())); // Бесконечный цикл!
  return <div>{data}</div>;
}
```

### ❌ Использование `use()` в обработчиках событий

```jsx
function Bad() {
  const handleClick = () => {
    const data = use(somePromise); // ❌ Ошибка: не во время рендера
    console.log(data);
  };
  return <button onClick={handleClick}>Click</button>;
}
```

### ❌ Замена всей асинхронной логики на `use()`

```jsx
// ❌ Не каждое действие с Promise должно идти через use()
function Bad({ dataPromise }) {
  const data = use(dataPromise);
  const handleSave = async () => {
    await saveData(data); // Обычный async/await в обработчике — это нормально
  };
  return <button onClick={handleSave}>Save</button>;
}
```

`use()` предназначен для **чтения данных при рендере**. Мутации, отправка форм, аналитика — по-прежнему используют `async/await` в обработчиках или серверные действия React 19.

### ❌ Отсутствие ErrorBoundary при работе с Promise

Без `ErrorBoundary` отклонённый Promise в `use()` приведёт к необработанной ошибке и потенциальному «белому экрану»:

```jsx
// ❌ Опасно — нет ErrorBoundary
<Suspense fallback={<Loading />}>
  <Component promise={unreliablePromise} />
</Suspense>

// ✅ Безопасно
<ErrorBoundary fallback={<ErrorUI />}>
  <Suspense fallback={<Loading />}>
    <Component promise={unreliablePromise} />
  </Suspense>
</ErrorBoundary>
```

### ❌ Глубокая вложенность Suspense без координации

Слишком много вложенных `<Suspense>` на одной странице может привести к «прыгающему» UI, когда секции появляются в разное время. Группируйте связанный контент под общим `<Suspense>`, а независимые секции изолируйте отдельными границами.

---

## История появления

| Версия | Дата | Событие |
|---|---|---|
| React 16.6 | Октябрь 2018 | Появление `<Suspense>` (только для `lazy()`). Заложен фундамент throw-promise механики. |
| React 18.0 | Март 2022 | Конкурентный рендеринг. `<Suspense>` начинает работать с data fetching (через сторонние библиотеки вроде Relay, SWR). |
| **React 19.0** | **Декабрь 2024** | **Хук `use()` становится публичным API.** Чтение Promise и Context, условный вызов, полная интеграция с Suspense и Error Boundaries. |
| React 19.1 | Март 2025 | Улучшения Suspense на клиенте, сервере и при гидратации. Косвенно усиливает стабильность `use()`. |
| React 19.2 | Октябрь 2025 | Батчинг границ Suspense в SSR, поддержка Web Streams. Расширение `use()` — более глубокая координация с конкурентным рендерингом. |

На момент 2026 года хук `use()` является рекомендованным способом чтения Promise и Context в клиентских компонентах React. В сочетании с `<Suspense>`, `<ErrorBoundary>`, серверными компонентами и конкурентным рендерингом он формирует основу современной архитектуры загрузки данных в React-приложениях.
