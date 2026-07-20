# Синтетические события в React

Синтетические события (Synthetic Events) — это кросс-браузерная обёртка React над нативными DOM-событиями. Они обеспечивают единый API для обработки событий, абстрагируя разработчика от различий между браузерами и оптимизируя производительность через делегирование.

---

## Что такое синтетические события

Каждый обработчик события в React получает объект `SyntheticEvent` — экземпляр класса, который оборачивает нативное браузерное событие и предоставляет стандартизированный интерфейс:

```jsx
function Button() {
  const handleClick = (e) => {
    // e — SyntheticEvent, а не нативный MouseEvent
    e.preventDefault();
    e.stopPropagation();
    console.log(e.target);       // стандартизированный API
    console.log(e.nativeEvent);  // доступ к оригинальному событию
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

### Ключевые свойства SyntheticEvent

| Свойство | Описание |
|---|---|
| `e.nativeEvent` | Оригинальное браузерное событие (MouseEvent, KeyboardEvent и т.д.) |
| `e.target` | Элемент, инициировавший событие |
| `e.currentTarget` | Элемент, на котором висит обработчик |
| `e.type` | Тип события (`'click'`, `'change'` и т.д.) |
| `e.preventDefault()` | Отмена действия по умолчанию |
| `e.stopPropagation()` | Остановка всплытия |

---

## Как работает система событий React

### Делегирование событий

React не вешает обработчики на каждый элемент. Вместо этого используется **один делегированный listener** на корне приложения:

```
DOM-структура:
<div id="root">                    ← здесь слушает React
  <div>
    <button onClick={handler}>     ← нет listener'а
      <span>Click</span>           ← нет listener'а
    </button>
  </div>
</div>
```

Когда пользователь кликает на `<span>`, событие всплывает до `#root`, где React перехватывает его, определяет целевой компонент и вызывает нужный обработчик.

### Эволюция делегирования

**React ≤ 16:** делегирование на `document`

```js
// Все события слушаются на document
document.addEventListener('click', reactDispatcher);
```

**React 17+:** делегирование на root-контейнер

```js
const root = document.getElementById('root');
ReactDOM.createRoot(root).render(<App />);
// События слушаются на root, не на document
```

Это изменение решило проблемы:
- Несколько React-приложений на одной странице не конфликтуют
- Микрофронтенды могут сосуществовать без перехвата событий
- Легче интегрировать React в существующие приложения

---

## Зачем нужны синтетические события

### 1. Кросс-браузерная совместимость

React нормализует различия между браузерами. Например, `e.target` всегда работает одинаково, независимо от того, как браузер обрабатывает события:

```jsx
// В IE/старых браузерах event.target может отличаться
// React гарантирует единообразное поведение
const handleChange = (e) => {
  console.log(e.target.value); // работает везде одинаково
};
```

### 2. Производительность через делегирование

Один listener на всё приложение вместо тысяч на каждом элементе:

```jsx
// Без делегирования: 1000 кнопок = 1000 listeners
// С делегированием: 1 listener на root
function List() {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => handleItemClick(item.id)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

### 3. Интеграция с системой обновлений React

Синтетические события участвуют в батчинге обновлений:

```jsx
// React 18+: автоматический батчинг работает для всех событий
function Form() {
  const handleClick = () => {
    setA(1);  // все три обновления — один рендер
    setB(2);
    setC(3);
  };

  return <button onClick={handleClick}>Update</button>;
}
```

---

## Изменения в React 17+

### Удаление пула событий (Event Pooling)

**React 16 и ниже:** SyntheticEvent пулился для экономии памяти. После обработки события объект обнулялся:

```jsx
// React 16 — НЕ РАБОТАЕТ
const handleClick = async (e) => {
  await fetchData();
  console.log(e.target); // null! Событие уже пулировано
};

// React 16 — нужно вызывать persist()
const handleClick = async (e) => {
  e.persist(); // сохранить событие
  await fetchData();
  console.log(e.target); // работает
};
```

**React 17+:** пул событий удалён. SyntheticEvent можно использовать асинхронно:

```jsx
// React 17+ — РАБОТАЕТ
const handleClick = async (e) => {
  await fetchData();
  console.log(e.target); // SyntheticEvent доступен
  console.log(e.nativeEvent); // тоже доступен
};
```

### Изменение делегирования

React 17 изменил точку делегирования с `document` на root-контейнер. Это важно для микрофронтендов и встраивания нескольких React-приложений.

---

## Доступ к нативным событиям

Если нужен полный контроль над нативным событием:

```jsx
const handleClick = (e) => {
  // Синтетическое событие
  e.stopPropagation();

  // Нативное событие
  e.nativeEvent.stopImmediatePropagation();

  // Информация о браузере
  console.log(e.nativeEvent.timeStamp);
  console.log(e.nativeEvent.isTrusted);
};
```

---

## Синтетические события в Concurrent Mode

В React 18 с Concurrent Mode синтетические события работают с приоритетами:

```jsx
function SearchInput() {
  const handleChange = (e) => {
    // Обновление стейта через событие
    // React может прервать рендер, если придёт более приоритетное событие
    setQuery(e.target.value);
  };

  return <input onChange={handleChange} />;
}
```

### Приоритеты событий

React назначает приоритеты событиям для Concurrent Mode:

| Событие | Приоритет |
|---|---|
| `click`, `keydown`, `keyup` | Discrete (высокий) |
| `input`, `change` | Continuous (средний) |
| `pointerMove`, `drag` | Default (низкий) |

Это позволяет React прерывать обработку low-priority событий ради high-priority.

---

## Типы поддерживаемых событий

React поддерживает все стандартные DOM-события:

```jsx
// Mouse Events
onClick, onContextMenu, onDoubleClick, onDrag, onDrop

