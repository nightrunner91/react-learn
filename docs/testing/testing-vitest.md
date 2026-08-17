# Unit-тестирование с Vitest

## Содержание

1. [Что такое Vitest](#что-такое-vitest)
2. [Установка и настройка](#установка-и-настройка)
3. [Базовый API: describe, it, expect](#базовый-api-describe-it-expect)
4. [Matchers — проверки значений](#matchers--проверки-значений)
5. [Асинхронные тесты](#асинхронные-тесты)
6. [Мокирование: vi.fn, vi.spyOn, vi.mock](#мокирование-vifn-vispyon-vimock)
7. [Фейковые таймеры](#фейковые-таймеры)
8. [Snapshot-тестирование](#snapshot-тестирование)
9. [Параметризованные тесты](#параметризованные-тесты)
10. [Тестирование хуков](#тестирование-хуков)
11. [Покрытие кода](#покрытие-кода)
12. [Конфигурация для React и Next.js](#конфигурация-для-react-и-nextjs)
13. [Vitest vs Jest](#vitest-vs-jest)
14. [Лучшие практики](#лучшие-практики)

---

## Что такое Vitest

**Vitest** — это фреймворк для тестирования, созданный командой Vite. Он предоставляет API, совместимый с Jest, но работает нативно с Vite — использует те же трансформации, конфигурацию и систему модулей.

### Почему Vitest, а не Jest

| Характеристика | Vitest | Jest |
|---|---|---|
| **Интеграция с Vite** | Нативная — одна конфигурация | Требует отдельную настройку (babel, ts-jest) |
| **ESM** | Поддержка из коробки | Частичная, требует экспериментальных флагов |
| **TypeScript** | Работает без дополнительной настройки | Требует ts-jest или babel-jest |
| **Скорость** | Быстрый (использует Vite-кэш) | Средний |
| **Watch mode** | Нативный HMR-режим | File-based watch |
| **Snapshot** | Совместим с Jest snapshots | Jest snapshots |
| **API** | Совместим с Jest | Оригинальный API |
| **Coverage** | Встроенный (v8, istanbul) | Встроенный (v8, babel) |

> 💡 **Правило:** Если проект на Vite — используйте Vitest. Если проект на Webpack — Jest остаётся хорошим выбором. Миграция с Jest на Vitest обычно безболезненна благодаря совместимости API.

---

## Установка и настройка

Настройка Vitest сводится к установке пакета и созданию конфигурационного файла. Главное отличие от Jest — Vitest использует ту же конфигурационную инфраструктуру, что и Vite, поэтому не нужно дублировать настройки трансформаций, алиасов и плагинов.

### Базовая установка

Установка минимальна — достаточно одного пакета. Vitest не требует Babel, ts-jest или других трансформеров, потому что использует Vite-пайплайн.

```bash
npm install -D vitest
```

### Конфигурация

Конфигурационный файл `vitest.config.ts` определяет, какие файлы считать тестами, в каком окружении их запускать и что исключать. Окружение `node` подходит для утилит и серверного кода; для React-компонентов потребуется `jsdom` (см. ниже).

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["**/*.{test,spec}.{ts,tsx}"],
    exclude: ["node_modules", "dist"],
  },
});
```

### Для React-проекта

Для React нужно три вещи: плагин `@vitejs/plugin-react` (чтобы JSX трансформировался), окружение `jsdom` (эмуляция DOM в Node.js) и setup-файл для глобальной настройки тестов. Опция `css: true` позволяет тестам работать с CSS-модулями без ошибок.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    css: true,
    include: ["src/**/*.{test,spec}.{ts,tsx}"],
  },
});
```

### Setup-файл

Setup-файл выполняется перед каждым тестовым файлом. Здесь подключают дополнительные matcher'ы из `@testing-library/jest-dom` и настраивают автоматическую очистку DOM после каждого теста, чтобы тесты не влияли друг на друга.

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach } from "vitest";

afterEach(() => {
  cleanup();
});
```

### Расширенная конфигурация

Расширенная конфигурация добавляет coverage-провайдер и алиасы путей. Coverage-секция определяет пороги покрытия — CI упадёт, если покрытие опустится ниже указанных значений. Алиас `@` → `/src` дублируется в `resolve.alias` (для Vite) и `test.alias` (для Vitest), потому что это два разных пайплайна разрешения модулей.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    css: true,

    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html"],
      include: ["src/**/*.{ts,tsx}"],
      exclude: [
        "src/test/**",
        "src/**/*.d.ts",
        "src/**/*.stories.tsx",
      ],
      thresholds: {
        statements: 80,
        branches: 70,
        functions: 85,
        lines: 80,
      },
    },

    alias: {
      "@": "/src",
    },
  },
  resolve: {
    alias: {
      "@": "/src",
    },
  },
});
```

### Скрипты в package.json

Типичный набор скриптов: `vitest` без аргументов запускает watch-режим (перезапуск при изменениях), `vitest run` — одноразовый прогон для CI, `--coverage` — генерация отчёта покрытия.

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage",
    "test:watch": "vitest watch"
  }
}
```

---

## Базовый API: describe, it, expect

Три кита любого теста: `describe` группирует связанные проверки в блок, `it` (или `test`) описывает один конкретный сценарий, `expect` проверяет результат. Хорошая структура `describe`/`it` читается как документация — по ней видно, что должна делать функция, без чтения кода.

### describe — группировка тестов

`describe` создаёт контекстную группу тестов. Вложенные `describe` уточняют условия: «для премиум-пользователей», «для пустого массива», «при сетевой ошибке». Это помогает организовать тесты по принципу «функция → сценарий → конкретная проверка».

```ts
describe("calculateDiscount", () => {
  describe("for premium users", () => {
    it("returns 10% discount", () => {
      expect(calculateDiscount(100, true)).toBe(10);
    });
  });

  describe("for regular users", () => {
    it("returns no discount", () => {
      expect(calculateDiscount(100, false)).toBe(0);
    });
  });
});
```

### it / test — определение теста

`it` и `test` — полные алиасы. Первый аргумент — строковое описание того, что проверяется. Пишите его как утверждение: «возвращает сумму двух чисел», а не «проверяет функцию add». Если тест падает, это описание — первое, что вы увидите в логе.

```ts
it("returns sum of two numbers", () => {
  expect(add(2, 3)).toBe(5);
});

// test — алиас для it
test("returns sum of two numbers", () => {
  expect(add(2, 3)).toBe(5);
});
```

### expect — проверки

`expect` оборачивает значение и предоставляет matcher'ы для его проверки. `toBe` использует строгое равенство (`===`) — подходит для примитивов. `toEqual` сравнивает объекты и массивы по содержимому (глубокое равенство). `toBeTruthy`/`toBeFalsy` проверяют «truthiness» — полезно, когда конкретное значение не важно, важен лишь факт его наличия или отсутствия.

```ts
expect(value).toBe(42);           // Строгое равенство (===)
expect(value).toEqual({ a: 1 });  // Глубокое равенство
expect(value).toBeTruthy();       // truthy-значение
expect(value).toBeDefined();      // !== undefined
expect(value).toBeNull();         // === null
expect(value).toBeUndefined();    // === undefined
```

### Хуки жизненного цикла

Хуки позволяют вынести повторяющуюся подготовку и очистку. `beforeAll`/`afterAll` выполняются один раз на всю группу — идеально для дорогих операций вроде подключения к БД. `beforeEach`/`afterEach` — перед каждым тестом, что гарантирует изоляцию: каждый тест начинает с чистого состояния.

Правило: если тесты делят мутабельное состояние (объект репозитория, мок), используйте `beforeEach`. Если ресурс иммутабельный или дорогой — `beforeAll`.

```ts
describe("UserRepository", () => {
  let repo: UserRepository;

  beforeAll(async () => {
    // Выполняется один раз перед всеми тестами
    await initDatabase();
  });

  afterAll(async () => {
    // Выполняется один раз после всех тестов
    await closeDatabase();
  });

  beforeEach(() => {
    // Выполняется перед каждым тестом
    repo = new UserRepository();
  });

  afterEach(() => {
    // Выполняется после каждого теста
    vi.clearAllMocks();
  });

  it("creates user", () => {
    const user = repo.create({ name: "Alice" });
    expect(user.name).toBe("Alice");
  });
});
```

### Порядок выполнения

Понимание порядка важно для отладки: если `beforeEach` случайно мутирует состояние, которое проверяет `afterEach`, тесты будут непредсказуемо падать.

```
beforeAll
  beforeEach
    it("test 1")
  afterEach
  beforeEach
    it("test 2")
  afterEach
afterAll
```

---

## Matchers — проверки значений

Matchers — это методы `expect`, которые определяют, *как* сравнивать значение. Правильный выбор matcher'а делает тесты точнее и понятнее. `toBe` проверяет ссылочное равенство — для объектов это значит «это тот же самый объект в памяти». `toEqual` сравнивает содержимое — два разных объекта с одинаковыми полями пройдут проверку. Ошибка здесь — частый источник путаницы: `expect({a:1}).toBe({a:1})` упадёт, потому что это два разных объекта.

### Равенство

Для примитивов (`number`, `string`, `boolean`) всегда используйте `toBe` — это быстрее и понятнее. Для объектов и массивов — `toEqual`, если важно содержимое, или `toBe`, если важна именно та ссылка. `not` инвертирует проверку.

```ts
// Примитивы
expect(42).toBe(42);
expect("hello").toBe("hello");
expect(true).toBe(true);

// null, undefined
expect(null).toBeNull();
expect(undefined).toBeUndefined();
expect(42).not.toBeUndefined();

// Объекты и массивы (глубокое сравнение)
expect({ a: 1, b: 2 }).toEqual({ a: 1, b: 2 });
expect([1, 2, 3]).toEqual([1, 2, 3]);

// Ссылочное равенство
const obj = { a: 1 };
expect(obj).toBe(obj);
expect(obj).not.toBe({ a: 1 }); // Другой объект — другая ссылка
```

### Числа

Для сравнения чисел с плавающей точкой используйте `toBeCloseTo` вместо `toBe` — из-за особенностей IEEE 754 выражение `0.1 + 0.2 === 0.3` возвращает `false`. Второй аргумент `toBeCloseTo` — количество значимых десятичных разрядов.

```ts
expect(42).toBeGreaterThan(40);
expect(42).toBeGreaterThanOrEqual(42);
expect(42).toBeLessThan(50);
expect(42).toBeLessThanOrEqual(42);
expect(0.1 + 0.2).toBeCloseTo(0.3, 5); // Для float-сравнений
expect(NaN).toBeNaN();
```

### Строки

`toContain` проверяет наличие подстроки, `toMatch` — регулярное выражение. Для проверки длины строки используйте `toHaveLength`.

```ts
expect("hello world").toContain("world");
expect("hello world").toMatch(/hello/);
expect("hello").toHaveLength(5);
expect("Hello World").toMatch(/^hello$/i);
```

### Массивы

`toContain` проверяет наличие элемента по `===`. `toContainEqual` — по глубокому равенству. `expect.arrayContaining` проверяет, что массив содержит *хотя бы* указанные элементы, игнорируя остальные — полезно, когда вам важен только фрагмент результата.

```ts
expect([1, 2, 3]).toContain(2);
expect([1, 2, 3]).toContainEqual({ id: 1 });
expect([1, 2, 3]).toHaveLength(3);
expect([1, 2, 3]).toEqual(expect.arrayContaining([1, 2]));
```

### Объекты

`toHaveProperty` проверяет наличие ключа, в том числе вложенного через точечную нотацию. `expect.objectContaining` — «частичное» сравнение: объект проходит, если содержит указанные поля, даже если есть дополнительные. Это полезно для проверки API-ответов, где сервер может вернуть больше полей, чем вы ожидаете.

```ts
expect({ a: 1, b: 2, c: 3 }).toHaveProperty("a");
expect({ a: 1, b: 2, c: 3 }).toHaveProperty("a", 1);
expect({ a: { b: 1 } }).toHaveProperty("a.b", 1);

expect({ a: 1, b: 2 }).toEqual(
  expect.objectContaining({ a: 1 }) // Может содержать дополнительные поля
);
```

### Boolean и типы

`toBeTruthy`/`toBeFalsy` проверяют «truthiness» — любое значение, которое приводит к `true`/`false` в булевом контексте. `0`, `""`, `null`, `undefined`, `NaN` — falsy. Всё остальное — truthy. `toBeInstanceOf` проверяет прототипную цепочку.

```ts
expect(true).toBeTruthy();
expect(false).toBeFalsy();
expect(1).toBeTruthy();
expect(0).toBeFalsy();
expect("").toBeFalsy();
expect(null).toBeFalsy();
expect(undefined).toBeFalsy();

expect("hello").toBeInstanceOf(String);
expect([1, 2]).toBeInstanceOf(Array);
expect(new Date()).toBeInstanceOf(Date);
```

### Исключения

`toThrow` принимает строку, RegExp или класс ошибки. Для асинхронных функций используйте `rejects`/`resolves` в сочетании с `await` — без `await` тест пройдёт, даже если промис отклонится.

```ts
expect(() => {
  throw new Error("boom");
}).toThrow("boom");

expect(() => {
  throw new TypeError("wrong type");
}).toThrow(TypeError);

expect(() => {
  throw new Error("boom");
}).toThrow(/boom/);

// Асинхронные исключения
await expect(asyncFunction()).rejects.toThrow("error message");
await expect(asyncFunction()).resolves.toBe(42);
```

### DOM (с @testing-library/jest-dom)

Эти matcher'ы добавляются библиотекой `@testing-library/jest-dom` и предназначены для проверки DOM-элементов. Они дают более читаемые сообщения об ошибках по сравнению с ручной проверкой атрибутов и стилей. Подключаются через setup-файл.

```ts
expect(element).toBeInTheDocument();
expect(element).toBeVisible();
expect(element).toBeEmpty();
expect(element).toBeDisabled();
expect(element).toBeEnabled();
expect(element).toBeChecked();
expect(element).toBeRequired();
expect(element).toBeValid();
expect(element).toBeInvalid();

expect(element).toHaveTextContent("Hello");
expect(element).toHaveTextContent(/Hello/);
expect(element).toHaveAttribute("href", "/about");
expect(element).toHaveClass("active");
expect(element).toHaveStyle("color: red");
expect(element).toHaveFocus();
expect(element).toHaveValue("hello");
expect(element).toHaveFormValues({ name: "Alice" });

expect(element).toHaveAccessibleName("Search");
expect(element).toHaveAccessibleDescription("Find products");
expect(element).toHaveRole("button");
```

---

## Асинхронные тесты

Асинхронные тесты — любая логика, связанная с промисами: HTTP-запросы, таймеры, чтение файлов. Главная ошибка — забыть `await` или `return`. Без этого Vitest считает тест завершённым сразу после вызова асинхронной функции, и тест проходит, даже если промис отклонится. Всегда используйте `async/await` — это самый надёжный и читаемый способ.

### async/await

Предпочтительный способ. Функция помечается `async`, промис ожидается через `await`. Если промис отклонится — тест упадёт с понятной ошибкой.

```ts
it("fetches user data", async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe("Alice");
});
```

### Промисы

Альтернатива `async/await` — вернуть промис из теста. Vitest дождётся его разрешения. Работает, но менее читаемо, чем `async/await`.

```ts
it("fetches user data", () => {
  return fetchUser(1).then((user) => {
    expect(user.name).toBe("Alice");
  });
});
```

### resolves / rejects

`resolves` и `rejects` — matcher'ы для промисов. Они «разворачивают» промис и позволяют применять к результату обычные matcher'ы. `rejects` проверяет, что промис отклонился, и позволяет проверить текст или тип ошибки.

```ts
it("resolves with user data", async () => {
  await expect(fetchUser(1)).resolves.toEqual({
    id: 1,
    name: "Alice",
  });
});

it("rejects with error for unknown user", async () => {
  await expect(fetchUser(999)).rejects.toThrow("User not found");
});
```

### done-колбэк (редко используется)

`done` — legacy-подход для колбэк-стиля. Vitest не завершит тест, пока вы не вызовете `done()`. Используйте только для колбэк-API (WebSocket, EventEmitter). Для промисов всегда предпочитайте `async/await`.

```ts
it("subscribes to events", (done) => {
  const callback = (data: string) => {
    expect(data).toBe("hello");
    done();
  };
  subscribe(callback);
});
```

> 💡 **Рекомендация:** Используйте async/await вместо done-колбэка. Это чище и понятнее.

### Таймауты

Если тест выполняется дольше 5 секунд (дефолт), он упадёт по таймауту. Увеличьте таймаут для конкретного теста, если он legitimately медленный (например, интеграционный тест с реальной БД). Но если тест тормозит без причины — это сигнал, что нужно использовать моки или фейковые таймеры.

```ts
it("takes long to process", async () => {
  const result = await slowOperation();
  expect(result).toBe("done");
}, 10000); // Таймаут 10 секунд для этого теста
```

---

## Мокирование: vi.fn, vi.spyOn, vi.mock

Мокирование заменяет реальные зависимости тестового кода контролируемыми заглушками. Без моков тесты превращаются в интеграционные: они зависят от внешних API, БД, таймеров — становятся медленными и хрупкими.

Три уровня мокирования в Vitest:
- `vi.fn()` — создаёт функцию-пустышку с нуля. Используйте, когда передаёте колбэк или подменяете зависимость целиком.
- `vi.spyOn()` — оборачивает существующий метод объекта, сохраняя оригинальную реализацию (по умолчанию). Используйте, когда нужно проверить, что метод был вызван, но не менять его поведение.
- `vi.mock()` — подменяет целый модуль. Используйте, когда зависимость — внешний модуль (API-клиент, роутер, хранилище).

### vi.fn() — создание mock-функции

`vi.fn()` создаёт функцию, которая запоминает все свои вызовы. После выполнения кода можно проверить, сколько раз её вызывали и с какими аргументами. Это основа для проверки взаимодействий — не «что вернул код», а «что код сделал».

```ts
const mockFn = vi.fn();

mockFn("hello");
mockFn("world");

expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith("hello");
expect(mockFn).toHaveBeenLastCalledWith("world");
```

### vi.fn() с реализацией

По умолчанию mock-функция возвращает `undefined`. Чтобы она вела себя реалистично, задайте реализацию: `mockReturnValue` для статического значения, `mockResolvedValue` для промиса, `mockReturnValueOnce` для последовательности значений. Последовательные значения полезны, когда нужно протестировать, например, «первый запрос успешен, второй — ошибка».

```ts
// Mock с возвращаемым значением
const mockAdd = vi.fn((a: number, b: number) => a + b);
expect(mockAdd(2, 3)).toBe(5);

// Mock с последовательными возвращаемыми значениями
const mockFn = vi.fn()
  .mockReturnValueOnce("first")
  .mockReturnValueOnce("second")
  .mockReturnValue("default");

expect(mockFn()).toBe("first");
expect(mockFn()).toBe("second");
expect(mockFn()).toBe("default");
expect(mockFn()).toBe("default");

// Mock с async-реализацией
const mockFetch = vi.fn().mockResolvedValue({ data: [1, 2, 3] });
const result = await mockFetch();
expect(result.data).toEqual([1, 2, 3]);

// Mock с отклонением
const mockFetch = vi.fn().mockRejectedValue(new Error("Network error"));
await expect(mockFetch()).rejects.toThrow("Network error");
```

### vi.spyOn() — шпионаж за методом

`vi.spyOn` оборачивает метод объекта, не меняя его поведение (если не указать `mockReturnValue`). Это менее инвазивно, чем `vi.mock`: оригинальный код остаётся на месте, вы лишь наблюдаете за вызовами. Хорошо подходит для проверки побочных эффектов — вызовов `console.log`, `window.open`, и т.д.

Важно: после теста вызывайте `mockRestore()`, иначе шпон останется на месте и может сломать другие тесты.

```ts
const consoleSpy = vi.spyOn(console, "log");

doSomething();

expect(consoleSpy).toHaveBeenCalledWith("Processing...");

consoleSpy.mockRestore(); // Восстановить оригинальный метод
```

### vi.spyOn() для переопределения

`spyOn` можно комбинировать с `mockReturnValue` — тогда метод и шпионится, и подменяется. Это полезно, когда нужно изменить поведение одного метода объекта, не трогая остальные.

```ts
const user = {
  getName: () => "Alice",
};

const spy = vi.spyOn(user, "getName").mockReturnValue("Bob");

expect(user.getName()).toBe("Bob");

spy.mockRestore();
expect(user.getName()).toBe("Alice");
```

### vi.mock() — мокирование модулей

`vi.mock()` подменяет весь модуль. Это необходимо, когда тестируемый код импортирует зависимость — например, API-клиент. Без мока тест будет делать реальные HTTP-запросы. `vi.mock` вызывается *до* импортов (Vitest автоматически «поднимает» его), поэтому переменные из теста недоступны внутри factory-функции — для этого используйте `vi.hoisted()` (см. ниже).

```ts
// Полное мокирование модуля
vi.mock("./api", () => ({
  fetchUsers: vi.fn().mockResolvedValue([{ id: 1, name: "Alice" }]),
  deleteUser: vi.fn().mockResolvedValue(true),
}));

// Использование в тесте
import { fetchUsers } from "./api";

it("loads users", async () => {
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

### vi.mock() с factory-функцией (частичный mock)

Часто нужно подменить только часть модуля, оставив остальное нетронутым. Для этого factory-функция принимает `importOriginal` — вызовите его, чтобы получить реальную реализацию, и перезапишите только нужные экспорты.

```ts
vi.mock("./api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("./api")>();
  return {
    ...actual,
    fetchUsers: vi.fn().mockResolvedValue([{ id: 1, name: "Mocked" }]),
    // deleteUser остаётся оригинальным
  };
});
```

### Автоматическое мокирование

Если один и тот же модуль нужно мокать во всех тестах проекта, вынесите мок в setup-файл. Это избавит от дублирования `vi.mock` в каждом тестовом файле.

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    setupFiles: ["./src/test/setup.ts"],
  },
});

// src/test/setup.ts — моки по умолчанию
vi.mock("./api", () => ({
  fetchUsers: vi.fn().mockResolvedValue([]),
}));
```

### vi.hoisted() — переменные для vi.mock

`vi.mock` вызывается до импортов, поэтому обычные `const`/`let` из файла недоступны внутри factory-функции. `vi.hoisted()` «поднимает» значения наверх, делая их доступными в factory. Это нужно, когда вы хотите настроить мок в одном месте, а использовать — в другом.

```ts
// vi.mock вызывается до импортов, поэтому обычные переменные недоступны
// vi.hoisted() решает эту проблему

const { mockFetchUsers, mockDeleteUser } = vi.hoisted(() => ({
  mockFetchUsers: vi.fn(),
  mockDeleteUser: vi.fn(),
}));

vi.mock("./api", () => ({
  fetchUsers: mockFetchUsers,
  deleteUser: mockDeleteUser,
}));

it("fetches users", async () => {
  mockFetchUsers.mockResolvedValue([{ id: 1 }]);
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1 }]);
});
```

### Очистка моков

Моки сохраняют историю вызовов между тестами. Если не очищать, тест №2 увидит вызовы из теста №1. Настраивайте очистку в `afterEach` глобально — в setup-файле.

Разница между методами: `clearAllMocks` — очищает историю, но сохраняет реализацию; `restoreAllMocks` — убирает моки и восстанавливает оригиналы (важно для `spyOn`).

```ts
afterEach(() => {
  vi.clearAllMocks();     // Очищает историю вызовов всех моков
  vi.restoreAllMocks();   // Восстанавливает оригинальные реализации
});

// Или для конкретного мока:
mockFn.mockClear();       // Очищает историю вызовов
mockFn.mockReset();       // Очищает + убирает реализацию
mockFn.mockRestore();     // Reset + восстанавливает оригинал (только для spyOn)
```

### Разница между clear, reset, restore

| Метод | История вызовов | Реализация | Оригинал (spyOn) |
|---|---|---|---|
| `mockClear()` | Очищает | Сохраняет | Сохраняет |
| `mockReset()` | Очищает | Убирает (→ undefined) | Сохраняет |
| `mockRestore()` | Очищает | Убирает | Восстанавливает |

---

## Фейковые таймеры

### Зачем нужны

Тесты не должны ждать реального времени. Фейковые таймеры позволяют контролировать `setTimeout`, `setInterval`, `Date` и другие временные API.

### Базовое использование

```ts
import { vi, it, expect } from "vitest";

it("shows notification after delay", () => {
  vi.useFakeTimers();

  const callback = vi.fn();
  setTimeout(callback, 5000);

  expect(callback).not.toHaveBeenCalled();

  vi.advanceTimersByTime(5000);

  expect(callback).toHaveBeenCalledTimes(1);

  vi.useRealTimers();
});
```

### advanceTimersByTime

```ts
it("polls every 10 seconds", () => {
  vi.useFakeTimers();

  const poll = vi.fn();
  setInterval(poll, 10000);

  vi.advanceTimersByTime(30000); // Продвигаем на 30 секунд

  expect(poll).toHaveBeenCalledTimes(3);

  vi.useRealTimers();
});
```

### runAllTimers

```ts
it("executes all pending timers", () => {
  vi.useFakeTimers();

  setTimeout(() => console.log("1"), 100);
  setTimeout(() => console.log("2"), 500);
  setTimeout(() => console.log("3"), 1000);

  vi.runAllTimers(); // Выполняет все таймеры

  vi.useRealTimers();
});
```

### runOnlyPendingTimers

```ts
it("executes only pending timers", () => {
  vi.useFakeTimers();

  setTimeout(() => {
    console.log("1");
    setTimeout(() => console.log("2"), 100); // Рекурсивный таймер
  }, 100);

  vi.runOnlyPendingTimers(); // Выполнит только первый, не рекурсивный

  vi.useRealTimers();
});
```

### setSystemTime

```ts
it("formats date correctly", () => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date("2026-01-15T12:00:00Z"));

  expect(formatRelativeDate(new Date())).toBe("today");

  vi.useRealTimers();
});
```

### Использование с async/await

```ts
it("shows loading state", async () => {
  vi.useFakeTimers();

  const { result } = renderHook(() => useDelayedData());

  expect(result.current.isLoading).toBe(true);

  await vi.advanceTimersByTimeAsync(3000);

  expect(result.current.isLoading).toBe(false);
  expect(result.current.data).toEqual([1, 2, 3]);

  vi.useRealTimers();
});
```

---

## Snapshot-тестирование

Snapshot-тесты фиксируют текущее состояние вывода (обычно DOM) и сравнивают будущие запуски с ним. Это быстрый способ гарантировать, что unintended-изменения не проскочат. Но snapshot — это «слепая» проверка: он не понимает, правильное ли изменение произошло. Если вы намеренно поменяли CSS-класс, snapshot упадёт и потребует обновления — и тут легко по привычке нажать `-u`, пропустив баг.

**Когда snapshot оправдан:** стабильные, редко меняющиеся элементы (иконки, базовые UI-примитивы). **Когда вреден:** компоненты с бизнес-логикой, динамическими данными, частыми итерациями дизайна.

### Базовые snapshot-тесты

Vitest рендерит компонент, сериализует DOM в строку и сохраняет в файл `__snapshots__/*.snap`. При следующем запуске сравнивает текущий рендер с сохранённым.

```ts
it("renders button correctly", () => {
  const { container } = render(<Button variant="primary">Click me</Button>);
  expect(container).toMatchSnapshot();
});
```

Vitest создаёт файл `__snapshots__/button.test.ts.snap` с сериализованным DOM:

```
// Vitest Snapshot 1, https://vitest.dev/guide/snapshot.html

exports[`renders button correctly 1`] = `
<div>
  <button
    class="btn btn--primary"
  >
    Click me
  </button>
</div>
`;
```

### Inline snapshots

Inline-снапшоты хранятся прямо в тестовом файле — удобнее для ревью, но загромождают код. Используйте, когда снапшот короткий и стабибильный.

```ts
it("renders button with inline snapshot", () => {
  const { container } = render(<Button>Click</Button>);
  expect(container.innerHTML).toMatchInlineSnapshot(`
    "<button class="btn btn--primary">Click</button>"
  `);
});
```

### Когда использовать snapshot

| Сценарий | Snapshot? |
|---|---|
| Иконки, SVG | ✅ Да — стабильные, редко меняются |
| Базовые UI-компоненты (Button, Input) | ✅ Да — если не меняются часто |
| Сложные компоненты с бизнес-логикой | ❌ Нет — snapshot будет ломаться при каждом рефакторе |
| Компоненты с динамическими данными | ❌ Нет — snapshot зависит от данных |

### Обновление snapshot

Обновляйте снапшоты только после того, как проверили diff. Слепое обновление (`-u`) — частая причина пропуска багов в production.

```bash
vitest --update    # Обновить все snapshots
vitest -u          # Короткая форма
```

> ⚠️ **Внимание:** Обновляйте snapshot только после проверки diff'а. Слепое обновление скрывает нежелательные изменения.

---

## Параметризованные тесты

Параметризованные тесты позволяют запустить одну и ту же проверку с разными входными данными без дублирования кода. Это критически важно для edge cases: пустая строка, `null`, `undefined`, граничные значения. Без `each` вам пришлось бы писать 5 одинаковых `it`-блоков — с `each` вы пишете один и получаете 5 отдельных тестов с понятными именами.

### each — тесты с разными данными

`it.each` принимает массив массивов. Каждый вложенный массив — набор аргументов для одного прогона. Плейсхолдеры `%i` (число), `%s` (строка) подставляются в название теста, чтобы в логе было видно, какой именно набор упал.

```ts
it.each([
  [1, 1, 2],
  [2, 3, 5],
  [10, 20, 30],
  [-1, 1, 0],
])("add(%i, %i) returns %i", (a, b, expected) => {
  expect(add(a, b)).toBe(expected);
});
```

### each с объектами

Когда параметров больше двух-трёх, массивы становятся нечитаемыми. Объекты решают эту проблему — каждое поле именовано, и в плейсхолдерах можно использовать `$имяПоля`.

```ts
it.each([
  { input: "hello", expected: "Hello" },
  { input: "WORLD", expected: "World" },
  { input: "fOo", expected: "Foo" },
])("capitalize($input) returns $expected", ({ input, expected }) => {
  expect(capitalize(input)).toBe(expected);
});
```

### each для edge cases

Edge cases — главная ценность `each`. Функция может работать на «hello world», но падать на пустой строке или строке с пробелами. Перечислите все известные граничные случаи в одном `each` — и каждый станет отдельным тестом с понятным описанием.

```ts
it.each([
  { input: "", expected: 0, description: "empty string" },
  { input: "a", expected: 1, description: "single char" },
  { input: "hello world", expected: 2, description: "two words" },
  { input: "  spaces  ", expected: 1, description: "with spaces" },
  { input: "a,b,c", expected: 3, description: "comma separated" },
])("countWords: $description", ({ input, expected }) => {
  expect(countWords(input)).toBe(expected);
});
```

### Тестирование типов

`each` удобен для проверки «truthiness» и типобезопасности — когда нужно убедиться, что функция корректно обрабатывает все возможные типы входных данных.

```ts
it.each([
  { value: null, expected: false },
  { value: undefined, expected: false },
  { value: "", expected: false },
  { value: 0, expected: false },
  { value: "hello", expected: true },
  { value: 42, expected: true },
  { value: [], expected: true },
  { value: {}, expected: true },
])("isTruthy($value) returns $expected", ({ value, expected }) => {
  expect(isTruthy(value)).toBe(expected);
});
```

---

## Тестирование хуков

Хуки нельзя вызывать вне React-компонента — они используют внутренний механизм React (Fiber). `renderHook` из `@testing-library/react` оборачивает хук в минимальный компонент и даёт доступ к его возвращаемому значению через `result.current`. Любое обновление хука (setState, эффект) должно происходить внутри `act()` — иначе React warning и непредсказуемое поведение.

### renderHook

`renderHook` принимает функцию, вызывающую хук, и возвращает `result` с текущим значением. Для хуков без провайдеров и асинхронности это самый простой случай.

```ts
import { renderHook, act } from "@testing-library/react";

it("useCounter increments", () => {
  const { result } = renderHook(() => useCounter());

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

### Хуки с провайдерами

Многие хуки зависят от контекста (тема, локаль, роутер). `renderHook` принимает опцию `wrapper` — компонент-обёртку, который оборачивает тестовый хук нужными провайдерами. Это аналог `<ThemeProvider>` в реальном приложении.

```ts
import { renderHook } from "@testing-library/react";
import { ThemeProvider } from "./ThemeProvider";

it("useTheme returns current theme", () => {
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <ThemeProvider initialTheme="dark">{children}</ThemeProvider>
  );

  const { result } = renderHook(() => useTheme(), { wrapper });

  expect(result.current.theme).toBe("dark");
});
```

### Тестирование асинхронных хуков

Хуки, которые делают запросы, проходят через несколько состояний: loading → success/error. `waitFor` ждёт, пока условие выполнится (с таймаутом), вместо того чтобы гадать, сколько миллисекунд ждать. Внутри теста мокаем API через MSW (см. «Лучшие практики»).

```ts
it("useFetch loads data", async () => {
  server.use(
    http.get("/api/user/1", () => HttpResponse.json({ id: 1, name: "Alice" }))
  );

  const { result } = renderHook(() => useFetch("/api/user/1"));

  expect(result.current.isLoading).toBe(true);

  await waitFor(() => {
    expect(result.current.isLoading).toBe(false);
  });

  expect(result.current.data).toEqual({ id: 1, name: "Alice" });
});
```

### Тестирование хуков с эффектами

Хуки с побочными эффектами (запись в localStorage, подписки) проверяются через `act` для вызова и прямую проверку хранилища/мока для эффекта.

```ts
it("useLocalStorage persists value", () => {
  const { result } = renderHook(() => useLocalStorage("key", "default"));

  expect(result.current[0]).toBe("default");

  act(() => {
    result.current[1]("new value");
  });

  expect(result.current[0]).toBe("new value");
  expect(localStorage.getItem("key")).toBe('"new value"');
});
```

---

## Покрытие кода

Покрытие кода показывает, *какие строки* выполняются во время тестов. Это метрика «слепых зон» — она не гарантирует, что тесты хорошие, но показывает, какие части кода вообще не тронуты. High coverage ≠ багов нет: можно покрыть 100% строк, но не проверить ни одного edge case.

Два провайдера: `v8` (нативный, быстрый, точный для ESM) и `istanbul` (медленнее, но работает в большем числе окружений). Для Vite-проектов используйте `v8`.

### Установка coverage-провайдера

```bash
npm install -D @vitest/coverage-v8
```

### Конфигурация

`thresholds` — пороги покрытия. Если покрытие упадёт ниже, CI отклонит PR. Это мощный инструмент, но устанавливайте реалистичные пороги: 100% — нереалистично и ведёт к бессмысленным тестам «для галочки». 80% statements — разумный компромисс для большинства проектов.

`include`/`exclude` определяют, какие файлы считать. Исключайте тесты, типы (`.d.ts`), сторис — они не несут бизнес-логики.

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html", "lcov"],
      reportsDirectory: "./coverage",

      include: ["src/**/*.{ts,tsx}"],
      exclude: [
        "src/test/**",
        "src/**/*.d.ts",
        "src/**/*.stories.tsx",
        "src/main.tsx",
        "src/vite-env.d.ts",
      ],

      thresholds: {
        "**/*": {
          statements: 80,
          branches: 70,
          functions: 85,
          lines: 80,
        },
        "src/utils/**": {
          statements: 90,
          branches: 80,
          functions: 90,
          lines: 90,
        },
      },
    },
  },
});
```

### Запуск с покрытием

```bash
vitest run --coverage
```

### Анализ отчёта

Четыре метрики: **Statements** — процент выполненных выражений; **Branches** — процент пройденных веток (`if`/`else`, `switch`); **Functions** — процент вызванных функций; **Lines** — процент выполненных строк. Branches обычно самая низкая метрика — каждая `if`-ветка считается отдельно.

```
% Coverage report from v8
-----------|---------|----------|---------|---------|
File       | % Stmts | % Branch | % Funcs | % Lines |
-----------|---------|----------|---------|---------|
All files  |   82.60 |    71.42 |   86.36 |   82.60 |
 utils/    |   95.00 |    90.00 |  100.00 |   95.00 |
  format   |  100.00 |   100.00 |  100.00 |  100.00 |
  validate |   90.00 |    80.00 |  100.00 |   90.00 |
 hooks/    |   75.00 |    60.00 |   80.00 |   75.00 |
  useAuth  |   80.00 |    66.66 |   85.71 |   80.00 |
  useCart  |   70.00 |    53.33 |   75.00 |   70.00 |
