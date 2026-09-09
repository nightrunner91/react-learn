# Асинхронность: Promise, async/await

JavaScript — однопоточный язык, но большая часть реальных задач требует асинхронности: сетевые запросы, таймеры, работа с файлами, события пользователя. Современный способ работать с асинхронностью — **Promise** и синтаксический сахар **async/await**. Понимание их поведения, приоритетов в Event Loop и обработки ошибок — обязательно для собеседований и для написания стабильного кода.

## Содержание

1. [Зачем нужна асинхронность](#зачем-нужна-асинхронность)
2. [Promise: состояния и жизненный цикл](#promise-состояния-и-жизненный-цикл)
3. [Promise и Event Loop](#promise-и-event-loop)
4. [Методы Promise: all, race, allSettled, any](#методы-promise-all-race-allsettled-any)
5. [async/await](#asyncawait)
6. [Последовательное и параллельное выполнение](#последовательное-и-параллельное-выполнение)
7. [Обработка ошибок](#обработка-ошибок)
8. [Отмена асинхронных операций: AbortController](#отмена-асинхронных-операций-abortcontroller)
9. [Чеклист](#чеклист)

---

## Зачем нужна асинхронность

В синхронном коде каждая операция блокирует выполнение следующей. Если запрос к серверу занимает секунду, весь остальной код ждёт:

```js
const data = fetchDataSync(); // блокирует на 1 секунду
console.log('Done');
```

Асинхронность позволяет начать операцию, передать callback или Promise, и продолжить выполнять другой код. Когда операция завершится, движок вызовет обработчик.

До Promise асинхронность реализовывалась через вложенные callback'и:

```js
getData((err, data) => {
  if (err) { /* ... */ }
  processData(data, (err, result) => {
    if (err) { /* ... */ }
    saveResult(result, (err) => {
      // callback hell
    });
  });
});
```

Promise и `async/await` заменили этот подход плоским, читаемым кодом.

---

## Promise: состояния и жизненный цикл

**Promise** — это объект, представляющий результат асинхронной операции. У него три состояния:

- **pending** — операция ещё не завершена.
- **fulfilled** — операция успешно завершена, есть результат.
- **rejected** — операция завершилась с ошибкой.

Состояние может измениться только один раз: из `pending` в `fulfilled` или из `pending` в `rejected`. После этого Promise становится **settled**.

```js
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('Done');
  }, 1000);
});

promise
  .then((value) => console.log(value))   // Done
  .catch((error) => console.error(error))
  .finally(() => console.log('Finished'));
```

### Методы Promise

- `.then(onFulfilled, onRejected)` — обработка успеха или ошибки.
- `.catch(onRejected)` — обработка ошибки (синоним `.then(null, onRejected)`).
- `.finally(onFinally)` — выполняется в любом случае, не получает аргументов.

```js
fetch('/api/user')
  .then((response) => response.json())
  .then((user) => console.log(user))
  .catch((error) => console.error('Request failed:', error))
  .finally(() => console.log('Cleanup'));
```

### Цепочки then

Каждый `.then` возвращает новый Promise. Если callback возвращает значение, следующий `.then` получит его. Если возвращается Promise, цепочка дождётся его выполнения.

```js
Promise.resolve(1)
  .then((x) => x + 1)          // 2
  .then((x) => Promise.resolve(x * 2)) // 4
  .then((x) => console.log(x)); // 4
```

---

## Promise и Event Loop

Promise используют **микрозадачи** (microtasks). Callback из `.then` / `.catch` / `.finally` попадает в микрозадачную очередь, а не в очередь макрозадач.

```js
console.log('Start');

setTimeout(() => console.log('Timeout'), 0);

Promise.resolve().then(() => console.log('Promise'));

console.log('End');
// Start
// End
// Promise
// Timeout
```

Почему `Promise` выполняется раньше `setTimeout`? Потому что после синхронного кода Event Loop опустошает **все** микрозадачи, и только потом берёт следующую макрозадачу.

Это важно для `async/await`: код после `await` тоже становится микрозадачей.

```js
async function example() {
  console.log('Before await');
  await Promise.resolve();
  console.log('After await'); // микрозадача
}

example();
console.log('Sync');
// Before await
// Sync
// After await
```

---

## Методы Promise: all, race, allSettled, any

### Promise.all

Ожидает выполнения **всех** Promise. Возвращает массив результатов в том же порядке. Если хотя бы один Promise отклонён — весь `Promise.all` отклоняется.

```js
const [users, posts] = await Promise.all([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/posts').then(r => r.json()),
]);
```

### Promise.race

Возвращает результат **первого** завершившегося Promise — успешного или отклонённого.

```js
const timeout = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('Timeout')), 5000)
);

const data = fetch('/api/data');

const result = await Promise.race([data, timeout]);
```

### Promise.allSettled

Ожидает **всех**, но никогда не отклоняется. Возвращает массив объектов:

```js
const results = await Promise.allSettled([
  Promise.resolve(1),
  Promise.reject('error'),
]);

// [
//   { status: 'fulfilled', value: 1 },
//   { status: 'rejected', reason: 'error' }
// ]
```

### Promise.any

Возвращает результат **первого успешно** выполнившегося Promise. Если все отклонены — возвращает `AggregateError`.

```js
try {
  const fastest = await Promise.any([
    fetch('/mirror-1'),
    fetch('/mirror-2'),
    fetch('/mirror-3'),
  ]);
} catch (e) {
  console.log('All failed:', e.errors);
}
```

### Сравнение

| Метод | Успех | Ошибка | Когда использовать |
|---|---|---|---|
| `Promise.all` | Все выполнились | Первый отказ | Параллельные зависимые запросы |
| `Promise.race` | Первый результат | Первый отказ | Таймауты |
| `Promise.allSettled` | Всегда | Никогда | Нужно узнать результат каждого |
| `Promise.any` | Первый успех | Все отказали | Зеркала, fallback'ы |

---

## async/await

`async/await` — синтаксический сахар над Promise. Функция, объявленная с `async`, всегда возвращает Promise.

```js
async function getUser() {
  return { id: 1, name: 'Alice' };
}

getUser().then((user) => console.log(user));
```

`await` приостанавливает выполнение функции до завершения Promise. Он может использоваться только внутри `async`-функции (или на верхнем уровне модулей ES).

```js
async function loadUser(id) {
  const response = await fetch(`/api/users/${id}`);
  const user = await response.json();
  return user;
}
```

Важно: `await` **не блокирует** весь Event Loop, только текущую `async`-функцию. Остальной код продолжает выполняться.

---

## Последовательное и параллельное выполнение

### Последовательное

```js
async function sequential() {
  const user = await fetchUser();
  const posts = await fetchPosts(user.id);
  return posts;
}
```

Запросы выполняются один за другим. Общее время — сумма задержек.

### Параллельное

```js
async function parallel() {
  const [user, posts] = await Promise.all([
    fetchUser(),
    fetchPosts(),
  ]);
  return { user, posts };
}
```

Запросы начинаются одновременно. Общее время — максимальная задержка.

### Распространённая ошибка

```js
async function wrong() {
  const user = await fetchUser();
  const posts = await fetchPosts();
  // user и posts не зависят друг от друга, но выполняются последовательно
}
```

Если запросы независимы, используйте `Promise.all`.

---

## Обработка ошибок

### try/catch с await

```js
async function loadData() {
  try {
    const response = await fetch('/api/data');
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    console.error('Failed to load data:', error.message);
    return null;
  }
}
```

### .catch() для Promise

```js
fetch('/api/data')
  .then((response) => response.json())
  .catch((error) => console.error(error));
```

### Unhandled rejection

Если Promise отклонён, но нет обработчика ошибки, возникает **unhandledrejection**:

```js
Promise.reject(new Error('Oops'));
// Uncaught (in promise) Error: Oops
```

Можно отловить глобально:

```js
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled rejection:', event.reason);
});
```

Важно: если ошибка обработана через `.catch` или `try/catch`, событие не возникает.

### Обработка ошибок в Promise.all

```js
try {
  const [users, posts] = await Promise.all([
    fetchUsers(),
    fetchPosts(),
  ]);
} catch (error) {
  // если отказал хотя бы один запрос
  console.error(error);
}
```

Если нужно узнать, какой именно запрос упал, используйте `Promise.allSettled`.

---

## Отмена асинхронных операций: AbortController

`AbortController` позволяет отменять асинхронные операции, чаще всего — `fetch`.

```js
const controller = new AbortController();

fetch('/api/data', { signal: controller.signal })
  .then((response) => response.json())
  .catch((error) => {
    if (error.name === 'AbortError') {
      console.log('Request was aborted');
    } else {
      console.error('Request failed:', error);
    }
  });

// Отмена через 5 секунд
setTimeout(() => controller.abort(), 5000);
```

### С async/await

```js
async function loadUser(id, signal) {
  const response = await fetch(`/api/users/${id}`, { signal });
  return response.json();
}

const controller = new AbortController();
loadUser(1, controller.signal);

// При размонтировании компонента или смене запроса:
controller.abort();
```

### React-пример

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then((response) => response.json())
    .then(setResults)
    .catch((error) => {
      if (error.name !== 'AbortError') {
        setError(error);
      }
    });

  return () => controller.abort();
}, [query]);
```

`AbortController` помогает избежать состояний-гонок (race conditions), когда старый запрос приходит позже нового.

---

## Чеклист

- [ ] В чём разница между `Promise.all` и `Promise.race`.
- [ ] Когда использовать `Promise.allSettled` вместо `Promise.all`.
- [ ] Как `async/await` связан с Promise и микрозадачами.
- [ ] Как обработать ошибку в `async/await`.
- [ ] Что такое unhandled rejection и как его избежать.
- [ ] Как отменить асинхронную операцию с помощью `AbortController`.
- [ ] Когда запросы лучше выполнять параллельно, а когда последовательно.
- [ ] Почему `Promise.then` выполняется раньше `setTimeout(..., 0)`.
