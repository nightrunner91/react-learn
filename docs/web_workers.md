# Web Workers — вынос тяжёлых вычислений из основного потока

Web Workers — это механизм, позволяющий выполнять JavaScript-код в **фоновом потоке**, не блокируя основной поток, в котором работает UI. Когда приложение выполняет ресурсоёмкие вычисления (обработка изображений, парсинг больших JSON, шифрование, сложная фильтрация), основной поток "замирает" — интерфейс не реагирует на клики, скролл, ввод. Web Workers решают эту проблему, вынося вычисления в отдельный поток.

---

## Почему это важно: однопоточность JavaScript

JavaScript в браузере выполняется в **одном потоке** (main thread). Это значит:

- Только одна операция выполняется в любой момент времени
- Долгая операция блокирует всё: рендеринг, обработку событий, анимации
- Интерфейс "замирает" до завершения операции

```js
// ❌ Блокирует UI на 2 секунды
function handleButtonClick() {
  const result = heavyComputation(); // 2 секунды
  updateUI(result);
}
// Пользователь не может кликнуть, скроллить, ввести текст 2 секунды
```

**Важно:** `useTransition` и `useDeferredValue` **не решают** эту проблему. Они лишь перераспределяют приоритеты обновлений, но вычисления всё равно выполняются в основном потоке. Если фильтрация занимает 500мс, она заблокирует поток на 500мс — просто React отложит её до лучших времён.

Web Workers — единственный способ **физически** вынести вычисления из основного потока.

---

## Базовый синтаксис Web Worker

### Создание воркера

```js
// main.js — основной поток
const worker = new Worker('worker.js');

// Отправка данных в воркер
worker.postMessage({ type: 'compute', data: largeArray });

// Получение результата из воркера
worker.onmessage = (event) => {
  console.log('Результат:', event.data);
};

// Обработка ошибок
worker.onerror = (error) => {
  console.error('Ошибка в воркере:', error);
};
```

```js
// worker.js — фоновый поток
self.onmessage = (event) => {
  const { type, data } = event.data;
  
  if (type === 'compute') {
    const result = heavyComputation(data);
    self.postMessage({ type: 'result', data: result });
  }
};

function heavyComputation(data) {
  // Тяжёлые вычисления: фильтрация, сортировка, парсинг
  return data.filter(item => item.value > 100).sort((a, b) => b.value - a.value);
}
```

### Ключевые особенности

**Изолированная среда выполнения:**
- Воркер не имеет доступа к `window`, `document`, DOM
- Нет прямого доступа к React, состоянию, компонентам
- Общение с основным потоком **только через сообщения** (`postMessage`)

**Передача данных:**
- Данные **копируются** (structured clone algorithm) — не разделяются между потоками
- Большие объекты (ArrayBuffer, Blob) можно передавать без копирования через **transferable objects**
- Для сложных структур данных используйте `MessageChannel` или библиотеки

```js
// Передача ArrayBuffer без копирования (быстрее)
const buffer = new ArrayBuffer(1024 * 1024); // 1 MB
worker.postMessage(buffer, [buffer]); // buffer передаётся, в основном потоке он больше недоступен
```

---

## Интеграция с React

### Паттерн 1: Хук useWorker

Инкапсулируем работу с воркером в переиспользуемый хук:

```jsx
// useWorker.js
import { useState, useEffect, useRef, useCallback } from 'react';

export function useWorker(workerUrl) {
  const [result, setResult] = useState(null);
  const [isComputing, setIsComputing] = useState(false);
  const workerRef = useRef(null);

  useEffect(() => {
    workerRef.current = new Worker(workerUrl);
    
    workerRef.current.onmessage = (event) => {
      setResult(event.data);
      setIsComputing(false);
    };

    return () => {
      workerRef.current?.terminate();
    };
  }, [workerUrl]);

  const postMessage = useCallback((message) => {
    if (workerRef.current) {
      setIsComputing(true);
      workerRef.current.postMessage(message);
    }
  }, []);

  return { result, isComputing, postMessage };
}
```