-----------|---------|----------|---------|---------|
```

---

## Конфигурация для React, Next.js, Vue и Nuxt

Каждый фреймворк требует специфичной настройки тестового окружения: эмуляция DOM, трансформация компонентов, мокирование фреймворк-специфичных API. Без `jsdom` тесты не смогут рендерить компоненты — в Node.js нет `document`. Без соответствующего Vite-плагина Vitest не поймёт `.jsx`, `.tsx` или `.vue` файлы.

### React + Vite

Минимальная рабочая конфигурация: `jsdom` для DOM, `react`-плагин для JSX, setup-файл для cleanup и matcher'ов. Эта конфигурация покрывает 90% React-проектов на Vite.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    css: true,
  },
});
```

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach } from "vitest";

afterEach(() => {
  cleanup();
});
```

### Next.js

Next.js добавляет сложности: серверные компоненты, серверные действия, собственный роутер. Для тестирования клиентских компонентов конфигурация похожа на обычный React, но нужно настроить алиасы путей (`@` → `./src`) явно, потому что Next.js не всегда пробрасывает их в Vitest.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import { resolve } from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    include: ["src/**/*.{test,spec}.{ts,tsx}"],
    alias: {
      "@": resolve(__dirname, "./src"),
    },
  },
});
```

### Мокирование Next.js API

