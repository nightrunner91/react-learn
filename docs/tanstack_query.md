# TanStack Query — Глубокое погружение

## Содержание

1. [Что такое TanStack Query](#что-такое-tanstack-query)
2. [Какую проблему решает TanStack Query](#какую-проблему-решает-tanstack-query)
3. [Ключевые понятия](#ключевые-понятия)
4. [Базовое использование](#базовое-использование)
5. [Сценарии применения](#сценарии-применения)
6. [Мутации](#мутации)
7. [Продвинутые возможности](#продвинутые-возможности)
8. [TanStack Query vs Zustand](#tanstack-query-vs-zustand)
9. [Миграция с RTK Query](#миграция-с-rtk-query)
10. [Лучшие практики](#лучшие-практики)
11. [Антипаттерны](#антипаттерны)

---

## Что такое TanStack Query

TanStack Query (ранее React Query) — это **мощная библиотека для управления серверным состоянием** в JavaScript-приложениях. Она предоставляет хуки для загрузки, кэширования, синхронизации и обновления данных с сервера, избавляя разработчика от ручной реализации всей этой инфраструктуры.

```jsx
import { useQuery } from "@tanstack/react-query";

function UserList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then((res) => res.json()),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <ul>
      {data.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

Ключевые особенности TanStack Query:
- **Автоматическое кэширование** — данные кэшируются и переиспользуются между компонентами.
- **Фоновое обновление** — данные обновляются в фоне, пока пользователь видит кэшированную версию.
- **Дедупликация запросов** — несколько компонентов, запрашивающих одни данные, получают один запрос.
- **Оптимистичные обновления** — UI обновляется до подтверждения сервером.¹
- **Пагинация и бесконечная прокрутка** — встроенная поддержка.
- **Инвалидация кэша** — автоматическое обновление данных после мутаций.
- **Размер** — ~15 KB (minified + gzipped).

---

¹ **Оптимистичные обновления** — это паттерн, при котором интерфейс обновляется немедленно, предполагая успех операции, ещё до получения ответа от сервера. Если сервер возвращает ошибку, изменения автоматически откатываются.

**Пример:** Пользователь нажимает «Нравится» на посте. Вместо того чтобы ждать ответа сервера (200-500 мс), счётчик лайков увеличивается мгновенно. Если запрос к серверу завершается ошибкой, счётчик возвращается к предыдущему значению.

```
Без оптимистичных обновлений:
[Клик] → [Запрос к серверу] → [Ждём 300мс] → [Обновляем UI]
                                          ↑ Пользователь видит задержку

С оптимистичными обновлениями:
[Клик] → [Сразу обновляем UI] → [Запрос к серверу в фоне]
                ↑ Мгновенная реакция
                Если ошибка → откатываем изменения
```

Этот паттерн делает интерфейс более отзывчивым, но требует обработки отката при ошибках. TanStack Query предоставляет встроенную поддержку через `onMutate`, `onError` и `onSettled` в мутациях (см. раздел [Мутации](#мутации)).

---

## Какую проблему решает TanStack Query

### Проблема 1: Ручное управление состоянием загрузки

Без TanStack Query каждый компонент, загружающий данные, вынужден вручную управлять состояниями `loading`, `error`, `success`:

```jsx
// ❌ Традиционный подход: гонка состояний и бойлерплейт
function UserProfile({ id }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetch(`/api/users/${id}`)
      .then((res) => res.json())
      .then((data) => {
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      })
      .catch((err) => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });
    return () => {
      cancelled = true;
    };
  }, [id]);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <Profile user={user} />;
}
```

**Проблемы этого подхода:**

1. **Бойлерплейт.** Каждый компонент дублирует логику `loading`/`error`/`success`.
2. **Гонка состояний.** При быстрой смене `id` старый ответ может прийти позже нового.
3. **Нет кэширования.** При возврате на страницу данные загружаются заново.
4. **Нет повторных запросов.** Данные не обновляются автоматически.
5. **Нет дедупликации.** Два компонента, запрашивающие одни данные, делают два запроса.

**TanStack Query решает эти проблемы:**

```jsx
// ✅ TanStack Query: вся логика в двух параметрах
function UserProfile({ id }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ["user", id],
    queryFn: () => fetch(`/api/users/${id}`).then((res) => res.json()),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <Profile user={user} />;
}
```

| Проблема | Решение TanStack Query |
|---|---|
| Бойлерплейт | Один хук `useQuery` вместо `useState` + `useEffect` |
| Гонка состояний | Автоматическая отмена устаревших запросов |
| Нет кэширования | Встроенное кэширование с настраиваемым TTL |
| Нет повторных запросов | Автоматическое фоновое обновление |
| Нет дедупликации | Дедупликация по `queryKey` |

### Проблема 2: Кэширование данных

Без TanStack Query кэширование требует ручной реализации:

```jsx
// ❌ Ручное кэширование: сложно и подвержено ошибкам
const cache = new Map();

function useUser(id) {
  const [user, setUser] = useState(cache.get(id) || null);
  const [loading, setLoading] = useState(!cache.has(id));

  useEffect(() => {
    if (cache.has(id)) {
      setUser(cache.get(id));
      setLoading(false);
      // Но как обновить кэшированные данные?
      // Как инвалидировать кэш?
    }
    fetch(`/api/users/${id}`)
      .then((res) => res.json())
      .then((data) => {
        cache.set(id, data);
        setUser(data);
        setLoading(false);
      });
  }, [id]);

  return { user, loading };
}
```

TanStack Query предоставляет кэширование из коробки:

```jsx
// ✅ TanStack Query: кэширование автоматически
function useUser(id) {
  return useQuery({
    queryKey: ["user", id],
    queryFn: () => fetch(`/api/users/${id}`).then((res) => res.json()),
    staleTime: 5 * 60 * 1000, // Данные актуальны 5 минут
    gcTime: 10 * 60 * 1000, // Кэш хранится 10 минут
  });
}
```

### Проблема 3: Оптимистичные обновления

Без TanStack Query оптимистичные обновления требуют сложной ручной реализации:

```jsx
// ❌ Ручные оптимистичные обновления
function TodoItem({ todo }) {
  const [optimisticDone, setOptimisticDone] = useState(todo.done);
  const [saving, setSaving] = useState(false);

  const handleToggle = async () => {
    const prevDone = optimisticDone;
    setOptimisticDone(!prevDone); // Оптимистичное обновление
    setSaving(true);

    try {
      await api.patch(`/todos/${todo.id}`, { done: !prevDone });
    } catch {
      setOptimisticDone(prevDone); // Откат при ошибке
    } finally {
      setSaving(false);
    }
  };

  return (
    <li>
      <input
        type="checkbox"
        checked={optimisticDone}
        onChange={handleToggle}
        disabled={saving}
      />
      {todo.title}
    </li>
  );
}
```

TanStack Query предоставляет встроенные оптимистичные обновления через `onMutate`:

```jsx
// ✅ TanStack Query: оптимистичные обновления из коробки
function useToggleTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (todo) =>
      api.patch(`/todos/${todo.id}`, { done: !todo.done }),
    onMutate: async (todo) => {
      await queryClient.cancelQueries({ queryKey: ["todos"] });
      const previousTodos = queryClient.getQueryData(["todos"]);
      queryClient.setQueryData(["todos"], (old) =>
        old.map((t) =>
          t.id === todo.id ? { ...t, done: !t.done } : t
        )
      );
      return { previousTodos };
    },
    onError: (err, todo, context) => {
      queryClient.setQueryData(["todos"], context.previousTodos);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });
}
```

---

## Ключевые понятия

### Query (Запрос)

Query — это асинхронный источник данных, привязанный к уникальному ключу (`queryKey`). Query может находиться в одном из состояний:

- **`pending`** — данных ещё нет (первый запрос).
- **`fetching`** — запрос выполняется (данные могут быть в кэше).
- **`success`** — данные успешно загружены.
- **`error`** — запрос завершился с ошибкой.
- **`stale`** — данные устарели и нуждаются в обновлении.

```jsx
const { data, isLoading, isError, status } = useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
});
// status: "pending" | "error" | "success"
```

### Query Key (Ключ запроса)

`queryKey` — уникальный идентификатор запроса. TanStack Query использует его для кэширования, дедупликации и инвалидации.

```jsx
// Простой ключ
useQuery({ queryKey: ["users"], queryFn: fetchUsers });

// Ключ с параметрами
useQuery({ queryKey: ["user", id], queryFn: () => fetchUser(id) });

// Ключ с объектом
useQuery({
  queryKey: ["users", { status: "active", page: 1 }],
  queryFn: () => fetchUsers({ status: "active", page: 1 }),
});
```

> ⚠️ **Важно:** `queryKey` должен быть **стабильным** — не создавайте новые массивы/объекты при каждом рендере. Используйте `useMemo` или выносите константы.

### Query Function (Функция запроса)

`queryFn` — асинхронная функция, возвращающая данные. Должна либо вернуть данные, либо выбросить ошибку:

```jsx
// ✅ Возврат данных
useQuery({
  queryKey: ["users"],
  queryFn: async () => {
    const res = await fetch("/api/users");
    if (!res.ok) throw new Error("Failed to fetch");
    return res.json();
  },
});
```

### Stale Time и GC Time

Два ключевых параметра управления кэшем:

```jsx
useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
  staleTime: 5 * 60 * 1000,    // 5 минут: данные считаются «свежими»
  gcTime: 10 * 60 * 1000,      // 10 минут: кэш хранится после размонтирования
});
```

- **`staleTime`** — время, в течение которого данные считаются актуальными. После истечения TanStack Query запускает фоновое обновление.
- **`gcTime`** (ранее `cacheTime`) — время хранения кэша после того, как все компоненты отписались. По умолчанию 5 минут.

```
staleTime:
[────────── свежий ──────────|──── устарел → фоновое обновление ────]
gcTime:
[────────────── кэш хранится в памяти ────────────────────────────────|удалён]
```

### Query Client

`QueryClient` — центральный объект, управляющий кэшем запросов. Создаётся один раз и передаётся через `QueryClientProvider`:

```jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      gcTime: 10 * 60 * 1000,
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyApp />
    </QueryClientProvider>
  );
}
```

### Где хранится кэш

Кэш TanStack Query по умолчанию хранится **в оперативной памяти (RAM)** браузера. Это означает:

| Событие | Что происходит с кэшем |
|---|---|
| Навигация между страницами (SPA) | ✅ Кэш сохраняется |
| Обновление страницы (F5) | ❌ Кэш теряется |
| Закрытие вкладки | ❌ Кэш теряется |
| Переход по ссылке и возврат назад | ✅ Кэш сохраняется (если вкладка не закрывалась) |

**Почему в памяти, а не в localStorage?**

1. **Производительность.** Чтение из памяти происходит мгновенно, тогда как localStorage требует синхронного чтения с диска и десериализации JSON.
2. **Размер данных.** localStorage ограничен ~5-10 MB, тогда как RAM может хранить значительно больше.
3. **Свежесть данных.** Серверное состояние может измениться в любой момент, поэтому при каждой загрузке страницы лучше получать актуальные данные.

**Как сохранить кэш между перезагрузками?**

Для персистентного кэша используйте `persistQueryClient` с `localStorage` или `sessionStorage`:

```jsx
import { QueryClient } from "@tanstack/react-query";
import { persistQueryClient } from "@tanstack/react-query-persist-client";
import { createSyncStoragePersister } from "@tanstack/query-sync-storage-persister";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 60 * 24, // 24 часа
    },
  },
});

