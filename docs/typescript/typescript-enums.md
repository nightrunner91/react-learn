# TypeScript Enums — полное руководство для React и Vue

Перечисления (Enums) — это способ определить именованный набор констант в TypeScript. Они позволяют создавать типы с ограниченным набором значений, делая код более читаемым и типобезопасным. В React enums часто используются для статусов, типов действий и конфигураций. Во Vue — для тех же целей, плюс для типизации пропсов и emit-событий. В этой статье разберём все виды enum, их особенности, альтернативы и лучшие практики.

---

## Содержание

1. [Что такое Enum и зачем он нужен](#что-такое-enum-и-зачем-он-нужен)
2. [Numeric Enums](#numeric-enums)
3. [String Enums](#string-enums)
4. [Const Enums](#const-enums)
5. [Reverse Mapping](#reverse-mapping)
6. [Enum как тип](#enum-как-тип)
7. [Enum с методами и вычислениями](#enum-с-методами-и-вычислениями)
8. [Enum в React](#enum-в-react)
9. [Enum во Vue](#enum-во-vue)
10. [Enum vs Union Types](#enum-vs-union-types)
11. [Enum в runtime](#enum-в-runtime)
12. [Типичные ошибки](#типичные-ошибки)
13. [Лучшие практики](#лучшие-практики)

---

## Что такое Enum и зачем он нужен

Enum определяет набор именованных констант. Без enum вам пришлось бы использовать магические числа или строки:

```typescript
// Плохо — магические числа
function setStatus(status: number) {
  if (status === 0) { /* idle */ }
  if (status === 1) { /* loading */ }
  if (status === 2) { /* success */ }
  if (status === 3) { /* error */ }
}

// Плохо — строковые литералы
function setStatus(status: string) {
  if (status === "idle") { /* ... */ }
  if (status === "loading") { /* ... */ }
}

// Хорошо — enum
enum Status {
  Idle,
  Loading,
  Success,
  Error
}

function setStatus(status: Status) {
  if (status === Status.Idle) { /* ... */ }
  if (status === Status.Loading) { /* ... */ }
}
```

> **Аналогия:** Enum — как меню в ресторане. Вы не можете заказать «что-то между пиццей и пастой» — только конкретные позиции из меню. Так же enum ограничивает значения только определёнными вариантами.

---

## Numeric Enums

Числовые enum — самый простой вид. Значения автоматически нумеруются начиная с 0.

### Базовый numeric enum

```typescript
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right  // 3
}

const dir: Direction = Direction.Up;
console.log(dir); // 0
console.log(Direction[dir]); // "Up"
```

### Явное задание значений

```typescript
enum HttpStatus {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  NotFound = 404,
  InternalError = 500
}

const status: HttpStatus = HttpStatus.NotFound;
console.log(status); // 404
```

### Автоинкремент с явного значения

```typescript
enum Priority {
  Low = 1,
  Medium,  // 2
  High,    // 3
  Critical // 4
}
```

### Вычисляемые значения

```typescript
enum FileAccess {
  None = 0,
  Read = 1 << 0,   // 1
  Write = 1 << 1,  // 2
  Execute = 1 << 2 // 4
}

const access = FileAccess.Read | FileAccess.Write; // 3
```

---

## String Enums

Строковые enum хранят строки вместо чисел. Они более читаемы при отладке.

### Базовый string enum

```typescript
enum Status {
  Idle = "IDLE",
  Loading = "LOADING",
  Success = "SUCCESS",
  Error = "ERROR"
}

const status: Status = Status.Loading;
console.log(status); // "LOADING"
```

### Почему string enum лучше для отладки

```typescript
enum NumericStatus {
  Idle,
  Loading,
  Success,
  Error
}

enum StringStatus {
  Idle = "idle",
  Loading = "loading",
  Success = "success",
  Error = "error"
}

// В консоли:
console.log(NumericStatus.Loading); // 1 — непонятно
console.log(StringStatus.Loading);  // "loading" — понятно
```

### String enum с одинаковыми значениями

```typescript
enum Color {
  Red = "RED",
  Crimson = "RED",  // Алиас
  Blue = "BLUE"
}

const c = Color.Crimson; // "RED"
```

---

## Const Enums

`const enum` — оптимизированная версия, которая полностью удаляется при компиляции. Вместо обращения к enum подставляются конкретные значения.

### Обычный enum vs const enum

```typescript
// Обычный enum — генерирует объект в JS
enum Direction {
  Up,
  Down,
  Left,
  Right
}

const dir = Direction.Up;
// JS: const dir = Direction.Up; (обращение к объекту)

// Const enum — значения подставляются напрямую
const enum ConstDirection {
  Up,
  Down,
  Left,
  Right
}

const dir2 = ConstDirection.Up;
// JS: const dir2 = 0; (значение подставлено)
```

### Сгенерированный код

```typescript
// TypeScript
const enum Status {
  Idle = "idle",
  Loading = "loading"
}

const s = Status.Idle;

// Компилируется в JavaScript:
const s = "idle";
// Status полностью удалён из кода
```

### Ограничения const enum

```typescript
// Error — нельзя использовать вычисляемые значения
const enum Math {
  Pi = 3.14,
  TwoPi = Math.Pi * 2 // Error!
}

// Error — нельзя использовать в computed properties
const enum Keys {
  A = "a",
  B = "b"
}
const obj = { [Keys.A]: 1 }; // Error с isolatedModules
```

> **Важно:** `const enum` не работает с `isolatedModules: true` (стандартная настройка в Vite, Next.js). Используйте обычные enum или union types.

---

## Reverse Mapping

Numeric enums поддерживают обратное маппинг — по значению можно получить имя:

```typescript
enum Status {
  Idle,     // 0
  Loading,  // 1
  Success,  // 2
  Error     // 3
}

// Прямое обращение
const code = Status.Loading; // 1

// Обратное обращение (reverse mapping)
const name = Status[1]; // "Loading"
```

### Как это работает под капотом

```typescript
// TypeScript
enum Status { Idle, Loading }

// Скомпилированный JavaScript
var Status;
(function (Status) {
  Status[Status["Idle"] = 0] = "Idle";
  Status[Status["Loading"] = 1] = "Loading";
})(Status || (Status = {}));

// Результат:
// Status[0] === "Idle"
// Status["Idle"] === 0
```

### String enum не поддерживает reverse mapping

```typescript
enum Color {
  Red = "RED",
  Blue = "BLUE"
}

console.log(Color["Red"]); // "RED"
console.log(Color["RED"]); // undefined — нет reverse mapping!
```

---

## Enum как тип

Enum можно использовать как тип для переменных, параметров и возвращаемых значений.

### Enum как тип параметра

```typescript
enum Role {
  Admin = "ADMIN",
  User = "USER",
  Guest = "GUEST"
}

function hasPermission(role: Role, permission: string): boolean {
  if (role === Role.Admin) return true;
  if (role === Role.User && permission === "read") return true;
  return false;
}

hasPermission(Role.Admin, "delete"); // true
hasPermission(Role.Guest, "read");   // false
```

### Enum в union types

```typescript
enum Status {
  Idle = "idle",
  Loading = "loading",
  Success = "success",
  Error = "error"
}

type State =
  | { status: Status.Idle }
  | { status: Status.Loading }
  | { status: Status.Success; data: any }
  | { status: Status.Error; message: string };

function handleState(state: State) {
  switch (state.status) {
    case Status.Idle:
      console.log("Ready to start");
      break;
    case Status.Loading:
      console.log("Loading...");
      break;
    case Status.Success:
      console.log("Data:", state.data);
      break;
    case Status.Error:
      console.log("Error:", state.message);
      break;
  }
}
```

### Enum как тип возвращаемого значения

```typescript
enum Result {
  Success,
  Failure
}

function validate(input: string): Result {
  if (input.length > 0) return Result.Success;
  return Result.Failure;
}

const result = validate("hello");
if (result === Result.Success) {
  console.log("Valid!");
}
```

---

## Enum с методами и вычислениями

Enum нельзя расширить методами напрямую, но можно создать вспомогательные функции:

### Вспомогательные функции

```typescript
enum Status {
  Idle = "idle",
  Loading = "loading",
  Success = "success",
  Error = "error"
}

function isTerminal(status: Status): boolean {
  return status === Status.Success || status === Status.Error;
}

function getLabel(status: Status): string {
  const labels: Record<Status, string> = {
    [Status.Idle]: "Ожидание",
    [Status.Loading]: "Загрузка",
    [Status.Success]: "Успех",
    [Status.Error]: "Ошибка"
  };
  return labels[status];
}

const status = Status.Loading;
console.log(isTerminal(status)); // false
console.log(getLabel(status));   // "Загрузка"
```

### Enum + Map для расширенных данных

```typescript
enum Priority {
  Low = "low",
  Medium = "medium",
  High = "high",
  Critical = "critical"
}

interface PriorityConfig {
  label: string;
  color: string;
  order: number;
}

const priorityConfig: Record<Priority, PriorityConfig> = {
  [Priority.Low]: { label: "Низкий", color: "green", order: 0 },
  [Priority.Medium]: { label: "Средний", color: "yellow", order: 1 },
  [Priority.High]: { label: "Высокий", color: "orange", order: 2 },
  [Priority.Critical]: { label: "Критический", color: "red", order: 3 }
};

function getPriorityConfig(p: Priority): PriorityConfig {
  return priorityConfig[p];
}
```

---

## Enum в React

### Enum для статусов

```typescript
enum FetchStatus {
  Idle = "idle",
  Loading = "loading",
  Success = "success",
  Error = "error"
}

interface FetchState<T> {
  status: FetchStatus;
  data: T | null;
  error: string | null;
}

function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    status: FetchStatus.Idle,
    data: null,
    error: null
  });

  useEffect(() => {
    setState(s => ({ ...s, status: FetchStatus.Loading }));

    fetch(url)
      .then(res => res.json())
      .then(data => setState({ status: FetchStatus.Success, data, error: null }))
      .catch(error => setState({ status: FetchStatus.Error, data: null, error: error.message }));
  }, [url]);

  return state;
}

// Использование в компоненте
function UserList() {
  const { status, data, error } = useFetch<User[]>("/api/users");

  switch (status) {
    case FetchStatus.Loading:
      return <Spinner />;
    case FetchStatus.Error:
      return <Error message={error!} />;
    case FetchStatus.Success:
      return <ul>{data!.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
    default:
      return null;
  }
}
```

### Enum для типов действий

```typescript
enum ActionType {
  Add = "ADD",
  Remove = "REMOVE",
  Update = "UPDATE"
}

interface Action {
  type: ActionType;
  payload?: any;
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case ActionType.Add:
      return { ...state, items: [...state.items, action.payload] };
    case ActionType.Remove:
      return { ...state, items: state.items.filter(i => i.id !== action.payload) };
    case ActionType.Update:
      return { ...state, items: state.items.map(i => i.id === action.payload.id ? action.payload : i) };
    default:
      return state;
  }
}
```

### Enum для пропсов

```typescript
enum ButtonVariant {
  Primary = "primary",
  Secondary = "secondary",
  Danger = "danger"
}

enum ButtonSize {
  Small = "sm",
  Medium = "md",
  Large = "lg"
}

interface ButtonProps {
  variant?: ButtonVariant;
  size?: ButtonSize;
  children: React.ReactNode;
}

function Button({ variant = ButtonVariant.Primary, size = ButtonSize.Medium, children }: ButtonProps) {
  return (
    <button className={`btn btn-${variant} btn-${size}`}>
      {children}
    </button>
  );
}

// Использование
<Button variant={ButtonVariant.Danger} size={ButtonSize.Large}>
  Удалить
</Button>
```

---

## Enum во Vue

### Enum для пропсов

```vue
<script setup lang="ts">
enum AlertType {
  Info = "info",
  Warning = "warning",
  Error = "error",
  Success = "success"
}

interface Props {
  type?: AlertType;
  message: string;
}

const props = withDefaults(defineProps<Props>(), {
  type: AlertType.Info
});
</script>

<template>
  <div :class="`alert alert-${type}`">
    {{ message }}
  </div>
</template>
```

### Enum для emit-событий

```vue
<script setup lang="ts">
enum EventType {
  Submit = "submit",
  Cancel = "cancel",
  Reset = "reset"
}

const emit = defineEmits<{
  (e: EventType.Submit, data: FormData): void;
  (e: EventType.Cancel): void;
  (e: EventType.Reset): void;
}>();

function handleSubmit() {
  emit(EventType.Submit, new FormData());
}
</script>
```

### Enum в composables

```typescript
enum StorageType {
  Local = "localStorage",
  Session = "sessionStorage"
}

function useStorage<T>(key: string, initialValue: T, type: StorageType = StorageType.Local) {
  const storage = type === StorageType.Local ? localStorage : sessionStorage;

  const value = ref<T>(initialValue) as Ref<T>;

  try {
    const item = storage.getItem(key);
    if (item) {
      value.value = JSON.parse(item) as T;
    }
  } catch {
    value.value = initialValue;
  }

  function setValue(newValue: T) {
    value.value = newValue;
    storage.setItem(key, JSON.stringify(newValue));
  }

  return { value, setValue };
}

// Использование
const { value: theme, setValue: setTheme } = useStorage(
  "theme",
  "light",
  StorageType.Local
);
```

---

## Enum vs Union Types

TypeScript предлагает альтернативу enum — union types. Сравним:

### Enum подход

```typescript
enum Status {
  Idle = "idle",
  Loading = "loading",
  Success = "success",
  Error = "error"
}

function handle(status: Status) {
  // ...
}
```

### Union type подход

```typescript
type Status = "idle" | "loading" | "success" | "error";

function handle(status: Status) {
  // ...
}
```

### Сравнение

| Критерий | Enum | Union Type |
|---|---|---|
| Runtime-представление | Объект в JS | Удаляется при компиляции |
| Reverse mapping | Да (numeric) | Нет |
| Размер бандла | Больше | Меньше |
| Итерация по значениям | `Object.values(Status)` | Нет прямого способа |
| Автодополнение | Да | Да |
| Совместимость с const | `const enum` | По умолчанию const |

### Когда использовать enum

- Нужна итерация по всем значениям
- Нужны числовые значения с reverse mapping
- Нужна группа связанных констант с общим пространством имён

### Когда использовать union type

- Простой набор строковых значений
- Важно минимизировать размер бандла
- Работаете с `isolatedModules: true`
- Значения не связаны логически

---

## Enum в runtime

Enum — это реальный объект в runtime (кроме const enum):

```typescript
enum Status {
  Idle = "idle",
  Loading = "loading"
}

// В runtime это:
// { Idle: "idle", Loading: "loading" }

// Можно итерировать
Object.values(Status); // ["idle", "loading"]
Object.keys(Status);   // ["Idle", "Loading"]
Object.entries(Status); // [["Idle", "idle"], ["Loading", "loading"]]
```

### Проверка принадлежности к enum

```typescript
enum Color {
  Red = "red",
  Blue = "blue",
  Green = "green"
}

function isColor(value: string): value is Color {
  return Object.values(Color).includes(value as Color);
}

const input = "red";
if (isColor(input)) {
  console.log(input as Color); // Color.Red
}
```

### Enum из API-ответа

```typescript
enum ApiStatus {
  Active = "active",
  Inactive = "inactive",
  Pending = "pending"
}

interface User {
  id: number;
  status: ApiStatus;
}

async function fetchUsers(): Promise<User[]> {
  const res = await fetch("/api/users");
  const data = await res.json();
  return data.map((u: any) => ({
    ...u,
    status: u.status as ApiStatus
  }));
}
```

---

## Типичные ошибки

### 1. Смешивание string и numeric в одном enum

```typescript
// Error — нельзя смешивать
enum Mixed {
  A = 1,
  B = "hello" // Error!
}

// OK — только один тип
enum Numeric { A = 1, B = 2 }
enum String { A = "a", B = "b" }
```

### 2. Использование enum как значения и типа одновременно

```typescript
enum Status {
  Idle = "idle",
  Loading = "loading"
}

// Error — Status уже определён
const Status = { Idle: "idle" }; // Error: Duplicate identifier
```

### 3. Отсутствие exhaustiveness check

```typescript
enum Status {
  Idle,
  Loading,
  Success,
  Error
}

function handle(status: Status): string {
  switch (status) {
    case Status.Idle:
      return "idle";
    case Status.Loading:
      return "loading";
    case Status.Success:
      return "success";
    // Забыли Error — компилятор не предупредит!
  }
}

// Хорошо — exhaustiveness check
function handle(status: Status): string {
  switch (status) {
    case Status.Idle:
      return "idle";
    case Status.Loading:
      return "loading";
    case Status.Success:
      return "success";
    case Status.Error:
      return "error";
    default: {
      const exhaustive: never = status;
      return exhaustive;
    }
  }
}
```

### 4. Enum с дублирующимися значениями

```typescript
enum Direction {
  Up = 1,
  North = 1, // Алиас — допустимо, но может запутать
  Down = 2,
  South = 2
}

console.log(Direction.North); // 1
console.log(Direction[1]);    // "Up" — вернёт первое имя!
```

### 5. Использование const enum с isolatedModules

```json
// tsconfig.json
{
  "compilerOptions": {
    "isolatedModules": true // Включено в Vite, Next.js
  }
}
```

```typescript
// Error — const enum не работает с isolatedModules
const enum Status {
  Idle = "idle"
}

// OK — используйте обычный enum
enum Status {
  Idle = "idle"
}

// Или union type
type Status = "idle" | "loading";
```

---

## Лучшие практики

### 1. Используйте string enum для читаемости

```typescript
// Хорошо — видно значение при отладке
enum Status {
  Idle = "idle",
  Loading = "loading",
  Success = "success",
  Error = "error"
}
```

### 2. Используйте enum для связанных констант

```typescript
// Хорошо — логически связанные значения
enum HttpStatus {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  NotFound = 404
}
```

### 3. Используйте union type для простых случаев

```typescript
// Лучше enum — проще, легче, без runtime-объекта
type Alignment = "left" | "center" | "right";
type Size = "sm" | "md" | "lg";
```

### 4. Группируйте enum по домену

```typescript
// Хорошо — каждый enum в своём файле или секции
// enums/status.ts
export enum FetchStatus {
  Idle = "idle",
  Loading = "loading",
  Success = "success",
  Error = "error"
}

// enums/role.ts
export enum UserRole {
  Admin = "admin",
  User = "user",
  Guest = "guest"
}
```

### 5. Используйте namespace для связанных enum

```typescript
namespace Order {
  export enum Status {
    Pending = "pending",
    Processing = "processing",
    Shipped = "shipped",
    Delivered = "delivered"
  }

  export enum PaymentMethod {
    CreditCard = "credit_card",
    PayPal = "paypal",
    BankTransfer = "bank_transfer"
  }
}

const status: Order.Status = Order.Status.Pending;
const payment: Order.PaymentMethod = Order.PaymentMethod.CreditCard;
```

---

## Заключение

Enum — полезный инструмент TypeScript для создания именованных наборов констант.

**Используйте enum, когда:**
- Нужна группа логически связанных констант
- Нужна итерация по значениям
- Нужны числовые значения с reverse mapping
- Нужны вычисляемые значения (битовые флаги)

**Используйте union type, когда:**
- Простой набор строковых значений
- Важен минимальный размер бандла
- Работаете с `isolatedModules: true`

**Избегайте:**
- `const enum` в проектах с `isolatedModules`
- Смешивания типов в одном enum
- Магических чисел/строк без enum или union