Next.js-специфичные API (`useRouter`, `usePathname`, `Link`) не существуют вне Next.js-рантайма. Их нужно мокать полностью, иначе тесты упадут с «module not found». Для `Link` достаточно вернуть простой `<a>` — тестировать навигацию Next.js не нужно, это задача фреймворка.

```ts
// src/test/setup.ts
vi.mock("next/navigation", () => ({
  useRouter: vi.fn(() => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn(),
  })),
  usePathname: vi.fn(() => "/"),
  useSearchParams: vi.fn(() => new URLSearchParams()),
}));

vi.mock("next/link", () => ({
  default: ({ children, href }: { children: React.ReactNode; href: string }) =>
    `<a href="${href}">${children}</a>`,
}));
```

### Vue + Vite

Для Vue используется `@vitejs/plugin-vue`. Он компилирует однофайловые компоненты (`.vue`) для Vitest. Окружение `jsdom` или `happy-dom` эмулирует браузерный DOM.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: "jsdom",
    globals: true,
  },
});
```

```ts
// src/test/setup.ts
import { cleanup } from "@vue/test-utils";
import { afterEach, vi } from "vitest";

afterEach(() => {
  cleanup();
  vi.clearAllMocks();
});
```

### Nuxt 3

Nuxt 3 имеет официальный модуль `@nuxt/test-utils/module`, который автоматически настраивает Vitest: auto-imports, `.vue` файлы, runtime config, серверные маршруты.

```ts
// vitest.config.ts
import { defineVitestConfig } from "@nuxt/test-utils/config";

