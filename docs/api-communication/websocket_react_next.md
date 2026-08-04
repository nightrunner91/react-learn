# WebSocket в React и Next.js

## Содержание

1. [Что такое WebSocket](#что-такое-websocket)
2. [WebSocket vs HTTP vs SSE](#websocket-vs-http-vs-sse)
3. [Базовое использование в React](#базовое-использование-в-react)
4. [Хук useWebSocket](#хук-usewebsocket)
5. [Обработка переподключений](#обработка-переподключений)
6. [Интеграция с состоянием](#интеграция-с-состоянием)
7. [WebSocket и Suspense](#websocket-и-suspense)
8. [WebSocket в Next.js](#websocket-в-nextjs)
9. [Альтернативы: Socket.IO, Server-Sent Events](#альтернативы-socketio-server-sent-events)
10. [Масштабирование WebSocket](#масштабирование-websocket)
11. [Лучшие практики](#лучшие-практики)
12. [Антипаттерны](#антипаттерны)

---

## Что такое WebSocket

**WebSocket** — протокол полнодуплексной связи поверх TCP, позволяющий серверу и клиенту обмениваться сообщениями в реальном времени. В отличие от HTTP, где клиент всегда инициирует запрос, WebSocket устанавливает постоянное соединение, через которое обе стороны могут отправлять данные в любой момент.

```
HTTP:
Клиент → Запрос → Сервер
Клиент ← Ответ ← Сервер
(соединение закрыто)

WebSocket:
Клиент → Handshake (HTTP Upgrade) → Сервер
Клиент ←→ Постоянное соединение ←→ Сервер
(двусторонняя передача в реальном времени)
```

### Когда нужен WebSocket

- **Чат и мессенджеры** — мгновенная доставка сообщений
- **Коллаборативные редакторы** — синхронизация изменений между пользователями (Google Docs, Figma)
- **Торговые платформы** — обновления цен в реальном времени
- **Мультиплеерные игры** — синхронизация состояния игры
- **Уведомления** — push-уведомления в веб-приложениях
- **Мониторинг** — логи, метрики, дашборды в реальном времени

---

## WebSocket vs HTTP vs SSE

| | HTTP (REST) | Server-Sent Events (SSE) | WebSocket |
|---|---|---|---|
| Направление | Запрос-ответ | Сервер → Клиент | Двустороннее |
| Соединение | Кратковременное | Долговременное | Долговременное |
| Формат данных | JSON/XML | Текст (обычно JSON) | Текст / бинарные данные |
| Переподключение | Не нужно | Автоматическое | Ручное |
| Масштабирование | Легко | Средне | Сложнее |
| Поддержка | Везде | Все современные браузеры | Все современные браузеры |
| Прокси/CDN | Полная поддержка | Хорошая | Может блокироваться |

### Когда что использовать

```
Нужны ли клиенту данные от сервера в реальном времени?
├── Нет → HTTP REST
└── Да
    ├── Нужна ли двусторонняя связь?
    │   ├── Нет → SSE (проще, автоматическое переподключение)
    │   └── Да → WebSocket
    └── Нужна ли бинарная передача?
        ├── Нет → SSE или WebSocket
        └── Да → WebSocket
```

---

## Базовое использование в React

### Нативный WebSocket API

```jsx
function ChatRoom() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");
  const wsRef = useRef(null);

  useEffect(() => {
    const ws = new WebSocket("wss://chat.example.com");
    wsRef.current = ws;

    ws.onopen = () => {
      console.log("Connected");
    };

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      setMessages((prev) => [...prev, message]);
    };

    ws.onerror = (error) => {
      console.error("WebSocket error:", error);
    };

    ws.onclose = () => {
      console.log("Disconnected");
    };

    return () => {
      ws.close();
    };
  }, []);

  const sendMessage = () => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({ text: input, timestamp: Date.now() }));
      setInput("");
    }
  };

  return (
    <div>
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i}>{msg.text}</div>
        ))}
      </div>
      <input value={input} onChange={(e) => setInput(e.target.value)} />
      <button onClick={sendMessage}>Send</button>
    </div>
  );
}
```

### Проблемы базового подхода

- Нет переподключения при обрыве связи
- Нет обработки потери сообщений
- Нет очереди сообщений для отправки
- Нет интеграции с глобальным состоянием
- Сложно переиспользовать между компонентами

---

## Хук useWebSocket

Абстрагируем логику WebSocket в переиспользуемый хук:

```jsx
function useWebSocket(url) {
  const [status, setStatus] = useState("connecting");
  const [lastMessage, setLastMessage] = useState(null);
  const wsRef = useRef(null);
  const reconnectTimeoutRef = useRef(null);
  const reconnectAttemptsRef = useRef(0);

  const connect = useCallback(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      setStatus("connected");
      reconnectAttemptsRef.current = 0;
    };

    ws.onmessage = (event) => {
      setLastMessage(JSON.parse(event.data));
    };

    ws.onerror = () => {
      setStatus("error");
    };

    ws.onclose = () => {
      setStatus("disconnected");
      scheduleReconnect();
    };
  }, [url]);

  const scheduleReconnect = useCallback(() => {
    const maxDelay = 30000;
    const delay = Math.min(1000 * 2 ** reconnectAttemptsRef.current, maxDelay);
    reconnectAttemptsRef.current += 1;

    reconnectTimeoutRef.current = setTimeout(() => {
      connect();
    }, delay);
  }, [connect]);

  const sendMessage = useCallback((data) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    }
  }, []);

  const disconnect = useCallback(() => {
    clearTimeout(reconnectTimeoutRef.current);
    wsRef.current?.close();
  }, []);

  useEffect(() => {
    connect();
    return () => disconnect();
  }, [connect, disconnect]);

  return { status, lastMessage, sendMessage, disconnect };
}
```

### Использование

```jsx
function ChatRoom({ roomId }) {
  const { status, lastMessage, sendMessage } = useWebSocket(
    `wss://chat.example.com/rooms/${roomId}`
  );

  useEffect(() => {
    if (lastMessage) {
      // обработка входящего сообщения
      console.log("Received:", lastMessage);
    }
  }, [lastMessage]);

  return (
    <div>
      <span>Status: {status}</span>
      <button onClick={() => sendMessage({ text: "Hello!" })}>Send</button>
    </div>
  );
}
```

---

## Обработка переподключений

### Экспоненциальная задержка (Exponential Backoff)

```jsx
function useWebSocketWithBackoff(url) {
  const wsRef = useRef(null);
  const reconnectRef = useRef(null);
  const attemptRef = useRef(0);

  const connect = useCallback(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      attemptRef.current = 0;
    };

    ws.onclose = () => {
      const delay = Math.min(1000 * 2 ** attemptRef.current, 30000);
      attemptRef.current += 1;
      reconnectRef.current = setTimeout(connect, delay);
    };
  }, [url]);

  useEffect(() => {
    connect();
    return () => {
      clearTimeout(reconnectRef.current);
      wsRef.current?.close();
    };
  }, [connect]);
}
```

### Очередь сообщений

Если соединение разорвано, сообщения ставятся в очередь и отправляются при восстановлении:

```jsx
function useWebSocketWithQueue(url) {
  const wsRef = useRef(null);
  const queueRef = useRef([]);
  const [status, setStatus] = useState("connecting");

  const flushQueue = useCallback(() => {
    while (queueRef.current.length > 0 && wsRef.current?.readyState === WebSocket.OPEN) {
      const message = queueRef.current.shift();
      wsRef.current.send(JSON.stringify(message));
    }
  }, []);

  const sendMessage = useCallback((data) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    } else {
      queueRef.current.push(data);
    }
  }, []);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      setStatus("connected");
      flushQueue();
    };

    ws.onclose = () => {
      setStatus("disconnected");
    };

    return () => ws.close();
  }, [url, flushQueue]);

  return { status, sendMessage };
}
```

### Heartbeat (Ping/Pong)

Для обнаружения «мёртвых» соединений:

```jsx
function useWebSocketWithHeartbeat(url) {
  const wsRef = useRef(null);
  const heartbeatRef = useRef(null);

  const startHeartbeat = useCallback(() => {
    heartbeatRef.current = setInterval(() => {
      if (wsRef.current?.readyState === WebSocket.OPEN) {
        wsRef.current.send(JSON.stringify({ type: "ping" }));
      }
    }, 30000);
  }, []);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      startHeartbeat();
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (data.type === "pong") return;
      // обработка обычных сообщений
    };

    ws.onclose = () => {
      clearInterval(heartbeatRef.current);
    };

    return () => {
      clearInterval(heartbeatRef.current);
      ws.close();
    };
  }, [url, startHeartbeat]);
}
```

---

## Интеграция с состоянием

### Zustand + WebSocket

```jsx
import { create } from "zustand";

