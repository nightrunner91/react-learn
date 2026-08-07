# Основы тестирования: пирамида, принципы, стратегии

## Содержание

1. [Что такое тестирование и зачем оно нужно](#что-такое-тестирование-и-зачем-оно-нужно)
2. [Пирамида тестирования](#пирамида-тестирования)
3. [Testing Trophy — альтернативная модель](#testing-trophy--альтернативная-модель)
4. [FIRST-принципы](#first-принципы)
5. [Паттерн AAA (Arrange-Act-Assert)](#паттерн-aaa-arrange-act-assert)
6. [TDD: Red-Green-Refactor](#tdd-red-green-refactor)
7. [Что тестировать, что не тестировать](#что-тестировать-что-не-тестировать)
8. [Покрытие кода (Code Coverage)](#покрытие-кода-code-coverage)
9. [Классификация тестов по назначению](#классификация-тестов-по-назначению)
10. [Стоимость тестов](#стоимость-тестов)
11. [Лучшие практики](#лучшие-практики)
12. [Антипаттерны](#антипаттерны)

---

## Что такое тестирование и зачем оно нужно

**Тестирование** — это процесс проверки того, что код работает так, как ожидается. Автоматизированные тесты — это код, который проверяет другой код.

### Зачем тестировать

| Проблема без тестов | Как решают тесты |
|---|---|
| Рефакторинг ломает существующий функционал | Тесты ловят регрессии до деплоя |
| Непонятно, что должна делать функция | Тесты — живая документация ожидаемого поведения |
| Страх менять код («а вдруг сломается») | Тесты дают уверенность при изменениях |
| Ручная проверка каждого релиза | Автотесты выполняются за секунды в CI |
| Баги находят пользователи | Баги обнаруживаются на этапе разработки |

### Когда тесты окупаются

Тесты окупаются не всегда одинаково. Вот факторы, влияющие на ROI:

**Высокий ROI — тестировать обязательно:**
- Критичная бизнес-логика (платежи, авторизация, права доступа)
- Код, который часто меняется
- Код, от которого зависят другие модули (утилиты, API-клиенты)
- Командная разработка (несколько человек работают с одним кодом)
- Долгосрочные проекты

**Низкий ROI — тестировать опционально:**
- Прототипы и одноразовые скрипты
- UI-компоненты с чисто визуальной логикой (лучше — визуальное тестирование)
- Код, который скоро будет удалён

> 💡 **Правило:** Если вы боитесь менять код — ему нужны тесты. Если вам всё равно — тесты не нужны.

---

## Пирамида тестирования

Пирамида тестирования — модель, предложенная Майком Коном в 2009 году. Она описывает оптимальное соотношение тестов разных уровней.

```
            ╱╲
           ╱  ╲
          ╱ E2E╲              ← 10% — мало, дорого, медленно
         ╱──────╲
        ╱        ╲
       ╱Интеграц. ╲           ← 20% — средняя цена
      ╱────────────╲
     ╱              ╲
    ╱   Unit-тесты   ╲        ← 70% — много, дёшево, быстро
   ╱──────────────────╲
```

### Unit-тесты (модульные)

**Что тестируют:** Отдельные функции, утилиты, хуки — изолированно от остальных модулей.

**Характеристики:**
- Самые быстрые (тысячи тестов за секунды)
- Самые стабильные (не зависят от DOM, сети, браузера)
- Самые дешёвые в написании и поддержке

**Пример:**

```ts
import { calculateDiscount } from "./pricing";

describe("calculateDiscount", () => {
  it("returns 0 for non-premium users", () => {
    expect(calculateDiscount(100, false)).toBe(0);
  });

  it("returns 10% for premium users", () => {
    expect(calculateDiscount(100, true)).toBe(10);
  });

  it("returns 0 for negative prices", () => {
    expect(calculateDiscount(-50, true)).toBe(0);
  });

  it("handles zero price", () => {
    expect(calculateDiscount(0, true)).toBe(0);
  });
});
```

**Что подходит для unit-тестов:**
- Утилиты (форматирование, валидация, трансформация данных)
- Чистые функции (без побочных эффектов)
- Бизнес-логика (расчёты, правила, условия)
- Хуки (в изоляции, с моками зависимостей)

### Интеграционные тесты

**Что тестируют:** Взаимодействие между модулями — несколько функций, компонент + хук, хук + API.

**Характеристики:**
- Средние по скорости
- Проверяют, что модули работают вместе корректно
- Ловят ошибки контрактов между модулями

**Пример:**

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { SearchPage } from "./SearchPage";

describe("SearchPage", () => {
  it("fetches and displays results on submit", async () => {
    render(<SearchPage />);

    const user = userEvent.setup();
    await user.type(screen.getByRole("textbox", { name: /search/i }), "react");
    await user.click(screen.getByRole("button", { name: /search/i }));

    expect(await screen.findByText("React Documentation")).toBeInTheDocument();
    expect(screen.queryByText("No results")).not.toBeInTheDocument();
  });

  it("shows error message on API failure", async () => {
    // Переопределяем mock API для конкретного теста
    server.use(
      http.get("/api/search", () => HttpResponse.json(
        { error: "Server error" },
        { status: 500 }
      ))
    );

    render(<SearchPage />);

    const user = userEvent.setup();
    await user.type(screen.getByRole("textbox", { name: /search/i }), "react");
    await user.click(screen.getByRole("button", { name: /search/i }));

    expect(await screen.findByText(/server error/i)).toBeInTheDocument();
  });
});
```

**Что подходит для интеграционных тестов:**
- Компонент + контекст / провайдер
- Хук + API-запрос
- Форма + валидация + отправка
- Компонент + роутинг

### E2E-тесты (сквозные)

**Что тестируют:** Полные пользовательские сценарии в реальном браузере — от клика до результата.

**Характеристики:**
- Самые медленные (минуты на выполнение)
- Самые хрупкие (зависят от сети, UI, данных)
- Самые дорогие в написании и поддержке
- Дают наибольшую уверенность

**Пример:**

```ts
import { test, expect } from "@playwright/test";

test("user can complete checkout flow", async ({ page }) => {
  await page.goto("/products");
  await page.getByRole("link", { name: "React Handbook" }).click();
  await page.getByRole("button", { name: "Add to Cart" }).click();

  await page.goto("/cart");
  await expect(page.getByText("React Handbook")).toBeVisible();
  await expect(page.getByText("$49.99")).toBeVisible();

  await page.getByRole("button", { name: "Checkout" }).click();
  await page.fill('[name="email"]', "user@example.com");
  await page.fill('[name="card"]', "4242424242424242");
  await page.getByRole("button", { name: "Pay" }).click();

  await expect(page.getByText("Order confirmed")).toBeVisible();
});
```

**Что подходит для E2E-тестов:**
- Критические бизнес-сценарии (регистрация → покупка → оплата)
- Авторизация и защищённые маршруты
- Мультистраничные сценарии
- Smoke-тесты после деплоя

### Соотношение тестов

| Характеристика | Unit | Интеграционные | E2E |
|---|---|---|---|
| **Количество** | ~70% | ~20% | ~10% |
| **Скорость** | Миллисекунды | Сотни мс | Секунды-минуты |
| **Стабильность** | Высокая | Средняя | Низкая |
| **Стоимость написания** | Низкая | Средняя | Высокая |
| **Стоимость поддержки** | Низкая | Средняя | Высокая |
| **Уверенность** | Локальная | Средняя | Максимальная |
| **Обратная связь** | Мгновенная | Быстрая | Медленная |
| **Зависимости** | Моки | Реальные модули | Реальный браузер + API |

### Почему не наоборот

Распространённая ошибка — «перевёрнутый торт» (ice cream cone):

```
         ╱──────────╲
        ╱   E2E-тесты ╲       ← ❌ Слишком много E2E
       ╱────────────────╲
      ╱                  ╲
     ╱  Мало unit-тестов   ╲    ← ❌ Мало unit
    ╱────────────────────────╲
```

**Почему это плохо:**
- E2E-тесты медленные — CI выполняется часами
- E2E-тесты хрупкие — ломаются при любом изменении UI
- E2E-тесты дорогие — написание и поддержка требуют много ресурсов
- E2E-тесты плохо локализуют баги — падение может быть в любом месте стека

---

## Testing Trophy — альтернативная модель

Kent C. Dodds (автор React Testing Library) предложил альтернативную модель — **Testing Trophy**:

```
        ╱────────────╲
       ╱  E2E-тесты   ╲        ← Критические сценарии
      ╱────────────────╲
     ╱                  ╲
    ╱  Интеграционные    ╲     ← Основная масса тестов!
   ╱                      ╲
  ╱──── Unit-тесты ─────────╲  ← Утилиты и сложная логика
  ╱──── Статические ─────────╲  ← TypeScript, ESLint
 ╱────────────────────────────╲
```

### Ключевое отличие от пирамиды

| | Пирамида (Mike Cohn) | Testing Trophy (Kent C. Dodds) |
|---|---|---|
| **Основной уровень** | Unit-тесты | Интеграционные тесты |
| **Философия** | Тестировать изолированно | Тестировать взаимодействие |
| **Аргумент** | Unit-тесты быстрые и стабильные | Интеграционные тесты дают больше уверенности |
| **Компонентные тесты** | Относятся к unit | Выделены в отдельный уровень |

### Почему Kent C. Dodds прав для React

React-компоненты — это не чистые функции. Они зависят от:
- DOM (рендеринг)
- Контекста (тема, аутентификация)
- Хуков (состояние, эффекты)
- API (данные с сервера)

Тестирование компонента в изоляции (unit) часто проверяет моки, а не реальное поведение. Интеграционный тест — рендерит компонент с реальными провайдерами и проверяет поведение пользователя.

> 💡 **Вывод:** Для React-приложений интеграционные тесты компонентов дают больше уверенности, чем unit-тесты. Unit-тесты остаются полезны для утилит и чистой бизнес-логики.

### Практическое соотношение для React-проекта

```
Статические (TypeScript + ESLint)  ──── автоматическая защита
Unit-тесты (утилиты, хуки)         ──── ~20%
Интеграционные (компоненты)        ──── ~60%
E2E (критические сценарии)         ──── ~20%
```

---

## FIRST-принципы

FIRST — акроним, описывающий свойства хорошего теста.

### F — Fast (Быстрый)

Тесты должны выполняться быстро. Медленные тесты — демотивируют разработчиков и замедляют CI.

```
❌ Тест, который ходит в реальное API — 2 секунды
✅ Тест с моком — 5 миллисекунд
```

**Как обеспечить:**
- Мокировать внешние зависимости (API, БД, файловая система)
- Параллельный запуск тестов
- Не создавать реальные файлы/БД для каждого теста

### I — Isolated (Изолированный)

Тесты не должны зависеть друг от друга. Каждый тест выполняется в независимом окружении.

```ts
// ❌ Тесты зависят от порядка выполнения
let counter = 0;

it("increments counter", () => {
  counter++;
  expect(counter).toBe(1);
});

it("counter is 2 after two increments", () => {
  counter++;
  expect(counter).toBe(2); // ❌ Падает, если тесты запускаются в другом порядке
});

// ✅ Каждый тест независим
it("increments counter", () => {
  const store = createCounterStore();
  store.increment();
  expect(store.count).toBe(1);
});

it("starts from zero", () => {
  const store = createCounterStore();
  expect(store.count).toBe(0);
});
```

**Как обеспечить:**
- Каждый тест создаёт своё состояние
- `beforeEach` / `afterEach` для очистки
- `vi.clearAllMocks()` между тестами
- Не использовать глобальное изменяемое состояние

### R — Repeatable (Повторяемый)

Тест должен давать одинаковый результат при любом запуске — локально, в CI, на другом компьютере.

```
❌ Тест, который проходит локально, но падает в CI — flaky-тест
❌ Тест, который зависит от текущего времени — не повторяемый
✅ Тест с фиксированными данными и моками — повторяемый
```

**Источники flaky-тестов:**
- Зависимость от реального времени (используйте `vi.useFakeTimers()`)
- Зависимость от сети (используйте MSW)
- Зависимость от порядка тестов
- Гонки состояний (race conditions)
- Зависимость от состояния файловой системы

### S — Self-Validating (Самопроверяемый)

Тест должен сам определять, прошёл он или упал. Никакой ручной проверки.

```
❌ Тест выводит данные в консоль, и разработчик смотрит — правильно ли
✅ Тест содержит expect() и автоматически проходит/падает
```

```ts
// ❌ Нет assertions — тест всегда проходит
it("fetches user", async () => {
  const user = await fetchUser(1);
  console.log(user); // Разработчик смотрит глазами
});

// ✅ Самопроверяемый
it("fetches user", async () => {
  const user = await fetchUser(1);
  expect(user).toEqual({ id: 1, name: "Alice" });
});
```

### T — Timely (Своевременный)

Тесты должны писаться вовремя — в идеале до кода (TDD) или одновременно с ним.

```
❌ «Напишем тесты потом» → тесты не пишутся никогда
❌ Тесты пишутся через месяц после кода → тесты не покрывают edge cases
✅ Тесты пишутся вместе с кодом → покрытие растёт органично
```

---

## Паттерн AAA (Arrange-Act-Assert)

**AAA** — структура организации теста. Каждый тест состоит из трёх фаз:

```ts
it("adds item to cart", () => {
  // Arrange — подготовка данных и окружения
  const cart = createCart();
  const item = createItem({ id: 1, name: "Book", price: 29.99 });

  // Act — выполнение тестируемого действия
  addToCart(cart, item);

  // Assert — проверка результата
  expect(cart.items).toHaveLength(1);
  expect(cart.items[0].name).toBe("Book");
  expect(cart.total).toBe(29.99);
});
```

### Зачем разделять на AAA

- **Читаемость** — сразу видно, что подготавливается, что выполняется, что проверяется
- **Отладка** — если тест падает, понятно, на какой фазе проблема
- **Поддержка** — легко добавить новый тест по образцу

### Расширенный паттерн AAAA

Для тестов с очисткой добавляют четвёртую фазу:

```ts
it("subscribes to notifications", () => {
  // Arrange
  const user = createUser();
  const service = createNotificationService();

  // Act
  service.subscribe(user);

  // Assert
  expect(service.subscribers).toContain(user);

  // Cleanup (Arange-Act-Assert-Cleanup)
  service.unsubscribe(user);
});
```

> 💡 В Vitest/Jest очистку лучше делать в `afterEach` — это надёжнее, чем ручная очистка в каждом тесте.

### Пример для React-компонента

```tsx
it("disables submit button while saving", async () => {
  // Arrange
  const user = userEvent.setup();
  render(<ProfileForm />);

  const nameInput = screen.getByRole("textbox", { name: /name/i });
  const submitBtn = screen.getByRole("button", { name: /save/i });

  // Act
  await user.type(nameInput, "Alice");
  await user.click(submitBtn);

  // Assert — кнопка заблокирована во время сохранения
  expect(submitBtn).toBeDisabled();
  expect(screen.getByText(/saving/i)).toBeInTheDocument();

  // Assert — после завершения кнопка разблокирована
  expect(await screen.findByText(/saved/i)).toBeInTheDocument();
  expect(submitBtn).not.toBeDisabled();
});
```

---

## TDD: Red-Green-Refactor

**TDD (Test-Driven Development)** — методология разработки, при которой тесты пишутся **до** кода.

### Цикл TDD

```
  ┌──────────────────────────────────────────┐
  │                                          │
  ▼                                          │
┌──────┐    ┌───────┐    ┌──────────┐       │
│ RED  │───►│ GREEN │───►│ REFACTOR │───────┘
│      │    │       │    │          │
└──────┘    └───────┘    └──────────┘
```

### Red — написать падающий тест

Пишем тест, описывающий желаемое поведение. Тест падает, потому что код ещё не написан.

```ts
// Тест падает — функция formatPrice ещё не существует
it("formats price with currency", () => {
  expect(formatPrice(1234.5, "USD")).toBe("$1,234.50");
});
```

### Green — написать минимальный код

Пишем **минимальный** код, чтобы тест прошёл. Не оптимальный, не красивый — просто рабочий.

```ts
function formatPrice(price: number, currency: string): string {
  return "$1,234.50"; // Хардкод — лишь бы тест прошёл
}
```

### Refactor — улучшить код

Улучшаем код, сохраняя passing-тесты. Тесты — страховка при рефакторинге.

```ts
function formatPrice(price: number, currency: string): string {
  const symbols: Record<string, string> = { USD: "$", EUR: "€", RUB: "₽" };
  const symbol = symbols[currency] || currency;
  return `${symbol}${price.toLocaleString("en-US", {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  })}`;
}
```

### Когда использовать TDD

| Ситуация | TDD? |
|---|---|
| Новая утилита с чёткими правилами | ✅ Да |
| Сложная бизнес-логика | ✅ Да |
| Прототип / исследование | ❌ Нет |
| UI-компонент с неясным дизайном | ❌ Нет |
| Рефакторинг существующего кода | ⚠️ Сначала тесты на текущее поведение, потом рефакторинг |
| Исправление бага | ✅ Да (сначала тест, воспроизводящий баг, потом фикс) |

### Преимущества TDD

- Тесты покрывают 100% новой функциональности
- Код получается более тестируемым (вы пишете код, который можно тестировать)
- Меньше багов — каждый edge case продумывается до реализации
- Документация — тесты описывают контракт функции

### Недостатки TDD

- Замедляет начало разработки (но ускоряет в целом)
- Требует дисциплины и практики
- Не подходит для exploratory coding
- Избыточен для простых CRUD-операций

---

## Что тестировать, что не тестировать

### Что тестировать

| Категория | Примеры | Уровень |
|---|---|---|
| **Бизнес-логика** | Расчёт скидки, валидация формы, правила доступа | Unit |
| **Утилиты** | Форматирование даты, парсинг, трансформация данных | Unit |
| **Хуки** | useAuth, useCart, useDebounce | Unit / Интеграционный |
| **Компоненты** | Рендеринг, взаимодействие, формы, ошибки | Интеграционный |
| **Критические сценарии** | Регистрация, оплата, авторизация | E2E |
| **API-клиенты** | Обработка ответов, retry-логика, ошибки | Интеграционный |

### Что НЕ тестировать

| Категория | Почему |
|---|---|
| **Фреймворк** | React, Next.js, Zustand — они уже протестированы |
| **Третестные библиотеки** | Не тестируйте lodash, axios, zod |
| **Тривиальный код** | `const x = () => true` — не нуждается в тесте |
| **Приватные методы** | Тестируйте публичный API, не внутреннюю реализацию |
| **Конструкторы / конфигурацию** | `new Date()`, `{ theme: "dark" }` |

### Правило: тестируйте поведение, не реализацию

```ts
// ❌ Тестирует реализацию — хрупкий тест
it("calls setState and re-renders", () => {
  const setState = vi.spyOn(React, "useState");
  render(<Counter />);
  expect(setState).toHaveBeenCalled();
});

// ✅ Тестирует поведение — устойчивый тест
it("increments count on button click", async () => {
  const user = userEvent.setup();
  render(<Counter />);
  await user.click(screen.getByRole("button", { name: /increment/i }));
  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

---

## Покрытие кода (Code Coverage)

**Code Coverage** — метрика, показывающая, какая доля исходного кода выполняется во время тестов.

### Виды покрытия

| Метрика | Что измеряет |
|---|---|
| **Statements** | % исполненных выражений |
| **Branches** | % исполненных веток (if/else, switch/case) |
| **Functions** | % вызванных функций |
| **Lines** | % исполненных строк |

### Целевые значения

| Метрика | Цель | Комментарий |
|---|---|---|
| Statements | > 80% | Здоровый минимум для production-кода |
| Branches | > 70% | Ветвление — частый источник багов |
| Functions | > 85% | Каждая функция должна вызываться в тестах |
| Lines | > 80% | Близко к statements |

> ⚠️ **Не стремитесь к 100%.** Покрытие — это метрика, а не цель. 100% покрытие не гарантирует отсутствие багов. Лучше 80% осмысленного покрытия, чем 100% покрытия ради галочки.

### Как использовать покрытие

```
1. Настроить coverage в vitest.config.ts
2. Запускать тесты с флагом --coverage
3. Анализировать отчёт
4. Писать тесты для непокрытых критичных модулей
```

### Покрытие как порог в CI

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      thresholds: {
        statements: 80,
        branches: 70,
        functions: 85,
        lines: 80,
      },
    },
  },
});
```

Если покрытие падает ниже порога — CI падает. Это гарантирует, что качество тестов не деградирует.

---

## Классификация тестов по назначению

### Smoke-тесты

Быстрая проверка того, что основные функции работают. Выполняются после каждого деплоя.

```ts
test("homepage loads", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByRole("heading", { name: "Welcome" })).toBeVisible();
});

test("login page loads", async ({ page }) => {
  await page.goto("/login");
  await expect(page.getByRole("textbox", { name: /email/i })).toBeVisible();
});
```

### Regression-тесты

Тесты, написанные для конкретных исправленных багов. Гарантируют, что баг не вернётся.

```ts
// Баг: при пустой корзине итоговая сумма показывала NaN
test("shows zero total for empty cart", () => {
  const cart = createEmptyCart();
  expect(getTotal(cart)).toBe(0); // Не NaN
});
```

### Sanity-тесты

Узкие тесты, проверяющие конкретную функцию после изменений. Быстрее полного прогона.

### Acceptance-тесты

Проверяют, что функциональность соответствует требованиям заказчика. Обычно — E2E.

---

## Стоимость тестов

### Матрица стоимости

| | Написание | Выполнение | Поддержка | Локализация бага |
|---|---|---|---|---|
| **Unit** | Дёшево | Дёшево | Дёшево | Точная |
| **Интеграционные** | Средне | Средне | Средне | Хорошая |
| **E2E** | Дорого | Дорого | Дорого | Приблизительная |

### Закон стоимости

> Стоимость исправления бага растёт экспоненциально с каждой фазой, на которой он обнаружен:

```
Написан    → Исправить за 1 минуту
Unit-тест  → Исправить за 5 минут
CI         → Исправить за 30 минут
QA         → Исправить за 2 часа
Production → Исправить за 1 день + репутационные потери
```

---

## Лучшие практики

### 1. Тестируйте поведение, а не реализацию

Тест описывает **что** делает код, а не **как** он это делает.

```ts
// ❌ Зависит от реализации
it("calls useEffect and fetches data", () => {
  const useEffect = vi.spyOn(React, "useEffect");
  render(<UserList />);
  expect(useEffect).toHaveBeenCalled();
});

// ✅ Зависит от поведения
it("displays user list after loading", async () => {
  render(<UserList />);
  expect(await screen.findByText("Alice")).toBeInTheDocument();
  expect(await screen.findByText("Bob")).toBeInTheDocument();
});
```

### 2. Один тест — одна проверка

Каждый `it` проверяет одно конкретное поведение.

```ts
// ❌ Один тест проверяет слишком много
it("handles login", async () => {
  // Логин, переход в профиль, изменение настроек, выход
});

// ✅ Разбито на отдельные тесты
it("shows error for invalid credentials", async () => { ... });
it("redirects to dashboard after login", async () => { ... });
it("displays user name in header", async () => { ... });
```

### 3. Используйте factory-функции для тестовых данных

```ts
const createUser = (overrides?: Partial<User>): User => ({
  id: crypto.randomUUID(),
  name: "Test User",
  email: "test@example.com",
  role: "user",
  ...overrides,
});

// Использование
const admin = createUser({ role: "admin" });
const anonymous = createUser({ name: undefined });
```

### 4. Называйте тесты по шаблону «должен ... когда ...»

```ts
it("returns discounted price when user is premium", () => { ... });
it("throws error when price is negative", () => { ... });
it("shows loading spinner when data is fetching", () => { ... });
it("disables button when form is invalid", () => { ... });
```

### 5. Группируйте тесты по describe

```ts
describe("calculateDiscount", () => {
  describe("for premium users", () => {
    it("returns 10% discount", () => { ... });
    it("applies maximum discount cap", () => { ... });
  });

  describe("for regular users", () => {
    it("returns no discount", () => { ... });
  });

  describe("edge cases", () => {
    it("handles zero price", () => { ... });
    it("handles negative price", () => { ... });
    it("handles NaN input", () => { ... });
  });
});
```

### 6. Очищайте состояние между тестами

```ts
afterEach(() => {
  vi.clearAllMocks();
  vi.restoreAllMocks();
});
```

### 7. Используйте `beforeEach` для повторяющейся настройки

```ts
describe("UserProfile", () => {
  let user: User;

  beforeEach(() => {
    user = createUser({ name: "Alice" });
    server.use(
      http.get("/api/user", () => HttpResponse.json(user))
    );
  });

  it("displays user name", async () => {
    render(<UserProfile />);
    expect(await screen.findByText("Alice")).toBeInTheDocument();
  });

  it("shows edit button for own profile", async () => {
    render(<UserProfile isOwnProfile />);
    expect(screen.getByRole("button", { name: /edit/i })).toBeInTheDocument();
  });
});
```

---

## Антипаттерны

### 1. Тестирование деталей реализации

```ts
// ❌ Зависит от внутреннего state
expect(component.state.isOpen).toBe(true);

// ✅ Зависит от видимого поведения
expect(screen.getByRole("dialog")).toBeInTheDocument();
```

### 2. Зависимость тестов друг от друга

```ts
// ❌ Второй тест зависит от первого
it("creates user", () => {
  createUser("Alice");
  expect(getUsers()).toHaveLength(1);
});

it("finds user", () => {
  // ❌ Alice создана в предыдущем тесте!
  expect(findUser("Alice")).not.toBeNull();
});

// ✅ Каждый тест сам создаёт данные
it("finds user", () => {
  createUser("Alice");
  expect(findUser("Alice")).not.toBeNull();
});
```

### 3. Тестирование фреймворка

```ts
// ❌ Тестирует React, а не ваш код
it("calls useState", () => {
  const useState = vi.spyOn(React, "useState");
  render(<MyComponent />);
  expect(useState).toHaveBeenCalled();
});
```

### 4. Хрупкие селекторы

```ts
// ❌ Ломается при первом изменении разметки
expect(document.querySelector("div > div > span")).toHaveText("Hello");

// ✅ Устойчивый семантический селектор
expect(screen.getByRole("heading", { name: "Hello" })).toBeInTheDocument();
```

### 5. Snapshot-тестирование всего

```ts
// ❌ Snapshot ломается при каждом изменении разметки
expect(container).toMatchSnapshot();

// ✅ Snapshot только для стабильных компонентов
expect(iconElement).toMatchSnapshot();
```

### 6. Игнорирование async/await

```ts
// ❌ Синхронная проверка асинхронного результата
it("loads data", () => {
  render(<DataComponent />);
  expect(screen.getByText("Loaded")).toBeInTheDocument(); // FAIL — данные не загрузились
});

// ✅ Ожидание через findBy
it("loads data", async () => {
  render(<DataComponent />);
  expect(await screen.findByText("Loaded")).toBeInTheDocument();
});
```

### 7. Избыточное мокирование

```ts
// ❌ Мокирует всё — тест проверяет моки, а не код
vi.mock("./utils", () => ({ format: vi.fn().mockReturnValue("formatted") }));
vi.mock("./api", () => ({ fetch: vi.fn().mockReturnValue({ data: [] }) }));
vi.mock("./hooks", () => ({ useAuth: vi.fn().mockReturnValue({ user: {} }) }));

// ✅ Мокирует только внешние зависимости
// utils, hooks — реальные; API — мок (внешняя зависимость)
server.use(http.get("/api/data", () => HttpResponse.json({ data: [] })));
```

### 8. Тесты без assertions

```ts
// ❌ Нет expect — тест всегда проходит
it("renders component", () => {
  render(<MyComponent />);
  // Ничего не проверяется!
});

// ✅ Есть проверка
it("renders component", () => {
  render(<MyComponent />);
  expect(screen.getByText("Hello")).toBeInTheDocument();
});
```
