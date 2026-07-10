# Хуки React — глубокое погружение

Хуки (Hooks) — фундаментальная концепция React, позволяющая функциональным компонентам работать с состоянием, побочными эффектами и контекстом без использования классов. Появившись в React 16.8 (февраль 2019), хуки стали стандартом разработки и практически полностью вытеснили классовые компоненты. В этой статье мы подробно разберём ключевые встроенные хуки, их назначение, правильное использование и типичные ошибки.

---

## useRef — храним значение между рендерами без перерисовки

`useRef` возвращает изменяемый ref-объект, свойство `.current` которого сохраняется между рендерами, но в отличие от `useState`, его изменение **не вызывает повторный рендер**. Это делает `useRef` идеальным инструментом в двух ключевых сценариях.

> **Vue-аналог:** `ref()` для хранения значения и доступ через `.value` (реактивный, но один ref-объект), плюс **template ref** — атрибут `ref` на HTML-элементе + `const el = ref(null)` в `<script setup>` — для доступа к DOM. Ключевое отличие: Vue-`ref()` реактивен и вызывает перерисовку, а React-`useRef` — нет. Для нереактивного хранения в Vue используйте обычную переменную вне `setup` или `shallowRef`.

### Доступ к DOM-элементам

Самое частое применение — получение прямой ссылки на DOM-узел. Через ref можно управлять фокусом, прокруткой, замерами размеров или интегрироваться со сторонними библиотеками:

```jsx
function SearchInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus();
  }, []);

  const selectText = () => {
    inputRef.current.select();
  };

  return (
    <>
      <input ref={inputRef} placeholder="Поиск..." />
      <button onClick={selectText}>Выделить текст</button>
    </>
  );
}
```

### Хранение изменяемых значений без ререндера

`useRef` подходит для хранения ID таймеров, предыдущих значений пропсов/стейта, флагов монтирования — любых данных, чьё изменение не должно вызывать перерисовку:

```jsx
function Chat({ roomId }) {
  const prevRoomId = useRef(roomId);

  useEffect(() => {
    if (prevRoomId.current !== roomId) {
      console.log(`Переключились с ${prevRoomId.current} на ${roomId}`);
      prevRoomId.current = roomId;
    }
  });
}
```

С React 19 ref можно передавать как обычный пропс (`ref={myRef}`), без необходимости оборачивать компонент в `forwardRef`.

---

## useState — фундамент локального состояния

`useState` — базовый хук для добавления состояния в функциональный компонент. Он возвращает кортеж из текущего значения и функции-сеттера.

> **Vue-аналог:** `ref()` — прямой и ближайший эквивалент. Оба возвращают реактивное значение и сеттер: React `const [val, setVal] = useState(0)` ↔ Vue `const val = ref(0)`. Разница: в React сеттер — отдельная функция с заменой значения, в Vue мутация через `val.value = ...` отслеживается Proxy-системой автоматически. Vue `reactive()` — аналог для объектов, но в отличие от React, мутирует их напрямую без необходимости spread-копирования.

### Корректное обновление объектов и массивов

React использует сравнение по ссылке (`Object.is`) для определения необходимости ререндера, поэтому состояние всегда должно обновляться **иммутабельно**:

```jsx
const [user, setUser] = useState({ name: "", email: "" });

// ✅ Правильно: создаём новый объект через spread
setUser(prev => ({ ...prev, name: "Alice" }));

// ❌ Неправильно: мутация существующего объекта
user.name = "Alice";
setUser(user); // React не увидит изменений
```

Тот же принцип работает для массивов — всегда создавайте новый массив через `filter`, `map`, `concat` или spread вместо мутирующих методов вроде `push`/`splice`:

```jsx
// ✅ Правильно
setItems(prev => [...prev, newItem]);
setItems(prev => prev.filter(item => item.id !== deletedId));

// ❌ Неправильно
items.push(newItem);
setItems(items);
```

### Функциональное обновление

