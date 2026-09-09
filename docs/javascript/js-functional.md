# Функциональное программирование

Функциональное программирование (ФП) — это парадигма, в которой вычисления строятся из функций, а не из изменяемого состояния. JavaScript не является чисто функциональным языком, но заимствует у ФП много полезных техник: чистые функции, иммутабельность, методы массивов, каррирование, композицию. Эти приёмы делают код предсказуемее, проще для тестирования и удобнее для поддержки.

## Содержание

1. [Чистые функции](#чистые-функции)
2. [Иммутабельность](#иммутабельность)
3. [Методы массивов](#методы-массивов)
4. [Каррирование и частичное применение](#каррирование-и-частичное-применение)
5. [Композиция функций](#композиция-функций)
6. [`debounce` и `throttle`](#debounce-и-throttle)
7. [Чеклист](#чеклист)

---

## Чистые функции

**Чистая функция** — это функция, которая:

1. Для одних и тех же аргументов всегда возвращает один и тот же результат.
2. Не имеет побочных эффектов: не изменяет внешнее состояние, не делает сетевых запросов, не пишет в DOM, не использует `Math.random()` и текущее время.

```js
// чистая функция
function add(a, b) {
  return a + b;
}

// нечистая функция: зависит от внешнего состояния
let tax = 0.2;
function calculatePrice(price) {
  return price * (1 + tax);
}

// нечистая функция: побочный эффект
function logAndDouble(x) {
  console.log(x);
  return x * 2;
}
```

Преимущества чистых функций:

- их легко тестировать — достаточно проверить вход и выход;
- их легко распараллеливать и кэшировать (memoization);
- их поведение предсказуемо и не зависит от порядка вызовов.

В реальном коде полностью избежать побочных эффектов невозможно: рендеринг, запросы к серверу, работа с DOM — всё это эффекты. Важно **изолировать** их и не смешивать с вычислениями.

---

## Иммутабельность

**Иммутабельность** — это отказ от изменения существующих данных. Вместо изменения объекта или массива создаётся новый объект на основе старого.

```js
const numbers = [1, 2, 3];

// мутация исходного массива
numbers.push(4);

// иммутабельный подход
const nextNumbers = [...numbers, 4];
```

Почему это важно:

- иммутабельные данные безопаснее в многопоточных/асинхронных сценариях;
- их удобно сравнивать по ссылке (React использует это для `React.memo` и `useEffect`);
- история изменений сохраняется автоматически.

```js
const user = { name: 'Alice', age: 25 };

// неправильно: мутация
user.age = 26;

// правильно: новый объект
const updatedUser = { ...user, age: 26 };
```

Для глубокой иммутабельности и сложных структур используют библиотеки вроде Immer или Immutable.js. В простых случаях хватает spread, `map`, `filter` и `reduce`.

---

## Методы массивов

Методы массивов в JavaScript — основной инструмент функционального стиля. Они не мутируют исходный массив, а возвращают новое значение.

### `map`

Преобразует каждый элемент массива и возвращает новый массив той же длины.

```js
const numbers = [1, 2, 3];
const doubled = numbers.map((n) => n * 2);

console.log(doubled); // [2, 4, 6]
console.log(numbers); // [1, 2, 3]
```

### `filter`

Возвращает новый массив, оставляя только элементы, прошедшие проверку.

```js
const numbers = [1, 2, 3, 4, 5];
const evens = numbers.filter((n) => n % 2 === 0);

console.log(evens); // [2, 4]
```

### `reduce`

Сворачивает массив к одному значению. Принимает аккумулятор и текущий элемент.

```js
const numbers = [1, 2, 3, 4];
const sum = numbers.reduce((acc, n) => acc + n, 0);

console.log(sum); // 10
```

`reduce` можно использовать не только для чисел:

```js
const people = [
  { name: 'Alice', age: 25 },
  { name: 'Bob', age: 30 },
];

const byName = people.reduce((acc, person) => {
  acc[person.name] = person;
  return acc;
}, {});

console.log(byName);
// { Alice: { name: 'Alice', age: 25 }, Bob: { name: 'Bob', age: 30 } }
```

### `find`, `some`, `every`

```js
const users = [
  { name: 'Alice', admin: true },
  { name: 'Bob', admin: false },
];

const admin = users.find((u) => u.admin);
console.log(admin.name); // Alice

const hasAdmins = users.some((u) => u.admin);
console.log(hasAdmins); // true

const allAdmins = users.every((u) => u.admin);
console.log(allAdmins); // false
```

### Сравнение методов

| Метод | Что возвращает | Когда использовать |
|---|---|---|
| `map` | Новый массив той же длины | Преобразовать каждый элемент |
| `filter` | Новый массив, меньшей или равной длины | Оставить элементы по условию |
| `reduce` | Одно значение | Свернуть массив: сумма, объект, группировка |
| `find` | Первый подходящий элемент или `undefined` | Найти один элемент |
| `some` | `true`, если хотя бы один подходит | Проверить наличие |
| `every` | `true`, если все подходят | Проверить все |

---

## Каррирование и частичное применение

**Каррирование** — преобразование функции от нескольких аргументов в цепочку функций от одного аргумента.

```js
function add(a) {
  return function (b) {
    return a + b;
  };
}

const addFive = add(5);
console.log(addFive(3)); // 8
console.log(add(2)(3));  // 5
```

В современном синтаксисе:

```js
const multiply = (a) => (b) => a * b;
const double = multiply(2);

console.log(double(7)); // 14
```

**Частичное применение** — фиксация части аргументов функции заранее.

```js
function greet(greeting, name) {
  return `${greeting}, ${name}!`;
}

const sayHello = greet.bind(null, 'Hello');
console.log(sayHello('Alice')); // Hello, Alice!
```

Или через стрелки:

```js
const greet = (greeting) => (name) => `${greeting}, ${name}!`;
const sayHi = greet('Hi');
console.log(sayHi('Bob')); // Hi, Bob!
```

Каррирование полезно, когда нужно создавать специализированные функции на основе общих.

---

## Композиция функций

**Композиция** — объединение нескольких простых функций в одну, где результат одной передаётся в следующую.

```js
const trim = (s) => s.trim();
const toUpper = (s) => s.toUpperCase();
const exclaim = (s) => `${s}!`;

const shout = (s) => exclaim(toUpper(trim(s)));
console.log(shout('  hello  ')); // HELLO!
```

Для удобства пишут универсальную функцию композиции:

```js
const compose = (...fns) => (x) =>
  fns.reduceRight((acc, fn) => fn(acc), x);

const shout = compose(exclaim, toUpper, trim);
console.log(shout('  hello  ')); // HELLO!
```

Если читать функции слева направо, используют `pipe`:

```js
const pipe = (...fns) => (x) =>
  fns.reduce((acc, fn) => fn(acc), x);

const shout = pipe(trim, toUpper, exclaim);
console.log(shout('  hello  ')); // HELLO!
```

Композиция помогает строить сложную логику из маленьких независимых функций.

---

## `debounce` и `throttle`

Эти два приёма ограничивают частоту вызова функций — например, при обработке событий `scroll`, `resize` или `input`.

### `debounce`

**Debounce** откладывает выполнение функции до тех пор, пока не пройдёт заданное время с момента последнего вызова.

```js
function debounce(fn, delay) {
  let timerId;

  return function (...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => fn.apply(this, args), delay);
  };
}

const handleInput = debounce((value) => {
  console.log('Search:', value);
}, 300);

// handleInput('a');
// handleInput('ab');
// handleInput('abc'); // выполнится только этот вызов, спустя 300 мс после него
```

Применение: поиск при вводе текста, валидация формы после окончания ввода.

### `throttle`

**Throttle** гарантирует, что функция выполняется не чаще, чем раз в заданный интервал.

```js
function throttle(fn, interval) {
  let lastTime = 0;

  return function (...args) {
    const now = Date.now();

    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

const handleScroll = throttle(() => {
  console.log('Scroll position:', window.scrollY);
}, 200);

window.addEventListener('scroll', handleScroll);
```

Применение: обработка скролла, ресайза, отслеживание координат мыши.

### Сравнение

| Приём | Когда вызывает функцию | Пример использования |
|---|---|---|
| `debounce` | После паузы между вызовами | Поиск по вводу |
| `throttle` | Регулярно, не чаще интервала | Обработка скролла |

---

## Чеклист

- [ ] Что такое чистая функция и почему она предсказуема.
- [ ] Почему иммутабельность важна во фронтенде и в React.
- [ ] Как работает `reduce` и для каких задач он подходит.
- [ ] Чем `map` отличается от `forEach`.
- [ ] Что такое каррирование и частичное применение.
- [ ] Чем `compose` отличается от `pipe`.
- [ ] Чем `debounce` отличается от `throttle` и когда какой использовать.
- [ ] Как написать иммутабельное обновление объекта или массива.
