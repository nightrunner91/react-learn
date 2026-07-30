# useTransition и useDeferredValue — управление приоритетами в Concurrent Mode

React 18 представил **конкурентный режим** (Concurrent Mode) — новую модель рендеринга, позволяющую React прерывать и возобновлять работу над обновлениями. Ключевая идея: не все обновления равнозначны. Клик пользователя важнее, чем фильтрация списка из 10 000 элементов. Хуки `useTransition` и `useDeferredValue` дают разработчику инструмент для явной маркировки приоритетов.

---

## Приоритеты обновлений в Concurrent Mode

React классифицирует обновления по срочности. Каждое обновление состояния получает приоритет, определяющий, когда и как React будет его обрабатывать:

### Уровни приоритета (от высшего к низшему)

| Приоритет | Описание | Примеры |
|-----------|----------|---------|
| **Sync** (синхронный) | Наивысший, не может быть прерван. React обрабатывает немедленно и блокирующе. | `flushSync`, обновления из `useLayoutEffect`, события `pointerdown` |
| **InputContinuous** (непрерывный ввод) | Высокий, для непрерывных взаимодействий. Может прерывать default-приоритет, но уступает sync. | `pointermove`, `touchmove`, `wheel`, `drag` |
| **Default** (по умолчанию) | Средний. Стандартные обновления состояния. | `useState`, `useReducer`, большинство кликов, сетевые запросы |
| **Transition** (переход) | Низкий. Для несрочных обновлений, которые можно отложить. | `useTransition`, `startTransition` |
| **Idle** (простой) | Самый низкий. Для обновлений, которые должны происходить только когда браузер простаивает. | `requestIdleCallback` (экспериментально) |

### Как работает прерывание

Когда React обрабатывает обновление с низким приоритетом (например, transition) и поступает обновление с высоким приоритетом (например, клик), React:

1. **Приостанавливает** текущую работу над transition-обновлением.
2. **Обрабатывает** срочное обновление (клик).
3. **Возобновляет** transition-обновление с того места, где остановился.

Это позволяет интерфейсу оставаться отзывчивым даже при тяжёлых вычислениях.

### Практическое правило

- **Sync** — используйте редко, только когда нужно гарантировать синхронное обновление DOM (например, для измерений).
- **Default** — стандартное поведение, не требует явного указания.
- **Transition** — для тяжёлых вычислений, которые не блокируют взаимодействие пользователя.
- **Idle** — для фоновых задач, не критичных для UX.

---

## useTransition — явная маркировка несрочных обновлений

`useTransition` позволяет поместить обновление состояния в **transition-приоритет**, явно указывая React, что это обновление можно отложить.

### Сигнатура

```js
const [isPending, startTransition] = useTransition();
```

**Возвращает:**
- `isPending` — `true`, пока transition-обновление обрабатывается.
- `startTransition` — функция, принимающая колбэк с обновлениями состояния.

### Базовый пример: фильтрация списка

Классический сценарий: пользователь вводит текст в инпут, а список из тысяч элементов фильтруется. Без `useTransition` каждый символ ввода блокирует интерфейс на время фильтрации.