const localStoragePersister = createSyncStoragePersister({
  storage: window.localStorage,
});

// Или sessionStorage (очищается при закрытии вкладки)
const sessionStoragePersister = createSyncStoragePersister({
  storage: window.sessionStorage,
});

persistQueryClient({
  queryClient,
  persister: localStoragePersister,
  maxAge: 1000 * 60 * 60 * 12, // 12 часов — максимальный возраст кэша
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyApp />
    </QueryClientProvider>
  );
}
```

> ⚠️ **Важно:** При использовании персистентного кэша данные могут быть устаревшими. Убедитесь, что `staleTime` настроен appropriately, чтобы TanStack Query обновлял данные в фоне после восстановления из localStorage.

### Инвалидация кэша

Инвалидация помечает запросы как устаревшие, заставляя TanStack Query выполнить повторный запрос:

```jsx
const queryClient = useQueryClient();

// Инвалидировать все запросы с ключом "users"
queryClient.invalidateQueries({ queryKey: ["users"] });

// Инвалидировать все запросы, начинающиеся с "users"
queryClient.invalidateQueries({ queryKey: ["users"], exact: false });

// Инвалидировать только точное совпадение
queryClient.invalidateQueries({
  queryKey: ["users", { status: "active" }],
  exact: true,
});
```

---

## Базовое использование

### Установка и настройка

```bash
npm install @tanstack/react-query
```

```jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyApp />
    </QueryClientProvider>
  );
}
```

### Простой запрос

```jsx
import { useQuery } from "@tanstack/react-query";

