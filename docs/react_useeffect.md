# React `useEffect` — Глубокое погружение

## Содержание

1. [Что такое useEffect](#что-такое-useeffect)
2. [Механика работы](#механика-работы)
3. [Массив зависимостей](#массив-зависимостей)
4. [Функция очистки](#функция-очистки)
5. [Типичные сценарии использования](#типичные-сценарии-использования)
6. [Сравнение с useLayoutEffect](#сравнение-с-uselayouteffect)
7. [Сравнение с Vue-аналогами](#сравнение-с-vue-аналогами)
8. [Лучшие практики](#лучшие-практики)
9. [Распространённые ошибки](#распространённые-ошибки)
10. [Антипаттерны](#антипаттерны)
11. [Шпаргалка: useEffect ↔ Vue](#шпаргалка-useeffect--vue)

---

## Что такое useEffect

`useEffect` — хук для выполнения **побочных эффектов** (side effects) в функциональных компонентах React. Побочный эффект — это любое взаимодействие с внешним миром: запрос к API, подписка на события DOM, работа с таймерами, прямое изменение DOM-элемента, логирование, аналитика, интеграция со сторонними библиотеками.

Название отражает суть: **effect** — эффект, который происходит **после** (*use Effect*) того, как React закоммитил изменения в DOM.

```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();

    return () => {
      connection.disconnect();
    };
  }, [roomId]);

  return <h1>Добро пожаловать в комнату {roomId}!</h1>;
}
```

> **Vue-аналог:** комбинация хуков жизненного цикла и вотчеров. `onMounted` + `watch` наиболее близки. Подробное сравнение — в разделе [Сравнение с Vue-аналогами](#сравнение-с-vue-аналогами).

---

## Механика работы

### Когда срабатывает useEffect

React гарантирует, что эффект выполняется **после того, как изменения попали в DOM** — то есть после фазы коммита (commit phase). Это ключевое отличие от фазы рендера (render phase), которая должна быть чистой и не содержать побочных эффектов.

Цикл жизни компонента с `useEffect`:

```
Монтирование:
  1. Вызов компонента (render) → генерация виртуального DOM
  2. React коммитит изменения в реальный DOM
  3. Запуск useEffect → выполнение побочного эффекта
  
Обновление:
  1. Изменение state/props → новый render
  2. React коммитит изменения в реальный DOM
  3. Запуск функции очистки от предыдущего эффекта (если есть)
  4. Запуск useEffect с новыми зависимостями
  
Размонтирование:
  1. Запуск функции очистки
  2. Удаление компонента из DOM
```

### Очерёдность выполнения

Если на странице несколько компонентов со своими `useEffect`, их эффекты выполняются в порядке монтирования компонентов в дереве — сверху вниз. Функции очистки, наоборот, вызываются снизу вверх при размонтировании.

```jsx
function Parent() {
  useEffect(() => {
    console.log("Parent: эффект");
    return () => console.log("Parent: очистка");
  }, []);

  return <Child />;
}

function Child() {
  useEffect(() => {
    console.log("Child: эффект");
    return () => console.log("Child: очистка");
  }, []);

  return <div>Child</div>;
}

// При монтировании:
// "Parent: эффект"
// "Child: эффект"

// При размонтировании:
// "Child: очистка"
// "Parent: очистка"
```

### Строгий режим (StrictMode)

В режиме разработки с `<StrictMode>` React намеренно **дважды** вызывает эффект при монтировании (смонтировать → размонтировать → смонтировать), чтобы выявить проблемы с отсутствующей или некорректной очисткой. В production-режиме эффект выполняется один раз.

```jsx
useEffect(() => {
  // В StrictMode при монтировании этот код выполнится дважды:
  //   Вызов 1: connect() → disconnect() → connect()
  const socket = connect();
  return () => socket.disconnect();
}, []);
```

Именно поэтому функция очистки обязательна для подписок, таймеров и асинхронных операций — ваш код должен корректно пережить двойной вызов.

---

## Массив зависимостей

Второй аргумент `useEffect` — массив зависимостей. React сравнивает каждое значение с предыдущим рендером через `Object.is`. Если хотя бы одно значение изменилось — эффект перезапускается.

### Варианты массива зависимостей

| Массив | Поведение | Аналогия |
|---|---|---|
| `useEffect(() => { ... })` | После **каждого** рендера | Опасно в продакшене — почти всегда ошибка |
| `useEffect(() => { ... }, [])` | Один раз при монтировании | `componentDidMount` |
| `useEffect(() => { ... }, [a, b])` | При изменении `a` или `b` | `componentDidUpdate` с проверкой |
| + `return () => { ... }` | Очистка при размонтировании | `componentWillUnmount` |

### useEffect без массива зависимостей

Эффект выполняется после каждого рендера — включая самый первый. На практике используется крайне редко и обычно является ошибкой:

```jsx
function Broken({ count }) {
  useEffect(() => {
    console.log(`Рендер с count = ${count}`);
    // Этот эффект вызывается после КАЖДОГО рендера
  }); // ← нет массива зависимостей
}
```

### useEffect с пустым массивом `[]`

Гарантирует однократное выполнение при монтировании:

```jsx
function AppInit() {
  useEffect(() => {
    // Выполнится только один раз
    analytics.track("app_opened");
    setupGlobalListeners();

    return () => {
      // Выполнится только при размонтировании
      teardownGlobalListeners();
    };
  }, []); // ← пустой массив — эффект не зависит ни от чего

  return <Main />;
}
```

### useEffect с конкретными зависимостями

Самый распространённый паттерн. Эффект перезапускается только при изменении указанных значений:

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let cancelled = false;

    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) setUser(data);
      });

    return () => {
      cancelled = true;
    };
  }, [userId]); // ← перезапуск только при смене userId

  if (!user) return <Skeleton />;
  return <div>{user.name}</div>;
}
```

### Примитивы vs объекты/массивы в зависимостях

`Object.is` — поверхностное сравнение. Два объекта с одинаковым содержимым, но разными ссылками — **разные** зависимости:

```jsx
function Bad({ options }) {
  useEffect(() => {
    // ❌ options — новый объект на каждом рендере родителя
    applyOptions(options);
  }, [options]); // Эффект перезапускается каждый рендер!
}

// ✅ Исправление: передавайте примитивы или стабильные ссылки
function Good({ fontSize, lineHeight }) {
  useEffect(() => {
    applyOptions({ fontSize, lineHeight });
  }, [fontSize, lineHeight]); // Перезапуск только при изменении примитивов
}
```

### Честные зависимости

Любая переменная из замыкания, используемая внутри эффекта, **обязана** быть в массиве зависимостей. ESLint-плагин `eslint-plugin-react-hooks` с правилом `react-hooks/exhaustive-deps` автоматически проверяет это.

Ложь о зависимостях приводит к **stale closure** — эффект «видит» устаревшие значения из прошлого рендера:

```jsx
function StaleClosure({ step }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // ❌ count всегда равен 0 — значение из первого рендера!
      setCount(count + step);
    }, 1000);

    return () => clearInterval(id);
  }, []); // ESLint предупредит: missing deps: count, step

  return <div>{count}</div>;
}

// ✅ Исправление через функциональный setState
function Fixed({ step }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + step); // Читаем актуальное значение
    }, 1000);

    return () => clearInterval(id);
  }, [step]); // Честная зависимость

  return <div>{count}</div>;
}
```

---

## Функция очистки

Функция, возвращаемая из эффекта — это **cleanup-функция**. React вызывает её в двух случаях:

1. **Перед повторным запуском эффекта** (изменились зависимости).
2. **При размонтировании компонента.**

```jsx
useEffect(() => {
  // Установка
  const controller = new AbortController();
  const { signal } = controller;

  fetch(url, { signal })
    .then(res => res.json())
    .then(setData)
    .catch(err => {
      if (err.name !== "AbortError") setError(err);
    });

  // Очистка
  return () => {
    controller.abort();
  };
}, [url]);
```

### Что обязательно чистить

| Ресурс | Создание | Очистка |
|---|---|---|
| `setInterval` / `setTimeout` | `const id = setInterval(fn, ms)` | `clearInterval(id)` |
| `addEventListener` | `window.addEventListener("resize", fn)` | `window.removeEventListener("resize", fn)` |
| WebSocket / SSE | `const ws = new WebSocket(url)` | `ws.close()` |
| IntersectionObserver | `const io = new IntersectionObserver(fn)` | `io.disconnect()` |
| `fetch` / AbortController | `const ctrl = new AbortController()` | `ctrl.abort()` |
| Подписки (EventEmitter) | `emitter.on("event", fn)` | `emitter.off("event", fn)` |
| Прямые DOM-мутации | `el.style.color = "red"` | `el.style.color = ""` (восстановить) |

### Очистка не нужна

Некоторым эффектам не нужна очистка — например, отправка аналитики, логирование или однократная настройка глобального состояния:

```jsx
useEffect(() => {
  analytics.track("page_view", { page: pathname });
}, [pathname]);
// Нет возврата cleanup-функции — это нормально
```

---

## Типичные сценарии использования

### 1. Подписка на внешний источник данных

```jsx
function StockTicker({ symbol }) {
  const [price, setPrice] = useState(null);

  useEffect(() => {
    const ws = new WebSocket(`wss://stocks.example.com/${symbol}`);

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setPrice(data.price);
    };

    ws.onerror = (err) => {
      console.error("WebSocket error", err);
    };

    return () => {
      ws.close();
    };
  }, [symbol]);

  return <div>{symbol}: ${price ?? "..."}</div>;
}
```

### 2. Синхронизация с DOM API (классы, атрибуты, фокус)

```jsx
function Modal({ isOpen, children }) {
  const dialogRef = useRef(null);

  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;

    if (isOpen) {
      dialog.showModal();
      document.body.style.overflow = "hidden";
    } else {
      dialog.close();
      document.body.style.overflow = "";
    }

    return () => {
      document.body.style.overflow = "";
    };
  }, [isOpen]);

  return <dialog ref={dialogRef}>{children}</dialog>;
}
```

### 3. Загрузка данных (fetch)

```jsx
function ProductPage({ productId }) {
  const [product, setProduct] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    const controller = new AbortController();

    setLoading(true);
    setError(null);

    fetch(`/api/products/${productId}`, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        if (!cancelled) {
          setProduct(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled && err.name !== "AbortError") {
          setError(err);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true;
      controller.abort();
    };
  }, [productId]);

  if (loading) return <Skeleton />;
  if (error) return <ErrorBanner error={error} />;
  return <ProductDetail product={product} />;
}
```

> **Примечание:** В 2026 году для загрузки данных рекомендуется использовать **TanStack Query** (ранее React Query) или хук `use()` из React 19 + `<Suspense>`, а не `useEffect` + `useState`. Подробнее: [TanStack Query](./tanstack_query.md) и [React `use()`](./react_use.md).

### 4. Интеграция со сторонними библиотеками

```jsx
function Chart({ data, width, height }) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);

  useEffect(() => {
    if (!containerRef.current) return;

    // Инициализация D3 / Chart.js / Leaflet
    const chart = new ThirdPartyChart(containerRef.current, {
      width,
      height,
    });

    chart.render(data);
    chartRef.current = chart;

    return () => {
      chart.destroy();
      chartRef.current = null;
    };
  }, [data, width, height]);

  return <div ref={containerRef} />;
}
```

### 5. Обработка глобальных событий (клавиатура, resize, scroll)

```jsx
function useKeyPress(targetKey) {
  const [isPressed, setIsPressed] = useState(false);

  useEffect(() => {
    const handleDown = (e) => {
      if (e.key === targetKey) setIsPressed(true);
    };
    const handleUp = (e) => {
      if (e.key === targetKey) setIsPressed(false);
    };

    window.addEventListener("keydown", handleDown);
    window.addEventListener("keyup", handleUp);

    return () => {
      window.removeEventListener("keydown", handleDown);
      window.removeEventListener("keyup", handleUp);
    };
  }, [targetKey]);

  return isPressed;
}
```

### 6. Установка и очистка интервалов

```jsx
function Countdown({ from }) {
  const [count, setCount] = useState(from);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => {
        if (c <= 1) {
          clearInterval(id);
          return 0;
        }
        return c - 1;
      });
    }, 1000);

    return () => clearInterval(id);
  }, [from]);

  return <div>{count}</div>;
}
```

---

## Сравнение с useLayoutEffect

`useLayoutEffect` — менее известный «брат» `useEffect`. Разница — в моменте выполнения:

| | `useEffect` | `useLayoutEffect` |
|---|---|---|
| Момент выполнения | **После** отрисовки в браузере (асинхронно) | **До** отрисовки в браузере (синхронно, блокирует) |
| Пользователь видит | Промежуточное состояние → эффект → перерисовка | Сразу финальное состояние |
| Для чего | Подавляющее большинство эффектов | Измерения DOM, синхронные мутации DOM |
| Производительность | Не блокирует отрисовку | Блокирует — используйте редко |

```jsx
function Tooltip({ x, y, content }) {
  const tooltipRef = useRef(null);
  const [adjustedY, setAdjustedY] = useState(y);

  // ❌ useEffect: пользователь увидит моргание — сначала старый y, потом скорректированный
  // ✅ useLayoutEffect: коррекция ДО отрисовки, моргания нет
  useLayoutEffect(() => {
    if (!tooltipRef.current) return;
    const { height } = tooltipRef.current.getBoundingClientRect();
    if (y + height > window.innerHeight) {
      setAdjustedY(window.innerHeight - height - 10);
    } else {
      setAdjustedY(y);
    }
  }, [y]);

  return (
    <div ref={tooltipRef} style={{ position: "fixed", left: x, top: adjustedY }}>
      {content}
    </div>
  );
}
```

> **Правило:** всегда начинайте с `useEffect`. Если видите визуальное моргание или неправильные замеры — переходите на `useLayoutEffect`.

> **Vue-аналог:** `useEffect` ≈ `watch` / `watchEffect` с `nextTick` (после обновления DOM). `useLayoutEffect` ≈ `watch` с `flush: "sync"` или `onMounted` / `onUpdated` — вызываются синхронно после мутации DOM, но до отрисовки браузером.

---

## Сравнение с Vue-аналогами

В Vue нет единого хука, идентичного `useEffect`. Функциональность `useEffect` распределена между несколькими API Composition API:

### 1. `onMounted` + `onUnmounted` — аналог `useEffect(fn, [])`

```ts
// React: однократное выполнение с очисткой
useEffect(() => {
  const ws = connect(url);
  return () => ws.close();
}, []);