```jsx
import { useState, useTransition } from 'react';

function SearchableList({ items }) {
  const [query, setQuery] = useState('');
  const [filteredItems, setFilteredItems] = useState(items);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    const value = e.target.value;
    
    // Срочное обновление: инпут обновляется немедленно
    setQuery(value);

    // Несрочное обновление: фильтрация может подождать
    startTransition(() => {
      setFilteredItems(
        items.filter(item =>
          item.name.toLowerCase().includes(value.toLowerCase())
        )
      );
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={handleChange}
        placeholder="Поиск..."
      />
      
      {isPending ? (
        <div className="spinner">Загрузка результатов...</div>
      ) : (
        <ul>
          {filteredItems.map(item => (
            <li key={item.id}>{item.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**Что происходит:**
1. Пользователь вводит символ → `setQuery` обновляет инпут немедленно (default-приоритет).
2. `startTransition` помещает фильтрацию в transition-приоритет.
3. Если пользователь продолжает вводить быстро, React прерывает фильтрацию и обрабатывает следующий символ.
4. `isPending` показывает, что фильтрация в процессе.
5. Когда фильтрация завершена, `isPending` становится `false`, и список обновляется.

### Пример: переключение вкладок с тяжёлым контентом

```jsx
function TabContainer() {
  const [activeTab, setActiveTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  const handleTabChange = (tab) => {
    startTransition(() => {
      setActiveTab(tab);
    });
  };

  return (
    <div>
      <nav>
        <button onClick={() => handleTabChange('home')}>Home</button>
        <button onClick={() => handleTabChange('settings')}>Settings</button>
        <button onClick={() => handleTabChange('profile')}>Profile</button>
      </nav>

      {isPending && <div className="transition-indicator">Загрузка...</div>}

      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        {activeTab === 'home' && <HomeTab />}
        {activeTab === 'settings' && <SettingsTab />}
        {activeTab === 'profile' && <ProfileTab />}
      </div>
    </div>
  );
}
```

Здесь переключение вкладок помечается как transition, что позволяет React отрисовать новую вкладку в фоне, не блокируя интерфейс.

### ⚠️ Распространённое заблуждение: useTransition ≠ анимация

Название `useTransition` сбивает с толку. **"Transition" здесь — не CSS-анимация**, а термин из конкурентного режима React, означающий "переход между состояниями с низким приоритетом".

**Что делает `useTransition`:**
- Понижает приоритет обновления (React может его прервать)
- Возвращает флаг `isPending` для визуальной обратной связи
- Список "запаздывает" — показывает старые данные, пока идёт фильтрация

**Что НЕ делает `useTransition`:**
- Не анимирует элементы автоматически
- Не добавляет плавные переходы
- Не заменяет CSS `transition` или `framer-motion`

Визуальную обратную связь (спиннер, opacity, skeleton) добавляешь **ты сам** через `isPending`:

```jsx
// Пример: полупрозрачность списка во время фильтрации
<ul style={{
  opacity: isPending ? 0.5 : 1,
  transition: 'opacity 0.2s ease' // ← это CSS-анимация, не React
}}>
  {filtered.map(item => <li key={item.id}>{item.name}</li>)}
</ul>
```

**Почему название "transition"?**  
Это отсылка к "transition between states" — переход UI из одного состояния в другое, где сам переход не срочный. Не путай с CSS `transition`.

### Когда использовать useTransition

- **Тяжёлые вычисления на стороне клиента:** фильтрация, сортировка, группировка больших массивов.
- **Переключение между сложными представлениями:** вкладки, роуты, модальные окна с тяжёлым контентом.
- **Анимации, зависящие от состояния:** когда обновление состояния вызывает перерисовку множества компонентов.

### Ограничения useTransition

- Работает только с обновлениями состояния внутри колбэка `startTransition`.
- Не подходит для обновлений, которые должны произойти синхронно (например, обновление позиции при перетаскивании).
- Требует React 18+.

---

## useDeferredValue — отложенное значение для неконтролируемых обновлений

`useDeferredValue` решает ту же задачу, что и `useTransition`, но для случаев, когда вы **не контролируете источник обновления**. Типичный сценарий: значение приходит через пропсы от родительского компонента.

### Сигнатура

```js
const deferredValue = useDeferredValue(value);
```

**Возвращает:** отложенную версию переданного значения. React обновляет её только после завершения всех срочных обновлений.

### Базовый пример: поиск с пропсами

```jsx
function Parent() {
  const [query, setQuery] = useState('');

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Поиск..."
      />
      <SearchResults query={query} />
    </div>
  );
}