function UserList() {
  const {
    data: users,
    isLoading,
    isError,
    error,
    refetch,
  } = useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then((res) => res.json()),
  });

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage error={error} />;

  return (
    <div>
      <button onClick={() => refetch()}>Refresh</button>
      <ul>
        {users.map((user) => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Запрос с параметрами

```jsx
function UserProfile({ userId }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetch(`/api/users/${userId}`).then((res) => res.json()),
    enabled: !!userId, // Запрос выполняется только если userId задан
  });

  if (isLoading) return <Spinner />;
  return <Profile user={user} />;
}
```

### Зависимые запросы

Когда один запрос зависит от результата другого:

```jsx
function UserPosts({ userId }) {
  // 1. Сначала загружаем пользователя
  const { data: user } = useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
  });

  // 2. Затем загружаем посты пользователя
  const { data: posts } = useQuery({
    queryKey: ["posts", user?.id],
    queryFn: () => fetchPosts(user.id),
    enabled: !!user, // Запрос выполняется только после загрузки user
  });

  return (
    <div>
      <h1>{user?.name}</h1>
      {posts?.map((post) => <Post key={post.id} post={post} />)}
    </div>
  );
}
```

---

## Сценарии применения

### 1. Загрузка списков с фильтрацией

```jsx
function ProductList() {
  const [filters, setFilters] = useState({
    category: "all",
    sort: "name",
    search: "",
  });

  const { data: products, isLoading } = useQuery({
    queryKey: ["products", filters],
    queryFn: () => fetchProducts(filters),
    keepPreviousData: true, // Показывать предыдущие данные при изменении фильтров
  });

  return (
    <div>
      <Filters filters={filters} onChange={setFilters} />
      {isLoading ? <Spinner /> : <ProductGrid products={products} />}
    </div>
  );
}
```

### 2. Пагинация

```jsx
function PaginatedList() {
  const [page, setPage] = useState(1);

  const { data, isLoading } = useQuery({
    queryKey: ["items", page],
    queryFn: () => fetchItems(page),
    keepPreviousData: true,
  });

  return (
    <div>
      {data.items.map((item) => <Item key={item.id} item={item} />)}
      <button
        onClick={() => setPage((p) => p - 1)}
        disabled={page === 1}
      >
        Previous
      </button>
      <button
        onClick={() => setPage((p) => p + 1)}
        disabled={!data.hasNextPage}
      >
        Next
      </button>
    </div>
  );
}
```

### 3. Бесконечная прокрутка

```jsx
import { useInfiniteQuery } from "@tanstack/react-query";

function InfiniteList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ["items"],
    queryFn: ({ pageParam = 1 }) => fetchItems(pageParam),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    initialPageParam: 1,
  });

  return (
    <div>
      {data.pages.map((page) =>
        page.items.map((item) => <Item key={item.id} item={item} />)
      )}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? "Loading..." : "Load More"}
      </button>
    </div>
  );
}
```

### 4. Загрузка с повторными попытками

```jsx
const { data } = useQuery({
  queryKey: ["unstable-data"],
  queryFn: fetchUnstableData,
  retry: 3,              // 3 попытки
  retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000), // Экпоненциальная задержка
  retryOnMount: true,    // Повторить при монтировании после ошибки
});
```

### 5. Параллельные запросы

```jsx
function Dashboard({ userId }) {
  // Все запросы выполняются параллельно
  const { data: user } = useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
  });

  const { data: posts } = useQuery({
    queryKey: ["posts", userId],
    queryFn: () => fetchPosts(userId),
  });

  const { data: stats } = useQuery({
    queryKey: ["stats", userId],
    queryFn: () => fetchStats(userId),
  });

  return (
    <div>
      <UserHeader user={user} />
      <StatsPanel stats={stats} />
      <PostList posts={posts} />
    </div>
  );
}
```

### 6. Предзагрузка данных (Prefetching)

```jsx
function ProductList() {
  const queryClient = useQueryClient();

  const handleHover = (productId) => {
    queryClient.prefetchQuery({
      queryKey: ["product", productId],
      queryFn: () => fetchProduct(productId),
    });
  };

  return (
    <ul>
      {products.map((product) => (
        <li
          key={product.id}
          onMouseEnter={() => handleHover(product.id)}
        >
          <Link to={`/products/${product.id}`}>
            {product.name}
          </Link>
        </li>
      ))}
    </ul>
  );
}
```

### 7. Оптимистичные обновления

```jsx
function TodoList() {
  const queryClient = useQueryClient();

  const { data: todos } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  });

  const toggleMutation = useMutation({
    mutationFn: (todo) =>
      api.patch(`/todos/${todo.id}`, { done: !todo.done }),

    onMutate: async (todo) => {
      await queryClient.cancelQueries({ queryKey: ["todos"] });
      const previousTodos = queryClient.getQueryData(["todos"]);

      queryClient.setQueryData(["todos"], (old) =>
        old.map((t) =>
          t.id === todo.id ? { ...t, done: !t.done } : t
        )
      );

      return { previousTodos };
    },

    onError: (err, todo, context) => {
      queryClient.setQueryData(["todos"], context.previousTodos);
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => toggleMutation.mutate(todo)}
          />
          {todo.title}
        </li>
      ))}
    </ul>
  );
}
```

---

## Мутации

Мутации используются для создания, обновления и удаления данных.

### Базовая мутация

```jsx
import { useMutation } from "@tanstack/react-query";

function CreatePost() {
  const mutation = useMutation({
    mutationFn: (newPost) =>
      fetch("/api/posts", {
        method: "POST",
        body: JSON.stringify(newPost),
      }).then((res) => res.json()),
  });

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const formData = new FormData(e.target);
        mutation.mutate({
          title: formData.get("title"),
          body: formData.get("body"),
        });
      }}
    >
      <input name="title" />
      <textarea name="body" />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? "Creating..." : "Create"}
      </button>
      {mutation.isError && <p>Error: {mutation.error.message}</p>}
      {mutation.isSuccess && <p>Post created!</p>}
    </form>
  );
}
```

### Мутация с инвалидацией кэша

```jsx
function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newPost) => api.post("/posts", newPost),
    onSuccess: () => {
      // Инвалидировать список постов после создания
      queryClient.invalidateQueries({ queryKey: ["posts"] });
    },
  });
}
```

### Мутация с обновлением кэша

```jsx
function useUpdateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (updatedTodo) =>
      api.patch(`/todos/${updatedTodo.id}`, updatedTodo),

    onMutate: async (updatedTodo) => {
      await queryClient.cancelQueries({
        queryKey: ["todos", updatedTodo.id],
      });

      const previousTodo = queryClient.getQueryData([
        "todos",
        updatedTodo.id,
      ]);

      queryClient.setQueryData(
        ["todos", updatedTodo.id],
        updatedTodo
      );

      return { previousTodo };
    },

    onError: (err, updatedTodo, context) => {
      queryClient.setQueryData(
        ["todos", updatedTodo.id],
        context.previousTodo
      );
    },

    onSettled: (updatedTodo) => {
      queryClient.invalidateQueries({
        queryKey: ["todos", updatedTodo.id],
      });
    },
  });
}
```

---

## Продвинутые возможности

### Query Invalidation

Инвалидация кэша — механизм обновления данных после мутаций:

```jsx
const queryClient = useQueryClient();

// Инвалидировать все запросы "todos"
queryClient.invalidateQueries({ queryKey: ["todos"] });

// Инвалидировать только точное совпадение
queryClient.invalidateQueries({
  queryKey: ["todos", { status: "active" }],
  exact: true,
});

// Инвалидировать все запросы, начинающиеся с "todos"
queryClient.invalidateQueries({
  predicate: (query) => query.queryKey[0] === "todos",
});
```

### Scroll Restoration

TanStack Query помогает восстановить позицию прокрутки при навигации:

```jsx
import { useScrollRestoration } from "@tanstack/react-query";

function ProductList() {
  const { data: products } = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });

  useScrollRestoration(["products"]);

  return (
    <div>
      {products.map((product) => <ProductCard key={product.id} />)}
    </div>
  );
}
```

### Offline Support

TanStack Query поддерживает работу офлайн через `persistQueryClient`:

```jsx
import { persistQueryClient } from "@tanstack/react-query-persist-client";
import { createSyncStoragePersister } from "@tanstack/query-sync-storage-persister";

const queryClient = new QueryClient();

const persister = createSyncStoragePersister({
  storage: window.localStorage,
});

persistQueryClient({
  queryClient,
  persister,
  maxAge: 1000 * 60 * 60 * 24, // 24 часа
});
```

### DevTools

TanStack Query DevTools — мощный инструмент для отладки:

```jsx
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

DevTools позволяют:
- Просматривать все активные и неактивные запросы
- Инвалидировать запросы вручную
- Инспектировать кэшированные данные
- Отслеживать статусы запросов

### TypeScript

TanStack Query отлично работает с TypeScript:

```tsx
interface User {
  id: number;
  name: string;
  email: string;
}

interface UsersResponse {
  users: User[];
  total: number;
}

function UserList() {
  const { data, error } = useQuery<UsersResponse, Error>({
    queryKey: ["users"],
    queryFn: async (): Promise<UsersResponse> => {
      const res = await fetch("/api/users");
      if (!res.ok) throw new Error("Failed to fetch");
      return res.json();
    },
  });

  // data: UsersResponse | undefined
  // error: Error | null
}
```

### Custom Hooks

Инкапсуляция логики загрузки данных в пользовательские хуки:

```jsx
// hooks/useUsers.js
export function useUsers(filters = {}) {
  return useQuery({
    queryKey: ["users", filters],
    queryFn: () => fetchUsers(filters),
    staleTime: 5 * 60 * 1000,
  });
}

// hooks/useUser.js
export function useUser(id) {
  return useQuery({
    queryKey: ["user", id],
    queryFn: () => fetchUser(id),
    enabled: !!id,
  });
}

// hooks/useCreateUser.js
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newUser) => api.post("/users", newUser),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] });
    },
  });
}

