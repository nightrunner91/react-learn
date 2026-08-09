# Типизация кастомных хуков и async-паттерны в React

Кастомные хуки — основной механизм переиспользования логики в React. TypeScript превращает их из удобного инструмента в мощный контракт: вы точно знаете, что хук принимает, что возвращает, и как ведёт себя при разных входных данных. Async-паттерны добавляют свою сложность: AbortController, типизация ответов API, обработка ошибок в эффектах. В этой статье разберём оба аспекта — от базовых возвращаемых типов до продвинутых паттернов вроде fetch-машины с discriminated unions.

---

## Содержание

1. [Типизация возвращаемых значений](#типизация-возвращаемых-значений)
2. [Дженерики в кастомных хуках](#дженерики-в-кастомных-хуках)
3. [Перегрузки хуков](#перегрузки-хуков)
4. [Практические паттерны кастомных хуков](#практические-паттерны-кастомных-хуков)
5. [Async-паттерны: Promise в компонентах](#async-паттерны-promise-в-компонентах)
6. [AbortController и отмена запросов](#abortcontroller-и-отмена-запросов)
7. [Типизация fetch и API-ответов](#типизация-fetch-и-api-ответов)
8. [Обработка ошибок с типами](#обработка-ошибок-с-типами)
9. [Fetch-машина с discriminated unions](#fetch-машина-с-discriminated-unions)
10. [Vue vs React: шпаргалка по хукам и async](#vue-vs-react-шпаргалка-по-хукам-и-async)
11. [Типичные ошибки](#типичные-ошибки)

---

## Типизация возвращаемых значений

Кастомные хуки возвращают данные через кортежи (как `useState`) или объекты. Выбор влияет на удобство использования и типобезопасность.

> **Vue-аналог:** Vue Composables обычно возвращают объект с именованными полями: `return { data, error, isLoading }`. В React оба подхода равноправны, но кортежи — для простых случаев (один-два значения), объекты — для сложных (3+ поля).

### Кортежи — для простых случаев

```tsx
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// Использование
const [isOpen, toggleOpen] = useToggle(false);
```

Тип возвращаемого значения явно указан как `[boolean, () => void]` — кортеж фиксированной длины с конкретными типами.

### Объекты — для сложных случаев

```tsx
interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

function useCounter(initial = 0): UseCounterReturn {
  const [count, setCount] = useState(initial);

  const increment = useCallback(() => setCount(c => c + 1), []);
  const decrement = useCallback(() => setCount(c => c - 1), []);
  const reset = useCallback(() => setCount(initial), [initial]);

  return { count, increment, decrement, reset };
}

// Использование — деструктуризация с именованными полями
const { count, increment, reset } = useCounter(10);
```

Преимущества объекта:
- Именованные поля — не нужно помнить порядок.
- Можно возвращать только часть: `const { count } = useCounter()`.
- Легко расширять без breaking changes.

### Когда что использовать

| Подход | Когда использовать | Пример |
|---|---|---|
| Кортеж | 1-2 значения, порядок важен | `useState`, `useToggle` |
| Объект | 3+ поля, нужна гибкость | `useCounter`, `useFetch` |

---

## Дженерики в кастомных хуках

Дженерики позволяют создавать хуки, работающие с любыми типами данных, сохраняя типобезопасность.

> **Vue-аналог:** `function useLocalStorage<T>(key: string, initialValue: T): Ref<T>` — синтаксис идентичен. В React — `function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void]`.

### Базовый пример

```tsx
function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T) => {
    setStoredValue(value);
    localStorage.setItem(key, JSON.stringify(value));
  };

  return [storedValue, setValue];
}

// Использование — тип выводится автоматически
const [theme, setTheme] = useLocalStorage("theme", "light"); // string
const [user, setUser] = useLocalStorage<User>("user", null); // User | null (если указать явно)
```

### Дженерики с ограничениями

```tsx
interface HasId {
  id: string;
}

function useItemsById<T extends HasId>(items: T[]): Map<string, T> {
  return useMemo(() => {
    const map = new Map<string, T>();
    items.forEach(item => map.set(item.id, item));
    return map;
  }, [items]);
}

// T автоматически ограничивается типами с полем id
const userMap = useItemsById([
  { id: "1", name: "Alice" },
  { id: "2", name: "Bob" },
]);
// userMap: Map<string, { id: string; name: string }>
```

### Дженерики с несколькими параметрами

```tsx
function useAsync<TData, TError = Error>(
  asyncFn: () => Promise<TData>
): {
  data: TData | null;
  error: TError | null;
  isLoading: boolean;
  execute: () => Promise<void>;
} {
  const [data, setData] = useState<TData | null>(null);
  const [error, setError] = useState<TError | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const execute = useCallback(async () => {
    setIsLoading(true);
    setError(null);
    try {
      const result = await asyncFn();
      setData(result);
    } catch (err) {
      setError(err as TError);
    } finally {
      setIsLoading(false);
    }
  }, [asyncFn]);

  return { data, error, isLoading, execute };
}

// Использование
const { data, error, isLoading, execute } = useAsync<User, ApiError>(
  () => fetchUser(userId)
);
```

---

## Перегрузки хуков

Перегрузки позволяют хуку возвращать разные типы в зависимости от входных параметров.

### Пример: условный возврат

```tsx
// Перегрузки
function useMediaQuery(query: string): boolean;
function useMediaQuery(query: string, defaultValue: boolean): boolean;
function useMediaQuery(query: string, defaultValue?: boolean): boolean {
  const [matches, setMatches] = useState(() => {
    if (typeof window === "undefined") return defaultValue ?? false;
    return window.matchMedia(query).matches;
  });

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    
    mediaQuery.addEventListener("change", handler);
    return () => mediaQuery.removeEventListener("change", handler);
  }, [query]);

  return matches;
}

// Использование
const isMobile = useMediaQuery("(max-width: 768px)"); // boolean
const isDarkMode = useMediaQuery("(prefers-color-scheme: dark)", false); // boolean с дефолтом
```

### Пример: возврат кортежа или объекта

```tsx
// Перегрузки
function useCounter(initial: number): { count: number; set: (value: number) => void };
function useCounter(initial: number, asTuple: true): [number, (value: number) => void];
function useCounter(initial: number, asTuple?: boolean) {
  const [count, setCount] = useState(initial);
  const set = useCallback((value: number) => setCount(value), []);

  if (asTuple) {
    return [count, set] as const;
  }
  return { count, set };
}

// Использование
const { count, set } = useCounter(0); // объект
const [count, set] = useCounter(0, true); // кортеж
```

---

## Практические паттерны кастомных хуков

### useDebounce

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Использование
const [query, setQuery] = useState("");
const debouncedQuery = useDebounce(query, 300);

useEffect(() => {
  if (debouncedQuery) {
    search(debouncedQuery);
  }
}, [debouncedQuery]);
```

### useOnClickOutside

```tsx
function useOnClickOutside<T extends HTMLElement>(
  ref: React.RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
): void {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      if (!ref.current || ref.current.contains(event.target as Node)) {
        return;
      }
      handler(event);
    };

    document.addEventListener("mousedown", listener);
    document.addEventListener("touchstart", listener);

    return () => {
      document.removeEventListener("mousedown", listener);
      document.removeEventListener("touchstart", listener);
    };
  }, [ref, handler]);
}

// Использование
function Dropdown() {
  const ref = useRef<HTMLDivElement>(null);
  const [isOpen, setIsOpen] = useState(false);

  useOnClickOutside(ref, () => setIsOpen(false));

  return (
    <div ref={ref}>
      <button onClick={() => setIsOpen(true)}>Open</button>
      {isOpen && <div className="dropdown">...</div>}
    </div>
  );
}
```

### usePrevious

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// Использование
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}, Previous: {prevCount}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

---

## Async-паттерны: Promise в компонентах

### Базовый паттерн: useEffect + async

```tsx
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let isMounted = true;

    const loadUser = async () => {
      try {
        setIsLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) throw new Error("Failed to load user");
        
        const data = await response.json();
        if (isMounted) {
          setUser(data);
          setError(null);
        }
      } catch (err) {
        if (isMounted) {
          setError(err instanceof Error ? err : new Error("Unknown error"));
        }
      } finally {
        if (isMounted) {
          setIsLoading(false);
        }
      }
    };

    loadUser();

    return () => {
      isMounted = false;
    };
  }, [userId]);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!user) return null;

  return <div>{user.name}</div>;
}
```

Ключевой момент: флаг `isMounted` предотвращает обновление состояния после размонтирования компонента.

### Альтернатива: async-функция внутри useEffect

```tsx
useEffect(() => {
  const fetchData = async () => {
    const result = await api.getData();
    setData(result);
  };

  fetchData();
}, []);
```

Нельзя делать сам `useEffect` async — React ожидает, что эффект вернёт `void` или cleanup-функцию, а Promise нарушает этот контракт.

---

## AbortController и отмена запросов

AbortController позволяет отменить fetch-запрос при размонтировании компонента или изменении зависимостей.

> **Vue-аналог:** во Vue Composables часто используется `watch` с `onCleanup` или `watchEffect` с автоматической отменой. В React — `AbortController` + `useEffect` cleanup.

### Базовый пример

```tsx
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    const controller = new AbortController();

    const fetchResults = async () => {
      try {
        setIsLoading(true);
        const response = await fetch(
          `/api/search?q=${encodeURIComponent(query)}`,
          { signal: controller.signal }
        );

        if (!response.ok) throw new Error("Search failed");

        const data = await response.json();
        setResults(data);
      } catch (err) {
        if (err instanceof Error && err.name === "AbortError") {
          console.log("Request aborted");
          return;
        }
        console.error("Search error:", err);
      } finally {
        setIsLoading(false);
      }
    };

    fetchResults();

    return () => {
      controller.abort();
    };
  }, [query]);

  return (
    <div>
      {isLoading && <Spinner />}
      <ul>
        {results.map(result => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

`AbortError` — специальное исключение, которое выбрасывается при отмене запроса. Его нужно обрабатывать отдельно, чтобы не показывать пользователю ошибку.

### AbortController + debounce

```tsx
function useSearch(query: string, delay = 300) {
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    const controller = new AbortController();
    const timeoutId = setTimeout(async () => {
      try {
        setIsLoading(true);
        const response = await fetch(
          `/api/search?q=${encodeURIComponent(query)}`,
          { signal: controller.signal }
        );
        const data = await response.json();
        setResults(data);
      } catch (err) {
        if (!(err instanceof Error && err.name === "AbortError")) {
          console.error(err);
        }
      } finally {
        setIsLoading(false);
      }
    }, delay);

    return () => {
      clearTimeout(timeoutId);
      controller.abort();
    };
  }, [query, delay]);

  return { results, isLoading };
}
```

---

## Типизация fetch и API-ответов

### Базовая типизация

```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

async function fetchUser(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const data: User = await response.json();
  return data;
}
```

Проблема: `response.json()` возвращает `Promise<any>`, поэтому TypeScript не проверяет, что данные соответствуют типу `User`. Решение — каст с проверкой:

```tsx
function isUser(data: unknown): data is User {
  return (
    typeof data === "object" &&
    data !== null &&
    "id" in data &&
    "name" in data &&
    "email" in data
  );
}

async function fetchUserSafe(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();

  if (!isUser(data)) {
    throw new Error("Invalid user data");
  }

  return data;
}
```

### Типизация ошибок API

```tsx
class ApiError extends Error {
  constructor(
    public status: number,
    public statusText: string,
    public data?: unknown
  ) {
    super(`API Error: ${status} ${statusText}`);
    this.name = "ApiError";
  }
}

async function fetchUserWithApiError(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);

  if (!response.ok) {
    const errorData = await response.json().catch(() => null);
    throw new ApiError(response.status, response.statusText, errorData);
  }

  return response.json();
}

// Использование
try {
  const user = await fetchUserWithApiError(userId);
} catch (err) {
  if (err instanceof ApiError) {
    console.error(`API error ${err.status}:`, err.data);
  } else {
    console.error("Network error:", err);
  }
}
```

### Generic fetch-обёртка

```tsx
interface FetchOptions extends RequestInit {
  params?: Record<string, string>;
}

async function apiFetch<T>(
  endpoint: string,
  options: FetchOptions = {}
): Promise<T> {
  const { params, ...init } = options;

  const url = new URL(endpoint, window.location.origin);
  if (params) {
    Object.entries(params).forEach(([key, value]) => {
      url.searchParams.set(key, value);
    });
  }

  const response = await fetch(url, {
    ...init,
    headers: {
      "Content-Type": "application/json",
      ...init.headers,
    },
  });

  if (!response.ok) {
    throw new ApiError(response.status, response.statusText);
  }

  return response.json();
}

// Использование
const users = await apiFetch<User[]>("/api/users", {
  params: { role: "admin" },
});

const newUser = await apiFetch<User>("/api/users", {
  method: "POST",
  body: JSON.stringify({ name: "Alice", email: "alice@example.com" }),
});
```

---

## Обработка ошибок с типами

### Type guards для ошибок

```tsx
function isApiError(err: unknown): err is ApiError {
  return err instanceof ApiError;
}

function isNetworkError(err: unknown): boolean {
  return err instanceof TypeError && err.message.includes("fetch");
}

function handleError(err: unknown): void {
  if (isApiError(err)) {
    console.error(`API error ${err.status}:`, err.data);
    showToast(`Error: ${err.statusText}`);
  } else if (isNetworkError(err)) {
    console.error("Network error:", err);
    showToast("No internet connection");
  } else if (err instanceof Error) {
    console.error("Unexpected error:", err.message);
    showToast("Something went wrong");
  } else {
    console.error("Unknown error:", err);
    showToast("Unknown error");
  }
}
```

### Обработка ошибок в useEffect

```tsx
useEffect(() => {
  const controller = new AbortController();

  const loadData = async () => {
    try {
      const data = await fetchUser(userId, controller.signal);
      setUser(data);
    } catch (err) {
      if (err instanceof Error && err.name === "AbortError") {
        return;
      }
      handleError(err);
    }
  };

  loadData();

  return () => controller.abort();
}, [userId]);
```

---

## Fetch-машина с discriminated unions

Fetch-машина — паттерн моделирования состояний загрузки через discriminated unions. TypeScript гарантирует, что вы обработали все возможные состояния.

> **Vue-аналог:** во Vue Composables часто используется объект `{ data, error, isLoading }`, но это позволяет бессмысленные состояния вроде `{ data: null, error: null, isLoading: false }`. Discriminated unions исключают такие случаи.

### Определение состояний

```tsx
type FetchState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };
```

### Хук useFetch

```tsx
type FetchState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

type FetchAction<T> =
  | { type: "FETCH_START" }
  | { type: "FETCH_SUCCESS"; data: T }
  | { type: "FETCH_ERROR"; error: Error }
  | { type: "RESET" };

function fetchReducer<T>(state: FetchState<T>, action: FetchAction<T>): FetchState<T> {
  switch (action.type) {
    case "FETCH_START":
      return { status: "loading" };
    case "FETCH_SUCCESS":
      return { status: "success", data: action.data };
    case "FETCH_ERROR":
      return { status: "error", error: action.error };
    case "RESET":
      return { status: "idle" };
  }
}

function useFetch<T>(url: string) {
  const [state, dispatch] = useReducer(fetchReducer<T>, { status: "idle" });

  useEffect(() => {
    const controller = new AbortController();

    const fetchData = async () => {
      try {
        dispatch({ type: "FETCH_START" });
        const response = await fetch(url, { signal: controller.signal });
        
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }

        const data: T = await response.json();
        dispatch({ type: "FETCH_SUCCESS", data });
      } catch (err) {
        if (err instanceof Error && err.name === "AbortError") {
          return;
        }
        dispatch({
          type: "FETCH_ERROR",
          error: err instanceof Error ? err : new Error("Unknown error"),
        });
      }
    };

    fetchData();

    return () => controller.abort();
  }, [url]);

  return {
    ...state,
    refetch: () => dispatch({ type: "RESET" }),
  };
}
```

### Использование с exhaustiveness checking

```tsx
function UserProfile({ userId }: { userId: string }) {
  const { status, data, error, refetch } = useFetch<User>(`/api/users/${userId}`);

  switch (status) {
    case "idle":
    case "loading":
      return <Spinner />;
    case "success":
      return <div>{data.name}</div>;
    case "error":
      return (
        <div>
          <p>Error: {error.message}</p>
          <button onClick={refetch}>Retry</button>
        </div>
      );
    default: {
      const _exhaustive: never = status;
      return _exhaustive;
    }
  }
}
```

TypeScript проверяет, что все варианты обработаны. Если добавить новое состояние в `FetchState`, компилятор выдаст ошибку в `default`.

---

## Vue vs React: шпаргалка по хукам и async

| Концепция | Vue | React |
|---|---|---|
| Composable/Hook | `function useX(): { data, error }` | `function useX(): [data, setData]` или `{ data, error }` |
| Дженерики | `function useLocalStorage<T>(key: string, value: T): Ref<T>` | `function useLocalStorage<T>(key: string, value: T): [T, (v: T) => void]` |
| Async в эффекте | `watchEffect(async () => { ... })` | `useEffect(() => { const fn = async () => {}; fn(); }, [])` |
| Отмена запросов | `onCleanup(() => controller.abort())` в `watch` | `return () => controller.abort()` в `useEffect` |
| Обработка ошибок | `try/catch` + `ref<Error \| null>` | `try/catch` + `useState<Error \| null>` |
| Fetch-машина | Ручная через `ref` + `computed` | `useReducer` с discriminated unions |
| AbortController | `watch` с `onCleanup` | `useEffect` cleanup |

---

## Типичные ошибки

### 1. Async-функция как useEffect

```tsx
// ❌ Ошибка: useEffect не может быть async
useEffect(async () => {
  const data = await fetchData();
  setData(data);
}, []);

// ✅ Правильно: async-функция внутри useEffect
useEffect(() => {
  const loadData = async () => {
    const data = await fetchData();
    setData(data);
  };
  loadData();
}, []);
```

### 2. Отсутствие cleanup для async-операций

```tsx
// ❌ Утечка памяти: обновление состояния после размонтирования
useEffect(() => {
  fetchData().then(data => setData(data));
}, []);

// ✅ Cleanup: флаг isMounted или AbortController
useEffect(() => {
  let isMounted = true;
  fetchData().then(data => {
    if (isMounted) setData(data);
  });
  return () => {
    isMounted = false;
  };
}, []);
```

### 3. Неправильный тип для Error

```tsx
// ❌ catch (err: Error) — TypeScript не позволяет типизировать err в catch
try {
  await fetchData();
} catch (err: Error) { // Ошибка компиляции
  console.error(err.message);
}

// ✅ catch (err: unknown) + type guard
try {
  await fetchData();
} catch (err: unknown) {
  if (err instanceof Error) {
    console.error(err.message);
  }
}
```

### 4. Игнорирование AbortError

```tsx
// ❌ AbortError обрабатывается как обычная ошибка
try {
  const response = await fetch(url, { signal });
  const data = await response.json();
  setData(data);
} catch (err) {
  setError(err); // AbortError тоже попадает сюда
}

// ✅ Проверка на AbortError
try {
  const response = await fetch(url, { signal });
  const data = await response.json();
  setData(data);
} catch (err) {
  if (err instanceof Error && err.name === "AbortError") {
    return; // Запрос отменён — не ошибка
  }
  setError(err);
}
```

### 5. Отсутствие типизации для response.json()

```tsx
// ❌ response.json() возвращает Promise<any>
const data = await response.json();
setUser(data); // Нет проверки типа

// ✅ Каст с проверкой
const data = await response.json();
if (isUser(data)) {
  setUser(data);
} else {
  throw new Error("Invalid user data");
}
```

### 6. Неполный массив зависимостей в useEffect

```tsx
// ❌ userId используется, но не указан в зависимостях
useEffect(() => {
  const loadData = async () => {
    const data = await fetchData(userId);
    setData(data);
  };
  loadData();
}, []); // Ошибка: userId отсутствует

// ✅ Правильно
useEffect(() => {
  const loadData = async () => {
    const data = await fetchData(userId);
    setData(data);
  };
  loadData();
}, [userId]);
```

### 7. Возврат кортежа без явного типа

```tsx
// ❌ Тип выводится как (string | number)[]
function useCounter() {
  const [count, setCount] = useState(0);
  const increment = () => setCount(c => c + 1);
  return [count, increment]; // TypeScript не знает, что это кортеж
}

// ✅ Явный тип кортежа
function useCounter(): [number, () => void] {
  const [count, setCount] = useState(0);
  const increment = () => setCount(c => c + 1);
  return [count, increment];
}
```
