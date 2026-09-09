# Proxy и Reflect

`Proxy` позволяет перехватывать и переопределять базовые операции над объектами: чтение свойств, запись, удаление, вызов функций, создание через `new` и другие. `Reflect` — это встроенный объект с методами, которые дублируют поведение этих операций и упрощают делегирование оригинальному поведению из ловушек Proxy. Вместе они дают мощный инструмент для метапрограммирования, валидации, логирования и реактивности. Proxy активно используется во фреймворках: именно на нём построена реактивность Vue 3.

## Содержание

1. [Что такое Proxy](#что-такое-proxy)
2. [Ловушки `get` и `set`](#ловушки-get-и-set)
3. [Другие ловушки](#другие-ловушки)
4. [Reflect](#reflect)
5. [Практические примеры](#практические-примеры)
6. [Proxy в реальных фреймворках: Vue 3](#proxy-в-реальных-фреймворках-vue-3)
7. [Ограничения Proxy](#ограничения-proxy)
8. [Чеклист](#чеклист)

---

## Что такое Proxy

`Proxy` оборачивает целевой объект и перехватывает операции через **ловушки** (traps) — функции-обработчики.

```js
const target = { name: 'Alice', age: 30 };

const handler = {
  get(target, prop) {
    console.log(`Reading ${prop}`);
    return target[prop];
  },
  set(target, prop, value) {
    console.log(`Setting ${prop} to ${value}`);
    target[prop] = value;
    return true;
  },
};

const proxy = new Proxy(target, handler);

console.log(proxy.name); // Reading name → Alice
proxy.age = 31;          // Setting age to 31
```

Если для операции нет ловушки, она выполняется напрямую над целевым объектом.

---

## Ловушки `get` и `set`

Самые частые ловушки — чтение и запись свойств.

### Валидация через `set`

```js
const user = {
  name: 'Alice',
  age: 30,
};

const validator = new Proxy(user, {
  set(target, prop, value) {
    if (prop === 'age' && (typeof value !== 'number' || value < 0)) {
      throw new TypeError('Age must be a positive number');
    }
    target[prop] = value;
    return true;
  },
});

validator.age = 31; // OK
validator.age = -1; // TypeError
```

### Значения по умолчанию через `get`

```js
const defaults = { theme: 'light' };

const settings = new Proxy(defaults, {
  get(target, prop) {
    if (prop in target) {
      return target[prop];
    }
    console.warn(`Unknown setting: ${prop}`);
    return null;
  },
});

console.log(settings.theme);   // 'light'
console.log(settings.language); // warning → null
```

---

## Другие ловушки

| Ловушка | Перехватывает |
|---|---|
| `get` | Чтение свойства |
| `set` | Запись свойства |
| `has` | Оператор `in` |
| `deleteProperty` | Оператор `delete` |
| `ownKeys` | `Object.keys`, `for...in`, `Object.getOwnPropertyNames` |
| `getOwnPropertyDescriptor` | `Object.getOwnPropertyDescriptor` |
| `defineProperty` | `Object.defineProperty` |
| `apply` | Вызов функции |
| `construct` | Вызов через `new` |
| `getPrototypeOf` / `setPrototypeOf` | Работу с прототипом |

### `has` и `deleteProperty`

```js
const data = { public: 1, _private: 2 };

const proxy = new Proxy(data, {
  has(target, prop) {
    if (String(prop).startsWith('_')) return false;
    return prop in target;
  },
  deleteProperty(target, prop) {
    if (String(prop).startsWith('_')) {
      throw new Error('Cannot delete private property');
    }
    delete target[prop];
    return true;
  },
});

console.log('public' in proxy);  // true
console.log('_private' in proxy); // false
delete proxy.public;             // OK
// delete proxy._private;        // Error
```

### `ownKeys`

```js
const obj = { a: 1, _b: 2, c: 3 };

const proxy = new Proxy(obj, {
  ownKeys(target) {
    return Object.keys(target).filter((key) => !key.startsWith('_'));
  },
});

console.log(Object.keys(proxy)); // ['a', 'c']
```

### `apply`

```js
function sum(a, b) {
  return a + b;
}

const loggedSum = new Proxy(sum, {
  apply(target, thisArg, args) {
    console.log(`Called with ${args}`);
    return target.apply(thisArg, args);
  },
});

console.log(loggedSum(2, 3)); // Called with 2,3 → 5
```

### `construct`

```js
class User {
  constructor(name) {
    this.name = name;
  }
}

const LoggedUser = new Proxy(User, {
  construct(target, args) {
    console.log(`Creating user ${args[0]}`);
    return new target(...args);
  },
});

const user = new LoggedUser('Alice'); // Creating user Alice
```

---

## Reflect

`Reflect` — это набор методов, которые повторяют поведение внутренних операций движка. Они полезны в ловушках Proxy, потому что позволяют вызвать стандартное поведение, не дублируя его вручную.

```js
const user = { name: 'Alice' };

const proxy = new Proxy(user, {
  get(target, prop, receiver) {
    console.log(`Reading ${prop}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    console.log(`Setting ${prop}`);
    return Reflect.set(target, prop, value, receiver);
  },
});

proxy.name = 'Bob';
console.log(proxy.name);
```

### Зачем нужен Reflect

- **Корректная передача `receiver`**. При работе с прототипами и getter/setter важно, чтобы `this` указывал на правильный объект.
- **Возврат булевых значений**. `Reflect.set` возвращает `true`/`false`, а прямое присваивание может молча не сработать, если свойство read-only.
- **Единообразие**. Методы `Reflect` соответствуют ловушкам Proxy один к одному.

### Пример с `receiver`

```js
const parent = {
  _name: 'Alice',
  get name() {
    return this._name;
  },
};

const child = {
  __proto__: parent,
  _name: 'Bob',
};

const proxy = new Proxy(parent, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);
  },
});

console.log(child.name);       // Bob — getter вызван с this = child
console.log(proxy.name);       // Alice — getter вызван с this = parent
```

Если бы вместо `Reflect.get` использовалось `target[prop]`, getter потерял бы корректный `this`.

---

## Практические примеры

### Логирование доступа к объекту

```js
function createLoggedObject(target) {
  return new Proxy(target, {
    get(target, prop) {
      console.log(`GET ${String(prop)}`);
      return Reflect.get(...arguments);
    },
    set(target, prop, value) {
      console.log(`SET ${String(prop)} = ${value}`);
      return Reflect.set(...arguments);
    },
  });
}

const user = createLoggedObject({ name: 'Alice' });
user.name;
user.age = 30;
```

### Реактивность (упрощённая)

```js
function createReactive(target, onChange) {
  return new Proxy(target, {
    set(target, prop, value) {
      const oldValue = target[prop];
      const result = Reflect.set(target, prop, value);
      if (oldValue !== value) {
        onChange(prop, value, oldValue);
      }
      return result;
    },
  });
}

const state = createReactive({ count: 0 }, (prop, value) => {
  console.log(`${prop} changed to ${value}`);
});

state.count++; // count changed to 1
```

### Мемоизация методов

```js
function memoize(fn) {
  const cache = new Map();

  return new Proxy(fn, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);
      if (cache.has(key)) return cache.get(key);

      const result = target.apply(thisArg, args);
      cache.set(key, result);
      return result;
    },
  });
}

