# Порядок рендеринга компонентов и вызова хуков в React

Понимание того, в каком порядке React рендерит компоненты и вызывает хуки, — ключ к написанию предсказуемого кода. В этой статье разберём жизненный цикл рендера: от вызова `setState` до обновления DOM, а также строгие правила, которые React накладывает на порядок вызова хуков.

---

## Две фазы рендеринга

Процесс обновления UI в React делится на две фазы:

| Фаза | Что происходит | Можно прервать? |
|------|---------------|-----------------|
| **Render phase** (фаза рендеринга) | React вызывает компоненты, вычисляет JSX, сравнивает с предыдущим деревом (reconciliation) | Да, может быть отложена или прервана |
| **Commit phase** (фаза фиксации) | React применяет изменения к DOM, вызывает эффекты | Нет, выполняется синхронно |

### Render phase

В этой фазе React:

1. Вызывает функцию компонента (или `render()` для классов).
2. Выполняет все хуки (`useState`, `useEffect` и т.д.) **в порядке их написания**.
3. Строит виртуальное дерево (React elements).
4. Сравнивает новое дерево с предыдущим (алгоритм reconciliation).
5. Вычисляет минимальный набор изменений (diff).

> **Важно:** в render phase **нельзя** выполнять побочные эффекты — мутации DOM, сетевые запросы, `console.log` с побочным результатом. React может вызвать компонент несколько раз (Strict Mode вызывает дважды в dev-режиме), и побочные эффекты приведут к багам.

### Commit phase

После того как diff вычислен, React синхронно применяет изменения:

1. **DOM-мутации** — вставка, удаление, обновление узлов.
2. **`useLayoutEffect`** — вызывается синхронно после мутаций DOM, но **до** отрисовки в браузере.
3. **Браузерная отрисовка** (paint) — браузер перерисовывает страницу.
4. **`useEffect`** — вызывается асинхронно **после** отрисовки.

---

## Порядок рендеринга компонентов

React рендерит компоненты **сверху вниз** — от родителя к детям. Это называется **render flow**.

```jsx
function App() {
  console.log('1. App render');
  return <Header />;
}

function Header() {
  console.log('2. Header render');
  return <Nav />;
}

function Nav() {
  console.log('3. Nav render');
  return <ul><li>Home</li></ul>;
}
```

В консоли будет: `1 → 2 → 3`. Родитель всегда рендерится первым, потом его дети по порядку появления в JSX.

### Порядок при обновлении

При повторном рендере (например, из-за `setState`) React снова проходит дерево сверху вниз, но **пропускает** поддеревья, которые не изменились (если они обёрнуты в `memo`, `PureComponent` или если пропсы не изменились для классовых компонентов).

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  console.log('Parent render');
  return (
    <>
      <ChildA />         {/* перерендерится всегда */}
      <ChildB value={count} /> {/* перерендерится при изменении count */}
    </>
  );
}
```

### Порядок рендеринга детей

Дети рендерятся **в порядке их объявления** в JSX, слева направо:

```jsx
<div>
  <First />   {/* рендерится первым */}
  <Second />  {/* рендерится вторым */}
  <Third />   {/* рендерится третьим */}
</div>
```

---

## Порядок вызова хуков

Хуки вызываются **строго в порядке их написания** сверху вниз внутри компонента. React запоминает этот порядок при первом рендере и при каждом следующем рендере вызывает хуки в том же порядке.

```jsx
function Counter() {
  const [count, setCount] = useState(0);       // хук #1
  const ref = useRef(null);                     // хук #2
  const value = useContext(MyContext);          // хук #3
  const memo = useMemo(() => count * 2, [count]); // хук #4

  useEffect(() => {                             // хук #5
    console.log('effect');
  }, [count]);

  useLayoutEffect(() => {                       // хук #6
    console.log('layout effect');
  }, []);

  return <div ref={ref}>{count}</div>;
}
```

При каждом рендере React вызовет хуки в порядке: 1 → 2 → 3 → 4 → 5 → 6.

### Почему порядок важен

React связывает каждый вызов хука с его позицией (индексом) в списке вызовов. Если порядок изменится между рендерами, React сопоставит состояния неправильно:

```jsx
// ПЛОХО — порядок хуков зависит от условия
function Bad({ flag }) {
  if (flag) {
    const [a, setA] = useState(0);  // хук #1 (только когда flag === true)
  }
  const [b, setB] = useState(0);    // хук #1 при flag === false, хук #2 при flag === true

  return <div>{b}</div>;
}
```

Это нарушает **Правила хуков** и приведёт к ошибке или непредсказуемому поведению.

---

## Правила хуков (Rules of Hooks)

1. **Вызывайте хуки только на верхнем уровне** — не внутри условий, циклов, вложенных функций.
2. **Вызывайте хуки только из функциональных компонентов** (или кастомных хуков).
3. **Порядок вызова должен быть одинаковым** при каждом рендере.

```jsx
// ПРАВИЛЬНО
function Good() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');
  useEffect(() => { /* ... */ }, [count]);
  return <div>{count} {name}</div>;
}