Если новое состояние зависит от предыдущего, всегда используйте форму с колбэком: `setState(prev => ...)`. Это предотвращает баги при батчинге обновлений:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  // ✅ Даже три вызова подряд корректно инкрементируют на 3
  const incrementThree = () => {
    setCount(c => c + 1);
    setCount(c => c + 1);
    setCount(c => c + 1);
  };

  return <button onClick={incrementThree}>+3 ({count})</button>;
}
```

### Ленивая инициализация

Если начальное значение требует дорогостоящих вычислений, передайте функцию — она вызовется ровно один раз при монтировании:

```jsx
const [data, setData] = useState(() => {
  const saved = localStorage.getItem("app-data");
  return saved ? JSON.parse(saved) : expensiveInitialComputation();
});
```

---

## useReducer — управление сложным состоянием

Когда состояние состоит из нескольких полей, обновляемых совместно, или логика перехода нетривиальна, `useReducer` предоставляет более структурированный подход, чем разрозненные `useState`. Редуктор — чистая функция `(state, action) => nextState`, принимающая текущее состояние и объект действия.

### Сигнатура и базовый пример

```js
const [state, dispatch] = useReducer(reducer, initialState);
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

**Аргументы:**
- `reducer` — `(state, action) => nextState`, чистая функция без побочных эффектов.
- `initialState` / `initialArg` — начальное состояние (или аргумент для ленивой инициализации).
- `init` **(опционально)** — функция `init(initialArg) => initialState` для ленивого вычисления стартового значения.

**Возвращает:** кортеж `[state, dispatch]`, где `dispatch` — функция, принимающая экшон и стабильная между рендерами (можно смело передавать как пропс без `useCallback`).

### Почему не просто useState

Возьмём типичный сценарий: форма с пятью полями, валидацией и состоянием отправки. На `useState` вы получите 6–7 вызовов и растянутую по компоненту логику обновления:

```jsx
// ❌ Фрагментированное состояние — сложно отследить согласованность
const [name, setName] = useState("");
const [email, setEmail] = useState("");
const [age, setAge] = useState("");
const [errors, setErrors] = useState({});
const [isSubmitting, setIsSubmitting] = useState(false);

// Где-то в обработчике сабмита:
setIsSubmitting(true);
setErrors({ ...errors, name: validateName(name) }); // уже устаревшее errors!
```

С `useReducer` те же обновления — один атомарный переход:

```jsx
function formReducer(state, action) {
  switch (action.type) {
    case "set_field":
      return { ...state, [action.field]: action.value, errors: { ...state.errors, [action.field]: "" } };
    case "validate":
      return { ...state, errors: validate(state) };
    case "submit_start":
      return { ...state, isSubmitting: true };
    case "submit_end":
      return { ...state, isSubmitting: false };
    default:
      return state;
  }
}

const [form, dispatch] = useReducer(formReducer, {
  name: "", email: "", age: "", errors: {}, isSubmitting: false,
});
```

Главное преимущество: каждое действие — именованный семантический переход, а не разрозненные `setX`. Читая код, вы видите **намерение**: `dispatch({ type: "submit_start" })` понятнее, чем три `setIsSubmitting` + `setErrors` + `setName`.

### Классический паттерн: fetch-машина

Самый частый use-case — три состояния асинхронной загрузки. `useReducer` гарантирует, что вы никогда не окажетесь в бессмысленном состоянии вроде `{ loading: true, error: "..." }` одновременно:

```ts
type State<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };
```

Эту же идею развивают библиотеки вроде **XState** и **TanStack Query**, но для 90% случаев встроенного `useReducer` достаточно.

### Ленивая инициализация

Третий аргумент `init` позволяет вычислить начальное состояние один раз, а не при каждом рендере. Особенно полезно, когда начальное состояние восстанавливается из localStorage или требует вычислений:

```jsx
function createInitialState(username) {
  const saved = localStorage.getItem(`draft-${username}`);
  return saved ? JSON.parse(saved) : defaultDraft;
}

function PostEditor({ username }) {
  const [draft, dispatch] = useReducer(
    draftReducer,
    username,          // initialArg — передаётся в init
    createInitialState // init — вызывается один раз
  );
  // ...
}
```

Без `init` пришлось бы вставлять ленивую логику внутрь редуктора (что загрязняет чистую функцию) или использовать `useState(() => ...)` + ручную логику перехода.