// Keyboard Events
onKeyDown, onKeyUp, onKeyPress

// Focus Events
onFocus, onBlur

// Form Events
onChange, onInput, onSubmit, onReset

// Touch Events
onTouchStart, onTouchMove, onTouchEnd, onTouchCancel

// Pointer Events
onPointerDown, onPointerMove, onPointerUp, onPointerCancel

// Wheel Events
onWheel, onScroll

// Media Events
onPlay, onPause, onEnded, onVolumeChange
```

Полный список: [React Events Documentation](https://reactjs.org/docs/events.html)

---

## Практические примеры

### Drag & Drop с синтетическими событиями

```jsx
function DraggableItem({ item }) {
  const handleDragStart = (e) => {
    e.dataTransfer.setData('text/plain', JSON.stringify(item));
    e.dataTransfer.effectAllowed = 'move';
  };

  return (
    <div
      draggable
      onDragStart={handleDragStart}
    >
      {item.name}
    </div>
  );
}

function DropZone() {
  const handleDrop = (e) => {
    e.preventDefault();
    const data = JSON.parse(e.dataTransfer.getData('text/plain'));
    console.log('Dropped:', data);
  };

  const handleDragOver = (e) => {
    e.preventDefault(); // обязательно для drop
    e.dataTransfer.dropEffect = 'move';
  };

  return (
    <div
      onDrop={handleDrop}
      onDragOver={handleDragOver}
    >
      Drop here
    </div>
  );
}
```

### Кастомные жесты

```jsx
function SwipeableCard({ onSwipeLeft, onSwipeRight }) {
  const touchStart = useRef(null);

  const handleTouchStart = (e) => {
    touchStart.current = e.touches[0].clientX;
  };

  const handleTouchEnd = (e) => {
    if (!touchStart.current) return;

    const touchEnd = e.changedTouches[0].clientX;
    const diff = touchEnd - touchStart.current;

    if (Math.abs(diff) > 50) {
      if (diff > 0) {
        onSwipeRight();
      } else {
        onSwipeLeft();
      }
    }
  };

  return (
    <div
      onTouchStart={handleTouchStart}
      onTouchEnd={handleTouchEnd}
    >
      Swipe me
    </div>
  );
}
```

---

## Когда синтетические события могут быть проблемой

### 1. Интеграция со сторонними библиотеками

Если библиотека ожидает нативные события, нужно использовать `e.nativeEvent`:

```jsx
import { someLibrary } from 'third-party-lib';

const handleClick = (e) => {
  // someLibrary ожидает нативный Event
  someLibrary.process(e.nativeEvent);
};
```

### 2. Прямая работа с DOM

Если используете `addEventListener` напрямую, синтетические события не помогут:

```jsx
function MapComponent() {
  const containerRef = useRef(null);

  useEffect(() => {
    const element = containerRef.current;

    // Нативный listener — нет SyntheticEvent
    const handler = (e) => {
      console.log(e); // нативный MouseEvent
    };

    element.addEventListener('click', handler);

    return () => {
      element.removeEventListener('click', handler);
    };
  }, []);

  return <div ref={containerRef} />;
}
```

### 3. Stop propagation и React Portals

Portals создают отдельную точку монтирования, но события всё равно всплывают через React-дерево:

```jsx
function Modal() {
  const handleClick = (e) => {
    // Событие всплывёт до родителя в React-дереве,
    // даже если Modal отрендерен в другом DOM-узле
    e.stopPropagation();
  };

  return ReactDOM.createPortal(
    <div onClick={handleClick}>Modal Content</div>,
    document.getElementById('modal-root')
  );
}

function Parent() {
  const handleClick = () => {
    console.log('Parent clicked'); // вызовется, если не остановить
  };

  return (
    <div onClick={handleClick}>
      <Modal />
    </div>
  );
}
```

---

## Отличия от Vue

| Аспект | React | Vue |
|---|---|---|
| Обёртка событий | SyntheticEvent | Нативный Event |
| Делегирование | На root/document | На элементе |
| Модификаторы | Вручную (`e.stopPropagation()`) | `.stop`, `.prevent` и т.д. |
| Пул событий | Удалён в React 17 | Нет пула |

```vue
<!-- Vue: модификаторы упрощают работу -->
<button @click.stop="handleClick">Click</button>
<button @click.prevent="handleSubmit">Submit</button>
<button @keyup.enter="handleEnter">Enter</button>
```

```jsx
// React: всё вручную
<button onClick={(e) => { e.stopPropagation(); handleClick(); }}>Click</button>
<button onClick={(e) => { e.preventDefault(); handleSubmit(); }}>Submit</button>
<input onKeyUp={(e) => { if (e.key === 'Enter') handleEnter(); }} />
```

---

## Итог

Синтетические события — основа системы обработки событий в React. Они обеспечивают:

- **Кросс-браузерную совместимость** через единый API
- **Производительность** через делегирование
- **Интеграцию** с системой обновлений и батчингом
- **Поддержку Concurrent Mode** через приоритеты

В React 18 синтетические события стали проще (удалён пул событий) и эффективнее (делегирование на root). Для повседневной разработки они работают прозрачно, но понимание внутренней механики помогает при интеграции со сторонними библиотеками и оптимизации производительности.
