# useContext — передача данных через дерево компонентов

## Содержание

1. [Что такое useContext](#что-такое-usecontext)
2. [Какую проблему решает](#какую-проблему-решает)
3. [Базовое использование](#базовое-использование)
4. [Vue-аналог: provide/inject](#vue-аналог-provideinject)
5. [Оптимизация рендеров](#оптимизация-рендеров)
6. [useReducer + useContext — мини-Redux](#usereducer--usecontext--мини-redux)
7. [use() в React 19 — замена useContext](#use-в-react-19--замена-usecontext)
8. [Сравнение с Zustand и Redux](#сравнение-с-zustand-и-redux)
9. [Лучшие практики](#лучшие-практики)
10. [Антипаттерны](#антипаттерны)

---

## Что такое useContext

`useContext` — встроенный хук React, позволяющий компоненту **получать данные из Context**, минуя промежуточные уровни компонентов. Вместе с `createContext` и `Provider` он образует механизм передачи данных через дерево без пробрасывания пропсов (prop drilling).

```jsx
import { createContext, useContext } from "react";

const ThemeContext = createContext("light");

function Button() {
  const theme = useContext(ThemeContext);
  return <button className={`btn btn--${theme}`}>Click me</button>;
}
```

Ключевая идея: вы создаёте **контекст** (объект-идентификатор), оборачиваете часть дерева в `<Provider value={...}>`, и любой дочерний компонент может прочитать значение через `useContext`, независимо от глубины вложенности.

---

## Какую проблему решает

### Проблема: Prop Drilling

Без контекста данные передаются через каждый уровень компонентов, даже если промежуточные компоненты их не используют:

```jsx
// ❌ Prop drilling: тема передаётся через 4 уровня
function App() {
  const [theme, setTheme] = useState("light");
  return <Layout theme={theme} setTheme={setTheme} />;
}

function Layout({ theme, setTheme }) {
  return <Header theme={theme} setTheme={setTheme} />;
}

function Header({ theme, setTheme }) {
  return <Nav theme={theme} setTheme={setTheme} />;
}

function Nav({ theme, setTheme }) {
  return <ThemeToggle theme={theme} setTheme={setTheme} />;
}

function ThemeToggle({ theme, setTheme }) {
  return <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>{theme}</button>;
}
```

Проблемы:
- Промежуточные компоненты (`Layout`, `Header`, `Nav`) вынуждены принимать и передавать пропсы, которые им не нужны.
- Изменение структуры данных требует изменения сигнатур всех промежуточных компонентов.
- Код становится хрупким и сложным для рефакторинга.

### Решение: Context

```jsx
// ✅ Context: данные доступны напрямую
const ThemeContext = createContext(null);

function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext>
  );
}

function Layout() {
  return <Header />;
}

function Header() {
  return <Nav />;
}

function Nav() {
  return <ThemeToggle />;
}

function ThemeToggle() {
  const { theme, setTheme } = useContext(ThemeContext);
  return <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>{theme}</button>;
}
```

Промежуточные компоненты больше не участвуют в передаче данных. `ThemeToggle` получает `theme` и `setTheme` напрямую из контекста, независимо от глубины вложенности.

---

## Базовое использование

### Шаг 1: Создание контекста

```jsx
import { createContext } from "react";

const ThemeContext = createContext("light");
```

`createContext` принимает **значение по умолчанию**, которое используется, если компонент читает контекст без `<Provider>` выше. Это полезно для тестирования и fallback-поведения.

### Шаг 2: Provider — предоставление значения

```jsx
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext value={{ theme, setTheme }}>
      <MainLayout />
    </ThemeContext>
  );
}
```

`<Provider>` оборачивает часть дерева компонентов и предоставляет значение всем дочерним компонентам. Если значение меняется, все потребители контекста перерендериваются.

### Шаг 3: useContext — чтение значения

```jsx
function ThemeToggle() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      Current: {theme}
    </button>
  );
}
```

`useContext` возвращает текущее значение контекста из ближайшего `<Provider>` выше в дереве.

### Полный пример: тема приложения

```jsx
import { createContext, useContext, useState } from "react";

const ThemeContext = createContext(null);

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext value={{ theme, setTheme }}>
      {children}
    </ThemeContext>
  );
}

function App() {
  return (
    <ThemeProvider>
      <Header />
      <MainContent />
    </ThemeProvider>
  );
}

function Header() {
  const { theme } = useContext(ThemeContext);
  return <header className={`header header--${theme}`}>Header</header>;
}

function MainContent() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <main>
      <p>Current theme: {theme}</p>
      <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
        Toggle Theme
      </button>
    </main>
  );
}
```

---

## Vue-аналог: provide/inject

В Vue эквивалент `useContext` — это `provide`/`inject`:

```js
// Vue 3 Composition API
import { provide, inject, ref } from "vue";

// App.vue
export default {
  setup() {
    const theme = ref("light");
    provide("theme", { theme });
  },
};

// ChildComponent.vue
export default {
  setup() {
    const { theme } = inject("theme");
    return { theme };
  },
};
```

### Сравнение React и Vue

| Концепт | React | Vue |
|---|---|---|
| **Создание контекста** | `createContext(defaultValue)` | Нет отдельного объекта, используется строковый ключ |
| **Предоставление** | `<Context.Provider value={...}>` | `provide(key, value)` |
| **Чтение** | `useContext(Context)` | `inject(key)` |
| **Типизация** | Через TypeScript-генерики | Через TypeScript или JSDoc |
| **Реактивность** | Перерендер при изменении value | Автоматическая реактивность через `ref`/`reactive` |

### Ключевые различия

**1. Идентификатор контекста:**
- React: `createContext` возвращает **объект**, который используется как идентификатор.
- Vue: используется **строковый ключ** (или Symbol).

```jsx
// React: объект-идентификатор
const ThemeContext = createContext("light");
<ThemeContext value={theme}>...</ThemeContext>
const theme = useContext(ThemeContext);
```

```js
// Vue: строковый ключ
provide("theme", theme);
const theme = inject("theme");
```

**2. Реактивность:**
- React: при изменении `value` в `<Provider>` все потребители перерендериваются.
- Vue: если передать `ref` или `reactive` через `provide`, изменения автоматически отслеживаются — повторный `inject` не нужен.

```js
// Vue: реактивность "из коробки"
const theme = ref("light");
provide("theme", theme);

// В дочернем компоненте:
const theme = inject("theme");
theme.value = "dark"; // Автоматически обновится во всех inject
```

**3. Значение по умолчанию:**
- React: задаётся в `createContext(defaultValue)`.
- Vue: задаётся вторым аргументом `inject(key, defaultValue)`.

---

## Оптимизация рендеров

### Проблема: избыточные перерендеры

Когда значение контекста меняется, **все** потребители перерендериваются, даже если они используют только часть данных:

```jsx
// ❌ Все потребители перерендериваются при изменении theme ИЛИ setTheme
const ThemeContext = createContext(null);

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext value={{ theme, setTheme }}>
      {children}
    </ThemeContext>
  );
}

function ThemeDisplay() {
  const { theme } = useContext(ThemeContext);
  return <span>{theme}</span>;
}

function ThemeToggle() {
  const { setTheme } = useContext(ThemeContext);
  return <button onClick={() => setTheme("dark")}>Toggle</button>;
}
```

При клике на `ThemeToggle` меняется `setTheme` (ссылка стабильна, но объект `{ theme, setTheme }` создаётся заново), и `ThemeDisplay` перерендеривается, хотя использует только `theme`.

### Решение 1: Разделение контекстов

Разделите `state` и `dispatch` на два контекста:

```jsx
// ✅ Два контекста: ThemeContext для чтения, ThemeDispatchContext для действий
const ThemeContext = createContext("light");
const ThemeDispatchContext = createContext(null);

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext value={theme}>
      <ThemeDispatchContext value={setTheme}>
        {children}
      </ThemeDispatchContext>
    </ThemeContext>
  );
}

function ThemeDisplay() {
  const theme = useContext(ThemeContext);
  return <span>{theme}</span>;
}

function ThemeToggle() {
  const setTheme = useContext(ThemeDispatchContext);
  return <button onClick={() => setTheme("dark")}>Toggle</button>;
}
```

Теперь `ThemeToggle` не перерендеривается при изменении `theme`, а `ThemeDisplay` не перерендеривается при изменении `setTheme` (хотя `setTheme` стабилен, поэтому это не проблема).

### Решение 2: Мемоизация значения контекста

Если значение контекста — объект, мемоизируйте его через `useMemo`:

```jsx
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return <ThemeContext value={value}>{children}</ThemeContext>;
}
```

Без `useMemo` объект `{ theme, setTheme }` создаётся заново при каждом рендере `ThemeProvider`, что вызывает перерендер всех потребителей, даже если `theme` не изменился.

### Решение 3: Мемоизация потребителей

Оберните компоненты, читающие контекст, в `React.memo`:

```jsx
const ThemeDisplay = React.memo(function ThemeDisplay() {
  const theme = useContext(ThemeContext);
  return <span>{theme}</span>;
});
```

`React.memo` предотвращает перерендер, если пропсы не изменились. Но для контекста это работает только если значение контекста стабильно (мемоизировано через `useMemo`).

---

## useReducer + useContext — мини-Redux

Связка `useReducer` + `useContext` — минималистичная альтернатива Redux для приложений малого и среднего размера:

```jsx
import { createContext, useContext, useReducer } from "react";

const CounterContext = createContext(null);
const CounterDispatchContext = createContext(null);

function counterReducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return { count: 0 };
    default:
      return state;
  }
}

function CounterProvider({ children }) {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });
  return (
    <CounterContext value={state}>
      <CounterDispatchContext value={dispatch}>
        {children}
      </CounterDispatchContext>
    </CounterContext>
  );
}

function CounterDisplay() {
  const state = useContext(CounterContext);
  return <span>Count: {state.count}</span>;
}

function CounterButtons() {
  const dispatch = useContext(CounterDispatchContext);
  return (
    <>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </>
  );
}

function App() {
  return (
    <CounterProvider>
      <CounterDisplay />
      <CounterButtons />
    </CounterProvider>
  );
}
```

### Преимущества

- **Zero dependencies:** не нужны Redux, Zustand или другие библиотеки.
- **Предсказуемость:** `useReducer` — чистая функция `(state, action) => nextState`.
- **Тестируемость:** редуктор тестируется без React.
- **Разделение state/dispatch:** компоненты, читающие только `dispatch`, не перерендериваются при изменении `state`.

### Ограничения

- Нет middleware (логирование, асинхронные экшоны, персистентность).
- Нет DevTools из коробки (Redux DevTools не работает).
- Нет селекторов — все потребители перерендериваются при изменении `state`.

### Когда использовать

- Простые приложения с глобальным состоянием.
- Нет необходимости в middleware и DevTools.
- Хотите избежать дополнительных зависимостей.

Для сложных случаев используйте [Zustand](./zustand.md) или Redux Toolkit.

---

## use() в React 19 — замена useContext

В React 19 хук `use(context)` полностью заменяет `useContext(context)` с дополнительной возможностью **условного вызова**:

```jsx
import { use, createContext } from "react";

const ThemeContext = createContext("light");

function Button() {
  const theme = use(ThemeContext); // Вместо useContext(ThemeContext)
  return <button className={`btn btn--${theme}`}>Click me</button>;
}
```

### Ключевое отличие: условный вызов

`useContext` запрещено вызывать условно (внутри `if`, `for`, после `return`), а `use` — разрешено:

```jsx
function Header({ variant }) {
  if (variant === "minimal") {
    return <MinimalHeader />;
  }

  const theme = use(ThemeContext); // ✅ Условный вызов разрешён
  return <FullHeader theme={theme} />;
}
```

### Сравнение

| | `useContext()` | `use()` |
|---|---|---|
| Условный вызов | ❌ Запрещён | ✅ Разрешён |
| В циклах | ❌ Запрещён | ✅ Разрешён |
| После return | ❌ Запрещён | ✅ Разрешён |
| Синтаксис | `useContext(MyContext)` | `use(MyContext)` |
| Совместимость | React 16.8+ | React 19.0+ |

В новых проектах на React 19 предпочитайте `use(context)` вместо `useContext(context)`.

(Подробнее: [React `use()` — Полное руководство](./react_use.md))

---

## Сравнение с Zustand и Redux

### Context vs Zustand

| Характеристика | Context | Zustand |
|---|---|---|
| **Основное назначение** | Передача данных через дерево | Глобальное управление состоянием |
| **Оптимизация рендеров** | Все потребители перерендериваются | Селекторы для точечной подписки |
| **Провайдер** | Требуется `<Provider>` | Не требуется |
| **Доступ вне React** | ❌ Нет | ✅ Да (`useStore.getState()`) |
| **Middleware** | ❌ Нет | ✅ Да (`persist`, `devtools`, `immer`) |
| **DevTools** | ❌ Нет | ✅ Redux DevTools |
| **Размер** | Встроен в React | ~1 KB |

**Когда использовать Context:**
- Тема, локаль, авторизация — данные, которые меняются редко.
- Нужно передать данные через несколько уровней компонентов.
- Нет необходимости в сложной логике обновления.

**Когда использовать Zustand:**
- Высокочастотные обновления состояния (счётчики, формы, корзина).
- Нужны селекторы для оптимизации рендеров.
- Требуется доступ к состоянию вне React (обработчики событий, утилиты).
- Нужны middleware (персистентность, логирование, DevTools).

### Context vs Redux

Redux — это **надстройка** над Context (или похожим механизмом), добавляющая:
- Единое хранилище (store) для всего приложения.
- Редукторы и экшоны для предсказуемого обновления состояния.
- Middleware для асинхронности, логирования, персистентности.
- Redux DevTools для отладки с time-travel.

Context — это **примитив** для передачи данных. Redux использует его (или похожий механизм подписки) под капотом, но добавляет богатую экосистему поверх.

**Когда использовать Context:**
- Простые приложения без сложной логики состояния.
- Нет необходимости в DevTools и middleware.
- Хотите избежать дополнительных зависимостей.

**Когда использовать Redux:**
- Крупные приложения с десятками разрозненных компонентов.
- Нужна предсказуемость: `(state, action) => nextState`.
- Требуется time-travel debugging и middleware.
- Команда уже знакома с Redux.

Для большинства современных приложений **Zustand проще и легче**, чем Redux, и покрывает 90% случаев.

---

## Лучшие практики

### 1. Используйте отдельные контексты для state и dispatch

```jsx
// ✅ Два контекста: для чтения и для действий
const ThemeContext = createContext("light");
const ThemeDispatchContext = createContext(null);

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext value={theme}>
      <ThemeDispatchContext value={setTheme}>
        {children}
      </ThemeDispatchContext>
    </ThemeContext>
  );
}
```

Это предотвращает перерендер компонентов, читающих только `dispatch`, при изменении `state`.

### 2. Мемоизируйте значение контекста

Если значение контекста — объект, используйте `useMemo`:

```jsx
function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);

  const value = useMemo(() => ({ user, loading, setUser }), [user, loading]);

  return <UserContext value={value}>{children}</UserContext>;
}
```

Без `useMemo` объект создаётся заново при каждом рендере, вызывая перерендер всех потребителей.

### 3. Используйте Context для редко изменяющихся данных

Context идеален для данных, которые меняются нечасто:
- Тема приложения
- Локаль (язык)
- Аутентификация (текущий пользователь)
- Конфигурация приложения

```jsx
const LocaleContext = createContext("en");
const AuthContext = createContext(null);
const ConfigContext = createContext(null);
```

Для высокочастотных обновлений (счётчики, формы, ввод текста) используйте `useState` или Zustand.

### 4. Выносите Provider на верхний уровень

Оборачивайте всё приложение в Provider в корневом компоненте:

```jsx
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <LocaleProvider>
          <MainLayout />
        </LocaleProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}
```

### 5. Используйте use() в React 19

Если ваш проект на React 19+, предпочитайте `use(context)` вместо `useContext(context)`:

```jsx
// ✅ React 19+
const theme = use(ThemeContext);

// ✅ React 16.8–18.x
const theme = useContext(ThemeContext);
```

`use()` добавляет возможность условного вызова, что открывает новые паттерны.

---

## Антипаттерны

### ❌ Создание нового объекта value при каждом рендере

```jsx
// ❌ Плохо: объект создаётся заново при каждом рендере
function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <UserContext value={{ user, setUser }}>
      {children}
    </UserContext>
  );
}

// ✅ Хорошо: мемоизация через useMemo
function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({ user, setUser }), [user]);
  return <UserContext value={value}>{children}</UserContext>;
}
```

Без `useMemo` все потребители перерендериваются при каждом рендере `UserProvider`, даже если `user` не изменился.

### ❌ Использование Context для высокочастотных обновлений

```jsx
// ❌ Плохо: все потребители перерендериваются при каждом изменении count
const CounterContext = createContext(null);

function CounterProvider({ children }) {
  const [count, setCount] = useState(0);
  return (
    <CounterContext value={{ count, setCount }}>
      {children}
    </CounterContext>
  );
}

function Counter() {
  const { count } = useContext(CounterContext);
  return <span>{count}</span>;
}

function IncrementButton() {
  const { setCount } = useContext(CounterContext);
  return <button onClick={() => setCount((c) => c + 1)}>+</button>;
}
```

При каждом клике на `IncrementButton` все потребители `CounterContext` перерендериваются. Для высокочастотных обновлений используйте Zustand с селекторами:

```jsx
// ✅ Zustand: перерендер только при изменении count
const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

function Counter() {
  const count = useCounterStore((state) => state.count);
  return <span>{count}</span>;
}
```

### ❌ Чтение Context без Provider

```jsx
const ThemeContext = createContext("light");

function Button() {
  const theme = useContext(ThemeContext);
  return <button>{theme}</button>;
}

// ❌ Button рендерится без Provider — используется значение по умолчанию "light"
function App() {
  return <Button />;
}
```

Если компонент читает контекст без `<Provider>` выше, он получает значение по умолчанию из `createContext`. Это может привести к неожиданному поведению, если вы забыли обернуть дерево в Provider.

Решение: используйте `null` как значение по умолчанию и проверяйте его:

```jsx
const ThemeContext = createContext(null);

function Button() {
  const theme = useContext(ThemeContext);
  if (!theme) {
    throw new Error("Button must be used within ThemeProvider");
  }
  return <button>{theme}</button>;
}
```

### ❌ Вложенные Provider без необходимости

```jsx
// ❌ Плохо: избыточная вложенность
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <LocaleProvider>
          <ConfigProvider>
            <ThemeSubProvider>
              <MainLayout />
            </ThemeSubProvider>
          </ConfigProvider>
        </LocaleProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

// ✅ Хорошо: плоская структура
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <LocaleProvider>
          <ConfigProvider>
            <MainLayout />
          </ConfigProvider>
        </LocaleProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}
```

Избегайте избыточной вложенности Provider'ов. Каждый Provider добавляет уровень вложенности и усложняет чтение кода.

### ❌ Использование Context для всего глобального состояния

```jsx
// ❌ Плохо: один контекст для всего
const AppContext = createContext(null);

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [cart, setCart] = useState([]);
  const [theme, setTheme] = useState("light");
  const [locale, setLocale] = useState("en");

  const value = useMemo(
    () => ({ user, cart, theme, locale, setUser, setCart, setTheme, setLocale }),
    [user, cart, theme, locale]
  );

  return <AppContext value={value}>{children}</AppContext>;
}
```

Проблемы:
- Все потребители перерендериваются при изменении любого поля.
- Сложно отследить, какие компоненты используют какие данные.
- Трудно рефакторить.

Решение: разделите состояние на отдельные контексты или используйте Zustand:

```jsx
// ✅ Хорошо: отдельные контексты
const UserContext = createContext(null);
const CartContext = createContext(null);
const ThemeContext = createContext(null);
const LocaleContext = createContext(null);

// ✅ Ещё лучше: Zustand для всего глобального состояния
const useAppStore = create((set) => ({
  user: null,
  cart: [],
  theme: "light",
  locale: "en",
  setUser: (user) => set({ user }),
  addToCart: (item) => set((state) => ({ cart: [...state.cart, item] })),
  setTheme: (theme) => set({ theme }),
  setLocale: (locale) => set({ locale }),
}));
```

---

## Итог

`useContext` — это **механизм передачи данных через дерево компонентов** без пробрасывания пропсов. Он решает конкретную проблему prop drilling, но не является полноценной заменой библиотек управления состоянием.

**Когда использовать Context:**
- Тема, локаль, авторизация — данные, которые меняются редко.
- Нужно передать данные через несколько уровней компонентов.
- Простые приложения без сложной логики состояния.

**Когда использовать Zustand/Redux:**
- Высокочастотные обновления состояния.
- Нужны селекторы для оптимизации рендеров.
- Требуется доступ к состоянию вне React.
- Нужны middleware и DevTools.

**Связка useReducer + useContext** — минималистичная альтернатива Redux для приложений малого и среднего размера. Для сложных случаев используйте [Zustand](./zustand.md) или Redux Toolkit.

В React 19 `use(context)` заменяет `useContext(context)` и добавляет возможность условного вызова.
