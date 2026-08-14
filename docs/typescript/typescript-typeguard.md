# TypeGuard в TypeScript

TypeGuard — механизм сужения типов (type narrowing), позволяющий TypeScript понять, какой тип имеет переменная после проверки.

## Проблема

JavaScript-проверки работают только в рантайме. TypeScript не всегда может вывести тип из них:

```ts
const data: unknown = { name: "Ann", age: 25 };

if (typeof data === "object" && data !== null && "name" in data) {
  data.name; // ❌ Error: Property 'name' does not exist on type 'object'
}
```

Несмотря на корректную проверку, TS считает `data` типом `object` и не даёт доступ к свойствам.

## Встроенные TypeGuard

TypeScript автоматически сужает типы для некоторых проверок:

```ts
function process(value: string | number) {
  if (typeof value === "string") {
    value.toUpperCase(); // ✅ value: string
  }
}

function handle(value: Error | string) {
  if (value instanceof Error) {
    value.message; // ✅ value: Error
  }
}

function check(value: { type: "a" } | { type: "b" }) {
  if ("type" in value) {
    value.type; // ✅ value: { type: "a" } | { type: "b" }
  }
}

function validate(value: unknown) {
  if (Array.isArray(value)) {
    value.length; // ✅ value: unknown[]
  }
}
```

## Пользовательские TypeGuard

Для сложных проверок используются функции с предикатом типа `value is Type`:

```ts
interface User {
  name: string;
  age: number;
}

function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" &&
    obj !== null &&
    "name" in obj &&
    typeof (obj as any).name === "string" &&
    "age" in obj &&
    typeof (obj as any).age === "number"
  );
}

const data: unknown = { name: "Ann", age: 25 };

if (isUser(data)) {
  data.name; // ✅ data: User
  data.age;  // ✅ data: User
}
```

Ключевая часть — `obj is User`. Это предикат типа, который сообщает TypeScript: если функция вернула `true`, то `obj` имеет тип `User`.

## Практические примеры

### Валидация API-ответов

```ts
interface ApiResponse {
  status: "success" | "error";
  data?: { id: number; title: string };
  error?: string;
}

function isSuccessResponse(res: ApiResponse): res is ApiResponse & { data: NonNullable<ApiResponse["data"]> } {
  return res.status === "success" && res.data !== undefined;
}

async function fetchData(): Promise<ApiResponse> {
  const response = await fetch("/api/data");
  return response.json();
}

const result = await fetchData();

if (isSuccessResponse(result)) {
  console.log(result.data.id); // ✅ TypeScript знает, что data существует
} else {
  console.log(result.error); // ✅ TypeScript знает, что это error-ответ
}
```

### Discriminated Unions

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function isCircle(shape: Shape): shape is Extract<Shape, { kind: "circle" }> {
  return shape.kind === "circle";
}

function getArea(shape: Shape): number {
  if (isCircle(shape)) {
    return Math.PI * shape.radius ** 2; // ✅ shape: { kind: "circle"; radius: number }
  }

  if (shape.kind === "rectangle") {
    return shape.width * shape.height;
  }

  return (shape.base * shape.height) / 2;
}
```

### Валидация форм

```ts
interface FormData {
  email?: string;
  password?: string;
}

interface ValidFormData {
  email: string;
  password: string;
}

function isValidFormData(data: FormData): data is ValidFormData {
  return typeof data.email === "string" && data.email.length > 0 &&
         typeof data.password === "string" && data.password.length >= 8;
}

function submitForm(data: FormData) {
  if (!isValidFormData(data)) {
    throw new Error("Invalid form data");
  }

  // ✅ data: ValidFormData
  sendToServer(data.email, data.password);
}
```

## Ключевые моменты

- **`value is Type`** — предикат типа, сообщает компилятору о сужении
- Работают с `if`, `while`, тернарными операторами, `.filter()`, `.find()`
- Без предиката TypeScript не сузит тип `unknown` до конкретного
- Полезны для валидации данных из API, обработки `unknown`, discriminated unions
- Предикат должен соответствовать реальной логике проверки, иначе возникнут баги