// Vue: отдельные хуки жизненного цикла
import { onMounted, onUnmounted } from "vue";

let ws;
onMounted(() => {
  ws = connect(url);
});
onUnmounted(() => {
  ws?.close();
});
```

### 2. `watchEffect` — аналог `useEffect` с авто-отслеживанием зависимостей

```ts
// React: явный массив зависимостей
useEffect(() => {
  console.log(`Count изменился: ${count}`);
}, [count]);

// Vue: автоматическое отслеживание
watchEffect(() => {
  console.log(`Count изменился: ${count.value}`);
  // Vue сам понимает, что эффект зависит от count
});
```

`watchEffect` немедленно выполняет эффект и автоматически отслеживает все реактивные значения, использованные внутри. Это ближайший аналог React-эффекта **без необходимости вручную указывать зависимости**.

### 3. `watch` — аналог `useEffect` с явными зависимостями

```ts
// React
useEffect(() => {
  fetchUser(userId, role);
}, [userId, role]);

// Vue
watch(
  [() => userId.value, () => role.value],
  ([newUserId, newRole]) => {
    fetchUser(newUserId, newRole);
  }
);
```

### 4. Cleanup-функция

```ts
// React: возврат cleanup-функции
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);

// Vue: onCleanup внутри watchEffect / watch
watchEffect((onCleanup) => {
  const id = setInterval(tick, 1000);
  onCleanup(() => clearInterval(id));
});
```

### 5. `onUpdated` — аналог `useEffect` без зависимостей

```ts
// React: эффект после каждого рендера
useEffect(() => {
  console.log("Компонент обновился");
});