const fib = memoize(function fib(n) {
  return n <= 1 ? n : fib(n - 1) + fib(n - 2);
});

console.log(fib(40));
```

---

## Proxy в реальных фреймворках: Vue 3

Во Vue 3 реактивность построена на `Proxy`. Это ключевое отличие от Vue 2, где использовался `Object.defineProperty`.

### Почему именно Proxy

`Object.defineProperty` мог отслеживать только уже существующие свойства объекта. У него были ограничения:

- Добавление нового свойства в объект не отслеживалось автоматически (`Vue.set` был необходим).
- Изменение элементов массива по индексу требовало специальной обработки.
- Удаление свойства через `delete` не отслеживалось.

`Proxy` перехватывает все операции над объектом, включая `get`, `set`, `deleteProperty`, `has` и `ownKeys`. Поэтому Vue 3 может реагировать на любое изменение без дополнительных методов вроде `Vue.set`.

### Упрощённая реализация реактивности, похожая на Vue 3

```js
const targetMap = new WeakMap();

function track(target, prop) {
  if (!activeEffect) return;

  let depsMap = targetMap.get(target);
  if (!depsMap) {
    depsMap = new Map();
    targetMap.set(target, depsMap);
  }

  let dep = depsMap.get(prop);
  if (!dep) {
    dep = new Set();
    depsMap.set(prop, dep);
  }

  dep.add(activeEffect);
}

