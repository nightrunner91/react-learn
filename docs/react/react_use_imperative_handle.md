# useImperativeHandle

`useImperativeHandle` — хук, который настраивает, какие методы и свойства будут доступны родительскому компоненту через `ref`. Он позволяет явно определить публичный API дочернего компонента, скрывая внутреннюю реализацию.

## Когда нужен

React по умолчанию не даёт родителю доступ к внутреннему DOM или методам дочернего компонента. `useImperativeHandle` вместе с `forwardRef` решает эту задачу, но используется только тогда, когда императивный доступ действительно необходим.

Типичные сценарии:

- Фокус, прокрутка, выделение текста.
- Управление плеером (`play`, `pause`, `seek`).
- Управление анимациями (например, через библиотеку GSAP).
- Интеграция с императивными сторонними библиотеками.

## Синтаксис

```jsx
useImperativeHandle(ref, createHandle, deps);
```

- `ref` — объект ref из `forwardRef`.
- `createHandle` — функция, возвращающая объект с методами/свойствами, доступными родителю.
- `deps` — массив зависимостей (как в `useEffect`).

## Базовый пример

```jsx
import { forwardRef, useRef, useImperativeHandle } from 'react';

const CustomInput = forwardRef((props, ref) => {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    blur: () => inputRef.current?.blur(),
    getValue: () => inputRef.current?.value,
    setValue: (val) => {
      if (inputRef.current) inputRef.current.value = val;
    },
  }), []);

  return <input ref={inputRef} {...props} />;
});

function Form() {
  const inputRef = useRef(null);

  const handleClick = () => {
    inputRef.current?.focus();
    console.log(inputRef.current?.getValue());
  };

  return (
    <>
      <CustomInput ref={inputRef} />
      <button onClick={handleClick}>Focus</button>
    </>
  );
}
```

## Зависимости

Если handle использует значения из пропсов или состояния, укажите их в deps:

```jsx
useImperativeHandle(ref, () => ({
  greet: () => alert(`Hello, ${name}`),
}), [name]);
```

Без deps метод будет пересоздаваться при каждом рендере, что может привести к лишним вычислениям.

## React 19: ref как проп

В React 19 `forwardRef` больше не обязателен — `ref` передаётся как обычный проп:

```jsx
function CustomInput({ label, ref }) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }), []);

  return <input ref={inputRef} />;
}
```

Это упрощает типизацию и убирает необходимость обёртки `forwardRef`.

## Практический пример: VideoPlayer

```jsx
const VideoPlayer = forwardRef(({ src }, ref) => {
  const videoRef = useRef(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
    seek: (time) => {
      if (videoRef.current) videoRef.current.currentTime = time;
    },
    getCurrentTime: () => videoRef.current?.currentTime || 0,
  }), []);

  return <video ref={videoRef} src={src} controls />;
});

function App() {
  const playerRef = useRef(null);

  return (
    <>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current?.play()}>Play</button>
      <button onClick={() => playerRef.current?.seek(120)}>Seek to 2:00</button>
    </>
  );
}
```

## TypeScript

```tsx
import { forwardRef, useRef, useImperativeHandle } from 'react';

interface InputHandle {
  focus: () => void;
  getValue: () => string;
}

interface InputProps {
  placeholder?: string;
}

const CustomInput = forwardRef<InputHandle, InputProps>(
  ({ placeholder }, ref) => {
    const inputRef = useRef<HTMLInputElement>(null);

    useImperativeHandle(ref, () => ({
      focus: () => inputRef.current?.focus(),
      getValue: () => inputRef.current?.value || '',
    }), []);

    return <input ref={inputRef} placeholder={placeholder} />;
  }
);

function Form() {
  const inputRef = useRef<InputHandle>(null);

  return (
    <>
      <CustomInput ref={inputRef} placeholder="Enter text" />
      <button onClick={() => inputRef.current?.focus()}>Focus</button>
    </>
  );
}
```

## Аналогия с Vue

В Vue 3 Composition API аналог — `defineExpose()`:

```vue
<script setup>
import { ref } from 'vue';
const inputRef = ref();

defineExpose({
  focus: () => inputRef.value?.focus(),
  getValue: () => inputRef.value?.value,
});
</script>

<template><input ref="inputRef" /></template>
```

Оба механизма решают одну задачу: ограничить публичный API дочернего компонента, доступный через ref.

## Когда использовать

- Работа с DOM: фокус, прокрутка, анимации.
- Управление вложенными компонентами (плеер, карта, редактор).
- Интеграция с императивными библиотеками.

## Когда НЕ использовать

- Если данные можно передать через props — используйте props.
- Для потока данных в обратном направлении — используйте callback-пропсы.
- Как замену state management — это приведёт к запутанной архитектуре.

## Ограничения и подводные камни

- Работает только с `forwardRef` (до React 19) или с `ref` как пропом (React 19+).
- Нельзя использовать для передачи данных в обратном направлении.
- Не заменяет нормальное проектирование пропсов.
- В Vue `defineExpose` работает без обёрток, в React до 19 нужен `forwardRef`.

## Итог

`useImperativeHandle` — мощный, но редко нужный инструмент. Используйте его, когда без императивного доступа к DOM или внутренним методам не обойтись. Во всех остальных случаях отдавайте предпочтение props и композиции.
