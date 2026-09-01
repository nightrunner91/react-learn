# Шпаргалка Vue <-> React

Справочник соответствий между Vue 3 Composition API и React.

## Реактивность и состояние

| Vue | React | Описание |
|-----|-------|----------|
| `ref()` | `useState()` | Реактивное состояние |
| `reactive()` | `useState({})` | Реактивный объект |
| `computed()` | `useMemo()` | Вычисляемое значение |
| `watch()` | `useEffect()` | Слежение за изменениями |
| `watchEffect()` | `useEffect(fn)` | Авто-слежение за зависимостями |

```js
// Vue
const count = ref(0);
const doubled = computed(() => count.value * 2);
watch(count, (newVal) => console.log(newVal));
```

```jsx
// React
const [count, setCount] = useState(0);
const doubled = useMemo(() => count * 2, [count]);
useEffect(() => console.log(count), [count]);
```

## Жизненный цикл

| Vue | React | Когда вызывается |
|-----|-------|------------------|
| `onMounted()` | `useEffect(() => {}, [])` | После монтирования |
| `onUpdated()` | `useEffect(() => {})` | После обновления |
| `onUnmounted()` | `useEffect(() => () => cleanup, [])` | Перед размонтированием |
| `onBeforeMount()` | — | До монтирования |
| `onBeforeUpdate()` | — | До обновления |
| `onBeforeUnmount()` | — | Перед размонтированием |

## Props и события

| Vue | React |
|-----|-------|
| `defineProps()` | `props` (параметр компонента) |
| `defineEmits()` | Callback-функции в props |
| `slots` | `children` или именованные props |

```vue
<!-- Vue -->
<script setup>
const props = defineProps(['title']);
const emit = defineEmits(['update']);
</script>
<template><button @click="emit('update')">{{ title }}</button></template>
```

```jsx
// React
function Button({ title, onUpdate }) {
  return <button onClick={onUpdate}>{title}</button>;
}
```

## Provide / Inject vs Context

| Vue | React |
|-----|-------|
| `provide()` / `inject()` | `createContext()` + `useContext()` |

```js
// Vue
provide('theme', 'dark');
const theme = inject('theme');
```

```jsx
// React
const ThemeContext = createContext();
<ThemeContext.Provider value="dark">
  <Child />
</ThemeContext.Provider>
const theme = useContext(ThemeContext);
```

## Template Refs

| Vue | React |
|-----|-------|
| `ref="input"` + `const input = ref()` | `useRef()` + `forwardRef` |
| `defineExpose()` | `useImperativeHandle()` |

```vue
<!-- Vue -->
<script setup>
const input = ref();
defineExpose({ focus: () => input.value.focus() });
</script>
<template><input ref="input" /></template>
```

```jsx
// React
const Input = forwardRef((props, ref) => {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
  }), []);
  return <input ref={inputRef} />;
});
```

## Директивы vs JSX

| Vue | React |
|-----|-------|
| `v-if` | `{condition && <Element />}` |
| `v-else` | `{condition ? <A /> : <B />}` |
| `v-show` | `style={{ display: show ? 'block' : 'none' }}` |
| `v-for` | `{items.map(item => <Element key={item.id} />)}` |
| `v-model` | `value={val}` + `onChange={e => setVal(e.target.value)}` |
| `v-bind` (`:`) | `{...props}` или `prop={value}` |
| `v-on` (`@`) | `onClick={handler}` |
| `v-html` | `dangerouslySetInnerHTML={{ __html: html }}` |

## Composition API vs Hooks

| Vue | React |
|-----|-------|
| `<script setup>` | Функциональный компонент |
| `ref()`, `reactive()` | `useState()` |
| `computed()` | `useMemo()` |
| `watch()` | `useEffect()` |
| `onMounted()` | `useEffect(() => {}, [])` |
| `provide()` / `inject()` | `useContext()` |
| `defineExpose()` | `useImperativeHandle()` |
| `defineProps()` | `props` |
| `defineEmits()` | Callback props |

## Suspense и асинхронные компоненты

| Vue | React | Описание |
|-----|-------|----------|
| `<Suspense>` | `<Suspense>` | Отложенная загрузка асинхронных компонентов |
| `defineAsyncComponent()` | `React.lazy()` + `Suspense` | Ленивая загрузка компонентов |

