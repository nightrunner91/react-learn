# Progressive Web Apps (PWA) в React и Next.js

## Содержание

1. [Что такое PWA](#что-такое-pwa)
2. [Три столпа PWA](#три-столпа-pwa)
3. [Web App Manifest](#web-app-manifest)
4. [Service Workers](#service-workers)
5. [Кэширование стратегий](#кэширование-стратегий)
6. [Offline-режим](#offline-режим)
7. [Push-уведомления](#push-уведомления)
8. [Background Sync](#background-sync)
9. [PWA в Next.js](#pwa-в-nextjs)
10. [Установка на устройство](#установка-на-устройство)
11. [Тестирование PWA](#тестирование-pwa)
12. [Лучшие практики](#лучшие-практики)
13. [Ограничения PWA](#ограничения-pwa)

---

## Что такое PWA

**Progressive Web App (PWA)** — веб-приложение, использующее современные браузерные API для работы как нативное приложение: офлайн-режим, push-уведомления, установка на домашний экран, доступ к аппаратным возможностям.

PWA — это не отдельная технология, а набор практик:

```
PWA = HTTPS + Web App Manifest + Service Worker
```

Ключевые преимущества:
- **Не нужен App Store** — установка прямо из браузера
- **Автоматические обновления** — как у обычного сайта
- **Работает офлайн** — через Service Worker
- **Push-уведомления** — как нативное приложение
- **Быстрая загрузка** — агрессивное кэширование
- **Меньший размер** — несколько MB вместо 100+ MB нативного приложения

---

## Три столпа PWA

### 1. HTTPS

PWA требует HTTPS (кроме localhost для разработки). Это гарантирует:
- Безопасность данных пользователя
- Целостность Service Worker (нельзя подменить)
- Безопасность push-уведомлений

### 2. Web App Manifest

JSON-файл, описывающий приложение для браузера:

```json
// public/manifest.json
{
  "name": "My React App",
  "short_name": "MyApp",
  "description": "A progressive web app built with React",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "orientation": "portrait",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512-maskable.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ],
  "categories": ["productivity", "utilities"],
  "screenshots": [
    {
      "src": "/screenshots/home.png",
      "sizes": "1280x720",
      "type": "image/png"
    }
  ]
}
```

```html
<!-- index.html -->
<link rel="manifest" href="/manifest.json" />
<meta name="theme-color" content="#3b82f6" />
<link rel="apple-touch-icon" href="/icons/icon-192.png" />
```

### 3. Service Worker

JavaScript-скрипт, работающий в фоне, отдельно от страницы:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Browser    │     │   Service    │     │   Network    │
│   Tab        │◄───►│   Worker     │◄───►│   / Cache    │
│              │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
```

Service Worker перехватывает сетевые запросы и может:
- Отдавать контент из кэша (офлайн)
- Кэшировать ресурсы
- Обрабатывать push-уведомления
- Выполнять фоновую синхронизацию

---

## Web App Manifest

### Display modes

| Mode | Описание | Когда использовать |
|---|---|---|
| `standalone` | Выглядит как нативное приложение (без адресной строки) | Большинство PWA |
| `fullscreen` | Полноэкранный режим | Игры, медиа |
| `minimal-ui` | Минимальный UI (кнопки навигации) | Когда нужен контроль, но с навигацией |
| `browser` | Обычная вкладка браузера | Fallback |

### Icons

Требования для иконок:
- **192x192** — минимальный размер для Android
- **512x512** — для splash screen
- **maskable** — для адаптивных иконок Android

```json
"icons": [
  {
    "src": "/icons/icon-192.png",
    "sizes": "192x192",
    "type": "image/png"
  },
  {
    "src": "/icons/icon-512.png",
    "sizes": "512x512",
    "type": "image/png",
    "purpose": "any"
  },
  {
    "src": "/icons/icon-512-maskable.png",
    "sizes": "512x512",
    "type": "image/png",
    "purpose": "maskable"
  }
]
```

### Scope и start_url

```json
{
  "start_url": "/dashboard",
  "scope": "/"
}
```

- `start_url` — страница, открываемая при запуске PWA
- `scope` — границы навигации PWA (выход за scope открывает браузер)

---

## Service Workers

### Регистрация

```jsx
// app.tsx или index.tsx
if ("serviceWorker" in navigator) {
  window.addEventListener("load", () => {
    navigator.serviceWorker
      .register("/sw.js")
      .then((registration) => {
        console.log("SW registered:", registration.scope);
      })
      .catch((error) => {
        console.log("SW registration failed:", error);
      });
  });
}
```

### Базовый Service Worker

```js
// public/sw.js
const CACHE_NAME = "my-app-v1";
const urlsToCache = [
  "/",
  "/index.html",
  "/static/js/main.js",
  "/static/css/main.css",
  "/manifest.json",
];

// Install — кэширование ресурсов
self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(urlsToCache);
    })
  );
});

// Activate — очистка старых кэшей
self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      );
    })
  );
});

// Fetch — обслуживание запросов из кэша
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request);
    })
  );
});
```

### Жизненный цикл Service Worker

```
1. Register → Downloading → Installed → Waiting → Activating → Activated
                                                         ↑
                                              (старый SW всё ещё активен)

