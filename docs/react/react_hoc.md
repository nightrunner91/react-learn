# Higher-Order Components (HOC) — глубокое погружение

Higher-Order Component (HOC, компонент высшего порядка) — один из классических паттернов React для переиспользования логики между компонентами. HOC — это **функция, которая принимает компонент и возвращает новый компонент** с расширенным поведением. До появления хуков HOC был основным механизмом разделения логики (наряду с render props), и до сих пор широко используется в экосистеме — `React.memo`, `React.forwardRef` и многие библиотеки (Redux `connect`, React Router `withRouter`) построены на этом паттерне.

---

## Содержание

1. [Что такое HOC и откуда он взялся](#что-такое-hoc-и-откуда-он-взялся)
2. [Базовая механика](#базовая-механика)
3. [Реальные примеры HOC](#реальные-примеры-hoc)
4. [Композиция HOC](#композиция-hoc)
5. [Проблемы и подводные камни](#проблемы-и-подводные-камни)
6. [HOC vs хуки](#hoc-vs-хуки)
7. [HOC vs Render Props](#hoc-vs-render-props)
8. [Vue-аналоги](#vue-аналоги)
9. [Когда HOC всё ещё актуален](#когда-hoc-всё-ещё-актуален)
10. [Итог](#итог)

---

## Что такое HOC и откуда он взялся

HOC — это паттерн, заимствованный из функционального программирования. Компонент высшего порядка — это функция:

```
hocComponent = (WrappedComponent) => EnhancedComponent
```

В раннем React (до хуков, до 16.8) не было способа вынести логику с состоянием из компонента. Классы предоставляли `componentDidMount`, `setState`, но переиспользовать эту логику между компонентами было сложно. HOC стал ответом — обёртка, которая добавляет поведение любому компоненту, не меняя его кода.

> **Vue-аналог:** во Vue аналогичную роль играют **миксины** (legacy) и **композаблы** (Composition API). Композаблы — более современный и предпочтительный подход, функционально эквивалентный React-хукам. HOC во Vue встречается редко, но технически реализуем через `defineComponent` с обёрткой.

---

## Базовая механика

HOC — обычная JavaScript-функция, возвращающая компонент. Внутри она рендерит обёрнутый компонент, добавляя свои пропсы или поведение:

```jsx
function withLogging(WrappedComponent) {
  return function EnhancedComponent(props) {
    useEffect(() => {
      console.log(`Rendered ${WrappedComponent.name}`, props);
    });

    return <WrappedComponent {...props} />;
  };
}

// Использование
const MyComponentWithLogging = withLogging(MyComponent);
```

### Ключевые принципы

1. **HOC не мутирует исходный компонент.** Он создаёт новый, обёрнутый.
2. **HOC — чистая функция** без побочных эффектов в самом факте создания (логика живёт внутри рендера или эффектов).
3. **Пропсы передаются через** — HOC добавляет свои пропсы, но не удаляет существующие (контракт «передай всё, что получил»).
4. **Имя обёрнутого компонента** полезно для DevTools — установите `displayName`:

```jsx
function withAuth(WrappedComponent) {
  function EnhancedComponent(props) {
    // ...
    return <WrappedComponent {...props} />;
  }
  EnhancedComponent.displayName = `WithAuth(${WrappedComponent.displayName || WrappedComponent.name || "Component"})`;
  return EnhancedComponent;
}
```

---

## Реальные примеры HOC

### 1. Авторизация — защита маршрута

Классический сценарий: компонент рендерится только если пользователь авторизован, иначе — редирект:

```jsx
function withAuth(WrappedComponent) {
  return function ProtectedRoute(props) {
    const user = useAuth();

    if (!user) {
      return <Navigate to="/login" replace />;
    }

    return <WrappedComponent {...props} user={user} />;
  };
}

// Использование
const Dashboard = withAuth(DashboardPage);
const Settings = withAuth(SettingsPage);
```

### 2. Загрузка данных

HOC, который загружает данные и передаёт их в обёрнутый компонент:

```jsx
function withData(WrappedComponent, dataFetcher) {
  return function DataComponent(props) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
      let cancelled = false;
      setLoading(true);

      dataFetcher(props.id)
        .then(result => {
          if (!cancelled) {
            setData(result);
            setLoading(false);
          }
        })
        .catch(err => {
          if (!cancelled) {
            setError(err.message);
            setLoading(false);
          }
        });

      return () => { cancelled = true; };
    }, [props.id]);

    if (loading) return <Spinner />;
    if (error) return <ErrorMessage message={error} />;

    return <WrappedComponent {...props} data={data} />;
  };
}

// Использование
const UserProfileWithData = withData(UserProfile, fetchUserProfile);
const ProductDetailsWithData = withData(ProductDetails, fetchProduct);
```

### 3. Подписка на внешние источники

```jsx
function withSubscription(WrappedComponent, subscribeFn) {
  return function SubscribedComponent(props) {
    const [data, setData] = useState(() => subscribeFn.getData());

    useEffect(() => {
      const handleChange = () => setData(subscribeFn.getData());
      subscribeFn.subscribe(handleChange);
      return () => subscribeFn.unsubscribe(handleChange);
    }, []);

    return <WrappedComponent {...props} data={data} />;
  };
}
```

### 4. React.memo — встроенный HOC

`React.memo` — самый распространённый HOC в любом React-приложении. Он принимает компонент и возвращает мемоизированную версию, пропускающую ререндер при неизменных пропсах:

```jsx
const MemoizedButton = React.memo(function Button({ onClick, children }) {
  return <button onClick={onClick}>{children}</button>;
});
```

### 5. React.forwardRef — HOC для проброса ref

```jsx
const FancyInput = React.forwardRef(function FancyInput(props, ref) {
  return <input ref={ref} className="fancy" {...props} />;
});
```

---

## Композиция HOC

HOC легко комбинируются — один HOC оборачивает результат другого. Это позволяет собирать сложное поведение из простых кирпичиков:

```jsx
// Компонент получает: авторизацию + данные + логирование
const EnhancedDashboard = withLogging(
  withData(
    withAuth(DashboardPage),
    fetchDashboardData
  )
);
```

### Функция-композиция для читаемости

Вложенные вызовы читаются снизу вверх, что неудобно. Утилита `compose` (из Redux) инвертирует порядок:

```jsx
const compose = (...fns) => fns.reduce((a, b) => (...args) => a(b(...args)));

const enhance = compose(
  withAuth,
  withData(fetchDashboardData),
  withLogging
);

const EnhancedDashboard = enhance(DashboardPage);
// Читается слева направо: авторизация → данные → логирование
```

### Порядок имеет значение

Порядок композиции влияет на поведение:

```jsx
// withAuth снаружи: если нет пользователя, withData не вызовется
const A = withAuth(withData(Component, fetcher));

// withData снаружи: данные загружаются даже без авторизации
const B = withData(withAuth(Component), fetcher);
```

---

## Проблемы и подводные камни

### 1. «Обёртка в обёртке» в DevTools

Без `displayName` дерево компонентов превращается в кашу из безымянных функций. Всегда устанавливайте имя:

```jsx
EnhancedComponent.displayName = `WithData(${WrappedComponent.name})`;
```

### 2. Потеря статических методов

HOC возвращает новый компонент, и статические методы оригинала теряются:

```jsx
class Original extends React.Component {
  static fetchData = () => { /* ... */ };
  render() { return <div />; }
}

const Enhanced = withAuth(Original);
Enhanced.fetchData; // undefined!
```

Решение — библиотека `hoist-non-react-statics`, которая копирует нереактовские статические методы на обёртку:

```jsx
import hoistNonReactStatics from "hoist-non-react-statics";

function withAuth(WrappedComponent) {
  function EnhancedComponent(props) { /* ... */ }
  hoistNonReactStatics(EnhancedComponent, WrappedComponent);
  return EnhancedComponent;
}
```

### 3. Конфликт пропсов

Если HOC добавляет пропс `user`, и обёрнутый компонент тоже ожидает `user` — возникает коллизия. Решение — именовать пропсы с префиксом или использовать соглашение:

```jsx
// HOC добавляет authUser, а не user, чтобы избежать конфликта
return <WrappedComponent {...props} authUser={user} />;
```

### 4. Проблемы с типизацией в TypeScript

HOC усложняют вывод типов — обёрнутый компонент теряет часть типизации, и TypeScript не всегда корректно определяет пропсы. Хуки в этом плане значительно проще:

```tsx
// С HOC — нужно вручную пробрасывать типы
function withAuth<P extends { user: User }>(
  WrappedComponent: React.ComponentType<P>
): React.ComponentType<Omit<P, "user"> & { optionalUser?: User }> {
  // ...
}

// С хуком — типы работают естественно
function useAuth(): User | null { /* ... */ }
function Dashboard({ user }: { user: User }) { /* ... */ }
```

### 5. HOC внутри рендера — антипаттерн

Никогда не создавайте HOC внутри рендера другого компонента — это приведёт к размонтированию и пересозданию дерева при каждом ререндере:

```jsx
// ❌ Катастрофа: Enhanced создаётся заново при каждом рендере,
// что приводит к потере состояния и DOM
function Parent() {
  const Enhanced = useMemo(() => withAuth(MyComponent), []);
  return <Enhanced />;
}

// ✅ HOC создаётся один раз, на уровне модуля
const Enhanced = withAuth(MyComponent);
function Parent() {
  return <Enhanced />;
}
```

---

## HOC vs хуки

С появлением хуков в React 16.8 (2019) HOC во многом утратил актуальность как способ переиспользования логики с состоянием. Сравним:

### Один и тот же сценарий: подписка на window size

**HOC:**

```jsx
function withWindowSize(WrappedComponent) {
  return function WindowSizeComponent(props) {
    const [size, setSize] = useState({
      width: window.innerWidth,
      height: window.innerHeight,
    });

    useEffect(() => {
      const handleResize = () =>
        setSize({ width: window.innerWidth, height: window.innerHeight });
      window.addEventListener("resize", handleResize);
      return () => window.removeEventListener("resize", handleResize);
    }, []);

    return <WrappedComponent {...props} windowSize={size} />;
  };
}

// Использование
const MyComponent = withWindowSize(OriginalComponent);
```

**Хук:**

```jsx
function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    const handleResize = () =>
      setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return size;
}

// Использование
function MyComponent() {
  const size = useWindowSize();
  // ...
}
```

### Сравнительная таблица

| Критерий | HOC | Хуки |
|---|---|---|
| Переиспользование логики с состоянием | Да | Да |
| Прозрачность пропсов | Непрозрачная — добавляются «магические» пропсы | Прозрачная — переменная в компоненте |
| Типизация в TypeScript | Сложная — нужно пробрасывать дженерики | Простая — обычные типы |
| Вложенность | Глубокая вложенность обёрток | Плоский код, композиция через вызовы |
| DevTools | «Обёртка в обёртке», нужна displayName | Чистое дерево компонентов |
| Статические методы | Теряются, нужен hoist-non-react-statics | Не проблема |
| Совместимость с классами | Работает с классами и функциями | Только функциональные компоненты |
| Реальные библиотеки | `React.memo`, `connect` (Redux), `withRouter` | Практически все современные библиотеки |

### Почему хуки вытеснили HOC

1. **Прозрачность данных.** В хуке значение — локальная переменная, видно откуда взялось. В HOC — «магический» пропс, пришедший неизвестно откуда.
2. **Отсутствие обёрток.** Хук не создаёт дополнительный компонент в дереве — нет проблем с DevTools и DOM-структурой.
3. **Плоская композиция.** Несколько хуков вызываются последовательно, без вложенности.
4. **Лучшая типизация.** TypeScript работает естественно, без дженерик-конструкций.

---

## HOC vs Render Props

Render Props — ещё один паттерн переиспользования логики, популярный в переходный период между HOC и хуками. Вместо обёртки компонента HOC передаёт логику через проп-функцию:

```jsx
// Render Props
function WindowSize({ children }) {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });
  // ...подписка на resize...
  return children(size);
}

// Использование
<WindowSize>
  {({ width, height }) => <div>{width}x{height}</div>}
</WindowSize>
```

### Сравнение

| Критерий | HOC | Render Props |
|---|---|---|
| Структура дерева | Добавляет обёртку | Не добавляет (рендерит children) |
| Именование пропсов | HOC решает, какие пропсы добавить | Пользователь решает через деструктуризацию |
| JSX-вложенность | Не нужна | Может привести к «advent of Christmas» — глубокой вложенности render props |
| Типизация | Сложная | Проще, но всё ещё требует дженериков |

Оба паттерна в 2026 году в значительной степени заменены хуками, но встречаются в legacy-коде и некоторых библиотеках.

---

## Vue-аналоги

Во Vue паттерн HOC существует, но используется крайне редко. Основные механизмы переиспользования логики:

### Композаблы — современный аналог

```vue
<script setup>
import { ref, onMounted, onUnmounted } from "vue";

export function useWindowSize() {
  const width = ref(window.innerWidth);
  const height = ref(window.innerHeight);

  function onResize() {
    width.value = window.innerWidth;
    height.value = window.innerHeight;
  }

  onMounted(() => window.addEventListener("resize", onResize));
  onUnmounted(() => window.removeEventListener("resize", onResize));

  return { width, height };
}
</script>
```

### Миксины — legacy-аналог HOC

```js
// Миксин — объект с опциями, подмешиваемый в компонент
const windowSizeMixin = {
  data() {
    return { width: window.innerWidth, height: window.innerHeight };
  },
  mounted() {
    window.addEventListener("resize", this.onResize);
  },
  unmounted() {
    window.removeEventListener("resize", this.onResize);
  },
  methods: {
    onResize() {
      this.width = window.innerWidth;
      this.height = window.innerHeight;
    },
  },
};

export default {
  mixins: [windowSizeMixin],
  // ...
};
```

Миксины имеют серьёзные проблемы (неявные источники данных, конфликты имён), поэтому в Vue 3 предпочтительны композаблы — функционально эквивалентные React-хукам.

### HOC во Vue (технически возможен)

```js
import { defineComponent, h } from "vue";

function withAuth(WrappedComponent) {
  return defineComponent({
    setup(props, { attrs, slots }) {
      const user = useAuth();
      if (!user) return () => h(Redirect, { to: "/login" });
      return () => h(WrappedComponent, { ...props, ...attrs, user }, slots);
    },
  });
}
```

На практике это не нужно — композаблы покрывают те же сценарии проще и прозрачнее.

---

## Когда HOC всё ещё актуален

Несмотря на доминирование хуков, HOC остаётся востребованным в нескольких сценариях:

### 1. Встроенные HOC React

- **`React.memo`** — мемоизация компонентов.
- **`React.forwardRef`** — проброс ref через обёртку.
- **`React.lazy`** — ленивая загрузка компонентов.

Эти HOC — часть API React и не имеют альтернатив в виде хуков.

### 2. Библиотеки, поддерживающие классовые компоненты

Redux `connect`, React Router `withRouter` и другие библиотеки эпохи классов продолжают использовать HOC для обратной совместимости. В новых проектах они заменены хуками (`useSelector`, `useNavigate`), но в legacy-коде встречаются повсеместно.

### 3. Поведенческие обёртки, не связанные с состоянием

HOC удобен, когда нужно добавить компоненту поведение, не требующее собственного состояния:

```jsx
// Добавление data-testid ко всем элементам для тестирования
function withTestId(WrappedComponent, prefix) {
  return function WithTestId(props) {
    return <WrappedComponent {...props} data-testid={`${prefix}-${props.id}`} />;
  };
}
```

### 4. Интеграция с системами, работающими на уровне компонентов

Некоторые системы (Error Boundaries, Suspense-обёртки) требуют компонентного уровня — их нельзя реализовать хуком. HOC здесь естественный выбор:

```jsx
function withErrorBoundary(WrappedComponent, fallback) {
  return functionWithErrorBoundary(props) {
    return (
      <ErrorBoundary fallback={fallback}>
        <WrappedComponent {...props} />
      </ErrorBoundary>
    );
  };
}
```

---

## Итог

Higher-Order Components — важный паттерн в истории React, решавший задачу переиспользования логики до появления хуков. HOC — это функция, принимающая компонент и возвращающая новый компонент с расширенным поведением.

**Ключевые принципы:**
- HOC — чистая функция, не мутирующая исходный компонент.
- Пропсы передаются через, HOC добавляет свои.
- `displayName` обязателен для отладки.
- Композиция HOC позволяет собирать сложное поведение из простых кирпичиков.

**В 2026 году:**
- Для переиспользования логики с состоянием — используйте **хуки**.
- Для мемоизации компонентов — используйте **`React.memo`** (встроенный HOC).
- Для ленивой загрузки — **`React.lazy`** (встроенный HOC).
- Для Error Boundary и Suspense — HOC остаётся естественным выбором.
- В legacy-коде HOC встречается часто — понимание паттерна обязательно для чтения чужого кода.

HOC не умер — он занял своё место в нише, где хуки не работают: компонентные обёртки, Error Boundaries, мемоизация и интеграция с legacy-библиотеками.
