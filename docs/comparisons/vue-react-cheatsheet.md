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

## Ключевые отличия

- **Реактивность**: Vue автоматически отслеживает зависимости, React требует явного указания deps
- **Мутабельность**: Vue `ref.value = x`, React `setState(x)` (нельзя мутировать state)
- **Ререндер**: Vue перерисовывает только компоненты с изменёнными данными, React — весь компонент и детей
- **Двустороннее связывание**: Vue `v-model`, React — контролируемые компоненты
- **Шаблоны**: Vue — template-синтаксис, React — JSX (JavaScript)