### Композиция: useReducer + useContext

Связка `useReducer` + `Context` — минималистичная альтернатива Redux, достаточная для приложений малого и среднего размера. Без дополнительных библиотек, без бойлерплейта:

```jsx
const CounterContext = createContext(null);
const CounterDispatchContext = createContext(null);

function CounterProvider({ children }) {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });
  return (
    <CounterContext value={state}>
      <CounterDispatchContext value={dispatch}>
        {children}
      </CounterDispatchContext>
    </CounterContext>
  );
}

// В любом дочернем компоненте:
const state = use(CounterContext);           // React 19: use() вместо useContext
const dispatch = use(CounterDispatchContext);
```

Разделение `state` и `dispatch` на два контекста — осознанный выбор: компоненты, которым нужен только `dispatch`, не перерендерятся при изменении состояния. Альтернативно, передавайте `dispatch` через единый контекст — ссылка `dispatch` стабильна, поэтому компоненты без чтения `state` всё равно не перерендерятся.

### Сравнение useReducer и Redux

`useReducer` — это **локальный** Redux одного компонента. Сравним:

| | useReducer | Redux Toolkit |
|---|---|---|
| Область видимости | Один компонент (или дерево через Context) | Всё приложение |
| Middleware | Нет; оборачивайте редуктор вручную | Встроенные: thunk, saga, listener |
| DevTools | Нет из коробки | Redux DevTools с time-travel |
| Селекторы | Ручные | `createSelector` с мемоизацией |
| Асинхронные экшоны | Не поддерживаются — диспатчите из `useEffect` или обработчиков | `createAsyncThunk`, RTK Query |
| Бойлерплейт | Минимальный | Существенный, но RTK сокращает |
| Когда использовать | Локально-сложное состояние формы/мастера/игры | Глобальное состояние: юзер, корзина, флаги |

**Правило:** если состояние нужно одному компоненту — `useReducer`. Если двум-трём соседним — поднимите reducer через Context. Если десяткам разбросанных по приложению — Redux/Zustand.

### Сравнение useReducer и Zustand

Zustand ещё проще Redux: один вызов `create`, никаких провайдеров, экшоны — просто функции внутри стора:

```ts
const useCounter = create((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}));
```

Когда `useReducer` выигрывает у Zustand:
- Состояние не должно жить дольше компонента (форма размонтируется — состояние умирает).
- Переходы явно моделируются — тестировать редьюсер проще, чем тестировать Zustand-стор.
- Хотите zero-dependency и не тащить пакет ради трёх полей формы.

Когда Zustand выигрывает:
- Состояние нужно за пределами React (в `fetch`-интерсепторе, WebSocket-обработчике).
- Нужна подписка на конкретный кусок состояния без ререндера всего дерева.
- Нужны middleware вроде `persist`, `devtools`, `immer`.

### Иммутабельность в редукторе

Редуктор обязан возвращать новый объект. Мутация исходного `state` не вызовет ререндер — React сравнивает ссылки через `Object.is`:

```js
// ❌ Не вызовет ререндер
function reducer(state, action) {
  state.count += 1;
  return state;
}

// ✅ Новый объект — ререндер гарантирован
function reducer(state, action) {
  return { ...state, count: state.count + 1 };
}
```

Для глубоко вложенных состояний используйте `structuredClone`, библиотеку `immer` (особенно популярна в связке с Redux, но работает и с `useReducer`), или просто дробите состояние на несколько `useReducer`.

### Тестируемость

Редуктор тестируется как обычная функция — ни React, ни DOM, ни моков рендера:

```ts
test("increment adds 1 to count", () => {
  const state = { count: 5 };
  const next = counterReducer(state, { type: "increment" });
  expect(next.count).toBe(6);
  expect(next).not.toBe(state); // иммутабельность
});

test("reset returns to initial state", () => {
  const next = counterReducer({ count: 99 }, { type: "reset" });
  expect(next.count).toBe(0);
});
```

Это главное преимущество перед `useState`: логику перехода состояний можно покрыть тестами изолированно, без рендера компонента и без симуляции пользовательских действий.

### Когда НЕ использовать useReducer