function SearchResults({ query }) {
  // Отложенная версия query
  const deferredQuery = useDeferredValue(query);
  
  // Флаг: текущее значение отстаёт от актуального
  const isStale = query !== deferredQuery;

  return (
    <div style={{ 
      opacity: isStale ? 0.5 : 1,
      transition: 'opacity 0.2s'
    }}>
      <HeavySearchResults query={deferredQuery} />
    </div>
  );
}
```

**Что происходит:**
1. Пользователь вводит текст → `query` обновляется немедленно.
2. `deferredQuery` остаётся старым, пока React обрабатывает срочные обновления.
3. `isStale` становится `true`, и контейнер становится полупрозрачным.
4. React завершает срочные обновления и обновляет `deferredQuery`.
5. `HeavySearchResults` перерисовывается с новым значением.
6. `isStale` становится `false`, прозрачность возвращается к 1.

### Сравнение: useTransition vs useDeferredValue

| Критерий | useTransition | useDeferredValue |
|----------|---------------|------------------|
| **Контроль над источником** | Вы контролируете сеттер состояния | Значение приходит извне (пропсы, контекст) |
| **Синтаксис** | `startTransition(() => setState(...))` | `useDeferredValue(value)` |
| **Гибкость** | Можно обернуть несколько обновлений в один transition | Работает с одним значением |
| **Индикатор загрузки** | `isPending` из хука | Сравнение `value !== deferredValue` |
| **Типичный сценарий** | Локальное состояние компонента | Пропсы от родителя |

### Когда использовать useDeferredValue

- **Значение приходит через пропсы:** родительский компонент обновляет состояние, дочерний хочет отложить реакцию.
- **Контекст:** значение из `useContext` обновляется часто, но тяжёлый компонент может реагировать с задержкой.
- **Нет доступа к сеттеру:** вы получаете значение из библиотеки или внешнего источника и не можете обернуть его в `startTransition`.

### Пример: виртуализированный список с поиском

```jsx
function VirtualizedList({ items, filterText }) {
  const deferredFilterText = useDeferredValue(filterText);
  const isStale = filterText !== deferredFilterText;

  const filteredItems = useMemo(() => {
    return items.filter(item =>
      item.name.toLowerCase().includes(deferredFilterText.toLowerCase())
    );
  }, [items, deferredFilterText]);

  return (
    <div style={{ opacity: isStale ? 0.6 : 1 }}>
      <WindowScroller>
        {({ height, isScrolling, onChildScroll, scrollTop }) => (
          <List
            height={height}
            itemCount={filteredItems.length}
            itemSize={50}
            onScroll={onChildScroll}
            scrollTop={scrollTop}
          >
            {({ index, style }) => (
              <div style={style}>
                {filteredItems[index].name}
              </div>
            )}
          </List>
        )}
      </WindowScroller>
    </div>
  );
}
```

Здесь `useDeferredValue` позволяет инпуту оставаться отзывчивым, пока виртуализированный список фильтруется в фоне.

---

## Практические паттерны и рекомендации

### Паттерн 1: Индикатор загрузки для transition

```jsx
function DataGrid({ data }) {
  const [sortConfig, setSortConfig] = useState({ key: 'name', direction: 'asc' });
  const [isPending, startTransition] = useTransition();

  const handleSort = (key) => {
    startTransition(() => {
      setSortConfig(prev => ({
        key,
        direction: prev.key === key && prev.direction === 'asc' ? 'desc' : 'asc'
      }));
    });
  };

  const sortedData = useMemo(() => {
    return [...data].sort((a, b) => {
      const modifier = sortConfig.direction === 'asc' ? 1 : -1;
      return a[sortConfig.key] > b[sortConfig.key] ? modifier : -modifier;
    });
  }, [data, sortConfig]);

  return (
    <div>
      <table>
        <thead>
          <tr>
            <th onClick={() => handleSort('name')}>
              Name {sortConfig.key === 'name' && (sortConfig.direction === 'asc' ? '↑' : '↓')}
            </th>
            <th onClick={() => handleSort('age')}>
              Age {sortConfig.key === 'age' && (sortConfig.direction === 'asc' ? '↑' : '↓')}
            </th>
          </tr>
        </thead>
        <tbody style={{ opacity: isPending ? 0.5 : 1 }}>
          {sortedData.map(row => (
            <tr key={row.id}>
              <td>{row.name}</td>
              <td>{row.age}</td>
            </tr>
          ))}
        </tbody>
      </table>
      {isPending && <div className="loading-overlay">Сортировка...</div>}
    </div>
  );
}
```

### Паттерн 2: Комбинация useTransition и Suspense

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value) => {
    setQuery(value);
    startTransition(() => {
      // Обновление состояния, которое триггерит Suspense
      setSearchParams({ q: value });
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
      />
      
      <Suspense fallback={isPending ? <Spinner /> : null}>
        <SearchResults query={query} />
      </Suspense>
    </div>
  );
}
```