```vue
<!-- Vue -->
<template>
  <Suspense>
    <template #default><AsyncMap /></template>
    <template #fallback><Loader /></template>
  </Suspense>
</template>
```

```jsx
// React
const AsyncMap = React.lazy(() => import('./Map'));

<Suspense fallback={<Loader />}>
  <AsyncMap />
</Suspense>
```

## Keep-alive / кэширование состояния

| Vue | React | Описание |
|-----|-------|----------|
| `<KeepAlive>` | Нет прямого аналога | Сохранение состояния размонтированных компонентов |
| — | Собственная реализация через `useRef` / state lifting | React не имеет встроенного KeepAlive |

В React для аналогичного поведения обычно поднимают состояние выше или используют библиотеки.

## Transition / анимации

| Vue | React | Описание |
|-----|-------|----------|
| `<Transition>` / `<TransitionGroup>` | `framer-motion`, `react-transition-group` | Анимация появления/исчезновения элементов |

```vue
<!-- Vue -->
<Transition name="fade">
  <div v-if="show">Hello</div>
</Transition>
```

```jsx
// React (framer-motion)
import { motion, AnimatePresence } from 'framer-motion';

<AnimatePresence>
  {show && (
    <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}>
      Hello
    </motion.div>
  )}
</AnimatePresence>
```

## Computed с setter

```vue
<!-- Vue -->
<script setup>
const firstName = ref('John');
const lastName = ref('Doe');

const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (val) => {
    [firstName.value, lastName.value] = val.split(' ');
  },
});
</script>
```

```jsx
// React
const [firstName, setFirstName] = useState('John');
const [lastName, setLastName] = useState('Doe');

const fullName = useMemo(
  () => `${firstName} ${lastName}`,
  [firstName, lastName]
);

function setFullName(val) {
  const [first, last] = val.split(' ');
  setFirstName(first);
  setLastName(last);
}
```

## Provide / Inject и реактивность

```vue
<!-- Vue -->
<script setup>
import { provide, ref } from 'vue';
const theme = ref('dark');
provide('theme', theme); // реактивная ссылка
</script>
```

```jsx
// React
const ThemeContext = createContext({ theme: 'dark', setTheme: () => {} });

function Provider({ children }) {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

## watch vs useEffect: важные нюансы

| Аспект | Vue `watch` | React `useEffect` |
|--------|-------------|-------------------|
| Отслеживание | Явно указываем источники | Зависимости в массиве |
| Deep watch | `{ deep: true }` | Нужна ручная сериализация зависимостей |
| Immediate | `{ immediate: true }` | Пустой deps или deps с начальным значением |
| Очистка | Возвращаемая функция | Cleanup-функция |

```vue
<!-- Vue: глубокое отслеживание -->
<script setup>
watch(user, (newVal, oldVal) => { ... }, { deep: true });
</script>
```

```jsx
// React: нужно следить за стабильностью зависимостей
useEffect(() => {
  // ...
}, [JSON.stringify(user)]); // или кастомный хук useDeepCompareEffect
```

## Модификаторы событий

| Vue | React | Примечание |
|-----|-------|------------|
| `@click.stop` | `e.stopPropagation()` | React без модификаторов |
| `@submit.prevent` | `e.preventDefault()` | Вручную в обработчике |
| `@keydown.enter` | `if (e.key === 'Enter')` | Проверка в обработчике |

```vue
<!-- Vue -->
<form @submit.prevent="handleSubmit">
  <input @keydown.enter="submit" />
</form>
```

```jsx
// React
<form onSubmit={(e) => { e.preventDefault(); handleSubmit(); }}>
  <input onKeyDown={(e) => e.key === 'Enter' && submit()} />
</form>
```

## Ключевые отличия

- **Реактивность**: Vue автоматически отслеживает зависимости, React требует явного указания deps.
- **Мутабельность**: Vue `ref.value = x`, React `setState(x)` (нельзя мутировать state).
- **Ререндер**: Vue обновляет только затронутые компоненты, React — весь компонент и детей (если не используется мемоизация).
- **Двустороннее связывание**: Vue `v-model`, React — контролируемые компоненты.
- **Шаблоны**: Vue — template-синтаксис с директивами, React — JSX (JavaScript).
- **Асинхронные компоненты**: оба фреймворка имеют `Suspense`, но API загрузки отличается.
- **Анимации и KeepAlive**: у Vue встроенные решения, в React — внешние библиотеки.
