# Модули

По мере роста приложения код перестаёт помещаться в один файл. **Модули** позволяют разбивать программу на изолированные части с явными зависимостями. В JavaScript существуют две основные системы модулей: **ES Modules (ESM)** — современный стандарт, и **CommonJS (CJS)** — классическая система Node.js. Понимание их различий важно для работы с фронтендом, бэкендом и сборщиками.

## Содержание

1. [Зачем нужны модули](#зачем-нужны-модули)
2. [ES Modules: import/export](#es-modules-importexport)
3. [CommonJS: require/module.exports](#commonjs-requiremoduleexports)
4. [Динамический импорт `import()`](#динамический-импорт-import)
5. [Циклические зависимости](#циклические-зависимости)
6. [Разница между ESM и CJS](#разница-между-esm-и-cjs)
7. [Tree shaking и side effects](#tree-shaking-и-side-effects)
8. [Практические задачи](#практические-задачи)
9. [Чеклист](#чеклист)

---

## Зачем нужны модули

Без модулей переменные из разных файлов попадают в глобальную область видимости и конфликтуют друг с другом. Модуль создаёт собственное лексическое окружение: переменные верхнего уровня видны только внутри модуля, если их явно не экспортировать.

```js
// utils.js
const secret = 'hidden'; // не экспортировано — доступно только внутри модуля
export const name = 'utils';
```

```js
// main.js
import { name } from './utils.js';
console.log(name);     // utils
console.log(secret);   // ReferenceError: secret is not defined
```

---

## ES Modules: import/export

ES Modules — стандарт, введённый в ES2015. Он используется в браузерах (через `<script type="module">`) и в Node.js (для файлов `.mjs` или при `"type": "module"` в `package.json`).

### Именованные экспорты

```js
// math.js
export const PI = 3.14159;

export function sum(a, b) {
  return a + b;
}

export class Calculator {
  // ...
}
```

```js
// main.js
import { PI, sum } from './math.js';

console.log(PI);      // 3.14159
console.log(sum(2, 3)); // 5
```

### Дефолтный экспорт

Один модуль может иметь только один `default`-экспорт.

```js
// logger.js
export default function log(message) {
  console.log(`[LOG] ${message}`);
}
```

```js
// main.js
import log from './logger.js';

log('Hello'); // [LOG] Hello
```

### Смешанный экспорт

```js
// api.js
export const BASE_URL = '/api';

export default function fetchUser(id) {
  return fetch(`${BASE_URL}/users/${id}`);
}
```

```js
import fetchUser, { BASE_URL } from './api.js';
```

### Переименование

```js
import { sum as add } from './math.js';
import * as math from './math.js';

console.log(add(1, 2));
console.log(math.PI);
```

### Re-export

```js
// index.js
export { sum, PI } from './math.js';
export { default as logger } from './logger.js';
```

---

## CommonJS: require/module.exports

CommonJS — система модулей Node.js. Она появилась раньше ES Modules и до сих пор широко используется.

```js
// math.js
const PI = 3.14159;

function sum(a, b) {
  return a + b;
}

module.exports = { PI, sum };
```

```js
// main.js
const { PI, sum } = require('./math.js');

console.log(PI);
console.log(sum(2, 3));
```

### Эквивалентные формы

```js
module.exports.foo = 1;
exports.bar = 2;

// Но перезапись exports не работает:
exports = { foo: 1 }; // module.exports не изменится
```

`exports` — это ссылка на `module.exports`. Если присвоить в `exports` новый объект, ссылка сломается.

---

## Динамический импорт `import()`

Динамический импорт возвращает Promise и позволяет загружать модули по условию или лениво.

```js
if (user.isAdmin) {
  const { adminPanel } = await import('./admin.js');
  adminPanel.init();
}
```

```js
// Ленивый роутинг в React
const LazyComponent = React.lazy(() => import('./HeavyComponent.jsx'));
```

Динамический импорт работает и в ESM, и в CJS-окружении (хотя в чистом CJS обычно используют `require` для динамики).

---

## Циклические зависимости

**Циклическая зависимость** возникает, когда модуль A импортирует модуль B, а B импортирует A.

```js
// a.js
import { b } from './b.js';
export const a = 'a';
console.log(b);
```

```js
// b.js
import { a } from './a.js';
export const b = 'b';
console.log(a);
```

### Поведение в ESM

При циклическом импорте ESM возвращает **частично инициализированный** модуль. В момент выполнения `b.js` модуль `a.js` ещё не завершил инициализацию, поэтому `a` будет `undefined`.

### Поведение в CJS

CJS кэширует `module.exports`. Если модуль ещё не завершился, `require` вернёт текущее состояние `module.exports` — обычно пустой объект `{}`.

### Как бороться

- Разбивать модули так, чтобы общий код вынести в третий модуль.
- Использовать динамический импорт внутри функции, а не на верхнем уровне.
- Пересматривать архитектуру: циклические зависимости часто сигнализируют о смешении ответственности.

```js
// вместо импорта на верхнем уровне
function init() {
  const { a } = require('./a.js'); // динамически, когда нужно
  return a;
}
```

---

## Разница между ESM и CJS

| Особенность | ES Modules | CommonJS |
|---|---|---|
| Синтаксис | `import` / `export` | `require` / `module.exports` |
| Выполнение | Статический анализ перед выполнением | Выполнение по мере вызова `require` |
| Топ-level await | Поддерживается | Не поддерживается |
| Strict mode | Всегда включён | Зависит от файла |
| Значения | Связь live binding (live bindings) | Копия значения на момент `require` |
| This в модуле | `undefined` | `module.exports` |
| Использование в браузере | Нативно через `<script type="module">` | Требует сборщика |

### Live bindings

В ESM импортированные переменные связаны с экспортом. Если экспорт изменится, импорт тоже изменится.

```js
// counter.js
export let count = 0;

export function increment() {
  count++;
}
```

```js
// main.js
import { count, increment } from './counter.js';

console.log(count); // 0
increment();
console.log(count); // 1
```

В CJS так не работает: `require` копирует значение.

```js
// counter.js
let count = 0;

function increment() {
  count++;
}

module.exports = { count, increment };
```

```js
const { count, increment } = require('./counter.js');

console.log(count); // 0
increment();
console.log(count); // 0 — копия не изменилась
```

Чтобы получить актуальное значение в CJS, экспортируйте getter:

```js
module.exports = {
  get count() { return count; },
  increment,
};
```

### Статический vs динамический

ESM анализирует зависимости до выполнения кода. Это позволяет сборщикам делать **tree shaking** — удалять неиспользуемый код.

CJS выполняется динамически: `require` можно вызывать внутри `if`, цикла или функции. Поэтому сборщикам сложнее определить, что реально используется.

---

## Tree shaking и side effects

**Tree shaking** — удаление неиспользуемого кода при сборке. Оно работает лучше с ESM, потому что `import`/`export` статические.

```js
// utils.js
export function used() { /* ... */ }
export function unused() { /* ... */ }
```

```js
// main.js
import { used } from './utils.js';
used();
```

Сборщик может удалить `unused`, если уверен, что её вызов не даёт **side effects** — побочных эффектов, таких как изменение глобального состояния, регистрация полифилов или инициализация библиотеки.

### Side effects

```js
// polyfill.js
if (!Array.prototype.toSorted) {
  Array.prototype.toSorted = function () { /* ... */ };
}
```

Такой файл нельзя удалять, даже если из него ничего не импортируется. В `package.json` библиотек указывают:

```json
{
  "sideEffects": ["./src/polyfill.js", "*.css"]
}
```

Если `sideEffects: false`, сборщик считает, что любой неиспользуемый экспорт можно удалить.

---

## Практические задачи

### Задача 1

Что выведет код?

```js
// a.js
import { b } from './b.js';
export const a = 'a';
console.log('a:', b);
```

```js
// b.js
import { a } from './a.js';
export const b = 'b';
console.log('b:', a);
```

**Ответ:** в ESM при циклическом импорте один из модулей получит неинициализированное значение. Вывод примерно такой:

```
b: undefined
a: b
```

Порядок зависит от того, какой модуль импортируется первым.

---

### Задача 2

Какой результат у `require` в CJS, если модуль ещё не завершил инициализацию?

**Ответ:** `require` вернёт текущее состояние `module.exports`, обычно пустой объект `{}`.

---

### Задача 3

Напишите модуль `counter` на ESM, который экспортирует `count` и `increment`, и покажите, почему `count` обновляется в импортёре.

**Решение:**

```js
// counter.js
export let count = 0;

export function increment() {
  count++;
}
```

```js
// main.js
import { count, increment } from './counter.js';

increment();
console.log(count); // 1
```

ESM поддерживает live bindings: импортированная переменная связана с оригиналом.

---

### Задача 4

Почему этот код не работает как ожидается в CJS?

```js
// counter.js
let count = 0;
module.exports = { count, increment: () => count++ };
```

```js
// main.js
const { count, increment } = require('./counter.js');
increment();
console.log(count);
```

**Ответ:** `count` копируется в момент `require`. Внутри модуля `count` увеличивается, но в импортёре остаётся старая копия. Вывод: `0`.

---

### Задача 5

Когда использовать динамический импорт?

**Ответ:**

- Ленивая загрузка кода (code splitting).
- Загрузка модулей по условию.
- Роутинг в SPA.
- Загрузка полифилов только при необходимости.

---

## Чеклист

- [ ] Чем ESM отличается от CommonJS.
- [ ] Как работают именованные и дефолтные экспорты в ESM.
- [ ] Почему в CJS `exports = {}` не заменяет `module.exports`.
- [ ] Как работает динамический импорт `import()`.
- [ ] Что такое циклическая зависимость и как с ней бороться.
- [ ] Что такое live bindings и почему они работают в ESM, но не в CJS.
- [ ] Что такое tree shaking и side effects.
- [ ] Когда стоит использовать ESM, а когда CJS.