- **Одно-два примитивных поля:** `useState` проще и читаемее.
- **Состояние нужно за пределами React:** выбирайте Zustand.
- **Нужен time-travel debugging:** нужен Redux DevTools.
- **Много асинхронной логики:** `createAsyncThunk` из Redux или TanStack Query удобнее, чем `useEffect` + `dispatch`.
- **Данные приходят с сервера и не мутируют клиентом:** TanStack Query покрывает этот случай из коробки (кэш, перезапрос, optimistic update).

### Кратко: плюсы и минусы

**Плюсы:**
- Предсказуемость: `(state, action) => nextState` — чистая функция, никаких сайд-эффектов.
- Атомарность: связанные поля обновляются одной операцией.
- Тестируемость: редуктор тестируется без React.
- `dispatch` стабилен (не меняет ссылку), не нужно оборачивать в `useCallback`.
- Хорошо сочетается с TypeScript — discriminated union по полю `type`.
- Ленивая инициализация через третий аргумент.

**Минусы:**
- Бойлерплейт: `switch/case`, константы экшонов, типы — по сравнению с `useState` больше кода ради кода.
- Нет middleware из коробки (логирование, асинхронность, persistance).
- Без `immer` глубокие обновления утомительны.
- При частых обновлениях каждый диспатч вызывает ререндер компонента (в отличие от Zustand с селекторами).

---

## useEffect — побочные эффекты и их очистка

`useEffect` выполняет императивные побочные эффекты после коммита React в DOM: запросы к API, подписки на события, управление таймерами, работу с внешними библиотеками.

> **Vue-аналог:** комбинация `watch` / `watchEffect` (реакция на изменения) и `onMounted` / `onUnmounted` (жизненный цикл). `watchEffect` наиболее близок к `useEffect` без зависимостей — автоматически отслеживает использованные реактивные значения и перезапускается при их изменении. `watch` с массивом источников — аналог `useEffect` с явным массивом зависимостей. Функция очистки в React (`return () => { ... }`) соответствует возврату cleanup-функции из `watch`/`watchEffect` или использованию `onUnmounted`.

### Базовый синтаксис и массив зависимостей

```jsx
useEffect(() => {
  // Тело эффекта: выполняется после рендера
  const subscription = api.subscribe(channelId, onUpdate);

  // Функция очистки: выполняется перед размонтированием
  // и перед следующим запуском эффекта
  return () => subscription.unsubscribe();
}, [channelId]); // Эффект перезапускается при каждом изменении channelId
```

Варианты массива зависимостей:
- `[]` — эффект срабатывает **один раз** при монтировании (аналог `componentDidMount`).
- `[dep1, dep2]` — перезапускается при изменении любой из зависимостей.
- Без массива — выполняется **после каждого рендера** (редко нужно на практике).

### Ключевое правило: честные зависимости

Никогда не лгите о зависимостях — это ведёт к чтению замыканием устаревших значений (stale closure). Если значение используется внутри эффекта, оно должно быть в массиве зависимостей. Исключение — `useEffectEvent` из React 19.2, который намеренно отделяет событийную логику от реактивной.

### Распространённые ошибки

**Бесконечный цикл ререндеров** возникает, когда эффект обновляет состояние без условий, которое само является его зависимостью:

```jsx
// ❌ Бесконечный цикл: эффект → setData → ререндер → эффект → ...
useEffect(() => {
  setData(transform(data));
}, [data]);
```

Решения: либо убрать `setData` из эффекта (вычислять `data` прямо в рендере), либо добавить условие досрочного выхода.

**Загрузка данных в эффекте** — допустимый, но не всегда оптимальный паттерн. В 2026 году предпочтительнее использовать TanStack Query для асинхронных данных или серверные компоненты React с прямым доступом к БД на сервере.

---

## useMemo и useCallback — мемоизация значений и функций