// Vue
onUpdated(() => {
  console.log("Компонент обновился");
});
```

### Сводная таблица соответствий

| React | Vue (Composition API) |
|---|---|
| `useEffect(fn, [])` — монтирование | `onMounted(fn)` |
| `return () => cleanup()` внутри `useEffect` | `onUnmounted(fn)` |
| `useEffect(fn, [a, b])` — по зависимостям | `watch([a, b], fn)` |
| `useEffect(fn)` — после каждого рендера | `onUpdated(fn)` |
| `useEffect(fn)` с авто-отслеживанием | `watchEffect(fn)` |
| Cleanup-функция | `onCleanup(fn)` — переданная в `watch`/`watchEffect` |
| `useLayoutEffect(fn, deps)` | `watch(fn, { flush: "sync" })` |

> **Ключевое архитектурное различие:** React требует явного массива зависимостей для `useEffect`. Vue, благодаря системе реактивности на Proxy, автоматически собирает зависимости внутри `watchEffect`, что исключает ошибки пропущенных или лишних зависимостей. С другой стороны, явный массив в React даёт более предсказуемое поведение и упрощает дебаггинг — вы всегда видите, от чего зависит эффект.

---

## Лучшие практики

### 1. Каждый эффект — одна ответственность

Не смешивайте несвязанную логику в одном `useEffect`. Лучше несколько узконаправленных эффектов:

```jsx
// ❌ Смешанная логика
useEffect(() => {
  analytics.track("page_view");
  document.title = `${user.name} — Dashboard`;
  const id = setInterval(pollNotifications, 5000);
  return () => clearInterval(id);
}, [user]);

