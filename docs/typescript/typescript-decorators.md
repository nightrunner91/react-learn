# TypeScript Decorators — полное руководство для React и Vue

Декораторы — мощный механизм TypeScript, позволяющий модифицировать поведение классов, методов, свойств и параметров через специальные аннотации. Они широко используются в фреймворках вроде NestJS, TypeORM и MobX. В React декораторы встречаются реже (в основном в legacy-коде с MobX), но во Vue 3 они стали основой для работы с class-based компонентами и декораторами свойств. В этой статье разберём, как работают декораторы, какие виды существуют и где применяются.

---

## Содержание

1. [Что такое декораторы](#что-такое-декораторы)
2. [Виды декораторов](#виды-декораторов)
3. [Декораторы классов](#декораторы-классов)
4. [Декораторы методов](#декораторы-методов)
5. [Декораторы свойств](#декораторы-свойств)
6. [Декораторы параметров](#декораторы-параметров)
7. [Декораторы доступа (get/set)](#декораторы-доступа-getset)
8. [Декораторы во Vue](#декораторы-во-vue)
9. [Декораторы в React](#декораторы-в-react)
10. [Декораторы в NestJS](#декораторы-в-nestjs)
11. [Метаданные и reflect-metadata](#метапрограммирование-и-reflect-metadata)
12. [Stage 3 Decorators (TC39)](#stage-3-decorators-tc39)
13. [Типичные ошибки](#типичные-ошибки)

---

## Что такое декораторы

Декоратор — это специальная функция, которая оборачивает элемент (класс, метод, свойство) и модифицирует его поведение. Синтаксически декораторы выглядят как `@имяДекоратора` перед объявлением.

```typescript
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class User {
  constructor(public name: string) {}
}
```

> **Аналогия:** Декоратор — как обёртка подарка. Подарок (класс/метод) остаётся тем же, но получает дополнительную «обёртку» — новое поведение или метаданные.

### Как работают декораторы

Декоратор — это обычная функция, которая вызывается при определении класса (не при создании экземпляра). Она получает информацию о декорируемом элементе и может:

- Модифицировать его
- Заменить его
- Добавить метаданные
- Вернуть новый дескриптор свойства

---

## Виды декораторов

| Вид | Что декорирует | Сигнатура |
|---|---|---|
| Class | Класс целиком | `(constructor: Function) => void` |
| Method | Метод класса | `(target, key, descriptor) => void` |
| Property | Свойство класса | `(target, key) => void` |
| Parameter | Параметр метода | `(target, key, index) => void` |
| Accessor | Getter/Setter | `(target, key, descriptor) => void` |

---

## Декораторы классов

Декоратор класса получает конструктор и может его модифицировать или заменить.

### Базовый декоратор класса

```typescript
function logClass(target: Function) {
  console.log(`Class created: ${target.name}`);
}

@logClass
class User {
  constructor(public name: string) {}
}

// При загрузке модуля выведет: "Class created: User"
```

### Декоратор, добавляющий метод

```typescript
function withTimestamp<T extends { new(...args: any[]): {} }>(constructor: T) {
  return class extends constructor {
    createdAt = new Date();

    getCreatedAt() {
      return this.createdAt;
    }
  };
}

@withTimestamp
class User {
  constructor(public name: string) {}
}

const user = new User("John");
console.log(user.getCreatedAt()); // Date
```

### Декоратор с параметрами

```typescript
function singleton(scope: "global" | "request" = "global") {
  return function <T extends { new(...args: any[]): {} }>(constructor: T) {
    let instance: any;

    return class extends constructor {
      constructor(...args: any[]) {
        if (!instance) {
          instance = new constructor(...args);
        }
        return instance;
      }
    };
  };
}

@singleton("global")
class Database {
  constructor(public connectionString: string) {}
}

const db1 = new Database("postgres://...");
const db2 = new Database("postgres://...");
console.log(db1 === db2); // true — один экземпляр
```

---

## Декораторы методов

Декоратор метода получает три аргумента:
- `target` — прототип класса (для static — сам класс)
- `key` — имя метода
- `descriptor` — дескриптор свойства (можно модифицировать)

### Логирование вызовов

```typescript
function logMethod(
  target: any,
  key: string,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  const original = descriptor.value;

  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${key} with`, args);
    const result = original.apply(this, args);
    console.log(`${key} returned`, result);
    return result;
  };

  return descriptor;
}

class Calculator {
  @logMethod
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
// "Calling add with [2, 3]"
// "add returned 5"
```

### Декоратор с параметрами

```typescript
function deprecated(reason: string) {
  return function (
    target: any,
    key: string,
    descriptor: PropertyDescriptor
  ): PropertyDescriptor {
    const original = descriptor.value;

    descriptor.value = function (...args: any[]) {
      console.warn(`DEPRECATED: ${key} is deprecated. ${reason}`);
      return original.apply(this, args);
    };

    return descriptor;
  };
}

class UserService {
  @deprecated("Use getUserById instead")
  getUser(id: number) {
    // ...
  }
}
```

### Кэширование

```typescript
function memoize() {
  return function (
    target: any,
    key: string,
    descriptor: PropertyDescriptor
  ): PropertyDescriptor {
    const original = descriptor.value;
    const cache = new Map<string, any>();

    descriptor.value = function (...args: any[]) {
      const cacheKey = JSON.stringify(args);

      if (cache.has(cacheKey)) {
        return cache.get(cacheKey);
      }

      const result = original.apply(this, args);
      cache.set(cacheKey, result);
      return result;
    };

    return descriptor;
  };
}

class MathService {
  @memoize()
  fibonacci(n: number): number {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
}
```

---

## Декораторы свойств

Декораторы свойств получают `target` (прототип) и `key` (имя свойства). Они **не могут** напрямую модифицировать значение — только добавлять метаданные.

### Базовый декоратор свойства

```typescript
function readonly(target: any, key: string) {
  Object.defineProperty(target, key, {
    writable: false,
    configurable: false
  });
}

class Config {
  @readonly
  apiUrl = "https://api.example.com";
}

const config = new Config();
config.apiUrl = "other"; // Error в strict mode
```

### Декоратор с валидацией

```typescript
function validate(pattern: RegExp, message: string) {
  return function (target: any, key: string) {
    let value: any;

    Object.defineProperty(target, key, {
      get() {
        return value;
      },
      set(newValue: any) {
        if (!pattern.test(newValue)) {
          throw new Error(message);
        }
        value = newValue;
      },
      configurable: true
    });
  };
}

class User {
  @validate(/^[a-zA-Z]+$/, "Name must contain only letters")
  name: string;

  @validate(/^\d{3}-\d{3}-\d{4}$/, "Invalid phone format")
  phone: string;
}

const user = new User();
user.name = "John";    // OK
user.name = "John123"; // Error: Name must contain only letters
```

---

## Декораторы параметров

Декораторы параметров не могут модифицировать значение — они используются для добавления метаданных.

### Базовый декоратор параметра

```typescript
function logParam(
  target: any,
  key: string,
  index: number
) {
  console.log(`Param ${index} in ${key}`);
}

class UserService {
  createUser(
    @logParam name: string,
    @logParam age: number
  ) {
    // ...
  }
}
// Выведет:
// "Param 0 in createUser"
// "Param 1 in createUser"
```

### Практическое применение — валидация

```typescript
const VALIDATION_KEY = Symbol("validation");

function required(target: any, key: string, index: number) {
  const existing = Reflect.getMetadata(VALIDATION_KEY, target, key) || [];
  existing.push({ index, rule: "required" });
  Reflect.defineMetadata(VALIDATION_KEY, existing, target, key);
}

function validateParams(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  const rules = Reflect.getMetadata(VALIDATION_KEY, target, key) || [];

  descriptor.value = function (...args: any[]) {
    for (const rule of rules) {
      if (rule.rule === "required" && (args[rule.index] === null || args[rule.index] === undefined)) {
        throw new Error(`Parameter ${rule.index} is required`);
      }
    }
    return original.apply(this, args);
  };

  return descriptor;
}

class Controller {
  @validateParams
  createUser(@required name: string, @required age: number) {
    // ...
  }
}
```

---

## Декораторы доступа (get/set)

Декораторы доступа применяются к getter/setter и работают аналогично декораторам методов.

```typescript
function format(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.get!;

  descriptor.get = function () {
    const value = original.call(this);
    return value.toUpperCase();
  };

  return descriptor;
}

class User {
  private _name = "john";

  @format
  get name() {
    return this._name;
  }
}

const user = new User();
console.log(user.name); // "JOHN"
```

---

## Декораторы во Vue

### vue-property-decorator

Во Vue 2/3 с class-based API декораторы используются для объявления компонентов:

```typescript
import { Component, Prop, Emit, Watch, Vue } from "vue-property-decorator";

@Component
export default class UserCard extends Vue {
  @Prop({ required: true })
  userId!: number;

  @Prop({ default: "Guest" })
  displayName!: string;

  user: User | null = null;

  @Watch("userId", { immediate: true })
  async onUserIdChange(newId: number) {
    this.user = await fetchUser(newId);
  }

  @Emit("selected")
  selectUser() {
    return this.user;
  }

  get fullName() {
    return this.user ? `${this.user.firstName} ${this.user.lastName}` : "";
  }
}
```

### Vue 3 + Class Component (vue-facing-decorator)

```typescript
import { Component, Prop, Watch, Vue } from "vue-facing-decorator";

@Component({
  emits: ["selected"]
})
export default class UserCard extends Vue {
  @Prop({ required: true })
  userId!: number;

  user: User | null = null;

  @Watch("userId")
  onUserIdChange() {
    this.fetchUser();
  }

  mounted() {
    this.fetchUser();
  }

  fetchUser() {
    // ...
  }
}
```

### Декораторы для реактивности

```typescript
import { Mutation, Action, State } from "vuex-class";

@Component
export default class UserComponent extends Vue {
  @State("users")
  users!: User[];

  @Mutation("SET_USERS")
  setUsers!: (users: User[]) => void;

  @Action("fetchUsers")
  fetchUsers!: () => Promise<void>;

  async created() {
    await this.fetchUsers();
  }
}
```

---

## Декораторы в React

В React декораторы используются реже, но встречаются в связке с MobX:

### MobX декораторы

```typescript
import { makeObservable, observable, computed, action } from "mobx";

class UserStore {
  @observable
  users: User[] = [];

  @observable
  loading = false;

  @computed
  get activeUsers() {
    return this.users.filter(u => u.isActive);
  }

  @action
  addUser(user: User) {
    this.users.push(user);
  }

  constructor() {
    makeObservable(this);
  }
}
```

### Декораторы для компонентов (legacy)

```typescript
// Декоратор для подключения к контексту
function withTheme<T extends object>(
  WrappedComponent: React.ComponentType<T>
) {
  return class extends React.Component<T> {
    static contextType = ThemeContext;

    render() {
      return (
        <WrappedComponent
          {...this.props}
          theme={this.context}
        />
      );
    }
  };
}

@withTheme
class Button extends React.Component<ButtonProps & { theme: Theme }> {
  render() {
    return <button style={{ color: this.props.theme.primary }}>Click</button>;
  }
}
```

---

## Декораторы в NestJS

NestJS активно использует декораторы для построения приложений:

### Контроллеры и маршруты

```typescript
import { Controller, Get, Post, Body, Param } from "@nestjs/common";

@Controller("users")
export class UserController {
  @Get()
  findAll(): Promise<User[]> {
    return this.userService.findAll();
  }

  @Get(":id")
  findOne(@Param("id") id: string): Promise<User> {
    return this.userService.findOne(id);
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto): Promise<User> {
    return this.userService.create(createUserDto);
  }
}
```

### Сервисы и инъекция зависимостей

```typescript
import { Injectable, Inject } from "@nestjs/common";

@Injectable()
export class UserService {
  constructor(
    @Inject("USER_REPOSITORY")
    private userRepository: Repository<User>
  ) {}

  async findAll(): Promise<User[]> {
    return this.userRepository.find();
  }
}
```

### Валидация с декораторами

```typescript
import { IsString, IsEmail, MinLength, MaxLength } from "class-validator";

export class CreateUserDto {
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

---

## Метапрограммирование и reflect-metadata

Декораторы часто используются вместе с `reflect-metadata` для добавления метаданных:

### Установка

```bash
npm install reflect-metadata
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

### Практическое применение

```typescript
import "reflect-metadata";

const ROUTE_KEY = Symbol("route");

function Get(path: string) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    Reflect.defineMetadata(ROUTE_KEY, { method: "GET", path }, target, key);
  };
}

function Post(path: string) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    Reflect.defineMetadata(ROUTE_KEY, { method: "POST", path }, target, key);
  };
}

class UserController {
  @Get("/users")
  findAll() {
    return [];
  }

  @Post("/users")
  create() {
    return {};
  }
}

// Чтение метаданных
const metadata = Reflect.getMetadata(ROUTE_KEY, UserController.prototype, "findAll");
console.log(metadata); // { method: "GET", path: "/users" }
```

---

## Stage 3 Decorators (TC39)

TypeScript поддерживает как legacy-декораторы (experimental), так и новые Stage 3 декораторы (стандарт ECMAScript).

### Отличия

| Характеристика | Legacy | Stage 3 |
|---|---|---|
| Сигнатура | `(target, key, descriptor)` | `(value, context)` |
| Поддержка TS | `experimentalDecorators: true` | По умолчанию (TS 5.0+) |
| Статус | Устаревший | Стандарт |
| Доступ к `this` | В runtime | Через `context.access` |

### Stage 3 синтаксис

```typescript
// Stage 3 декоратор метода
function logged(value: any, context: ClassMethodDecoratorContext) {
  if (context.kind === "method") {
    return function (...args: any[]) {
      console.log(`Calling ${String(context.name)}`);
      return value.apply(this, args);
    };
  }
}

class User {
  @logged
  greet() {
    return "Hello";
  }
}
```

### Stage 3 декоратор класса

```typescript
function registered<T extends new (...args: any[]) => any>(
  value: T,
  context: ClassDecoratorContext
) {
  return class extends value {
    constructor(...args: any[]) {
      super(...args);
      console.log(`Instance of ${String(context.name)} created`);
    }
  };
}

@registered
class User {
  constructor(public name: string) {}
}
```

---

## Типичные ошибки

### 1. Забытый experimentalDecorators

```json
// tsconfig.json — без этого декораторы не работают
{
  "compilerOptions": {
    "experimentalDecorators": true
  }
}
```

### 2. Неправильная сигнатура декоратора

```typescript
// Error — декоратор метода должен возвращать PropertyDescriptor
function log(target: any, key: string) {
  // Нет descriptor — нельзя модифицировать метод
}

// OK
function log(target: any, key: string, descriptor: PropertyDescriptor) {
  // ...
  return descriptor;
}
```

### 3. Путаница между legacy и Stage 3

```typescript
// Legacy
function legacy(target: any, key: string, descriptor: PropertyDescriptor) {
  // ...
}

// Stage 3
function stage3(value: any, context: ClassMethodDecoratorContext) {
  // ...
}
```

### 4. Декоратор свойства не может изменить значение

```typescript
// Error — декоратор свойства не получает descriptor
function init(target: any, key: string) {
  target[key] = "initial"; // Это не сработает как ожидается
}

// OK — используйте getter/setter
function init(target: any, key: string) {
  let value: any;
  Object.defineProperty(target, key, {
    get() { return value ?? "default"; },
    set(v: any) { value = v; }
  });
}
```

### 5. Отсутствие reflect-metadata

```typescript
// Error — Reflect.defineMetadata не существует
Reflect.defineMetadata("key", "value", target);

// OK — импортируйте polyfill
import "reflect-metadata";
Reflect.defineMetadata("key", "value", target);
```

---

## Заключение

Декораторы — мощный инструмент для:

- **Метапрограммирования** — добавление метаданных к классам и методам
- **AOP (Aspect-Oriented Programming)** — логирование, валидация, кэширование
- **DI (Dependency Injection)** — инъекция зависимостей в NestJS
- **Реактивности** — MobX, Vuex class-based stores

**Когда использовать:**
- В NestJS — обязательно (контроллеры, сервисы, DTO)
- В MobX — для class-based stores
- Во Vue — для class-based компонентов (legacy)
- В React — редко, в основном с MobX

**Когда не использовать:**
- В функциональном коде (React hooks, Vue composables)
- Когда достаточно обычных функций или HOC
- Если команда не знакома с паттерном
