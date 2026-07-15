# Zustand — Глубокое погружение

## Содержание

1. [Что такое Zustand](#что-такое-zustand)
2. [Какую проблему решает Zustand](#какую-проблему-решает-zustand)
3. [Ключевые понятия](#ключевые-понятия)
4. [Базовое использование](#базовое-использование)
5. [Сценарии применения](#сценарии-применения)
6. [Продвинутые возможности](#продвинутые-возможности)
7. [Zustand vs TanStack Query](#zustand-vs-tanstack-query)
8. [Миграция с Redux](#миграция-с-redux)
9. [Лучшие практики](#лучшие-практики)
10. [Антипаттерны](#антипаттерны)

---

## Что такое Zustand

Zustand — это **минималистичная библиотека управления состоянием** для React (и не только). Название происходит от немецкого «Zustand» — «состояние». Она предоставляет простой API для создания глобального состояния без бойлерплейта, провайдеров и обёрток.

```jsx
import { create } from "zustand";

const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
}));

function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  return <button onClick={increment}>Count: {count}</button>;
}
```

Ключевые особенности Zustand:
- **Нет провайдеров** — не нужно оборачивать приложение в `<Provider>`.
- **Минимальный бойлерплейт** — один хук `create` для создания хранилища.
- **Селекторы** — компоненты подписываются только на нужные части состояния.
- **Работает вне React** — доступ к состоянию из любого места (обработчики событий, утилиты, middleware).
- **Размер** — ~1 KB (minified + gzipped).

---

## Какую проблему решает Zustand

### Проблема 1: Prop Drilling

Без глобального состояния данные приходится передавать через множество промежуточных компонентов:

```jsx
// ❌ Prop drilling: тема передаётся через 5 уровней
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

// ... и так далее
```

Zustand решает это, предоставляя глобальное хранилище, доступное из любого компонента:

```jsx
// ✅ Zustand: доступ к состоянию без пробрасывания пропсов
const useThemeStore = create((set) => ({
  theme: "light",
  toggleTheme: () => set((state) => ({
    theme: state.theme === "light" ? "dark" : "light"
  })),
}));

function Header() {
  const theme = useThemeStore((state) => state.theme);
  const toggleTheme = useThemeStore((state) => state.toggleTheme);
  return <button onClick={toggleTheme}>Current: {theme}</button>;
}
```

### Проблема 2: Избыточный бойлерплейт Redux

Redux требует определения действий, редукторов, создателей действий, типов (в TypeScript) и обёртки в `<Provider>`:

```jsx
// ❌ Redux: много бойлерплейта для простого счётчика
// actions.js
export const INCREMENT = "INCREMENT";
export const increment = () => ({ type: INCREMENT });

// reducers.js
export const counterReducer = (state = 0, action) => {
  switch (action.type) {
    case INCREMENT: return state + 1;
    default: return state;
  }
};

// store.js
export const store = createStore(counterReducer);

// App.jsx
<Provider store={store}>
  <App />
</Provider>

// Counter.jsx
const count = useSelector((state) => state);
const dispatch = useDispatch();
dispatch(increment());
```

Zustand упрощает это до нескольких строк:

```jsx
// ✅ Zustand: тот же счётчик без бойлерплейта
const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  return <button onClick={increment}>Count: {count}</button>;
}
```

### Проблема 3: Контекст для высокочастотных обновлений

React Context не оптимизирован для частых обновлений — при изменении значения все потребители контекста перерендериваются:

```jsx
// ❌ Context: все потребители перерендериваются при каждом изменении
const CounterContext = createContext();

function App() {
  const [count, setCount] = useState(0);
  return (
    <CounterContext value={{ count, setCount }}>
      <Counter />
      <AnotherComponent />
    </CounterContext>
  );
}

function Counter() {
  const { count } = use(CounterContext); // Перерендер при каждом изменении count
  return <div>{count}</div>;
}
```

Zustand использует селекторы, позволяя компонентам подписываться только на нужные части состояния:

```jsx
// ✅ Zustand: перерендер только при изменении нужного значения
const useCounterStore = create((set) => ({
  count: 0,
  name: "Counter",
  increment: () => set((state) => ({ count: state.count + 1 })),
  setName: (name) => set({ name }),
}));

function Counter() {
  const count = useCounterStore((state) => state.count);
  // ✅ Перерендер только при изменении count, не name
  return <div>{count}</div>;
}
```

---

## Ключевые понятия

### Хранилище (Store)

Хранилище — это объект, содержащий состояние и действия для его обновления. Создаётся с помощью `create`:

```jsx
import { create } from "zustand";

const useStore = create((set, get) => ({
  // Состояние
  user: null,
  theme: "light",
  
  // Действия
  setUser: (user) => set({ user }),
  setTheme: (theme) => set({ theme }),
  
  // Доступ к текущему состоянию через get()
  resetUser: () => {
    const currentUser = get().user;
    console.log("Resetting user:", currentUser);
    set({ user: null });
  },
}));
```

### Селекторы

Селекторы позволяют компонентам подписываться только на нужные части состояния, предотвращая ненужные рендеры:

```jsx
function UserProfile() {
  // ✅ Подписка только на user
  const user = useStore((state) => state.user);
  return <div>{user?.name}</div>;
}

function ThemeToggle() {
  // ✅ Подписка только на theme
  const theme = useStore((state) => state.theme);
  const setTheme = useStore((state) => state.setTheme);
  return <button onClick={() => setTheme("dark")}>{theme}</button>;
}
```

Без селекторов компонент перерендеривается при любом изменении хранилища:

```jsx
function BadComponent() {
  // ❌ Перерендер при любом изменении хранилища
  const store = useStore();
  return <div>{store.user?.name}</div>;
}
```

### Immer для неизменяемости

Zustand по умолчанию поддерживает Immer для удобного обновления вложенных объектов:

```jsx
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

const useStore = create(
  immer((set) => ({
    user: {
      profile: {
        name: "Alice",
        settings: { notifications: true },
      },
    },
    
    // ✅ Мутации разрешены благодаря Immer
    updateNotification: (enabled) =>
      set((state) => {
        state.user.profile.settings.notifications = enabled;
      }),
  }))
);
```

### Middleware

Zustand поддерживает middleware для расширения функциональности:

```jsx
import { create } from "zustand";
import { persist, devtools, logger } from "zustand/middleware";

const useStore = create(
  devtools(
    persist(
      (set) => ({
        count: 0,
        increment: () => set((state) => ({ count: state.count + 1 })),
      }),
      { name: "counter-storage" } // Сохранение в localStorage
    )
  )
);
```

Популярные middleware:
- `persist` — сохранение состояния в localStorage/sessionStorage
- `devtools` — интеграция с Redux DevTools
- `logger` — логирование изменений состояния
- `immer` — поддержка мутаций через Immer
- `combine` — объединение нескольких хранилищ

---

## Базовое использование

### Создание хранилища

```jsx
import { create } from "zustand";

const useAuthStore = create((set) => ({
  // Состояние
  user: null,
  isAuthenticated: false,
  
  // Действия
  login: async (credentials) => {
    const user = await api.login(credentials);
    set({ user, isAuthenticated: true });
  },
  
  logout: () => {
    set({ user: null, isAuthenticated: false });
  },
}));
```

### Использование в компонентах

```jsx
function LoginPage() {
  const login = useAuthStore((state) => state.login);
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated);
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    await login({
      email: formData.get("email"),
      password: formData.get("password"),
    });
  };
  
  if (isAuthenticated) return <Redirect to="/dashboard" />;
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button type="submit">Login</button>
    </form>
  );
}

function UserProfile() {
  const user = useAuthStore((state) => state.user);
  const logout = useAuthStore((state) => state.logout);
  
  return (
    <div>
      <h1>{user?.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### Доступ вне React

Zustand позволяет работать с состоянием вне компонентов React:

```jsx
// Утилиты, обработчики событий, middleware
import { useAuthStore } from "./stores/auth";

function setupInterceptors() {
  axios.interceptors.request.use((config) => {
    const user = useAuthStore.getState().user;
    if (user) {
      config.headers.Authorization = `Bearer ${user.token}`;
    }
    return config;
  });
}

// Обработчик события вне компонента
document.addEventListener("visibilitychange", () => {
  if (document.hidden) {
    useAuthStore.getState().logout();
  }
});
```

---

## Сценарии применения

### 1. Глобальное UI-состояние

Zustand идеально подходит для состояния, которое должно быть доступно во всём приложении:

```jsx
import { create } from "zustand";

const useUIStore = create((set) => ({
  // Модальные окна
  modals: {
    login: false,
    settings: false,
    confirm: false,
  },
  
  openModal: (modal) => set((state) => ({
    modals: { ...state.modals, [modal]: true },
  })),
  
  closeModal: (modal) => set((state) => ({
    modals: { ...state.modals, [modal]: false },
  })),
  
  // Боковая панель
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({
    sidebarOpen: !state.sidebarOpen,
  })),
  
  // Уведомления
  notifications: [],
  addNotification: (notification) => set((state) => ({
    notifications: [...state.notifications, notification],
  })),
  removeNotification: (id) => set((state) => ({
    notifications: state.notifications.filter((n) => n.id !== id),
  })),
}));

// Использование
function Header() {
  const openModal = useUIStore((state) => state.openModal);
  const sidebarOpen = useUIStore((state) => state.sidebarOpen);
  const toggleSidebar = useUIStore((state) => state.toggleSidebar);
  
  return (
    <header>
      <button onClick={toggleSidebar}>
        {sidebarOpen ? "Close" : "Open"} Sidebar
      </button>
      <button onClick={() => openModal("login")}>Login</button>
    </header>
  );
}
```

### 2. Состояние формы

Для сложных форм с множеством полей и валидацией:

```jsx
import { create } from "zustand";

const useFormStore = create((set) => ({
  values: {
    name: "",
    email: "",
    password: "",
  },
  
  errors: {},
  
  touched: {},
  
  setValue: (field, value) => set((state) => ({
    values: { ...state.values, [field]: value },
    touched: { ...state.touched, [field]: true },
  })),
  
  setErrors: (errors) => set({ errors }),
  
  validate: () => {
    const errors = {};
    const { values } = useStore.getState();
    
    if (!values.name) errors.name = "Name is required";
    if (!values.email) errors.email = "Email is required";
    if (!values.password) errors.password = "Password is required";
    
    set({ errors });
    return Object.keys(errors).length === 0;
  },
  
  reset: () => set({
    values: { name: "", email: "", password: "" },
    errors: {},
    touched: {},
  }),
}));

function RegistrationForm() {
  const values = useFormStore((state) => state.values);
  const errors = useFormStore((state) => state.errors);
  const touched = useFormStore((state) => state.touched);
  const setValue = useFormStore((state) => state.setValue);
  const validate = useFormStore((state) => state.validate);
  
  const handleSubmit = (e) => {
    e.preventDefault();
    if (validate()) {
      // Отправка формы
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={values.name}
        onChange={(e) => setValue("name", e.target.value)}
      />
      {touched.name && errors.name && <span>{errors.name}</span>}
      
      <input
        name="email"
        type="email"
        value={values.email}
        onChange={(e) => setValue("email", e.target.value)}
      />
      {touched.email && errors.email && <span>{errors.email}</span>}
      
      <button type="submit">Register</button>
    </form>
  );
}
```

### 3. Состояние корзины (E-commerce)

```jsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useCartStore = create(
  persist(
    (set, get) => ({
      items: [],
      
      addItem: (product) => set((state) => {
        const existing = state.items.find((i) => i.id === product.id);
        if (existing) {
          return {
            items: state.items.map((i) =>
              i.id === product.id ? { ...i, quantity: i.quantity + 1 } : i
            ),
          };
        }
        return { items: [...state.items, { ...product, quantity: 1 }] };
      }),
      
      removeItem: (productId) => set((state) => ({
        items: state.items.filter((i) => i.id !== productId),
      })),
      
      updateQuantity: (productId, quantity) => set((state) => ({
        items: state.items.map((i) =>
          i.id === productId ? { ...i, quantity } : i
        ),
      })),
      
      clearCart: () => set({ items: [] }),
      
      // Вычисляемые значения
      get totalItems() {
        return get().items.reduce((sum, i) => sum + i.quantity, 0);
      },
      
      get totalPrice() {
        return get().items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      },
    }),
    { name: "cart-storage" } // Сохранение в localStorage
  )
);

function CartIcon() {
  const totalItems = useCartStore((state) => state.items.length);
  return <span>Cart ({totalItems})</span>;
}

function CartPage() {
  const items = useCartStore((state) => state.items);
  const removeItem = useCartStore((state) => state.removeItem);
  const updateQuantity = useCartStore((state) => state.updateQuantity);
  
  return (
    <div>
      {items.map((item) => (
        <div key={item.id}>
          <h3>{item.name}</h3>
          <p>Price: ${item.price}</p>
          <input
            type="number"
            value={item.quantity}
            onChange={(e) => updateQuantity(item.id, Number(e.target.value))}
          />
          <button onClick={() => removeItem(item.id)}>Remove</button>
        </div>
      ))}
    </div>
  );
}
```

### 4. Состояние аутентификации

```jsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useAuthStore = create(
  persist(
    (set) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      
      login: async (credentials) => {
        try {
          const response = await api.post("/auth/login", credentials);
          const { user, token } = response.data;
          set({ user, token, isAuthenticated: true });
          return { success: true };
        } catch (error) {
          return { success: false, error: error.message };
        }
      },
      
      logout: () => {
        set({ user: null, token: null, isAuthenticated: false });
        // Очистка токена из заголовков axios
        delete axios.defaults.headers.common["Authorization"];
      },
      
      setUser: (user) => set({ user }),
    }),
    {
      name: "auth-storage",
      partialize: (state) => ({ token: state.token }), // Сохраняем только токен
    }
  )
);

function ProtectedRoute({ children }) {
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated);
  const navigate = useNavigate();
  
  useEffect(() => {
    if (!isAuthenticated) {
      navigate("/login");
    }
  }, [isAuthenticated, navigate]);
  
  return isAuthenticated ? children : null;
}

function LoginPage() {
  const login = useAuthStore((state) => state.login);
  const navigate = useNavigate();
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    const result = await login({
      email: formData.get("email"),
      password: formData.get("password"),
    });
    
    if (result.success) {
      navigate("/dashboard");
    } else {
      alert(result.error);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" required />
      <input name="password" type="password" required />
      <button type="submit">Login</button>
    </form>
  );
}
```

### 5. Состояние темы и настроек

```jsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: "light",
      language: "en",
      fontSize: 16,
      
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      setFontSize: (fontSize) => set({ fontSize }),
      
      resetSettings: () => set({
        theme: "light",
        language: "en",
        fontSize: 16,
      }),
    }),
    { name: "settings-storage" }
  )
);

function ThemeProvider({ children }) {
  const theme = useSettingsStore((state) => state.theme);
  
  useEffect(() => {
    document.documentElement.setAttribute("data-theme", theme);
  }, [theme]);
  
  return children;
}

function ThemeToggle() {
  const theme = useSettingsStore((state) => state.theme);
  const setTheme = useSettingsStore((state) => state.setTheme);
  
  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      Current theme: {theme}
    </button>
  );
}
```

---

## Продвинутые возможности

### Несколько хранилищ

Zustand позволяет создавать несколько независимых хранилищ:

```jsx
// stores/auth.js
export const useAuthStore = create((set) => ({
  user: null,
  login: (user) => set({ user }),
  logout: () => set({ user: null }),
}));

// stores/cart.js
export const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
}));