// ПЛОХО — хук внутри условия
function Bad({ show }) {
  const [count, setCount] = useState(0);
  if (show) {
    const [name, setName] = useState(''); // нарушение правил!
  }
  return <div>{count}</div>;
}
```

Для автоматической проверки этих правил существует ESLint-плагин [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks).

---

## Порядок вызова эффектов (useEffect / useLayoutEffect)

Эффекты вызываются **после** commit phase, в определённом порядке:

### useLayoutEffect

1. Вызывается **синхронно** сразу после мутаций DOM.
2. Выполняется **до** отрисовки браузера.
3. Блокирует визуальное обновление — используйте только для синхронных измерений DOM.

### useEffect

1. Вызывается **асинхронно** после отрисовки браузера.
2. Не блокирует paint.
3. Подходит для сетевых запросов, подписок, логирования.

### Порядок вызова эффектов в дереве компонентов

Эффекты вызываются **снизу вверх** — от детей к родителям. Это позволяет дочерним компонентам установить свои эффекты до того, как родительские эффекты попытаются взаимодействовать с детьми.

```jsx
function Parent() {
  useLayoutEffect(() => console.log('Parent layout effect'));
  useEffect(() => console.log('Parent effect'));
  return <Child />;
}

function Child() {
  useLayoutEffect(() => console.log('Child layout effect'));
  useEffect(() => console.log('Child effect'));
  return <div>Child</div>;
}
```

Порядок в консоли:

```
Child layout effect
Parent layout effect
(браузер рисует)
Child effect
Parent effect
```

### Почему эффекты вызываются снизу вверх?

Это кажется парадоксальным: рендер идёт **сверху вниз** (родитель → дети), а эффекты — **снизу вверх** (дети → родитель). Но в этом есть логика.

**Разберём по шагам:**

1. **Render phase (сверху вниз):**
   - React вызывает `Parent()` → получает JSX с `<Child />`
   - React вызывает `Child()` → получает JSX с `<div>`
   - Все компоненты вызваны, React знает всё дерево

2. **Commit phase:**
   - React обновляет DOM (вставляет все узлы)
   - Теперь React запускает эффекты

**Почему снизу вверх?** Представьте стек:

```
Во время рендера React "складывает" эффекты в стек:

Рендер Parent → в стек: [Parent effect]
Рендер Child  → в стек: [Parent effect, Child effect]

Теперь стек выглядит так (вершина справа):
[Parent effect, Child effect]
                  ↑
              вершина стека
```

Когда React запускает эффекты, он берёт из стека **сверху** (LIFO — Last In, First Out):

```
1. Забираем Child effect (последний добавлен)
2. Забираем Parent effect
```

**Практическая причина:** родительские эффекты могут захотеть взаимодействовать с DOM дочерних компонентов. Если бы родительский эффект запустился первым, он мог бы обратиться к DOM, который ещё не инициализирован дочерними эффектами.

**Пример:**

```jsx
function Parent() {
  const parentRef = useRef(null);
  
  useEffect(() => {
    // Родитель хочет измерить размер контейнера, включая детей
    const height = parentRef.current.offsetHeight;
    console.log('Parent height:', height);
  }, []);
  
  return (
    <div ref={parentRef}>
      <Child />
    </div>
  );
}

function Child() {
  useEffect(() => {
    // Дочерний эффект настраивает что-то в DOM
    console.log('Child effect: DOM настроен');
  }, []);
  
  return <div style={{ height: '100px' }}>Child content</div>;
}
```

Если бы эффекты запускались сверху вниз:
```
Parent effect → пытается измерить высоту, но Child effect ещё не настроил DOM
Child effect  → настраивает DOM
```

Но React запускает снизу вверх:
```
Child effect  → настраивает DOM
Parent effect → измеряет высоту (DOM уже готов)
```

**Аналогия:** представьте, что вы строите дом. Сначала рабочие (дети) кладут кирпичи, а потом прораб (родитель) проверяет результат. Было бы странно, если бы прораб проверял стену до того, как рабочие её достроили.

**Важно:** это касается только **запуска** эффектов. **Cleanup-функции** (то, что возвращает `useEffect`) тоже вызываются снизу вверх — в обратном порядке относительно создания.

### Cleanup-функции

При размонтировании или перед повторным запуском эффекта React вызывает **cleanup-функцию** (то, что вернул эффект). Порядок cleanup — тоже **снизу вверх**:

```jsx
useEffect(() => {
  const subscription = subscribe();
  return () => subscription.unsubscribe(); // cleanup
}, [dependency]);
```

---

## Полный порядок при монтировании

```
1. Render phase (сверху вниз):
   ├── App() — вызов функции, вызов хуков (useState, useMemo, ...)
   ├── Header() — вызов функции, вызов хуков
   └── Content() — вызов функции, вызов хуков

