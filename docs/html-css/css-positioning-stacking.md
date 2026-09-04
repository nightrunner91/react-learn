# Positioning, stacking context, `z-index` и paint order

Позиционирование — это один из самых частых источников путаницы в CSS. Разработчики знают, что `position: absolute` выводит элемент из потока, но теряются, когда речь заходит о containing block, stacking context’ах и порядке отрисовки. Эта статья связывает позиционирование, наложение и порядок рисования в единую картину.

## Глубокий разбор

### Типы позиционирования

CSS предлагает пять значений `position`:

- `static` — значение по умолчанию. Элемент находится в нормальном потоке, свойства смещения (`top`, `right`, `bottom`, `left`, `inset`) игнорируются.
- `relative` — элемент остаётся в потоке, но может смещаться относительно своего нормального положения.
- `absolute` — элемент выводится из потока и позиционируется относительно containing block.
- `fixed` — элемент позиционируется относительно viewport (или containing block, если он создаётся `transform`/`filter`/`perspective`/`contain` у предка).
- `sticky` — гибрид `relative` и `fixed`: элемент ведёт себя как `relative`, пока не достигнет заданного порога в пределах своего ближайшего прокручиваемого предка.

### `position: relative`

**Relative positioning** (относительное позиционирование) смещает элемент, не меняя его место в потоке. Окружающие элементы по-прежнему считают, что он занимает исходную позицию.

```css
.box {
  position: relative;
  top: 20px;
  left: 20px;
}
```

Элемент сдвинется вниз и вправо на 20px, но пространство, которое он занимал, останется пустым. Это удобно для небольших сдвигов, но плохо подходит для раскладки.

### `position: absolute`

**Absolute positioning** (абсолютное позиционирование) полностью выводит элемент из нормального потока. Его размер и местоположение определяются относительно **containing block** (содержащего блока).

Containing block для `absolute` — это ближайший предок, у которого `position` не `static`, либо предок с `transform`, `filter`, `perspective` или `contain: paint/layout`. Если такого нет — containing block станет `<html>`.

```css
.card {
  position: relative;
}

.badge {
  position: absolute;
  top: 8px;
  right: 8px;
}
```

Важные нюансы:

- Абсолютно позиционированный элемент сжимается до ширины контента, если явно не задана ширина.
- `margin: auto` в сочетании с `inset: 0` и фиксированными размерами центрирует элемент в containing block.

### `position: fixed`

**Fixed positioning** (фиксированное позиционирование) похоже на `absolute`, но containing block по умолчанию — viewport. Элемент не прокручивается вместе со страницей.

```css
.toast {
  position: fixed;
  bottom: 24px;
  right: 24px;
}
```

Исключение: если предок имеет `transform`, `filter`, `perspective` или `contain`, он становится containing block’ом для `fixed`, и элемент начинает позиционироваться относительно него. Это часто ломает модальные окна, вложенные в анимированные контейнеры.

### `position: sticky`

**Sticky positioning** (липкое позиционирование) требует указания хотя бы одного порога (`top`, `right`, `bottom`, `left`). Элемент ведёт себя как `relative`, пока его позиция в прокручиваемом контейнере не достигнет порога. После этого он «прилипает».

```css
.header {
  position: sticky;
  top: 0;
}
```

Условия работы:

- Ближайший предок с прокруткой должен быть выше sticky-элемента в DOM.
- Родительский контейнер должен быть достаточно высоким, чтобы была область для «прилипания».
- Свойство `overflow` у предков может влиять на поведение sticky.

### Stacking context

**Stacking context** (контекст наложения) — это трёхмерная концепция: элементы внутри одного stacking context’а рисуются слоями от дальних к ближним. `z-index` работает только внутри одного stacking context’а и не позволяет элементу «пробить» границу родительского контекста.

Stacking context создаётся:

- Корневой элемент (`<html>`).
- Элемент с `position: absolute`/`relative`/`fixed`/`sticky` и явным `z-index`, отличным от `auto`.
- Элемент с `opacity` меньше 1.
- Элемент с `transform`, `filter`, `perspective`, `clip-path`, `mask`.
- Элемент с `isolation: isolate`.
- Элемент с `mix-blend-mode`, отличным от `normal`.
- Flex/grid-контейнер, у которого дети имеют `z-index`, отличный от `auto`.
- Элемент с `will-change`.
- Элемент с `contain: layout`/`paint`/`strict`/`content`.

### `z-index`

**`z-index`** управляет порядком наложения элементов внутри одного stacking context’а. Большое значение рисуется ближе к пользователю.

```css
.modal {
  position: fixed;
  z-index: 100;
}

.tooltip {
  position: absolute;
  z-index: 10;
}
```

Ключевые моменты:

- `z-index` не работает для `position: static`.
- `z-index: auto` не создаёт нового stacking context’а.
- Дочерний элемент с огромным `z-index` не может выйти за пределы stacking context’а родителя. Если родитель `.tooltip` лежит под `.modal`, дочерний элемент `.tooltip` не перекроет `.modal`, сколько бы `z-index` ему ни задали.

### Порядок отрисовки внутри stacking context

Внутри одного stacking context’а браузер рисует элементы в таком порядке (от дальнего к ближнему):

1. Фон и border контекста.
2. Дочерние элементы с отрицательным `z-index`.
3. Элементы в нормальном потоке (`static`, `relative` без `z-index`).
4. Плавающие элементы (`float`).
5. Строчные элементы (inline).
6. Дочерние элементы с `position` и `z-index: auto` (или без `z-index`).
7. Дочерние элементы с положительным `z-index`.

