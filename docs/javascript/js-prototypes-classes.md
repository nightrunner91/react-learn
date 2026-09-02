# Прототипы и классы

JavaScript — язык с **прототипным наследованием**, а не классовым, как Java или C++. Даже синтаксис `class` из ES6 — это «синтаксический сахар» над прототипами. На собеседованиях часто спрашивают именно о том, что скрывается под этим сахаром: цепочки прототипов, конструкторы, `Object.create`, статические и приватные члены.

## Содержание

1. [Что такое прототип](#что-такое-прототип)
2. [Цепочка прототипов](#цепочка-прототипов)
3. [Object.create, setPrototypeOf, getPrototypeOf](#objectcreate-setprototypeof-getprototypeof)
4. [Конструкторы и `prototype`](#конструкторы-и-prototype)
5. [Наследование без `class`](#наследование-без-class)
6. [Синтаксис `class`](#синтаксис-class)
7. [`extends` и `super`](#extends-и-super)
8. [Приватные поля и методы `#`](#приватные-поля-и-методы-)
9. [Статические методы и свойства](#статические-методы-и-свойства)
10. [`instanceof` и его ограничения](#instanceof-и-его-ограничения)
11. [Чем `class` в JS отличается от классов в Java/C++](#чем-class-в-js-отличается-от-классов-в-javac)
12. [Чеклист](#чеклист)

---

## Что такое прототип

Каждый объект в JavaScript имеет скрытое внутреннее свойство `[[Prototype]]`. Оно ссылается на другой объект — **прототип**. Когда движок ищет свойство или метод, он сначала смотрит в самом объекте, затем в его прототипе, затем в прототипе прототипа — и так до `null`.

```js
const animal = {
  eats: true,
  walk() {
    console.log("I'm walking");
  }
};

const dog = {
  barks: true
};

Object.setPrototypeOf(dog, animal);

console.log(dog.eats); // true — найдено в прототипе
console.log(dog.barks); // true — найдено в самом объекте
dog.walk(); // "I'm walking" — найдено в прототипе
```

Если свойство не найдено ни в одном звене цепочки, возвращается `undefined`.

---

## Цепочка прототипов

Цепочка прототипов — это последовательность объектов, по которой движок ищет свойства. Она заканчивается `null`:

```js
const obj = {};
console.log(obj.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__); // null
```

`__proto__` — устаревший, но широко известный геттер/сеттер для `[[Prototype]]`. В современном коде лучше использовать `Object.getPrototypeOf` / `Object.setPrototypeOf`.

Важно: прототипное наследование — это **делегирование**, а не копирование. `dog` не содержит метод `walk`, он лишь **делегирует** его поиск прототипу `animal`.

---

## Object.create, setPrototypeOf, getPrototypeOf

`Object.create(proto)` создаёт новый объект с указанным прототипом:

```js
const animal = { eats: true };
const rabbit = Object.create(animal);

console.log(rabbit.eats); // true
console.log(Object.getPrototypeOf(rabbit) === animal); // true
```

Можно передать второй аргумент — описатели свойств:

```js
const user = Object.create(null, {
  name: { value: 'Alice', writable: true, enumerable: true }
});
```

`Object.setPrototypeOf` меняет прототип уже существующего объекта. Это медленная операция, поэтому менять прототип «на лету» не рекомендуется — лучше создать объект с нужным прототипом сразу.

`Object.getPrototypeOf` возвращает прототип объекта — безопасная альтернатива `__proto__`.

---

## Конструкторы и `prototype`

До появления `class` наследование делали через функции-конструкторы и свойство `prototype`:

```js
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  console.log(`Hello, ${this.name}`);
};

const alice = new Person('Alice');
alice.greet(); // "Hello, Alice"
```

Что происходит при `new Person('Alice')`:

1. Создаётся новый пустой объект.
2. Его `[[Prototype]]` устанавливается на `Person.prototype`.
3. Вызывается `Person` с `this`, указывающим на новый объект.
4. Возвращается этот объект.

Методы хранятся в `Person.prototype`, поэтому все экземпляры разделяют одну и ту же функцию — она не дублируется в памяти.

```js
console.log(alice.__proto__ === Person.prototype); // true
console.log(Person.prototype.constructor === Person); // true
```

---

## Наследование без `class`

Чтобы реализовать наследование вручную, нужно связать прототипы и вызвать родительский конструктор:

```js
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function () {
  console.log(`${this.name} makes a sound`);
};

function Dog(name, breed) {
  Animal.call(this, name); // вызываем родительский конструктор
  this.breed = breed;
}

Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // восстанавливаем constructor

Dog.prototype.bark = function () {
  console.log(`${this.name} barks`);
};

const rex = new Dog('Rex', ' shepherd');
rex.speak(); // "Rex makes a sound"
rex.bark(); // "Rex barks"
```

Почему именно `Object.create(Animal.prototype)`, а не `new Animal()`? Потому что `new Animal()` мог бы выполнить лишний код конструктора и создать ненужные побочные эффекты. `Object.create` лишь устанавливает прототип.

---

## Синтаксис `class`

`class` делает то же самое, но читабельнее:

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

console.log(typeof Person); // "function"
console.log(alice.__proto__ === Person.prototype); // true
```

Методы внутри `class` автоматически попадают в `Person.prototype`. Тело класса выполняется в строгом режиме (`use strict`).

Важное отличие: функции-конструкторы поднимаются полностью, а классы — нет. Класс остаётся в TDZ до строки объявления:

```js
const dog = new Dog(); // ReferenceError

class Dog {}
```

---

## `extends` и `super`

`extends` связывает прототипы, а `super` вызывает методы родителя:

```js
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(`${this.name} makes a sound`);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // обязательно до использования this
    this.breed = breed;
  }

  speak() {
    super.speak(); // вызов метода родителя
    console.log(`${this.name} barks`);
  }
}

const rex = new Dog('Rex', 'Shepherd');
rex.speak();
// "Rex makes a sound"
// "Rex barks"
```

В производном классе конструктор обязан вызвать `super()` до первого обращения к `this`. В непроизводном классе `this` создаётся автоматически.

---

## Приватные поля и методы `#`

Современный синтаксис позволяет объявлять настоящие приватные члены класса, недоступные снаружи:

```js
class BankAccount {
  #balance = 0;

  deposit(amount) {
    if (amount <= 0) throw new Error('Invalid amount');
    this.#balance += amount;
  }

  withdraw(amount) {
    if (amount > this.#balance) throw new Error('Insufficient funds');
    this.#balance -= amount;
  }

  getBalance() {
    return this.#balance;
  }
}

const account = new BankAccount();
account.deposit(1000);
console.log(account.getBalance()); // 1000
console.log(account.#balance); // SyntaxError: Private field must be declared
```

Приватные поля:

- доступны только внутри класса, где объявлены;
- не наследуются напрямую (потомок не может обратиться к `#balance` родителя);
- нельзя получить через `Reflect.ownKeys` или подобные методы;
- настоящие приватные члены, а не просто соглашение об именовании.

Можно объявлять и приватные методы:

```js
class User {
  #validateName(name) {
    return name.trim().length > 0;
  }

  constructor(name) {
    if (!this.#validateName(name)) throw new Error('Invalid name');
    this.name = name;
  }
}
```

---

## Статические методы и свойства

Статические члены принадлежат самому классу, а не его экземплярам. Они часто используются для фабрик, утилит и констант:

```js
class User {
  static role = 'guest';

  static createAdmin(name) {
    const user = new User(name);
    user.role = 'admin';
    return user;
  }

  constructor(name) {
    this.name = name;
  }
}

console.log(User.role); // "guest"
const admin = User.createAdmin('Alice');
console.log(admin.role); // "admin"
```

Статические методы наследуются. Внутри статического метода `this` ссылается на сам класс:

```js
class Animal {
  static isAnimal(obj) {
    return obj instanceof this;
  }
}

class Dog extends Animal {}

console.log(Dog.isAnimal(new Dog())); // true
```

---

## `instanceof` и его ограничения

`instanceof` проверяет, есть ли класс/функция-конструктор в цепочке прототипов объекта:

```js
class Animal {}
class Dog extends Animal {}

const rex = new Dog();

console.log(rex instanceof Dog); // true
console.log(rex instanceof Animal); // true
console.log(rex instanceof Object); // true
```

Под капотом `instanceof` использует `prototype`: `rex.__proto__.__proto__... === Dog.prototype`.

### Ограничения

1. **Не работает с примитивами.**

   ```js
   'hello' instanceof String; // false
   ```

   Для примитивов используйте `typeof` или `Object.prototype.toString.call`.

2. **Зависит от прототипа, который можно изменить.**

   ```js
   const arr = [];
   Object.setPrototypeOf(arr, null);
   console.log(arr instanceof Array); // false
   ```

3. **Не учитывает приватные поля.** `instanceof` говорит лишь о цепочке прототипов, а не о том, есть ли у объекта нужные внутренние поля.

4. **С объектами из других iframe/Window прототипы различаются.** `Array` из одного окна не равен `Array` из другого, поэтому `instanceof` может дать `false` для валидного массива.

В таких случаях надёжнее использовать `Array.isArray(obj)` или `Object.prototype.toString.call(obj)`.

---

## Чем `class` в JS отличается от классов в Java/C++

| Аспект | JavaScript | Java/C++ |
|--------|------------|----------|
| Модель наследования | Прототипная | Классовая |
| Класс — это | функция-конструктор + объект `prototype` | чертёг/тип во время компиляции |
| Методы | хранятся в `prototype`, общие для всех экземпляров | обычно часть каждого объекта или vtable |
| Множественное наследование | Нет, только цепочка прототипов | Есть (Java — интерфейсы, C++ — множественное) |
| Приватность | `#поля` (настоящая) или замыкания | модификаторы доступа |
| Расширение объектов | можно менять прототипы на лету | нельзя |

В JavaScript «класс» — это договорённость и удобный синтаксис. Под капотом всё равно работают прототипы.

---

## Чеклист

- [ ] Как работает цепочка прототипов.
- [ ] Чем `Object.create` отличается от `Object.setPrototypeOf`.
- [ ] Как реализовать наследование без `class` через `prototype`.
- [ ] Что происходит при вызове `new Constructor()`.
- [ ] Зачем нужен `super()` в производных классах.
- [ ] Как объявлять приватные поля и методы через `#`.
- [ ] Когда использовать статические методы и свойства.
- [ ] Как работает `instanceof` и в каких случаях он может обмануть.
- [ ] Чем `class` в JavaScript отличается от классов в Java/C++.
