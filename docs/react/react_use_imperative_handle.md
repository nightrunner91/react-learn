# useImperativeHandle

Хук настраивает, что доступно через `ref` у родительского компонента. Позволяет явно определить публичный API компонента, скрывая внутреннюю реализацию.

## Аналогия с Vue

В Vue 3 Composition API аналог — `defineExpose()`:

```vue
<!-- Vue -->
<script setup>
import { ref } from 'vue';
const inputRef = ref();

defineExpose({
  focus: () => inputRef.value.focus(),
  getValue: () => inputRef.value,
});
</script>

<template><input ref="inputRef" /></template>
```

```jsx
// React
const Input = forwardRef((props, ref) => {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    getValue: () => inputRef.current.value,
  }), []);
  return <input ref={inputRef} />;
});
```

Оба механизма решают одну задачу: ограничить то, что родитель может делать с дочерним компонентом через ref.

## Синтаксис

```jsx
useImperativeHandle(ref, createHandle, [deps]);
```

- `ref` — объект ref из `forwardRef`
- `createHandle` — функция, возвращающая объект с методами/свойствами
- `deps` — массив зависимостей (как в `useEffect`)

## Базовый пример

```jsx
const CustomInput = forwardRef((props, ref) => {
  const inputRef = useRef();

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    blur: () => inputRef.current.blur(),
    getValue: () => inputRef.current.value,
    setValue: (val) => { inputRef.current.value = val; },
  }), []);

  return <input ref={inputRef} {...props} />;
});

function Form() {
  const inputRef = useRef();
  const handleClick = () => {
    inputRef.current.focus();
    console.log(inputRef.current.getValue());
  };
  return <CustomInput ref={inputRef} />;
}
```

## Зависимости

Если handle использует значения из пропсов или состояния, укажите их в deps:

```jsx
useImperativeHandle(ref, () => ({
  greet: () => alert(`Hello, ${name}`),
}), [name]);
```

Без deps метод пересоздаётся при каждом рендере.

## Когда использовать

- Работа с DOM: фокус, прокрутка, анимации
- Управление вложенными компонентами (плеер, модалка)
- Интеграция с императивными библиотеками

## Когда НЕ использовать

- Если данные можно передать через props
- Для потоков данных — используйте props/callbacks
## Практический пример: VideoPlayer

```jsx
const VideoPlayer = forwardRef((props, ref) => {
  const videoRef = useRef();
  useImperativeHandle(ref, () => ({
    play: () => videoRef.current.play(),
    pause: () => videoRef.current.pause(),
    seek: (time) => { videoRef.current.currentTime = time; },
  }), []);
  return <video ref={videoRef} src={props.src} />;
});
```

## Ограничения

- Работает только с `forwardRef`
- Нельзя использовать для передачи данных в обратном направлении
- В Vue `defineExpose` работает без обёрток, в React нужен `forwardRef`