# Классические паттерны GoF во Frontend-разработке (React и Vue)

## Содержание

1. [Введение](#введение)
2. [Абстрактная фабрика](#абстрактная-фабрика)
3. [Декораторы](#декораторы)
4. [Синглтон](#синглтон)
5. [Наблюдатель](#наблюдатель)
6. [Сравнительная таблица](#сравнительная-таблица)
7. [Заключение](#заключение)

---

## Введение

Паттерны проектирования из книги «Приёмы объектно-ориентированного проектирования. Паттерны проектирования» (GoF — Gang of Four) были описаны в 1994 году. Они не потеряли актуальности и во frontend-разработке, хотя их реализация выглядит иначе, чем в классическом ООП-коде.

Во frontend мы редко создаем классы с наследованием. Вместо этого паттерны адаптируются под:
- **React** — хуки, компоненты, Context, JSX-композицию
- **Vue** — реактивность, composables, директивы, provide/inject

В этой статье разберём четыре паттерна и покажем их практическое применение в обоих фреймворках.

---

## Абстрактная фабрика

### Суть паттерна

**Абстрактная фабрика** предоставляет интерфейс для создания семейств связанных объектов, не конкретизируя их классы. Клиент получает фабрику и работает с ней через общий интерфейс, не зная, какие конкретно объекты будут созданы.

### Классический пример

```javascript
// Интерфейс фабрики
class UIFactory {
  createButton() { throw new Error("Abstract method"); }
  createInput() { throw new Error("Abstract method"); }
  createModal() { throw new Error("Abstract method"); }
}

// Конкретная фабрика для светлой темы
class LightUIFactory extends UIFactory {
  createButton() { return new LightButton(); }
  createInput() { return new LightInput(); }
  createModal() { return new LightModal(); }
}

// Конкретная фабрика для тёмной темы
class DarkUIFactory extends UIFactory {
  createButton() { return new DarkButton(); }
  createInput() { return new DarkInput(); }
  createModal() { return new DarkModal(); }
}

// Клиент не знает, какая фабрика используется
const factory = theme === "dark" ? new DarkUIFactory() : new LightUIFactory();
const button = factory.createButton();
const input = factory.createInput();
```

### В React

В React абстрактная фабрика реализуется через **Context + фабричные функции** или через **рендер-пропсы / children**. Вместо классов — функции и компоненты.

#### Паттерн: фабрика компонентов через Context

```jsx
// ui/factories/theme-factory.jsx
import { createContext, use, useMemo } from "react";

const ThemeFactoryContext = createContext(null);

const lightFactory = {
  Button: ({ children, ...props }) => (
    <button className="btn btn--light" {...props}>{children}</button>
  ),
  Input: (props) => (
    <input className="input input--light" {...props} />
  ),
  Modal: ({ children, onClose }) => (
    <div className="modal modal--light">
      <div className="modal__overlay" onClick={onClose} />
      <div className="modal__content">{children}</div>
    </div>
  ),
};

const darkFactory = {
  Button: ({ children, ...props }) => (
    <button className="btn btn--dark" {...props}>{children}</button>
  ),
  Input: (props) => (
    <input className="input input--dark" {...props} />
  ),
  Modal: ({ children, onClose }) => (
    <div className="modal modal--dark">
      <div className="modal__overlay" onClick={onClose} />
      <div className="modal__content">{children}</div>
    </div>
  ),
};

export function ThemeFactoryProvider({ theme, children }) {
  const factory = useMemo(
    () => (theme === "dark" ? darkFactory : lightFactory),
    [theme]
  );

  return (
    <ThemeFactoryContext value={factory}>
      {children}
    </ThemeFactoryContext>
  );
}

export function useUI() {
  const factory = use(ThemeFactoryContext);
  if (!factory) throw new Error("ThemeFactoryProvider is missing");
  return factory;
}
```

Использование — компоненты-потребители не знают о конкретной теме:

```jsx
function LoginForm() {
  const { Button, Input, Modal } = useUI();

  return (
    <Modal>
      <h2>Вход</h2>
      <Input type="email" placeholder="Email" />
      <Input type="password" placeholder="Пароль" />
      <Button>Войти</Button>
    </Modal>
  );
}

// Корневой компонент
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeFactoryProvider theme={theme}>
      <LoginForm />
      <button onClick={() => setTheme(t => t === "light" ? "dark" : "light")}>
        Переключить тему
      </button>
    </ThemeFactoryProvider>
  );
}
```

#### Паттерн: фабрика для разных платформ

Абстрактная фабрика полезна, когда нужно создавать компоненты для разных платформ или сред:

```jsx
// factories/platform-factory.jsx
const webFactory = {
  createDatePicker: () => WebDatePicker,
  createFilePicker: () => WebFilePicker,
  createNotification: (msg) => new WebNotification(msg),
};

const mobileFactory = {
  createDatePicker: () => NativeDatePicker,
  createFilePicker: () => NativeFilePicker,
  createNotification: (msg) => new PushNotification(msg),
};

function getFactory(platform) {
  switch (platform) {
    case "web": return webFactory;
    case "mobile": return mobileFactory;
    default: return webFactory;
  }
}

// Использование
function DatePickerScreen() {
  const factory = getFactory(currentPlatform);
  const DatePicker = factory.createDatePicker();
  return <DatePicker onChange={handleDateChange} />;
}
```

### В Vue

Во Vue абстрактная фабрика реализуется через **provide/inject** или **composables**.

#### Паттерн: фабрика через provide/inject

```vue
<!-- composables/useUIFactory.js -->
<script setup>
import { provide, inject, computed } from "vue";

const UI_FACTORY_KEY = Symbol("uiFactory");

const lightFactory = {
  button: "LightButton",
  input: "LightInput",
  modal: "LightModal",
};

const darkFactory = {
  button: "DarkButton",
  input: "DarkInput",
  modal: "DarkModal",
};

export function provideUIFactory(theme) {
  const factory = computed(() =>
    theme.value === "dark" ? darkFactory : lightFactory
  );
  provide(UI_FACTORY_KEY, factory);
}

export function useUIFactory() {
  const factory = inject(UI_FACTORY_KEY);
  if (!factory) throw new Error("UI Factory not provided");
  return factory;
}
</script>
```

```vue
<!-- App.vue -->
<script setup>
import { ref } from "vue";
import { provideUIFactory } from "./composables/useUIFactory";

const theme = ref("light");
provideUIFactory(theme);
</script>
```

```vue
<!-- LoginForm.vue -->
<script setup>
import { useUIFactory } from "./composables/useUIFactory";
import LightButton from "./components/LightButton.vue";
import DarkButton from "./components/DarkButton.vue";
// ...

const factory = useUIFactory();
</script>

<template>
  <component :is="factory.value.button">Войти</component>
</template>
```

### Когда использовать

- **Темизация** — разные наборы UI-компонентов для разных тем
- **Мультиплатформенность** — разные реализации для web, mobile, desktop
- **A/B тестирование** — разные варианты компонентов для разных групп пользователей
- **White-label решения** — один код, разные бренды с разными UI

### Когда НЕ использовать

- Когда различия между вариантами можно решить через CSS-классы или пропсы
- Когда вариантов всего один-два — проще условный рендеринг
- Для простых UI-китов с 3–5 компонентами

---

## Декораторы

### Суть паттерна

**Декоратор** динамически добавляет объекту новую ответственность, оборачивая его. Это гибкая альтернатива наследованию для расширения функциональности.

### Классический пример

```javascript
class Coffee {
  cost() { return 100; }
  description() { return "Кофе"; }
}

class MilkDecorator {
  constructor(coffee) { this.coffee = coffee; }
  cost() { return this.coffee.cost() + 30; }
  description() { return this.coffee.description() + " + молоко"; }
}

class SugarDecorator {
  constructor(coffee) { this.coffee = coffee; }
  cost() { return this.coffee.cost() + 10; }
  description() { return this.coffee.description() + " + сахар"; }
}

const order = new SugarDecorator(new MilkDecorator(new Coffee()));
order.cost();         // 140
order.description();  // "Кофе + молоко + сахар"
```

### В React

В React декораторы реализуются через **компоненты высшего порядка (HOC)**, **хуки** и **JSX-обёртки**.

#### HOC как декоратор

```jsx
// Декоратор: добавляет логирование
function withLogging(WrappedComponent) {
  return function WithLogging(props) {
    useEffect(() => {
      console.log(`[Mount] ${WrappedComponent.name}`, props);
      return () => console.log(`[Unmount] ${WrappedComponent.name}`);
    }, []);

    return <WrappedComponent {...props} />;
  };
}

// Декоратор: добавляет загрузку данных
function withData(WrappedComponent, fetchData) {
  return function WithData(props) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      fetchData().then((result) => {
        setData(result);
        setLoading(false);
      });
    }, []);

    if (loading) return <Spinner />;
    return <WrappedComponent {...props} data={data} />;
  };
}

// Композиция декораторов
const EnhancedDashboard = withLogging(withData(Dashboard, fetchDashboardData));
```

#### Хуки как декораторы (современный подход)

В 2026 году хуки заменили HOC для большинства случаев:

```jsx
// Декоратор: логирование через хук
function useLogging(componentName, props) {
  useEffect(() => {
    console.log(`[Mount] ${componentName}`, props);
    return () => console.log(`[Unmount] ${componentName}`);
  }, []);
}

// Декоратор: загрузка данных через хук
function useData(fetchFn) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchFn().then((result) => {
      setData(result);
      setLoading(false);
    });
  }, []);

  return { data, loading };
}

// Использование — декларативно и без вложенности
function Dashboard() {
  useLogging("Dashboard");
  const { data, loading } = useData(fetchDashboardData);

  if (loading) return <Spinner />;
  return <DashboardView data={data} />;
}
```

#### Декоратор через JSX-обёртку

```jsx
// Декоратор: добавляет анимацию появления
function FadeIn({ children, delay = 0 }) {
  const [visible, setVisible] = useState(false);

  useEffect(() => {
    const timer = setTimeout(() => setVisible(true), delay);
    return () => clearTimeout(timer);
  }, [delay]);

  return (
    <div
      style={{
        opacity: visible ? 1 : 0,
        transition: "opacity 0.3s ease",
      }}
    >
      {children}
    </div>
  );
}

// Декоратор: добавляет обработку ошибок
function ErrorBoundary({ children, fallback }) {
  // ... реализация Error Boundary
}

// Композиция декораторов в JSX — читаемая цепочка
<FadeIn delay={200}>
  <ErrorBoundary fallback={<ErrorScreen />}>
    <Suspense fallback={<Spinner />}>
      <Dashboard />
    </Suspense>
  </ErrorBoundary>
</FadeIn>
```

#### Декоратор: добавление поведения через пропсы

```jsx
// Декоратор: делает любой компонент draggable
function Draggable({ children }) {
  const ref = useRef(null);
  const [position, setPosition] = useState({ x: 0, y: 0 });

  // логика drag & drop...

  return (
    <div ref={ref} style={{ transform: `translate(${position.x}px, ${position.y}px)` }}>
      {children}
    </div>
  );
}

// Любой компонент становится перетаскиваемым
<Draggable>
  <Card>Перетащи меня</Card>
</Draggable>
```

### В Vue

Во Vue декораторы реализуются через **директивы**, **composables** и **обёрточные компоненты**.

#### Директивы как декораторы

Директивы — это встроенный механизм декорирования DOM-элементов:

```javascript
// Директива: добавляет tooltip
const vTooltip = {
  mounted(el, binding) {
    el._tooltip = createTooltip(el, binding.value);
  },
  updated(el, binding) {
    el._tooltip?.update(binding.value);
  },
  unmounted(el) {
    el._tooltip?.destroy();
  },
};

// Директива: добавляет анимацию появления
const vFadeIn = {
  mounted(el) {
    el.style.opacity = "0";
    el.style.transition = "opacity 0.3s ease";
    requestAnimationFrame(() => {
      el.style.opacity = "1";
    });
  },
};

// Использование в шаблоне — декларативно
<button v-tooltip="'Подсказка'" v-fade-in>Нажми меня</button>
```

#### Composables как декораторы

```javascript
// composable: декоратор логирования
function useLogging(componentName) {
  onMounted(() => console.log(`[Mount] ${componentName}`));
  onUnmounted(() => console.log(`[Unmount] ${componentName}`));
}

// composable: декоратор обработки ошибок
function useErrorHandler() {
  const error = ref(null);

  const wrap = (fn) => async (...args) => {
    try {
      error.value = null;
      return await fn(...args);
    } catch (e) {
      error.value = e;
      console.error(e);
    }
  };

  return { error, wrap };
}

// Использование
<script setup>
import { useLogging, useErrorHandler } from "./composables";

useLogging("Dashboard");
const { error, wrap } = useErrorHandler();

const loadData = wrap(async () => {
  const res = await fetch("/api/data");
  return res.json();
});
</script>
```

#### Обёрточные компоненты

```vue
<!-- FadeIn.vue — декоратор анимации -->
<template>
  <Transition name="fade">
    <slot />
  </Transition>
</template>

<style>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>
```

```vue
<!-- Использование -->
<script setup>
import FadeIn from "./FadeIn.vue";
</script>

<template>
  <FadeIn>
    <Dashboard />
  </FadeIn>
</template>
```

### Когда использовать

- **Сквозная логика** — логирование, аналитика, аутентификация
- **Поведенческие расширения** — draggable, resizable, tooltip
- **Обработка ошибок** — обёртка над компонентами с fallback
- **Условная функциональность** — добавление возможностей по условию

### Когда НЕ использовать

- Когда расширение можно реализовать через пропсы или слоты
- Когда декораторов больше 3 в цепочке — код становится нечитаемым
- Для простой логики, не связанной с рендерингом — используйте хуки/composables

---

## Синглтон

### Суть паттерна

**Синглтон** гарантирует, что у класса есть только один экземпляр, и предоставляет глобальную точку доступа к нему.

### Классический пример

```javascript
class DatabaseConnection {
  static instance = null;

  static getInstance() {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }

  constructor() {
    if (DatabaseConnection.instance) {
      return DatabaseConnection.instance;
    }
    this.connection = this.connect();
    DatabaseConnection.instance = this;
  }

  connect() {
    // expensive connection setup
    return { /* connection object */ };
  }

  query(sql) {
    return this.connection.execute(sql);
  }
}

// Всегда один и тот же экземпляр
const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();
db1 === db2; // true
```

### Проблема синглтона в frontend

В классическом виде синглтон считается **антипаттерном** во frontend:
- Затрудняет тестирование (глобальное состояние)
- Создаёт скрытые зависимости
- Проблемы с SSR (общее состояние между запросами)

Однако **концепция единственного экземпляра** используется повсеместно — через модули, сторы, контексты.

### В React

#### Модуль как синглтон (ES-модули)

ES-модули в JavaScript по своей природе являются синглтонами — они загружаются и выполняются один раз:

```javascript
// services/api-client.js
// Этот модуль — синглтон. Все импорты получают один и тот же объект.

let authToken = null;

const apiClient = {
  setToken(token) {
    authToken = token;
  },

  async get(url) {
    const res = await fetch(url, {
      headers: { Authorization: `Bearer ${authToken}` },
    });
    return res.json();
  },

  async post(url, body) {
    const res = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${authToken}`,
      },
      body: JSON.stringify(body),
    });
    return res.json();
  },
};

export default apiClient;
```

```jsx
// Использование — один экземпляр на всё приложение
import apiClient from "./services/api-client";

apiClient.setToken("jwt-token");
const users = await apiClient.get("/api/users");
```

#### Синглтон-стор через Zustand

```javascript
// stores/auth-store.js
import { create } from "zustand";

// Zustand-стор — синглтон по дизайну
export const useAuthStore = create((set, get) => ({
  user: null,
  token: null,
  isAuthenticated: false,

  login: async (credentials) => {
    const { user, token } = await apiClient.post("/auth/login", credentials);
    set({ user, token, isAuthenticated: true });
  },

  logout: () => {
    set({ user: null, token: null, isAuthenticated: false });
  },
}));
```

```jsx
// Все компоненты работают с одним и тем же стором
function Header() {
  const { user, isAuthenticated, logout } = useAuthStore();
  return (
    <header>
      {isAuthenticated ? (
        <>
          <span>{user.name}</span>
          <button onClick={logout}>Выйти</button>
        </>
      ) : (
        <a href="/login">Войти</a>
      )}
    </header>
  );
}
```

#### Синглтон через Context (для SSR-безопасности)

```jsx
// contexts/theme-context.jsx
import { createContext, use, useState, useMemo } from "react";

const ThemeContext = createContext(null);

export function ThemeProvider({ children, defaultTheme = "light" }) {
  const [theme, setTheme] = useState(defaultTheme);

  const value = useMemo(() => ({
    theme,
    setTheme,
    toggle: () => setTheme((t) => (t === "light" ? "dark" : "light")),
  }), [theme]);

  return (
    <ThemeContext value={value}>
      {children}
    </ThemeContext>
  );
}

export function useTheme() {
  const ctx = use(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}
```

```jsx
// Один провайдер на всё приложение — единый источник истины
function App() {
  return (
    <ThemeProvider defaultTheme="light">
      <Layout />
    </ThemeProvider>
  );
}
```

### В Vue

#### Модуль как синглтон

```javascript
// stores/counter.js
import { reactive } from "vue";

// Модуль — синглтон. reactive-объект создаётся один раз.
export const counterState = reactive({
  count: 0,
  increment() { counterState.count++; },
  decrement() { counterState.count--; },
});
```

```vue
<!-- Все компоненты работают с одним состоянием -->
<script setup>
import { counterState } from "./stores/counter";
</script>

<template>
  <p>Count: {{ counterState.count }}</p>
  <button @click="counterState.increment()">+</button>
</template>
```

#### Pinia-стор как синглтон

```javascript
// stores/auth.js
import { defineStore } from "pinia";
import { ref, computed } from "vue";

export const useAuthStore = defineStore("auth", () => {
  const user = ref(null);
  const token = ref(null);
  const isAuthenticated = computed(() => !!token.value);

  async function login(credentials) {
    const response = await api.post("/auth/login", credentials);
    user.value = response.user;
    token.value = response.token;
  }

  function logout() {
    user.value = null;
    token.value = null;
  }

  return { user, token, isAuthenticated, login, logout };
});
```

```vue
<script setup>
import { useAuthStore } from "./stores/auth";

const auth = useAuthStore();
</script>

<template>
  <div v-if="auth.isAuthenticated">
    <span>{{ auth.user.name }}</span>
    <button @click="auth.logout()">Выйти</button>
  </div>
  <router-link v-else to="/login">Войти</router-link>
</template>
```

#### provide/inject для синглтона в дереве компонентов

```javascript
// composables/useLogger.js
import { provide, inject } from "vue";

const LOGGER_KEY = Symbol("logger");

class Logger {
  log(message) {
    console.log(`[LOG ${new Date().toISOString()}] ${message}`);
  }
  error(message) {
    console.error(`[ERR ${new Date().toISOString()}] ${message}`);
  }
}

export function provideLogger() {
  provide(LOGGER_KEY, new Logger());
}

export function useLogger() {
  const logger = inject(LOGGER_KEY);
  if (!logger) throw new Error("Logger not provided");
  return logger;
}
```

```vue
<!-- App.vue — один экземпляр на дерево -->
<script setup>
import { provideLogger } from "./composables/useLogger";
provideLogger();
</script>
```

### Когда использовать

- **Глобальные сервисы** — API-клиент, логгер, аналитика
- **Состояние приложения** — сторы (Zustand, Pinia, Redux)
- **Конфигурация** — единая конфигурация для всего приложения
- **Кэш** — единый кэш запросов (TanStack Query, RTK Query)

### Когда НЕ использовать

- Для состояния, которое должно быть изолировано между компонентами
- Когда нужна возможность подмены в тестах — используйте DI или Context
- В SSR — глобальное состояние «протекает» между запросами

> ⚠️ **SSR-предостережение:** При серверном рендеринге синглтоны опасны — модули загружаются один раз на уровне сервера, и состояние может «протекать» между запросами разных пользователей. Используйте Request-scoped контексты или пересоздавайте сторы для каждого запроса.

---

## Наблюдатель

### Суть паттерна

**Наблюдатель (Observer)** определяет зависимость «один ко многим» между объектами. Когда один объект (subject) меняет состояние, все зависимые объекты (observers) автоматически уведомляются и обновляются.

### Классический пример

```javascript
class EventEmitter {
  constructor() {
    this.listeners = {};
  }

  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
  }

  off(event, callback) {
    this.listeners[event] = this.listeners[event]?.filter(cb => cb !== callback);
  }

  emit(event, data) {
    this.listeners[event]?.forEach(callback => callback(data));
  }
}

// Subject (издатель)
class WeatherStation extends EventEmitter {
  setTemperature(temp) {
    this.temperature = temp;
    this.emit("temperatureChange", temp);
  }
}

// Observers (подписчики)
const station = new WeatherStation();

station.on("temperatureChange", (temp) => {
  console.log(`Телефон: температура ${temp}°C`);
});

station.on("temperatureChange", (temp) => {
  if (temp > 30) console.log("Кондиционер: включение");
});

station.setTemperature(32);
// Телефон: температура 32°C
// Кондиционер: включение
```

### В React

В React паттерн Наблюдатель встроен в саму библиотеку — **реактивность состояния** это и есть Observer.

#### useState/useReducer — встроенный Observer

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  // count — это "subject"
  // Компонент — "observer", автоматически перерендеривается при изменении count

  return <button onClick={() => setCount(c => c + 1)}>Clicked {count} times</button>;
}
```

#### Context API — Observer для дерева компонентов

```jsx
const ThemeContext = createContext("light");

// Subject — ThemeContext
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext value={theme}>
      <Toolbar />
    </ThemeContext>
  );
}

// Observers — все компоненты, использующие use(ThemeContext)
function ThemedButton() {
  const theme = use(ThemeContext); // подписка на изменения
  return <button className={`btn btn--${theme}`}>Click</button>;
}
```

#### Внешние сторы — Observer через подписки

```javascript
// stores/event-store.js
// Простой стор на основе паттерна Observer

function createObservableStore(initialState) {
  let state = initialState;
  const listeners = new Set();

  return {
    getState() {
      return state;
    },

    setState(updater) {
      state = typeof updater === "function" ? updater(state) : updater;
      listeners.forEach((listener) => listener(state));
    },

    subscribe(listener) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

export const store = createObservableStore({ count: 0, user: null });
```

```jsx
// Хук для подписки — связывает React с Observer-стором
function useStore(selector) {
  const [state, setState] = useState(() => selector(store.getState()));

  useEffect(() => {
    const unsubscribe = store.subscribe((newState) => {
      const selected = selector(newState);
      setState(selected);
    });
    return unsubscribe;
  }, [selector]);

  return state;
}

// Использование
function Counter() {
  const count = useStore((state) => state.count);
  return (
    <button onClick={() => store.setState((s) => ({ ...s, count: s.count + 1 }))}>
      {count}
    </button>
  );
}
```

#### EventEmitter для межкомпонентного общения

```jsx
// services/event-bus.js
class EventBus {
  constructor() {
    this.listeners = {};
  }

  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
    return () => this.off(event, callback);
  }

  off(event, callback) {
    this.listeners[event] = this.listeners[event]?.filter((cb) => cb !== callback);
  }

  emit(event, data) {
    this.listeners[event]?.forEach((cb) => cb(data));
  }
}

export const eventBus = new EventBus();
```

```jsx
// Компонент-издатель
function NotificationButton() {
  const handleClick = () => {
    eventBus.emit("notification", { message: "Действие выполнено!", type: "success" });
  };

  return <button onClick={handleClick}>Выполнить</button>;
}

// Компонент-подписчик
function NotificationToast() {
  const [notification, setNotification] = useState(null);

  useEffect(() => {
    const unsubscribe = eventBus.on("notification", (data) => {
      setNotification(data);
      setTimeout(() => setNotification(null), 3000);
    });
    return unsubscribe;
  }, []);

  if (!notification) return null;

  return (
    <div className={`toast toast--${notification.type}`}>
      {notification.message}
    </div>
  );
}
```

#### Хук useSyncExternalStore (React 18+)

React 18 предоставляет встроенный хук для интеграции с внешними Observer-сторами:

```jsx
import { useSyncExternalStore } from "react";

function subscribe(callback) {
  const unsubscribe = store.subscribe(callback);
  return unsubscribe;
}

function getSnapshot() {
  return store.getState();
}

function Counter() {
  const state = useSyncExternalStore(subscribe, getSnapshot);
  return <p>Count: {state.count}</p>;
}
```

### В Vue

Во Vue паттерн Наблюдатель — основа всей реактивной системы.

#### Реактивность Vue — встроенный Observer

```vue
<script setup>
import { ref, watch } from "vue";

// ref — "subject"
const count = ref(0);

// watch — "observer", автоматически срабатывает при изменении count
watch(count, (newVal, oldVal) => {
  console.log(`Count changed: ${oldVal} -> ${newVal}`);
});

// Компонент тоже observer — автоматически перерендеривается
</script>

<template>
  <button @click="count++">Clicked {{ count }} times</button>
</template>
```

#### EventEmitter для межкомпонентного общения

```javascript
// composables/useEventBus.js
import { reactive } from "vue";

class VueEventBus {
  constructor() {
    this.listeners = {};
  }

  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
    return () => this.off(event, callback);
  }

  off(event, callback) {
    this.listeners[event] = this.listeners[event]?.filter((cb) => cb !== callback);
  }

  emit(event, data) {
    this.listeners[event]?.forEach((cb) => cb(data));
  }
}

// Синглтон event bus
export const eventBus = new VueEventBus();
```

```vue
<!-- Компонент-издатель -->
<script setup>
import { eventBus } from "./composables/useEventBus";

function handleAction() {
  eventBus.emit("notification", { message: "Готово!", type: "success" });
}
</script>

<template>
  <button @click="handleAction">Выполнить</button>
</template>
```

```vue
<!-- Компонент-подписчик -->
<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import { eventBus } from "./composables/useEventBus";

const notification = ref(null);
let unsubscribe = null;

onMounted(() => {
  unsubscribe = eventBus.on("notification", (data) => {
    notification.value = data;
    setTimeout(() => { notification.value = null; }, 3000);
  });
});

onUnmounted(() => {
  unsubscribe?.();
});
</script>

<template>
  <div v-if="notification" :class="`toast toast--${notification.type}`">
    {{ notification.message }}
  </div>
</template>
```

#### provide/inject + реактивность — Observer для дерева

```javascript
// composables/useTheme.js
import { ref, provide, inject, readonly } from "vue";

const THEME_KEY = Symbol("theme");

export function provideTheme(defaultTheme = "light") {
  const theme = ref(defaultTheme);

  function toggle() {
    theme.value = theme.value === "light" ? "dark" : "light";
  }

  provide(THEME_KEY, { theme: readonly(theme), toggle });
}

export function useTheme() {
  const themeContext = inject(THEME_KEY);
  if (!themeContext) throw new Error("Theme not provided");
  return themeContext;
}
```

```vue
<!-- App.vue — subject -->
<script setup>
import { provideTheme } from "./composables/useTheme";
provideTheme("light");
</script>
```

```vue
<!-- ThemedButton.vue — observer -->
<script setup>
import { useTheme } from "./composables/useTheme";
const { theme, toggle } = useTheme();
</script>

<template>
  <button :class="`btn btn--${theme}`" @click="toggle">
    Текущая тема: {{ theme }}
  </button>
</template>
```

### Когда использовать

- **Реактивное состояние** — автоматическое обновление UI при изменении данных
- **Межкомпонентное общение** — события между несвязанными компонентами
- **Подписки на внешние источники** — WebSocket, DOM-события, таймеры
- **Система уведомлений** — toast-уведомления, модальные окна

### Когда НЕ использовать

- Когда компоненты связаны иерархически — используйте пропсы/emits
- Для простых случаев — Context/provide-inject проще, чем event bus
- Когда подписчиков слишком много — сложно отлаживать поток данных

---

## Сравнительная таблица

| Паттерн | Классический GoF | React | Vue |
|---------|-----------------|-------|-----|
| **Абстрактная фабрика** | Классы с наследованием | Context + фабричные объекты, render-пропсы | provide/inject + динамические компоненты |
| **Декоратор** | Обёртка объекта | HOC, хуки, JSX-обёртки | Директивы, composables, обёрточные компоненты |
| **Синглтон** | Статический экземпляр | ES-модули, Zustand-сторы, Context | ES-модули, Pinia-сторы, provide/inject |
| **Наблюдатель** | Subject + Observer list | useState, Context, useSyncExternalStore | ref/reactive, watch, provide/inject |

---

## Заключение

Классические паттерны GoF не устарели — они **трансформировались** под реалии frontend-разработки:

- **Абстрактная фабрика** из классов с наследованием превратилась в фабрики компонентов через Context и provide/inject
- **Декоратор** из обёртки объекта стал HOC, хуком, директивой или JSX-обёрткой
- **Синглтон** из статического экземпляра стал ES-модулем, стором или провайдером контекста
- **Наблюдатель** из ручной подписки стал встроенной реактивностью фреймворков

Ключевой принцип остаётся тем же: **паттерны — это не про код, а про решения повторяющихся проблем**. Понимание классических паттернов помогает принимать правильные архитектурные решения, даже если их реализация выглядит иначе, чем в учебнике 1994 года.