export default defineVitestConfig({
  test: {
    environment: "nuxt",
    globals: true,
    environmentOptions: {
      nuxt: {
        domEnvironment: "happy-dom",
      },
    },
  },
});
```

> 💡 `environment: "nuxt"` — специальное окружение от `@nuxt/test-utils`. Оно эмулирует Nuxt-рантайм и отличается от обычного `jsdom` для Vue.

---

## Vitest vs Jest

| Характеристика | Vitest | Jest |
|---|---|---|
| **Совместимость API** | Совместим с Jest | Оригинальный API |
| **ESM** | Нативная поддержка | Экспериментальная |
| **TypeScript** | Из коробки | Требует ts-jest |
| **Vite-проекты** | Нативная интеграция | Отдельная настройка |
| **Watch mode** | HMR-based | File watcher |
| **Coverage** | v8, istanbul | v8, babel |
| **Snapshot** | Совместимы | Оригинальные |
| **Workspace** | Встроенный | Нет |
| **Browser mode** | Встроенный (Playwright) | Нет (jest-environment-jsdom) |
| **Миграция с Jest** | Минимальная | — |

### Когда выбрать Jest

- Проект на Webpack (без Vite)
- Legacy-проект с уже настроенным Jest
- Нужна специфичная Jest-экосистема (jest-circus, jest-environment-node)

### Когда выбрать Vitest

- Проект на Vite
- Новый проект
- Нужна нативная ESM-поддержка
- Нужен TypeScript без дополнительной настройки
- Нужен browser mode

---

## Лучшие практики

Эти практики сформированы на опыте реальных проектов. Они не догма, но отклонение от них должно быть осознанным.

### 1. Используйте `globals: true` для чистого API

Без `globals: true` каждый тестовый файл начинается с `import { describe, it, expect } from "vitest"`. Это загромождает код и не несёт ценности — эти импорты нужны везде. `globals: true` регистрирует их глобально, как в Jest. Trade-off: вы теряете явность импортов, но выигрываете в читаемости тестов.

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
  },
});

// Теперь не нужно импортировать describe, it, expect
// describe("...", () => { it("...", () => { expect(...) }) })
```