Это объясняет, почему иногда `position: relative` без `z-index` перекрывает float, а иногда нет: порядок рисования зависит от комбинации факторов.

## Практические примеры

### Пример 1: центрирование через `position: absolute`

```html
<div class="overlay">
  <div class="dialog">Dialog</div>
</div>
```

```css
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
}

.dialog {
  position: absolute;
  inset: 0;
  width: 400px;
  height: 200px;
  margin: auto;
}
```

`inset: 0` растягивает `.dialog` до границ `.overlay`, а `margin: auto` центрирует его по заданным размерам.

### Пример 2: `z-index` не пробивает родительский stacking context

```html
<div class="parent-a">
  <div class="child-a">A child</div>
</div>

<div class="parent-b">
  <div class="child-b">B child</div>
</div>
```

```css
.parent-a,
.parent-b {
  position: relative;
  z-index: 1;
}

.child-a {
  position: absolute;
  z-index: 9999;
}

.child-b {
  position: absolute;
  z-index: 2;
}
```

Если `.parent-b` в DOM идёт после `.parent-a`, он рисуется поверх него. `child-a` с `z-index: 9999` всё равно останется под `.parent-b`, потому что он «заперт» внутри stacking context’а `.parent-a`.

### Пример 3: sticky-шапка таблицы

```html
<table>
  <thead>
    <tr><th>Name</th><th>Value</th></tr>
  </thead>
  <tbody>...</tbody>
</table>
```

```css
thead th {
  position: sticky;
  top: 0;
  background: white;
}
```

Шапка прилипает к верху viewport при прокрутке таблицы. Важно задать фон, иначе содержимое строк будет просвечивать сквозь шапку.

### Пример 4: создание stacking context без побочных эффектов

```css
.dropdown {
  position: relative;
  isolation: isolate;
}

.dropdown-menu {
  position: absolute;
  z-index: 10;
}
```

`isolation: isolate` создаёт новый stacking context, не добавляя трансформаций и не меняя прозрачность. Это полезно для компонентов вроде dropdown: меню будет рисоваться поверх соседей, но не выйдет за пределы своего компонента.

### Пример 5: `fixed` внутри трансформированного контейнера

```html
<div class="transformed">
  <div class="fixed">Fixed</div>
</div>
```

```css
.transformed {
  transform: translateX(0);
}

.fixed {
  position: fixed;
  top: 0;
  left: 0;
}
```

Несмотря на `position: fixed`, элемент `.fixed` будет позиционироваться относительно `.transformed`, а не viewport. Это одна из самых неприятных ловушек при работе с модальными окнами и поповерами.

## Типичные ошибки и антипаттерны

- **Большие значения `z-index` как решение всех проблем.** Если элемент не перекрывает соседа, чаще всего дело в stacking context’е, а не в недостаточном `z-index`.
- **Забытое `position: relative` у предка absolute-элемента.** Элемент улетит к ближайшему позиционированному предку или к viewport.
- **Использование `z-index` без позиционирования.** У `position: static` `z-index` не работает.
- **Попытки вынести `fixed`-элемент за пределы трансформированного предка.** Любой предок с `transform`/`filter`/`perspective`/`contain` превращается в containing block для `fixed`.
- **Sticky, который не работает из-за `overflow`.** Если все предки имеют `overflow: hidden` без прокрутки, sticky может не «прилипнуть».
- **Путаница визуального и DOM-порядка.** `z-index` меняет только визуальное наложение; таб-фокус и скринридеры по-прежнему следуют DOM.
- **Непонимание paint order.** Даже без `z-index` браузер рисует элементы в строгом порядке: фон, отрицательный `z-index`, поток, float, inline, позиционированные элементы, положительный `z-index`.

## Ключевые тезисы для интервью

- `position` бывает `static`, `relative`, `absolute`, `fixed`, `sticky`. Только `relative`/`absolute`/`fixed`/`sticky` создают позиционированный элемент.
- Containing block для `absolute` — ближайший не-static предок или предок с `transform`/`filter`/`perspective`/`contain`; для `fixed` — обычно viewport.
- `position: sticky` требует порога (`top`/`bottom`/`left`/`right`) и прокручиваемого предка.
- Stacking context — изолированная группа слоёв. `z-index` работает только внутри одного stacking context’а.
- Stacking context создаёт `z-index` у позиционированного элемента, `opacity < 1`, `transform`, `filter`, `isolation: isolate`, flex/grid-контейнер с `z-index` у детей и ряд других свойств.
- Дочерний элемент не может перекрыть элемент за пределами stacking context’а своего родителя, даже с огромным `z-index`.
- Порядок отрисовки внутри stacking context: фон контекста → отрицательный `z-index` → поток → float → inline → позиционированные → положительный `z-index`.
- `transform`/`filter`/`perspective` у предка ломает `position: fixed`, делая предка containing block’ом.

## Полезные ссылки

- [Positioning](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Positioning)
- [The stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Understanding_z-index/Stacking_context)
- [z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/z-index)
- [Stacking without z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Understanding_z-index/Stacking_without_z-index)
- [Stacking with floated blocks](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Understanding_z-index/Stacking_and_float)
- [CSS Positioned Layout Module Level 3](https://www.w3.org/TR/css-position-3/)