2. При обновлении: новый SW устанавливается, но ждёт, пока все вкладки
   со старым SW не будут закрыты. Затем активируется.

3. skipWaiting() — активировать новый SW немедленно
4. clients.claim() — новый SW сразу контролирует все вкладки
```

```js
// sw.js — немедленная активация
self.addEventListener("install", (event) => {
  self.skipWaiting();
});

self.addEventListener("activate", (event) => {
  event.waitUntil(clients.claim());
});
```

---

## Кэширование стратегий

### Cache First (кэш первый)

```js
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      if (cached) return cached;
      return fetch(event.request).then((response) => {
        if (response.ok) {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((cache) => cache.put(event.request, clone));
        }
        return response;
      });
    })
  );
});
```

**Когда использовать:** Статические ресурсы (JS, CSS, шрифты, изображения).

### Network First (сеть первая)

```js
self.addEventListener("fetch", (event) => {
  event.respondWith(
    fetch(event.request)
      .then((response) => {
        if (response.ok) {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((cache) => cache.put(event.request, clone));
        }
        return response;
      })
      .catch(() => {
        return caches.match(event.request);
      })
  );
});
```

**Когда использовать:** API-запросы, динамический контент.

### Stale While Revalidate

```js
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      const fetchPromise = fetch(event.request).then((response) => {
        if (response.ok) {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((cache) => cache.put(event.request, clone));
        }
        return response;
      });

      return cached || fetchPromise;
    })
  );
});
```

**Когда использовать:** Контент, который должен быть быстрым, но обновляться в фоне.

### Cache Only

```js
self.addEventListener("fetch", (event) => {
  event.respondWith(caches.match(event.request));
});
```

**Когда использовать:** Ресурсы, которые никогда не меняются (версионированные файлы).

---

## Offline-режим

### Кэширование HTML для офлайн

```js
const STATIC_CACHE = "static-v1";
const PAGES_CACHE = "pages-v1";

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) => {
      return cache.addAll([
        "/offline.html",
        "/static/js/main.js",
        "/static/css/main.css",
      ]);
    })
  );
});

self.addEventListener("fetch", (event) => {
  if (event.request.mode === "navigate") {
    event.respondWith(
      fetch(event.request).catch(() => {
        return caches.match("/offline.html");
      })
    );
    return;
  }

  event.respondWith(
    caches.match(event.request).then((cached) => {
      return cached || fetch(event.request);
    })
  );
});
```

### Offline-страница

```html
<!-- public/offline.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Offline</title>
  <style>
    body { font-family: system-ui; text-align: center; padding: 50px; }
    h1 { font-size: 24px; }
    p { color: #666; }
  </style>
</head>
<body>
  <h1>You are offline</h1>
  <p>Please check your internet connection and try again.</p>
  <button onclick="location.reload()">Retry</button>
</body>
</html>
```

### Индикатор офлайн-статуса в React

```jsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return isOnline;
}

