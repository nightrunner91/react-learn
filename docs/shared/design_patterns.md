# Паттерны проектирования и структура файлов в React / Next.js

## Содержание

1. [Введение](#введение)
2. [Композиция компонентов](#композиция-компонентов)
3. [Compound Components](#compound-components)
4. [Render Props](#render-props)
5. [Custom Hooks как паттерн](#custom-hooks-как-паттерн)
6. [Container / Presentational](#container--presentational)
7. [State Reducer](#state-reducer)
8. [Control Props](#control-props)
9. [Props Collection](#props-collection)
10. [Структура файлов: подходы](#структура-файлов-подходы)
11. [Структура файлов в Next.js App Router](#структура-файлов-в-nextjs-app-router)
12. [Организация общего кода](#организация-общего-кода)
13. [Лучшие практики](#лучшие-практики)
14. [Антипаттерны](#антипаттерны)

---

## Введение

Паттерны проектирования в React — это проверенные подходы к организации компонентов, управлению состоянием и построению архитектуры приложения. Они не навязаны фреймворком, а сформировались в экосистеме как ответы на повторяющиеся задачи.

Понимание паттернов отличает код, который «работает», от кода, который легко расширять, тестировать и передавать другим разработчикам. На собеседовании уровня Middle+ вас спросят не «что такое компонент», а «как вы организуете архитектуру приложения» и «какие паттерны используете».

В этой статье мы разберём:
- **Паттерны компонентов** — как строить переиспользуемые, гибкие API компонентов
- **Архитектурные паттерны** — как организовать логику и данные
- **Структуру файлов** — как расположить код в проекте

> 💡 Многие паттерны из этой статьи в 2026 году частично заменены хуками и серверными компонентами, но они по-прежнему встречаются в кодовых базах и полезны для понимания дизайна API.

---

## Композиция компонентов

Композиция — главный паттерн React. Вместо наследования (как в ООП) React строит сложное поведение через вложение и передачу компонентов друг другу.

### children — базовая композиция

```jsx
function Card({ children }) {
  return (
    <div className="card">
      {children}
    </div>
  );
}

function App() {
  return (
    <Card>
      <h2>Заголовок</h2>
      <p>Содержимое карточки</p>
    </Card>
  );
}
```

### Именованные слоты через пропсы

Когда нужно несколько «слотов» для разных частей UI:

```jsx
function Dialog({ header, body, footer }) {
  return (
    <div className="dialog">
      <div className="dialog__header">{header}</div>
      <div className="dialog__body">{body}</div>
      <div className="dialog__footer">{footer}</div>
    </div>
  );
}

<Dialog
  header={<h2>Подтверждение</h2>}
  body={<p>Вы уверены?</p>}
  footer={<Button onClick={handleConfirm}>Да</Button>}
/>
```

### Композиция через render-пропсы (см. ниже)

Композиция позволяет менять поведение компонента без его изменения — ключевой принцип гибкой архитектуры.

---

## Compound Components

**Compound Components** (составные компоненты) — паттерн, при котором несколько компонентов работают вместе, разделяя неявное состояние через Context. Пользователь API собирает компонент из частей, получая максимальную гибкость.

### Зачем нужен

Представьте `<Select>` с фиксированным API:

```jsx
// ❌ Жёсткий API — нельзя кастомизировать internals
<Select
  items={items}
  renderItem={(item) => <span>{item.name}</span>}
  renderTrigger={(open) => <button>{open ? "Close" : "Open"}</button>}
/>
```

Compound-подход даёт полный контроль над структурой:

```jsx
// ✅ Compound Components — полная гибкость
<Select items={items}>
  <Select.Trigger>
    <Select.Icon />
    <Select.Value placeholder="Выберите..." />
  </Select.Trigger>
  <Select.Content>
    <Select.Search />
    <Select.List>
      {(item) => (
        <Select.Item key={item.id} value={item.id}>
          <Avatar src={item.avatar} />
          <span>{item.name}</span>
        </Select.Item>
      )}
    </Select.List>
  </Select.Content>
</Select>
```

### Реализация через Context

```jsx
const SelectContext = createContext(null);

function Select({ children, items, value, onChange }) {
  const [isOpen, setIsOpen] = useState(false);

  const context = {
    items,
    value,
    onChange,
    isOpen,
    setIsOpen,
  };

  return (
    <SelectContext value={context}>
      <div className="select">
        {children}
      </div>
    </SelectContext>
  );
}

function Trigger({ children }) {
  const { isOpen, setIsOpen } = use(SelectContext);
  return (
    <button onClick={() => setIsOpen(!isOpen)}>
      {children}
    </button>
  );
}

function Content({ children }) {
  const { isOpen } = use(SelectContext);
  if (!isOpen) return null;
  return <div className="select__content">{children}</div>;
}

function Item({ value, children }) {
  const { onChange, setIsOpen } = use(SelectContext);
  return (
    <div
      className="select__item"
      onClick={() => {
        onChange(value);
        setIsOpen(false);
      }}
    >
      {children}
    </div>
  );
}

Select.Trigger = Trigger;
Select.Content = Content;
Select.Item = Item;
```

### Когда использовать

- Библиотечные компоненты с кастомизацией (Radix UI, Headless UI)
- Компоненты, где количество вариантов конфигурации слишком велико для пропсов
- Когда нужно дать пользователю контроль над структурой DOM

### Когда НЕ использовать

- Простые компоненты с 2–3 пропсами
- Когда составные части не разделяют состояние

> 💡 **Популярные библиотеки на Compound Components:** Radix Primitives, Headless UI, Chakra UI, Ark UI. Все они предоставляют «headless» API — логику и доступность без стилей.

---

## Render Props

**Render Props** — паттерн, при котором компонент принимает функцию, возвращающую React-элемент. Эта функция вызывается внутри компонента с данными, которые нужно передать наружу.

```jsx
function MouseTracker({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handleMove = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", handleMove);
    return () => window.removeEventListener("mousemove", handleMove);
  }, []);

  return render(position);
}

<MouseTracker
  render={({ x, y }) => <p>Cursor: {x}, {y}</p>}
/>
```

### Render Props vs Custom Hooks

До появления хуков render props был основным способом переиспользования логики. В 2026 году **custom hooks заменили render props в большинстве случаев**:

```jsx
// ❌ Render Props (устаревший подход для переиспользования логики)
<MouseTracker render={({ x, y }) => <p>{x}, {y}</p>} />

// ✅ Custom Hook (современный подход)
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handleMove = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", handleMove);
    return () => window.removeEventListener("mousemove", handleMove);
  }, []);
  return position;
}

function Cursor() {
  const { x, y } = useMousePosition();
  return <p>Cursor: {x}, {y}</p>;
}
```

### Когда render props всё ещё полезен

- В библиотеках, где нужно инвертировать контроль над рендерингом
- Когда логика тесно связана с конкретным местом в JSX
- Для «inversion of control» в composition-паттернах

```jsx
function DataTable({ data, columns, renderRow }) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => <th key={col.key}>{col.title}</th>)}
        </tr>
      </thead>
      <tbody>
        {data.map((row, i) => (
          <tr key={i}>
            {renderRow(row)}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

---

## Custom Hooks как паттерн

Пользовательские хуки — основной паттерн переиспользования логики в современном React. Хук инкапсулирует состояние и побочные эффекты, возвращая чистый API.

### Паттерн: инкапсуляция DOM-логики

```jsx
function useClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (event) => {
      if (!ref.current || ref.current.contains(event.target)) return;
      handler(event);
    };
    document.addEventListener("mousedown", listener);
    document.addEventListener("touchstart", listener);
    return () => {
      document.removeEventListener("mousedown", listener);
      document.removeEventListener("touchstart", listener);
    };
  }, [ref, handler]);
}

function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const ref = useRef(null);
  useClickOutside(ref, () => setIsOpen(false));
  return <div ref={ref}>...</div>;
}
```

### Паттерн: абстракция API-запросов

```jsx
function useApi(url, options = {}) {
  const [state, setState] = useState({
    data: null,
    error: null,
    isLoading: true,
  });

  useEffect(() => {
    const controller = new AbortController();

    async function fetchData() {
      try {
        setState((s) => ({ ...s, isLoading: true }));
        const res = await fetch(url, { ...options, signal: controller.signal });
        if (!res.ok) throw new Error(res.statusText);
        const data = await res.json();
        setState({ data, error: null, isLoading: false });
      } catch (error) {
        if (error.name !== "AbortError") {
          setState({ data: null, error, isLoading: false });
        }
      }
    }

    fetchData();
    return () => controller.abort();
  }, [url]);

  return state;
}
```

### Паттерн: композиция хуков

Хуки можно комбинировать, создавая более высокоуровневые абстракции:

```jsx
function useAuth() {
  const [user, setUser] = useState(null);
  const { data, isLoading } = useApi("/api/me");

  useEffect(() => {
    if (data) setUser(data.user);
  }, [data]);

  const login = async (credentials) => {
    const res = await fetch("/api/login", {
      method: "POST",
      body: JSON.stringify(credentials),
    });
    const data = await res.json();
    setUser(data.user);
  };

  const logout = () => {
    setUser(null);
    fetch("/api/logout", { method: "POST" });
  };

  return { user, isLoading, login, logout };
}
```

---

## Container / Presentational

Паттерн разделения компонентов на два типа:

- **Container (умный)** — отвечает за данные, логику, побочные эффекты
- **Presentational (глупый)** — отвечает только за отображение, получает всё через пропсы

```jsx
// Presentational — ничего не знает об источнике данных
function UserList({ users, isLoading, onUserClick }) {
  if (isLoading) return <Spinner />;
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id} onClick={() => onUserClick(user)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}

// Container — загружает данные и передаёт в презентационный
function UserListContainer() {
  const { data: users, isLoading } = useApi("/api/users");
  const navigate = useNavigate();

  return (
    <UserList
      users={users ?? []}
      isLoading={isLoading}
      onUserClick={(user) => navigate(`/users/${user.id}`)}
    />
  );
}
```

### Актуальность в 2026

С появлением серверных компонентов и хуков паттерн Container/Presentational **трансформировался**:

- Серверные компоненты по умолчанию являются «контейнерами» — они загружают данные
- Клиентские компоненты становятся «presentational» — они получают данные через пропсы
- Хуки заменили необходимость в отдельных контейнерных компонентах для большинства случаев

```jsx
// Современная интерпретация: Server Component = Container
async function UserListPage() {
  const users = await db.users.findMany();
  return <UserList users={users} />;
}

// Клиентский компонент = Presentational
"use client";
function UserList({ users }) {
  // только UI-логика
}
```

Паттерн по-прежнему полезен для понимания разделения ответственности, но явное создание пар «контейнер + презентация» теперь избыточно.

---

## State Reducer

Паттерн **State Reducer** позволяет потребителю компонента контролировать, как обновляется внутреннее состояние компонента, не меняя его поведение по умолчанию.

```jsx
function Toggle({ children, reducer: externalReducer }) {
  const [on, setOn] = useState(false);

  const defaultReducer = (state, action) => {
    switch (action.type) {
      case "toggle": return { on: !state.on };
      case "reset": return { on: false };
      default: return state;
    }
  };

  const reducer = externalReducer
    ? (state, action) => externalReducer(defaultReducer, state, action)
    : defaultReducer;

  const handleToggle = () => {
    setState((prev) => reducer(prev, { type: "toggle" }));
  };

  return children({ on, toggle: handleToggle });
}

// Использование: отключаем toggle при определённом условии
<Toggle
  reducer={(defaultReducer, state, action) => {
    if (state.on && action.type === "toggle") return state;
    return defaultReducer(state, action);
  }}
>
  {({ on, toggle }) => <button onClick={toggle}>{on ? "ON" : "OFF"}</button>}
</Toggle>
```

Этот паттерн часто встречается в Headless UI-библиотеках, где нужно дать пользователю контроль над поведением без переписывания всей логики.

---

## Control Props

Паттерн **Control Props** позволяет компоненту работать в двух режимах:
- **Uncontrolled** — компонент сам управляет своим состоянием
- **Controlled** — состояние управляется извне через пропсы

Это тот же паттерн, что используется в `<input value={val} onChange={...} />` vs `<input defaultValue="x" />`.

```jsx
function Toggle({ on: controlledOn, onChange, defaultOn = false }) {
  const [internalOn, setInternalOn] = useState(defaultOn);

  const isControlled = controlledOn !== undefined;
  const on = isControlled ? controlledOn : internalOn;

  const handleToggle = () => {
    const newValue = !on;
    if (!isControlled) {
      setInternalOn(newValue);
    }
    onChange?.(newValue);
  };

  return <button onClick={handleToggle}>{on ? "ON" : "OFF"}</button>;
}

// Uncontrolled
<Toggle defaultOn={false} onChange={(val) => console.log(val)} />

// Controlled
const [isOn, setIsOn] = useState(false);
<Toggle on={isOn} onChange={setIsOn} />
```

### Когда использовать

- Библиотечные компоненты (формы, модалки, аккордеоны)
- Когда нужно поддержать оба режима использования
- Для совместимости с управляемыми и неуправляемыми формами

---

## Props Collection (State Aggregator)

Паттерн **Props Collection** группирует связанные пропсы в объект, упрощая API:

```jsx
// ❌ Много отдельных пропсов
<DataTable
  sortField="name"
  sortOrder="asc"
  onSortFieldChange={setSortField}
  onSortOrderChange={setSortOrder}
  filterText=""
  onFilterTextChange={setFilterText}
/>

// ✅ Группировка в объекты
<DataTable
  sort={{ field: "name", order: "asc", onChange: setSort }}
  filter={{ text: "", onChange: setFilterText }}
/>
```

---

## Структура файлов: подходы

Организация файлов — одна из самых субъективных тем в React-разработке. Нет «правильного» подхода, но есть проверенные стратегии.

### 1. По типу (Type-based)

Группировка по техническому назначению файлов:

```
src/
├── components/
│   ├── Button.tsx
│   ├── Modal.tsx
│   └── Input.tsx
├── hooks/
│   ├── useAuth.ts
│   └── useFetch.ts
├── services/
│   └── api.ts
├── utils/
│   └── format.ts
├── pages/
│   ├── Home.tsx
│   └── Dashboard.tsx
└── types/
    └── index.ts
```

**Плюсы:** Простая навигация, понятна для маленьких проектов.
**Минусы:** При росте проекта папки `components/` раздуваются до сотен файлов. Связанные файлы разбросаны по разным директориям.

### 2. По фиче (Feature-based)

Группировка по бизнес-домену:

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignupForm.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── api/
│   │   │   └── authApi.ts
│   │   ├── types.ts
│   │   └── index.ts
│   ├── products/
│   │   ├── components/
│   │   │   ├── ProductCard.tsx
│   │   │   └── ProductList.tsx
│   │   ├── hooks/
│   │   │   └── useProducts.ts
│   │   ├── api/
│   │   │   └── productsApi.ts
│   │   └── index.ts
│   └── cart/
│       ├── components/
│       ├── hooks/
│       │   └── useCart.ts
│       ├── store/
│       │   └── cartStore.ts
│       └── index.ts
├── shared/
│   ├── components/
│   │   ├── Button.tsx
│   │   └── Modal.tsx
│   ├── hooks/
│   │   └── useDebounce.ts
│   ├── utils/
│   │   └── format.ts
│   └── types/
│       └── api.ts
└── app/
    ├── layout.tsx
    ├── providers.tsx
    └── router.tsx
```

**Плюсы:** Всё, что относится к фиче, в одном месте. Легко удалить фичу целиком. Масштабируется.
**Минусы:** Дублирование инфраструктуры (каждая фича имеет свою `components/`, `hooks/`).

### 3. По домену (Domain-driven / Screaming Architecture)

Структура «кричит» о бизнес-домене:

```
src/
├── domains/
│   ├── user/
│   │   ├── User.ts
│   │   ├── UserRepository.ts
│   │   └── UserService.ts
│   ├── order/
│   │   ├── Order.ts
│   │   ├── OrderRepository.ts
│   │   └── OrderService.ts
│   └── product/
├── infrastructure/
│   ├── http/
│   │   └── apiClient.ts
│   ├── storage/
│   │   └── localStorage.ts
│   └── auth/
│       └── authProvider.ts
├── application/
│   ├── useCases/
│   │   ├── loginUser.ts
│   │   └── createOrder.ts
│   └── queries/
│       └── getProducts.ts
└── presentation/
    ├── pages/
    ├── components/
    └── hooks/
```

**Плюсы:** Чёткое разделение ответственности. Бизнес-логика независима от фреймворка.
**Минусы:** Избыточная сложность для небольших проектов. Требует зрелой команды.

### 4. Feature-Sliced Design (FSD)

Популярная в СНГ-сообществе методология:

```
src/
├── app/              # инициализация приложения, провайдеры, роутинг
├── processes/        # сквозные бизнес-процессы (авторизация, оплата)
├── pages/            # страницы приложения
├── widgets/          # самостоятельные блоки UI (Sidebar, Header)
├── features/         # части функциональности (LikeButton, AddToCart)
├── entities/         # бизнес-сущности (User, Product, Order)
└── shared/           # переиспользуемый код (UI-kit, lib, API-клиент)
```

Каждый слой может импортировать только из слоёв ниже. Нарушение границ — ошибка архитектуры.

**Плюсы:** Строгие правила, предсказуемая структура, хорошо для больших команд.
**Минусы:** Много уровней абстракции, избыточно для проектов < 50 страниц.

---

## Структура файлов в Next.js App Router

Next.js App Router навязывает структуру через файловую систему:

```
src/
├── app/
│   ├── layout.tsx              # корневой layout
│   ├── page.tsx                # главная страница
│   ├── loading.tsx             # глобальный loading
│   ├── error.tsx               # глобальный error
│   ├── not-found.tsx           # 404
│   │
│   ├── (auth)/                 # route group — не влияет на URL
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── register/
│   │   │   └── page.tsx
│   │   └── layout.tsx          # layout для auth-группы
│   │
│   ├── (dashboard)/
│   │   ├── layout.tsx          # layout с сайдбаром
│   │   ├── overview/
│   │   │   └── page.tsx
│   │   ├── settings/
│   │   │   └── page.tsx
│   │   └── profile/
│   │       └── page.tsx
│   │
│   ├── api/                    # Route Handlers
│   │   ├── users/
│   │   │   └── route.ts
│   │   └── webhooks/
│   │       └── stripe/
│   │           └── route.ts
│   │
│   └── [...slug]/
│       └── page.tsx            # catch-all
│
├── components/
│   ├── ui/                     # переиспользуемые UI-компоненты
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   └── dialog.tsx
│   ├── features/               # компонентные части фич
│   │   ├── auth/
│   │   │   ├── login-form.tsx
│   │   │   └── signup-form.tsx
│   │   └── products/
│   │       ├── product-card.tsx
│   │       └── product-grid.tsx
│   └── layouts/                # layout-компоненты
│       ├── sidebar.tsx
│       └── header.tsx
│
├── lib/                        # утилиты и конфигурация
│   ├── api/                    # API-клиент, запросы
│   │   ├── client.ts
│   │   ├── users.ts
│   │   └── products.ts
│   ├── auth/
│   │   └── session.ts
│   ├── db/
│   │   └── index.ts
│   └── utils.ts
│
├── hooks/                      # переиспользуемые хуки
│   ├── use-debounce.ts
│   └── use-media-query.ts
│
├── stores/                     # глобальное состояние (Zustand и т.д.)
│   ├── auth-store.ts
│   └── cart-store.ts
│
├── types/                      # TypeScript-типы
│   ├── api.ts
│   ├── auth.ts
│   └── product.ts
│
└── config/                     # конфигурация
    ├── site.ts
    └── env.ts
```

### Рекомендации для Next.js

- **Route Groups** `(group)` — для логической группировки без влияния на URL
- **Parallel Routes** `@modal`, `@team` — для одновременного рендеринга нескольких страниц в одном layout
- **Intercepting Routes** `(.)route` — для модалок и overlay-навигации
- **Колocate Server Actions** с компонентами, которые их используют, или вынесите в `lib/actions/`

---

## Организация общего кода

### UI-кит (shared/components/ui)

Базовые компоненты без бизнес-логики:

```
components/ui/
├── button.tsx
├── input.tsx
├── dialog.tsx
├── dropdown-menu.tsx
├── toast.tsx
└── index.ts
```

Эти компоненты:
- Не знают о бизнес-домене
- Не делают API-запросов
- Не используют глобальное состояние
- Принимают всё через пропсы

### Хуки (shared/hooks)

Переиспользуемые хуки без привязки к фиче:

```
hooks/
├── use-debounce.ts
├── use-media-query.ts
├── use-click-outside.ts
├── use-local-storage.ts
└── use-intersection-observer.ts
```

### Утилиты (shared/lib)

Чистые функции и конфигурация:

```
lib/
├── utils.ts          # format, parse, validate
├── constants.ts      # enum, конфиг
├── api/
│   ├── client.ts     # настроенный fetch/axios
│   └── endpoints.ts  # URL-ы API
└── validators/
    ├── auth.ts       # Zod-схемы
    └── product.ts
```

---

## Лучшие практики

### 1. Баррел-файлы (index.ts) для публичных API

```tsx
// features/auth/index.ts
export { LoginForm } from "./components/login-form";
export { SignupForm } from "./components/signup-form";
export { useAuth } from "./hooks/use-auth";
export type { User, AuthState } from "./types";
```

> ⚠️ Баррел-файлы могут ухудшить tree-shaking в бандлерах. Используйте их для публичных API модулей, но не для `components/ui/` с десятками компонентов.

### 2. Colocation — держите связанное рядом

```
features/auth/
├── login-form.tsx
├── login-form.test.tsx
├── login-form.stories.tsx
├── use-login.ts
├── login.schema.ts
└── types.ts
```

Тесты, сторибуки, схемы валидации и типы живут рядом с компонентом, а не в отдельных `__tests__/`, `__stories__/` директориях.

### 3. Абсолютные импорты через алиасы

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/features/*": ["./src/features/*"],
      "@/lib/*": ["./src/lib/*"],
      "@/hooks/*": ["./src/hooks/*"]
    }
  }
}
```

```tsx
// ✅ Читаемый импорт
import { Button } from "@/components/ui/button";
import { useAuth } from "@/features/auth";

// ❌ Относительный импорт через 4 уровня
import { Button } from "../../../../components/ui/button";
```

### 4. Ограничение глубины вложенности

Максимум 3 уровня вложенности директорий. Если глубже — признак, что фича слишком большая и её нужно разделить.

### 5. Единая точка входа для API

```tsx
// lib/api/client.ts
const apiClient = {
  async get<T>(url: string): Promise<T> {
    const res = await fetch(url, { headers: await getAuthHeaders() });
    if (!res.ok) throw new ApiError(res);
    return res.json();
  },
  async post<T>(url: string, body: unknown): Promise<T> {
    const res = await fetch(url, {
      method: "POST",
      headers: { "Content-Type": "application/json", ...(await getAuthHeaders()) },
      body: JSON.stringify(body),
    });
    if (!res.ok) throw new ApiError(res);
    return res.json();
  },
};
```

Все запросы проходят через один клиент — легко добавить interceptors, логирование, обработку ошибок.

---

## Антипаттерны

### 1. God Component

Один компонент, который делает всё:

```tsx
// ❌ 500+ строк, вся бизнес-логика в одном месте
function Dashboard() {
  const [users, setUsers] = useState([]);
  const [products, setProducts] = useState([]);
  const [orders, setOrders] = useState([]);
  const [filters, setFilters] = useState({});
  const [sort, setSort] = useState({});
  const [page, setPage] = useState(1);
  const [modal, setModal] = useState(null);

  useEffect(() => { /* загрузка пользователей */ }, []);
  useEffect(() => { /* загрузка продуктов */ }, []);
  useEffect(() => { /* загрузка заказов */ }, []);

  const handleUserClick = () => { /* ... */ };
  const handleProductDelete = () => { /* ... */ };
  const handleOrderUpdate = () => { /* ... */ };
  const handleFilterChange = () => { /* ... */ };
  const handleSortChange = () => { /* ... */ };
  const handlePageChange = () => { /* ... */ };
  const handleModalOpen = () => { /* ... */ };
  const handleExport = () => { /* ... */ };

  return (
    <div>
      {/* 500 строк JSX */}
    </div>
  );
}
```

Разбивайте на компоненты, хуки и утилиты.

### 2. Случайная структура

```
src/
├── components/
│   ├── NewButton.tsx
│   ├── old_modal.tsx
│   ├── userCard.tsx
│   └── temp/
│       └── test.tsx
├── utils/
│   ├── helpers.ts
│   ├── helpers2.ts
│   └── old_helpers.ts
└── api/
    ├── api.ts
    └── newApi.ts
```

Нет конвенции именования, дублирование, временные файлы становятся постоянными.

### 3. Circular Dependencies

```tsx
// user.ts
import { Order } from "./order";
export type User = { orders: Order[] };

// order.ts
import { User } from "./user";
export type Order = { user: User };
```

Используйте интерфейсы, type-only imports (`import type`) или рефакторите в общий файл типов.

### 4. Избыточная абстракция

```tsx
// ❌ 5 файлов для одного компонента
// Button.types.ts
// Button.styles.ts
// Button.utils.ts
// Button.constants.ts
// Button.tsx

// ✅ Имеет смысл только если действительно 5+ вариантов и сложная логика
// Button.tsx — всё в одном файле
```

### 5. Import from implementation details

```tsx
// ❌ Импорт из внутреннего пути фичи
import { LoginForm } from "@/features/auth/components/login-form";

// ✅ Импорт из публичного API
import { LoginForm } from "@/features/auth";
```
