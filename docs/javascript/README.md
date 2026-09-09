# JavaScript Core

Раздел углубляет понимание языка: как JS хранит данные, как работает область видимости, замыкания, `this`, прототипы, Event Loop и продвинутые возможности. Ориентирован на собеседования и уверенную разработку.

## Начни здесь

Если ты только знакомишься с глубинами JavaScript или хочешь закрыть пробелы:

1. **[Типы данных и приведение типов](./js-data-types.md)** — примитивы и объекты, `typeof`, coercion, `==`/`===`, `Object.is`, `Number.isNaN`, собеседовательские ловушки.
2. **[Область видимости и hoisting](./js-scope-hoisting.md)** — `var`/`let`/`const`, TDZ, лексическое окружение, hoisting функций и классов, практические примеры.
3. **[Замыкания](./js-closures.md)** — определение и работа под капотом, приватные переменные, каррирование, IIFE, модули как замыкания, stale closures в React.
4. **[Контекст выполнения и `this`](./js-this.md)** — правила определения `this`, `call`/`apply`/`bind`, потеря контекста, стрелочные функции, `this` в классах и обработчиках событий React.

## Продвинутые темы

Когда база усвоена, переходи к мощным абстракциям:

- **[Прототипы и классы](./js-prototypes-classes.md)** — прототипное наследование, конструкторы и `prototype`, `class`, `extends`, приватные поля `#`, статические методы, `instanceof`, отличия от классов в Java/C++.
- **[Event Loop](./js-event-loop.md)** — call stack, Web API, task и microtask queue, `setTimeout(fn, 0)`, `queueMicrotask`, `requestAnimationFrame`, render queue, `process.nextTick`, визуализация Event Loop.
- **[Асинхронность: Promise, async/await](./js-async.md)** — состояния Promise, методы `Promise.all`/`race`/`allSettled`/`any`, `async/await`, последовательное и параллельное выполнение, обработка ошибок, `AbortController`.
- **[Модули](./js-modules.md)** — ES Modules и CommonJS, `import`/`export`, `require`/`module.exports`, динамический импорт, циклические зависимости, tree shaking и side effects.
- **[Коллекции, итераторы, генераторы](./js-collections-iterators.md)** — `Map`, `Set`, `WeakMap`, `WeakSet`, разница с объектом, `Symbol.iterator`, `for...of`, spread, деструктуризация, генераторы `function*`.
- **[Proxy и Reflect](./js-proxy-reflect.md)** — ловушки Proxy, `Reflect`, валидация, логирование, реактивность, ограничения Proxy, отзываемые прокси.
- **[Функциональное программирование](./js-functional.md)** — чистые функции, иммутабельность, методы массивов, каррирование, частичное применение, композиция, `debounce` и `throttle`.
- **[Память и производительность](./js-memory.md)** — стек и куча, сборка мусора, типичные утечки памяти, `WeakRef` и `FinalizationRegistry`, инструменты DevTools.

## Как пользоваться

1. Читай раздел «Начни здесь» по порядку — темы логически связаны.
2. После каждой статьи пройди чеклист вопросов в её конце.
3. Возвращайся к замыканиям, `this` и Event Loop перед собеседованиями.
4. Переходи к продвинутым темам, когда база усвоена.
