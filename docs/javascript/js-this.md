# Контекст выполнения и `this`

`this` — одна из самых запутанных концепций JavaScript, потому что его значение определяется **не местом определения функции, а способом её вызова**. Это фундаментально отличается от замыканий, где переменные привязаны к месту создания функции.

## Содержание

1. [Как определяется `this`](#как-определяется-this)
2. [Глобальный контекст и обычный вызов](#глобальный-контекст-и-обычный-вызов)
3. [Метод объекта](#метод-объекта)
4. [Конструктор](#конструктор)
5. [`call`, `apply`, `bind`](#call-apply-bind)
6. [Потеря `this`](#потеря-this)
7. [Стрелочные функции](#стрелочные-функции)
8. [`this` в классах](#this-в-классах)
9. [`this` в обработчиках событий React](#this-в-обработчиках-событий-react)
10. [Чеклист](#чеклист)

---

## Как определяется `this`

Значение `this` зависит от **контекста вызова**:

| Ситуация | Значение `this` |
| --- | --- |
| Глобальный контекст | `window` в браузере, `global` в Node.js |
| Обычный вызов функции | `window`/`global` в нестрогом режиме, `undefined` в строгом |
| Метод объекта | Сам объект |
| Конструктор (`new`) | Новый создаваемый объект |
| `call` / `apply` / `bind` | Явно переданный объект |
| Стрелочная функция | `this` из внешнего лексического окружения |

Важное правило: `this` вычисляется в момент вызова, а не в момент объявления функции.

---

## Глобальный контекст и обычный вызов

В глобальной области `this` указывает на глобальный объект:

```js
console.log(this); // window (браузер) или global (Node.js)
```

При обычном вызове функции поведение зависит от строгого режима:

```js
function example() {
  console.log(this);
}

example(); // window / global в нестрогом режиме
```

```js
'use strict';

function example() {
  console.log(this);
}

example(); // undefined
```

В строгом режиме (`"use strict"`) `this` не подменяется глобальным объектом, а остаётся `undefined`.

---

## Метод объекта

Когда функция вызывается как метод объекта, `this` указывает на объект перед точкой:

```js
const user = {
  name: 'Alice',
  greet() {
    console.log(`Hello, ${this.name}`);
  },
};

user.greet(); // "Hello, Alice"
```

Но стоит передать метод как callback, как `this` теряется:

```js
const greet = user.greet;
greet(); // "Hello, undefined" (или "Hello, " + window.name)
```

Теперь функция вызывается без контекста объекта, поэтому `this` указывает на глобальный объект или `undefined` в строгом режиме.

---

## Конструктор

При вызове функции с `new` создаётся новый объект, и `this` указывает на него:

```js
function Person(name) {
  this.name = name;
}

const alice = new Person('Alice');
console.log(alice.name); // "Alice"
```

Если забыть `new`, `this` будет указывать на глобальный объект (в нестрогом режиме), что приведёт к созданию глобальной переменной:

```js
const bob = Person('Bob');
console.log(bob);       // undefined
console.log(name);      // "Bob" — создано глобальное свойство!
```

В классах такая ошибка невозможна: если вызвать класс без `new`, будет ошибка `TypeError`.

---

## `call`, `apply`, `bind`

Эти методы позволяют явно управлять `this`.

### `call`

Вызывает функцию с указанным `this` и аргументами через запятую:

```js
function greet(greeting) {
  console.log(`${greeting}, ${this.name}`);
}

const user = { name: 'Alice' };
greet.call(user, 'Hello'); // "Hello, Alice"
```

### `apply`

Работает как `call`, но аргументы передаются массивом:

```js
greet.apply(user, ['Hi']); // "Hi, Alice"
```

### `bind`

Возвращает **новую функцию** с навсегда привязанным `this`:

```js
const greetUser = greet.bind(user);
greetUser('Hey'); // "Hey, Alice"
```

`bind` полезен, когда нужно передать метод как callback:

```js
setTimeout(user.greet.bind(user), 100); // "Hello, Alice"
```

---

## Потеря `this`

Самая частая ошибка — потеря контекста при передаче метода как callback:

```js
const user = {
  name: 'Alice',
  greet() {
    console.log(`Hello, ${this.name}`);
  },
};

setTimeout(user.greet, 1000); // "Hello, undefined"
```

Почему? `setTimeout` вызывает переданную функцию без контекста объекта. `user.greet` — это просто ссылка на функцию, объект теряется.

### Способы исправить

**1. Обернуть в стрелочную функцию:**

```js
setTimeout(() => user.greet(), 1000); // "Hello, Alice"
```

**2. Использовать `bind`:**

```js
setTimeout(user.greet.bind(user), 1000); // "Hello, Alice"
```

**3. Сохранить `this` в переменную (устаревший подход):**

```js
const self = this;
setTimeout(function () {
  self.greet();
}, 1000);
```

---

## Стрелочные функции

Стрелочные функции **не имеют собственного `this`**. Они берут `this` из внешнего лексического окружения в момент создания.

```js
const user = {
  name: 'Alice',
  greet: () => {
    console.log(`Hello, ${this.name}`);
  },
};

user.greet(); // "Hello, undefined"
```

Здесь стрелочная функция определена в глобальном контексте, поэтому `this` указывает на глобальный объект.

Стрелочные функции полезны внутри методов, когда нужно сохранить `this` из внешнего контекста:

```js
const user = {
  name: 'Alice',
  greetDelayed() {
    setTimeout(() => {
      console.log(`Hello, ${this.name}`); // this из greetDelayed
    }, 1000);
  },
};

user.greetDelayed(); // "Hello, Alice"
```

Внутри `setTimeout` стрелочная функция «захватывает" `this` из `greetDelayed`, который указывает на `user`.

### Разница между обычной и стрелочной функцией в методе

```js
const obj = {
  value: 42,
  regular() {
    console.log(this.value); // 42
  },
  arrow: () => {
    console.log(this.value); // undefined (this из внешнего окружения)
  },
};

obj.regular(); // 42
obj.arrow();   // undefined
```

Обычная функция получает `this` от способа вызова, стрелочная — от места создания.

---

## `this` в классах

### Обычные методы

В методах класса `this` указывает на экземпляр:

```js
class Person {
  constructor(name) {
    this.name = name;
  }

  greet() {
    console.log(`Hello, ${this.name}`);
  }
}

const alice = new Person('Alice');
alice.greet(); // "Hello, Alice"
```

### Потеря `this` в методах класса

Если передать метод класса как callback, `this` потеряется:

```js
const greet = alice.greet;
greet(); // TypeError: Cannot read properties of undefined
```

### Привязка в конструкторе

В классовых компонентах React методы обычно привязывали в конструкторе:

```js
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState({ count: this.state.count + 1 });
  }

  render() {
    return <button onClick={this.handleClick}>Click</button>;
  }
}
```

### Стрелочные методы классов

Современный способ — использовать стрелочную функцию как поле класса. Она не имеет собственного `this` и захватывает `this` экземпляра:

```js
class Counter extends React.Component {
  state = { count: 0 };

  handleClick = () => {
    this.setState({ count: this.state.count + 1 });
  };

  render() {
    return <button onClick={this.handleClick}>Click</button>;
  }
}
```

Это удобно, но создаёт новую функцию на каждом экземпляре, а не в прототипе.

---

## `this` в обработчиках событий React

### Функциональные компоненты

В функциональных компонентах проблема `this` отсутствует, потому что нет методов объекта. Но важно помнить о замыканиях:

```jsx
function User({ name }) {
  function handleClick() {
    console.log(name); // замыкание, а не this
  }

  return <button onClick={handleClick}>Show name</button>;
}
```

### Inline-стрелки

```jsx
function User({ name }) {
  return <button onClick={() => console.log(name)}>Show name</button>;
}
```

Inline-обработчики удобны, но создают новую функцию на каждом рендере. Для простых случаев это приемлемо, для сложных — лучше `useCallback`.

### Пример с `this` в классовом компоненте

```jsx
class Search extends React.Component {
  constructor(props) {
    super(props);
    this.handleSubmit = this.handleSubmit.bind(this);
  }

  handleSubmit(e) {
    e.preventDefault();
    console.log(this.props.query);
  }

  render() {
    return <form onSubmit={this.handleSubmit}>...</form>;
  }
}
```

Если не привязать `handleSubmit`, React вызовет его как обычную функцию, и `this` будет `undefined` в строгом режиме.

---

## Чеклист

- [ ] Как определяется `this` в разных случаях.
- [ ] Чем стрелочная функция отличается от обычной по `this`.
- [ ] Как исправить потерю `this` в `setTimeout`.
- [ ] Как работают `call`, `apply` и `bind`.
- [ ] Что происходит, если вызвать функцию-конструктор без `new`.
- [ ] Как `this` ведёт себя в методах классов.
- [ ] Почему в обработчиках событий React методы классов нужно привязывать.
- [ ] Почему стрелочная функция внутри метода сохраняет правильный `this`.