> **Vue-аналог:** `useMemo` ↔ `computed()` — классическое соответствие. Оба кэшируют результат вычисления и пересчитывают только при изменении отслеживаемых зависимостей: React `const v = useMemo(() => heavy(a), [a])` ↔ Vue `const v = computed(() => heavy(a))`. Ключевая разница: Vue `computed` автоматически собирает зависимости из реактивных переменных, не требуя явного массива. `useCallback` **не имеет прямого аналога** в Vue — Vue не страдает от проблемы смены идентичности функций между рендерами, так как шаблон компилируется в эффективный рендер-код, а не перевызывается целиком как React-компонент.

### useMemo — кэширование результатов вычислений

`useMemo` мемоизирует результат дорогостоящей функции между рендерами. Пересчёт происходит только при изменении переданных зависимостей:

```jsx
function ProductList({ items, filterQuery }) {
  const filteredItems = useMemo(
    () => items.filter(item =>
      item.name.toLowerCase().includes(filterQuery.toLowerCase())
    ),
    [items, filterQuery]
  );

  return filteredItems.map(item => <ProductCard key={item.id} item={item} />);
}
```

### useCallback — стабильная ссылка на функцию

`useCallback` мемоизирует саму функцию, предотвращая её пересоздание при каждом рендере. Актуально, когда функция передаётся в пропсах дочерним компонентам, обёрнутым в `React.memo`:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const increment = useCallback(
    () => setCount(c => c + 1),
    [] // Функция не зависит от count, используем prev => prev
  );

  return <ExpensiveChild onIncrement={increment} />;
}

