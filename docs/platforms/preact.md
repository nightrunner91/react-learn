# Preact — лёгкий React-альтернативный рантайм

## Содержание

1. [Что такое Preact](#что-такое-preact)
2. [Preact vs React](#preact-vs-react)
3. [Совместимость и миграция](#совместимость-и-миграция)
4. [Preact Signals](#preact-signals)
5. [Когда выбирать Preact](#когда-выбирать-preact)
6. [Когда оставить React](#когда-оставить-react)

---

## Что такое Preact

**Preact** — это быстрая и лёгкая альтернатива React с тем же API. Она реализует Virtual DOM, хуки, компоненты и JSX, занимая при этом значительно меньше места в бандле.

```
React + ReactDOM  ~ 40-50 kB (gzip)
Preact (compat)   ~ 10-12 kB (gzip)
Preact (core)     ~  3-4 kB (gzip)
```

Preact создан для сценариев, где важен размер бандла, скорость загрузки и производительность на слабых устройствах, но при этом нужна привычная экосистема React.

---

## Preact vs React

### Базовое сравнение

| | React | Preact |
|---|---|---|
| Размер | ~40-50 kB gzip | ~3-4 kB core / ~10-12 kB compat |
| API | Полный API React | Совместимый с React, меньше внутренних фич |
| Virtual DOM | Да | Да, упрощённая реализация |
| Хуки | `useState`, `useEffect`, и др. | Полная поддержка хуков |
| JSX | `React.createElement` / `jsx-runtime` | `h()` / `jsx-runtime` |
| DevTools | React DevTools | Preact DevTools |
| Concurrent features | Полная поддержка | Ограниченная / отсутствует |

### Ключевые отличия внутри

**1. Размер за счёт упрощений**

Preact убирает часть внутренних абстракций React: нет Fiber, нет приоритетов обновлений, нет Concurrent Mode. Это делает рендеринг проще и предсказуемее, но не даёт таких возможностей, как `useTransition` или Suspense на сервере.

**2. События**

React использует Synthetic Event System с пулингом событий (в старых версиях) и нормализацией. Preact использует нативные DOM-события напрямую, без слоя абстракции:

```jsx
// React: SyntheticEvent
function ReactButton() {
  return <button onClick={(e) => console.log(e.nativeEvent)}>Click</button>;
}

// Preact: нативное событие
function PreactButton() {
  return <button onClick={(e) => console.log(e)}>Click</button>;
}
```

**3. `class` вместо `className`**

Preact поддерживает оба варианта, но в документации рекомендует использовать стандартный HTML-атрибут `class`:

```jsx
// В Preact работает и так, и так
<div class="container" className="container">Content</div>
```

---

## Совместимость и миграция

### Preact Compat

`preact/compat` — это тонкая прослойка, которая делает Preact совместимым с большинством библиотек и кода, написанных для React.

```js
// vite.config.js
import { defineConfig } from "vite";
import preact from "@preact/preset-vite";

export default defineConfig({
  plugins: [preact()],
  resolve: {
    alias: {
      react: "preact/compat",
      "react-dom": "preact/compat",
      "react/jsx-runtime": "preact/jsx-runtime",
    },
  },
});
```

После такого алиаса можно импортировать `React` через обычные пути:

```jsx
import { useState } from "react";

function App() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### Постепенная миграция

Если проект на React, но хочется уменьшить размер бандла, часто используют Preact только в production-сборке, а в development остаются на React для удобства отладки:

```js
// webpack.config.js
const isProd = process.env.NODE_ENV === "production";

module.exports = {
  resolve: {
    alias: isProd
      ? {
          react: "preact/compat",
          "react-dom": "preact/compat",
        }
      : {},
  },
};
```

---

## Preact Signals

**Signals** — система реактивного состояния от команды Preact. В отличие от `useState`, сигналы не вызывают ререндер всего компонента: обновляется только тот DOM-узел, который зависит от значения.

```jsx
import { signal, computed } from "@preact/signals";

const count = signal(0);
const double = computed(() => count.value * 2);

function Counter() {
  return (
    <div>
      <p>Count: {count.value}</p>
      <p>Double: {double.value}</p>
      <button onClick={() => count.value++}>Increment</button>
    </div>
  );
}
```

Особенности Signals:

- Значение читается через `.value`
- Можно передавать в пропсы и контекст без лишних ререндеров
- Работают вне компонентов — как глобальное состояние
- Поддерживаются и в React, и в Preact через `@preact/signals-react`

---

## Когда выбирать Preact

Preact хорош, если:

- **Критичен размер бандла** — landing page, виджеты, встраиваемые скрипты
- **Много микрофронтендов или embed-кода** — меньший рантайм = быстрее загрузка
- **Простое приложение** без Concurrent React и сложного SSR
- **Нужна миграция с React без переписывания** — `preact/compat` покрывает большинство кейсов
- **Хочется попробовать Signals** — реактивность без лишних ререндеров

---

## Когда оставить React

Preact не всегда подходит:

- **Next.js и RSC** — Server Components, Server Actions, App Router тесно связаны с React-рантаймом
- **Concurrent features** — `useTransition`, `useDeferredValue`, Suspense boundaries работают не полностью
- **React Compiler и новые фичи React 19+** — появляются раньше и лучше поддерживаются в React
- **Библиотеки с прямой зависимостью от React internals** — некоторые пакеты ломаются при алиасинге

> 💡 Preact — не замена React для всех проектов, а инструмент для сценариев, где размер и скорость важнее последних фич экосистемы.
