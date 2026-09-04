# Flexbox в глубину: оси, `flex-basis`, grow/shrink, alignment

Flexbox — первый по-настоящему осевой layout-механизм в CSS. Он решает задачи, которые раньше требовали float, inline-block и кучу хаков: выравнивание по центру, равномерное распределение места, адаптивные списки. Чтобы не теряться в «магии» flex, нужно понимать не отдельные свойства, а алгоритм распределения пространства.

## Глубокий разбор

### Две оси: main axis и cross axis

Каждый flex-контейнер задаёт две оси.

- **Main axis** (главная ось) — направление, вдоль которого располагаются flex-элементы. Задаётся `flex-direction`.
- **Cross axis** (поперечная ось) — перпендикулярно главной.

```css
.flex-row {
  display: flex;
  flex-direction: row; /* main axis слева направо (по умолчанию) */
}

.flex-column {
  display: flex;
  flex-direction: column; /* main axis сверху вниз */
}
```

`flex-direction` принимает значения `row`, `row-reverse`, `column`, `column-reverse`. Вместе с `writing-mode` они определяют, где физическое начало и конец осей. Поэтому важны логические свойства:

- `justify-content` выравнивает по main axis.
- `align-items` выравнивает по cross axis.

Свойства `justify-*` всегда относятся к main axis, `align-*` — к cross axis. Это справедливо и для Grid.

### Flex-элементы: что происходит с детьми

Прямые дети flex-контейнера становятся flex-элементами. При этом:

- `float` и `clear` не работают.
- `vertical-align` игнорируется.
- Margin’ы не схлопываются.
- Анонимные текстовые узлы оборачиваются в анонимные flex-элементы.

По умолчанию каждый flex-элемент:

```css
flex: 0 1 auto;
```

Это означает: не расти (`flex-grow: 0`), сжиматься при необходимости (`flex-shrink: 1`), базовый размер по контенту (`flex-basis: auto`).

### `flex-basis` vs `width`

**`flex-basis`** — это начальный размер flex-элемента до распределения свободного места. Для `flex-direction: row` `flex-basis` заменяет `width`; для `column` — `height`.

```css
.item {
  flex-basis: 200px;
}
```

Ключевые отличия от `width`:

- `flex-basis` работает только во flex-контексте.
- Если заданы и `flex-basis`, и `width` (для row), приоритет у `flex-basis`, кроме значения `auto` у `flex-basis` — тогда используется `width`.
- `flex-basis: content` задаёт размер по контенту, включая возможность переноса строк.

Важный нюанс: `flex-basis` учитывает `box-sizing`. Если у элемента `box-sizing: border-box`, то `flex-basis: 200px` включает padding и border.

### `flex-grow`: как делится лишнее место

**`flex-grow`** определяет, как flex-элемент забирает свободное пространство вдоль main axis. Значение — не процент, а коэффициент.

```css
.container {
  display: flex;
  width: 600px;
}

.a { flex-grow: 1; }
.b { flex-grow: 2; }
```

Если суммарный базовый размер элементов меньше 600px, оставшееся место делится в пропорции 1:2. Важно: делится не вся ширина контейнера, а только **свободное пространство**.

Если `flex-basis: 0`, элемент как будто не имеет собственного размера, и весь контейнер делится пропорционально `flex-grow`.

```css
.equal {
  flex: 1 1 0; /* равные колонки независимо от контента */
}
```

### `flex-shrink`: как сжимается при нехватке места

**`flex-shrink`** определяет, как элемент уменьшается, когда сумма базовых размеров превышает размер контейнера.

```css
.container {
  display: flex;
  width: 400px;
}

.a { flex-basis: 300px; flex-shrink: 1; }
.b { flex-basis: 300px; flex-shrink: 2; }
```

Избыток: 300 + 300 − 400 = 200px. Он распределяется обратно пропорционально `flex-shrink` с учётом базового размера. Элемент `b` сожмётся сильнее, чем `a`, но не ровно вдвое — формула учитывает `flex-basis * flex-shrink`.