const useChatStore = create((set, get) => ({
  messages: [],
  status: "disconnected",
  ws: null,

  connect: (url) => {
    const ws = new WebSocket(url);

    ws.onopen = () => set({ status: "connected", ws });
    ws.onclose = () => set({ status: "disconnected", ws: null });
    ws.onerror = () => set({ status: "error" });

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      set((state) => ({
        messages: [...state.messages, message],
      }));
    };
  },

  sendMessage: (text) => {
    const { ws } = get();
    if (ws?.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ text, timestamp: Date.now() }));
    }
  },

  disconnect: () => {
    const { ws } = get();
    ws?.close();
  },
}));

function ChatRoom({ roomId }) {
  const { messages, status, connect, sendMessage, disconnect } = useChatStore();

  useEffect(() => {
    connect(`wss://chat.example.com/rooms/${roomId}`);
    return () => disconnect();
  }, [roomId, connect, disconnect]);

  return (
    <div>
      <span>Status: {status}</span>
      <div>{messages.map((msg, i) => <div key={i}>{msg.text}</div>)}</div>
      <button onClick={() => sendMessage("Hello!")}>Send</button>
    </div>
  );
}
```

### TanStack Query + WebSocket

Для интеграции WebSocket с кэшем TanStack Query:

```jsx
function useWebSocketQuery(queryKey, url, messageSelector) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const ws = new WebSocket(url);

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      const relevantData = messageSelector(data);

      if (relevantData) {
        queryClient.setQueryData(queryKey, (old) => {
          // обновление кэша на основе WebSocket-сообщения
          return { ...old, ...relevantData };
        });
      }
    };

    return () => ws.close();
  }, [url, queryKey, messageSelector, queryClient]);
}