// Использование в компонентах
function UserList() {
  const { data: users, isLoading } = useUsers({ status: "active" });
  // ...
}
```

---

## TanStack Query vs Zustand

TanStack Query и [Zustand](./zustand.md) решают разные задачи. Понимание различий критично для правильного выбора.

### Когда использовать TanStack Query

TanStack Query подходит для **серверного состояния** (server state):

| Сценарий | Пример |
|---|---|
| Загрузка данных с API | GET-запросы, списки, детали |
| Кэширование | Избежание повторных запросов |
| Мутации | POST, PUT, DELETE с оптимистичными обновлениями |
| Пагинация и бесконечная прокрутка | Списки с курсором или offset |
| Фоновое обновление | Автоматическое обновление данных |
| Инвалидация кэша | Обновление данных после мутаций |
| Повторные попытки | Автоматический retry при ошибках |

```jsx
// ✅ TanStack Query для серверных данных
function UserList() {
  const { data: users, isLoading, error } = useQuery({
    queryKey: ["users"],
    queryFn: () => api.get("/users").then((res) => res.data),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Когда использовать Zustand

Zustand подходит для **клиентского состояния** (UI state):

| Сценарий | Пример |
|---|---|
| Глобальное UI-состояние | Модальные окна, боковые панели, уведомления |
| Состояние форм | Значения полей, валидация, отправка |
| Состояние корзины | Товары, количество, общая сумма |
| Аутентификация | Текущий пользователь, токен |
| Настройки приложения | Тема, язык, размер шрифта |
| Состояние, не связанное с сервером | Счётчики, таймеры, локальные данные |

```jsx
// ✅ Zustand для UI-состояния
const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

### Сравнительная таблица

| Характеристика | TanStack Query | Zustand |
|---|---|---|
| **Основное назначение** | Серверное состояние | Клиентское состояние |
| **Кэширование** | ✅ Встроенное | ❌ Нет встроенного |
| **Повторные запросы** | ✅ Автоматические | ❌ Ручная реализация |
| **Инвалидация кэша** | ✅ Автоматическая | ❌ Ручная реализация |
| **Оптимистичные обновления** | ✅ Встроенные | ❌ Ручная реализация |
| **Пагинация** | ✅ Встроенная | ❌ Ручная реализация |
| **Фоновое обновление** | ✅ Автоматическое | ❌ Ручная реализация |
| **Размер** | ~15 KB | ~1 KB |
| **Сложность** | Средняя | Минимальная |
| **Интеграция с React** | Хуки + QueryClient | Хуки |

### Рекомендуемый паттерн

Современный подход 2026 года: **использовать оба инструмента вместе**:

```jsx
// Zustand для клиентского состояния
const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));

// TanStack Query для серверных данных
function UserList() {
  const { data: users, isLoading } = useQuery({
    queryKey: ["users"],
    queryFn: fetchUsers,
  });

  const sidebarOpen = useUIStore((state) => state.sidebarOpen);

  return (
    <div>
      <button onClick={() => useUIStore.getState().toggleSidebar()}>
        Toggle Sidebar
      </button>
      {sidebarOpen && <Sidebar />}
      {isLoading ? <Spinner /> : <UserGrid users={users} />}
    </div>
  );
}
```

### Когда можно обойтись одним

**Только TanStack Query:**
- Приложения, где всё состояние связано с сервером
- Нет глобального UI-состояния (кроме локального состояния компонентов)
- Простые CRUD-приложения

**Только Zustand:**
- Простые приложения без серверного взаимодействия
- Статические сайты с минимальной динамикой
- Библиотеки компонентов без загрузки данных

**Оба инструмента:**
- Большинство современных приложений
- Сложные UI с модальными окнами, боковыми панелями, уведомлениями
- Приложения с аутентификацией и пользовательскими настройками

> 💡 **Правило:** Если данные приходят с сервера — используйте TanStack Query. Если данные создаются и живут только в браузере — используйте Zustand.

---

## Миграция с RTK Query

RTK Query — это слой для загрузки данных, встроенный в Redux Toolkit. TanStack Query может заменить RTK Query, предоставляя более гибкий и мощный API.

### RTK Query

```jsx
// api.js
const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  endpoints: (builder) => ({
    getUsers: builder.query({
      query: () => "/users",
    }),
    getUser: builder.query({
      query: (id) => `/users/${id}`,
    }),
    createUser: builder.mutation({
      query: (body) => ({ url: "/users", method: "POST", body }),
    }),
  }),
});