function trigger(target, prop) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;

  const dep = depsMap.get(prop);
  if (dep) {
    dep.forEach((effect) => effect());
  }
}

let activeEffect = null;

function watchEffect(effect) {
  activeEffect = effect;
  effect();
  activeEffect = null;
}

function reactive(target) {
  return new Proxy(target, {
    get(target, prop, receiver) {
      track(target, prop);
      return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
      const oldValue = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      if (oldValue !== value) {
        trigger(target, prop);
      }
      return result;
    },
    deleteProperty(target, prop) {
      const hadKey = prop in target;
      const result = Reflect.deleteProperty(target, prop);
      if (hadKey) {
        trigger(target, prop);
      }
      return result;
    },
  });
}
```

### Как это работает

```js
const state = reactive({ count: 0 });

watchEffect(() => {
  console.log('count =', state.count);
});

state.count++; // count = 1
state.count++; // count = 2
```

1. При первом запуске `watchEffect` читает `state.count`.
2. Ловушка `get` вызывает `track`, которая сохраняет текущий эффект в `targetMap`.
3. Когда `state.count` изменяется, ловушка `set` вызывает `trigger`.
4. `trigger` находит все эффекты, зависящие от `count`, и перезапускает их.

### Что отслеживает Vue 3 через Proxy

- Чтение и запись свойств объекта.
- Добавление новых свойств.
- Удаление свойств.
- Итерацию (`for...in`, `Object.keys`).
- Операции с `Map`, `Set`, `WeakMap`, `WeakSet`.

### Важный нюанс

Vue 3 не делает реактивными вложенные объекты заранее полностью. Он оборачивает их в `Proxy` при первом доступе — это называется **ленивая реактивность**. Также примитивные значения оборачиваются в `ref`, который внутри тоже использует `get`/`set` для доступа к `.value`.

---

## Ограничения Proxy

1. **Только объект.** `Proxy` нельзя создать над примитивом.

   ```js
   new Proxy(42, {}); // TypeError
   ```

2. **Нет прозрачности для операторов.** `typeof proxy` вернёт `'object'` даже для функции-прокси; `proxy === target` — `false`.

3. **Некоторые операции не перехватываются.** Например, строгое равенство `===`, `instanceof` (можно перехватить через `Symbol.hasInstance` отдельно), сравнение примитивов.

4. **Производительность.** Proxy медленнее прямого доступа к объекту. Не стоит оборачивать всё подряд.

5. **Не клонирует поведение встроенных объектов.** `new Proxy(Array, ...)` может вести себя не так, как ожидается, потому что движок оптимизирует встроенные типы.

6. **Отзываемый Proxy.** `Proxy.revocable` создаёт прокси, который можно отключить вызовом `revoke()`.

   ```js
   const { proxy, revoke } = Proxy.revocable({ value: 42 }, {});
   console.log(proxy.value); // 42
   revoke();
   console.log(proxy.value); // TypeError
   ```

---

## Чеклист

- [ ] Что такое `Proxy` и для чего применяется.
- [ ] Какие ловушки есть у `Proxy` и что они перехватывают.
- [ ] Как работают ловушки `get`, `set`, `has`, `deleteProperty`, `ownKeys`, `apply`, `construct`.
- [ ] Зачем нужен `Reflect` и чем он отличается от прямого доступа к объекту.
- [ ] Как использовать `Reflect` для корректной передачи `receiver`.
- [ ] Как Vue 3 использует `Proxy` для реактивности.
- [ ] Чем реактивность Vue 3 на Proxy отличается от Vue 2 на `Object.defineProperty`.
- [ ] Какие ограничения у `Proxy`.
- [ ] Как создать отзываемый прокси через `Proxy.revocable`.