// ✅ Раздельные эффекты
useEffect(() => {
  analytics.track("page_view", { page: pathname });
}, [pathname]);

useEffect(() => {
  document.title = `${user.name} — Dashboard`;
}, [user.name]);

useEffect(() => {
  const id = setInterval(pollNotifications, 5000);
  return () => clearInterval(id);
}, []);
```

### 2. Выносите логику в пользовательские хуки

```jsx
function useDocumentTitle(title) {
  useEffect(() => {
    const prev = document.title;
    document.title = title;
    return () => {
      document.title = prev;
    };
  }, [title]);
}

function useInterval(callback, delay) {
  const savedCallback = useRef(callback);

  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return;
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

### 3. Всегда предусматривайте отмену асинхронных операций

```jsx
useEffect(() => {
  let cancelled = false;

  fetchData(id).then(data => {
    if (!cancelled) setData(data);
  });

  return () => {
    cancelled = true;
  };
}, [id]);
```

### 4. Используйте функциональный setState внутри эффектов

Когда новое состояние зависит от предыдущего — чтобы избежать `count` в зависимостях:

```jsx
// ❌ count в зависимостях — эффект перезапускается при каждом изменении
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);

// ✅ Функциональный setState — count не нужен в зависимостях
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

### 5. Предпочитайте `use()` + `<Suspense>` для загрузки данных

В React 19 классический паттерн `useEffect` + `useState` для загрузки данных устарел. Современная альтернатива:

```jsx
// ❌ Устаревший подход (React 16–18)
function Profile({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch(`/api/user/${userId}`).then(r => r.json()).then(setUser);
  }, [userId]);
  if (!user) return <Loading />;
  return <div>{user.name}</div>;
}