### Паттерн 3: Отложенная анимация

```jsx
function AnimatedChart({ data }) {
  const deferredData = useDeferredValue(data);
  const isStale = data !== deferredData;

  return (
    <svg
      style={{
        transition: isStale ? 'none' : 'all 0.3s ease-out',
        opacity: isStale ? 0.7 : 1
      }}
    >
      {/* Сложная визуализация с множеством элементов */}
      {deferredData.map((point, i) => (
        <circle
          key={i}
          cx={point.x}
          cy={point.y}
          r={point.value}
        />
      ))}
    </svg>
  );
}
```

### Паттерн 4: Избежание лишних transition

Не оборачивайте в `startTransition` обновления, которые должны быть срочными:

```jsx
// ❌ Плохо: позиция курсора должна обновляться немедленно
function DraggableElement() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [isPending, startTransition] = useTransition();

  const handleMouseMove = (e) => {
    startTransition(() => {
      setPosition({ x: e.clientX, y: e.clientY });
    });
  };

  return <div onMouseMove={handleMouseMove} style={{ /* ... */ }} />;
}

// ✅ Хорошо: позиция обновляется синхронно
function DraggableElement() {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e) => {
    setPosition({ x: e.clientX, y: e.clientY });
  };

  return <div onMouseMove={handleMouseMove} style={{ /* ... */ }} />;
}
```

---

## Производительность и отладка

### Когда useTransition/useDeferredValue не помогают

- **Синхронные вычисления блокируют основной поток:** если фильтрация занимает 500мс, она всё равно заблокирует поток, просто отложится. Используйте Web Workers для действительно тяжёлых вычислений.
- **Слишком частые обновления:** если `startTransition` вызывается 100 раз в секунду, React всё равно будет обрабатывать много обновлений. Комбинируйте с `debounce`.
- **Медленные компоненты без мемоизации:** если дочерние компоненты не обёрнуты в `React.memo`, они будут перерисовываться при каждом обновлении родителя, даже если пропсы не изменились.

### Инструменты отладки

**React DevTools Profiler** показывает приоритеты обновлений:
1. Откройте React DevTools → Profiler.
2. Запишите взаимодействие.
3. В flamechart обновления с transition-приоритетом помечены отдельно.

**console.log для отслеживания:**

```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  
  console.log('Render:', { query, deferredQuery, stale: query !== deferredQuery });
  
  return <HeavyComponent query={deferredQuery} />;
}
```

---

## Сравнение с Vue

В Vue нет прямого эквивалента конкурентного режима React. Однако схожие паттерны реализуются через:

| React | Vue | Комментарий |
|-------|-----|-------------|
| `useTransition` | `useDebouncedRef` из VueUse | Имитация через debounce, без интеграции в рендерер |
| `useDeferredValue` | `ref` + `watch` с `nextTick` | Ручное отложенное обновление |
| Приоритеты обновлений | Нет | Vue использует синхронный рендеринг |

Vue 3.4+ экспериментирует с асинхронным рендерингом, но модель отличается от React.

---

## Итог

`useTransition` и `useDeferredValue` — мощные инструменты для создания отзывчивых интерфейсов в React 18+. Они позволяют явно управлять приоритетами обновлений, предотвращая блокировку интерфейса при тяжёлых вычислениях.

**Ключевые принципы:**
- Используйте `useTransition`, когда контролируете сеттер состояния.
- Используйте `useDeferredValue`, когда значение приходит извне.
- Помечайте как transition только несрочные обновления.
- Показывайте индикаторы загрузки через `isPending` или сравнение значений.
- Комбинируйте с `React.memo` и `useMemo` для максимальной производительности.

Конкурентный режим — не серебряная пуля. Он не ускоряет вычисления, а лишь перераспределяет их, чтобы интерфейс оставался отзывчивым. Для действительно тяжёлых задач используйте Web Workers, виртуализацию и другие оптимизации.