export const { useGetUsersQuery, useGetUserQuery, useCreateUserMutation } = api;

// store.js
export const store = configureStore({
  reducer: { [api.reducerPath]: api.reducer },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(api.middleware),
});

// App.jsx
<Provider store={store}>
  <App />
</Provider>

// Component.jsx
function UserList() {
  const { data: users, isLoading } = useGetUsersQuery();
  // ...
}
```

### TanStack Query (эквивалент)

```jsx
// hooks/useUsers.js
export function useUsers() {
  return useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then((res) => res.json()),
  });
}

// hooks/useUser.js
export function useUser(id) {
  return useQuery({
    queryKey: ["user", id],
    queryFn: () => fetch(`/api/users/${id}`).then((res) => res.json()),
    enabled: !!id,
  });
}

// hooks/useCreateUser.js
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newUser) =>
      fetch("/api/users", {
        method: "POST",
        body: JSON.stringify(newUser),
      }).then((res) => res.json()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] });
    },
  });
}

// App.jsx
<QueryClientProvider client={queryClient}>
  <App />
</QueryClientProvider>

// Component.jsx
function UserList() {
  const { data: users, isLoading } = useUsers();
  // ...
}
```

Преимущества миграции на TanStack Query:
- Нет зависимости от Redux
- Более гибкий API
- Лучшая поддержка TypeScript
- Меньше бойлерплейта
- Можно использовать вместе с Zustand для клиентского состояния

---

## Лучшие практики

### 1. Используйте стабильные queryKey

Не создавайте новые массивы/объекты при каждом рендере:

```jsx
// ❌ Плохо: новый массив при каждом рендере
function Component({ id }) {
  const { data } = useQuery({
    queryKey: ["user", id],
    queryFn: () => fetchUser(id),
  });
}