// ✅ Современный подход (React 19+)
function Profile({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}
```

Подробнее — [React `use()` — Полное руководство](./react_use.md).

### 6. Не бойтесь нескольких useEffect

Больше эффектов = читаемее код. React группирует обновления от нескольких `setState` внутри одного эффекта в один батч, так что дополнительные рендеры не возникают.

---

## Распространённые ошибки

### 1. Бесконечный цикл ререндеров

Эффект обновляет состояние, которое указано в его зависимостях — и так по кругу:

```jsx
// ❌ Бесконечный цикл
useEffect(() => {
  setData(transform(data));  // data изменяется → эффект запускается → data изменяется → ...
}, [data]);

// ✅ Решение: вычислять transform в рендере, а не в эффекте
const transformedData = useMemo(() => transform(rawData), [rawData]);
```

### 2. Stale closure (устаревшее замыкание)

```jsx
function Bad({ userId }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(`User ${userId}, count ${count}`);
      // userId и count — значения на момент создания эффекта!
    }, 1000);
    return () => clearInterval(id);
  }, []); // ← пустой массив, хотя userId и count используются внутри
}
```

### 3. Пропуск функции очистки

```jsx
// ❌ Утечка: интервал продолжает тикать после размонтирования
useEffect(() => {
  setInterval(() => {
    setCount(c => c + 1);
  }, 1000);
}, []);

// ✅ Очистка
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### 4. Объект как зависимость

```jsx
function Bad() {
  const [options, setOptions] = useState({ page: 1, size: 10 });

  useEffect(() => {
    fetchPage(options.page, options.size);
  }, [options]); // ❌ options — новый объект при каждом setOptions
}

// ✅ Примитивы
function Good() {
  const [page, setPage] = useState(1);
  const [size, setSize] = useState(10);

  useEffect(() => {
    fetchPage(page, size);
  }, [page, size]); // ✅ Сравнение Object.is примитивов
}
```

### 5. useEffect там, где эффект не нужен

Не всякая логика, вызываемая «при изменении», должна идти в `useEffect`:

