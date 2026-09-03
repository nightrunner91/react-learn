# BFC, IFC, FFC, GFC: formatting contexts и containing block

CSS рисует элементы не по одному, а группами, объединёнными общими правилами раскладки. Эти группы называются **formatting contexts** (контекстами форматирования). Понимание того, какой контекст создаёт элемент, объясняет многие «магические» поведения: схлопывание margin’ов, обтекание float, выравнивание inline-элементов и разницу между `width` и размером flex/grid-элемента.

## Глубокий разбор

### Что такое formatting context

**Formatting context** — это область документа, внутри которой блоки раскладываются по единому набору правил. Все элементы внутри одного контекста влияют друг на друга: например, вертикальные margin’ы блоков в нормальном потоке схлопываются, а inline-элементы распределяются по строкам.

Существует четыре основных типа:

- **BFC** — Block Formatting Context.
- **IFC** — Inline Formatting Context.
- **FFC** — Flex Formatting Context.
- **GFC** — Grid Formatting Context.

### Block Formatting Context (BFC)

**BFC** (блочный контекст форматирования) — это область, в которой блочные элементы располагаются вертикально друг под другом, а margin’ы между ними схлопываются.

Элемент создаёт новый BFC, если у него:

- `float` не `none`;
- `position` равно `absolute` или `fixed`;
- `display: inline-block`, `table-cell`, `table-caption`, `flow-root`;
- `overflow` не `visible`;
- `display: flex` или `grid` у самого элемента (для его детей создаётся FFC/GFC, но сам flex/grid-контейнер тоже изолирован).

Самый современный и чистый способ создать BFC — `display: flow-root`:

```css
.bfc {
  display: flow-root;
}
```

`flow-root` создаёт BFC без побочных эффектов вроде скролла или изменения inline-поведения.

Что даёт BFC:

- **Изоляция float.** Элементы внутри BFC не выходят за его границы, и сам BFC не обтекает float-соседей, если он блочный.
- **Предотвращение схлопывания margin’ов** между родителем и первым/последним потомком.
- **Остановка обтекания** float-элементов соседними блоками.

```css
.clearfix-modern {
  display: flow-root;
}
```

### Inline Formatting Context (IFC)

**IFC** (строчный контекст форматирования) возникает внутри блочного контейнера, когда в нём находятся inline-элементы или текст. Элементы располагаются в строках, переносятся по `white-space` и `word-break`, и выравниваются по базовой линии.

Важные особенности IFC:

- Высота строки определяется `line-height`, а не суммой высот inline-элементов.
- `vertical-align` влияет на положение inline-элемента относительно строки.
- Блочные элементы внутри IFC прерывают его и создают анонимные блочные боксы.

```css
.tagline {
  font-size: 1.25rem;
  line-height: 1.5;
}

.tagline strong {
  vertical-align: middle;
}
```

Проблемы с IFC часто возникают, когда inline-элементы с разными `font-size` или `vertical-align` создают «лишнее» пространство под строкой. Это одна из причин, почему изображения внутри ссылок иногда имеют небольшой отступ снизу.

### Flex Formatting Context (FFC)

**FFC** (flex-контекст форматирования) создаётся элементом с `display: flex` или `display: inline-flex`. Все прямые дети становятся flex-элементами и раскладываются по главной и поперечной осям.

Особенности FFC:

- Flex-элементы по умолчанию стремятся уместиться в одну линию (`flex-wrap: nowrap`).
- `margin` в flex-контексте не схлопывается.
- `float` и `clear` у flex-элементов не работают.
- Размер flex-элемента определяется не только `width`/`height`, но и `flex-basis`, `flex-grow`, `flex-shrink`.

### Grid Formatting Context (GFC)

**GFC** (grid-контекст форматирования) создаётся элементом с `display: grid` или `display: inline-grid`. Дети располагаются в ячейках сетки.

Особенности GFC:

- Grid-элементы не обтекают float.
- `margin` не схлопывается.
- `z-index` работает у grid-элементов даже без `position`.
- Размеры элементов управляются треками сетки, а не их собственными `width`/`height`.

### Containing block

**Containing block** (содержащий блок) — это прямоугольная область, относительно которой вычисляются размеры и позиция элемента. Для элементов в нормальном потоке containing block — это content-box ближайшего блочного предка. Но есть исключения:

- Для элемента с `position: fixed` containing block — viewport.
- Для элемента с `position: absolute` containing block — ближайший позиционированный предок (не `static`).
- Для элемента с `position: absolute`, у которого предок имеет `transform`, `filter`, `perspective` или `contain: paint/layout`, containing block может стать этот предок, даже если у него `position: static`.

```css
.modal {
  position: fixed;
  inset: 0;
  margin: auto;
  width: 400px;
  height: 200px;
}
```

Здесь `inset: 0` растягивает элемент до границ viewport, а `margin: auto` центрирует его по размерам `width`/`height`. containing block — viewport.

### Margin collapse

**Margin collapse** (схлопывание margin’ов) — это одно из самых неочевидных поведений BFC. Вертикальные margin’ы соседних блочных элементов в одном BFC объединяются, и остаётся только больший из них.

Схлопываются:

- соседние блочные элементы;
- margin родителя и первого/последнего потомка, если между ними нет padding, border или BFC;
- пустые блочные элементы, если у них нет padding, border, height и min-height.

