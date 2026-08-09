# TypeScript в React — полное руководство для Vue-разработчика

TypeScript и React — одна из самых распространённых связок в современной frontend-разработке. Если вы уже используете TypeScript во Vue, многие концепции покажутся знакомыми, но есть и специфичные для React паттерны: типизация JSX-пропсов, синтетических событий, хуков, дженериков в компонентах и `ref`. В этой статье мы разберём все ключевые аспекты TypeScript в React, проводя параллели с Vue и показывая, как перенести накопленные знания.

---

## Содержание

1. [Базовая типизация пропсов](#базовая-типизация-пропсов)
2. [Типизация событий](#типизация-событий)
3. [Типизация хуков](#типизация-хуков)
4. [Дженерики в компонентах](#дженерики-в-компонентах)
5. [Дочерние элементы: children и слоты](#дочерние-элементы-children-и-слоты)
6. [Типизация ref и доступ к DOM](#типизация-ref-и-доступ-к-dom)
7. [Типизация контекста](#типизация-контекста)
8. [Utility-типы для пропсов](#utility-типы-для-пропсов)
9. [Discriminated unions для пропсов и редукторов](#discriminated-unions-для-пропсов-и-редукторов)
10. [Типизация forwardRef](#типизация-forwardref)
11. [Типизация форм](#типизация-форм)
12. [Vue vs React: шпаргалка по типизации](#vue-vs-react-шпаргалка-по-типизации)
13. [Типичные ошибки](#типичные-ошибки)

---

## Базовая типизация пропсов

Во Vue вы типизируете пропсы через `defineProps<T>()` или `withDefaults`. В React пропсы — это аргумент функции-компонента, и типизируется он как обычный параметр.

> **Vue-аналог:** `defineProps<{ title: string; count?: number }>()` в `<script setup lang="ts">`. В React — `interface Props` или `type Props` у параметра функции. Концептуально то же самое, но синтаксис другой.

### Через interface

```tsx
interface ButtonProps {
  label: string;
  variant?: "primary" | "secondary" | "danger";
  disabled?: boolean;
  onClick: () => void;
}

function Button({ label, variant = "primary", disabled = false, onClick }: ButtonProps) {
  return (
    <button className={`btn-${variant}`} disabled={disabled} onClick={onClick}>
      {label}
    </button>
  );
}
```

### Через type

```tsx
type CardProps = {
  title: string;
  children: React.ReactNode;
};

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h3>{title}</h3>
      {children}
    </div>
  );
}
```

### Деструктуризация с дефолтными значениями

Дефолтные значения в деструктуризации автоматически сужают тип — TypeScript понимает, что `variant` внутри компонента всегда `string`, а не `string | undefined`:

```tsx
interface AlertProps {
  message: string;
  type?: "info" | "warning" | "error";
}

function Alert({ message, type = "info" }: AlertProps) {
  // type здесь: "info" | "warning" | "error" — без undefined
  return <div className={`alert-${type}`}>{message}</div>;
}
```

### Rest-пропсы

```tsx
interface InputProps {
  label: string;
  error?: string;
}

function Input({ label, error, ...rest }: InputProps & React.InputHTMLAttributes<HTMLInputElement>) {
  return (
    <div>
      <label>{label}</label>
      <input {...rest} />
      {error && <span className="error">{error}</span>}
    </div>
  );
}
```

`React.InputHTMLAttributes<HTMLInputElement>` — встроенный тип, содержащий все стандартные атрибуты `<input>`: `type`, `placeholder`, `onChange`, `value` и т.д. Оператор `&` (intersection) объединяет ваши кастомные пропсы со стандартными.

> **Vue-аналог:** во Vue атрибуты автоматически пробрасываются на корневой элемент (fallthrough attributes). В React это делается явно через `...rest` + `React.InputHTMLAttributes`.

---

## Типизация событий

React использует **синтетические события** — обёртку над нативными DOM-событиями с кроссбраузерным API. Каждый тип события имеет свой дженерик.

> **Vue-аналог:** во Vue события типизируются через `@change="(e: Event) => ..."` или `defineEmits<{ (e: 'change', value: string): void }>()`. В React — через `React.ChangeEvent<HTMLInputElement>` и подобные типы.

### Основные типы событий

| Событие | Тип React | Нативный аналог |
|---|---|---|
| `onClick` | `React.MouseEvent<HTMLButtonElement>` | `MouseEvent` |
| `onChange` | `React.ChangeEvent<HTMLInputElement>` | `Event` |
| `onSubmit` | `React.FormEvent<HTMLFormElement>` | `SubmitEvent` |
| `onKeyDown` | `React.KeyboardEvent<HTMLInputElement>` | `KeyboardEvent` |
| `onFocus` | `React.FocusEvent<HTMLInputElement>` | `FocusEvent` |
| `onDrag` | `React.DragEvent<HTMLDivElement>` | `DragEvent` |
| `onWheel` | `React.WheelEvent<HTMLDivElement>` | `WheelEvent` |
| `onTouchStart` | `React.TouchEvent<HTMLDivElement>` | `TouchEvent` |
| `onPointerDown` | `React.PointerEvent<HTMLDivElement>` | `PointerEvent` |

### Примеры

```tsx
function SearchInput() {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value; // string
    console.log(value);
  };

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") {
      e.preventDefault();
      console.log("Submitted via Enter");
    }
  };

  return <input onChange={handleChange} onKeyDown={handleKeyDown} />;
}
```

```tsx
function FormExample() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    console.log(formData.get("email"));
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Кастомные обработчики в пропсах

```tsx
interface ModalProps {
  onClose: () => void;
  onConfirm: (data: FormData) => void;
  onSelect: (id: string, index: number) => void;
}

function Modal({ onClose, onConfirm, onSelect }: ModalProps) {
  return (
    <div>
      <button onClick={onClose}>Close</button>
      <button onClick={() => onConfirm(new FormData())}>Confirm</button>
      <button onClick={() => onSelect("item-1", 0)}>Select</button>
    </div>
  );
}
```

### Доступ к текущему элементу

```tsx
function AutoResizeTextarea() {
  const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    const textarea = e.currentTarget; // HTMLTextAreaElement
    textarea.style.height = "auto";
    textarea.style.height = `${textarea.scrollHeight}px`;
  };

  return <textarea onChange={handleChange} />;
}
```

Разница между `e.target` и `e.currentTarget`:
- `e.target` — элемент, **на котором** произошло событие (может быть дочерний).
- `e.currentTarget` — элемент, **на котором** висит обработчик.

---

## Типизация хуков

Хуки — основная абстракция React, и каждый из них имеет свои TypeScript-паттерны.

### useState

```tsx
// Примитив — тип выводится автоматически
const [count, setCount] = useState(0); // number

// Объект — указываем тип явно
const [user, setUser] = useState<User | null>(null);

// Массив с дженериком
const [items, setItems] = useState<string[]>([]);

// Union-тип
const [status, setStatus] = useState<"idle" | "loading" | "success" | "error">("idle");
```

> **Vue-аналог:** `const count = ref(0)` — тип выводится из начального значения. `const user = ref<User | null>(null)` — явный дженерик. То же самое, но без деструктуризации.

### useRef

```tsx
// Ref на DOM-элемент — дженерик + null
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  inputRef.current?.focus(); // optional chaining — current может быть null
}, []);

return <input ref={inputRef} />;
```

```tsx
// Ref для хранения значения (не DOM)
const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

const startTimer = () => {
  timerRef.current = setInterval(() => {
    console.log("tick");
  }, 1000);
};

const stopTimer = () => {
  if (timerRef.current) {
    clearInterval(timerRef.current);
    timerRef.current = null;
  }
};
```

> **Vue-аналог:** `const inputRef = ref<HTMLInputElement | null>(null)` для DOM. Для нереактивного хранения — обычная переменная `let timer: number | null = null` вне `setup`.

### useMemo и useCallback

```tsx
// useMemo — тип выводится из возвращаемого значения
const total = useMemo(() => items.reduce((sum, item) => sum + item.price, 0), [items]);
// total: number

// Явный тип, если нужно
const sorted = useMemo<string[]>(() => {
  return [...items].sort((a, b) => a.name.localeCompare(b.name));
}, [items]);
```

```tsx
// useCallback — типизируется как обычная функция
const handleClick = useCallback((id: string) => {
  console.log(id);
}, []);

// Если useCallback зависит от пропсов
const handleUpdate = useCallback(
  (newName: string) => {
    onUpdate(userId, newName);
  },
  [userId, onUpdate]
);
```

### useReducer

```tsx
// 1. Описываем состояния через discriminated union
type State =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: string[] }
  | { status: "error"; error: string };

// 2. Описываем действия через discriminated union
type Action =
  | { type: "FETCH_START" }
  | { type: "FETCH_SUCCESS"; data: string[] }
  | { type: "FETCH_ERROR"; error: string }
  | { type: "RESET" };

// 3. Редуктор — TypeScript проверяет exhaustiveness
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "FETCH_START":
      return { status: "loading" };
    case "FETCH_SUCCESS":
      return { status: "success", data: action.data };
    case "FETCH_ERROR":
      return { status: "error", error: action.error };
    case "RESET":
      return { status: "idle" };
  }
}

// 4. Использование в компоненте
function DataFetcher() {
  const [state, dispatch] = useReducer(reducer, { status: "idle" });

  useEffect(() => {
    dispatch({ type: "FETCH_START" });
    fetchData()
      .then(data => dispatch({ type: "FETCH_SUCCESS", data }))
      .catch(error => dispatch({ type: "FETCH_ERROR", error: String(error) }));
  }, []);

  switch (state.status) {
    case "loading":
      return <Spinner />;
    case "success":
      return <List items={state.data} />;
    case "error":
      return <ErrorMessage message={state.error} />;
    case "idle":
      return null;
  }
}
```

Discriminated union по полю `type` — один из самых мощных паттернов TypeScript в React. Компилятор гарантирует, что вы обработали все возможные действия, и сужает тип `action` внутри каждой ветки `switch`.

---

## Дженерики в компонентах

Дженерики позволяют создавать переиспользуемые компоненты, работающие с любыми типами данных.

> **Vue-аналог:** `<script setup lang="ts" generic="T extends Item">` во Vue 3.3+. В React — `function List<T extends Item>({ items }: ListProps<T>)`. Концептуально идентично.

### Базовый пример

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Использование — тип выводится автоматически
<List
  items={[{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }]}
  renderItem={user => <span>{user.name}</span>}
  keyExtractor={user => String(user.id)}
/>
```

### Ограничение дженерика

```tsx
interface SelectProps<T extends { id: string; label: string }> {
  options: T[];
  value: T;
  onChange: (value: T) => void;
}

function Select<T extends { id: string; label: string }>({
  options,
  value,
  onChange,
}: SelectProps<T>) {
  return (
    <select
      value={value.id}
      onChange={e => {
        const selected = options.find(opt => opt.id === e.target.value);
        if (selected) onChange(selected);
      }}
    >
      {options.map(opt => (
        <option key={opt.id} value={opt.id}>
          {opt.label}
        </option>
      ))}
    </select>
  );
}
```

### Дженерики с forwardRef

```tsx
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(function Input(
  { label, ...rest },
  ref
) {
  return (
    <label>
      {label}
      <input ref={ref} {...rest} />
    </label>
  );
});
```

---

## Дочерние элементы: children и слоты

Во Vue для дочернего контента используется `<slot />`. В React — пропс `children`.

> **Vue-аналог:** `<slot />` для дефолтного слота, `<slot name="header" />` для именованного. В React `children` — это дефолтный «слот», а именованные слоты реализуются через отдельные пропсы.

### React.ReactNode

`React.ReactNode` — тип всего, что может быть отрендерено в JSX:

```tsx
interface LayoutProps {
  children: React.ReactNode;
}

function Layout({ children }: LayoutProps) {
  return <div className="layout">{children}</div>;
}
```

`React.ReactNode` включает: `string`, `number`, `boolean`, `null`, `undefined`, `ReactElement`, `ReactFragment`, `ReactPortal`.

### Именованные «слоты» через пропсы

```tsx
interface PageProps {
  header: React.ReactNode;
  sidebar: React.ReactNode;
  children: React.ReactNode;
}

function Page({ header, sidebar, children }: PageProps) {
  return (
    <>
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
    </>
  );
}

// Использование
<Page
  header={<h1>Dashboard</h1>}
  sidebar={<nav>...</nav>}
>
  <p>Main content</p>
</Page>
```

### React.ReactElement vs React.ReactNode

```tsx
interface WrapperProps {
  // ReactElement — только JSX-элемент (<Component />)
  // Не принимает string, number, fragment
  icon: React.ReactElement;

  // ReactNode — что угодно: string, number, JSX, null, fragment
  label: React.ReactNode;
}
```

На практике `React.ReactNode` используется в 95% случаев. `React.ReactElement` нужен редко — когда вы хотите ограничить тип именно JSX-элементом.

---

## Типизация ref и доступ к DOM

### Ref на DOM-элемент

```tsx
function VideoPlayer() {
  const videoRef = useRef<HTMLVideoElement>(null);

  const play = () => {
    videoRef.current?.play();
  };

  return (
    <>
      <video ref={videoRef} src="/video.mp4" />
      <button onClick={play}>Play</button>
    </>
  );
}
```

### Ref на кастомный компонент

С React 19 ref передаётся как обычный пропс — `forwardRef` больше не обязателен:

```tsx
interface CustomInputProps {
  label: string;
  ref?: React.Ref<HTMLInputElement>;
}

function CustomInput({ label, ref }: CustomInputProps) {
  return (
    <label>
      {label}
      <input ref={ref} />
    </label>
  );
}

// Использование
function Parent() {
  const inputRef = useRef<HTMLInputElement>(null);

  return <CustomInput ref={inputRef} label="Name" />;
}
```

> **Vue-аналог:** `const inputRef = ref<HTMLInputElement | null>(null)` + `<input ref="inputRef" />` в шаблоне. В React — `useRef<HTMLInputElement>(null)` + `ref={inputRef}` в JSX.

### Ref для хранения императивного значения

```tsx
function AnimationController() {
  const frameRef = useRef<number>(0);

  useEffect(() => {
    const animate = () => {
      frameRef.current = requestAnimationFrame(animate);
    };
    frameRef.current = requestAnimationFrame(animate);

    return () => cancelAnimationFrame(frameRef.current);
  }, []);

  return <div className="animated" />;
}
```

---

## Типизация контекста

> **Vue-аналог:** `provide`/`inject` с `InjectionKey<T>`. В React — `createContext<T>` + `useContext`.

### Базовый пример

```tsx
interface ThemeContextValue {
  theme: "light" | "dark";
  toggle: () => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  const toggle = () => setTheme(t => (t === "light" ? "dark" : "light"));

  return (
    <ThemeContext value={{ theme, toggle }}>
      {children}
    </ThemeContext>
  );
}

function ThemedButton() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("ThemedButton must be used within ThemeProvider");

  return (
    <button className={`btn-${ctx.theme}`} onClick={ctx.toggle}>
      Current: {ctx.theme}
    </button>
  );
}
```

### Кастомный хук для безопасного доступа

Чтобы избежать проверки `if (!ctx)` в каждом компоненте, вынесите её в хук:

```tsx
const ThemeContext = createContext<ThemeContextValue | null>(null);

function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}

// Использование — чисто и безопасно
function ThemedButton() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>{theme}</button>;
}
```

---

## Utility-типы для пропсов

TypeScript предоставляет набор utility-типов, которые особенно полезны при работе с пропсами компонентов.

### Partial, Required, Pick, Omit

```tsx
interface UserCardProps {
  name: string;
  email: string;
  avatar: string;
  role: "admin" | "user" | "guest";
}

// Все поля необязательны — для формы редактирования
type EditableUserCard = Partial<UserCardProps>;

// Только name и email — для компактного отображения
type UserPreview = Pick<UserCardProps, "name" | "email">;

// Всё, кроме avatar — для формы без загрузки изображения
type UserWithoutAvatar = Omit<UserCardProps, "avatar">;

// Все поля обязательны — для полного объекта
type FullUserCard = Required<UserCardProps>;
```

### React.ComponentProps и варианты

```tsx
// Все пропсы существующего компонента — для обёрток и HOC
type ButtonProps = React.ComponentProps<typeof Button>;

// Только HTML-атрибуты элемента
type DivProps = React.HTMLAttributes<HTMLDivElement>;
type InputProps = React.InputHTMLAttributes<HTMLInputElement>;
type AnchorProps = React.AnchorHTMLAttributes<HTMLAnchorElement>;

// Только пропсы без children
type ModalProps = React.ComponentPropsWithoutRef<typeof Modal>;

// Пропсы с ref
type InputPropsWithRef = React.ComponentPropsWithRef<"input">;
```

### Практический пример: обёртка над HTML-элементом

```tsx
interface ContainerProps extends React.HTMLAttributes<HTMLDivElement> {
  fluid?: boolean;
}

function Container({ fluid = false, className, ...rest }: ContainerProps) {
  return (
    <div
      className={`${fluid ? "container-fluid" : "container"} ${className ?? ""}`}
      {...rest}
    />
  );
}

// Теперь Container принимает все стандартные атрибуты div:
// style, id, onClick, role, aria-*, data-* и т.д.
<Container fluid onClick={() => console.log("clicked")} role="main">
  Content
</Container>
```

---

## Discriminated unions для пропсов и редукторов

Discriminated union — тип с общим полем-«дискриминатором», по которому TypeScript сужает тип. Один из самых мощных паттернов.

### Пропсы с вариантами

```tsx
type ButtonProps =
  | { variant: "link"; href: string }
  | { variant: "button"; onClick: () => void };

function Button(props: ButtonProps) {
  if (props.variant === "link") {
    // TypeScript знает: props.href существует, props.onClick — нет
    return <a href={props.href}>Link</a>;
  }
  // TypeScript знает: props.onClick существует, props.href — нет
  return <button onClick={props.onClick}>Button</button>;
}

// ✅ Корректно
<Button variant="link" href="/about" />
<Button variant="button" onClick={() => {}} />

// ❌ Ошибка компиляции
<Button variant="link" onClick={() => {}} /> // onClick не существует для link
<Button variant="button" href="/about" />    // href не существует для button
```

### Exhaustiveness checking

```tsx
type Status = "idle" | "loading" | "success" | "error";

function getStatusLabel(status: Status): string {
  switch (status) {
    case "idle":
      return "Ready";
    case "loading":
      return "Loading...";
    case "success":
      return "Done";
    case "error":
      return "Failed";
    default: {
      // Если добавить новый вариант в Status — TypeScript выдаст ошибку здесь
      const _exhaustive: never = status;
      return _exhaustive;
    }
  }
}
```

Паттерн `never` в `default` гарантирует, что при добавлении нового значения в union-тип компилятор напомнит обработать его.

---

## Типизация forwardRef

С React 19 `forwardRef` стал необязательным — ref передаётся как обычный пропс. Но в legacy-коде и библиотеках он всё ещё встречается.

### Синтаксис

```tsx
interface InputProps {
  label: string;
  error?: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(function Input(
  { label, error, ...rest },
  ref
) {
  return (
    <div>
      <label>{label}</label>
      <input ref={ref} {...rest} />
      {error && <span>{error}</span>}
    </div>
  );
});
```

Первый дженерик — тип ref (DOM-элемент), второй — тип пропсов.

### forwardRef + дженерики компонента

```tsx
interface SelectProps<T> {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (item: T) => string;
}

function SelectInner<T>(
  { options, value, onChange, getLabel }: SelectProps<T>,
  ref: React.ForwardedRef<HTMLSelectElement>
) {
  return (
    <select
      ref={ref}
      value={value ? getLabel(value) : ""}
      onChange={e => {
        const selected = options.find(opt => getLabel(opt) === e.target.value);
        if (selected) onChange(selected);
      }}
    >
      {options.map(opt => (
        <option key={getLabel(opt)} value={getLabel(opt)}>
          {getLabel(opt)}
        </option>
      ))}
    </select>
  );
}

const Select = forwardRef(SelectInner) as <T>(
  props: SelectProps<T> & { ref?: React.Ref<HTMLSelectElement> }
) => React.ReactElement;
```

---

## Типизация форм

### Базовая типизация FormData

```tsx
function LoginForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);

    const email = formData.get("email"); // FormDataEntryValue | null
    const password = formData.get("password");

    if (typeof email === "string" && typeof password === "string") {
      login(email, password);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Типизация состояния формы

```tsx
interface FormState {
  email: string;
  password: string;
  errors: Partial<Record<"email" | "password", string>>;
  isSubmitting: boolean;
}

function LoginForm() {
  const [form, setForm] = useState<FormState>({
    email: "",
    password: "",
    errors: {},
    isSubmitting: false,
  });

  const updateField = (field: "email" | "password", value: string) => {
    setForm(prev => ({
      ...prev,
      [field]: value,
      errors: { ...prev.errors, [field]: undefined },
    }));
  };

  return (
    <form>
      <input
        value={form.email}
        onChange={e => updateField("email", e.target.value)}
      />
      {form.errors.email && <span>{form.errors.email}</span>}

      <input
        value={form.password}
        onChange={e => updateField("password", e.target.value)}
      />
      {form.errors.password && <span>{form.errors.password}</span>}
    </form>
  );
}
```

`Partial<Record<"email" | "password", string>>` — элегантный способ типизировать объект ошибок: каждое поле либо `string`, либо `undefined`.

---

## Vue vs React: шпаргалка по типизации

| Концепция | Vue | React |
|---|---|---|
| Пропсы компонента | `defineProps<{ title: string }>()` | `interface Props { title: string }` + параметр функции |
| Дефолтные пропсы | `withDefaults(defineProps(), { title: "Default" })` | Деструктуризация: `{ title = "Default" }: Props` |
| События (emit) | `defineEmits<{ (e: "change", v: string): void }>()` | Пропс-функция: `onChange: (value: string) => void` |
| Слоты | `<slot />`, `<slot name="header" />` | `children`, именованные пропсы: `header: ReactNode` |
| Ref на DOM | `const el = ref<HTMLInputElement \| null>(null)` | `const el = useRef<HTMLInputElement>(null)` |
| Provide/Inject | `provide<T>(key, value)` / `inject<T>(key)` | `createContext<T>()` / `useContext()` |
| Дженерики | `<script setup generic="T">` | `function Comp<T>(props: Props<T>)` |
| События | `@click="(e: Event) => ..."` | `onClick={(e: React.MouseEvent) => ...}` |
| Формы | `v-model` с типизацией | `value` + `onChange` с `React.ChangeEvent` |
| Ref-передача | `ref="childRef"` в шаблоне | `ref={childRef}` в JSX |

---

## Типичные ошибки

### 1. Использование `any` для событий

```tsx
// ❌ Теряется типобезопасность
const handleChange = (e: any) => { ... };

// ✅ Конкретный тип события
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { ... };
```

### 2. Отсутствие null-check для ref.current

```tsx
// ❌ Runtime-ошибка: current может быть null
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current.focus();

// ✅ Optional chaining
inputRef.current?.focus();
```

### 3. Inline-объекты в типе пропсов

```tsx
// ❌ Сложно переиспользовать, засоряет сигнатуру
function Card(props: { title: string; style: { color: string; fontSize: number } }) {}

// ✅ Вынести в interface
interface CardProps {
  title: string;
  style: React.CSSProperties;
}
```

### 4. Неправильный тип для children

```tsx
// ❌ Слишком узкий — не принимает string, number, fragment
interface ModalProps {
  children: React.ReactElement;
}

// ✅ Правильно
interface ModalProps {
  children: React.ReactNode;
}
```

### 5. Забытый дженерик у useState

```tsx
// ❌ Тип выводится как null — setUser не принимает User
const [user, setUser] = useState(null);
setUser({ name: "Alice" }); // Ошибка!

// ✅ Явный дженерик
const [user, setUser] = useState<User | null>(null);
setUser({ name: "Alice" }); // OK
```

### 6. Intersection вместо union для взаимоисключающих пропсов

```tsx
// ❌ Позволяет одновременно href и onClick — логическая ошибка
type ButtonProps = { href: string } & { onClick: () => void };

// ✅ Discriminated union — только один вариант за раз
type ButtonProps =
  | { href: string; onClick?: never }
  | { href?: never; onClick: () => void };
```

### 7. Игнорирование React.CSSProperties

```tsx
// ❌ Нет автодополнения, нет проверки свойств
const style = { colour: "red" }; // опечатка не ловится

// ✅ Типизированный объект
const style: React.CSSProperties = { color: "red" }; // опечатка — ошибка компиляции
```
