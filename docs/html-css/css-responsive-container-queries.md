# Media queries, container queries, viewport units и `prefers-*`

Адаптивность изначально строилась вокруг viewport: ширина экрана определяла, как выглядит интерфейс. Но в компонентной эре этого недостаточно: один и тот же компонент может жить и в узком сайдбаре, и в широком основном контенте. Container queries решают эту задачу, смещая фокус с экрана на размер самого компонента. Вместе с viewport units и `prefers-*`-запросами они формируют современный инструментарий адаптивного CSS.

## Глубокий разбор

### Media queries

**Media queries** (медиа-запросы) позволяют применять стили в зависимости от характеристик устройства, браузера или предпочтений пользователя.

```css
@media (min-width: 768px) {
  .layout {
    display: grid;
    grid-template-columns: 240px 1fr;
  }
}
```

Основные media features:

- `width`, `min-width`, `max-width` — ширина viewport.
- `height`, `min-height`, `max-height` — высота viewport.
- `aspect-ratio` — соотношение сторон viewport.
- `orientation` — `portrait` или `landscape`.
- `hover` — есть ли устройство, поддерживающее наведение (`hover: hover` / `hover: none`).
- `pointer` — точность указателя (`coarse` для тачскрина, `fine` для мыши/стилуса).
- `prefers-*` — пользовательские предпочтения (о них ниже).

Медиа-запросы логично комбинировать:

```css
@media (min-width: 768px) and (max-width: 1199px) {
  /* только планшеты */
}
```

### Container queries

**Container queries** (контейнерные запросы) применяют стили на основе размеров не viewport, а конкретного контейнера. Это позволяет компоненту адаптироваться под пространство, которое ему выделено.

Чтобы использовать container queries, нужно сначала объявить контейнер:

```css
.card-list {
  container-type: inline-size;
}
```

`container-type` может принимать значения:

- `size` — запросы могут проверять и ширину, и высоту контейнера.
- `inline-size` — запросы могут проверять только inline-размер (ширину в горизонтальном письме).
- `normal` — контейнер не создаётся.

После этого можно писать запросы:

```css
@container (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 120px 1fr;
  }
}
```

Контейнеры можно именовать:

```css
.card-list {
  container-type: inline-size;
  container-name: cards;
}

@container cards (min-width: 400px) {
  .card { ... }
}
```

### Viewport units

**Viewport units** (единицы viewport) выражают размер относительно viewport.

- `vw` — 1% ширины viewport.
- `vh` — 1% высоты viewport.
- `vmin` — меньшее из `vw` и `vh`.
- `vmax` — большее из `vw` и `vh`.

```css
.hero {
  min-height: 100vh;
}
```

В мобильных браузерах появились учитывающие интерфейс браузера единицы:

- `svh` / `svw` — small viewport (минимальные размеры без учёта скрывающихся панелей).
- `lvh` / `lvw` — large viewport (максимальные размеры).
- `dvh` / `dvw` — dynamic viewport (текущие размеры с учётом панелей).

```css
.hero {
  min-height: 100dvh; /* адаптируется под появление/скрытие панелей */
}
```

Также существуют единицы относительно контейнера:

- `cqw` — 1% ширины контейнера.
- `cqh` — 1% высоты контейнера.
- `cqi` — 1% inline-размера контейнера.
- `cqb` — 1% block-размера контейнера.

### `prefers-*` и доступность

Группа media features `prefers-*` позволяет адаптировать интерфейс под пользовательские системные настройки.

#### `prefers-color-scheme`

```css
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #111827;
    --text: #f9fafb;
  }
}
```

#### `prefers-reduced-motion`

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

Важно: жёсткий сброс анимаций может ломать компоненты, которые используют transition для состояний. Лучше сбрасывать конкретные анимации, а не все подряд.

#### `prefers-contrast`

```css
@media (prefers-contrast: more) {
  .button {
    border: 2px solid currentColor;
  }
}
```

#### `prefers-reduced-transparency`

```css
@media (prefers-reduced-transparency: reduce) {
  .glass {
    background: rgba(255, 255, 255, 0.95);
  }
}
```

## Практические примеры

### Пример 1: карточка, адаптирующаяся под ширину родителя

```html
<div class="card-list">
  <article class="card">
    <img src="thumb.jpg" alt="">
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </article>
</div>
```

```css
.card-list {
  container-type: inline-size;
  container-name: cards;
}

.card {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

@container cards (min-width: 400px) {
  .card {
    flex-direction: row;
  }

  .card img {
    width: 120px;
    flex-shrink: 0;
  }
}
```