```jsx
// Компонент с фильтрацией в воркере
function SearchableList({ items }) {
  const [query, setQuery] = useState('');
  const { result: filteredItems, isComputing, postMessage } = useWorker('/filter.worker.js');

  useEffect(() => {
    if (query) {
      postMessage({ query, items });
    }
  }, [query, items, postMessage]);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Поиск..."
      />
      
      {isComputing && <Spinner />}
      
      <ul>
        {(filteredItems || []).map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

```js
// filter.worker.js
self.onmessage = (event) => {
  const { query, items } = event.data;
  
  const filtered = items.filter(item =>
    item.name.toLowerCase().includes(query.toLowerCase())
  );
  
  self.postMessage(filtered);
};
```

### Паттерн 2: Комбинация с useTransition для UX

```jsx
function AdvancedSearch({ items }) {
  const [query, setQuery] = useState('');
  const [immediateResults, setImmediateResults] = useState(items);
  const [workerResults, setWorkerResults] = useState(items);
  const [isPending, startTransition] = useTransition();
  const workerRef = useRef(null);

  useEffect(() => {
    workerRef.current = new Worker('/filter.worker.js');
    workerRef.current.onmessage = (e) => {
      startTransition(() => {
        setWorkerResults(e.data);
      });
    };
    return () => workerRef.current?.terminate();
  }, []);

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);

    // Быстрая фильтрация на клиенте для мгновенной обратной связи
    const quickFiltered = items.filter(item =>
      item.name.toLowerCase().includes(value.toLowerCase())
    ).slice(0, 50); // только первые 50
    setImmediateResults(quickFiltered);

    // Полная фильтрация в воркере
    workerRef.current?.postMessage({ query: value, items });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      
      {isPending ? (
        <>
          {/* Показываем быстрые результаты, пока воркер думает */}
          <QuickResults items={immediateResults} />
          <Spinner />
        </>
      ) : (
        <FullResults items={workerResults} />
      )}
    </div>
  );
}
```

### Паттерн 3: Пул воркеров для параллельных вычислений

```jsx
// useWorkerPool.js
import { useRef, useCallback } from 'react';

export function useWorkerPool(workerUrl, poolSize = 4) {
  const workersRef = useRef([]);
  const queueRef = useRef([]);

  // Инициализация пула
  if (workersRef.current.length === 0) {
    workersRef.current = Array.from({ length: poolSize }, () => {
      const worker = new Worker(workerUrl);
      worker.isBusy = false;
      return worker;
    });
  }

  const getFreeWorker = useCallback(() => {
    return workersRef.current.find(w => !w.isBusy);
  }, []);

  const postTask = useCallback((message) => {
    return new Promise((resolve) => {
      const worker = getFreeWorker();
      
      if (worker) {
        worker.isBusy = true;
        worker.onmessage = (e) => {
          worker.isBusy = false;
          resolve(e.data);
          
          // Обработка очереди
          const nextTask = queueRef.current.shift();
          if (nextTask) nextTask();
        };
        worker.postMessage(message);
      } else {
        // Все воркеры заняты — добавляем в очередь
        queueRef.current.push(() => postTask(message).then(resolve));
      }
    });
  }, [getFreeWorker]);

  return { postTask };
}
```

```jsx
// Использование: параллельная обработка изображений
function ImageProcessor({ images }) {
  const { postTask } = useWorkerPool('/image.worker.js', 4);
  const [processedImages, setProcessedImages] = useState([]);

  const processAll = async () => {
    const results = await Promise.all(
      images.map(img => postTask({ image: img, operation: 'resize' }))
    );
    setProcessedImages(results);
  };

  return (
    <button onClick={processAll}>
      Обработать {images.length} изображений
    </button>
  );
}
```

---

## Inline Workers: воркеры без отдельного файла

Создание воркера из Blob позволяет избежать отдельного файла:

```jsx
function InlineWorkerExample() {
  const workerRef = useRef(null);

  useEffect(() => {
    const workerCode = `
      self.onmessage = (e) => {
        const result = e.data * 2;
        self.postMessage(result);
      };
    `;
    
    const blob = new Blob([workerCode], { type: 'application/javascript' });
    const workerUrl = URL.createObjectURL(blob);
    workerRef.current = new Worker(workerUrl);

    workerRef.current.onmessage = (e) => {
      console.log('Результат:', e.data);
    };

    return () => {
      workerRef.current?.terminate();
      URL.revokeObjectURL(workerUrl);
    };
  }, []);

  const sendToWorker = () => {
    workerRef.current?.postMessage(21);
  };

  return <button onClick={sendToWorker}>Отправить в воркер</button>;
}
```

**Преимущества:**
- Весь код в одном файле
- Удобно для небольших воркеров
- Можно генерировать код динамически

**Недостатки:**
- Нет подсветки синтаксиса в IDE
- Сложнее отлаживать
- Не подходит для больших воркеров

---

## Shared Workers: один воркер для нескольких вкладок

`SharedWorker` позволяет нескольким вкладкам/окнам одного происхождения общаться с одним воркером:

```js
// main.js — вкладка 1
const sharedWorker = new SharedWorker('shared.worker.js');
sharedWorker.port.onmessage = (e) => {
  console.log('Получено:', e.data);
};
sharedWorker.port.postMessage('Привет от вкладки 1');
```

```js
// shared.worker.js
const ports = new Set();

