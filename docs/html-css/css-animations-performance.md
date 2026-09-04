# Transitions, animations и производительность CSS

CSS-анимации — не просто «украшательство». На собеседованиях их спрашивают через призму производительности: почему одни свойства тормозят, а другие нет, когда стоит использовать `will-change`, что такое compositor thread и почему `transform` предпочтительнее `top`/`left`. В этой статье разберём механизм отрисовки анимаций и научимся писать их так, чтобы они не ломали производительность страницы.

## Глубокий разбор

### Transitions

**Transition** (переход) плавно меняет значение CSS-свойства между двумя состояниями при изменении условий. Срабатывает, когда браузер может интерполировать начальное и конечное значение.

```css
.button {
  background: #3b82f6;
  transition: background 0.2s ease, transform 0.2s ease;
}

.button:hover {
  background: #2563eb;
  transform: scale(1.02);
}
```

Ключевые подсвойства:

- `transition-property` — какие свойства анимируются.
- `transition-duration` — длительность.
- `transition-timing-function` — кривая ускорения.
- `transition-delay` — задержка перед стартом.

Transition запускается только на изменении вычисленного значения. Если свойство не изменилось (например, элемент уже имел нужный класс при загрузке), анимации не будет.

### Animations и `@keyframes`

**CSS animations** позволяют описывать многошаговые анимации через `@keyframes` и управлять ими независимо от состояния элемента.

```css
@keyframes pulse {
  0%, 100% {
    opacity: 1;
    transform: scale(1);
  }
  50% {
    opacity: 0.7;
    transform: scale(1.05);
  }
}

.badge {
  animation: pulse 2s ease-in-out infinite;
}
```

Ключевые свойства:

- `animation-name` — имя keyframes.
- `animation-duration` — длительность цикла.
- `animation-timing-function` — кривая.
- `animation-delay` — задержка.
- `animation-iteration-count` — количество повторов (`1`, `2`, `infinite`).
- `animation-direction` — направление (`normal`, `reverse`, `alternate`).
- `animation-fill-mode` — как применяются стили до/после анимации (`forwards`, `backwards`, `both`).
- `animation-play-state` — `running` или `paused`.

`animation-fill-mode: forwards` полезна, когда финальное состояние анимации должно остаться после завершения. `both` применяет стили и до старта, и после финиша.

### Как браузер рисует анимацию

Браузер проходит несколько этапов при каждом кадре:

1. **Style** — пересчёт стилей, если что-то изменилось.
2. **Layout** — расчёт геометрии: размеров и положения элементов.
3. **Paint** — отрисовка пикселей: текста, фона, теней, рамок.
4. **Composite** — сборка слоёв в финальную картинку.

Анимации разных свойств затрагивают разные этапы:

| Свойство | Этапы | Производительность |
|----------|-------|-------------------|
| `transform`, `opacity` | Composite | Лучшая |
| `color`, `background-color`, `box-shadow` | Paint | Средняя |
| `width`, `height`, `top`, `left`, `margin` | Layout + Paint + Composite | Худшая |
| `filter` (кроме `opacity` внутри) | Paint/Composite | Зависит от фильтра |

Чем больше этапов задействовано, тем дороже анимация. Layout — самый дорогой, потому что вынуждает пересчитывать геометрию всего дерева, которое зависит от изменившегося элемента.

### Composite-only свойства

**Compositor-only properties** — свойства, которые могут быть обработаны compositor thread без участия main thread. К ним относятся прежде всего:

- `transform` (translate, scale, rotate)
- `opacity`

Compositor thread отвечает за сборку финального кадра из заранее подготовленных слоёв. Анимации на этом потоке не блокируются JavaScript, layout и paint, поэтому они плавные даже под нагрузкой.

```css
/* Хорошо: анимация только compositor */
.card {
  transition: transform 0.3s ease, opacity 0.3s ease;
}

.card:hover {
  transform: translateY(-4px);
  opacity: 0.9;
}
```

По возможности анимации перемещения стоит делать через `transform: translateX(...)`, а не `left`/`margin-left`. Изменение `left` вынуждает браузер делать layout на каждом кадре.

### `will-change`

**`will-change`** — подсказка браузеру, что элемент скоро будет анимироваться, и стоит подготовить отдельный слой или другие ресурсы.

```css
.slider-thumb {
  will-change: transform;
}
```

Важные нюансы:

- `will-change` создаёт отдельный compositor layer, что расходует память. Слишком много слоёв может привести к out-of-memory на слабых устройствах.
- Не стоит вешать `will-change` на все элементы заранее. Лучше добавлять перед анимацией и убирать после.
- Значение `will-change: auto` снимает оптимизацию.
- Некоторые свойства вроде `will-change: width` заставляют браузер держать элемент на main thread, поэтому пользы мало.

Рекомендуемый паттерн — включать `will-change` по триггеру, а не держать постоянно:

```css
.card {
  transition: transform 0.3s ease;
}

.card:hover {
  will-change: transform;
  transform: scale(1.02);
}
```

### CSS containment: `contain`

**Containment** (изоляция) ограничивает область влияния элемента, позволяя браузеру оптимизировать рендеринг. Свойство `contain` принимает значения:

- `layout` — внутреннее расположение элементов не влияет наружу, и наоборот.
- `paint` — дети не могут выходить за границы элемента; браузер может рисовать их в отдельный слой.
- `size` — размеры элемента не зависят от детей.
- `style` — счётчики и quote-свойства изолированы.
- `content` — комбинация `layout paint style`.
- `strict` — комбинация `layout paint size style`.

```css
.widget {
  contain: layout paint;
}
```

Для анимаций `contain: paint` особенно полезен: он гарантирует, что перерисовка ограничится элементом и не затронет соседей.

