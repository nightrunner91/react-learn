# Grid: explicit/implicit, `fr`, `minmax`, `auto-fit`/`auto-fill`, subgrid

CSS Grid — это двумерная система раскладки: одновременно по строкам и колонкам. В отличие от Flexbox, который распределяет пространство вдоль одной оси, Grid позволяет точно описывать структуру макета. Понимание explicit и implicit grid, единиц `fr` и функции `minmax` отделяет уверенное использование Grid от «метода тыка».

## Глубокий разбор

### Explicit и implicit grid

**Explicit grid** (явная сетка) — это треки, которые вы задали явно через `grid-template-columns`, `grid-template-rows` или `grid-template-areas`.

**Implicit grid** (неявная сетка) — треки, которые браузер добавляет автоматически, когда элементов больше, чем явных ячеек, или когда элемент выходит за границы явной сетки.

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: 100px;
}
```

Здесь три колонки заданы явно. Если элементов больше девяти, браузер создаст дополнительные строки высотой 100px — это implicit grid.

Размер implicit-треков управляется `grid-auto-columns` и `grid-auto-rows`. Если они не заданы, implicit треки получают размер по контенту.

### Единица `fr`

**`fr`** (fraction, доля) — это гибкая единица, которая распределяет **доступное** пространство после вычета фиксированных треков и gap’ов.

```css
.grid {
  display: grid;
  grid-template-columns: 200px 1fr 2fr;
}
```

После вычета 200px оставшееся место делится в пропорции 1:2. Важно: `fr` распределяет именно свободное пространство, а не всю ширину. Если контент в `1fr` больше, чем его доля, трек может стать шире — но не шире доступного места.

`fr` нельзя напрямую комбинировать с `auto` в одном выражении, но можно писать `1fr auto 2fr`. В таком случае `auto` сначала займёт место под контент, а `fr` поделит остаток.

### `minmax()`

Функция **`minmax(min, max)`** задаёт диапазон размеров трека. Трек не будет уже `min` и не шире `max`.

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(200px, 1fr));
}
```

Каждая колонка занимает равную долю, но не менее 200px. Когда места станет меньше 600px, Grid создаст переполнение, если не задан `auto-fit`/`auto-fill`.

Частое сочетание:

```css
.grid {
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
}
```

Это создаёт адаптивную сетку: колонки не уже 280px, а свободное место распределяется поровну.

### `auto-fit` vs `auto-fill`

`repeat()` может принимать `auto-fit` или `auto-fill` вместо фиксированного числа повторений.

- **`auto-fill`** — создаёт столько треков, сколько помещается в контейнер, даже пустых.
- **`auto-fit`** — создаёт столько треков, сколько есть элементов, и схлопывает пустые треки до нуля.

```css
.fill {
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
}

.fit {
  grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
}
```

Разница заметна, когда элементов меньше, чем может поместиться. В `auto-fill` останутся пустые треки; в `auto-fit` оставшееся место распределится между заполненными.

Для большинства интерфейсов используйте `auto-fit`: оно даёт поведение «растянуть элементы на всю ширину».

### `grid-template-areas`

Именованные области позволяют описывать макет визуально:

```css
.layout {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  min-height: 100vh;
}

.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main { grid-area: main; }
.footer { grid-area: footer; }
```

Преимущества:

- Макет читается как ASCII-арт.
- Перестроение на мобильных делается сменой `grid-template-areas`, а не селекторов.
- Области должны быть прямоугольниками без разрывов.

### Размещение элементов: линии и span

Элемент можно разместить по номерам линий:

```css
.item {
  grid-column: 1 / 3;
  grid-row: 2 / span 2;
}
```

`span 2` означает «занять два трека». Отрицательные номера линий отсчитываются с конца: `grid-column: 1 / -1` растянет элемент на всю ширину.

### Alignment в Grid

В Grid те же свойства, что и в Flexbox, но применяются к двум осям одновременно:

- `justify-items` — выравнивание содержимого ячеек по inline-оси (горизонталь).
- `align-items` — выравнивание по block-оси (вертикаль).
- `justify-content` — выравнивание всей сетки, если она уже контейнера.
- `align-content` — то же по вертикали.
- `justify-self` / `align-self` — переопределение для отдельного элемента.

```css
.grid {
  display: grid;
  place-items: center; /* сокращение для align-items + justify-items */
}
```

### Gap

```css
.grid {
  gap: 16px 24px; /* row-gap column-gap */
}
```

В отличие от margin’ов, gap не схлопывается и не создаёт лишних отступов по краям.

### Subgrid

**Subgrid** позволяет вложенному grid-контейнеру наследовать треки родительской сетки.

```css
.parent {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
}

.child {
  display: grid;
  grid-template-columns: subgrid;
  grid-column: 1 / -1;
}
```

Элемент `.child` занимает всю ширину родителя и использует его же колонки. Это решает проблему выравивания контента внутри вложенных компонентов по общей сетке.

На момент написания subgrid поддерживается в современных Firefox, Chrome, Edge и Safari. Для продакшена стоит проверять через `@supports`:

```css
.child {
  display: grid;
}

@supports (grid-template-columns: subgrid) {
  .child {
    grid-template-columns: subgrid;
  }
}
```

### Implicit vs explicit placement