2. Commit phase:
   ├── DOM-мутации (вставка узлов)
   ├── useLayoutEffect (снизу вверх):
   │   ├── Content layout effect
   │   ├── Header layout effect
   │   └── App layout effect
   ├── Браузерная отрисовка (paint)
   └── useEffect (снизу вверх):
       ├── Content effect
       ├── Header effect
       └── App effect
```

## Полный порядок при обновлении (re-render)

```
1. Render phase (сверху вниз):
   ├── App() — хуки вызываются в том же порядке, сравнение с предыдущим состоянием
   ├── Header() — пропсы не изменились → пропущен (если memo)
   └── Content() — хуки вызываются, новый diff

2. Commit phase:
   ├── DOM-мутации (только изменённые узлы)
   ├── useLayoutEffect cleanup (снизу вверх, только для изменившихся зависимостей)
   ├── useLayoutEffect (снизу вверх, только для изменившихся зависимостей)
   ├── Браузерная отрисовка
   ├── useEffect cleanup (снизу вверх, только для изменившихся зависимостей)
   └── useEffect (снизу вверх, только для изменившихся зависимостей)
```

## Полный порядок при размонтировании

```
1. useEffect cleanup (снизу вверх)
2. useLayoutEffect cleanup (снизу вверх)
3. DOM-удаление узлов
```

---

## Strict Mode и двойной вызов

В режиме разработки (`<React.StrictMode>`) React **намеренно** вызывает компоненты и эффекты дважды, чтобы помочь найти побочные эффекты:

```jsx
<StrictMode>
  <App />
</StrictMode>
```

При монтировании:

```
Render: App()
Render: App()          ← второй вызов
Layout effects: App()
Layout effects cleanup ← cleanup
Layout effects: App()  ← повторный вызов
(Paint)
Effects: App()
Effects cleanup        ← cleanup
Effects: App()         ← повторный вызов
```

Это помогает убедиться, что cleanup-функции корректно «отменяют» эффекты. В production-сборке двойной вызов **не происходит**.

---

## Практические следствия

### 1. Не вызывайте хуки условно

```jsx
// ПЛОХО
if (condition) {
  useEffect(() => { ... });
}

// ХОРОШО
useEffect(() => {
  if (condition) {
    // логика внутри эффекта
  }
}, [condition]);
```

### 2. Учитывайте порядок эффектов в дереве

Если родительский `useEffect` зависит от ref, который устанавливается в дочернем `useEffect`, будьте осторожны — дочерний эффект вызовется первым, но если он асинхронный, родительский может сработать до его завершения.

### 3. Размещайте хуки до ранних return

```jsx
// ПЛОХО — хук после return
function Component({ data }) {
  if (!data) return null;
  const [value, setValue] = useState(0); // нарушение правил!
  return <div>{value}</div>;
}

// ХОРОШО — все хуки до return
function Component({ data }) {
  const [value, setValue] = useState(0);
  if (!data) return null;
  return <div>{value}</div>;
}
```

### 4. useLayoutEffect vs useEffect

- `useLayoutEffect` — для синхронных измерений DOM (размеры, позиции), чтения/записи DOM до paint.
- `useEffect` — для всего остального: запросы, подписки, логирование.

### 5. Порядок setState при нескольких вызовах

React **батчит** (объединяет) несколько вызовов `setState` в один рендер:

```jsx
function Counter() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  const handleClick = () => {
    setA(1); // не вызывает рендер сразу
    setB(2); // не вызывает рендер сразу
    // React выполнит один ре-рендер с a=1, b=2
  };
}
```

---

## Краткая шпаргалка

| Что | Порядок | Когда |
|-----|---------|-------|
| Рендер компонентов | Сверху вниз (родитель → дети) | Render phase |
| Вызов хуков | По порядку написания | Render phase |
| DOM-мутации | По порядку дерева | Commit phase |
| `useLayoutEffect` | Снизу вверх (дети → родители) | После мутаций, до paint |
| `useLayoutEffect` cleanup | Снизу вверх | Перед повторным вызовом / размонтированием |
| `useEffect` | Снизу вверх (дети → родители) | После paint |
| `useEffect` cleanup | Снизу вверх | Перед повторным вызовом / размонтированием |

---

## Полезные ссылки

- [React Rendering Order](https://react.dev/learn/synchronizing-with-effects#how-does-the-effect-lifecycle-differ-from-components)
- [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)
- [useEffect lifecycle](https://react.dev/reference/react/useEffect#storing-information-from-previous-renders)
- [React Reconciler](https://github.com/facebook/react/tree/main/packages/react-reconciler)
