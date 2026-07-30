# Виртуализация списков — Глубокое погружение

## Содержание

1. [Что такое виртуализация списков](#что-такое-виртуализация-списков)
2. [Какую проблему решает виртуализация](#какую-проблему-решает-виртуализация)
3. [Как это работает](#как-это-работает)
4. [react-window](#react-window)
5. [react-virtuoso](#react-virtuoso)
6. [vue-virtual-scroller](#vue-virtual-scroller)
7. [vue-virtual-scroll-list](#vue-virtual-scroll-list)
8. [@tanstack/vue-virtual](#tanstackvue-virtual)
9. [Сравнение библиотек](#сравнение-библиотек)
10. [Когда что использовать](#когда-что-использовать)
11. [Лучшие практики](#лучшие-практики)
12. [Антипаттерны](#антипаттерны)

---

## Что такое виртуализация списков

Виртуализация списков — это техника рендеринга, при которой отображаются **только видимые элементы** списка (плюс небольшой буфер сверху и снизу). Остальные элементы существуют в памяти как данные, но **не создают DOM-узлов**.

```jsx
// Без виртуализации — 10 000 DOM-элементов
{items.map((item) => <Row key={item.id} data={item} />)}

// С виртуализацией — ~50 DOM-элементов
<VirtualList data={items} renderItem={(item) => <Row data={item} />} />
```

Это критично для производительности: браузер тратит ресурсы на каждый DOM-узел — layout, paint, memory. При тысячах элементов интерфейс начинает тормозить.

---

## Какую проблему решает виртуализация

### Проблема

Рендер большого списка (1000+ элементов) приводит к:

- **Долгий initial render** — браузеру нужно создать все DOM-узлы
- **Тяжёлый layout** — пересчёт позиций тысяч элементов
- **Высокое потребление памяти** — каждый DOM-узел занимает RAM
- **Медленный скролл** — браузер вынужден перерисовывать все элементы
- **Блокировка main thread** — интерфейс «замирает»

### Решение

Виртуализация рендерит только то, что пользователь **видит в данный момент**:

| Метрика | Без виртуализации | С виртуализацией |
|---------|-------------------|------------------|
| DOM-узлов (10 000 элементов) | 10 000 | ~50-100 |
| Время первого рендера | 2-5 сек | < 100 мс |
| Потребление памяти | Высокое | Минимальное |
| Плавность скролла | Лаги | 60 FPS |

---

## Как это работает

### Принцип

Все данные загружены в память целиком. В DOM рендерятся только видимые элементы + буфер. При скролле DOM-ноды **переиспользуются** — меняется их содержимое и позиция.

```
┌─────────────────────────────────┐
│  padding-top: 2400px            │ ← имитация элементов выше viewport
├─────────────────────────────────┤
│  Row 48  ← DOM-нода (переиспользуется)  │
│  Row 49  ← DOM-нода             │  ← видимая область
│  Row 50  ← DOM-нода             │     (~20 элементов)
│  ...                            │
│  Row 72  ← DOM-нода             │
├─────────────────────────────────┤
│  padding-bottom: 47500px        │ ← имитация элементов ниже viewport
└─────────────────────────────────┘
```

### Ключевые механизмы

1. **Контейнер с overflow: auto** — создаёт скроллбар
2. **padding-top / padding-bottom** — имитируют полную высоту списка, чтобы скроллбар вёл себя корректно
3. **Пересчёт при скролле** — на каждый кадр скролла библиотека определяет, какие элементы должны быть видимы, и обновляет DOM
4. **Переиспользование нод** — DOM-элементы не создаются заново, а перемещаются и наполняются новыми данными

### Фиксированная vs динамическая высота

**Фиксированная высота** — все элементы имеют одинаковый размер. Библиотека может точно вычислить, какие элементы видны, по формуле: `startIndex = scrollTop / itemHeight`. Быстрее и проще.

**Динамическая высота** — элементы разного размера. Библиотека измеряет каждый элемент после рендера и кэширует размеры. Сложнее, но гибче.

---

## react-window

Минималистичная библиотека от **Brian Vaughn** (команда React). Фокус на размере бандла и производительности.

### Установка

```bash
npm install react-window
```

### FixedSizeList — элементы одинакового размера

```jsx
import { FixedSizeList } from "react-window";

function UserList({ users }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={users.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style} className="user-row">
          {users[index].name}
        </div>
      )}
    </FixedSizeList>
  );
}
```

### VariableSizeList — элементы разного размера

```jsx
import { VariableSizeList } from "react-window";

function DynamicList({ items }) {
  const getItemSize = (index) => {
    return items[index].height;
  };

  return (
    <VariableSizeList
      height={600}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index].content}</div>
      )}
    </VariableSizeList>
  );
}
```

### FixedSizeGrid — двумерная сетка

```jsx
import { FixedSizeGrid } from "react-window";

function PhotoGrid({ photos }) {
  const columnCount = 4;
  const rowCount = Math.ceil(photos.length / columnCount);

  return (
    <FixedSizeGrid
      height={600}
      width="100%"
      columnCount={columnCount}
      columnWidth={150}
      rowCount={rowCount}
      rowHeight={150}
    >
      {({ columnIndex, rowIndex, style }) => {
        const photoIndex = rowIndex * columnCount + columnIndex;
        const photo = photos[photoIndex];
        if (!photo) return null;

        return (
          <div style={style}>
            <img src={photo.url} alt={photo.title} />
          </div>
        );
      }}
    </FixedSizeGrid>
  );
}
```

### Горизонтальный скролл

```jsx
<FixedSizeList
  height={100}
  itemCount={items.length}
  itemSize={150}
  width="100%"
  layout="horizontal"
>
  {({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  )}
</FixedSizeList>
```

### API

| Проп | Тип | Описание |
|------|-----|----------|
| `height` | `number` | Высота контейнера (обязательно) |
| `width` | `number \| string` | Ширина контейнера (обязательно) |
| `itemCount` | `number` | Общее количество элементов (обязательно) |
| `itemSize` | `number \| (index) => number` | Размер элемента (обязательно) |
| `overscanCount` | `number` | Количество элементов за viewport (по умолчанию 1) |
| `initialScrollOffset` | `number` | Начальная позиция скролла |
| `layout` | `"vertical" \| "horizontal"` | Направление скролла |

---

## react-virtuoso

Более функциональная библиотека «из коробки». Автоматически измеряет размеры элементов, поддерживает sticky headers, infinite scroll и автоскролл.

### Установка

```bash
npm install react-virtuoso
```

### Базовое использование

```jsx
import { Virtuoso } from "react-virtuoso";

function UserList({ users }) {
  return (
    <Virtuoso
      style={{ height: 600 }}
      data={users}
      itemContent={(index, user) => (
        <div className="user-row">{user.name}</div>
      )}
    />
  );
}
```

### Динамическая высота (автоматически)

```jsx
// react-virtuoso сам измеряет высоту каждого элемента
<Virtuoso
  style={{ height: 600 }}
  data={items}
  itemContent={(index, item) => (
    <div className="card">
      <h2>{item.title}</h2>
      <p>{item.description}</p>
    </div>
  )}
/>
```

### Группы со sticky headers

```jsx
import { Virtuoso, GroupedVirtuoso } from "react-virtuoso";

function GroupedList({ items, groupIndices, groupLabels }) {
  return (
    <GroupedVirtuoso
      style={{ height: 600 }}
      groupCounts={groupIndices}
      groupContent={(index) => (
        <div className="sticky-header">{groupLabels[index]}</div>
      )}
      itemContent={(index) => (
        <div className="item">{items[index].name}</div>
      )}
    />
  );
}
```

### Infinite scroll (подгрузка при скролле)

```jsx
import { Virtuoso } from "react-virtuoso";
import { useState, useCallback } from "react";

function InfiniteList() {
  const [items, setItems] = useState(() => generateItems(0, 20));

  const loadMore = useCallback(() => {
    const newItems = generateItems(items.length, items.length + 20);
    setItems((prev) => [...prev, ...newItems]);
  }, [items.length]);

  return (
    <Virtuoso
      style={{ height: 600 }}
      data={items}
      endReached={loadMore}
      itemContent={(index, item) => <div>{item.name}</div>}
    />
  );
}
```

### Автоскролл к последнему элементу (чат)

```jsx
import { Virtuoso } from "react-virtuoso";
import { useState, useEffect } from "react";

function Chat({ messages }) {
  return (
    <Virtuoso
      style={{ height: 600 }}
      data={messages}
      followOutput="smooth"
      itemContent={(index, message) => (
        <div className="message">{message.text}</div>
      )}
    />
  );
}
```

### TableVirtuoso — виртуализация таблиц

```jsx
import { TableVirtuoso } from "react-virtuoso";

function DataTable({ rows }) {
  return (
    <TableVirtuoso
      style={{ height: 600 }}
      data={rows}
      columns={["id", "name", "email", "role"]}
      itemContent={(index, row) => (
        <>
          <td>{row.id}</td>
          <td>{row.name}</td>
          <td>{row.email}</td>
          <td>{row.role}</td>
        </>
      )}
    />
  );
}
```

### API

| Проп | Тип | Описание |
|------|-----|----------|
| `data` | `any[]` | Массив данных |
| `itemContent` | `(index, data) => ReactNode` | Рендер элемента |
| `style` | `CSSProperties` | Стили контейнера |
| `totalListHeightChanged` | `(height) => void` | Callback при изменении высоты |
| `endReached` | `(index) => void` | Callback при достижении конца списка |
| `startReached` | `(index) => void` | Callback при достижении начала списка |
| `followOutput` | `boolean \| "smooth" \| "auto"` | Автоскролл к новому элементу |
| `initialTopMostItemIndex` | `number` | Начальный видимый элемент |

---

## vue-virtual-scroller

Самая популярная библиотека виртуализации для Vue. Поддерживает Vue 2 и Vue 3.

### Установка

```bash
npm install vue-virtual-scroller
```

### RecycleScroller — фиксированный размер

```vue
<template>
  <RecycleScroller
    class="scroller"
    :items="users"
    :item-size="42"
    key-field="id"
    v-slot="{ item }"
  >
    <div class="user-row">{{ item.name }}</div>
  </RecycleScroller>
</template>

<script setup>
import { RecycleScroller } from "vue-virtual-scroller";
import "vue-virtual-scroller/dist/vue-virtual-scroller.css";
</script>

<style>
.scroller {
  height: 600px;
}
</style>
```

### DynamicScroller — динамический размер

```vue
<template>
  <DynamicScroller
    class="scroller"
    :items="items"
    :min-item-size="42"
  >
    <template #default="{ item, index, active }">
      <DynamicScrollerItem
        :item="item"
        :active="active"
        :data-index="index"
      >
        <div class="card">
          <h3>{{ item.title }}</h3>
          <p>{{ item.description }}</p>
        </div>
      </DynamicScrollerItem>
    </template>
  </DynamicScroller>
</template>

<script setup>
import { DynamicScroller, DynamicScrollerItem } from "vue-virtual-scroller";
import "vue-virtual-scroller/dist/vue-virtual-scroller.css";
</script>
```

### Группы со sticky headers

```vue
<template>
  <RecycleScroller
    class="scroller"
    :items="groupedItems"
    :item-size="42"
    key-field="id"
  >
    <template #before>
      <div class="sticky-header">Group A</div>
    </template>
    <template #default="{ item }">
      <div>{{ item.name }}</div>
    </template>
  </RecycleScroller>
</template>
```

### API

| Проп | Тип | Описание |
|------|-----|----------|
| `items` | `Array` | Массив данных (обязательно) |
| `item-size` | `number \| null` | Размер элемента (null для динамического) |
| `key-field` | `string` | Поле уникального ключа (по умолчанию `"id"`) |
| `direction` | `"vertical" \| "horizontal"` | Направление скролла |
| `buffer` | `number` | Буфер за viewport в px (по умолчанию 200) |

---

## vue-virtual-scroll-list

Альтернативная библиотека с акцентом на производительность и гибкость.

### Установка

```bash
npm install vue-virtual-scroll-list
```

### Базовое использование

```vue
<template>
  <VirtualList
    ref="vslRef"
    :data-key="'id'"
    :data-sources="items"
    :data-component="ItemComponent"
    :estimate-size="50"
    :keeps="30"
  />
</template>

<script setup>
import VirtualList from "vue-virtual-scroll-list";
import { h, defineComponent } from "vue";

const ItemComponent = defineComponent({
  props: {
    source: { type: Object, required: true },
  },
  setup(props) {
    return () => h("div", { class: "item" }, props.source.name);
  },
});
</script>
```

### Виртуализация таблицы

```vue
<template>
  <VirtualList
    :data-key="'id'"
    :data-sources="rows"
    :data-component="RowComponent"
    :estimate-size="40"
  />
</template>

<script setup>
import VirtualList from "vue-virtual-scroll-list";
import { h, defineComponent } from "vue";

const RowComponent = defineComponent({
  props: {
    source: { type: Object, required: true },
  },
  setup(props) {
    return () =>
      h("div", { class: "table-row" }, [
        h("span", props.source.id),
        h("span", props.source.name),
        h("span", props.source.email),
      ]);
  },
});
</script>
```

### API

| Проп | Тип | Описание |
|------|-----|----------|
| `data-key` | `string` | Поле уникального ключа (обязательно) |
| `data-sources` | `Array` | Массив данных (обязательно) |
| `data-component` | `Component` | Компонент для рендера элемента (обязательно) |
| `estimate-size` | `number` | Ожидаемый размер элемента |
| `keeps` | `number` | Количество элементов в DOM (по умолчанию 30) |
| `direction` | `"vertical" \| "horizontal"` | Направление скролла |

---

## @tanstack/vue-virtual

Headless-решение от TanStack. Даёт только логику виртуализации, разметку пишешь сам.

### Установка

```bash
npm install @tanstack/vue-virtual
```

### Базовое использование

```vue
<template>
  <div ref="scrollRef" class="scroll-container">
    <div :style="{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }">
      <div
        v-for="row in virtualItems"
        :key="row.key"
        :style="{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: `${row.size}px`,
          transform: `translateY(${row.start}px)`,
        }"
      >
        {{ items[row.index].name }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import { useVirtualizer } from "@tanstack/vue-virtual";

const props = defineProps({
  items: { type: Array, required: true },
});

const scrollRef = ref(null);

const virtualizer = useVirtualizer({
  count: () => props.items.length,
  getScrollElement: () => scrollRef.value,
  estimateSize: () => 35,
  overscan: 5,
});

const virtualItems = computed(() => virtualizer.value.getVirtualItems());
</script>

<style>
.scroll-container {
  height: 600px;
  overflow: auto;
}
</style>
```

### Динамическая высота с measureElement

```vue
<template>
  <div ref="scrollRef" class="scroll-container">
    <div :style="{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }">
      <div
        v-for="row in virtualItems"
        :key="row.key"
        :ref="(el) => virtualizer.value.measureElement(el)"
        :data-index="row.index"
        :style="{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          transform: `translateY(${row.start}px)`,
        }"
      >
        <div class="dynamic-content">{{ items[row.index].content }}</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import { useVirtualizer } from "@tanstack/vue-virtual";

const props = defineProps({
  items: { type: Array, required: true },
});

const scrollRef = ref(null);

const virtualizer = useVirtualizer({
  count: () => props.items.length,
  getScrollElement: () => scrollRef.value,
  estimateSize: () => 50,
  measureElement: (el) => el.getBoundingClientRect().height,
});

const virtualItems = computed(() => virtualizer.value.getVirtualItems());
</script>
```

### API

| Опция | Тип | Описание |
|-------|-----|----------|
| `count` | `() => number` | Количество элементов (обязательно) |
| `getScrollElement` | `() => HTMLElement \| null` | Функция возврата scroll-контейнера (обязательно) |
| `estimateSize` | `(index) => number` | Ожидаемый размер элемента (обязательно) |
| `overscan` | `number` | Количество элементов за viewport (по умолчанию 1) |
| `measureElement` | `(el) => number` | Функция измерения реального размера |
| `scrollPaddingStart` | `number` | Отступ в начале в px |
| `scrollPaddingEnd` | `number` | Отступ в конце в px |

---

## Сравнение библиотек

### React

| Фича | react-window | react-virtuoso |
|------|-------------|----------------|
| Размер бандла | ~6 KB | ~30 KB |
| Фиксированная высота | ✅ | ✅ |
| Динамическая высота | ✅ (VariableSizeList) | ✅ (автоматически) |
| Grid (двумерная сетка) | ✅ | ❌ |
| Sticky headers | ❌ | ✅ |
| Infinite scroll | ❌ (вручную) | ✅ (endReached) |
| Автоскролл (чат) | ❌ (вручную) | ✅ (followOutput) |
| TableVirtuoso | ❌ | ✅ |
| Горизонтальный скролл | ✅ | ✅ |
| Простота API | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### Vue

| Фича | vue-virtual-scroller | vue-virtual-scroll-list | @tanstack/vue-virtual |
|------|---------------------|------------------------|----------------------|
| Размер бандла | ~15 KB | ~10 KB | ~8 KB |
| Фиксированная высота | ✅ | ✅ | ✅ |
| Динамическая высота | ✅ (DynamicScroller) | ✅ | ✅ (measureElement) |
| Группы / sticky headers | ✅ | ❌ | ❌ (вручную) |
| Горизонтальный скролл | ✅ | ✅ | ✅ |
| Headless | ❌ | ❌ | ✅ |
| Простота API | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

---

## Когда что использовать

### React

| Сценарий | Выбор |
|----------|-------|
| Фиксированная высота строк, минимальный бандл | **react-window** |
| Динамическая высота, чат с автоскроллом | **react-virtuoso** |
| Sticky group headers | **react-virtuoso** |
| Infinite scroll из коробки | **react-virtuoso** |
| Виртуализация таблиц | **react-virtuoso** (TableVirtuoso) |
| Двумерная сетка (фотогалерея) | **react-window** (FixedSizeGrid) |
| Критичен размер бандла | **react-window** |

### Vue

| Сценарий | Выбор |
|----------|-------|
| Быстрый старт, минимум кода | **vue-virtual-scroller** |
| Полный контроль над DOM | **@tanstack/vue-virtual** |
| Сложные кейсы, таблицы | **vue-virtual-scroll-list** |
| Группы и sticky headers | **vue-virtual-scroller** |
| Уже используешь TanStack | **@tanstack/vue-virtual** |

---

## Лучшие практики

### 1. Всегда указывай стабильные ключи

```jsx
// Плохо — index как ключ при фильтрации/сортировке
<Virtuoso
  data={items}
  itemContent={(index) => <Item key={index} data={items[index]} />}
/>

// Хорошо — уникальный ID
<Virtuoso
  data={items}
  itemContent={(index, item) => <Item key={item.id} data={item} />}
/>
```

### 2. Мемоизируй renderItem / itemContent

```jsx
const renderItem = useCallback(
  (index, item) => <Item data={item} />,
  []
);

<Virtuoso data={items} itemContent={renderItem} />
```

### 3. Используй overscan с умом

```jsx
// Слишком маленький — видно «подгрузку» при быстром скролле
<FixedSizeList overscanCount={0} ... />

// Оптимально — 5-10 элементов буфера
<FixedSizeList overscanCount={5} ... />

// Слишком большой — лишний рендер, трата памяти
<FixedSizeList overscanCount={50} ... />
```

### 4. Фиксированная высота быстрее

Если все элементы одинакового размера — используй `FixedSizeList` вместо `VariableSizeList`. Это быстрее, потому что не нужно измерять каждый элемент.

### 5. Ленивая загрузка данных

```jsx
const loadMore = useCallback(() => {
  if (page < totalPages) {
    fetchPage(page + 1).then((newItems) => {
      setItems((prev) => [...prev, ...newItems]);
      setPage((p) => p + 1);
    });
  }
}, [page, totalPages]);

<Virtuoso endReached={loadMore} data={items} ... />
```

### 6. Избегай тяжёлых вычислений в itemContent

```jsx
// Плохо — вычисление на каждый рендер
<Virtuoso
  itemContent={(index, item) => {
    const expensive = computeExpensiveValue(item);
    return <Item data={expensive} />;
  }}
/>

// Хорошо — мемоизация
const processedItems = useMemo(
  () => items.map((item) => ({ ...item, computed: computeExpensiveValue(item) })),
  [items]
);

<Virtuoso data={processedItems} itemContent={(i, item) => <Item data={item} />} />
```

---

## Антипаттерны

### 1. Виртуализация маленького списка

```jsx
// Плохо — виртуализация для 50 элементов избыточна
<Virtuoso data={items.slice(0, 50)} ... />

// Хорошо — обычный рендер
{items.map((item) => <Item key={item.id} data={item} />)}
```

**Правило**: виртуализация нужна при 500+ элементах. Для меньших списков overhead библиотеки превышает выгоду.

### 2. Нестабильные ключи при динамических данных

```jsx
// Плохо — при фильтрации индексы меняются, DOM пересоздаётся
{filteredItems.map((item, index) => <Item key={index} />)}

// Хорошо — стабильные ID
{filteredItems.map((item) => <Item key={item.id} />)}
```

### 3. Вложенные скроллящиеся контейнеры

```jsx
// Плохо — виртуализация внутри виртуализации
<Virtuoso
  data={groups}
  itemContent={(index, group) => (
    <div>
      <h2>{group.title}</h2>
      <Virtuoso data={group.items} ... />
    </div>
  )}
/>

// Хорошо — плоский список с группами
<GroupedVirtuoso
  groupCounts={groupCounts}
  groupContent={(index) => <h2>{groups[index].title}</h2>}
  itemContent={(index) => <Item data={allItems[index]} />}
/>
```

### 4. Игнорирование resize observer

Если контейнер может менять размер (responsive layout), убедись, что библиотека обрабатывает resize. `react-virtuoso` делает это автоматически, для `react-window` нужен `AutoSizer`:

```jsx
import { FixedSizeList, AutoSizer } from "react-window";

<AutoSizer>
  {({ height, width }) => (
    <FixedSizeList height={height} width={width} ...>
      {({ index, style }) => <div style={style}>...</div>}
    </FixedSizeList>
  )}
</AutoSizer>
```

### 5. Отсутствие fallback для пустого списка

```jsx
// Плохо — пустой контейнер без сообщения
<Virtuoso data={[]} itemContent={() => <Item />} />

// Хорошо — обработка пустого состояния
{items.length === 0 ? (
  <EmptyState />
) : (
  <Virtuoso data={items} itemContent={(i, item) => <Item data={item} />} />
)}
```