function OfflineBanner() {
  const isOnline = useOnlineStatus();

  if (isOnline) return null;

  return (
    <div style={{
      position: "fixed",
      top: 0,
      left: 0,
      right: 0,
      padding: "8px",
      backgroundColor: "#f59e0b",
      color: "white",
      textAlign: "center",
      zIndex: 9999,
    }}>
      You are offline. Some features may not work.
    </div>
  );
}
```

---

## Push-уведомления

### Запрос разрешения

```jsx
async function requestNotificationPermission() {
  if (!("Notification" in window)) {
    console.log("Notifications not supported");
    return;
  }

  const permission = await Notification.requestPermission();

  if (permission === "granted") {
    console.log("Permission granted");
    await subscribeToPush();
  } else if (permission === "denied") {
    console.log("Permission denied");
  }
}
```

### Подписка на push

```jsx
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready;

  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(
      "YOUR_VAPID_PUBLIC_KEY"
    ),
  });

  // Отправить subscription на сервер
  await fetch("/api/push/subscribe", {
    method: "POST",
    body: JSON.stringify(subscription),
  });
}

function urlBase64ToUint8Array(base64String) {
  const padding = "=".repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, "+").replace(/_/g, "/");
  const rawData = window.atob(base64);
  return new Uint8Array([...rawData].map((char) => char.charCodeAt(0)));
}
```

### Обработка push в Service Worker

```js
// sw.js
self.addEventListener("push", (event) => {
  const data = event.data?.json();
  const title = data.title || "New notification";
  const options = {
    body: data.body,
    icon: "/icons/icon-192.png",
    badge: "/icons/badge-72.png",
    data: data.url,
    actions: [
      { action: "open", title: "Open" },
      { action: "close", title: "Close" },
    ],
  };

  event.waitUntil(self.registration.showNotification(title, options));
});

self.addEventListener("notificationclick", (event) => {
  event.notification.close();

  if (event.action === "open" || !event.action) {
    event.waitUntil(
      clients.openWindow(event.notification.data || "/")
    );
  }
});
```

---

## Background Sync

Фоновая синхронизация позволяет отложить действия до восстановления соединения:

```jsx
// Регистрация синхронизации
async function scheduleSync() {
  const registration = await navigator.serviceWorker.ready;

  try {
    await registration.sync.register("sync-messages");
    console.log("Sync registered");
  } catch {
    console.log("Background sync not supported");
  }
}

// В форме
async function sendMessage(message) {
  if (navigator.onLine) {
    await api.sendMessage(message);
  } else {
    await saveToOutbox(message);
    await scheduleSync();
  }
}
```

```js
// sw.js
self.addEventListener("sync", (event) => {
  if (event.tag === "sync-messages") {
    event.waitUntil(syncMessages());
  }
});

async function syncMessages() {
  const messages = await getOutbox();
  for (const message of messages) {
    await fetch("/api/messages", {
      method: "POST",
      body: JSON.stringify(message),
    });
    await removeFromOutbox(message.id);
  }
}
```

---

## PWA в Next.js

### next-pwa

```bash
npm install next-pwa
```

```js
// next.config.js
const withPWA = require("next-pwa")({
  dest: "public",
  register: true,
  skipWaiting: true,
  disable: process.env.NODE_ENV === "development",
});

module.exports = withPWA({
  // ваша конфигурация Next.js
});
```

### Manifest в App Router

```tsx
// app/manifest.ts
import type { MetadataRoute } from "next";

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: "My Next.js App",
    short_name: "MyApp",
    description: "A PWA built with Next.js",
    start_url: "/",
    display: "standalone",
    background_color: "#ffffff",
    theme_color: "#3b82f6",
    icons: [
      { src: "/icons/icon-192.png", sizes: "192x192", type: "image/png" },
      { src: "/icons/icon-512.png", sizes: "512x512", type: "image/png" },
    ],
  };
}
```

### Custom Service Worker в Next.js

```js
// public/sw.js
import { precacheAndRoute } from "workbox-precaching";
import { registerRoute } from "workbox-routing";
import { StaleWhileRevalidate, CacheFirst } from "workbox-strategies";
import { ExpirationPlugin } from "workbox-expiration";

// Precache статических ресурсов (генерируется next-pwa)
precacheAndRoute(self.__WB_MANIFEST);

// API — Stale While Revalidate
registerRoute(
  ({ url }) => url.pathname.startsWith("/api/"),
  new StaleWhileRevalidate({
    cacheName: "api-cache",
  })
);

