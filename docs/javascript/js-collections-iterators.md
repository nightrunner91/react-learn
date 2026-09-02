# Коллекции, итераторы, генераторы

Помимо обычных объектов и массивов, в JavaScript есть специализированные коллекции — `Map`, `Set`, `WeakMap`, `WeakSet` — и механизм итераторов, который объединяет массивы, строки, `Map`, `Set` и пользовательские объекты под одним протоколом. Генераторы делают создание итераторов лаконичным. Эти темы регулярно встречаются на собеседованиях и в реальном коде: от уникализации массивов до реализации ленивых последовательностей.

## Содержание

1. [Object vs Map](#object-vs-map)
2. [Map](#map)
3. [Set](#set)
4. [WeakMap и WeakSet](#weakmap-и-weakset)
5. [Итераторы и `Symbol.iterator`](#итераторы-и-symboliterator)
6. [`for...of`, spread, деструктуризация](#forof-spread-деструктуризация)
7. [Генераторы](#генераторы)
8. [Практические задачи](#практические-задачи)
9. [Чеклист](#чеклист)

---

## Object vs Map

`Object` — универсальная ассоциативная структура, но у неё есть ограничения:

- Ключи приводятся к строке (или символу).
- Порядок перебора не гарантирован для числовых ключей.
- Нет удобных методов для работы с размером и проверкой наличия ключа.
- Наследует свойства от `Object.prototype`.

```js
const obj = {};
obj[1] = 'one';      // ключ станет строкой "1"
obj[{}] = 'object';  // ключ станет строкой "[object Object]"
```

`Map` создан специально для хранения пар ключ-значение и лишён этих недостатков.

| Особенность | Object | Map |
|---|---|---|
| Типы ключей | Строки, символы | Любые значения, включая объекты |
| Порядок вставки | Не гарантирован для чисел | Сохраняется |
| Размер | Вручную через `Object.keys` | `map.size` |
| Перебор | `for...in` с фильтрацией | `for...of`, `forEach` |
| Оптимизация | Универсальный объект | Оптимизирован под частое добавление/удаление |

---

## Map

`Map` хранит пары ключ-значение в порядке вставки. Ключом может быть любое значение.

```js
const user = { name: 'Alice' };
const visits = new Map();

visits.set(user, 5);
visits.set('homepage', true);
visits.set(42, 'answer');

console.log(visits.get(user));    // 5
console.log(visits.has('homepage')); // true
console.log(visits.size);         // 3

visits.delete(42);
visits.clear();
```

### Инициализация из массива пар

```js
const map = new Map([
  ['name', 'Alice'],
  ['age', 30],
]);

console.log(map.get('name')); // Alice
```

### Перебор Map

```js
const map = new Map([['a', 1], ['b', 2]]);

for (const [key, value] of map) {
  console.log(key, value);
}

map.forEach((value, key) => {
  console.log(key, value);
});

console.log([...map.keys()]);   // ['a', 'b']
console.log([...map.values()]); // [1, 2]
console.log([...map.entries()]); // [['a', 1], ['b', 2]]
```

### Когда использовать Map вместо объекта

- Ключи — не строки (например, объекты или DOM-элементы).
- Важен порядок вставки.
- Нужен частый перебор или изменение размера.
- Нужно избежать конфликтов с унаследованными свойствами.

---

## Set

`Set` — коллекция уникальных значений. Повторные добавления игнорируются.

```js
const set = new Set([1, 2, 2, 3, 3, 3]);

console.log(set); // Set { 1, 2, 3 }
console.log(set.size); // 3

set.add(4);
set.delete(2);
console.log(set.has(4)); // true
set.clear();
```

### Уникализация массива

Самый распространённый практический приём:

```js
const numbers = [1, 2, 2, 3, 4, 4, 5];
const unique = [...new Set(numbers)];

console.log(unique); // [1, 2, 3, 4, 5]
```

### Уникализация объектов

`Set` сравнивает значения через `SameValueZero` (похоже на `===`, но `NaN` равен `NaN`). Объекты сравниваются по ссылке:

```js
const a = { id: 1 };
const b = { id: 1 };

const set = new Set([a, a, b]);
console.log(set.size); // 2
```

Для уникализации объектов по содержимому используйте `Map` с ключом, вычисленным из данных, или `JSON.stringify` с осторожностью.

---

## WeakMap и WeakSet

`WeakMap` и `WeakSet` похожи на `Map` и `Set`, но хранят **слабые ссылки** на объекты. Если на объект больше нет сильных ссылок, сборщик мусора может удалить его вместе с записью в `WeakMap` / `WeakSet`.

### WeakMap

- Ключи могут быть только объектами.
- Нет перебора (`size`, `keys()`, `values()`, `forEach` отсутствуют).
- Нельзя очистить вручную (`clear` удалён из спецификации).

```js
const cache = new WeakMap();

let user = { name: 'Alice' };
cache.set(user, { visits: 5 });

console.log(cache.get(user)); // { visits: 5 }

user = null; // объект может быть собран сборщиком мусора
// запись в WeakMap исчезнет автоматически
```

### WeakSet

- Хранит только объекты.
- Используется, чтобы пометить объект, не удерживая его в памяти.

```js
const visited = new WeakSet();

let node = document.getElementById('app');
visited.add(node);

console.log(visited.has(node)); // true
```

### Зачем нужны WeakMap и WeakSet

- **Приватные данные объекта** — хранить метаданные без риска утечек памяти.
- **Кэширование** — автоматическая очистка, когда объект больше не нужен.
- **Пометки** — отслеживать, какие объекты уже обработаны, без владения ими.

---

## Итераторы и `Symbol.iterator`

**Итератор** — это объект с методом `next()`, который возвращает `{ value, done }`.

**Итерируемый объект** — это объект с методом `[Symbol.iterator]`, который возвращает итератор.

Массивы, строки, `Map`, `Set`, `TypedArray` — итерируемы по умолчанию.

```js
const arr = [1, 2, 3];
const iterator = arr[Symbol.iterator]();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

### Собственный итерируемый объект

```js
const range = {
  from: 1,
  to: 5,

  [Symbol.iterator]() {
    let current = this.from;
    return {
      next: () => {
        if (current <= this.to) {
          return { value: current++, done: false };
        }
        return { value: undefined, done: true };
      },
    };
  },
};

for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}
```

### Бесконечный итератор

```js
const fibonacci = {
  [Symbol.iterator]() {
    let a = 0;
    let b = 1;
    return {
      next: () => {
        const value = a;
        [a, b] = [b, a + b];
        return { value, done: false };
      },
    };
  },
};

const [first, second, third] = fibonacci;
console.log(first, second, third); // 0 1 1
```

---

## `for...of`, spread, деструктуризация

Все эти конструкции работают с итерируемыми объектами.

### `for...of`

```js
for (const char of 'abc') {
  console.log(char); // a, b, c
}

for (const [key, value] of new Map([['x', 1], ['y', 2]])) {
  console.log(key, value);
}
```

`for...of` отличается от `for...in`: первый перебирает значения итератора, второй — перечисляемые строковые ключи (включая унаследованные).

### Spread

```js
const set = new Set([1, 2, 3]);
const arr = [...set, 4, 5]; // [1, 2, 3, 4, 5]

const map = new Map([['a', 1], ['b', 2]]);
const obj = Object.fromEntries(map); // { a: 1, b: 2 }
```

### Деструктуризация

```js
const [first, second] = new Set([10, 20, 30]);
console.log(first, second); // 10 20

const [[key, value]] = new Map([['name', 'Alice']]);
console.log(key, value); // name Alice
```

---

## Генераторы

Генератор — это функция, объявленная с `function*`, которая может приостанавливать выполнение с помощью `yield` и возобновлять его при следующем вызове `next()`.

```js
function* counter() {
  let count = 0;
  while (true) {
    yield count++;
  }
}

const gen = counter();
console.log(gen.next().value); // 0
console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
```

Каждый вызов `next()` возвращает `{ value, done }`. Генераторы автоматически реализуют протокол итератора и итерируемого объекта.

### Генератор диапазона

```js
function* range(from, to) {
  for (let i = from; i <= to; i++) {
    yield i;
  }
}

for (const num of range(1, 5)) {
  console.log(num); // 1, 2, 3, 4, 5
}
```

### Двусторонняя связь через `next(value)`

В генератор можно передавать значения извне:

```js
function* dialog() {
  const name = yield 'Как вас зовут?';
  const age = yield `Привет, ${name}! Сколько вам лет?`;
  yield `${name}, вам ${age} лет.`;
}

const gen = dialog();
console.log(gen.next().value);         // 'Как вас зовут?'
console.log(gen.next('Alice').value);  // 'Привет, Alice! Сколько вам лет?'
console.log(gen.next(30).value);       // 'Alice, вам 30 лет.'
```

### Генераторы для ленивых вычислений

```js
function* powersOfTwo() {
  let n = 1;
  while (true) {
    yield n;
    n *= 2;
  }
}

const [a, b, c, d] = powersOfTwo();
console.log(a, b, c, d); // 1 2 4 8
```

### Где применяются генераторы

- Ленивые последовательности и бесконечные коллекции.
- Пользовательские итераторы (деревья, графы).
- Реализация корутин и конечных автоматов.
- Асинхронные генераторы `async function*` для потоков данных.

---

## Практические задачи

### Задача 1

Что выведет код?

```js
const map = new Map();
map.set({}, 1);
map.set({}, 1);

console.log(map.size);
```

**Ответ:** `2`. Каждый `{}` — это новый объект, поэтому они считаются разными ключами.

---

### Задача 2

Напишите функцию `uniqueBy(arr, key)`, которая возвращает массив объектов без дубликатов по указанному ключу.

**Решение:**

```js
function uniqueBy(arr, key) {
  const seen = new Set();
  return arr.filter((item) => {
    const value = item[key];
    if (seen.has(value)) return false;
    seen.add(value);
    return true;
  });
}

const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
  { id: 1, name: 'Clone' },
];

console.log(uniqueBy(users, 'id'));
// [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }]
```

---

### Задача 3

Что выведет код?

```js
const set = new Set([1, 2, 3, 4, NaN, NaN]);
console.log(set.size);
console.log(set.has(NaN));
```

**Ответ:** `5` и `true`. `Set` игнорирует дубликаты, включая `NaN === NaN` в его логике сравнения.

---

### Задача 4

Реализуйте итерируемый объект `countdown(from)`, который при переборе `for...of` выдаёт числа от `from` до 1.

**Решение:**

```js
const countdown = (from) => ({
  [Symbol.iterator]() {
    let current = from;
    return {
      next: () => {
        if (current > 0) {
          return { value: current--, done: false };
        }
        return { value: undefined, done: true };
      },
    };
  },
});

console.log([...countdown(5)]); // [5, 4, 3, 2, 1]
```

---

### Задача 5

Что выведет код?

```js
function* gen() {
  yield 1;
  yield 2;
  return 3;
}

const g = gen();
console.log([...g]);
```

**Ответ:** `[1, 2]`. Значение после `return` не включается в развёртку через spread, хотя оно доступно как `value` в последнем `{ value: 3, done: true }`.

---

## Чеклист

- [ ] Чем `Map` отличается от обычного объекта.
- [ ] Когда использовать `Map` вместо объекта.
- [ ] Как `Set` обеспечивает уникальность значений.
- [ ] Зачем нужны `WeakMap` и `WeakSet` и как они помогают избежать утечек памяти.
- [ ] Что такое итератор и итерируемый объект.
- [ ] Как работает `Symbol.iterator`.
- [ ] Как `for...of`, spread и деструктуризация используют итераторы.
- [ ] Что такое генератор и как он связан с итераторами.
- [ ] Как передать значение внутрь генератора через `next(value)`.
