# TypeScript Generics — полное руководство для React и Vue

Дженерики — одна из самых мощных и одновременно самых сложных концепций TypeScript. Они позволяют писать код, который работает с любыми типами, сохраняя при этом полную типобезопасность. В React дженерики используются повсеместно: в хуках, компонентах, utility-типах. Во Vue — в composables, пропсах, хелперах. В этой статье мы разберём дженерики от базовых концепций до продвинутых паттернов, применяемых в реальных проектах.

---

## Содержание

1. [Что такое дженерики и зачем они нужны](#что-такое-дженерики-и-зачем-они-нужны)
2. [Базовый синтаксис](#базовый-синтаксис)
3. [Дженерики в функциях](#дженерики-в-функциях)
4. [Дженерики в интерфейсах и типах](#дженерики-в-интерфейсах-и-типах)
5. [Дженерики в классах](#дженерики-в-классах)
6. [Constraints — ограничения типов](#constraints--ограничения-типов)
7. [Default types — типы по умолчанию](#default-types--типы-по-умолчанию)
8. [Дженерики в React](#дженерики-в-react)
9. [Дженерики во Vue](#дженерики-во-vue)
10. [Utility-типы как дженерики](#utility-типы-как-дженерики)
11. [Условные типы и infer](#условные-типы-и-infer)
12. [Mapped types](#mapped-types)
13. [Продвинутые паттерны](#продвинутые-паттерны)
14. [Типичные ошибки](#типичные-ошибки)

---

## Что такое дженерики и зачем они нужны

Представьте, что вам нужна функция, которая возвращает переданное значение. Без дженериков у вас два плохих варианта:

```typescript
// Вариант 1: теряем тип — возвращается any
function identity(value: any): any {
  return value;
}

const result = identity("hello"); // result: any — тип потерян

// Вариант 2: пишем отдельную функцию для каждого типа
function identityString(value: string): string {
  return value;
}
function identityNumber(value: number): number {
  return value;
}
```

Дженерики решают эту проблему — они позволяют передать тип как параметр:

```typescript
function identity<T>(value: T): T {
  return value;
}

const a = identity("hello");   // a: string
const b = identity(42);        // b: number
const c = identity(true);      // c: boolean
```

> **Аналогия:** Дженерик — это «переменная для типа». Как функция принимает значение и возвращает результат, так дженерик принимает тип и возвращает типизированный результат.

### Где дженерики встречаются в повседневном коде

| Контекст | Пример |
|---|---|
| React хуки | `useState<User>(initialUser)` |
| Vue composables | `useLocalStorage<User>('key', defaultUser)` |
| Fetch / API | `fetch<User>('/api/user')` |
| Коллекции | `Array<string>`, `Map<string, User>` |
| Utility-типы | `Partial<User>`, `Pick<User, 'name'>` |

---

## Базовый синтаксис

Дженерик объявляется угловыми скобками `<>` с именем типа (традиционно — заглавная буква):

```typescript
// Один параметр типа
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

// Несколько параметров типа
function pair<A, B>(a: A, b: B): [A, B] {
  return [a, b];
}

const p = pair("age", 25); // p: [string, number]
```

### Именование типов

Принятые соглашения:
- `T` — Type (один тип, самый частый случай)
- `K` — Key (ключ)
- `V` — Value (значение)
- `A, B` — когда нужно несколько разных типов
- `TData, TError` — описательные имена для сложных случаев

```typescript
// Хороший стиль для API-хука
function useFetch<TData, TError = Error>(
  url: string
): { data: TData | null; error: TError | null } {
  // ...
}
```

---

## Дженерики в функциях

### Функция с одним дженериком

```typescript
function getFirst<T>(items: T[]): T | undefined {
  return items[0];
}

const firstStr = getFirst(["a", "b", "c"]); // string | undefined
const firstNum = getFirst([1, 2, 3]);       // number | undefined
```

TypeScript **автоматически выводит** тип из аргументов — явно указывать `<string>` не нужно.

### Функция с несколькими дженериками

```typescript
function map<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn);
}

const lengths = map(["hello", "world"], s => s.length); // number[]
const upper = map(["hello", "world"], s => s.toUpperCase()); // string[]
```

### Функция с constraint (ограничением)

```typescript
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}

getLength("hello");     // OK — string имеет length
getLength([1, 2, 3]);   // OK — array имеет length
getLength(42);          // Error — number не имеет length
```

---

## Дженерики в интерфейсах и типах

### Generic interface

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

interface User {
  id: number;
  name: string;
}

const response: ApiResponse<User> = {
  data: { id: 1, name: "John" },
  status: 200,
  message: "OK"
};
```

### Generic type alias

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function parseJSON(json: string): Result<unknown> {
  try {
    return { ok: true, value: JSON.parse(json) };
  } catch (e) {
    return { ok: false, error: e as Error };
  }
}
```

### Интерфейс с несколькими типами

```typescript
interface Dictionary<K extends string | number, V> {
  get(key: K): V | undefined;
  set(key: K, value: V): void;
  keys(): K[];
}

type UserDict = Dictionary<string, { name: string; age: number }>;
```

---

## Дженерики в классах

### Базовый generic-класс

```typescript
class Container<T> {
  private items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  getFirst(): T | undefined {
    return this.items[0];
  }

  getAll(): T[] {
    return [...this.items];
  }
}

const numContainer = new Container<number>();
numContainer.add(1);
numContainer.add(2);
const first = numContainer.getFirst(); // number | undefined

const strContainer = new Container<string>();
strContainer.add("hello");
```

### Generic-класс с constraint

```typescript
interface Identifiable {
  id: string;
}

class Repository<T extends Identifiable> {
  private items: Map<string, T> = new Map();

  add(item: T): void {
    this.items.set(item.id, item);
  }

  getById(id: string): T | undefined {
    return this.items.get(id);
  }
}

interface User extends Identifiable {
  id: string;
  name: string;
}

const userRepo = new Repository<User>();
userRepo.add({ id: "1", name: "John" });
```

---

## Constraints — ограничения типов

Constraints позволяют ограничить, какие типы можно передать в дженерик.

### extends — базовое ограничение

```typescript
interface HasId {
  id: number;
}

function findById<T extends HasId>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);
}

interface User { id: number; name: string; }
interface Product { id: number; title: string; price: number; }

const users: User[] = [{ id: 1, name: "John" }];
const user = findById(users, 1); // User | undefined

const products: Product[] = [{ id: 1, title: "Phone", price: 500 }];
const product = findById(products, 1); // Product | undefined
```

### keyof — ограничение ключами объекта

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "John", age: 25 };

const name = getProperty(user, "name"); // string
const age = getProperty(user, "age");   // number
const wrong = getProperty(user, "email"); // Error — "email" не ключ user
```

### Multiple constraints

```typescript
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}

const merged = merge({ name: "John" }, { age: 25 });
// merged: { name: string } & { age: number }
```

---

## Default types — типы по умолчанию

```typescript
interface Pagination<T, TOrder extends "asc" | "desc" = "asc"> {
  items: T[];
  total: number;
  order: TOrder;
}

// TOrder по умолчанию "asc"
const page1: Pagination<User> = {
  items: [],
  total: 0,
  order: "asc" // можно не указывать
};

// Явно указываем "desc"
const page2: Pagination<User, "desc"> = {
  items: [],
  total: 0,
  order: "desc"
};
```

---

## Дженерики в React

### useState с дженериком

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile() {
  const [user, setUser] = useState<User | null>(null);
  const [users, setUsers] = useState<User[]>([]);

  // user: User | null
  // users: User[]
}
```

### useReducer с дженериком

```typescript
interface State {
  count: number;
  error: string | null;
}

type Action =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "setError"; payload: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + 1, error: null };
    case "decrement":
      return { ...state, count: state.count - 1, error: null };
    case "setError":
      return { ...state, error: action.payload };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, error: null });
}
```

### Generic-компоненты

```typescript
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
  items={[{ id: 1, name: "John" }, { id: 2, name: "Jane" }]}
  renderItem={user => <span>{user.name}</span>}
  keyExtractor={user => String(user.id)}
/>
```

### Generic forwardRef

```typescript
interface InputProps<T> {
  value: T;
  onChange: (value: T) => void;
  format?: (value: T) => string;
}

function InputInner<T>(
  { value, onChange, format }: InputProps<T>,
  ref: React.Ref<HTMLInputElement>
) {
  const display = format ? format(value) : String(value);
  return <input ref={ref} value={display} onChange={e => onChange(value)} />;
}

const Input = React.forwardRef(InputInner) as <T>(
  props: InputProps<T> & React.RefAttributes<HTMLInputElement>
) => React.ReactElement;
```

---

## Дженерики во Vue

### defineProps с дженериком

```vue
<script setup lang="ts" generic="T">
interface Props<T> {
  items: T[];
  selected: T | null;
  onSelect: (item: T) => void;
}

const props = defineProps<Props<T>>();
</script>

<template>
  <ul>
    <li v-for="item in items" @click="onSelect(item)">
      {{ item }}
    </li>
  </ul>
</template>
```

### Composable с дженериком

```typescript
import { ref, type Ref } from "vue";

function useLocalStorage<T>(key: string, initialValue: T) {
  const storedValue = ref<T>(initialValue) as Ref<T>;

  try {
    const item = localStorage.getItem(key);
    if (item) {
      storedValue.value = JSON.parse(item) as T;
    }
  } catch {
    storedValue.value = initialValue;
  }

  function setValue(value: T) {
    storedValue.value = value;
    localStorage.setItem(key, JSON.stringify(value));
  }

  return { value: storedValue, setValue };
}

// Использование
const { value: user, setValue: setUser } = useLocalStorage<User>("user", {
  id: 0,
  name: ""
});
```

### Generic defineEmits

```vue
<script setup lang="ts" generic="T extends { id: number }">
const props = defineProps<{
  items: T[];
}>();

const emit = defineEmits<{
  select: [item: T];
  remove: [id: number];
}>();
</script>
```

---

## Utility-типы как дженерики

TypeScript предоставляет встроенные utility-типы, которые сами являются дженериками:

### Partial, Required, Readonly

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

type PartialUser = Partial<User>;
// { name?: string; age?: number; email?: string; }

type RequiredUser = Required<User>;
// { name: string; age: number; email: string; }

type ReadonlyUser = Readonly<User>;
// { readonly name: string; readonly age: number; readonly email: string; }
```

### Pick, Omit

```typescript
type UserPreview = Pick<User, "name" | "age">;
// { name: string; age: number; }

type UserWithoutEmail = Omit<User, "email">;
// { name: string; age: number; }
```

### Record

```typescript
type UserById = Record<string, User>;
// { [key: string]: User }

type StatusCodes = Record<"success" | "error" | "loading", number>;
// { success: number; error: number; loading: number; }
```

### Extract, Exclude

```typescript
type Status = "idle" | "loading" | "success" | "error";

type ActiveStatus = Exclude<Status, "idle">;
// "loading" | "success" | "error"

type FinalStatus = Extract<Status, "success" | "error">;
// "success" | "error"
```

---

## Условные типы и infer

### Условные типы

```typescript
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<string>;  // "yes"
type B = IsString<number>;  // "no"
```

### infer — извлечение типа

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type Fn = () => { name: string };
type Result = ReturnType<Fn>; // { name: string }

type ElementType<T> = T extends (infer U)[] ? U : T;

type A = ElementType<string[]>; // string
type B = ElementType<number>;   // number
```

### Практический пример с infer

```typescript
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

type A = UnwrapPromise<Promise<string>>; // string
type B = UnwrapPromise<number>;          // number
```

---

## Mapped types

Mapped types позволяют создавать новые типы, трансформируя существующие:

### Базовый mapped type

```typescript
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

type Optional<T> = {
  [K in keyof T]?: T[K];
};
```

### Key remapping

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User {
  name: string;
  age: number;
}

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; }
```

---

## Продвинутые паттерны

### Фабрика с дженериком

```typescript
function createApiService<T extends { id: string }>(baseUrl: string) {
  return {
    async getAll(): Promise<T[]> {
      const res = await fetch(baseUrl);
      return res.json();
    },
    async getById(id: string): Promise<T | null> {
      const res = await fetch(`${baseUrl}/${id}`);
      return res.json();
    },
    async create(item: Omit<T, "id">): Promise<T> {
      const res = await fetch(baseUrl, {
        method: "POST",
        body: JSON.stringify(item)
      });
      return res.json();
    }
  };
}

interface User { id: string; name: string; }
const userService = createApiService<User>("/api/users");
```

### Типизированный event emitter

```typescript
type EventMap = Record<string, any[]>;

class TypedEmitter<T extends EventMap> {
  private listeners: { [K in keyof T]?: Array<(...args: T[K]) => void> } = {};

  on<K extends keyof T>(event: K, listener: (...args: T[K]) => void) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event]!.push(listener);
  }

  emit<K extends keyof T>(event: K, ...args: T[K]) {
    this.listeners[event]?.forEach(fn => fn(...args));
  }
}

interface AppEvents {
  login: [user: User];
  logout: [];
  error: [message: string, code: number];
}

const emitter = new TypedEmitter<AppEvents>();
emitter.on("login", user => console.log(user.name)); // OK
emitter.on("error", (msg, code) => console.log(msg, code)); // OK
```

### Curried функции с дженериками

```typescript
function curry<T, U, R>(fn: (a: T, b: U) => R) {
  return (a: T) => (b: U): R => fn(a, b);
}

const add = curry((a: number, b: number) => a + b);
const add5 = add(5);
const result = add5(3); // 8
```

---

## Типичные ошибки

### 1. Избыточные дженерики

```typescript
// Плохо — дженерик не нужен, тип и так выводится
function getValue<T>(value: T): T {
  return value;
}

// Хорошо — дженерик нужен для связи типов
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

### 2. Использование any вместо дженерика

```typescript
// Плохо — теряем типобезопасность
function first(arr: any[]): any {
  return arr[0];
}

// Хорошо — тип сохраняется
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

### 3. Отсутствие constraint при обращении к свойствам

```typescript
// Error — T может не иметь свойства length
function getLength<T>(item: T): number {
  return item.length;
}

// OK — ограничиваем тип
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}
```

### 4. Неправильное использование в React

```typescript
// Плохо — тип не передаётся в useState
const [user, setUser] = useState(null);
// user: null — нельзя присвоить User

// Хорошо — явно указываем тип
const [user, setUser] = useState<User | null>(null);
// user: User | null
```

### 5. Забытый дженерик в API-функции

```typescript
// Плохо — теряем тип ответа
async function fetchUser() {
  const res = await fetch("/api/user");
  return res.json(); // unknown
}

// Хорошо — типизируем
async function fetchUser(): Promise<User> {
  const res = await fetch("/api/user");
  return res.json() as Promise<User>;
}
```

---

## Заключение

Дженерики — фундамент типобезопасности в TypeScript. Они позволяют:

- Писать переиспользуемый код без потери типов
- Создавать гибкие API с проверкой на этапе компиляции
- Строить сложные utility-типы для трансформации данных
- Типизировать компоненты и хуки в React/Vue

**Правила для запоминания:**
1. Используйте дженерики, когда нужно связать типы входных и выходных данных
2. Добавляйте constraints (`extends`), когда нужны гарантии о структуре типа
3. Давайте описательные имена (`TData`, `TError`), если дженериков больше одного
4. Пользуйтесь встроенными utility-типами вместо написания своих