// Использование
function Dashboard() {
  const { data } = useQuery({
    queryKey: ["metrics"],
    queryFn: () => fetch("/api/metrics").then((r) => r.json()),
  });

  useWebSocketQuery(
    ["metrics"],
    "wss://metrics.example.com",
    (data) => data.type === "metrics_update" ? data.payload : null
  );

  return <MetricsChart data={data} />;
}
```

---

## WebSocket и Suspense

WebSocket по природе асинхронен и не интегрируется напрямую с Suspense (который работает с Promise). Но можно комбинировать:

```jsx
function createWebSocketResource(url) {
  let status = "pending";
  let result = null;
  let suspender = new Promise((resolve) => {
    const ws = new WebSocket(url);

    ws.onopen = () => {
      status = "success";
      result = ws;
      resolve(ws);
    };

    ws.onerror = (error) => {
      status = "error";
      result = error;
      resolve(null);
    };
  });

  return {
    read() {
      if (status === "pending") throw suspender;
      if (status === "error") throw result;
      return result;
    },
  };
}

function ChatRoom({ url }) {
  const resource = useMemo(() => createWebSocketResource(url), [url]);
  const ws = resource.read();

  // использование ws
}

function App() {
  return (
    <Suspense fallback={<div>Connecting to chat...</div>}>
      <ChatRoom url="wss://chat.example.com" />
    </Suspense>
  );
}
```

---

## WebSocket в Next.js

### Ограничения

Next.js — это фреймворк для серверного рендеринга. WebSocket-соединение — это клиентская концепция. Правила:

- WebSocket-соединение устанавливается **только на клиенте** (в `useEffect` или обработчиках событий)
- Нельзя использовать WebSocket в Server Components напрямую
- Next.js API Routes не поддерживают WebSocket «из коробки»

### Отдельный WebSocket-сервер

Для production используйте отдельный WebSocket-сервер:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Next.js    │     │   WebSocket  │     │   Database   │
│   (SSR/API)  │     │   Server     │     │              │
│              │     │  (ws://...)  │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
        ▲                    ▲                    ▲
        │                    │                    │
        └────────────────────┴────────────────────┘
                     Общий API / БД
```