// stores/ui.js
export const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

### Слайсы (Slices)

Для больших хранилищ можно разделить логику на слайсы:

```jsx
import { create } from "zustand";

// Слайс для пользователя
const createUserSlice = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  clearUser: () => set({ user: null }),
});

// Слайс для корзины
const createCartSlice = (set) => ({
  cart: [],
  addToCart: (item) => set((state) => ({ cart: [...state.cart, item] })),
  clearCart: () => set({ cart: [] }),
});

// Объединение слайсов
const useStore = create((...args) => ({
  ...createUserSlice(...args),
  ...createCartSlice(...args),
}));

// Использование
function Component() {
  const user = useStore((state) => state.user);
  const cart = useStore((state) => state.cart);
  const addToCart = useStore((state) => state.addToCart);
}
```

### Асинхронные действия

Zustand поддерживает асинхронные действия:

```jsx
const useDataStore = create((set) => ({
  data: null,
  loading: false,
  error: null,
  
  fetchData: async (id) => {
    set({ loading: true, error: null });
    try {
      const response = await api.get(`/data/${id}`);
      set({ data: response.data, loading: false });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  },
  
  reset: () => set({ data: null, loading: false, error: null }),
}));

function DataComponent({ id }) {
  const data = useDataStore((state) => state.data);
  const loading = useDataStore((state) => state.loading);
  const error = useDataStore((state) => state.error);
  const fetchData = useDataStore((state) => state.fetchData);
  
  useEffect(() => {
    fetchData(id);
  }, [id, fetchData]);
  
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <DataView data={data} />;
}
```