Не схлопываются:

- горизонтальные margin’ы;
- margin’ы элементов в разных BFC;
- margin’ы flex/grid-элементов;
- margin’ы элементов с `position: absolute`/`fixed`;
- margin’ы, у которых хотя бы один равен `auto`.

```css
/* Без схлопывания благодаря padding */
.card {
  padding-top: 1px;
}

.card h2 {
  margin-top: 24px;
}
```

### `box-sizing`

`box-sizing` определяет, что входит в `width` и `height`:

- `content-box` — `width`/`height` задают размер только content-area; padding и border прибавляются снаружи.
- `border-box` — `width`/`height` включают content, padding и border.

```css
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

`border-box` упрощает рассуждения о размерах, особенно в раскладках, где ширина задаётся в процентах. В flex/grid-контексте `border-box` также влияет на то, как браузер считает минимальный и максимальный размер.

## Практические примеры

### Пример 1: BFC предотвращает обтекание float

```html
<div class="media">
  <img class="avatar" src="avatar.png" alt="">
  <div class="content">
    <h3>Title</h3>
    <p>Description</p>
  </div>
</div>
```

```css
.avatar {
  float: left;
  width: 64px;
  height: 64px;
  margin-right: 16px;
}

.content {
  display: flow-root; /* создаёт BFC */
}
```

`.content` образует BFC и перестаёт обтекать float-аватарку. Текст внутри не залезет под изображение.

### Пример 2: схлопывание margin’ов и его предотвращение

```html
<article>
  <h2>Heading</h2>
  <p>Paragraph</p>
</article>
```

```css
article {
  background: #f3f4f6;
}

h2 {
  margin-top: 32px;
}
```

Без padding или border у `article` margin-top `h2` «выпадет» за пределы article, и визуально отступ появится сверху article, а не между article и h2. Решения:

```css
article {
  background: #f3f4f6;
  padding-top: 1px; /* или border-top, или display: flow-root */
}
```

### Пример 3: containing block для `position: absolute`

```html
<div class="card">
  <span class="badge">New</span>
</div>
```

```css
.card {
  position: relative;
  padding: 16px;
}

.badge {
  position: absolute;
  top: 8px;
  right: 8px;
}
```

`.badge` позиционируется относительно `.card`, потому что `.card` — ближайший позиционированный предок. Если убрать `position: relative` у `.card`, containing block станет `<html>`, и бейдж уедет в угол страницы.

### Пример 4: grid и `z-index` без позиционирования

```css
.grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
}

.grid > .overlap {
  grid-column: 1 / 3;
  grid-row: 1;
  z-index: 2;
}
```

В grid-контексте `z-index` работает без `position`. Это позволяет перекрывать grid-элементы, не выводя их из потока.

## Типичные ошибки и антипаттерны

- **Использование `overflow: hidden` для создания BFC.** Работает, но может обрезать контент и тени. `display: flow-root` — лучший выбор.
- **Clearfix через псевдоэлемент вместо BFC.** `.clearfix::after { content: ''; display: table; clear: both; }` — старый паттерн, который ещё нужен в редких случаях, но чаще достаточно `display: flow-root`.
- **Непонимание, почему margin «выпадает» из родителя.** Выпадение margin — нормальное поведение BFC, а не баг. Лечится padding, border или `display: flow-root`.
- **Попытка схлопнуть margin’ы в flex/grid.** В flex- и grid-контекстах margin’ы не схлопываются — это ожидаемо, и к этому нужно привыкнуть.
- **Забытое `position: relative` у предка absolute-элемента.** Если предок не позиционирован, элемент позиционируется относительно ближайшего подходящего предка или viewport.
- **Смешение `box-sizing` в одном проекте.** Глобальный `border-box` — стандарт де-факто; смешение `content-box` и `border-box` приводит к неожиданным размерам.

## Ключевые тезисы для интервью

- Formatting context — область с едиными правилами раскладки: BFC, IFC, FFC, GFC.
- BFC создаёт `display: flow-root`, `overflow` не `visible`, `float`, `position: absolute/fixed` и другие условия.
- Внутри BFC блочные элементы располагаются вертикально, а их вертикальные margin’ы схлопываются.
- `display: flow-root` — современный способ создать BFC без побочных эффектов.
- Containing block — область, относительно которой считаются размеры и позиция. Для `absolute` — ближайший позиционированный предок; для `fixed` — viewport.
- Margin collapse работает только в BFC, для соседних блоков и между родителем и крайними потомками.
- В flex/grid-контекстах margin’ы не схлопываются, а `z-index` работает без `position`.
- `box-sizing: border-box` включает padding и border в `width`/`height` и упрощает рассуждение о размерах.

## Полезные ссылки

- [Block formatting context](https://developer.mozilla.org/en-US/docs/Web/Guide/CSS/Block_formatting_context)
- [Inline formatting context](https://developer.mozilla.org/en-US/docs/Web/CSS/Inline_formatting_context)
- [Containing block](https://developer.mozilla.org/en-US/docs/Web/CSS/Containing_block)
- [Mastering margin collapsing](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_box_model/Mastering_margin_collapsing)
- [box-sizing](https://developer.mozilla.org/en-US/docs/Web/CSS/box-sizing)