```jsx
// ❌ Избыточный эффект — фильтрацию можно сделать прямо в рендере
function Bad({ items, query }) {
  const [filtered, setFiltered] = useState([]);

  useEffect(() => {
    setFiltered(items.filter(i => i.name.includes(query)));
  }, [items, query]);

  return <List items={filtered} />;
}

// ✅ Производные данные вычисляются в рендере (или через useMemo для оптимизации)
function Good({ items, query }) {
  const filtered = useMemo(
    () => items.filter(i => i.name.includes(query)),
    [items, query]
  );

  return <List items={filtered} />;
}
```

### 6. Асинхронная функция как колбэк useEffect

```jsx
// ❌ Не работает как ожидается — async-функция возвращает Promise, не cleanup
useEffect(async () => {
  const data = await fetchData();
  setData(data);
}, []);

// ✅ Правильно: определить async-функцию внутри, вызвать её
useEffect(() => {
  async function load() {
    const data = await fetchData();
    setData(data);
  }
  load();
}, []);
```

---

## Антипаттерны

### ❌ Огромный эффект со всей логикой компонента

```jsx
// ❌ 50+ строк в одном эффекте
useEffect(() => {
  // Загрузка данных
  // Подписка на события
  // Аналитика
  // Таймеры
  // Обновление document.title
  // ...
}, [dep1, dep2, dep3, dep4, dep5]);
```

### ❌ Подавление ESLint-правила `exhaustive-deps`

```jsx
// ❌ Отключение предупреждения — верный путь к stale closure
useEffect(() => {
  doSomething(count, userId);
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []);
```

### ❌ Использование useEffect только для запуска setState

```jsx
// ❌ Эффект, который просто синхронизирует пропс в состояние
function Bad({ initialValue }) {
  const [value, setValue] = useState(initialValue);

  useEffect(() => {
    setValue(initialValue);
  }, [initialValue]);
}

// ✅ Либо используйте пропс напрямую, либо key для сброса
function Good({ initialValue }) {
  const [value, setValue] = useState(initialValue);
  // Если initialValue меняется — используйте key на родителе для ремаунта
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

// Альтернативный родитель:
<ControlledInput key={userId} initialValue={fetchDefault(userId)} />
```

### ❌ Создание Promise прямо в эффекте без отмены

```jsx
// ❌ Race condition: если userId меняется быстро, старый запрос может перезаписать новый
useEffect(() => {
  fetchUser(userId).then(setUser);
}, [userId]);

// ✅ AbortController или флаг cancelled
useEffect(() => {
  const controller = new AbortController();
  fetchUser(userId, { signal: controller.signal })
    .then(setUser)
    .catch(err => {
      if (err.name !== "AbortError") throw err;
    });
  return () => controller.abort();
}, [userId]);
```

---

## Шпаргалка: useEffect ↔ Vue

| Ситуация | React (`useEffect`) | Vue (Composition API) |
|---|---|---|
| Выполнить один раз при монтировании | `useEffect(fn, [])` | `onMounted(fn)` |
| Очистка при размонтировании | `return () => cleanup()` | `onUnmounted(cleanup)` |
| Реакция на изменение `a` и `b` | `useEffect(fn, [a, b])` | `watch([a, b], fn)` |
| Автоматическое отслеживание зависимостей | Нет аналога (`useEffectEvent` — другое) | `watchEffect(fn)` |
| Выполнить перед каждым рендером | `useLayoutEffect(fn, deps)` | `watch(fn, { flush: "sync" })` |
| Очистка внутри watch | `return () => cleanup()` | `onCleanup(() => cleanup())` |
| Несколько эффектов | Несколько `useEffect` (рекомендуется) | Несколько `watch` / `onMounted` |
| Отказ от эффекта для загрузки данных | `use(promise)` + `<Suspense>` | `await` + `<Suspense>` |
| Отмена fetch при размонтировании | `AbortController` в cleanup | `AbortController` в `onCleanup` |
| StrictMode (двойной вызов в dev) | Да, встроен в React 18+ | Нет аналога |

> **Главный вывод:** React-разработчику, переходящему на Vue, нужно привыкнуть к автоматическому отслеживанию зависимостей в `watchEffect` и разделению логики монтирования/размонтирования на отдельные хуки `onMounted`/`onUnmounted`. Vue-разработчику, переходящему на React, — к явному массиву зависимостей и единому `useEffect` вместо разрозненных хуков жизненного цикла. Обе модели эквивалентны по возможностям, разница — в эргономике и подверженности ошибкам: React легче дебажить (видно все зависимости), Vue сложнее забыть зависимость (она отслеживается автоматически).
