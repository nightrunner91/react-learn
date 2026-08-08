# Мокирование и изоляция тестов

## Содержание

1. [Зачем мокировать](#зачем-мокировать)
2. [Stub, Spy, Mock — в чём разница](#stub-spy-mock--в-чём-разница)
3. [vi.fn() — создание mock-функций](#vifn--создание-mock-функций)
4. [vi.spyOn() — шпионаж за методами](#vispyon--шпионаж-за-методами)
5. [vi.mock() — мокирование модулей](#vimock--мокирование-модулей)
6. [vi.hoisted() — переменные для vi.mock](#vihoisted--переменные-для-vimock)
7. [MSW — мокирование API на сетевом уровне](#msw--мокирование-api-на-сетевом-уровне)
8. [MSW — настройка и структура](#msw--настройка-и-структура)
9. [Переопределение handlers в тестах](#переопределение-handlers-в-тестах)
10. [Мокирование fetch](#мокирование-fetch)
11. [Мокирование localStorage](#мокирование-localstorage)
12. [Мокирование таймеров](#мокирование-таймеров)
13. [Мокирование модулей браузера](#мокирование-модулей-браузера)
14. [Фабрики тестовых данных](#фабрики-тестовых-данных)
15. [Жизненный цикл моков: clear, reset, restore](#жизненный-цикл-моков-clear-reset-restore)
16. [Лучшие практики](#лучшие-практики)
17. [Антипаттерны](#антипаттерны)

---

## Зачем мокировать

Тесты должны быть **быстрыми**, **стабильными** и **изолированными**. Реальные зависимости нарушают эти принципы:

| Зависимость | Проблема | Решение |
|---|---|---|
| API (fetch, axios) | Медленные, нестабильные, требуют сеть | MSW или vi.mock |
| База данных | Требует реальную БД, медленная | vi.mock репозитория |
| localStorage | Сохраняется между тестами | Очистка или vi.spyOn |
| setTimeout/setInterval | Тесты ждут реальное время | vi.useFakeTimers() |
| Date.now() | Зависит от текущего времени | vi.setSystemTime() |
| console.log | Засоряет вывод тестов | vi.spyOn(console, "log") |
| Сторонние библиотеки | Могут менять поведение | vi.mock модуля |

### Что мокировать, что не мокировать

| Мокировать | Не мокировать |
|---|---|
| Внешние API (fetch, axios) | Утилиты вашего проекта |
| Базу данных | React-хуки (useState, useEffect) |
| Файловую систему | Компоненты (если не нужно) |
| Сетевые запросы | Бизнес-логику |
| Таймеры (setTimeout) | Чистые функции |

> 💡 **Правило:** Мокируйте только **внешние зависимости** — то, что выходит за рамки вашего кода. Не мокируйте внутренние модули, если они не являются внешними зависимостями.

---

## Stub, Spy, Mock — в чём разница

Эти термины пришли из теории тестирования и описывают разные стратегии замены зависимостей. Понимание разницы помогает выбрать правильный инструмент: иногда достаточно простой заглушки, а иногда нужен полноценный mock с отслеживанием вызовов. На практике Vitest стирает границы — `vi.fn()` и `vi.spyOn()` покрывают все три случая.

### Stub — заглушка

Stub — самая простая форма замены. Он нужен, когда вам важно только **вернуть определённое значение** из зависимости, а информация о том, как она вызывалась, не важна. Например, если компоненту нужен `fetchUser()`, который возвращает объект пользователя, и вы хотите проверить рендер, а не сам факт вызова.

```ts
// Stub — просто возвращает значение
const stub = () => 42;
expect(stub()).toBe(42);
```

### Spy — шпион

Spy используется, когда нужно **и сохранить реальное поведение, и проверить**, что функция была вызвана. Типичный сценарий — убедиться, что `console.log` вызвался, или что метод объекта был вызван с правильными аргументами, при этом не ломая его работу. Spy — это "наблюдатель", который не вмешивается.

```ts
const obj = { greet: (name: string) => `Hello, ${name}` };
const spy = vi.spyOn(obj, "greet");

obj.greet("Alice");

expect(spy).toHaveBeenCalledWith("Alice"); // Отслеживает вызов
expect(obj.greet("Bob")).toBe("Hello, Bob"); // Сохраняет поведение
```

### Mock — полноценная замена

Mock нужен, когда **и поведение нужно контролировать, и вызовы отслеживать**. Это самый мощный инструмент: можно задать возвращаемые значения, проверить количество вызовов, аргументы, порядок. Используйте mock, когда реальная зависимость медленная, нестабильная или недоступна в тестовой среде.

```ts
const mock = vi.fn();
mock("hello");

expect(mock).toHaveBeenCalledWith("hello"); // Отслеживает вызов
mock.mockReturnValue("world");
expect(mock()).toBe("world"); // Управляемое поведение
```

### Сравнительная таблица

| Характеристика | Stub | Spy | Mock |
|---|---|---|---|
| **Заменяет функцию** | ✅ | ✅ (оборачивает) | ✅ |
| **Отслеживает вызовы** | ❌ | ✅ | ✅ |
| **Сохраняет поведение** | ❌ | ✅ | ❌ (по умолчанию) |
| **Управляемое поведение** | ✅ (фиксированное) | ✅ (через mockReturnValue) | ✅ |
| **Восстановление оригинала** | ❌ | ✅ (mockRestore) | ❌ |

> 💡 В Vitest/Jest `vi.fn()` создаёт **mock**, `vi.spyOn()` создаёт **spy** (который можно превратить в mock через `mockReturnValue`). Термин "stub" используется редко — обычно это просто `vi.fn().mockReturnValue(...)`.

---

## vi.fn() — создание mock-функций

`vi.fn()` — основной инструмент для создания mock-функций в Vitest. Он возвращает специальную функцию, которая запоминает все свои вызовы, аргументы и возвращаемые значения. Это позволяет писать утверждения вроде "эта функция была вызвана 2 раза с аргументом 'hello'".

Используйте `vi.fn()`, когда нужно передать mock как колбэк, проп или зависимость. Если нужно "подшпиить" за **существующим** методом объекта — используйте `vi.spyOn()` вместо этого.

### Базовое использование

```ts
const mockFn = vi.fn();

mockFn("hello");
mockFn("world");

expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith("hello");
expect(mockFn).toHaveBeenLastCalledWith("world");
expect(mockFn).toHaveBeenNthCalledWith(1, "hello");
expect(mockFn).toHaveBeenNthCalledWith(2, "world");
```

### С реализацией

По умолчанию `vi.fn()` возвращает `undefined`. Если тестируемый код ожидает конкретное возвращаемое значение, его можно задать двумя способами: передать реализацию в конструктор (функцию) или использовать `.mockReturnValue()`. Первый способ гибче — он получает те же аргументы, что и оригинал.

```ts
// Mock с возвращаемым значением
const mockAdd = vi.fn((a: number, b: number) => a + b);
expect(mockAdd(2, 3)).toBe(5);

// Mock с фиксированным возвращаемым значением
const mockFn = vi.fn().mockReturnValue(42);
expect(mockFn()).toBe(42);
expect(mockFn("anything")).toBe(42); // Игнорирует аргументы
```

### Последовательные возвращаемые значения

Иногда функция вызывается несколько раз, и каждый вызов должен вернуть разное значение. Например, первый запрос к API возвращает данные, а второй — ошибку. `mockReturnValueOnce` позволяет задать поведение для каждого вызова по очереди, а `mockReturnValue` — дефолтное поведение для всех последующих вызовов.

```ts
const mockFn = vi.fn()
  .mockReturnValueOnce("first")
  .mockReturnValueOnce("second")
  .mockReturnValue("default");

expect(mockFn()).toBe("first");
expect(mockFn()).toBe("second");
expect(mockFn()).toBe("default");
expect(mockFn()).toBe("default"); // Все последующие — "default"
```

### Async mock

Большинство реальных API — асинхронные. Для мокирования таких функций используются `mockResolvedValue` (эквивалент `Promise.resolve`) и `mockRejectedValue` (эквивалент `Promise.reject`). Не забывайте `await` при вызове и `await expect(...).rejects` для проверки ошибок.

```ts
//Resolved value
const mockFetch = vi.fn().mockResolvedValue({ data: [1, 2, 3] });
const result = await mockFetch();
expect(result.data).toEqual([1, 2, 3]);

// Rejected value
const mockFetch = vi.fn().mockRejectedValue(new Error("Network error"));
await expect(mockFetch()).rejects.toThrow("Network error");

// Последовательные async-значения
const mockFn = vi.fn()
  .mockResolvedValueOnce("first")
  .mockResolvedValueOnce("second");
```

### Проверка вызовов

Помимо `toHaveBeenCalled` / `toHaveBeenCalledWith`, mock хранит полную историю в свойствах `.mock.calls` и `.mock.results`. Это полезно для сложных проверок: например, убедиться, что колбэк был вызван с определёнными аргументами в определённом порядке, или что функция вернула конкретные значения.

```ts
const mockFn = vi.fn();

mockFn("a", "b");
mockFn("c");

// Количество вызовов
expect(mockFn).toHaveBeenCalledTimes(2);

// Аргументы вызовов
expect(mockFn).toHaveBeenCalledWith("a", "b");
expect(mockFn).toHaveBeenLastCalledWith("c");
expect(mockFn).toHaveBeenNthCalledWith(1, "a", "b");

// Все вызовы
expect(mockFn.mock.calls).toEqual([["a", "b"], ["c"]]);

// Возвращаемые значения
const mockFn2 = vi.fn().mockReturnValue("result");
mockFn2();
expect(mockFn2.mock.results).toEqual([{ type: "return", value: "result" }]);
```

---

## vi.spyOn() — шпионаж за методами

`vi.spyOn()` нужен, когда метод уже существует на объекте, и вы хотите **не заменить его полностью, а подсмотреть** — был ли он вызван, с какими аргументами, сколько раз. По умолчанию spy сохраняет оригинальную реализацию, но её можно переопределить через `mockReturnValue` / `mockImplementation`.

Главное отличие от `vi.fn()`: spy оборачивает существующий метод, а `vi.fn()` создаёт новую функцию с нуля. После теста spy нужно восстановить через `mockRestore()`, иначе оригинальный метод останется подменённым для последующих тестов.

### Базовое использование

```ts
const consoleSpy = vi.spyOn(console, "log");

console.log("hello");

expect(consoleSpy).toHaveBeenCalledWith("hello");

consoleSpy.mockRestore(); // Восстановить оригинальный console.log
```

### Переопределение реализации

Spy по умолчанию сохраняет оригинальное поведение, но его можно переопределить — тогда spy ведёт себя как mock. Это полезно, когда нужно **и подменить поведение, и проверить вызовы**. После `mockReturnValue` spy перестаёт вызывать оригинальный метод. `mockRestore()` возвращает всё как было — и поведение, и оригинальный метод.

```ts
const user = {
  getName: () => "Alice",
};

const spy = vi.spyOn(user, "getName");

// Шпион сохраняет оригинальное поведение
expect(user.getName()).toBe("Alice");
expect(spy).toHaveBeenCalledTimes(1);

// Переопределение
spy.mockReturnValue("Bob");
expect(user.getName()).toBe("Bob");

// Восстановление
spy.mockRestore();
expect(user.getName()).toBe("Alice");
```

### Spy на модуле

Spy можно повесить и на экспорт модуля — для этого импортируйте модуль как namespace (`import * as`) и вызовите `vi.spyOn`. Это альтернатива `vi.mock()`: spy менее инвазивен, так как не заменяет весь модуль, а оборачивает один метод. Однако для ES-модулей в некоторых конфигурациях Vitest могут быть нюансы — если spy не работает, используйте `vi.mock()`.

```ts
import * as api from "./api";

it("calls api.fetchUsers", async () => {
  const spy = vi.spyOn(api, "fetchUsers").mockResolvedValue([{ id: 1 }]);
  
  const users = await api.fetchUsers();
  
  expect(spy).toHaveBeenCalled();
  expect(users).toEqual([{ id: 1 }]);
  
  spy.mockRestore();
});
```

### Spy на методах DOM

DOM-методы вроде `scrollTo`, `getBoundingClientRect`, `focus` часто вызываются в коде, но не имеют видимого эффекта в jsdom. Spy на таких методах позволяет проверить, что код действительно пытался скроллить или фокусировать элемент, не вызывая побочных эффектов.

```ts
const scrollSpy = vi.spyOn(HTMLElement.prototype, "scrollTo");

window.scrollTo(0, 100);

expect(scrollSpy).toHaveBeenCalledWith(0, 100);

scrollSpy.mockRestore();
```

---

## vi.mock() — мокирование модулей

`vi.mock()` — самый мощный механизм изоляции. Он **полностью заменяет модуль** на тестовую версию: все импорты этого модуля в тестируемом коде будут получать mock вместо оригинала. Vitest автоматически "поднимает" (hoists) вызов `vi.mock()` в начало файла — выше всех `import`, поэтому моки работают даже для ES-модулей с их статической природой.

**Когда использовать:** когда модуль — внешняя зависимость (API-клиент, библиотека) или когда нужно контролировать поведение модуля целиком. **Когда не использовать:** для внутренних утилит проекта — лучше тестировать их как есть, иначе тест проверяет моки, а не реальный код.

### Полное мокирование

Полное мокирование заменяет **все** экспорты модуля. Factory-функция возвращает объект, где каждый экспорт — `vi.fn()` с заданным поведением. Это максимально изолирует тест: никакой код из оригинального модуля не выполняется. Используйте, когда модуль — внешняя зависимость (API-клиент, SDK), и вам не нужна ни одна из его реальных функций.

```ts
// Полностью заменяет модуль
vi.mock("./api", () => ({
  fetchUsers: vi.fn().mockResolvedValue([{ id: 1, name: "Alice" }]),
  deleteUser: vi.fn().mockResolvedValue(true),
}));

// Использование
import { fetchUsers } from "./api";

it("loads users", async () => {
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

### Частичное мокирование (с сохранением оригинала)

Иногда нужно замокать только часть модуля, оставив остальные экспорты рабочими. Например, заменить `fetchUsers` на mock, но сохранить реальную реализацию `deleteUser`. Для этого factory-функция получает аргумент `importOriginal` — вызвав его, вы получаете реальный модуль и можете "размазать" его по mock-объекту через spread.

Это компромисс: вы получаете изоляцию для нужной части, но сохраняете реальное поведение для остального. Минус — тест всё ещё зависит от реальной реализации, и если она сломается — тест упадёт.

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

Если передать в `vi.mock()` только путь модуля без factory-функции, Vitest автоматически заменит все экспорты на `vi.fn()`, возвращающие `undefined`. Это полезно для быстрого прототипирования тестов, но **опасно для продакшн-тестов**: все функции возвращают `undefined`, и легко пропустить, что mock настроен неправильно. Используйте автоматическое мокирование только когда вам не важны возвращаемые значения.

```ts
// Автоматически мокирует все экспорты
vi.mock("./api");

// Все функции становятся vi.fn() с undefined
import { fetchUsers } from "./api";

it("fetchUsers is mocked", () => {
  expect(vi.mocked(fetchUsers)).toBeDefined();
  expect(fetchUsers()).toBeUndefined(); // По умолчанию возвращает undefined
});
```

### vi.mock() с типами

`vi.fn` принимает generic-тип сигнатуры функции. Это даёт полную типобезопасность: TypeScript проверит, что `mockResolvedValue` возвращает правильный тип, а `mockImplementation` принимает корректные аргументы. Используйте это, когда factory-функция сложная и возвращаемые значения должны соответствовать реальным типам.

```ts
vi.mock("./api", () => ({
  fetchUsers: vi.fn<() => Promise<User[]>>().mockResolvedValue([
    { id: 1, name: "Alice" },
  ]),
}));
```

### vi.mocked() — типизация моков

После `vi.mock()` TypeScript не знает, что импортированная функция теперь mock — он видит обычный тип `fetchUsers`. Метод `vi.mocked()` говорит компилятору: "эта функция — mock, дай мне доступ к `mockResolvedValue`, `mockReturnValue` и прочим mock-методам". Без `vi.mocked()` придётся использовать `as any` или `as Mock`, что ломает типобезопасность.

```ts
import { fetchUsers } from "./api";

vi.mock("./api");

it("uses typed mock", async () => {
  const mockFetch = vi.mocked(fetchUsers);
  mockFetch.mockResolvedValue([{ id: 1, name: "Alice" }]);
  
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

---

## vi.hoisted() — переменные для vi.mock

Vitest поднимает `vi.mock()` в самый верх файла — выше всех `import` и `const`. Это значит, что обычные переменные, объявленные рядом, ещё не существуют на момент вызова `vi.mock()`. Если вы попытаетесь использовать `mockFn` из внешнего `const` внутри factory-функции `vi.mock()`, получите `ReferenceError`.

`vi.hoisted()` решает эту проблему: он тоже поднимается в верх файла, вместе с `vi.mock()`, и становится доступен внутри factory.

### Проблема

`vi.mock()` вызывается **до** импортов (hoisting), поэтому обычные переменные недоступны внутри factory-функции:

```ts
// ❌ ОШИБКА — mockFn не доступен в vi.mock
const mockFn = vi.fn();

vi.mock("./api", () => ({
  fetchUsers: mockFn, // ❌ mockFn is not defined
}));
```

### Решение: vi.hoisted()

`vi.hoisted()` принимает factory-функцию и возвращает её результат, поднимая вызов в начало файла (на тот же уровень, что и `vi.mock()`). Это позволяет создать mock-переменные, которые будут доступны и в factory `vi.mock()`, и в теле тестов. Возвращайте объект с нужными переменными — деструктуризация сделает код чище.

```ts
// ✅ vi.hoisted() поднимает переменные вместе с vi.mock
const { mockFetchUsers, mockDeleteUser } = vi.hoisted(() => ({
  mockFetchUsers: vi.fn(),
  mockDeleteUser: vi.fn(),
}));

vi.mock("./api", () => ({
  fetchUsers: mockFetchUsers,
  deleteUser: mockDeleteUser,
}));

// Использование в тестах
import { fetchUsers } from "./api";

it("fetches users", async () => {
  mockFetchUsers.mockResolvedValue([{ id: 1 }]);
  
  const users = await fetchUsers();
  
  expect(users).toEqual([{ id: 1 }]);
  expect(mockFetchUsers).toHaveBeenCalled();
});
```

### vi.hoisted() с классами

`vi.hoisted()` полезен не только для mock-функций, но и для классов. Типичный пример — `IntersectionObserver`, которого нет в jsdom. Чтобы создать mock-класс и использовать его в `vi.mock()`, объявите класс внутри `vi.hoisted()`.

```ts
const { MockIntersectionObserver } = vi.hoisted(() => {
  class MockIntersectionObserver {
    observe = vi.fn();
    unobserve = vi.fn();
    disconnect = vi.fn();
  }
  return { MockIntersectionObserver };
});

vi.mock("./IntersectionObserver", () => ({
  IntersectionObserver: MockIntersectionObserver,
}));
```

---

## MSW — мокирование API на сетевом уровне

MSW — это другой уровень мокирования по сравнению с `vi.mock()` и `vi.fn()`. Вместо замены функций в коде он перехватывает **реальные HTTP-запросы** на уровне Service Worker. Код компонента делает обычный `fetch('/api/users')`, и MSW возвращает тестовые данные, как если бы они пришли с сервера.

**Преимущество:** тестируемый код не знает о моках — он работает так же, как в продакшне. Нет разницы между `fetch`, `axios`, `react-query` — MSW перехватывает всё. **Недостаток:** настройка сложнее, чем `vi.fn()`, и требует отдельной инфраструктуры (handlers, server).

### Что такое MSW

**MSW (Mock Service Worker)** — библиотека для мокирования HTTP-запросов на уровне Service Worker. Код компонента не знает, что работает с моком — он думает, что общается с реальным API.

### Преимущества MSW

| Преимущество | Описание |
|---|---|
| **Работает с любой библиотекой** | fetch, axios, react-query, swr — неважно |
| **Один handler для тестов и разработки** | Можно использовать в Storybook, моках для фронтенда |
| **Точечное переопределение** | Можно переопределить handler для конкретного теста |
| **Реалистичное мокирование** | Перехватывает запросы на сетевом уровне, не в коде |
| **Поддержка REST и GraphQL** | Из коробки |

### Установка

MSW — dev-зависимость: она не попадает в продакшн-бандл. В браузере она активируется только при явном вызове `setupWorker()`, в тестах — через `setupServer()`.

```bash
npm install -D msw
```

### Инициализация

`npx msw init` создаёт Service Worker-скрипт в публичной директории. Для тестов этот файл не нужен — MSW в тестах работает через `setupServer` (Node.js), а не через Service Worker. Файл `mockServiceWorker.js` нужен только для мокирования в браузере при разработке (например, в Storybook).

```bash
npx msw init public/ --save
```

Это создаст `public/mockServiceWorker.js` — Service Worker, который перехватывает запросы.

---

## MSW — настройка и структура

MSW требует начальной настройки: нужно создать handlers (описания моков для каждого endpoint), инициализировать сервер для тестов (`setupServer`) и подключить его в глобальном setup-файле. Структура файлов может выглядеть пугающе, но она логична: handlers переиспользуются между тестами, а `server.use()` позволяет точечно переопределять их для конкретных сценариев.

```
src/
  test/
    mocks/
      handlers.ts    — базовые handlers
      server.ts      — setupServer для Node.js
      browser.ts     — setupWorker для браузера (опционально)
    setup.ts         — глобальная настройка тестов
```

### Handlers

Handlers — это "контракт" вашего тестового API. Каждый handler описывает, что вернуть на конкретный HTTP-метод и URL. Пишите handlers как можно ближе к реальному API: те же пути, те же структуры ответов. Это позволяет переиспользовать их между тестами и даже использовать в Storybook или для разработки фронтенда без бэкенда.

```ts
// src/test/mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  // GET /api/users
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "Alice" },
      { id: 2, name: "Bob" },
    ]);
  }),

  // GET /api/users/:id
  http.get("/api/users/:id", ({ params }) => {
    const { id } = params;
    return HttpResponse.json({ id, name: "Alice" });
  }),

  // POST /api/users
  http.post("/api/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(
      { id: 3, ...body },
      { status: 201 }
    );
  }),

  // DELETE /api/users/:id
  http.delete("/api/users/:id", ({ params }) => {
    return new HttpResponse(null, { status: 204 });
  }),
];
```

### Server (для Node.js / тестов)

`setupServer` создаёт Node.js-совместимый HTTP-сервер, который перехватывает запросы в тестовой среде (jsdom/Node). Для браузера используется `setupWorker` — он регистрирует настоящий Service Worker. В тестах достаточно `setupServer`. Экспортируйте `server`, чтобы использовать `server.use()` для переопределений в отдельных тестах.

```ts
// src/test/mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

### Setup-файл

Этот файл подключается в `vitest.config.ts` через `setupFiles`. Он гарантирует, что MSW-сервер стартует до всех тестов, сбрасывается между ними и останавливается после. Без `resetHandlers()` в `afterEach` переопределения из одного теста "протекут" в следующий — частая причина хрупких тестов.

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach, beforeAll, afterAll } from "vitest";
import { server } from "./mocks/server";

// Запуск сервера перед всеми тестами
beforeAll(() => server.listen({ onUnhandledRequest: "error" }));

// Сброс handlers после каждого теста
afterEach(() => {
  server.resetHandlers();
  cleanup();
});

// Остановка сервера после всех тестов
afterAll(() => server.close());
```

### onUnhandledRequest

Эта опция контролирует, что делает MSW, когда код делает запрос, для которого нет handler-а. `"error"` — тест упадёт (строгий режим, рекомендуется). `"warn"` — выведет предупреждение, но тест пройдёт. `"bypass"` — запрос уйдёт в реальную сеть (или будет проигнорирован в jsdom). Используйте `"error"` для уверенности, что все запросы покрыты.

```ts
server.listen({
  onUnhandledRequest: "error", // Ошибка на необработанные запросы
  // onUnhandledRequest: "warn", // Предупреждение
  // onUnhandledRequest: "bypass", // Игнорировать
});
```

> 💡 **Рекомендация:** Используйте `"error"` в тестах — это гарантирует, что все запросы покрыты handlers. Если тест делает запрос без handler — вы сразу узнаете.

---

## Переопределение handlers в тестах

Базовые handlers покрывают "счастливый путь" — успешные ответы API. Но нужно тестировать и ошибки, пустые ответы, задержки. `server.use()` добавляет временные handler-переопределения, которые действуют только до следующего `resetHandlers()` (обычно в `afterEach`). Это позволяет писать изолированные тесты для каждого сценария без изменения глобальных handlers.

### server.use() — точечное переопределение

```ts
import { http, HttpResponse } from "msw";
import { server } from "../test/mocks/server";

it("handles server error", async () => {
  // Переопределяет handler только для этого теста
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json(
        { error: "Server error" },
        { status: 500 }
      );
    })
  );

  render(<UserList />);

  expect(await screen.findByText(/server error/i)).toBeInTheDocument();
});

it("handles empty list", async () => {
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json([]);
    })
  );

  render(<UserList />);

  expect(await screen.findByText(/no users found/i)).toBeInTheDocument();
});
```

### Как работает server.use()

Понимание жизненного цикла handlers помогает избежать багов. MSW хранит два слоя: **базовые handlers** (из `setupServer(...handlers)`) и **переопределения** (из `server.use()`). При запросе MSW сначала ищет среди переопределений, потом — среди базовых. `resetHandlers()` удаляет переопределения, возвращаясь к базовым handler-ам.

```
1. beforeAll: server.listen() — запускает сервер с базовыми handlers
2. Тест 1: server.use() — добавляет переопределение
3. afterEach: server.resetHandlers() — убирает переопределения
4. Тест 2: базовые handlers снова активны
```

### Переопределение с задержкой

В реальности API отвечает не мгновенно. Компоненты должны показывать loading-состояние, пока данные грузятся. Чтобы протестировать это, добавьте `await new Promise(resolve => setTimeout(resolve, ms))` в handler. В тестах с fake timers используйте `vi.advanceTimersByTimeAsync()` для управления временем.

> **Внимание:** при использовании `vi.useFakeTimers()` обычный `setTimeout` в handler не сработает — нужно использовать `advanceTimersByTimeAsync`.

```ts
it("shows loading state", async () => {
  server.use(
    http.get("/api/users", async () => {
      await new Promise((resolve) => setTimeout(resolve, 1000));
      return HttpResponse.json([{ id: 1, name: "Alice" }]);
    })
  );

  render(<UserList />);

  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

### Переопределение для конкретного URL

Иногда поведение handler-а зависит от параметров запроса — например, запрос несуществующего пользователя должен вернуть 404. Внутри handler-а доступны `params`, `request`, `cookies` — можно реализовать любую логику. Это особенно полезно для тестирования error-handling в компонентах.

```ts
server.use(
  http.get("/api/users/:id", ({ params }) => {
    if (params.id === "999") {
      return HttpResponse.json({ error: "Not found" }, { status: 404 });
    }
    return HttpResponse.json({ id: params.id, name: "Alice" });
  })
);
```

---

## Мокирование fetch

Если вы не используете MSW, нужно мокировать `fetch` вручную. Это более хрупкий подход: приходится вручную собирать объект Response с полями `ok`, `status`, `json()`. MSW делает всё это за вас. Ручной mock оправдан только в простых проектах, где MSW — избыточен, или для unit-тестов самой функции-обёртки над fetch.

### Без MSW — через vi.mock

```ts
// Глобальный mock fetch
global.fetch = vi.fn();

beforeEach(() => {
  vi.mocked(global.fetch).mockReset();
});

it("fetches data", async () => {
  vi.mocked(global.fetch).mockResolvedValueOnce({
    ok: true,
    json: async () => [{ id: 1, name: "Alice" }],
  });

  const users = await fetchUsers();

  expect(global.fetch).toHaveBeenCalledWith("/api/users");
  expect(users).toEqual([{ id: 1, name: "Alice" }]);
});
```

### С обработкой ошибок

Тестирование ошибок — одна из главных причин мокирования. В реальности сложно заставить API вернуть 500 или таймаут. С mock это делается одной строкой. Проверяйте как минимум два сценария: сетевая ошибка (fetch отклонён) и HTTP-ошибка (ответ с не-2xx статусом).

```ts
it("handles network error", async () => {
  vi.mocked(global.fetch).mockRejectedValueOnce(new Error("Network error"));

  await expect(fetchUsers()).rejects.toThrow("Network error");
});

it("handles API error", async () => {
  vi.mocked(global.fetch).mockResolvedValueOnce({
    ok: false,
    status: 500,
    json: async () => ({ error: "Server error" }),
  });

  await expect(fetchUsers()).rejects.toThrow("Server error");
});
```

---

## Мокирование localStorage

`localStorage` в jsdom работает, но данные **сохраняются между тестами** — это частая причина хрупких тестов. Решение: очищать `localStorage` в `beforeEach`. Если нужно проверить, что код правильно обрабатывает квоту или недоступность storage, используйте mock через `vi.spyOn` или `Object.defineProperty`.

### Очистка между тестами

```ts
beforeEach(() => {
  localStorage.clear();
});

it("persists data to localStorage", () => {
  render(<SettingsPage />);
  
  fireEvent.click(screen.getByLabelText(/dark mode/i));
  
  expect(localStorage.getItem("theme")).toBe("dark");
});
```

### Mock localStorage

Полный mock localStorage нужен, когда jsdom-реализация не подходит — например, нужно симулировать `QuotaExceededError` или проверить, что `setItem` вызывается с правильными аргументами. В большинстве случаев достаточно простой очистки через `localStorage.clear()` в `beforeEach`.

```ts
const localStorageMock = (() => {
  let store: Record<string, string> = {};
  return {
    getItem: vi.fn((key: string) => store[key] ?? null),
    setItem: vi.fn((key: string, value: string) => {
      store[key] = value;
    }),
    removeItem: vi.fn((key: string) => {
      delete store[key];
    }),
    clear: vi.fn(() => {
      store = {};
    }),
  };
})();

Object.defineProperty(window, "localStorage", {
  value: localStorageMock,
});
```

---

## Мокирование таймеров

Код с `setTimeout`, `setInterval`, `debounce`, `throttle` заставляет тесты ждать реальное время — это медленно и хрупко. `vi.useFakeTimers()` заменяет все таймеры на управляемые: тест сам "прокручивает время" через `vi.advanceTimersByTime()`. Это делает тесты мгновенными и детерминированными.

**Важно:** не забывайте `vi.useRealTimers()` после теста, иначе другие тесты сломаются. Лучше вынести `useFakeTimers` / `useRealTimers` в `beforeEach` / `afterEach`.

### vi.useFakeTimers()

```ts
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

### vi.setSystemTime()

`Date.now()` и `new Date()` зависят от текущего времени — тест, запущенный в разные дни, даст разные результаты. `vi.setSystemTime()` фиксирует "текущее время" для всех вызовов `Date` внутри теста. Полезно для тестирования относительных дат ("2 hours ago"), дедлайнов, календарей.

```ts
it("formats relative date", () => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date("2026-01-15T12:00:00Z"));

  expect(formatRelativeDate(new Date("2026-01-15T10:00:00Z"))).toBe("2 hours ago");

  vi.useRealTimers();
});
```

### advanceTimersByTimeAsync — для async/await

Обычный `advanceTimersByTime` не обрабатывает промисы, которые резолвятся внутри таймеров. Если тестируемая функция использует `async/await` с задержками, нужен `advanceTimersByTimeAsync` — он "прокручивает" время и ждёт разрешения всех промисов. Без этого `await promise` зависнет навсегда.

```ts
it("loads data after delay", async () => {
  vi.useFakeTimers();

  const promise = loadData();

  await vi.advanceTimersByTimeAsync(3000);

  const data = await promise;
  expect(data).toEqual([1, 2, 3]);

  vi.useRealTimers();
});
```

---

## Мокирование модулей браузера

jsdom реализует не все браузерные API. `IntersectionObserver`, `matchMedia`, `ResizeObserver`, `scrollTo` — всё это нужно мокировать вручную. Делайте это в глобальном setup-файле, чтобы mock был доступен во всех тестах. Если mock нужен только в одном тесте — используйте `vi.spyOn` с `mockRestore` в `afterEach`.

### IntersectionObserver

```ts
const mockIntersectionObserver = vi.fn();
mockIntersectionObserver.mockReturnValue({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
});

window.IntersectionObserver = mockIntersectionObserver;
```

### matchMedia

`matchMedia` используется для media-queries — например, для определения тёмной темы через `prefers-color-scheme` или адаптивной вёрстки. jsdom его не реализует, поэтому любой код, использующий `matchMedia`, упадёт без mock. Mock ниже возвращает объект с `matches: false` — для большинства тестов этого достаточно. Для теста тёмной темы установите `matches: true`.

```ts
Object.defineProperty(window, "matchMedia", {
  writable: true,
  value: vi.fn().mockImplementation((query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});
```

### window.scrollTo

`scrollTo` в jsdom существует, но не делает ничего — окно не скроллится. Mock нужен, чтобы проверить, что код действительно вызывает скролл (например, кнопка "наверх" или навигация по якорям). Альтернатива — `vi.spyOn(HTMLElement.prototype, "scrollTo")`, который автоматически восстанавливается.

```ts
window.scrollTo = vi.fn();

it("scrolls to top", () => {
  scrollToTop();
  expect(window.scrollTo).toHaveBeenCalledWith(0, 0);
});
```

---

## Фабрики тестовых данных

Фабрики — это функции, которые создают валидные тестовые объекты с разумными значениями по умолчанию. Они решают три проблемы: **дублирование** (один и тот же объект в каждом тесте), **хрупкость** (добавление поля в тип ломает все тесты) и **читаемость** (в тесте видны только релевантные поля через `overrides`).

Альтернатива фабрикам — фикстуры (заранее созданные объекты) и библиотеки вроде `fishery` или `@jackfranklin/test-data-bot`. Фабрики проще и не требуют зависимостей.

### Зачем нужны фабрики

```ts
// ❌ Дублирование данных в тестах
it("creates user", () => {
  const user = { id: 1, name: "Alice", email: "alice@example.com", role: "user" };
  // ...
});

it("displays user", () => {
  const user = { id: 2, name: "Bob", email: "bob@example.com", role: "admin" };
  // ...
});

// ✅ Фабрика — переиспользование и гибкость
const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  ...overrides,
});

it("creates user", () => {
  const user = createUser({ name: "Alice" });
  // ...
});
```

### Базовая фабрика

Паттерн прост: функция принимает `overrides` типа `Partial<T>` и возвращает объект с дефолтными значениями, поверх которых через spread ложатся переопределения. `crypto.randomUUID()` для `id` гарантирует уникальность между вызовами — тесты не будут влиять друг на друга через совпадающие ID.

```ts
// src/test/factories.ts
export const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  createdAt: new Date("2026-01-01"),
  ...overrides,
});

export const createProduct = (overrides?: Partial<Product>): Product => ({
  id: crypto.randomUUID(),
  name: "Test Product",
  price: 29.99,
  category: "electronics",
  inStock: true,
  ...overrides,
});
```

### Использование фабрик

Обратите внимание на читаемость: в каждом тесте видно только то, что **важно для этого конкретного сценария**. `createUser({ name: "Alice" })` говорит: "тест про имя Alice, остальное не важно". `createUser({ role: "admin" })` говорит: "тест про роль admin". Это делает тесты самодокументирующимися.

```ts
import { createUser } from "../test/factories";

it("displays user name", () => {
  const user = createUser({ name: "Alice" });
  render(<UserProfile user={user} />);
  expect(screen.getByText("Alice")).toBeInTheDocument();
});

it("shows admin badge for admin users", () => {
  const admin = createUser({ role: "admin" });
  render(<UserProfile user={admin} />);
  expect(screen.getByText(/admin/i)).toBeInTheDocument();
});

it("hides admin badge for regular users", () => {
  const user = createUser({ role: "user" });
  render(<UserProfile user={user} />);
  expect(screen.queryByText(/admin/i)).not.toBeInTheDocument();
});
```

### Фабрика с последовательностью

Иногда нужны предсказуемые, а не случайные данные — например, для snapshot-тестов или когда порядок важен. Счётчик `userIdCounter` даёт уникальные, предсказуемые ID. **Минус:** состояние счётчика сохраняется между тестами, поэтому его нужно сбрасывать в `beforeEach`.

```ts
let userIdCounter = 0;

export const createUser = (overrides?: Partial<User>): User => {
  userIdCounter++;
  return {
    id: userIdCounter,
    name: `User ${userIdCounter}`,
    email: `user${userIdCounter}@example.com`,
    role: "user",
    ...overrides,
  };
};

// Использование
const user1 = createUser(); // { id: 1, name: "User 1", ... }
const user2 = createUser(); // { id: 2, name: "User 2", ... }
const admin = createUser({ role: "admin" }); // { id: 3, name: "User 3", role: "admin" }
```

### Фабрика для массивов

Часто нужен не один объект, а список — например, для тестирования таблицы или виртуального скролла. Функция-обёртка `createUsers(count)` генерирует массив нужной длины. Все объекты будут идентичны (кроме `id`), что хорошо для проверки рендера списка, но плохо для проверки сортировки — для этого используйте `overrides`.

```ts
export const createUsers = (count: number, overrides?: Partial<User>): User[] => {
  return Array.from({ length: count }, () => createUser(overrides));
};

// Использование
const users = createUsers(5); // 5 пользователей
const admins = createUsers(3, { role: "admin" }); // 3 админа
```

---

## Жизненный цикл моков: clear, reset, restore

Vitest предоставляет три уровня "очистки" моков, и путаница между ними — частая причина багов. Разница в том, **что именно сбрасывается**: история вызовов, реализация (возвращаемые значения), или оригинальный метод (для spy).

Правило выбора:
- **clear** — если нужно просто "забыть" вызовы между тестами, но оставить реализацию.
- **reset** — если нужно сбросить и вызовы, и реализацию (mock вернёт `undefined`).
- **restore** — если нужно вернуть оригинальный метод (только для `vi.spyOn`).

### vi.clearAllMocks()

`clearAllMocks` — самый "мягкий" вариант. Он обнуляет счётчики вызовов (`mock.calls`, `mock.results`), но **не трогает реализацию**. Если mock возвращал `42` через `mockReturnValue(42)`, он продолжит возвращать `42` после очистки. Это самый частый выбор для `afterEach` — тесты получают чистую историю, но не теряют настройки моков.

Очищает **историю вызовов** всех моков, но сохраняет реализацию.

```ts
const mockFn = vi.fn().mockReturnValue(42);
mockFn("hello");

expect(mockFn).toHaveBeenCalledTimes(1);

vi.clearAllMocks();

expect(mockFn).toHaveBeenCalledTimes(0); // История очищена
expect(mockFn()).toBe(42); // Реализация сохранена
```

### vi.resetAllMocks()

`resetAllMocks` агрессивнее: он очищает историю **и** сбрасывает реализацию. После `reset` mock-функция возвращает `undefined`, пока вы заново не зададите `mockReturnValue`. Используйте, когда реализация mock-а задаётся в каждом тесте индивидуально и вы хотите гарантировать, что настройки предыдущего теста не протекли.

Очищает историю **и убирает реализацию** (моки возвращают `undefined`).

```ts
const mockFn = vi.fn().mockReturnValue(42);
mockFn("hello");

vi.resetAllMocks();

expect(mockFn).toHaveBeenCalledTimes(0); // История очищена
expect(mockFn()).toBeUndefined(); // Реализация убрана
```

### vi.restoreAllMocks()

`restoreAllMocks` — самый агрессивный. Он делает всё, что `resetAllMocks`, **плюс** восстанавливает оригинальные методы, подменённые через `vi.spyOn()`. Без этого spy остаётся "заражённым" — все последующие тесты будут видеть mock вместо оригинала. Если вы используете `vi.spyOn()` — `restoreAllMocks` обязателен в `afterEach`.

Очищает историю, убирает реализацию **и восстанавливает оригиналы** (для `vi.spyOn`).

```ts
const obj = { greet: () => "hello" };
const spy = vi.spyOn(obj, "greet").mockReturnValue("mocked");

expect(obj.greet()).toBe("mocked");

vi.restoreAllMocks();

expect(obj.greet()).toBe("hello"); // Оригинальная реализация восстановлена
```

### Сравнительная таблица

| Метод | История вызовов | Реализация | Оригинал (spyOn) |
|---|---|---|---|
| `mockClear()` / `clearAllMocks()` | Очищает | Сохраняет | Сохраняет |
| `mockReset()` / `resetAllMocks()` | Очищает | Убирает (→ undefined) | Сохраняет |
| `mockRestore()` / `restoreAllMocks()` | Очищает | Убирает | Восстанавливает |

### Когда использовать

В большинстве проектов достаточно `vi.clearAllMocks()` в `afterEach` — это сбрасывает историю вызовов, но сохраняет реализации. Если вы активно используете `vi.spyOn()`, лучше `vi.restoreAllMocks()` — он вернёт оригинальные методы. `vi.resetAllMocks()` используется реже — он агрессивнее всех и может сломать тесты, если mock-реализация задаётся в `beforeAll`.

```ts
afterEach(() => {
  // Обычно достаточно clearAllMocks
  vi.clearAllMocks();
});

// restoreAllMocks — если используете spyOn
afterEach(() => {
  vi.restoreAllMocks();
});
```

---

## Лучшие практики

Эти правила сформировались из опыта. Главное: **моки — это зло, которое приходится принимать**. Каждый mock — это место, где тест расходится с реальностью. Чем больше моков, тем меньше уверенности, что код работает в продакшне. Поэтому моцируйте минимум необходимого и выбирайте самый "нижний" уровень мокирования (MSW > vi.mock > vi.spyOn > vi.fn).

### 1. Используйте MSW для API

```ts
// ✅ MSW — реалистичное мокирование
server.use(
  http.get("/api/users", () => HttpResponse.json([{ id: 1 }]))
);

// ❌ Ручной mock fetch — хрупкий
global.fetch = vi.fn().mockResolvedValue({
  ok: true,
  json: async () => [{ id: 1 }],
});
```

### 2. Переопределяйте handlers точечно

```ts
// ✅ Точечное переопределение
it("handles error", async () => {
  server.use(
    http.get("/api/users", () => HttpResponse.json({ error: "Error" }, { status: 500 }))
  );
  // ...
});

// ❌ Глобальное изменение handlers
beforeEach(() => {
  handlers[0] = http.get("/api/users", () => ...); // Хрупко
});
```

### 3. Используйте фабрики для данных

```ts
// ✅ Фабрика — гибкость и переиспользование
const user = createUser({ name: "Alice" });

// ❌ Хардкод в каждом тесте
const user = { id: 1, name: "Alice", email: "alice@example.com", role: "user" };
```

### 4. Очищайте моки между тестами

```ts
afterEach(() => {
  vi.clearAllMocks();
  server.resetHandlers();
});
```

### 5. Мокируйте только внешние зависимости

```ts
// ✅ Мокируем API (внешняя зависимость)
server.use(http.get("/api/users", () => ...));

// ❌ Мокируем утилиты проекта (внутренняя зависимость)
vi.mock("./utils/format", () => ({ format: vi.fn() }));
```

### 6. Используйте vi.hoisted() для переменных в vi.mock

```ts
// ✅ vi.hoisted() — переменные доступны в vi.mock
const { mockFn } = vi.hoisted(() => ({ mockFn: vi.fn() }));
vi.mock("./api", () => ({ fetchUsers: mockFn }));

// ❌ Ошибка — переменная не доступна
const mockFn = vi.fn();
vi.mock("./api", () => ({ fetchUsers: mockFn })); // ❌
```

### 7. Типизируйте моки

```ts
// ✅ vi.mocked() — типизация
import { fetchUsers } from "./api";
vi.mock("./api");

const mockFetch = vi.mocked(fetchUsers);
mockFetch.mockResolvedValue([{ id: 1 }]);

// ❌ Без типизации — нет autocomplete
(fetchUsers as any).mockResolvedValue([{ id: 1 }]);
```

---

## Антипаттерны

Антипаттерны мокирования объединены одной идеей: **тест должен проверять поведение, а не реализацию**. Если тест ломается при рефакторинге, который не меняет поведение — это признак избыточного мокирования. Если тест проходит, но код не работает — это признак, что моки слишком далеки от реальности.

### 1. Избыточное мокирование

```ts
// ❌ Мокирует всё — тест проверяет моки, а не код
vi.mock("react", () => ({ useState: vi.fn() }));
vi.mock("./utils", () => ({ format: vi.fn() }));
vi.mock("./hooks", () => ({ useAuth: vi.fn() }));

// ✅ Мокирует только внешние зависимости
server.use(http.get("/api/data", () => HttpResponse.json({ data: [] })));
```

### 2. Мокирование внутренних модулей без необходимости

Если модуль — часть вашего проекта (утилиты, хелперы), не мокируйте его. Тест должен проверять, что **все части работают вместе**. Мокирование внутренней утилиты означает, что тест проверяет только тестируемый код, но не его взаимодействие с утилитой — а баги часто живут именно на стыке.

```ts
// ❌ Мокирует утилиту проекта
vi.mock("./utils/format", () => ({ formatPrice: vi.fn().mockReturnValue("$10") }));

// ✅ Использует реальную утилиту
import { formatPrice } from "./utils/format";
expect(formatPrice(10)).toBe("$10.00");
```

### 3. Забытый mockRestore

Каждый `vi.spyOn()` без последующего `mockRestore()` оставляет подменённый метод "заражённым" для всех последующих тестов. Это проявляется как странные баги: тест проходит изолированно, но падает при запуске всего сьюта. Решение — `vi.restoreAllMocks()` в `afterEach`, который восстанавливает все spy за раз.

```ts
// ❌ Не восстанавливает оригинал
const spy = vi.spyOn(console, "log");
// ... тест
// console.log остаётся замоканым для следующих тестов

// ✅ Восстанавливает оригинал
const spy = vi.spyOn(console, "log");
// ... тест
spy.mockRestore();

// ✅ Или в afterEach
afterEach(() => {
  vi.restoreAllMocks();
});
```

### 4. Зависимость от порядка тестов

Каждый тест должен быть **полностью независимым**. Если тест B зависит от того, что тест A создал пользователя — это хрупкий тест. При запуске в другом порядке, параллельно или с фильтром (`it.only`) он сломается. Vitest по умолчанию может запускать тесты в случайном порядке — это помогает ловить такие зависимости.

```ts
// ❌ Второй тест зависит от первого
it("creates user", () => {
  mockCreateUser({ id: 1 });
  expect(getUsers()).toHaveLength(1);
});

it("finds user", () => {
  // ❌ Пользователь создан в предыдущем тесте
  expect(findUser(1)).not.toBeNull();
});

// ✅ Каждый тест независим
it("finds user", () => {
  mockCreateUser({ id: 1 });
  expect(findUser(1)).not.toBeNull();
});
```

### 5. Хардкод данных вместо фабрик

Дублирование тестовых данных — тихий убийца. Когда в тип добавляется новое обязательное поле, приходится обновлять каждый тест. Фабрика решает это: обновление в одном месте. Кроме того, фабрика делает тесты читаемее — видно, **что именно** важно в тесте (через `overrides`), а что — просто заполнение полей.

```ts
// ❌ Дублирование данных
it("test 1", () => {
  const user = { id: 1, name: "Alice", email: "alice@example.com" };
});
it("test 2", () => {
  const user = { id: 2, name: "Bob", email: "bob@example.com" };
});

// ✅ Фабрика
it("test 1", () => {
  const user = createUser({ name: "Alice" });
});
it("test 2", () => {
  const user = createUser({ name: "Bob" });
});
```

### 6. Игнорирование onUnhandledRequest

По умолчанию MSW молча пропускает запросы без handlers. Это значит, что если ваш код делает запрос, который вы не предусмотрели, — тест пройдёт, но в продакшне будет ошибка. `onUnhandledRequest: "error"` превращает такие запросы в провал теста, заставляя вас явно описать все ожидаемые API-вызовы.

```ts
// ❌ Необработанные запросы игнорируются — сложно отловить ошибки
server.listen();

// ✅ Ошибка на необработанные запросы
server.listen({ onUnhandledRequest: "error" });
```

### 7. Мокирование без очистки

Моки накапливаются между тестами, если их не очищать. `server.use()` в одном тесте без `resetHandlers()` в `afterEach` изменит поведение MSW для всех последующих тестов. То же с `vi.fn()` — история вызовов растёт, и `toHaveBeenCalledTimes` будет считать вызовы из предыдущих тестов. Всегда очищайте моки в `afterEach`.

```ts
// ❌ Моки накапливаются между тестами
beforeAll(() => {
  vi.mock("./api");
});

// ✅ Очистка в afterEach
afterEach(() => {
  vi.clearAllMocks();
  server.resetHandlers();
});
```