Если `.card-list` шире 400px, карточка перестраивается в горизонтальный вид. Если этот же компонент вставить в узкий сайдбар, он останется вертикальным.

### Пример 2: комбинация media и container queries

```css
.page {
  display: grid;
  grid-template-columns: 1fr;
}

@media (min-width: 1024px) {
  .page {
    grid-template-columns: 280px 1fr;
  }
}

.widget {
  container-type: inline-size;
}

@container (min-width: 500px) {
  .widget {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }
}
```

Media query управляет глобальной раскладкой страницы, а container query — поведением виджета внутри выделенного ему пространства.

### Пример 3: viewport units с динамической высотой

```css
.fullscreen {
  min-height: 100dvh;
  display: grid;
  place-items: center;
}
```

`100dvh` лучше подходит для мобильных устройств, где адресная панель может появляться и скрываться. `100vh` на iOS может давать лишний скролл или обрезку контента.

### Пример 4: адаптация под предпочтения пользователя

```css
:root {
  --bg: #ffffff;
  --text: #111827;
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: #111827;
    --text: #f9fafb;
  }
}

@media (prefers-contrast: more) {
  :root {
    --text: #000000;
  }
}

@media (prefers-reduced-motion: reduce) {
  .spinner {
    animation: none;
  }
}
```

### Пример 5: container query units для типографики

```css
.banner {
  container-type: inline-size;
}

.banner-title {
  font-size: clamp(1.25rem, 5cqi, 3rem);
}
```

Размер заголовка масштабируется относительно ширины баннера, а не viewport. Это удобно для переиспользуемых компонентов.

## Типичные ошибки и антипаттерны

- **Использование viewport-запросов там, где нужны container queries.** Компонент не должен знать, на каком экране он находится; он должен реагировать на свой размер.
- **Забытое `container-type` у родителя.** Без него `@container` не найдёт контекста и не применится.
- **Использование `100vh` на мобильных без учёта панелей.** Для полноэкранных секций предпочтительнее `100dvh`/`100svh`.
- **Переопределение всех анимаций в `prefers-reduced-motion`.** Грубый сброс ломает микровзаимодействия. Отключайте именно те анимации, которые не несут смысловой нагрузки.
- **Игнорирование `prefers-contrast` и `prefers-reduced-transparency`.** Эти настройки важны для пользователей с нарушениями зрения.
- **Путаница `min-width` и `max-width` в media queries.** Mobile-first подход предполагает базовые стили для маленьких экранов и `@media (min-width: ...)` для больших.
- **Слишком много breakpoint’ов.** Если в проекте десятки breakpoints, возможно, стоит перейти к container queries или использовать относительные единицы.
- **Неучёт логических осей в container queries.** В вертикальном письме `inline-size` — это высота, а не ширина. `cqi` и `cqb` помогают писать более универсальный код.

## Ключевые тезисы для интервью

- Media queries реагируют на характеристики viewport и устройства: `width`, `height`, `orientation`, `hover`, `pointer`, `prefers-*`.
- Container queries реагируют на размер контейнера, а не экрана; требуют `container-type` у родителя.
- `container-type: inline-size` позволяет запрашивать ширину контейнера; `size` — и ширину, и высоту.
- Контейнеры можно именовать через `container-name` и использовать в `@container name (условие)`.
- Viewport units: `vw`/`vh`, `vmin`/`vmax`, а также `svh`/`lvh`/`dvh` для мобильных браузеров.
- Container query units: `cqw`/`cqh`/`cqi`/`cqb` выражают размер относительно контейнера.
- `prefers-color-scheme` — тёмная/светлая тема; `prefers-reduced-motion` — уменьшение анимаций; `prefers-contrast` и `prefers-reduced-transparency` — accessibility.
- `prefers-reduced-motion` стоит обрабатывать избирательно, а не отключать все transition и animation глобально.
- Container queries лучше media queries для компонентов, которые могут находиться в разных частях layout’а.

## Полезные ссылки

- [Using media queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_media_queries/Using_media_queries)
- [Container queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries)
- [container-type](https://developer.mozilla.org/en-US/docs/Web/CSS/container-type)
- [Viewport units](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Values_and_units#viewport_units)
- [prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
- [prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)
- [CSS Conditional Rules Module Level 5](https://www.w3.org/TR/css-conditional-5/)