self.onconnect = (e) => {
  const port = e.ports[0];
  ports.add(port);
  
  port.onmessage = (e) => {
    // Рассылаем сообщение всем подключённым вкладкам
    ports.forEach(p => {
      if (p !== port) {
        p.postMessage(`Получено от другой вкладки: ${e.data}`);
      }
    });
  };
  
  port.start();
};
```

**Типичные сценарии:**
- Синхронизация состояния между вкладками
- Общий WebSocket-коннект
- Кэширование данных для нескольких вкладок

---

## Service Workers: воркеры для сетевых запросов

`ServiceWorker` — специальный тип воркера для перехвата сетевых запросов, push-уведомлений, фоновой синхронизации. Это основа **Progressive Web Apps (PWA)**:

```js
// registration.js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/service-worker.js')
    .then(reg => console.log('Service Worker зарегистрирован'))
    .catch(err => console.error('Ошибка регистрации', err));
}
```

```js
// service-worker.js
self.addEventListener('install', (event) => {
  console.log('Service Worker установлен');
});

self.addEventListener('fetch', (event) => {
  // Перехват сетевых запросов
  event.respondWith(
    caches.match(event.request).then(cached => {
      return cached || fetch(event.request);
    })
  );
});
```

**Отличие от Web Workers:**
- Service Worker работает даже когда вкладка закрыта
- Не имеет доступа к DOM
- Используется для офлайн-режима, кэширования, push-уведомлений
- Web Worker — для вычислений, пока вкладка открыта

---

## Ограничения и подводные камни

### Нет доступа к DOM

```js
// ❌ Не работает в воркере
self.onmessage = (e) => {
  document.getElementById('result').textContent = e.data; // TypeError
};

// ✅ Отправляем результат в основной поток
self.onmessage = (e) => {
  self.postMessage({ result: e.data });
};
```

### Передача данных копируется

```js
// ❌ Медленно: большой объект копируется
const largeObject = { data: new Array(1000000).fill(0) };
worker.postMessage(largeObject); // Копирование ~8 MB

// ✅ Быстро: передаём ArrayBuffer по ссылке
const buffer = new ArrayBuffer(8 * 1000000);
worker.postMessage(buffer, [buffer]); // Передача без копирования
```

### Воркер не может быть сериализован

```js
// ❌ Не работает
const worker = new Worker('worker.js');
postMessage(worker); // DataCloneError

// ✅ Передаём только данные, не сам воркер
```

### Ограничения в некоторых окружениях

- **IE11:** не поддерживает (используйте полифилл или fallback)
- **Node.js:** использует `worker_threads` вместо Web Workers
- **React Native:** использует `@react-native-community/async-storage` или `react-native-async-storage`

---

## Когда использовать Web Workers

**Используйте Web Workers для:**
- Обработки больших массивов (фильтрация, сортировка, агрегация)
- Парсинга/сериализации больших JSON
- Обработки изображений (canvas, resize, filters)
- Шифрования/дешифрования
- Вычислений с числами (матрицы, графы, симуляции)
- Валидации больших форм
- Генерации отчётов

**НЕ используйте Web Workers для:**
- Простых вычислений (< 100мс) — накладные расходы на коммуникацию превысят выгоду
- Работы с DOM — воркер не имеет к нему доступа
- Асинхронных операций (fetch, setTimeout) — они и так не блокируют поток
- UI-логики (обработка событий, анимации) — используйте основной поток

---

## Инструменты и библиотеки

### comlink (от Google)

Упрощает общение с воркерами, делая их похожими на обычные объекты:

```js
// worker.js
import { expose } from 'comlink';

const api = {
  multiply(a, b) {
    return a * b;
  },
  async fetchData(url) {
    const response = await fetch(url);
    return response.json();
  }
};

expose(api);
```

```jsx
// main.js
import { wrap } from 'comlink';

const worker = new Worker('worker.js');
const api = wrap(worker);

const result = await api.multiply(6, 7); // 42
const data = await api.fetchData('/api/data');
```

**Преимущества:**
- Прозрачная работа с воркером как с обычным объектом
- Поддержка async/await
- Автоматическая сериализация/десериализация

### workerize-loader (Webpack)

Позволяет импортировать модуль как воркер:

```js
// worker.js
export function heavyComputation(data) {
  return data.filter(item => item.value > 100);
}
```

```jsx
// main.js
import Worker from 'workerize-loader!./worker.js';