// ✅ Хорошо: вынесите константы или используйте useMemo
const USER_KEY = "user";
function Component({ id }) {
  const { data } = useQuery({
    queryKey: [USER_KEY, id],
    queryFn: () => fetchUser(id),
  });
}
```

### 2. Используйте пользовательские хуки

Инкапсулируйте логику загрузки данных в переиспользуемые хуки:

```jsx
// hooks/useUsers.js
export function useUsers(filters = {}) {
  return useQuery({
    queryKey: ["users", filters],
    queryFn: () => fetchUsers(filters),
    staleTime: 5 * 60 * 1000,
  });
}

// Использование
function UserList() {
  const { data: users, isLoading } = useUsers({ status: "active" });
}
```

### 3. Настройте глобальные параметры

Избегайте дублирования параметров в каждом запросе:

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,       // 5 минут
      gcTime: 10 * 60 * 1000,          // 10 минут
      retry: 1,                        // 1 повторная попытка
      refetchOnWindowFocus: false,     // Не обновлять при фокусе окна
      refetchOnReconnect: true,        // Обновлять при восстановлении сети
    },
  },
});
```

### 4. Используйте `enabled` для условных запросов

```jsx
// ✅ Запрос выполняется только если userId задан
const { data: user } = useQuery({
  queryKey: ["user", userId],
  queryFn: () => fetchUser(userId),
  enabled: !!userId,
});
```