// Изображения — Cache First
registerRoute(
  ({ request }) => request.destination === "image",
  new CacheFirst({
    cacheName: "image-cache",
    plugins: [
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60,
      }),
    ],
  })
);
```

---

## Установка на устройство

### Кнопка установки

```jsx
function InstallButton() {
  const [deferredPrompt, setDeferredPrompt] = useState(null);
  const [isInstalled, setIsInstalled] = useState(false);

  useEffect(() => {
    const handler = (e) => {
      e.preventDefault();
      setDeferredPrompt(e);
    };

    window.addEventListener("beforeinstallprompt", handler);

    if (window.matchMedia("(display-mode: standalone)").matches) {
      setIsInstalled(true);
    }

    return () => window.removeEventListener("beforeinstallprompt", handler);
  }, []);

  const handleInstall = async () => {
    if (!deferredPrompt) return;

    deferredPrompt.prompt();
    const { outcome } = await deferredPrompt.userChoice;

    if (outcome === "accepted") {
      setIsInstalled(true);
    }

    setDeferredPrompt(null);
  };

  if (isInstalled) return null;
  if (!deferredPrompt) return null;

  return <button onClick={handleInstall}>Install App</button>;
}
```

### iOS (Safari)

iOS не поддерживает `beforeinstallprompt`. Пользователь должен вручную:
1. Открыть «Поделиться» (Share)
2. Выбрать «На экран Домой» (Add to Home Screen)

Можно показать инструкцию:

```jsx
function IOSInstallGuide() {
  const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
  const isStandalone = window.matchMedia("(display-mode: standalone)").matches;
  const [showGuide, setShowGuide] = useState(false);

  if (!isIOS || isStandalone) return null;

  return (
    <div>
      <button onClick={() => setShowGuide(true)}>Install App</button>
      {showGuide && (
        <div className="modal">
          <p>Tap the Share button and select "Add to Home Screen"</p>
          <button onClick={() => setShowGuide(false)}>Close</button>
        </div>
      )}
    </div>
  );
}
```

---

## Тестирование PWA

### Lighthouse

```bash
# Chrome DevTools → Lighthouse → Generate report
# Или через CLI
npx lighthouse https://myapp.com --view
```

Проверяет:
- Performance
- PWA compliance (manifest, service worker, HTTPS)
- Accessibility
- Best Practices
- SEO

### Chrome DevTools → Application

- **Manifest** — просмотр manifest.json
- **Service Workers** — регистрация, обновление, отладка
- **Cache Storage** — просмотр кэшей
- **Background Services** — push, sync, notifications

### Workbox CLI

```bash
npm install -g workbox-cli

workbox wizard
workbox generateSW workbox-config.js
```

---

## Лучшие практики

### 1. Версионирование кэша

```js
const CACHE_VERSION = "v2";
const CACHE_NAME = `my-app-${CACHE_VERSION}`;

self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys().then((names) => {
      return Promise.all(
        names
          .filter((name) => name.startsWith("my-app-") && name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      );
    })
  );
});
```

### 2. Кэширование по размеру

```js
import { ExpirationPlugin } from "workbox-expiration";

registerRoute(
  ({ request }) => request.destination === "image",
  new CacheFirst({
    cacheName: "images",
    plugins: [
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60,
      }),
    ],
  })
);
```

### 3. Обновление Service Worker

```jsx
// Проверка обновлений
navigator.serviceWorker.register("/sw.js").then((registration) => {
  setInterval(() => {
    registration.update();
  }, 60 * 60 * 1000);

  registration.addEventListener("updatefound", () => {
    const newWorker = registration.installing;
    newWorker.addEventListener("statechange", () => {
      if (newWorker.state === "installed" && navigator.serviceWorker.controller) {
        showUpdateNotification();
      }
    });
  });
});

function showUpdateNotification() {
  if (confirm("New version available. Reload?")) {
    window.location.reload();
  }
}
```

---

## Ограничения PWA

### iOS ограничения

- Push-уведомления — только с iOS 16.4+ и при установке на домашний экран
- Background Sync — ограниченная поддержка
- Storage — может быть очищен при нехватке места
- Нет доступа к Bluetooth, NFC, контактам
- Нет badge API (иконка с числом)

### Android ограничения

- Push-уведомления требуют Google Play Services (не работают на Huawei)
- Некоторые API требуют HTTPS (геолокация, камера)

### Когда PWA не подходит

- Нужен App Store для монетизации
- Требуется глубокая интеграция с ОС (виджеты, Siri, Google Assistant)
- Нужны сложные анимации / 3D-графика
- Требуется доступ к специфичным API (ARKit, ARCore)