### `content-visibility` для долгих списков

**`content-visibility: auto`** позволяет пропускать layout и paint для элементов, находящихся вне viewport. Это сильно ускоряет первоначальный рендеринг больших списков и страниц.

```css
.card {
  content-visibility: auto;
  contain-intrinsic-size: 0 200px;
}
```

`contain-intrinsic-size` задаёт примерный размер элемента, чтобы скроллбар не прыгал при подгрузке контента.

### `prefers-reduced-motion`

Пользователи могут отключать анимации в системе. Через медиа-запрос `prefers-reduced-motion` можно адаптировать интерфейс:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

Но глобальный сброс ломает анимации, которые несут смысл: загрузка, открытие модалки, переключение состояний. Лучше отключать конкретные анимации:

```css
@media (prefers-reduced-motion: reduce) {
  .carousel-slide {
    transition: none;
    animation: none;
  }
}
```

## Практические примеры

### Пример 1: перемещение через `transform`

```html
<div class="box"></div>
```

```css
.box {
  width: 100px;
  height: 100px;
  background: #3b82f6;
  transition: transform 0.3s ease;
}

.box:hover {
  transform: translateX(100px);
}
```

Перемещение через `transform` работает на compositor thread и не вызывает layout. В отличие от `left: 100px`, здесь не пересчитывается геометрия соседей.

### Пример 2: анимация появления через opacity

```html
<div class="toast">Saved</div>
```

```css
.toast {
  opacity: 0;
  transform: translateY(16px);
  transition: opacity 0.3s ease, transform 0.3s ease;
}

.toast.is-visible {
  opacity: 1;
  transform: translateY(0);
}
```

Одновременная анимация `opacity` и `transform` — безопасный и плавный способ показать элемент.

### Пример 3: правильное использование `will-change`

```css
.modal {
  opacity: 0;
  transform: translateY(-20px);
  transition: opacity 0.25s ease, transform 0.25s ease;
}

.modal.is-open {
  will-change: transform, opacity;
  opacity: 1;
  transform: translateY(0);
}

.modal.is-open.is-settled {
  will-change: auto;
}
```

После завершения анимации класс `is-settled` убирает `will-change`, освобождая ресурсы compositor layer.

### Пример 4: бесконечная анимация с `animation-play-state`

```css
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.spinner {
  animation: spin 1s linear infinite;
}

.spinner.is-paused {
  animation-play-state: paused;
}
```

Для управления анимацией не нужно убирать `animation` — достаточно поставить на паузу.

### Пример 5: `contain` для изолированного виджета

```css
.chart {
  contain: layout paint;
  will-change: transform;
}

.chart.is-updating {
  transform: translateZ(0);
}
```

`contain: layout paint` гарантирует, что перерисовка виджета не затронет страницу целиком. `translateZ(0)` — старый трюк для принудительного создания слоя, но `will-change` предпочтительнее.

### Пример 6: `content-visibility` для ленты

```css
.feed-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 300px;
}
```

Элементы ленты, находящиеся вне viewport, не участвуют в layout и paint до появления на экране.

## Типичные ошибки и антипаттерны

- **Анимация `width`/`height`/`top`/`left`/`margin` вместо `transform`.** Эти свойства вызывают layout на каждом кадре и часто приводят к dropped frames.
- **Постоянное `will-change` на всех элементах.** Создаёт лишние compositor-слои, жрёт память и может замедлить рендеринг на слабых устройствах.
- **Игнорирование `prefers-reduced-motion`.** Для многих пользователей анимации вызывают головокружение или дискомфорт; системная настройка должна уважаться.
- **Анимация `box-shadow` на больших площадях.** `box-shadow` рисуется на каждом кадре и часто дорог в paint.
- **Одновременная анимация десятков элементов.** Даже compositor-анимации имеют предел по количеству слоёв. Лучше анимировать один родительский слой, чем 50 детей.
- **Использование `@keyframes` там, где достаточно `transition`.** Animation удобна для циклических или многошаговых сценариев; для простых состояний проще и понятнее transition.
- **Отсутствие `contain-intrinsic-size` с `content-visibility: auto`.** Без него скроллбар может менять размеры при подгрузке элементов.
- **Анимация свойств с `!important`.** Браузер может отказываться интерполировать значения, помеченные `!important`, что ломает transition.

## Ключевые тезисы для интервью

- Браузер рисует кадр через этапы: Style → Layout → Paint → Composite. Чем раньше этап, который затрагивает анимация, тем дороже она обходится.
- Безопасные для производительности свойства: `transform` и `opacity`. Они выполняются на compositor thread и не вызывают layout/paint.
- Опасные для анимации: `width`, `height`, `top`, `left`, `margin`, `padding` — всё, что меняет геометрию.
- `will-change` подсказывает браузеру подготовить слой, но злоупотребление им вредит: каждый слой требует памяти.
- `contain: layout paint` изолирует элемент и ограничивает область перерисовки.
- `content-visibility: auto` пропускает рендеринг элементов вне viewport, ускоряя первичный рендеринг длинных списков.
- `prefers-reduced-motion: reduce` нужно уважать; лучше отключать конкретные анимации, а не все сразу.
- `transition` — между двумя состояниями; `animation`/@keyframes — для многошаговых и циклических сценариев.

## Полезные ссылки

- [Using CSS transitions](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_transitions/Using_CSS_transitions)
- [Using CSS animations](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_animations/Using_CSS_animations)
- [will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change)
- [contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain)
- [content-visibility](https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility)
- [CSS Triggers](https://csstriggers.com/)
- [High performance animations](https://web.dev/articles/animations-overview)
- [prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)