Если не задать `grid-column`/`grid-row`, элемент размещается по алгоритму **auto-placement**: слева направо, сверху вниз, заполняя пустые ячейки. Свойство `grid-auto-flow` управляет направлением:

```css
.grid {
  grid-auto-flow: row; /* по умолчанию */
}

.grid-dense {
  grid-auto-flow: row dense; /* заполнять пустоты при наличии */
}
```

`dense` пытается заполнить дырки, оставшиеся от больших элементов, но это может нарушить визуальный порядок.

## Практические примеры

### Пример 1: адаптивная сетка карточек

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
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 16px;
}
```

Карточки растягиваются на всю ширину, но не уже 280px. При уменьшении viewport они переносятся.

### Пример 2: классический layout с областями

```html
<div class="layout">
  <header class="header">Header</header>
  <aside class="sidebar">Sidebar</aside>
  <main class="main">Content</main>
  <footer class="footer">Footer</footer>
</div>
```

```css
.layout {
  display: grid;
  grid-template-columns: 240px 1fr;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  min-height: 100vh;
  gap: 16px;
}

.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main { grid-area: main; }
.footer { grid-area: footer; }

@media (max-width: 768px) {
  .layout {
    grid-template-columns: 1fr;
    grid-template-rows: auto auto 1fr auto;
    grid-template-areas:
      "header"
      "sidebar"
      "main"
      "footer";
  }
}
```

Перестроение на мобильных достигается только изменением `grid-template-areas`.

### Пример 3: неравные колонки с `fr`

```css
.hero {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 24px;
}
```

Левая часть занимает две трети свободного места, правая — одну треть.

### Пример 4: `minmax()` для контента

```css
.table-grid {
  display: grid;
  grid-template-columns: minmax(120px, 1fr) 2fr minmax(80px, auto);
  gap: 8px;
}
```

Первая колонка не уже 120px, но может расти. Последняя — не уже 80px и по ширине контента.

### Пример 5: subgrid для вложенных карточек

```html
<div class="products">
  <article class="product">
    <h3>Product A</h3>
    <p>Description</p>
    <span class="price">$10</span>
  </article>
  <article class="product">
    <h3>Product B with longer name</h3>
    <p>Another description</p>
    <span class="price">$20</span>
  </article>
</div>
```

```css
.products {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 24px;
}

.product {
  display: grid;
  grid-template-rows: auto 1fr auto;
  gap: 8px;
}

@supports (grid-template-rows: subgrid) {
  .product {
    grid-template-rows: subgrid;
    grid-row: span 3;
  }
}
```

С subgrid заголовки, описания и цены разных карточек выравниваются по общей сетке строк.

## Типичные ошибки и антипаттерны

- **Использование Grid там, где достаточно Flexbox.** Grid — для двумерных макетов; Flexbox — для одномерных распределений. Для списка в одну строку часто проще Flexbox.
- **Путаница `auto-fit` и `auto-fill`.** `auto-fill` оставляет пустые треки; `auto-fit` схлопывает их. В 90% случаев нужен `auto-fit`.
- **Ожидание, что `fr` учитывает контент.** `fr` делит доступное пространство, но треки всё равно уважают минимальный размер контента. Для строгих ограничений используйте `minmax(0, 1fr)`.
- **Переполнение с `minmax(200px, 1fr)` без `auto-fit`/`auto-fill`.** Фиксированное число колонок с `minmax` даст горизонтальный скролл, если места не хватит.
- **Неправильные области в `grid-template-areas`.** Область должна быть сплошным прямоугольником. Разорванная фигура вызовет ошибку.
- **Игнорирование implicit grid.** Если элементы выходят за пределы явной сетки, их размеры определяются `grid-auto-rows`/`grid-auto-columns`, которые по умолчанию равны `auto`.
- **`grid-gap` вместо `gap`.** Устаревший префикс. Используйте просто `gap`.
- **Subgrid без fallback.** Subgrid не везде поддерживается; проверяйте через `@supports`.

## Ключевые тезисы для интервью

- Grid — двумерная система раскладки; Flexbox — одномерная. Grid управляет строками и колонками одновременно.
- Explicit grid — заданные явно треки; implicit grid — автоматически добавленные браузером.
- `fr` распределяет **доступное** пространство после фиксированных треков и gap’ов.
- `minmax(min, max)` задаёт диапазон размеров трека; часто используется с `repeat(auto-fit, minmax(...))`.
- `auto-fit` схлопывает пустые треки и растягивает заполненные; `auto-fill` оставляет пустые треки.
- `grid-template-areas` позволяет описывать макет визуально и легко перестраивать его через media queries.
- `justify-items`/`align-items` выравнивают содержимое ячеек; `justify-content`/`align-content` — всю сетку в контейнере.
- Subgrid позволяет вложенной сетке наследовать треки родителя, что упрощает выравнивание сложных компонентов.

## Полезные ссылки

- [CSS Grid Layout](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout)
- [grid-template-columns](https://developer.mozilla.org/en-US/docs/Web/CSS/grid-template-columns)
- [minmax()](https://developer.mozilla.org/en-US/docs/Web/CSS/minmax)
- [repeat()](https://developer.mozilla.org/en-US/docs/Web/CSS/repeat)
- [Subgrid](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout/Subgrid)
- [A Complete Guide to Grid](https://css-tricks.com/snippets/css/complete-guide-grid/)