### 2. Структурируйте тестовые файлы рядом с исходными

Колокация теста и кода — стандарт индустрии. Тест рядом с файлом легче найти, легче поддерживать в актуальном состоянии при рефакторе, легче удалить вместе с кодом. Альтернатива (`__tests__/` в корне) работает для маленьких проектов, но масштабируется хуже.

```
src/
  utils/
    format.ts
    format.test.ts       ← Тест рядом с кодом
  hooks/
    useAuth.ts
    useAuth.test.ts
  components/
    Button.tsx
    Button.test.tsx
```

### 3. Выносите общие утилиты в setup-файлы

Если каждый тестовый файл начинается с одинаковых `cleanup()`, `clearAllMocks()`, `restoreAllMocks()` — вынесите это в setup. Это DRY-принцип для тестов. Очистка моков в `afterEach` критична: без неё тест №2 может сломаться из-за состояния, оставленного тестом №1.

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach, vi } from "vitest";

afterEach(() => {
  cleanup();
  vi.clearAllMocks();
  vi.restoreAllMocks();
});
```

### 4. Используйте MSW для мокирования API

MSW (Mock Service Worker) перехватывает HTTP-запросы на уровне сети — тот же код, что работает в браузере, работает и в тестах. Это лучше, чем мокать `fetch` напрямую: вы тестируете реальный код запроса, а не его обёртку. `onUnhandledRequest: "error"` гарантирует, что неожиданный запрос не пройдёт незамеченным.

```ts
// src/test/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./mocks/handlers";