const worker = new Worker();
worker.heavyComputation(data).then(result => {
  console.log(result);
});
```

---

## Отладка Web Workers

### Chrome DevTools

1. **Sources → Workers:** показывает все активные воркеры
2. **Console:** переключайтесь между контекстами (main, worker)
3. **Breakpoints:** можно ставить брейкпоинты в коде воркера
4. **Network:** запросы воркера отображаются отдельно

### Логирование

```js
// worker.js
self.onmessage = (e) => {
  console.log('[Worker] Получено:', e.data);
  const result = process(e.data);
  console.log('[Worker] Отправляю:', result);
  self.postMessage(result);
};
```

Логи воркера отображаются в консоли браузера с пометкой `[Worker]`.

---

## Сравнение: Web Workers vs useTransition vs requestIdleCallback

| Характеристика | Web Workers | useTransition | requestIdleCallback |
|----------------|-------------|---------------|---------------------|
| **Поток выполнения** | Отдельный поток | Основной поток | Основной поток |
| **Блокирует UI** | Нет | Может (если вычисление длинное) | Нет (только в idle) |
| **Доступ к DOM** | Нет | Да | Да |
| **Сложность** | Высокая | Низкая | Средняя |
| **Для чего** | Тяжёлые вычисления | Перераспределение приоритетов | Фоновые задачи |
| **Прерываемость** | Нет (воркер работает до конца) | Да (React прерывает) | Да (браузер прерывает) |

**Комбинированный подход:**

```jsx
function OptimizedSearch({ items }) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState(items);
  const [isPending, startTransition] = useTransition();
  const workerRef = useRef(null);

  useEffect(() => {
    workerRef.current = new Worker('/filter.worker.js');
    workerRef.current.onmessage = (e) => {
      startTransition(() => {
        setResults(e.data);
      });
    };
    return () => workerRef.current?.terminate();
  }, []);

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);
    
    // Выносим тяжёлую фильтрацию в воркер
    workerRef.current?.postMessage({ query: value, items });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList items={results} />
    </div>
  );
}
```

Здесь:
- **Web Worker** физически выносит вычисления из основного потока
- **useTransition** помечает обновление результатов как несрочное
- **Итог:** интерфейс отзывчивый, даже при очень тяжёлой фильтрации

---

## Когда Web Workers действительно нужны (а когда — over engineering)

В 99% фронтенд-приложений Web Workers — избыточны. CRUD-формы, списки, интернет-магазины, дашборды с десятком графиков — всё это отлично работает без них. Используй `useMemo`, `useTransition`, виртуализацию.

### Реальные сценарии, где без воркеров не обойтись

**Графические редакторы (Figma, Canva):** применение фильтра к изображению 4K — это ~200 миллионов операций с пикселями. В основном потоке UI замрёт на 2-5 секунд.

**Таблицы с формулами (Google Sheets):** пересчёт 500 000 ячеек с графом зависимостей — 1-3 секунды блокировки. Без воркера нельзя кликнуть или скроллить.

**IDE в браузере (VS Code Web):** TypeScript-компилятор (~30 MB) + линтинг 50 000 строк — 5-10 секунд. Печатать без воркеров невозможно.

**Трейдинговые платформы:** 1000 акций × расчёт индикаторов (RSI, MACD) каждые 100мс = 3 миллиона операций/сек. Основной поток не справляется, FPS падает.

**Видео/аудио обработка (FFmpeg.wasm):** перекодирование 5-минутного видео — 30-120 секунд. Без воркера вкладка зависнет или упадёт.

### Правило большого пальца

Используй Web Workers, если:
- Вычисление занимает > 100мс и блокирует UI
- Работаешь с данными > 10 MB
- Нужна параллельная обработка нескольких задач
- Профайлер показывает блокировку main thread > 50мс

**Не нужны для:** CRUD, формы, списки до 1000 строк, интернет-магазины, блоги, чаты, админки, дашборды с 10-20 графиками.

---

## Итог

Web Workers — мощный инструмент для выноса тяжёлых вычислений из основного потока. Они позволяют создать по-настоящему отзывчивый интерфейс, даже когда приложение обрабатывает большие объёмы данных.

**Ключевые принципы:**
- Используйте воркеры для вычислений > 100мс
- Общение с воркером только через `postMessage`
- Комбинируйте с `useTransition` для оптимального UX
- Используйте библиотеки (comlink, workerize-loader) для упрощения кода
- Не забывайте про отладку в DevTools

**Помните:** `useTransition` и `useDeferredValue` не заменяют Web Workers. Они решают разные задачи: React-хуки перераспределяют приоритеты обновлений, а воркеры физически выносят вычисления в отдельный поток. Для максимальной производительности используйте оба подхода вместе.