### 5. Инвалидируйте кэш после мутаций

```jsx
const mutation = useMutation({
  mutationFn: createUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["users"] });
  },
});
```

### 6. Используйте `keepPreviousData` для пагинации

```jsx
const { data } = useQuery({
  queryKey: ["items", page],
  queryFn: () => fetchItems(page),
  keepPreviousData: true, // Показывать предыдущие данные при изменении page
});
```

### 7. Используйте DevTools в разработке

```jsx
<QueryClientProvider client={queryClient}>
  <App />
  <ReactQueryDevtools initialIsOpen={false} />
</QueryClientProvider>
```

---

## Антипаттерны

### 1. Использование TanStack Query для клиентского состояния

```jsx
// ❌ Плохо: хранение UI-состояния в TanStack Query
const { data: sidebarOpen } = useQuery({
  queryKey: ["sidebar"],
  queryFn: () => Promise.resolve(false),
});

// ✅ Хорошо: используйте Zustand для UI-состояния
const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

### 2. Создание queryKey в render

```jsx
// ❌ Плохо: новый объект при каждом рендере
function Component({ filters }) {
  const { data } = useQuery({
    queryKey: ["items", { ...filters }], // Новый объект
    queryFn: () => fetchItems(filters),
  });
}

// ✅ Хорошо: стабильная ссылка
function Component({ filters }) {
  const queryKey = useMemo(() => ["items", filters], [filters]);
  const { data } = useQuery({
    queryKey,
    queryFn: () => fetchItems(filters),
  });
}
```

### 3. Отсутствие инвалидации кэша

```jsx
// ❌ Плохо: данные не обновляются после мутации
const mutation = useMutation({
  mutationFn: createUser,
  // Нет onSuccess → кэш не инвалидируется
});

