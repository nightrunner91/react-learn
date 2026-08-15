# TypeScript Utility Types — полное руководство

Utility-типы в TypeScript — это набор встроенных типов, которые позволяют трансформировать и модифицировать другие типы. Но прежде чем учить их наизусть, давай разберёмся, **зачем они вообще нужны** и **как работают под капотом**.

В этой статье мы не просто посмотрим на примеры — мы поймём философию каждого utility-типа, разберём его реализацию и научимся применять осознанно.

---

## Содержание

1. [Фундамент: что такое трансформация типов](#фундамент-что-такое-трансформация-типов)
2. [Mapped Types — основа всех utility-типов](#mapped-types--основа-всех-utility-типов)
3. [Conditional Types — условные типы](#conditional-types--условные-типы)
4. [Partial — когда не все поля обязательны](#partial--когда-не-все-поля-обязательны)
5. [Required — когда всё обязательно](#required--когда-всё-обязательно)
6. [Readonly — защита от мутаций](#readonly--защита-от-мутаций)
7. [Record — типизированные словари](#record--типизированные-словари)
8. [Pick — выбор нужных полей](#pick--выбор-нужных-полей)
9. [Omit — исключение лишних полей](#omit--исключение-лишних-полей)
10. [Exclude — фильтрация union типов](#exclude--фильтрация-union-типов)
11. [Extract — извлечение из union](#extract--извлечение-из-union)
12. [ReturnType — что возвращает функция](#returntype--что-возвращает-функция)
13. [Parameters — что принимает функция](#parameters--что-принимает-функция)
14. [NonNullable — прощай, null и undefined](#nonnullable--прощай-null-и-undefined)
15. [Awaited — распаковка Promise](#awaited--распаковка-promise)
16. [Комбинация utility-типов](#комбинация-utility-типов)
17. [Практика в React](#практика-в-react)
18. [Практика во Vue](#практика-во-vue)
19. [Типичные ошибки](#типичные-ошибки)
20. [Шпаргалка](#шпаргалка)

---

## Фундамент: что такое трансформация типов

Прежде чем изучать конкретные utility-типы, нужно понять **базовую концепцию**: в TypeScript типы — это не просто ярлыки. Это **значения**, которые можно трансформировать, комбинировать и передавать как параметры.

### Аналогия: типы как данные

Представь, что ты работаешь с массивами:

```typescript
const numbers = [1, 2, 3, 4, 5];

// Ты не создаёшь новый массив вручную — ты трансформируешь существующий
const doubled = numbers.map(n => n * 2); // [2, 4, 6, 8, 10]
const evens = numbers.filter(n => n % 2 === 0); // [2, 4]
```

С типами работает так же:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

// Ты не создаёшь новый тип вручную — ты трансформируешь существующий
type PartialUser = Partial<User>; // Все поля опциональны
type UserPreview = Pick<User, "id" | "name">; // Только id и name
```

> **Ключевая идея:** Utility-типы — это **функции для типов**. Они принимают тип как аргумент и возвращают новый тип.

### Почему это важно?

Без utility-типов тебе пришлось бы писать дублирующий код:

```typescript
// Плохо — дублирование
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

interface UserUpdate {
  id?: number;
  name?: string;
  email?: string;
  age?: number;
}

// Хорошо — трансформация
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type UserUpdate = Partial<User>; // Автоматически делает все поля опциональными
```

Что происходит, когда ты добавляешь новое поле в `User`? В первом случае тебе нужно не забыть обновить `UserUpdate`. Во втором случае — `Partial<User>` автоматически подхватит новое поле.

**Это и есть сила utility-типов: они устраняют дублирование и синхронизируют типы.**

---

## Mapped Types — основа всех utility-типов

Прежде чем разбирать конкретные utility-типы, нужно понять **mapped types** (отображаемые типы). Это механизм, на котором построены `Partial`, `Required`, `Readonly` и многие другие.

### Что такое mapped types?

Mapped types позволяют создавать новые типы, **итерируя по ключам существующего типа** и трансформируя их.

```typescript
// Синтаксис mapped type
type MappedType<T> = {
  [K in keyof T]: T[K];
};
```

Разберём по частям:

1. **`keyof T`** — это оператор, который возвращает union всех ключей типа `T`.
   
   ```typescript
   interface User {
     id: number;
     name: string;
   }
   
   type UserKeys = keyof User; // "id" | "name"
   ```

2. **`[K in keyof T]`** — это цикл, который перебирает каждый ключ.
   
   Представь это как `for...in` в JavaScript:
   ```typescript
   // JavaScript
   for (const key in object) { ... }
   
   // TypeScript (mapped type)
   type Mapped = {
     [K in keyof T]: T[K];
   };
   ```

3. **`T[K]`** — это **indexed access type**. Он возвращает тип значения для ключа `K`.
   
   ```typescript
   interface User {
     id: number;
     name: string;
   }
   
   type UserIdType = User["id"]; // number
   type UserNameType = User["name"]; // string
   ```

### Пример: создаём свой mapped type

Давай напишем свой utility-тип, который делает все поля массивами:

```typescript
interface User {
  id: number;
  name: string;
}

// Все поля становятся массивами
type Arrayify<T> = {
  [K in keyof T]: T[K][];
};

type UserArrays = Arrayify<User>;
// { id: number[]; name: string[]; }
```

Как это работает:
1. `keyof User` → `"id" | "name"`
2. Для каждого ключа `K` берём `T[K]` (тип значения) и добавляем `[]`
3. Результат: `{ id: number[]; name: string[]; }`

### Модификаторы в mapped types

В mapped types можно изменять модификаторы полей:

```typescript
// Добавить модификатор (readonly, ?)
type Readonly<T> = {
  readonly [K in keyof T]: T[K]; // Добавляем readonly
};

type Optional<T> = {
  [K in keyof T]?: T[K]; // Добавляем ? (опциональность)
};

// Убрать модификатор (используем -readonly или -?)
type Mutable<T> = {
  -readonly [K in keyof T]: T[K]; // Убираем readonly
};

type Required<T> = {
  [K in keyof T]-?: T[K]; // Убираем ? (делаем обязательным)
};
```

> **Важно:** Символ `-` перед модификатором означает "убрать этот модификатор". Это как `!` в CSS — переопределение.

### Почему это важно понимать?

Потому что `Partial`, `Required`, `Readonly` — это **всего лишь mapped types** с разными модификаторами. Когда ты поймёшь mapped types, ты поймёшь как работают все эти utility-типы.

---

## Conditional Types — условные типы

Второй фундаментальный механизм — это **conditional types** (условные типы). Они позволяют создавать типы, которые зависят от условия.

### Синтаксис

```typescript
type ConditionalType<T> = T extends U ? X : Y;
```

Читай это так: "Если `T` подходит под `U`, то результат — `X`, иначе — `Y`".

### Простой пример

```typescript
type IsString<T> = T extends string ? "да" : "нет";

type A = IsString<string>; // "да"
type B = IsString<number>; // "нет"
type C = IsString<"hello">; // "да" (литеральный тип подходит под string)
```

### Как это работает?

TypeScript проверяет, **можно ли присвоить `T` к `U`**:

```typescript
type Test1 = string extends string ? true : false; // true
type Test2 = number extends string ? true : false; // false
type Test3 = "hello" extends string ? true : false; // true (литерал подходит под string)
type Test4 = string extends "hello" ? true : false; // false (string не подходит под конкретный литерал)
```

### Distributive conditional types

Когда ты используешь union тип в conditional type, TypeScript **распределяет** условие по каждому члену union:

```typescript
type IsString<T> = T extends string ? "строка" : "не строка";

type Result = IsString<string | number | boolean>;
// "строка" | "не строка" | "не строка"
// Упрощается до: "строка" | "не строка"
```

Что произошло:
1. `string extends string ? "строка" : "не строка"` → `"строка"`
2. `number extends string ? "строка" : "не строка"` → `"не строка"`
3. `boolean extends string ? "строка" : "не строка"` → `"не строка"`
4. Результат: `"строка" | "не строка" | "не строка"` → `"строка" | "не строка"`

Это поведение называется **distributive** и оно критически важно для `Exclude` и `Extract`.

### Пример: извлечение типа из массива

```typescript
type ElementType<T> = T extends (infer U)[] ? U : never;

type A = ElementType<string[]>; // string
type B = ElementType<number[]>; // number
```

Здесь мы используем `infer` для извлечения типа элемента массива. `infer` позволяет "вывести" тип из паттерна.

### Почему это важно?

Conditional types — это основа для `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters` и многих других. Без понимания conditional types ты не поймёшь, как работают эти utility-типы.

---

## Partial — когда не все поля обязательны

### Какую проблему решает?

Представь, что у тебя есть тип `User`:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}
```

Теперь тебе нужно написать функцию обновления пользователя. Ты хочешь, чтобы можно было обновить **любое** поле, но не требовать все поля сразу:

```typescript
function updateUser(id: number, updates: ???): Promise<User> {
  // updates может содержать любые поля User
}
```

Без `Partial` тебе пришлось бы писать:

```typescript
interface UserUpdate {
  id?: number;
  name?: string;
  email?: string;
  age?: number;
}

function updateUser(id: number, updates: UserUpdate): Promise<User> {
  // ...
}
```

Но это дублирование! Когда ты добавишь новое поле в `User`, тебе нужно не забыть обновить `UserUpdate`.

`Partial` решает эту проблему:

```typescript
function updateUser(id: number, updates: Partial<User>): Promise<User> {
  // updates может содержать любые поля User, все они опциональны
}
```

### Что делает Partial?

`Partial<T>` делает **все свойства типа `T` опциональными** (добавляет `?`).

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>;
// Результат:
// {
//   id?: number;
//   name?: string;
//   email?: string;
// }
```

### Как реализован Partial?

```typescript
type Partial<T> = {
  [K in keyof T]?: T[K];
};
```

Разбор по шагам:

1. **`keyof T`** — получаем все ключи типа `T`.
   - Для `User` это `"id" | "name" | "email"`

2. **`[K in keyof T]`** — перебираем каждый ключ.
   - Это как цикл: для каждого ключа `K` из `"id" | "name" | "email"`

3. **`?`** — добавляем модификатор опциональности.
   - Каждое поле становится необязательным

4. **`T[K]`** — берём тип значения для ключа `K`.
   - Для `"id"` это `number`
   - Для `"name"` это `string`
   - Для `"email"` это `string`

5. **Результат** — новый тип с теми же ключами, но все поля опциональны.

### Визуализация

```
interface User {
  id: number;        →  id?: number;
  name: string;      →  name?: string;
  email: string;     →  email?: string;
}

Partial<User> превращает обязательные поля в опциональные
```

### Когда использовать?

**1. Обновление объекта (update)**

Когда ты хочешь обновить только некоторые поля объекта:

```typescript
function updateUser(id: number, updates: Partial<User>): Promise<User> {
  return api.put(`/users/${id}`, updates);
}

updateUser(1, { name: "John" }); // OK — обновляем только name
updateUser(1, { email: "john@example.com", age: 25 }); // OK — обновляем email и age
updateUser(1, {}); // OK — пустые обновления
```

**2. Начальное состояние формы**

Когда форма заполняется постепенно:

```typescript
interface FormState {
  username: string;
  password: string;
  email: string;
  rememberMe: boolean;
}

const initialForm: Partial<FormState> = {};
// Все поля опциональны — можно заполнять постепенно

initialForm.username = "john";
initialForm.password = "secret";
// и так далее
```

**3. Конфигурация с дефолтами**

Когда у тебя есть дефолтная конфигурация, и пользователь может переопределить только некоторые поля:

```typescript
interface Config {
  host: string;
  port: number;
  timeout: number;
  retries: number;
}

const defaultConfig: Config = {
  host: "localhost",
  port: 3000,
  timeout: 5000,
  retries: 3
};

function createConfig(options: Partial<Config>): Config {
  return { ...defaultConfig, ...options };
}

const config = createConfig({ port: 8080 });
// { host: "localhost", port: 8080, timeout: 5000, retries: 3 }
```

### Когда НЕ использовать?

Не используй `Partial` для объектов, где **все поля обязательны по смыслу**:

```typescript
// Плохо — все поля опциональны, но не все должны быть
interface FormState {
  username: string;
  password: string;
  email: string;
}

const form: Partial<FormState> = {}; // Можно создать пустой объект — это плохо!

// Хорошо — явное разделение
interface FormState {
  username: string;
  password: string;
  email: string;
}

interface PartialFormState {
  username?: string;
  password?: string;
  email?: string;
}
```

Если ты используешь `Partial<FormState>`, ты говоришь TypeScript: "Все поля опциональны". Но если по смыслу формы все поля обязательны, это приведёт к ошибкам.

---

## Required — когда всё обязательно

### Какую проблему решает?

`Required<T>` — это противоположность `Partial<T>`. Он делает **все свойства обязательными** (убирает `?`).

Типичный сценарий: у тебя есть интерфейс с опциональными полями, но после валидации ты хочешь гарантировать, что все поля заполнены.

```typescript
interface FormData {
  username?: string;
  email?: string;
  age?: number;
}

function validate(data: FormData): Required<FormData> {
  if (!data.username) throw new Error("Username required");
  if (!data.email) throw new Error("Email required");
  if (!data.age) throw new Error("Age required");
  
  // После валидации все поля гарантированно есть
  return data as Required<FormData>;
}

const validData = validate({ username: "john", email: "john@example.com", age: 25 });
// validData: { username: string; email: string; age: number; }
```

### Что делает Required?

`Required<T>` делает **все свойства типа `T` обязательными** (убирает `?`).

```typescript
interface User {
  id?: number;
  name?: string;
  email?: string;
}

type RequiredUser = Required<User>;
// Результат:
// {
//   id: number;
//   name: string;
//   email: string;
// }
```

### Как реализован Required?

```typescript
type Required<T> = {
  [K in keyof T]-?: T[K];
};
```

Разбор:

1. **`[K in keyof T]`** — перебираем все ключи типа `T`.

2. **`-?`** — **убираем** модификатор опциональности.
   - Символ `-` перед `?` означает "удалить этот модификатор"
   - Это как `!important` в CSS — переопределение

3. **`T[K]`** — берём тип значения для ключа `K`.

### Визуализация

```
interface User {
  id?: number;       →  id: number;
  name?: string;     →  name: string;
  email?: string;    →  email: string;
}

Required<User> превращает опциональные поля в обязательные
```

### Когда использовать?

**1. Валидация формы**

Когда ты хочешь гарантировать, что после валидации все поля заполнены:

```typescript
interface FormData {
  username?: string;
  email?: string;
  age?: number;
}

function validate(data: FormData): Required<FormData> {
  if (!data.username) throw new Error("Username required");
  if (!data.email) throw new Error("Email required");
  if (!data.age) throw new Error("Age required");
  
  return data as Required<FormData>;
}
```

**2. Конфигурация после применения дефолтов**

Когда у тебя есть опциональная конфигурация, но после применения дефолтов все поля обязательны:

```typescript
interface AppConfig {
  apiUrl?: string;
  timeout?: number;
  retries?: number;
}

const defaults: Required<AppConfig> = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
  retries: 3
};

function getConfig(userConfig: AppConfig): Required<AppConfig> {
  return { ...defaults, ...userConfig } as Required<AppConfig>;
}
```

---

## Readonly — защита от мутаций

### Какую проблему решает?

В JavaScript объекты передаются по ссылке. Это означает, что если ты передашь объект в функцию, она может его изменить:

```typescript
const user = { id: 1, name: "John" };

function updateUser(user: User) {
  user.name = "Jane"; // Мутирует оригинальный объект!
}

updateUser(user);
console.log(user.name); // "Jane" — оригинальный объект изменён!
```

В функциональном программировании и в React/Vue принято **не мутировать** объекты, а создавать новые. `Readonly` помогает соблюдать это правило на уровне типов.

### Что делает Readonly?

`Readonly<T>` делает **все свойства типа `T` readonly** (только для чтения).

```typescript
interface User {
  id: number;
  name: string;
}

type ReadonlyUser = Readonly<User>;
// Результат:
// {
//   readonly id: number;
//   readonly name: string;
// }

const user: ReadonlyUser = { id: 1, name: "John" };
user.name = "Jane"; // Error: Cannot assign to 'name' because it is a read-only property.
```

### Как реализован Readonly?

```typescript
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};
```

Разбор:

1. **`[K in keyof T]`** — перебираем все ключи типа `T`.

2. **`readonly`** — добавляем модификатор readonly.
   - Это запрещает изменение свойства после инициализации

3. **`T[K]`** — берём тип значения для ключа `K`.

### Визуализация

```
interface User {
  id: number;        →  readonly id: number;
  name: string;      →  readonly name: string;
}

Readonly<User> запрещает изменение свойств
```

### Когда использовать?

**1. Константы и неизменяемые данные**

Когда ты хочешь защитить конфигурацию от случайного изменения:

```typescript
interface Config {
  apiUrl: string;
  timeout: number;
}

const CONFIG: Readonly<Config> = {
  apiUrl: "https://api.example.com",
  timeout: 5000
};

CONFIG.timeout = 10000; // Error: Cannot assign to 'timeout' because it is a read-only property.
```

**2. Redux state / Vuex state**

В Redux и Vuex состояние должно быть неизменяемым. `Readonly` помогает соблюдать это правило:

```typescript
interface State {
  user: User | null;
  loading: boolean;
  error: string | null;
}

type ReadonlyState = Readonly<State>;

function reducer(state: ReadonlyState, action: Action): ReadonlyState {
  // state нельзя мутировать — только создавать новый объект
  return { ...state, loading: false };
}
```

**3. Защита от мутаций в функциях**

Когда ты хочешь гарантировать, что функция не изменит переданный объект:

```typescript
function processUser(user: Readonly<User>) {
  user.name = "Jane"; // Error — нельзя мутировать
  console.log(user.name); // OK — чтение разрешено
}
```

### Важное замечание

`Readonly` защищает только от изменения **ссылок на свойства**, а не от мутации вложенных объектов:

```typescript
interface User {
  name: string;
  address: {
    street: string;
    city: string;
  };
}

const user: Readonly<User> = {
  name: "John",
  address: { street: "Main St", city: "NY" }
};

user.name = "Jane"; // Error — нельзя изменить name
user.address.street = "Broadway"; // OK — можно изменить вложенный объект!
```

Для глубокой защиты нужен `DeepReadonly`:

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

---

## Record — типизированные словари

### Какую проблему решает?

В JavaScript часто используют объекты как словари (maps):

```typescript
const userById = {
  "1": { id: 1, name: "John" },
  "2": { id: 2, name: "Jane" }
};
```

Но как это типизировать? Можно написать:

```typescript
const userById: { [key: string]: User } = {
  "1": { id: 1, name: "John" },
  "2": { id: 2, name: "Jane" }
};
```

Но это многословно. `Record` предоставляет более читаемый синтаксис:

```typescript
type UserById = Record<string, User>;

const userById: UserById = {
  "1": { id: 1, name: "John" },
  "2": { id: 2, name: "Jane" }
};
```

### Что делает Record?

`Record<K, V>` создаёт тип объекта с ключами типа `K` и значениями типа `V`.

```typescript
type UserById = Record<string, User>;
// Эквивалентно:
// { [key: string]: User }
```

### Как реализован Record?

```typescript
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

Разбор:

1. **`K extends keyof any`** — ограничиваем `K` типами `string | number | symbol`.
   - Это значит, что ключи могут быть только строками, числами или символами

2. **`[P in K]`** — перебираем каждый ключ из `K`.

3. **`T`** — для каждого ключа значение имеет тип `T`.

### Примеры

**1. Словарь с числовыми ключами**

```typescript
interface User {
  id: number;
  name: string;
}

type UserById = Record<number, User>;

const userMap: UserById = {
  1: { id: 1, name: "John" },
  2: { id: 2, name: "Jane" }
};
```

**2. Статусы с значениями**

Когда у тебя есть набор статусов и каждому соответствует значение:

```typescript
type Status = "idle" | "loading" | "success" | "error";
type StatusLabel = Record<Status, string>;

const labels: StatusLabel = {
  idle: "Ожидание",
  loading: "Загрузка",
  success: "Успех",
  error: "Ошибка"
};
// TypeScript проверит, что все статусы указаны
```

**3. Конфигурация для разных окружений**

```typescript
type Environment = "dev" | "staging" | "production";
type ApiUrl = Record<Environment, string>;

const apiUrls: ApiUrl = {
  dev: "http://localhost:3000",
  staging: "https://staging.example.com",
  production: "https://example.com"
};
// TypeScript проверит, что все окружения указаны
```

**4. Группировка данных**

```typescript
type GroupedUsers = Record<string, User[]>;

const grouped: GroupedUsers = {
  admins: [{ id: 1, name: "Admin" }],
  users: [{ id: 2, name: "User1" }, { id: 3, name: "User2" }]
};
```

### Когда использовать?

- Когда тебе нужен **словарь** (map) с известными ключами
- Когда ты хочешь **гарантировать**, что все ключи указаны
- Когда ты хочешь **типовую безопасность** для значений

### Когда НЕ использовать?

Не используй `Record<string, T>`, если у тебя **динамические ключи** без ограничений:

```typescript
// Плохо — можно добавить любой ключ, нет типовой безопасности
const users: Record<string, User> = {};
users["any-key"] = user; // Нет проверки

// Хорошо — известные ключи
type Status = "idle" | "loading" | "success" | "error";
const statusLabels: Record<Status, string> = {
  idle: "Ожидание",
  loading: "Загрузка",
  success: "Успех",
  error: "Ошибка"
};
// TypeScript проверит, что все ключи указаны
```

---

## Pick — выбор нужных полей

### Какую проблему решает?

Часто у тебя есть большой тип, но тебе нужна только его часть. Например, у тебя есть `Product` с множеством полей, но для карточки товара нужны только `id`, `title`, `price` и `images`.

Без `Pick` тебе пришлось бы писать:

```typescript
interface Product {
  id: number;
  title: string;
  description: string;
  price: number;
  images: string[];
  category: string;
}

interface ProductCard {
  id: number;
  title: string;
  price: number;
  images: string[];
}
```

Это дублирование! Когда ты изменишь `Product`, тебе нужно не забыть обновить `ProductCard`.

`Pick` решает эту проблему:

```typescript
type ProductCard = Pick<Product, "id" | "title" | "price" | "images">;
```

### Что делает Pick?

`Pick<T, K>` создаёт новый тип, **выбирая только указанные свойства `K` из типа `T`**.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  address: string;
}

type UserPreview = Pick<User, "id" | "name">;
// Результат:
// {
//   id: number;
//   name: string;
// }
```

### Как реализован Pick?

```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

Разбор:

1. **`K extends keyof T`** — ограничиваем `K` только ключами из `T`.
   - Это значит, что ты не можешь выбрать несуществующие поля

2. **`[P in K]`** — перебираем только выбранные ключи.

3. **`T[P]`** — берём тип значения для ключа `P`.

### Визуализация

```
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

Pick<User, "id" | "name">:
  id: number;       ← выбираем
  name: string;     ← выбираем
  email: string;    ← игнорируем
  age: number;      ← игнорируем

Результат: { id: number; name: string; }
```

### Когда использовать?

**1. Превью/карточки**

Когда тебе нужна только часть полей для отображения:

```typescript
interface Product {
  id: number;
  title: string;
  description: string;
  price: number;
  images: string[];
  category: string;
}

type ProductCard = Pick<Product, "id" | "title" | "price" | "images">;

function ProductCard({ product }: { product: ProductCard }) {
  return (
    <div>
      <h3>{product.title}</h3>
      <img src={product.images[0]} alt={product.title} />
      <span>${product.price}</span>
    </div>
  );
}
```

**2. API ответы с частичными данными**

Когда API возвращает только часть полей:

```typescript
interface FullUser {
  id: number;
  name: string;
  email: string;
  phone: string;
  address: string;
  createdAt: Date;
  updatedAt: Date;
}

type UserSummary = Pick<FullUser, "id" | "name" | "email">;

async function getUsers(): Promise<UserSummary[]> {
  const response = await fetch("/api/users/summary");
  return response.json();
}
```

**3. Props компонента**

Когда компоненту нужны только некоторые поля:

```typescript
interface InputProps {
  type: "text" | "password" | "email";
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  disabled?: boolean;
  className?: string;
}

type CoreInputProps = Pick<InputProps, "type" | "value" | "onChange">;

function SimpleInput(props: CoreInputProps) {
  return <input {...props} />;
}
```

---

## Omit — исключение лишних полей

### Какую проблему решает?

`Omit<T, K>` — это противоположность `Pick<T, K>`. Вместо того чтобы выбирать нужные поля, ты **исключаешь** ненужные.

Типичный сценарий: у тебя есть тип `User` с полем `password`, но перед отправкой на клиент ты хочешь его убрать.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type PublicUser = Omit<User, "password">;
// { id: number; name: string; email: string; }
```

### Что делает Omit?

`Omit<T, K>` создаёт новый тип, **исключая указанные свойства `K` из типа `T`**.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type UserWithoutAge = Omit<User, "age">;
// Результат:
// {
//   id: number;
//   name: string;
//   email: string;
// }
```

### Как реализован Omit?

```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

Это может выглядеть сложно, но давай разберём по шагам:

1. **`keyof T`** — получаем все ключи типа `T`.
   - Для `User` это `"id" | "name" | "email" | "age"`

2. **`Exclude<keyof T, K>`** — исключаем ключи `K` из всех ключей.
   - `Exclude<"id" | "name" | "email" | "age", "age">` → `"id" | "name" | "email"`

3. **`Pick<T, ...>`** — выбираем оставшиеся ключи.
   - `Pick<User, "id" | "name" | "email">` → `{ id: number; name: string; email: string; }`

### Визуализация

```
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

Omit<User, "age">:
  id: number;       ← оставляем
  name: string;     ← оставляем
  email: string;    ← оставляем
  age: number;      ← исключаем

Результат: { id: number; name: string; email: string; }
```

### Когда использовать?

**1. Удаление чувствительных данных перед отправкой**

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: string;
}

type PublicUser = Omit<User, "password">;

function getPublicUser(user: User): PublicUser {
  const { password, ...publicUser } = user;
  return publicUser;
}
```

**2. Создание типа для создания (create)**

Когда ты создаёшь новый объект, и некоторые поля (id, createdAt, updatedAt) генерируются автоматически:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;
}

type CreateUser = Omit<User, "id" | "createdAt" | "updatedAt">;
// { name: string; email: string; }

async function createUser(data: CreateUser): Promise<User> {
  const response = await fetch("/api/users", {
    method: "POST",
    body: JSON.stringify(data)
  });
  return response.json();
}
```

**3. Упрощение типов**

Когда тебе не нужны все поля для определённой операции:

```typescript
interface DetailedProduct {
  id: number;
  title: string;
  description: string;
  price: number;
  images: string[];
  category: string;
  reviews: Review[];
  rating: number;
}

type SimpleProduct = Omit<DetailedProduct, "reviews" | "rating">;

function ProductList({ products }: { products: SimpleProduct[] }) {
  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>{product.title}</li>
      ))}
    </ul>
  );
}
```

---

## Exclude — фильтрация union типов

### Какую проблему решает?

У тебя есть union тип, и ты хочешь **исключить** некоторые его члены.

```typescript
type Status = "idle" | "loading" | "success" | "error";

// Ты хочешь тип, который исключает "idle" и "loading"
// То есть только финальные состояния: "success" | "error"
```

Без `Exclude` тебе пришлось бы писать вручную:

```typescript
type FinalStatus = "success" | "error";
```

Но если `Status` изменится, тебе нужно не забыть обновить `FinalStatus`.

`Exclude` решает эту проблему:

```typescript
type FinalStatus = Exclude<Status, "idle" | "loading">;
// "success" | "error"
```

### Что делает Exclude?

`Exclude<T, U>` исключает из union типа `T` все типы, которые присутствуют в `U`.

```typescript
type Status = "idle" | "loading" | "success" | "error";

type ActiveStatus = Exclude<Status, "idle">;
// "loading" | "success" | "error"

type FinalStatus = Exclude<Status, "idle" | "loading">;
// "success" | "error"
```

### Как реализован Exclude?

```typescript
type Exclude<T, U> = T extends U ? never : T;
```

Это **conditional type** с **дистрибутивным поведением**. Разберём по шагам:

1. **`T extends U`** — для каждого члена union `T` проверяем, подходит ли он под `U`.

2. **`? never : T`** — если подходит, возвращаем `never` (исключаем), иначе возвращаем сам тип.

3. **Дистрибутивность** — TypeScript применяет условие к каждому члену union отдельно.

### Пример работы

```typescript
type Status = "idle" | "loading" | "success" | "error";
type ExcludeIdle = Exclude<Status, "idle">;

// TypeScript делает следующее:
// "idle" extends "idle" ? never : "idle" → never
// "loading" extends "idle" ? never : "loading" → "loading"
// "success" extends "idle" ? never : "success" → "success"
// "error" extends "idle" ? never : "error" → "error"

// Результат: never | "loading" | "success" | "error"
// never исчезает из union, остаётся: "loading" | "success" | "error"
```

### Когда использовать?

**1. Исключение состояний**

```typescript
type RequestState = "idle" | "loading" | "success" | "error";

type CompletedState = Exclude<RequestState, "idle" | "loading">;
// "success" | "error"

function handleCompleted(state: CompletedState) {
  if (state === "success") {
    console.log("Успешно!");
  } else {
    console.log("Ошибка!");
  }
}
```

**2. Фильтрация типов**

```typescript
type AllEvents = "click" | "hover" | "focus" | "blur" | "submit";
type MouseEvents = Exclude<AllEvents, "focus" | "blur" | "submit">;
// "click" | "hover"
```

**3. Исключение null/undefined из union**

```typescript
type MaybeString = string | null | undefined;

type DefinitelyString = Exclude<MaybeString, null | undefined>;
// string
```

---

## Extract — извлечение из union

### Какую проблему решает?

`Extract<T, U>` — это противоположность `Exclude<T, U>`. Вместо исключения ты **извлекаешь** только те типы, которые присутствуют в `U`.

```typescript
type Status = "idle" | "loading" | "success" | "error";

// Ты хочешь только финальные состояния
type FinalStatus = Extract<Status, "success" | "error">;
// "success" | "error"
```

### Что делает Extract?

`Extract<T, U>` извлекает из union типа `T` только те типы, которые присутствуют в `U`.

```typescript
type Status = "idle" | "loading" | "success" | "error";

type FinalStatus = Extract<Status, "success" | "error">;
// "success" | "error"

type LoadingStatus = Extract<Status, "loading">;
// "loading"
```

### Как реализован Extract?

```typescript
type Extract<T, U> = T extends U ? T : never;
```

Это противоположность `Exclude`:

1. **`T extends U`** — для каждого члена union `T` проверяем, подходит ли он под `U`.

2. **`? T : never`** — если подходит, возвращаем сам тип, иначе возвращаем `never`.

### Пример работы

```typescript
type Status = "idle" | "loading" | "success" | "error";
type ExtractFinal = Extract<Status, "success" | "error">;

// TypeScript делает следующее:
// "idle" extends "success" | "error" ? "idle" : never → never
// "loading" extends "success" | "error" ? "loading" : never → never
// "success" extends "success" | "error" ? "success" : never → "success"
// "error" extends "success" | "error" ? "error" : never → "error"

// Результат: never | never | "success" | "error"
// Остаётся: "success" | "error"
```

### Когда использовать?

**1. Извлечение конкретных состояний**

```typescript
type RequestState = "idle" | "loading" | "success" | "error";

type SuccessState = Extract<RequestState, "success">;

function onSuccess(state: SuccessState) {
  // state гарантированно "success"
  console.log("Успех!");
}
```

**2. Извлечение примитивных типов**

```typescript
type Mixed = string | number | boolean | null;

type Primitives = Extract<Mixed, string | number>;
// string | number
```

**3. Извлечение типов из discriminated union**

```typescript
type Action =
  | { type: "increment"; payload: number }
  | { type: "decrement"; payload: number }
  | { type: "reset" };

type IncrementAction = Extract<Action, { type: "increment" }>;
// { type: "increment"; payload: number }
```

---

## ReturnType — что возвращает функция

### Какую проблему решает?

Часто тебе нужно узнать, что возвращает функция, но ты не хочешь дублировать тип:

```typescript
function getUser(): { id: number; name: string } {
  return { id: 1, name: "John" };
}

// Ты хочешь использовать тип возвращаемого значения
type User = { id: number; name: string }; // Дублирование!
```

`ReturnType` извлекает тип возвращаемого значения автоматически:

```typescript
function getUser(): { id: number; name: string } {
  return { id: 1, name: "John" };
}

type User = ReturnType<typeof getUser>;
// { id: number; name: string }
```

### Что делает ReturnType?

`ReturnType<T>` извлекает тип возвращаемого значения функции.

```typescript
function getUser(): { id: number; name: string } {
  return { id: 1, name: "John" };
}

type UserReturn = ReturnType<typeof getUser>;
// { id: number; name: string }
```

### Как реализован ReturnType?

```typescript
type ReturnType<T extends (...args: any) => any> = 
  T extends (...args: any) => infer R ? R : any;
```

Разбор:

1. **`T extends (...args: any) => any`** — ограничиваем `T` только функциями.

2. **`T extends (...args: any) => infer R`** — проверяем, что `T` — это функция, и **выводим** тип возвращаемого значения в `R`.

3. **`? R : any`** — если это функция, возвращаем `R`, иначе `any`.

### Когда использовать?

**1. Извлечение типа из API функций**

```typescript
async function fetchUsers(): Promise<User[]> {
  const response = await fetch("/api/users");
  return response.json();
}

type UsersType = ReturnType<typeof fetchUsers>;
// Promise<User[]>

type UsersTypeUnwrapped = Awaited<ReturnType<typeof fetchUsers>>;
// User[]
```

**2. Типизация хуков**

```typescript
function useUser(id: number) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(false);
  
  return { user, loading, setUser };
}

type UseUserReturn = ReturnType<typeof useUser>;
// { user: User | null; loading: boolean; setUser: ... }
```

**3. Извлечение типа из factory функций**

```typescript
function createCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count
  };
}

type Counter = ReturnType<typeof createCounter>;
// { increment: () => number; decrement: () => number; getCount: () => number }
```

---

## Parameters — что принимает функция

### Какую проблему решает?

`Parameters<T>` извлекает тип параметров функции как кортеж.

```typescript
function createUser(name: string, age: number): User {
  return { id: 1, name, age };
}

type CreateUserParams = Parameters<typeof createUser>;
// [name: string, age: number]

const params: CreateUserParams = ["John", 25];
const user = createUser(...params);
```

### Как реализован Parameters?

```typescript
type Parameters<T extends (...args: any) => any> = 
  T extends (...args: infer P) => any ? P : never;
```

Разбор:

1. **`T extends (...args: any) => any`** — ограничиваем `T` только функциями.

2. **`T extends (...args: infer P) => any`** — проверяем, что `T` — это функция, и **выводим** тип параметров в `P`.

3. **`? P : never`** — если это функция, возвращаем `P`, иначе `never`.

### Когда использовать?

**1. Типизация callback функций**

```typescript
function fetchData(url: string, options: RequestInit): Promise<Response> {
  return fetch(url, options);
}

type FetchParams = Parameters<typeof fetchData>;
// [url: string, options: RequestInit]

function customFetch(...args: FetchParams) {
  console.log("Fetching:", args[0]);
  return fetchData(...args);
}
```

**2. Извлечение типа первого параметра**

```typescript
type FirstParam<T> = Parameters<T>[0];

function handleClick(event: React.MouseEvent<HTMLButtonElement>) {
  // ...
}

type EventType = FirstParam<typeof handleClick>;
// React.MouseEvent<HTMLButtonElement>
```

**3. Обёртки над функциями**

```typescript
function logAndCall<T extends (...args: any[]) => any>(
  fn: T,
  ...args: Parameters<T>
): ReturnType<T> {
  console.log("Calling function with:", args);
  return fn(...args);
}

function add(a: number, b: number): number {
  return a + b;
}

const result = logAndCall(add, 1, 2); // 3
```

---

## NonNullable — прощай, null и undefined

### Какую проблему решает?

Часто у тебя есть тип, который может быть `null` или `undefined`, но ты хочешь работать только с "чистым" типом:

```typescript
type MaybeString = string | null | undefined;

// Ты хочешь только string
type DefinitelyString = NonNullable<MaybeString>;
// string
```

### Что делает NonNullable?

`NonNullable<T>` исключает `null` и `undefined` из типа `T`.

```typescript
type MaybeString = string | null | undefined;

type DefinitelyString = NonNullable<MaybeString>;
// string

type MaybeNumber = number | null;
type DefinitelyNumber = NonNullable<MaybeNumber>;
// number
```

### Как реализован NonNullable?

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
```

Разбор:

1. **`T extends null | undefined`** — для каждого члена union `T` проверяем, является ли он `null` или `undefined`.

2. **`? never : T`** — если да, возвращаем `never` (исключаем), иначе возвращаем сам тип.

### Когда использовать?

**1. После проверки на null**

```typescript
function processUser(user: User | null) {
  if (!user) return;
  
  const nonNullUser: NonNullable<typeof user> = user;
  console.log(user.name);
}
```

**2. Фильтрация массива**

```typescript
const values: (string | null | undefined)[] = ["a", null, "b", undefined, "c"];

const filtered: NonNullable<string | null | undefined>[] = values.filter(
  (v): v is NonNullable<typeof v> => v != null
);
// ["a", "b", "c"]
```

**3. Извлечение типа из опционального поля**

```typescript
interface User {
  name: string;
  email?: string | null;
}

type Email = NonNullable<User["email"]>;
// string
```

---

## Awaited — распаковка Promise

### Какую проблему решает?

Когда ты работаешь с async функциями, тип возвращаемого значения — это `Promise<T>`. Но тебе часто нужен сам `T`:

```typescript
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

type UserFromApi = Awaited<ReturnType<typeof fetchUser>>;
// User
```

### Что делает Awaited?

`Awaited<T>` извлекает тип из `Promise<T>`.

```typescript
type PromiseString = Promise<string>;
type StringValue = Awaited<PromiseString>;
// string

type PromiseNumber = Promise<number>;
type NumberValue = Awaited<PromiseNumber>;
// number
```

### Как реализован Awaited?

```typescript
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
```

Разбор:

1. **`T extends Promise<infer U>`** — проверяем, что `T` — это Promise, и **выводим** тип значения в `U`.

2. **`? Awaited<U> : T`** — если это Promise, рекурсивно вызываем `Awaited<U>` (на случай вложенных Promise), иначе возвращаем `T`.

### Когда использовать?

**1. Извлечение типа из API функций**

```typescript
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

type UserFromApi = Awaited<ReturnType<typeof fetchUser>>;
// User
```

**2. Работа с вложенными Promise**

```typescript
type NestedPromise = Promise<Promise<string>>;
type Unwrapped = Awaited<NestedPromise>;
// string (рекурсивно разворачивает все Promise)
```

**3. Типизация результатов Promise.all**

```typescript
const promises = [
  fetchUser(1),
  fetchPosts(),
  fetchComments()
];

type Results = Awaited<typeof promises[number]>;
// User | Post[] | Comment[]
```

---

## Комбинация utility-типов

Utility-типы можно комбинировать для создания сложных трансформаций.

### Пример 1: Partial + Pick

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type UserUpdate = Partial<Pick<User, "name" | "email" | "age">>;
// { name?: string; email?: string; age?: number; }
```

Что происходит:
1. `Pick<User, "name" | "email" | "age">` → `{ name: string; email: string; age: number; }`
2. `Partial<...>` → `{ name?: string; email?: string; age?: number; }`

### Пример 2: Omit + Partial

```typescript
type CreateUser = Partial<Omit<User, "id">>;
// { name?: string; email?: string; age?: number; }
```

Что происходит:
1. `Omit<User, "id">` → `{ name: string; email: string; age: number; }`
2. `Partial<...>` → `{ name?: string; email?: string; age?: number; }`

### Пример 3: Record + Pick

```typescript
type UserById = Record<number, Pick<User, "name" | "email">>;
// { [key: number]: { name: string; email: string; } }
```

### Пример 4: Deep Partial (кастомный)

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

interface User {
  name: string;
  address: {
    street: string;
    city: string;
    country: string;
  };
}

type PartialUser = DeepPartial<User>;
// {
//   name?: string;
//   address?: {
//     street?: string;
//     city?: string;
//     country?: string;
//   };
// }
```

### Пример 5: Mutable (противоположность Readonly)

```typescript
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type ReadonlyUser = Readonly<User>;
type MutableUser = Mutable<ReadonlyUser>;
// Снова можно изменять свойства
```

---

## Практика в React

### Partial в компонентах

```typescript
interface UserCardProps {
  user: User;
  showEmail?: boolean;
  showAge?: boolean;
  onEdit?: () => void;
}

function UserCard(props: UserCardProps) {
  const { user, showEmail, showAge, onEdit } = props;
  
  return (
    <div>
      <h3>{user.name}</h3>
      {showEmail && <p>{user.email}</p>}
      {showAge && <p>{user.age}</p>}
      {onEdit && <button onClick={onEdit}>Редактировать</button>}
    </div>
  );
}

// Использование Partial для дефолтных props
const defaultProps: Partial<UserCardProps> = {
  showEmail: true,
  showAge: false
};
```

### Pick для props

```typescript
interface InputProps {
  type: string;
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  disabled?: boolean;
  className?: string;
  id?: string;
}

type TextInputProps = Pick<InputProps, "value" | "onChange" | "placeholder" | "disabled">;

function TextInput(props: TextInputProps) {
  return <input type="text" {...props} />;
}
```

### ReturnType для хуков

```typescript
function useCounter(initialValue: number = 0) {
  const [count, setCount] = useState(initialValue);
  
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initialValue);
  
  return { count, increment, decrement, reset };
}

type CounterReturn = ReturnType<typeof useCounter>;
// { count: number; increment: () => void; decrement: () => void; reset: () => void }

function CounterDisplay({ counter }: { counter: CounterReturn }) {
  return (
    <div>
      <span>{counter.count}</span>
      <button onClick={counter.increment}>+</button>
      <button onClick={counter.decrement}>-</button>
      <button onClick={counter.reset}>Сброс</button>
    </div>
  );
}
```

### Omit для HOC

```typescript
interface WithLoadingProps {
  isLoading: boolean;
}

function withLoading<P extends object>(
  Component: React.ComponentType<P>
) {
  type Props = Omit<P, keyof WithLoadingProps> & WithLoadingProps;
  
  return function WithLoadingComponent(props: Props) {
    if (props.isLoading) {
      return <div>Загрузка...</div>;
    }
    
    const componentProps = props as unknown as P;
    return <Component {...componentProps} />;
  };
}
```

---

## Практика во Vue

### Partial в компонентах

```vue
<script setup lang="ts">
interface Props {
  user: User;
  showEmail?: boolean;
  showAge?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  showEmail: true,
  showAge: false
});

// Или использование Partial для опциональных обновлений
const updates: Partial<User> = { name: "John" };
</script>

<template>
  <div>
    <h3>{{ user.name }}</h3>
    <p v-if="showEmail">{{ user.email }}</p>
    <p v-if="showAge">{{ user.age }}</p>
  </div>
</template>
```

### Pick для props

```vue
<script setup lang="ts">
interface FullUserProps {
  id: number;
  name: string;
  email: string;
  age: number;
  address: string;
  phone: string;
}

type UserPreviewProps = Pick<FullUserProps, "id" | "name" | "email">;

const props = defineProps<UserPreviewProps>();
</script>

<template>
  <div>
    <h3>{{ name }}</h3>
    <p>{{ email }}</p>
  </div>
</template>
```

### ReturnType для composables

```typescript
function useCounter(initialValue: number = 0) {
  const count = ref(initialValue);
  
  const increment = () => count.value++;
  const decrement = () => count.value--;
  const reset = () => count.value = initialValue;
  
  return { count, increment, decrement, reset };
}

type CounterReturn = ReturnType<typeof useCounter>;

// Использование в другом компоненте
function CounterDisplay(props: { counter: CounterReturn }) {
  return {
    count: props.counter.count,
    increment: props.counter.increment
  };
}
```

### Record для словарей

```vue
<script setup lang="ts">
type Status = "idle" | "loading" | "success" | "error";
type StatusConfig = Record<Status, { label: string; color: string }>;

const statusConfig: StatusConfig = {
  idle: { label: "Ожидание", color: "gray" },
  loading: { label: "Загрузка", color: "blue" },
  success: { label: "Успех", color: "green" },
  error: { label: "Ошибка", color: "red" }
};

const currentStatus = ref<Status>("idle");
</script>

<template>
  <div :class="statusConfig[currentStatus].color">
    {{ statusConfig[currentStatus].label }}
  </div>
</template>
```

---

## Типичные ошибки

### 1. Использование utility-типов вместо интерфейсов

```typescript
// Плохо — теряется читаемость
type UserProps = Pick<User, "id" | "name"> & Partial<Pick<User, "email">>;

// Хорошо — явный интерфейс
interface UserProps {
  id: number;
  name: string;
  email?: string;
}
```

### 2. Избыточное использование Partial

```typescript
// Плохо — все поля опциональны, но не все должны быть
interface FormState {
  username: string;
  password: string;
  email: string;
}

const form: Partial<FormState> = {}; // Можно создать пустой объект

// Хорошо — явное разделение
interface FormState {
  username: string;
  password: string;
  email: string;
}

interface PartialFormState {
  username?: string;
  password?: string;
  email?: string;
}
```

### 3. Забывание ReturnType для сложных функций

```typescript
// Плохо — дублирование типа
function useUser() {
  const [user, setUser] = useState<User | null>(null);
  return { user, setUser };
}

type UserState = { user: User | null; setUser: (user: User | null) => void };

// Хорошо — извлечение типа
type UserState = ReturnType<typeof useUser>;
```

### 4. Неправильное использование Record

```typescript
// Плохо — можно добавить любой ключ
const users: Record<string, User> = {};
users["any-key"] = user; // Нет проверки

// Хорошо — известные ключи
type Status = "idle" | "loading" | "success" | "error";
const statusLabels: Record<Status, string> = {
  idle: "Ожидание",
  loading: "Загрузка",
  success: "Успех",
  error: "Ошибка"
};
// TypeScript проверит, что все ключи указаны
```

### 5. Игнорирование комбинаций utility-типов

```typescript
// Плохо — создание нового типа вручную
interface CreateUser {
  name?: string;
  email?: string;
  age?: number;
}

// Хорошо — использование utility-типов
type CreateUser = Partial<Omit<User, "id">>;
```

---

## Шпаргалка

### Основные utility-типы

```typescript
// Сделать все поля опциональными
type PartialUser = Partial<User>;

// Сделать все поля обязательными
type RequiredUser = Required<User>;

// Сделать все поля readonly
type ReadonlyUser = Readonly<User>;

// Создать словарь
type UserById = Record<string, User>;

// Выбрать только указанные поля
type UserPreview = Pick<User, "id" | "name">;

// Исключить указанные поля
type UserWithoutPassword = Omit<User, "password">;

// Исключить из union
type ActiveStatus = Exclude<Status, "idle">;

// Извлечь из union
type FinalStatus = Extract<Status, "success" | "error">;

// Тип возвращаемого значения функции
type UserReturn = ReturnType<typeof getUser>;

// Тип параметров функции
type CreateUserParams = Parameters<typeof createUser>;

// Исключить null и undefined
type DefinitelyString = NonNullable<MaybeString>;

// Извлечь тип из Promise
type StringValue = Awaited<Promise<string>>;
```

### Комбинации

```typescript
// Partial + Pick
type UserUpdate = Partial<Pick<User, "name" | "email">>;

// Omit + Partial
type CreateUser = Partial<Omit<User, "id">>;

// Record + Pick
type UserById = Record<number, Pick<User, "name" | "email">>;
```

### Когда что использовать

| Задача | Utility-тип |
|---|---|
| Обновление объекта | `Partial<T>` |
| Валидация формы | `Required<T>` |
| Неизменяемые данные | `Readonly<T>` |
| Словарь/мапа | `Record<K, V>` |
| Превью/карточка | `Pick<T, K>` |
| Создание объекта | `Omit<T, K>` |
| Исключение состояний | `Exclude<T, U>` |
| Извлечение состояний | `Extract<T, U>` |
| Типизация хуков | `ReturnType<T>` |
| Обёртки функций | `Parameters<T>` |
| Фильтрация null/undefined | `NonNullable<T>` |
| Работа с async/await | `Awaited<T>` |

---

## Заключение

Utility-типы — это мощный инструмент TypeScript для трансформации типов. Они позволяют:

- Избежать дублирования кода
- Создавать гибкие и переиспользуемые типы
- Улучшить типобезопасность
- Упростить работу с React/Vue компонентами

**Правила для запоминания:**
1. Используй `Partial` для обновлений и форм
2. Используй `Pick` для превью и карточек
3. Используй `Omit` для исключения чувствительных данных
4. Используй `Record` для словарей и мап
5. Комбинируй utility-типы для сложных трансформаций
6. Используй `ReturnType` и `Parameters` для типизации функций и хуков

Практикуйся — и через неделю ты будешь использовать utility-типы интуитивно.