### WebSocket-сервер на Node.js

```js
// ws-server.js
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  console.log("Client connected");

  ws.on("message", (data) => {
    const message = JSON.parse(data);
    console.log("Received:", message);

    // рассылка всем подключённым клиентам
    wss.clients.forEach((client) => {
      if (client.readyState === 1) {
        client.send(JSON.stringify(message));
      }
    });
  });

  ws.on("close", () => {
    console.log("Client disconnected");
  });
});
```

### Интеграция с Next.js через API

```tsx
// app/api/chat/route.ts — HTTP API для истории сообщений
export async function GET() {
  const messages = await db.messages.findMany({
    orderBy: { createdAt: "desc" },
    take: 50,
  });
  return Response.json(messages);
}

// components/ChatRoom.tsx — клиентский компонент для WebSocket
"use client";

function ChatRoom({ roomId }) {
  const { data: history } = useQuery({
    queryKey: ["chat-history", roomId],
    queryFn: () => fetch(`/api/chat?roomId=${roomId}`).then((r) => r.json()),
  });

  const { status, lastMessage, sendMessage } = useWebSocket(
    `wss://ws.example.com/rooms/${roomId}`
  );

  const messages = useMemo(() => {
    const live = lastMessage ? [lastMessage] : [];
    return [...(history ?? []), ...live];
  }, [history, lastMessage]);

  return (
    <div>
      <div>Status: {status}</div>
      {messages.map((msg, i) => <div key={i}>{msg.text}</div>)}
      <button onClick={() => sendMessage({ text: "Hello" })}>Send</button>
    </div>
  );
}
```

### WebSocket в Edge Runtime

Edge Runtime (Vercel Edge, Cloudflare Workers) **не поддерживает** WebSocket-серверы. Для edge-окружений используйте:

- **Cloudflare Durable Objects** — WebSocket в edge
- **Vercel Edge + внешний WebSocket-сервер** — раздельная архитектура
- **Pusher / Ably / Supabase Realtime** — managed WebSocket-сервисы

---

## Альтернативы: Socket.IO, Server-Sent Events

### Socket.IO

Библиотека над WebSocket с дополнительными возможностями:

```jsx
import { io } from "socket.io-client";

