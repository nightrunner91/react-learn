# Типы возврата функций в TypeScript

Типизация возвращаемого значения — одна из ключевых возможностей TypeScript. Она позволяет зафиксировать контракт функции: что именно получит вызывающий код.

## Конкретные типы

Самый простой случай — функция возвращает значение конкретного типа:

```ts
function getName(): string {
  return "Alice";
}

function getAge(): number {
  return 25;
}

function isActive(): boolean {
  return true;
}
```

TypeScript проверяет, что `return` действительно возвращает значение указанного типа. Если нет — ошибка компиляции:

```ts
function getCount(): number {
  return "five"; // ❌ Type 'string' is not assignable to type 'number'
}
```

## void

`void` означает, что функция **ничего не возвращает**. Точнее, она завершается, но результат не имеет значения:

```ts
function log(message: string): void {
  console.log(message);
}

function notify(user: string): void {
  alert(`Hello, ${user}`);
}
```

Функция с `void` может технически содержать `return`, но без значения:

```ts
function doSomething(): void {
  if (condition) {
    return; // допустимо
  }
  // ...
}
```

Типичное применение — обработчики событий, мутации, побочные эффекты.

## never

`never` — функция **никогда не завершается нормально**. Это не то же самое, что `void`:

```ts
function throwError(msg: string): never {
  throw new Error(msg);
}

function infiniteLoop(): never {
  while (true) {}
}
```

`void` — функция дошла до конца, но ничего не вернула.
`never` — функция не дошла до конца вообще (исключение, бесконечный цикл).

### Exhaustive checks

Главное практическое применение `never` — проверка полноты `switch`:

```ts
type Status = "loading" | "success" | "error";

function render(status: Status) {
  switch (status) {
    case "loading": return "Загрузка...";
    case "success": return "Данные загружены";
    case "error":   return "Ошибка";
    default:
      const _exhaustive: never = status;
      return _exhaustive;
  }
}
```

Если добавить `"empty"` в `Status`, TypeScript выдаст ошибку в `default`-ветке — `status` уже не `never`. Это страховка от забытых случаев.

## any и unknown

Оба типа принимают любое значение, но разница принципиальна:

```ts
function parseAny(input: string): any {
  return JSON.parse(input);
}

function parseUnknown(input: string): unknown {
  return JSON.parse(input);
}

const a = parseAny("42");
a.toFixed(); // ✅ компилятор не ругается, но в рантайме может упасть

const b = parseUnknown("42");
b.toFixed(); // ❌ Error: Object is of type 'unknown'
```

`any` отключает проверку типов — это как вернуться в JavaScript.
`unknown` заставляет вызывающий код проверить тип перед использованием.

Правило: **всегда предпочитайте `unknown` вместо `any`**, если только нет веской причины.

## Union types

Функция может возвращать одно из нескольких значений:

```ts
function findUser(id: number): User | null {
  return users[id] ?? null;
}

function parse(input: string): string | number {
  const n = Number(input);
  return isNaN(n) ? input : n;
}
```

Вызывающий код обязан обработать все варианты через narrowing:

```ts
const result = parse("42");

if (typeof result === "string") {
  result.toUpperCase(); // ✅ result: string
} else {
  result.toFixed();     // ✅ result: number
}
```

## Generics

Когда тип возврата зависит от типа аргумента, используются дженерики:

```ts
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const n = first([1, 2, 3]);       // n: number | undefined
const s = first(["a", "b", "c"]); // s: string | undefined
```

Без дженерика пришлось бы использовать `any` или `unknown`, теряя информацию о типе:

```ts
function firstBad(arr: any[]): any {
  return arr[0];
}

const x = firstBad([1, 2, 3]); // x: any — тип потерян
```

Дженерики позволяют сохранять связь между аргументами и возвращаемым значением.

## Асинхронные функции (Promise)

`async`-функция всегда возвращает `Promise<T>`, где `T` — тип значения из `return`:

```ts
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

async function findUser(id: number): Promise<User | null> {
  const user = await fetchUser(id);
  return user ?? null;
}
```

Тип `Promise` указывается явно. Внутри `async`-функции достаточно написать `return value` — TypeScript оборачивает его в `Promise` автоматически.

## Объекты и интерфейсы

Функция может возвращать объект:

```ts
interface Point {
  x: number;
  y: number;
}

function createPoint(x: number, y: number): Point {
  return { x, y };
}
```

Можно описать тип инлайн, без отдельного интерфейса:

```ts
function createUser(name: string): { name: string; id: number } {
  return { name, id: Date.now() };
}
```

Для больших объектов предпочтительнее выносить тип в отдельный `interface` или `type`.

## Массивы и кортежи

```ts
function getNames(): string[] {
  return ["Alice", "Bob"];
}

function getPair(): [string, number] {
  return ["Alice", 25];
}
```

Массив (`string[]`) — любое количество элементов одного типа.
Кортеж (`[string, number]`) — фиксированное количество элементов с конкретными типами.

## Функции как возвращаемое значение

Функция может возвращать другую функцию:

```ts
function createMultiplier(factor: number): (value: number) => number {
  return (value) => value * factor;
}

const double = createMultiplier(2);
double(5); // 10
```

Тип `(value: number) => number` описывает сигнатуру возвращаемой функции.

## Вывод типов (type inference)

TypeScript часто может вывести тип возврата автоматически:

```ts
function add(a: number, b: number) {
  return a + b; // TS сам выводит: number
}

function getUser() {
  return { name: "Alice", age: 25 }; // TS выводит: { name: string; age: number }
}
```

Однако для публичных API **рекомендуется** указывать тип возврата явно:

```ts
// Хорошо — контракт зафиксирован
export function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// Плохо — при изменении реализации может случайно измениться тип
export function calculateTotal(items: CartItem[]) {
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

Явный тип возврата — это документация и страховка от случайных изменений.

## Таблица

| Тип | Когда использовать |
|-----|-------------------|
| `string`, `number`, `boolean`, объекты | Функция возвращает конкретное значение |
| `void` | Функция завершается, но не возвращает значение (побочные эффекты) |
| `never` | Функция не завершается (throw, бесконечный цикл) + exhaustive checks |
| `unknown` | Функция возвращает значение, тип которого неизвестен, но нужна проверка |
| `any` | Избегать. Только для миграции с JS |
| `T \| null`, `T \| undefined` | Функция может не найти результат |
| `T \| U` | Функция может вернуть разные типы |
| `Promise<T>` | Асинхронная функция |
| `T[]`, `[A, B]` | Массив или кортеж |
| `(x: T) => U` | Функция возвращает другую функцию |

## Заключение

Правильный выбор типа возврата делает код предсказуемым и безопасным. Основные правила:

1. **Конкретные типы** — по умолчанию для всех функций.
2. **`void`** — для функций с побочными эффектами.
3. **`never`** — для функций, которые не завершаются, и для exhaustive checks.
4. **`unknown`** вместо `any` — когда тип заранее неизвестен.
5. **Дженерики** — когда тип возврата зависит от аргументов.
6. **`Promise<T>`** — для асинхронных функций.
7. **Указывайте тип возврата явно** в публичных API.
