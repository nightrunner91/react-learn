# Замыкания

Замыкание — это функция, которая «помнит» переменные из лексического окружения, в котором была создана, даже когда выполняется в другом месте. Это одна из самых важных концепций JavaScript: она объясняет работу хуков React, модулей, каррирования, фабрик и приватных переменных.

## Содержание

1. [Что такое замыкание](#что-такое-замыкание)
2. [Почему это работает](#почему-это-работает)
3. [Приватные переменные](#приватные-переменные)
4. [Каррирование и фабрики функций](#каррирование-и-фабрики-функций)
5. [Классическая ошибка с `var` в цикле](#классическая-ошибка-с-var-в-цикле)
6. [Модули как замыкания](#модули-как-замыкания)
7. [Замыкания и React-хуки](#замыкания-и-react-хуки)
8. [Stale closures](#stale-closures)
9. [Чеклист](#чеклист)

---

## Что такое замыкание

Замыкание возникает, когда функция сохраняет доступ к переменным внешней функции после того, как внешняя функция завершила выполнение.

```js
function createCounter() {
  let count = 0;

  return function () {
    count++;
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

Функция, возвращённая из `createCounter`, «замыкает» переменную `count`. Даже после завершения `createCounter` внутренняя функция имеет доступ к `count` и может её изменять.

Говоря коротко: **замыкание — это функция вместе со всеми внешними переменными, к которым она обращается.**

---

## Почему это работает

Когда функция вызывается, JavaScript создаёт **лексическое окружение** — внутреннюю структуру, которая хранит локальные переменные и ссылку на внешнее окружение. При поиске переменной движок идёт от текущего окружения к внешнему, пока не найдёт значение или не дойдёт до глобального.

**Замыкание возникает, когда внутренняя функция сохраняет ссылку на внешнее лексическое окружение, даже после того как внешняя функция завершилась.**

В примере с `createCounter`:

1. `createCounter()` создаёт окружение с `count = 0`.
2. Возвращаемая функция создаёт своё окружение, но сохраняет ссылку на родительское.
3. Когда `createCounter()` завершается, её окружение не удаляется, потому что на него всё ещё ссылается внутренняя функция.
4. Каждый вызов `counter()` обращается к одному и тому же `count`.

Если вызвать `createCounter()` несколько раз, создадутся **независимые** замыкания:

```js
const counterA = createCounter();
const counterB = createCounter();

console.log(counterA()); // 1
console.log(counterA()); // 2
console.log(counterB()); // 1
```

Каждый счётчик хранит собственное `count`.

---

## Приватные переменные

До появления приватных полей `#` в классах замыкания были основным способом сделать данные приватными:

```js
function createBankAccount(initialBalance) {
  let balance = initialBalance;

  return {
    deposit(amount) {
      balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount > balance) throw new Error('Insufficient funds');
      balance -= amount;
      return balance;
    },
    getBalance() {
      return balance;
    },
  };
}

const account = createBankAccount(1000);
account.deposit(500); // 1500
console.log(account.balance); // undefined — переменная скрыта
```

Переменная `balance` доступна только методам, созданным внутри `createBankAccount`. Снаружи её прочитать или изменить напрямую нельзя.

---

## Каррирование и фабрики функций

### Каррирование

Каррирование превращает функцию от нескольких аргументов в цепочку функций от одного аргумента. Каждая промежуточная функция замыкает предыдущий аргумент:

```js
function multiply(a) {
  return function (b) {
    return a * b;
  };
}

const double = multiply(2);
console.log(double(5));  // 10
console.log(double(10)); // 20
```

Функция `multiply(2)` возвращает новую функцию, которая «помнит», что `a = 2`.

### Фабрики функций

Фабрика создаёт функции с заранее заданным состоянием:

```js
function createLogger(prefix) {
  return function (message) {
    console.log(`[${prefix}] ${message}`);
  };
}

const infoLog = createLogger('INFO');
const errorLog = createLogger('ERROR');

infoLog('Server started');  // [INFO] Server started
errorLog('Connection failed'); // [ERROR] Connection failed
```

Каждый логгер хранит собственный `prefix` в замыкании.

---

## Классическая ошибка с `var` в цикле

На собеседованиях часто дают такой код:

```js
for (var i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i);
  }, 1000);
}
```

**Ответ:** пять раз `5`, а не `0, 1, 2, 3, 4`.

Почему? `var` имеет функциональную область видимости. Все пять функций `setTimeout` замыкают **одну и ту же** переменную `i`. К моменту выполнения таймеров цикл уже завершился, и `i = 5`.

**Решение 1 — `let`:**

```js
for (let i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i); // 0, 1, 2, 3, 4
  }, 1000);
}
```

`let` создаёт новую переменную для каждой итерации цикла.

**Решение 2 — IIFE:**

```js
for (var i = 0; i < 5; i++) {
  (function (j) {
    setTimeout(function () {
      console.log(j); // 0, 1, 2, 3, 4
    }, 1000);
  })(i);
}
```

IIFE (Immediately Invoked Function Expression) создаёт новое лексическое окружение на каждой итерации, и функция `setTimeout` замыкает `j`, а не `i`.

---

## Модули как замыкания

Модули ES (`import`/`export`) и старые модули CommonJS (`require`/`module.exports`) основаны на замыканиях. Каждый модуль выполняется в собственном лексическом окружении, поэтому переменные верхнего уровня модуля не попадают в глобальную область.

```js
// counter.js
let count = 0;

export function increment() {
  return ++count;
}

export function decrement() {
  return --count;
}
```

Переменная `count` скрыта внутри модуля. Все экспортированные функции разделяют одно и то же замыкание.

```js
// main.js
import { increment, decrement } from './counter.js';

console.log(increment()); // 1
console.log(increment()); // 2
console.log(decrement()); // 1
```

Это и есть приватное состояние модуля: оно доступно только через явно экспортированный интерфейс.

---

## Замыкания и React-хуки

В React вы используете замыкания каждый день, даже если не задумываетесь об этом:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setCount(count + 1);
    }, 1000);

    return () => clearInterval(timer);
  }, [count]);

  return <div>{count}</div>;
}
```

Функция внутри `setInterval` замыкает `count` из текущего рендера `Counter`. Это работает, но эффект пересоздаётся при каждом изменении `count`, потому что `count` указан в зависимостях.

Понимание замыканий помогает объяснить:

- почему массив зависимостей в `useEffect` важен;
- зачем нужны `useCallback` и `useMemo`;
- почему устаревшие замыкания (stale closures) — частый источник багов.

---

## Stale closures

**Stale closure** — это ситуация, когда функция использует устаревшее значение переменной из замыкания, потому что она не обновлялась при новом рендере.

### Пример с `useEffect`

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // count всегда равен 0 в этом замыкании
    }, 1000);

    return () => clearInterval(id);
  }, []); // эффект создаётся один раз

  return <div>{count}</div>;
}
```

Если убрать `count` из зависимостей, эффект создаётся один раз и замыкает `count = 0`. В итоге `setCount(count + 1)` всегда устанавливает `1`, а счётчик застрянет.

### Как исправить

**Вариант 1 — добавить зависимость:**

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => clearInterval(id);
}, [count]);
```

Эффект будет пересоздаваться при каждом изменении `count`.

**Вариант 2 — использовать функциональное обновление:**

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((prev) => prev + 1);
  }, 1000);

  return () => clearInterval(id);
}, []);
```

Функциональная форма `setCount` берёт актуальное значение из React, а не из замыкания, поэтому зависимость от `count` больше не нужна.

### Пример с `useCallback`

```jsx
function Search({ query }) {
  const handleSearch = useCallback(() => {
    fetch(`/api/search?q=${query}`);
  }, []); // stale closure: query устареет

  useEffect(() => {
    handleSearch();
  }, [handleSearch]);
}
```

Если `query` изменится, `handleSearch` всё равно будет использовать старое значение. Правильно:

```jsx
const handleSearch = useCallback(() => {
  fetch(`/api/search?q=${query}`);
}, [query]);
```

Важно указывать в зависимостях все значения, которые используются внутри замыкания.

---

## Чеклист

- [ ] Что такое замыкание и как оно работает под капотом.
- [ ] Почему внутренняя функция сохраняет доступ к переменным внешней функции.
- [ ] Как создать приватные переменные с помощью замыканий.
- [ ] Чем отличается каррирование от обычной функции.
- [ ] Как исправить вывод `setTimeout` в цикле с `var`.
- [ ] Почему модули можно рассматривать как замыкания.
- [ ] Как замыкания связаны с React-хуками.
- [ ] Что такое stale closure и как его избежать.