function ChatRoom() {
  const socketRef = useRef(null);

  useEffect(() => {
    socketRef.current = io("https://chat.example.com", {
      transports: ["websocket"],
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionAttempts: 10,
    });

    socketRef.current.on("connect", () => {
      console.log("Connected");
    });

    socketRef.current.on("message", (data) => {
      console.log("Received:", data);
    });

    socketRef.current.on("disconnect", () => {
      console.log("Disconnected");
    });

    return () => {
      socketRef.current?.disconnect();
    };
  }, []);

  const sendMessage = (text) => {
    socketRef.current?.emit("message", { text });
  };
}
```

**Socket.IO vs нативный WebSocket:**

| | WebSocket | Socket.IO |
|---|---|---|
| Протокол | Стандартный | Проприетарный (fallback на polling) |
| Переподключение | Ручное | Встроенное |
| Rooms/Namespaces | Ручная реализация | Встроенные |
| Размер | Нативный API | ~40 KB (клиент) |
| Совместимость | Требует WebSocket-сервер | Требует Socket.IO-сервер |

### Server-Sent Events (SSE)

Односторонняя передача от сервера к клиенту:

```jsx
function LiveFeed() {
  const [events, setEvents] = useState([]);

  useEffect(() => {
    const eventSource = new EventSource("/api/events");

    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setEvents((prev) => [...prev, data]);
    };

    eventSource.onerror = () => {
      eventSource.close();
    };

    return () => eventSource.close();
  }, []);

  return (
    <div>
      {events.map((event, i) => <div key={i}>{event.text}</div>)}
    </div>
  );
}
```

**SSE vs WebSocket:**

| | SSE | WebSocket |
|---|---|---|
| Направление | Сервер → Клиент | Двустороннее |
| Переподключение | Автоматическое | Ручное |
| Простота | Простой HTTP | Требует handshake |
| Бинарные данные | Нет | Да |
| Масштабирование | Легче | Сложнее |

**Когда SSE:**
- Уведомления, ленты событий
- Обновления дашбордов
- Стриминг данных от сервера

**Когда WebSocket:**
- Чат, коллаборативные редакторы
- Игры
- Двусторонняя связь

---

## Масштабирование WebSocket

### Проблема

Каждое WebSocket-соединение — это постоянное TCP-соединение. При 10 000 пользователей — 10 000 соединений на одном сервере.

### Решения

#### 1. Sticky Sessions (для нескольких серверов)

```
         ┌──────────┐
         │   Load   │
         │ Balancer │
         └────┬─────┘
              │
    ┌─────────┼─────────┐
    │         │         │
┌───┴──┐  ┌──┴───┐  ┌──┴───┐
│ WS-1 │  │ WS-2 │  │ WS-3 │
└──────┘  └──────┘  └──────┘
```

Sticky sessions гарантируют, что один клиент всегда подключён к одному серверу.

#### 2. Pub/Sub через Redis

```js
import Redis from "ioredis";

const pub = new Redis();
const sub = new Redis();

sub.subscribe("chat:room:1");

sub.on("message", (channel, message) => {
  // рассылка всем подключённым клиентам на этом сервере
  wss.clients.forEach((client) => {
    if (client.roomId === "1") {
      client.send(message);
    }
  });
});

// При получении сообщения от клиента
ws.on("message", (data) => {
  pub.publish("chat:room:1", data);
});
```

#### 3. Managed сервисы

- **Pusher** — hosted WebSocket-платформа
- **Ably** — realtime messaging
- **Supabase Realtime** — WebSocket поверх PostgreSQL
- **Firebase Realtime Database** — синхронизация данных
- **AWS AppSync** — GraphQL + WebSocket

```jsx
import Pusher from "pusher-js";

function LiveFeed() {
  useEffect(() => {
    const pusher = new Pusher("app-key", { cluster: "eu" });
    const channel = pusher.subscribe("updates");

    channel.bind("new-data", (data) => {
      console.log("Update:", data);
    });

    return () => pusher.disconnect();
  }, []);
}
```

---

## Лучшие практики

### 1. Используйте wss:// (WebSocket Secure)

```jsx
// ❌ Небезопасно
const ws = new WebSocket("ws://example.com");

