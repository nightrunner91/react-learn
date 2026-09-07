# Long Polling — обновления через обычный HTTP

## Содержание

1. [Что такое Long Polling](#что-такое-long-polling)
2. [Как работает](#как-работает)
3. [Когда использовать](#когда-использовать)
4. [Простая реализация на fetch](#простая-реализация-на-fetch)
5. [Реализация в React](#реализация-в-react)
6. [Управление жизненным циклом](#управление-жизненным-циклом)
7. [Обработка ошибок и переподключений](#обработка-ошибок-и-переподключений)
8. [Heartbeat и timeout](#heartbeat-и-timeout)
9. [Проблемы и ограничения](#проблемы-и-ограничения)
10. [Long Polling vs WebSocket vs SSE](#long-polling-vs-websocket-vs-sse)
11. [Лучшие практики](#лучшие-практики)
12. [Антипаттерны](#антипаттерны)

---

## Что такое Long Polling

**Long Polling** — техника получения обновлений с сервера в реальном времени через обычный HTTP. В отличие от классического polling, где клиент постоянно спрашивает «есть новые данные?», long polling держит соединение открытым до тех пор, пока сервер не пришлёт ответ.

### Обычный polling

```
Клиент → Запрос → Сервер
Клиент ← Пусто  ← Сервер  (через 1 секунду)
Клиент → Запрос → Сервер
Клиент ← Пусто  ← Сервер
Клиент → Запрос → Сервер
Клиент ← Данные ← Сервер
```

### Long polling

```
Клиент → Запрос → Сервер  (соединение держится открытым)
Клиент ← Данные ← Сервер  (когда появятся)
Клиент → Запрос → Сервер  (сразу после получения ответа)
```

---

## Как работает

1. Клиент отправляет HTTP-запрос на сервер.
2. Сервер не отвечает сразу, а ждёт появления новых данных.
3. Как только данные появляются, сервер отправляет ответ и закрывает соединение.
4. Клиент сразу отправляет новый запрос.

На сервере обычно используется timeout: если данных нет в течение 30–60 секунд, сервер возвращает пустой ответ или специальный статус, чтобы клиент не ждал вечно.

---

## Когда использовать

Long Polling хорош, когда:

- Нужны почти real-time обновления.
- Сервер или инфраструктура не поддерживает WebSocket.
- Нужна совместимость с HTTP-прокси и CDN.
- Нужна односторонняя доставка данных (сервер → клиент).
- WebSocket заблокирован корпоративным firewall.

Не используй long polling, если:

- Нужна двусторонняя низколатентная коммуникация — WebSocket лучше.
- Важна экономия ресурсов сервера — WebSocket/SSE эффективнее.
- Обновления происходят очень часто — long polling создаёт большую нагрузку.

---

## Простая реализация на fetch

```js
async function longPoll() {
  try {
    const response = await fetch('/api/notifications/poll', {
      method: 'GET',
      headers: {
        Accept: 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const data = await response.json();

    if (data.length > 0) {
      handleNotifications(data);
    }
  } catch (error) {
    console.error('Long polling error:', error);
    await delay(5000); // Пауза перед повтором при ошибке
  }

  // Сразу запускаем следующий запрос
  longPoll();
}

function delay(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

longPoll();
```

### Серверная часть (псевдокод)

```js
app.get('/api/notifications/poll', async (req, res) => {
  const timeout = setTimeout(() => {
    res.json([]); // Пустой ответ через 30 секунд
  }, 30000);

  const unsubscribe = eventBus.subscribe('notification', (notification) => {
    clearTimeout(timeout);
    res.json([notification]);
    unsubscribe();
  });
});
```

---

## Реализация в React

### Хук useLongPolling

```jsx
import { useEffect, useRef, useState, useCallback } from 'react';

function useLongPolling(url, options = {}) {
  const { onMessage, onError, enabled = true } = options;
  const [isRunning, setIsRunning] = useState(false);
  const abortControllerRef = useRef(null);
  const isMountedRef = useRef(true);

  const poll = useCallback(async () => {
    if (!isMountedRef.current || !enabled) return;

    setIsRunning(true);
    abortControllerRef.current = new AbortController();

    try {
      const response = await fetch(url, {
        signal: abortControllerRef.current.signal,
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      const data = await response.json();

      if (isMountedRef.current) {
        onMessage?.(data);
      }
    } catch (error) {
      if (error.name === 'AbortError') {
        return; // Нормальная отмена
      }

      onError?.(error);

      if (isMountedRef.current) {
        await delay(5000); // Пауза перед повтором
      }
    } finally {
      abortControllerRef.current = null;
      if (isMountedRef.current && enabled) {
        poll(); // Следующий запрос
      } else {
        setIsRunning(false);
      }
    }
  }, [url, onMessage, onError, enabled]);

  useEffect(() => {
    isMountedRef.current = true;
    if (enabled) {
      poll();
    }

    return () => {
      isMountedRef.current = false;
      abortControllerRef.current?.abort();
    };
  }, [enabled, poll]);

  return { isRunning };
}

function delay(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

### Использование

```jsx
function Notifications() {
  const [notifications, setNotifications] = useState([]);

  useLongPolling('/api/notifications/poll', {
    onMessage: (data) => {
      setNotifications((prev) => [...prev, ...data]);
    },
    onError: (error) => {
      console.error('Polling failed:', error);
    },
  });

  return (
    <ul>
      {notifications.map((n) => (
        <li key={n.id}>{n.message}</li>
      ))}
    </ul>
  );
}
```

---

## Управление жизненным циклом

Важно уметь запускать и останавливать long polling. Например, когда пользователь уходит со страницы или сворачивает вкладку.

### Остановка через AbortController

```js
const controller = new AbortController();

async function poll() {
  try {
    const response = await fetch('/api/poll', {
      signal: controller.signal,
    });
    // ...
  } catch (error) {
    if (error.name === 'AbortError') {
      console.log('Polling stopped');
      return;
    }
  }
}

// Остановка
controller.abort();
```

### Пауза при скрытии вкладки

```js
useEffect(() => {
  const handleVisibilityChange = () => {
    if (document.hidden) {
      stopPolling();
    } else {
      startPolling();
    }
  };

  document.addEventListener('visibilitychange', handleVisibilityChange);
  return () => document.removeEventListener('visibilitychange', handleVisibilityChange);
}, []);
```

---

## Обработка ошибок и переподключений

Long polling должен быть устойчив к сетевым проблемам. Базовая стратегия:

1. При ошибке подождать несколько секунд.
2. Увеличивать задержку при повторных ошибках (exponential backoff).
3. Сбрасывать задержку после успешного запроса.

```js
let attempts = 0;

async function poll() {
  try {
    const response = await fetch('/api/poll');
    if (!response.ok) throw new Error(`HTTP ${response.status}`);

    const data = await response.json();
    handleData(data);
    attempts = 0; // Сброс счётчика
  } catch (error) {
    const delay = Math.min(1000 * 2 ** attempts, 30000);
    attempts++;
    console.error(`Polling error, retry in ${delay}ms`);
    await sleep(delay);
  }

  poll();
}
```

---

## Heartbeat и timeout

Сервер должен иметь timeout, чтобы висящие соединения не копились. Клиент должен это учитывать:

```js
const response = await fetch('/api/poll', {
  signal: controller.signal,
});
```

Если сервер закрывает соединение по таймауту с пустым ответом, клиент просто делает новый запрос.

Если нужно гарантированно знать, что сервер жив, можно комбинировать long polling с периодическим heartbeat-запросом. Но часто это избыточно.

---

## Проблемы и ограничения

### 1. Нагрузка на сервер

Каждый long polling запрос удерживает соединение открытым. При тысячах клиентов сервер держит тысячи открытых соединений. Для масштабирования нужны:

- Асинхронные серверы (Node.js, Go, Erlang).
- Балансировка нагрузки.
- Разумные таймауты.

### 2. Задержка между обновлениями

После получения ответа клиент должен отправить новый запрос. Это добавляет задержку в 1 RTT.

### 3. Проблемы с HTTP-прокси

Некоторые прокси могут разрывать долгие соединения раньше, чем сервер отправит ответ.

### 4. Сложнее, чем SSE

Если нужна только односторонняя доставка от сервера, SSE проще и эффективнее.

---

## Long Polling vs WebSocket vs SSE

| Критерий | Long Polling | WebSocket | SSE |
|----------|--------------|-----------|-----|
| Протокол | HTTP | WebSocket (поверх TCP) | HTTP |
| Направление | Сервер → клиент | Двустороннее | Сервер → клиент |
| Задержка | ~1 RTT между запросами | Минимальная | Минимальная |
| Сложность | Средняя | Выше | Низкая |
| Поддержка прокси | Хорошая | Может блокироваться | Хорошая |
| Авто reconnect | Ручная реализация | Ручная реализация | Встроена |
| Бинарные данные | Нет | Да | Нет |
| Нагрузка на сервер | Средняя | Низкая при большом числе соединений | Низкая |

**Правило выбора:**

```
Нужны real-time обновления?
├── Да
│   ├── Нужна двусторонняя связь? → WebSocket
│   └── Только сервер → клиент?
│       ├── Можно использовать SSE? → SSE (проще)
│       └── SSE недоступен / нужна совместимость → Long Polling
└── Нет → Обычный HTTP polling
```

---

## Лучшие практики

### 1. Используй AbortController

Всегда имей возможность остановить polling, особенно в React.

### 2. Экспоненциальный backoff при ошибках

Не долбите сервер запросами каждую секунду при сбое.

### 3. Учитывайте visibilitychange

Останавливайте polling, когда вкладка не активна.

### 4. Устанавливайте серверный timeout

Не держите соединения вечно. 30–60 секунд — разумный лимит.

### 5. Возвращайте пустой ответ вместо зависания

Если данных нет, сервер должен вернуть пустой массив или `204 No Content`, а не молчать бесконечно.

### 6. Логируйте события polling

```js
console.log('[long-poll] connected');
console.log('[long-poll] message received', data.length);
console.log('[long-poll] error', error.message);
```

---

## Антипаттерны

### 1. Polling без задержки при ошибках

```js
// ❌ Плохо: при ошибке сразу новый запрос
async function badPoll() {
  try {
    await fetch('/api/poll');
  } catch (e) {
    // ничего
  }
  badPoll(); // мгновенно
}
```

### 2. Игнорирование AbortError

```js
// ❌ Плохо: логируем AbortError как ошибку
} catch (error) {
  console.error(error); // AbortError тоже попадёт сюда
}
```

### 3. Хранение состояния polling в useState

```js
// ❌ Плохо: лишние рендеры
const [controller, setController] = useState(null);

// ✅ Хорошо: useRef
const controllerRef = useRef(null);
```

### 4. Запуск нескольких polling-циклов

```js
// ❌ Плохо: каждый рендер запускает новый poll
useEffect(() => {
  poll();
}); // нет массива зависимостей
```

### 5. Бесконечное ожидание ответа от сервера

```js
// ❌ Плохо: нет timeout
const response = await fetch('/api/poll');

// ✅ Хорошо: AbortController + timeout
const controller = new AbortController();
setTimeout(() => controller.abort(), 35000);
```

---

## Итог

**Long Polling** — это простой способ получать обновления с сервера через обычный HTTP. Он проигрывает WebSocket и SSE по эффективности, но остаётся полезным, когда WebSocket недоступен, а SSE не подходит.

Ключевые моменты для Middle+ разработчика:

- Уметь реализовать polling с отменой через `AbortController`.
- Понимать разницу между polling, long polling, SSE и WebSocket.
- Обрабатывать ошибки и переподключения с exponential backoff.
- Управлять жизненным циклом: останавливать при размонтировании и скрытии вкладки.