Если нужно запретить сжатие:

```css
.item {
  flex-shrink: 0;
}
```

### Shorthand `flex`

Свойство `flex` объединяет `flex-grow`, `flex-shrink` и `flex-basis`. Возможны три ключевых значения:

```css
flex: initial;   /* 0 1 auto — по умолчанию */
flex: auto;      /* 1 1 auto — расти и сжиматься от размера контента */
flex: none;      /* 0 0 auto — фиксированный размер, не сжиматься */
flex: 1;         /* 1 1 0% — занять всё доступное место */
```

Частая ошибка: `flex: 1` устанавливает `flex-basis: 0%`, а не `auto`. Это важно, когда контент элементов сильно различается: с `flex: 1` колонки будут равными, с `flex: auto` — пропорциональными контенту.

### Выравнивание

#### `justify-content` — по main axis

```css
.container {
  justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly;
}
```

- `space-between` — первый и последний элементы у краёв, остальные равномерно.
- `space-around` — равные промежутки с половинками по краям.
- `space-evenly` — все промежутки, включая краевые, равны.

#### `align-items` — по cross axis для всех элементов в строке

```css
.container {
  align-items: stretch | flex-start | flex-end | center | baseline;
}
```

По умолчанию `stretch`: элементы растягиваются на всю высоту строки.

#### `align-self` — переопределение для одного элемента

```css
.item {
  align-self: flex-start;
}
```

#### `align-content` — распределение строк при переносе

Работает только тогда, когда `flex-wrap: wrap` и есть несколько строк. Управляет промежутками между строками по cross axis.

```css
.container {
  flex-wrap: wrap;
  align-content: flex-start | flex-end | center | space-between | space-around | space-evenly | stretch;
}
```

Без `align-content` строки растягиваются и распределяются по умолчанию. Если высота контейнера больше суммы высот строк, `align-content` решает, как заполнить пространство.

### `flex-wrap` и многострочность

По умолчанию `flex-wrap: nowrap` — все элементы в одну строку. Для переноса:

```css
.container {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}
```

При переносе каждая строка становится отдельной линией для выравнивания. `align-items` работает внутри строки, `align-content` — между строками.

### `gap` и `order`

`gap` задаёт расстояние между элементами по main и cross axis:

```css
.container {
  gap: 16px 24px; /* row-gap column-gap */
}
```

`order` меняет визуальный порядок элементов. По умолчанию 0; элементы с меньшим `order` идут раньше.

```css
.first { order: -1; }
```

Важно: `order` меняет только визуальный порядок, не DOM-порядок. Это критично для accessibility: скринридеры и фокус по-прежнему следуют исходному порядку.

### Минимальные размеры и переполнение

По умолчанию flex-элементы не могут быть меньше минимального размера контента (`min-width: auto`). Это часто приводит к тому, что элементы не сжимаются до нуля, даже при `flex-shrink: 1`.

```css
.item {
  min-width: 0; /* разрешить сжатие до нуля */
  overflow: hidden;
}
```

Это особенно важно для текстовых блоков внутри flex-контейнера: без `min-width: 0` текст не будет обрезаться `text-overflow: ellipsis`.

## Практические примеры

### Пример 1: равные колонки

```html
<div class="row">
  <div class="col">A</div>
  <div class="col">B</div>
  <div class="col">C</div>
</div>
```

```css
.row {
  display: flex;
  gap: 16px;
}

.col {
  flex: 1 1 0;
}
```

Все три колонки одинаковой ширины независимо от контента, потому что `flex-basis: 0` и `flex-grow` равны.

### Пример 2: фиксированный сайдбар и гибкий контент

```html
<div class="layout">
  <aside class="sidebar">Sidebar</aside>
  <main class="content">Main content</main>
</div>
```