> ⚠️ **Важно:** Для сложных сценариев загрузки данных (кэширование, повторные запросы, оптимистичные обновления) лучше использовать [TanStack Query](./tanstack_query.md). Zustand подходит для простого асинхронного состояния.

### Persist middleware

Сохранение состояния в localStorage:

```jsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useStore = create(
  persist(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 })),
    }),
    {
      name: "counter-storage", // Ключ в localStorage
      partialize: (state) => ({ count: state.count }), // Что сохранять
      onRehydrateStorage: () => {
        console.log("Hydration started");
        return (state, error) => {
          if (error) console.error("Hydration failed:", error);
          else console.log("Hydration complete:", state);
        };
      },
    }
  )
);
```

### Devtools middleware

Интеграция с Redux DevTools:

```jsx
import { create } from "zustand";
import { devtools } from "zustand/middleware";

const useStore = create(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 })),
    }),
    {
      name: "MyStore", // Имя хранилища в DevTools
      enabled: process.env.NODE_ENV === "development",
    }
  )
);
```

### TypeScript

Zustand отлично работает с TypeScript:

```tsx
import { create } from "zustand";

interface User {
  id: number;
  name: string;
  email: string;
}

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: { email: string; password: string }) => Promise<void>;
  logout: () => void;
}

const useAuthStore = create<AuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  
  login: async (credentials) => {
    const user = await api.login(credentials);
    set({ user, isAuthenticated: true });
  },
  
  logout: () => {
    set({ user: null, isAuthenticated: false });
  },
}));

// Использование с типами
function UserProfile() {
  const user = useAuthStore((state) => state.user);
  // user: User | null
  return <div>{user?.name}</div>;
}
```