// ✅ Хорошо: инвалидация после мутации
const mutation = useMutation({
  mutationFn: createUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["users"] });
  },
});
```

### 4. Использование useEffect для загрузки данных

```jsx
// ❌ Плохо: ручная загрузка данных
function Component() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchData().then((data) => {
      setData(data);
      setLoading(false);
    });
  }, []);

  return <div>{data?.name}</div>;
}

// ✅ Хорошо: используйте useQuery
function Component() {
  const { data, isLoading } = useQuery({
    queryKey: ["data"],
    queryFn: fetchData,
  });

  return <div>{data?.name}</div>;
}
```

### 5. Игнорирование error state

```jsx
// ❌ Плохо: ошибка не обрабатывается
function Component() {
  const { data } = useQuery({
    queryKey: ["data"],
    queryFn: fetchData,
  });

  return <div>{data?.name}</div>; // data может быть undefined
}

// ✅ Хорошо: обработка ошибки
function Component() {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ["data"],
    queryFn: fetchData,
  });

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage error={error} />;
  return <div>{data.name}</div>;
}
```

### 6. Использование TanStack Query без QueryClientProvider

```jsx
// ❌ Плохо: QueryClientProvider отсутствует
function App() {
  return <MyComponent />; // useQuery не будет работать
}

// ✅ Хорошо: QueryClientProvider оборачивает приложение
const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyComponent />
    </QueryClientProvider>
  );
}
```

---

## Итог

TanStack Query — это **мощная и гибкая библиотека** для управления серверным состоянием в JavaScript-приложениях. Она решает проблемы ручного управления загрузкой данных, кэшированием, мутациями и оптимистичными обновлениями, предоставляя декларативный API на основе хуков.

**Ключевые преимущества:**
- Автоматическое кэширование и дедупликация запросов
- Фоновое обновление данных
- Встроенная поддержка пагинации и бесконечной прокрутки
- Оптимистичные обновления из коробки
- Инвалидация кэша после мутаций
- Отличная поддержка TypeScript
- DevTools для отладки

**Когда использовать:**
- Загрузка данных с API
- Кэширование и инвалидация
- Мутации с оптимистичными обновлениями
- Пагинация и бесконечная прокрутка
- Фоновое обновление данных

**Когда использовать Zustand:**
- Глобальное UI-состояние (модальные окна, боковые панели)
- Состояние форм
- Корзина, аутентификация, настройки
- Любое состояние, не связанное с сервером

**Рекомендуемый паттерн:** Используйте TanStack Query для серверного состояния и [Zustand](./zustand.md) для клиентского состояния. Вместе они заменяют Redux в большинстве современных приложений.