```css
.layout {
  display: flex;
  gap: 24px;
}

.sidebar {
  flex: 0 0 240px;
}

.content {
  flex: 1 1 auto;
  min-width: 0;
}
```

Сайдбар фиксированной ширины, основной контент занимает оставшееся место. `min-width: 0` позволяет контенту сжиматься.

### Пример 3: центрирование по обеим осям

```css
.center {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}
```

Классический способ центрирования без absolute.

### Пример 4: карточки с переносом

```html
<div class="cards">
  <div class="card">1</div>
  <div class="card">2</div>
  <div class="card">3</div>
  <div class="card">4</div>
</div>
```

```css
.cards {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}

.card {
  flex: 1 1 280px;
}
```

Карточки стремятся занять не менее 280px. Если места меньше, переносятся на новую строку. Свободное место в строке распределяется пропорционально `flex-grow`.

### Пример 5: `text-overflow` внутри flex-элемента

```html
<div class="flex">
  <span class="truncate">Очень длинный текст, который должен обрезаться</span>
  <button>Action</button>
</div>
```

```css
.flex {
  display: flex;
  gap: 8px;
}

.truncate {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

Без `min-width: 0` flex-элемент с текстом не сожмётся, и `text-overflow` не сработает.

## Типичные ошибки и антипаттерны

- **Путаница `flex-basis` и `width`.** `flex-basis` определяет начальный размер до распределения пространства, а `width` — фактический размер только если `flex-basis: auto`. В flex-контексте предпочтительнее управлять размерами через `flex-basis`.
- **Использование `flex: 1` для «одинаковых колонок» без понимания `flex-basis: 0%`.** Это работает, но если нужна пропорциональность контенту, используйте `flex: auto`.
- **Ожидание, что `align-content` работает без `flex-wrap`.** `align-content` влияет только на распределение строк, поэтому без многострочности игнорируется.
- **Забытое `min-width: 0` при переполнении текста.** По умолчанию flex-элемент не может быть уже контента, и `text-overflow` не сработает.
- **Использование `order` для изменения логического порядка.** Визуальный порядок не должен расходиться с порядком в DOM — это ломает accessibility и навигацию с клавиатуры.
- **`justify-content: space-evenly` в старых Safari.** Поддержка есть, но в очень старых версиях может требоваться фолбэк.
- **Вертикальное выравнивание через `margin-top: auto` без понимания.** `margin: auto` в flex-контексте поглощает свободное пространство и может использоваться для прижатия элементов, но это не всегда очевидно для коллег.

## Ключевые тезисы для интервью

- Flexbox работает с двумя осями: main axis (`justify-*`) и cross axis (`align-*`).
- `flex-basis` — начальный размер элемента до распределения места; для row заменяет `width`, для column — `height`.
- `flex-grow` делит **свободное** пространство пропорционально коэффициентам, а не всю ширину контейнера.
- `flex-shrink` управляет сжатием при переполнении; формула учитывает `flex-basis * flex-shrink`.
- `flex: 1` = `1 1 0%`; `flex: auto` = `1 1 auto`; `flex: initial` = `0 1 auto`.
- `align-items` выравнивает элементы внутри строки по cross axis; `align-content` — распределяет сами строки при `flex-wrap: wrap`.
- `order` меняет только визуальный порядок, не DOM-порядок; это может нарушить accessibility.
- По умолчанию flex-элементы имеют `min-width: auto`, поэтому для обрезки текста нужно явно задавать `min-width: 0`.

## Полезные ссылки

- [CSS Flexible Box Layout](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_flexible_box_layout)
- [flex-basis](https://developer.mozilla.org/en-US/docs/Web/CSS/flex-basis)
- [flex-grow](https://developer.mozilla.org/en-US/docs/Web/CSS/flex-grow)
- [flex-shrink](https://developer.mozilla.org/en-US/docs/Web/CSS/flex-shrink)
- [Aligning items in a flex container](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_flexible_box_layout/Aligning_items_in_a_flex_container)
- [A Complete Guide to Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)