// ✅ Безопасно
const ws = new WebSocket("wss://example.com");
```

### 2. Валидируйте сообщения

```jsx
ws.onmessage = (event) => {
  try {
    const data = JSON.parse(event.data);

    if (!data.type || !data.payload) {
      console.warn("Invalid message format");
      return;
    }

    // обработка валидного сообщения
  } catch {
    console.warn("Failed to parse message");
  }
};
```

### 3. Обрабатывайте все состояния соединения

```jsx
const readyStates = {
  [WebSocket.CONNECTING]: "connecting",
  [WebSocket.OPEN]: "open",
  [WebSocket.CLOSING]: "closing",
  [WebSocket.CLOSED]: "closed",
};

function ConnectionStatus({ ws }) {
  const [status, setStatus] = useState(ws.readyState);

  useEffect(() => {
    const updateStatus = () => setStatus(ws.readyState);
    ws.addEventListener("open", updateStatus);
    ws.addEventListener("close", updateStatus);
    return () => {
      ws.removeEventListener("open", updateStatus);
      ws.removeEventListener("close", updateStatus);
    };
  }, [ws]);

  return <span>{readyStates[status]}</span>;
}
```

### 4. Закрывайте соединения при размонтировании

```jsx
useEffect(() => {
  const ws = new WebSocket(url);

  return () => {
    ws.close(1000, "Component unmounted");
  };
}, [url]);
```

### 5. Используйте типизированные сообщения

```tsx
type WebSocketMessage =
  | { type: "chat_message"; payload: { text: string; userId: string } }
  | { type: "user_joined"; payload: { userId: string } }
  | { type: "user_left"; payload: { userId: string } }
  | { type: "typing"; payload: { userId: string; isTyping: boolean } };

function useTypedWebSocket(url: string) {
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onmessage = (event) => {
      const message: WebSocketMessage = JSON.parse(event.data);

      switch (message.type) {
        case "chat_message":
          // обработка
          break;
        case "user_joined":
          // обработка
          break;
      }
    };

    return () => ws.close();
  }, [url]);
}
```

---

## Антипаттерны

### 1. Создание соединения в каждом компоненте

```jsx
// ❌ Каждое создание ChatRoom создаёт новое соединение
function ChatRoom({ roomId }) {
  useEffect(() => {
    const ws = new WebSocket(`wss://example.com/rooms/${roomId}`);
    return () => ws.close();
  }, [roomId]);
}

// ✅ Одно соединение, управление комнатами через сообщения
function ChatApp() {
  const { sendMessage } = useWebSocket("wss://example.com");

  const joinRoom = (roomId) => {
    sendMessage({ type: "join_room", roomId });
  };
}
```

### 2. Игнорирование состояний соединения

```jsx
// ❌ Отправка без проверки состояния
const sendMessage = (data) => {
  ws.send(JSON.stringify(data));
};

// ✅ Проверка перед отправкой
const sendMessage = (data) => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify(data));
  } else {
    queueMessage(data);
  }
};
```

### 3. Отсутствие обработки ошибок

```jsx
// ❌ Нет обработки ошибок
const ws = new WebSocket(url);
ws.onmessage = handleMessage;

// ✅ Полная обработка
const ws = new WebSocket(url);
ws.onopen = handleOpen;
ws.onmessage = handleMessage;
ws.onerror = handleError;
ws.onclose = handleClose;
```

### 4. Хранение WebSocket в состоянии

```jsx
// ❌ WebSocket в useState вызывает лишние рендеры
const [ws, setWs] = useState(null);

// ✅ WebSocket в useRef
const wsRef = useRef(null);
```

### 5. Отсутствие аутентификации

```jsx
// ❌ Анонимное соединение
const ws = new WebSocket("wss://example.com");

// ✅ Токен в URL или первом сообщении
const ws = new WebSocket(`wss://example.com?token=${authToken}`);

// Или через первое сообщение после подключения
ws.onopen = () => {
  ws.send(JSON.stringify({ type: "auth", token: authToken }));
};
```