const ExpensiveChild = React.memo(function ({ onIncrement }) {
  // Перерендер только при изменении onIncrement,
  // что не случится, пока не изменятся зависимости useCallback
  return <button onClick={onIncrement}>+1</button>;
});
```

### React Compiler и будущее мемоизации

С релизом **React Compiler** (стабильный в 2025) ручное использование `useMemo`/`useCallback` в значительной степени устарело. Компилятор на этапе сборки анализирует код и автоматически мемоизирует компоненты и выражения — точно так же, как это сделал бы опытный разработчик вручную, но без риска человеческой ошибки. В новых проектах рекомендуется включать компилятор и удалять ручные `useMemo`/`useCallback`, где это возможно.

---

## Пользовательские хуки — переиспользуемая логика

Пользовательский хук — это JavaScript-функция, имя которой начинается с `use`, вызывающая другие хуки. Она инкапсулирует логику с состоянием, которую можно переиспользовать между компонентами.

> **Vue-аналог:** **Vue Composables** — практически идентичный паттерн. Функции с префиксом `use*` (например, `useWindowSize`, `useFetch`), использующие Vue Composition API (`ref`, `watch`, `onMounted`) внутри `<script setup>` или других composables. Разница в деталях: React-хуки требуют соблюдения очерёдности вызовов между рендерами, Vue-композаблы — нет, так как Vue использует граф зависимостей, а не ордер-базированное связывание. Огромная экосистема **VueUse** (аналог react-use) предоставляет сотни готовых composables.

### Паттерн композиции хуков

```jsx
function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    let frameId;
    const handleResize = () => {
      cancelAnimationFrame(frameId);
      frameId = requestAnimationFrame(() => {
        setSize({ width: window.innerWidth, height: window.innerHeight });
      });
    };

    window.addEventListener("resize", handleResize);
    return () => {
      window.removeEventListener("resize", handleResize);
      cancelAnimationFrame(frameId);
    };
  }, []);

  return size;
}
```

Ключевые правила разработки пользовательских хуков:
- **Начинайте имя с `use`** — это зарезервированный префикс, позволяющий линтеру проверять правила хуков.
- **Храните один хук — одну ответственность.** Если хук делает и размер, и скролл, и инпут — разбейте на несколько.
- **Возвращайте консистентный API.** Либо кортеж `[value, setter]` (как `useState`), либо объект с именованными полями.
- **Не вызывайте хуки условно** — правило распространяется и на пользовательские хуки.

---

## useTransition — неблокирующие обновления

Конкурентный рендеринг React 18+ ввёл понятие приоритетов обновлений. `useTransition` помечает обновление состояния как **не срочное** (transition), позволяя React обслуживать более приоритетные события (клики, ввод) без задержек, пока фоновое обновление всё ещё вычисляется.

> **Vue-аналог:** прямого эквивалента в ядре Vue нет — Vue не имеет конкурентного планировщика и асинхронной прерываемой модели рендеринга. Ближайший функциональный аналог — `useDebouncedRef` из **VueUse** или ручной `ref` + `watch` с `setTimeout`/`requestAnimationFrame`, чтобы отложить обновление «тяжёлого» состояния и не блокировать инпут. Однако это лишь имитация на уровне пользовательского кода, без интеграции в сам рендерер как в React.

```jsx
function SearchResults({ data }) {
  const [query, setQuery] = useState("");
  const [filteredData, setFilteredData] = useState(data);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value); // Срочное обновление — инпут остаётся отзывчивым

    startTransition(() => {
      // Несрочное обновление — тяжёлая фильтрация
      setFilteredData(
        data.filter(item =>
          item.title.toLowerCase().includes(value.toLowerCase())
        )
      );
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? (
        <Spinner />
      ) : (
        <List items={filteredData} />
      )}
    </>
  );
}
```

`isPending` равен `true` в течение всего периода обработки перехода, позволяя показать индикатор загрузки. Важно: обновления внутри `startTransition` всегда являются несрочными и могут быть прерваны.

---

## useDeferredValue — отложенное значение

`useDeferredValue` решает ту же проблему, что и `useTransition`, но подходит для ситуаций, когда вы **не контролируете сеттер** состояния — например, значение приходит через пропсы:

> **Vue-аналог:** также нет прямого эквивалента в ядре. Ближайшее — паттерн с двумя `ref`: один мгновенно реагирует на изменение пропса, второй — «ленивый», обновляется через `watch` с `setTimeout(0)` или `nextTick`. Пакет **VueUse** предоставляет `useDeferredValue` с похожей семантикой. Низкоуровневый аналог — `shallowRef`, который не отслеживает глубокие изменения, но это решает другую задачу (производительность больших объектов), нежели приоритизация обновлений.

```jsx
function SearchPage({ query }) {
  // query приходит от родительского компонента,
  // мы не можем обернуть setQuery в startTransition
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <SlowSearchResults query={deferredQuery} />
    </div>
  );
}
```

`deferredQuery` «отстаёт» от `query` — React обновляет его только когда завершены все срочные обновления. В приведённом примере инпут родителя мгновенно реагирует на ввод, а тяжёлый список результатов обновляется с задержкой и полупрозрачностью.

**Когда использовать `useTransition` vs `useDeferredValue`:**
- `useTransition` — вы контролируете сеттер и хотите явно отметить обновление как несрочное.
- `useDeferredValue` — вы получаете значение «сверху» и хотите отложить реагирование на его изменение.

Оба хука опираются на конкурентный рендерер и требуют React 18+.

---

## Общие правила хуков

Хуки подчиняются строгим правилам, которые обеспечивают корректность их работы:

1. **Вызывайте хуки только на верхнем уровне.** Не в циклах, условиях и вложенных функциях. Порядок вызова хуков между рендерами должен быть одинаковым — React идентифицирует состояние хука по порядковому номеру вызова.

2. **Вызывайте хуки только из React-компонентов или пользовательских хуков.** Не из обычных JavaScript-функций.

3. **Правило зависимостей.** Все переменные из замыкания, используемые в эффекте, должны быть перечислены в массиве зависимостей. ESLint-плагин `eslint-plugin-react-hooks` автоматически проверяет это правило.

Современный React (19+) расширяет эту картину хуком `use()`, который **можно** вызывать условно — это исключение, допустимое благодаря специальной семантике разворачивания Promise/Context.

---

## Итог

Хуки — не просто синтаксический сахар, а переосмысление того, как в React моделируется логика с состоянием. Их главная сила в **композиции**: маленькие хуки вроде `useState`, `useRef`, `useEffect` комбинируются в пользовательские хуки, создавая понятный и переиспользуемый код. С появлением конкурентного рендеринга дополнительные хуки вроде `useTransition` и `useDeferredValue` позволяют создавать плавные, отзывчивые интерфейсы даже при тяжёлых вычислениях. Наконец, React Compiler снимает с разработчика бремя ручной мемоизации, делая код чище и производительнее без дополнительных усилий.
