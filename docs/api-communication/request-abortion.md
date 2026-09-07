# Отмена HTTP-запросов: AbortController, Race Conditions и cleanup

## Содержание

1. [Зачем отменять запросы](#зачем-отменять-запросы)
2. [AbortController и AbortSignal](#abortcontroller-и-abortsignal)
3. [Отмена fetch](#отмена-fetch)
4. [Отмена axios](#отмена-axios)
5. [Отмена ky](#отмена-ky)
6. [Отмена в React через cleanup](#отмена-в-react-через-cleanup)
7. [Race conditions](#race-conditions)
8. [AbortController + debounce](#abortcontroller--debounce)
9. [Отмена в TanStack Query](#отмена-в-tanstack-query)
10. [Лучшие практики](#лучшие-практики)
11. [Антипаттерны](#антипаттерны)

---

## Зачем отменять запросы

Отмена запросов решает три проблемы:

1. **Экономия ресурсов.** Не нужно ждать ответ, который больше не актуален.
2. **Предотвращение состояний-гонок (race conditions).** Старый запрос не должен перезаписывать результат нового.
3. **Избежание утечек памяти.** Компонент размонтировался, а callback запроса всё ещё может обновить state.

Классический пример: пользователь быстро переключает вкладки или вводит текст в поиск. Без отмены каждый символ порождает запрос, и ответы приходят в непредсказуемом порядке.

---

## AbortController и AbortSignal

`AbortController` — стандартный браузерный API для отмены асинхронных операций. Он состоит из двух частей:

- **`AbortController`** — объект, у которого есть метод `abort()`.
- **`AbortSignal`** — сигнал, передаваемый в запрос. Когда вызывается `abort()`, сигнал уведомляет запрос о необходимости прерваться.

```js
const controller = new AbortController();
const signal = controller.signal;

fetch('/api/users', { signal });

// Отмена
controller.abort();
```

---

## Отмена fetch

```js
const controller = new AbortController();

try {
  const response = await fetch('/api/users', {
    signal: controller.signal,
  });
  const data = await response.json();
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('Запрос отменён');
  } else {
    console.error('Ошибка запроса:', error);
  }
}
```

### Timeout через AbortController

```js
function fetchWithTimeout(url, options = {}, timeout = 5000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeout);

  return fetch(url, {
    ...options,
    signal: controller.signal,
  }).finally(() => clearTimeout(id));
}
```

---

## Отмена axios

Современные версии axios поддерживают `AbortController`. Старый способ через `CancelToken` объявлен deprecated.

### Через AbortController

```js
const controller = new AbortController();

try {
  const { data } = await axios.get('/api/users', {
    signal: controller.signal,
  });
} catch (error) {
  if (axios.isCancel(error)) {
    console.log('Запрос отменён');
  }
}

// Отмена
controller.abort();
```

### Устаревший CancelToken (не используйте в новом коде)

```js
// ❌ Deprecated
const source = axios.CancelToken.source();
await axios.get('/api/users', { cancelToken: source.token });
source.cancel();
```

---

## Отмена ky

`ky` использует `fetch` под капотом, поэтому отмена работает через `AbortController`.

```js
const controller = new AbortController();

try {
  const data = await ky.get('/api/users', {
    signal: controller.signal,
  }).json();
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('Запрос отменён');
  }
}
```

---

## Отмена в React через cleanup

Самый частый сценарий — отмена запроса при размонтировании компонента или изменении зависимостей.

### Простой пример

```jsx
import { useEffect, useState } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => res.json())
      .then(setUser)
      .catch((error) => {
        if (error.name !== 'AbortError') {
          console.error(error);
        }
      });

    return () => {
      controller.abort();
    };
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

### С async/await

```jsx
useEffect(() => {
  const controller = new AbortController();

  async function loadUser() {
    try {
      const response = await fetch(`/api/users/${userId}`, {
        signal: controller.signal,
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
      if (error.name !== 'AbortError') {
        setError(error);
      }
    }
  }

  loadUser();

  return () => controller.abort();
}, [userId]);
```

---

## Race conditions

Race condition возникает, когда несколько запросов выполняются параллельно, и более старый приходит позже нового.

### Пример проблемы

```jsx
function Search() {
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then((res) => res.json())
      .then(setResults);
  }, [query]);
}
```

Если пользователь быстро введёт «a», затем «ab», ответ на «a» может прийти позже и перезаписать результаты для «ab».

### Решение через AbortController

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then((res) => res.json())
    .then(setResults)
    .catch((error) => {
      if (error.name !== 'AbortError') {
        console.error(error);
      }
    });

  return () => controller.abort();
}, [query]);
```

При смене `query` старый запрос отменяется автоматически.

### Альтернатива: флаг cancelled

```jsx
useEffect(() => {
  let cancelled = false;

  fetch(`/api/search?q=${query}`)
    .then((res) => res.json())
    .then((data) => {
      if (!cancelled) {
        setResults(data);
      }
    });

  return () => {
    cancelled = true;
  };
}, [query]);
```

Флаг предотвращает обновление state, но не отменяет сам запрос. `AbortController` предпочтительнее, так как реально освобождает ресурсы.

---

## AbortController + debounce

Для поиска часто комбинируют debounce и отмену запросов.

```jsx
import { useEffect, useState } from 'react';

function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    const controller = new AbortController();
    const timeoutId = setTimeout(async () => {
      try {
        const response = await fetch(`/api/search?q=${query}`, {
          signal: controller.signal,
        });
        const data = await response.json();
        setResults(data);
      } catch (error) {
        if (error.name !== 'AbortError') {
          console.error(error);
        }
      }
    }, 300);

    return () => {
      clearTimeout(timeoutId);
      controller.abort();
    };
  }, [query]);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Поиск..."
      />
      <ul>
        {results.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

Здесь:

- `debounce` откладывает запрос на 300 мс.
- `AbortController` отменяет предыдущий запрос при новом вводе.
- `clearTimeout` отменяет ещё не начатый запрос.

---

## Отмена в TanStack Query

TanStack Query отменяет запросы автоматически, если компонент размонтировался или `queryKey` изменился. Но важно, чтобы `queryFn` поддерживала сигнал.

```ts
const { data } = useQuery({
  queryKey: ['user', userId],
  queryFn: async ({ signal }) => {
    const response = await fetch(`/api/users/${userId}`, {
      signal,
    });

    if (!response.ok) {
      throw new Error('Failed to load user');
    }

    return response.json();
  },
});
```

TanStack Query передаёт `signal` в `queryFn`. Если запрос отменяется, fetch получит `AbortError`, который TanStack Query обработает корректно.

### Отмена mutation

```ts
const mutation = useMutation({
  mutationFn: async (newUser, { signal }) => {
    return ky.post('/api/users', {
      json: newUser,
      signal,
    }).json();
  },
});

// Отмена
mutation.cancel();
```

---

## Лучшие практики

### 1. Всегда используй AbortController в useEffect

```jsx
useEffect(() => {
  const controller = new AbortController();
  // ...
  return () => controller.abort();
}, []);
```

### 2. Проверяйте AbortError отдельно

```js
catch (error) {
  if (error.name === 'AbortError') {
    return; // Не ошибка
  }
  // Реальная ошибка
}
```

### 3. Не смешивайте CancelToken и AbortController

Используйте `AbortController` в новом коде. `CancelToken` deprecated.

### 4. Передавайте signal в TanStack Query

Это позволяет библиотеке корректно отменять запросы.

### 5. Используйте AbortController для timeout

```js
const controller = new AbortController();
setTimeout(() => controller.abort(), 10000);
```

### 6. Отменяйте запросы при навигации

Если пользователь уходит со страницы, все активные запросы должны отменяться.

---

## Антипаттерны

### 1. Игнорирование cleanup

```jsx
// ❌ Плохо: запрос продолжает выполняться после размонтирования
useEffect(() => {
  fetch('/api/users').then(setUser);
}, []);
```

### 2. Логирование AbortError как ошибки

```js
// ❌ Плохо
} catch (error) {
  console.error(error); // AbortError здесь
}
```

### 3. Создание AbortController вне эффекта

```jsx
// ❌ Плохо: один контроллер на все запросы
const controller = new AbortController();

useEffect(() => {
  fetch('/api/users', { signal: controller.signal });
}, []);
```

Каждый запрос должен иметь свой контроллер.

### 4. Отмена без проверки

```js
// ❌ Плохо: abort безусловно
controller.abort();

// ✅ Хорошо: abort в cleanup
return () => controller.abort();
```

### 5. Пренебрежение race conditions

```jsx
// ❌ Плохо: старый ответ может перезаписать новый
useEffect(() => {
  fetch(`/api/search?q=${query}`).then(setResults);
}, [query]);
```

---

## Итог

**Отмена запросов** — обязательный навык Middle+ разработчика. Ключевые моменты:

- `AbortController` — стандартный способ отмены для `fetch`, `axios`, `ky`.
- В React отменяйте запросы в `useEffect` cleanup.
- `AbortController` решает проблему race conditions при быстрой смене зависимостей.
- TanStack Query поддерживает сигнал — передавайте его в `queryFn`.
- Всегда разделяйте `AbortError` и реальные ошибки.