export const server = setupServer(...handlers);
```

```ts
// src/test/setup.ts
import { server } from "./server";
import { beforeAll, afterEach, afterAll } from "vitest";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### 5. Фабрики для тестовых данных

Хардкод тестовых данных ведёт к дублированию и хрупкости: изменили одно поле в 20 тестах — получили 20 правок. Фабрики централизуют создание тестовых объектов. `Partial<T>` позволяет переопределять только нужные поля, остальное — дефолты.

```ts
// src/test/factories.ts
export const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  ...overrides,
});

export const createProduct = (overrides?: Partial<Product>): Product => ({
  id: crypto.randomUUID(),
  name: "Test Product",
  price: 29.99,
  category: "electronics",
  ...overrides,
});
```

### 6. Тестируйте edge cases параметризованно

Функция работает на happy path — это минимум. Реальные баги живут в edge cases: пустые строки, `null`, пробелы, спецсимволы. `it.each` позволяет перечислить их все в одном блоке, и каждый кейс становится отдельным тестом с понятным описанием.

```ts
it.each([
  { input: "", expected: false, description: "empty string" },
  { input: "   ", expected: false, description: "whitespace only" },
  { input: "a@b.c", expected: true, description: "valid email" },
  { input: "invalid", expected: false, description: "no @" },
  { input: "@b.c", expected: false, description: "no local part" },
  { input: "a@", expected: false, description: "no domain" },
])("isValidEmail($input) → $expected ($description)", ({ input, expected }) => {
  expect(isValidEmail(input)).toBe(expected);
});
```