---

## Zustand vs TanStack Query

Zustand и [TanStack Query](./tanstack_query.md) решают разные задачи. Понимание различий критично для правильного выбора.

### Когда использовать Zustand

Zustand подходит для **клиентского состояния** (UI state):

| Сценарий | Пример |
|---|---|
| Глобальное UI-состояние | Модальные окна, боковые панели, уведомления |
| Состояние форм | Значения полей, валидация, отправка |
| Состояние корзины | Товары, количество, общая сумма |
| Аутентификация | Текущий пользователь, токен |
| Настройки приложения | Тема, язык, размер шрифта |
| Состояние, не связанное с сервером | Счётчики, таймеры, локальные данные |

```jsx
// ✅ Zustand для UI-состояния
const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

### Когда использовать TanStack Query

TanStack Query подходит для **серверного состояния** (server state):

| Сценарий | Пример |
|---|---|
| Загрузка данных с API | GET-запросы, списки, детали |
| Кэширование | Избежание повторных запросов |
| Мутации | POST, PUT, DELETE с оптимистичными обновлениями |
| Пагинация и бесконечная прокрутка | Списки с курсором или offset |
| Фоновое обновление | Автоматическое обновление данных |
| Инвалидация кэша | Обновление данных после мутаций |

```jsx
// ✅ TanStack Query для серверных данных
function UserList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["users"],
    queryFn: () => api.get("/users").then((res) => res.data),
  });
  
  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  
  return (
    <ul>
      {data.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Сравнительная таблица

| Характеристика | Zustand | TanStack Query |
|---|---|---|
| **Основное назначение** | Клиентское состояние | Серверное состояние |
| **Кэширование** | ❌ Нет встроенного | ✅ Встроенное кэширование |
| **Повторные запросы** | ❌ Ручная реализация | ✅ Автоматические |
| **Инвалидация кэша** | ❌ Ручная реализация | ✅ Автоматическая |
| **Оптимистичные обновления** | ❌ Ручная реализация | ✅ Встроенные |
| **Пагинация** | ❌ Ручная реализация | ✅ Встроенная |
| **Фоновое обновление** | ❌ Ручная реализация | ✅ Автоматическое |
| **Размер** | ~1 KB | ~15 KB |
| **Сложность** | Минимальная | Средняя |
| **Интеграция с React** | Хуки | Хуки + QueryClient |

### Рекомендуемый паттерн

Современный подход 2026 года: **использовать оба инструмента вместе**:

```jsx
// Zustand для клиентского состояния
const useUIStore = create((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));

// TanStack Query для серверных данных
function UserList() {
  const { data: users, isLoading } = useQuery({
    queryKey: ["users"],
    queryFn: fetchUsers,
  });
  
  const sidebarOpen = useUIStore((state) => state.sidebarOpen);
  
  return (
    <div>
      <button onClick={() => useUIStore.getState().toggleSidebar()}>
        Toggle Sidebar
      </button>
      {sidebarOpen && <Sidebar />}
      {isLoading ? <Spinner /> : <UserGrid users={users} />}
    </div>
  );
}
```

### Когда можно обойтись одним

**Только Zustand:**
- Простые приложения без серверного взаимодействия
- Статические сайты с минимальной динамикой
- Библиотеки компонентов без загрузки данных

**Только TanStack Query:**
- Приложения, где всё состояние связано с сервером
- Нет глобального UI-состояния (кроме локального состояния компонентов)

**Оба инструмента:**
- Большинство современных приложений
- Сложные UI с модальными окнами, боковыми панелями, уведомлениями
- Приложения с аутентификацией и пользовательскими настройками

> 💡 **Правило:** Если данные приходят с сервера — используйте TanStack Query. Если данные создаются и живут только в браузере — используйте Zustand.

---

## Миграция с Redux

Zustand может заменить Redux в большинстве проектов. Вот как мигрировать:

### Redux

```jsx
// actions.js
export const INCREMENT = "INCREMENT";
export const increment = () => ({ type: INCREMENT });

// reducers.js
export const counterReducer = (state = 0, action) => {
  switch (action.type) {
    case INCREMENT: return state + 1;
    default: return state;
  }
};

// store.js
export const store = createStore(counterReducer);

// App.jsx
<Provider store={store}>
  <App />
</Provider>

// Counter.jsx
const count = useSelector((state) => state);
const dispatch = useDispatch();
dispatch(increment());
```

### Zustand (эквивалент)

```jsx
// stores/counter.js
export const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// Counter.jsx
const count = useCounterStore((state) => state.count);
const increment = useCounterStore((state) => state.increment);
increment();
```

### Redux Toolkit

```jsx
// counterSlice.js
const counterSlice = createSlice({
  name: "counter",
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },
  },
});

export const { increment } = counterSlice.actions;
export default counterSlice.reducer;

// store.js
export const store = configureStore({
  reducer: { counter: counterReducer },
});

// App.jsx
<Provider store={store}>
  <App />
</Provider>

// Counter.jsx
const count = useSelector((state) => state.counter.value);
const dispatch = useDispatch();
dispatch(increment());
```

### Zustand (эквивалент Redux Toolkit)

```jsx
// stores/counter.js
export const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// Counter.jsx
const count = useCounterStore((state) => state.count);
const increment = useCounterStore((state) => state.increment);
increment();
```

Преимущества миграции на Zustand:
- Меньше бойлерплейта
- Нет необходимости в `<Provider>`
- Проще тестировать
- Меньше зависимостей

---

## Лучшие практики

### 1. Используйте селекторы

Всегда используйте селекторы для подписки на конкретные части состояния:

```jsx
// ❌ Плохо: перерендер при любом изменении хранилища
function Component() {
  const store = useStore();
  return <div>{store.user?.name}</div>;
}

// ✅ Хорошо: перерендер только при изменении user
function Component() {
  const user = useStore((state) => state.user);
  return <div>{user?.name}</div>;
}
```

### 2. Разделяйте хранилища по доменам

Не создавайте одно огромное хранилище — разделяйте по ответственности:

```jsx
// ❌ Плохо: одно хранилище для всего
const useStore = create((set) => ({
  user: null,
  cart: [],
  theme: "light",
  sidebarOpen: false,
  // ... десятки полей
}));

// ✅ Хорошо: отдельные хранилища
const useAuthStore = create((set) => ({ user: null }));
const useCartStore = create((set) => ({ cart: [] }));
const useSettingsStore = create((set) => ({ theme: "light" }));
const useUIStore = create((set) => ({ sidebarOpen: false }));
```

### 3. Используйте persist для пользовательских настроек

Сохраняйте настройки пользователя в localStorage:

```jsx
const useSettingsStore = create(
  persist(
    (set) => ({
      theme: "light",
      language: "en",
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    { name: "settings-storage" }
  )
);
```

### 4. Используйте devtools в разработке

Интегрируйте с Redux DevTools для отладки:

```jsx
const useStore = create(
  devtools((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
  }))
);
```

### 5. Тестируйте хранилища изолированно

Zustand легко тестировать без React:

```jsx
import { useCounterStore } from "./stores/counter";

test("increment updates count", () => {
  const { increment, getState } = useCounterStore;
  
  expect(getState().count).toBe(0);
  increment();
  expect(getState().count).toBe(1);
});
```

---

## Антипаттерны

### 1. Подписка на всё хранилище

```jsx
// ❌ Плохо: перерендер при любом изменении
function Component() {
  const store = useStore();
  return <div>{store.user?.name}</div>;
}

// ✅ Хорошо: подписка только на нужное
function Component() {
  const user = useStore((state) => state.user);
  return <div>{user?.name}</div>;
}
```

### 2. Создание хранилища внутри компонента

```jsx
// ❌ Плохо: хранилище пересоздаётся при каждом рендере
function Component() {
  const useStore = create((set) => ({ count: 0 }));
  const count = useStore((state) => state.count);
}

// ✅ Хорошо: хранилище создаётся вне компонента
const useStore = create((set) => ({ count: 0 }));

function Component() {
  const count = useStore((state) => state.count);
}
```

### 3. Использование Zustand для серверных данных

```jsx
// ❌ Плохо: ручная реализация кэширования, повторных запросов
const useDataStore = create((set) => ({
  data: null,
  loading: false,
  fetchData: async (id) => {
    set({ loading: true });
    const data = await api.get(`/data/${id}`);
    set({ data, loading: false });
  },
}));

// ✅ Хорошо: используйте TanStack Query для серверных данных
function Component({ id }) {
  const { data, isLoading } = useQuery({
    queryKey: ["data", id],
    queryFn: () => api.get(`/data/${id}`),
  });
}
```

### 4. Мутация состояния напрямую

```jsx
// ❌ Плохо: прямая мутация
const useStore = create((set) => ({
  items: [],
  addItem: (item) => {
    const state = useStore.getState();
    state.items.push(item); // ❌ Мутация
  },
}));

// ✅ Хорошо: создание нового объекта
const useStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, item],
  })),
}));
```

### 5. Избыточное использование middleware

```jsx
// ❌ Плохо: слишком много middleware
const useStore = create(
  devtools(
    persist(
      logger(
        immer((set) => ({ count: 0 }))
      )
    )
  )
);

// ✅ Хорошо: только необходимые middleware
const useStore = create(
  devtools(
    persist((set) => ({ count: 0 }), { name: "counter" })
  )
);
```

---

## Итог

Zustand — это **минималистичная и мощная библиотека** для управления клиентским состоянием в React. Она решает проблемы prop drilling и избыточного бойлерплейта Redux, предоставляя простой API без провайдеров и обёрток.

**Ключевые преимущества:**
- Минимальный бойлерплейт
- Нет провайдеров
- Селекторы для оптимизации рендеров
- Работает вне React
- Отличная поддержка TypeScript
- Middleware для расширения функциональности

**Когда использовать:**
- Глобальное UI-состояние (модальные окна, боковые панели)
- Состояние форм
- Корзина, аутентификация, настройки
- Любое состояние, не связанное с сервером

**Когда использовать TanStack Query:**
- Загрузка данных с API
- Кэширование и инвалидация
- Мутации с оптимистичными обновлениями
- Пагинация и бесконечная прокрутка

**Рекомендуемый паттерн:** Используйте Zustand для клиентского состояния и [TanStack Query](./tanstack_query.md) для серверного состояния. Вместе они заменяют Redux в большинстве современных приложений.
