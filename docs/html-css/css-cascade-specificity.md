# Cascade, specificity, inheritance и `@layer`

CSS расшифровывается как Cascading Style Sheets, и именно каскад — сердце всей системы. Понимание cascade нужно не для заучивания формулы специфичности, а чтобы предсказывать, какое правило победит в неочевидных ситуациях: при подключении сторонней библиотеки, при переопределениях через `!important`, при использовании CSS-Layer’ов и при работе с Shadow DOM.

## Глубокий разбор

### Что такое cascade

**Cascade** (каскад) — это алгоритм, который браузер использует для разрешения конфликтов между декларациями, претендующими на один и тот же элемент и свойство. Он работает в четыре этапа:

1. **Origin и importance** — откуда пришла декларация и помечена ли она `!important`.
2. **Specificity** — насколько селектор точно нацелен на элемент.
3. **Order of appearance** — какая декларация появилась позже в потоке.
4. **Layer order** (с `@layer`) — в каком слое находится декларация.

Современный алгоритм можно записать так: сначала сравниваются origin + `!important`, затем layer, затем specificity, затем порядок. Если более ранний критерий различается, более поздние не проверяются.

### Origin: откуда берутся стили

Браузер собирает стили из трёх основных источников:

- **User agent stylesheet** — стили браузера по умолчанию. Это то, почему `<h1>` крупный, а `<a>` синий.
- **User stylesheet** — стили, заданные пользователем в настройках браузера или через расширения.
- **Author stylesheet** — стили разработчика: внешние файлы, `<style>`, inline-стили.

Кроме того, каждый источник делится на обычный и `!important`:

```
1. User agent normal
2. User normal
3. Author normal
4. Animations
5. Author !important
6. User !important
7. Transitions
```

Обратите внимание: `!important` в author-стилях проигрывает `!important` в user-стилях. Это сделано специально: пользователь должен иметь возможность переопределять авторский дизайн ради доступности. Animations и transitions живут в особых слоях, чтобы анимации могли временно переопределять обычные стили.

### CSS Layers: `@layer`

`@layer` позволяет явно управлять порядком каскада, не привязываясь к specificity. Слои объявляются один раз, и все правила внутри них считаются менее приоритетными, чем правила в слоях, объявленных позже.

```css
@layer reset, base, components, utilities;

@layer reset {
  body { margin: 0; }
}

@layer components {
  .button { background: blue; }
}

@layer utilities {
  .bg-red { background: red; }
}
```

Даже если `.button` имеет specificity `(0,1,0)`, а `.bg-red` — `(0,1,0)`, `.bg-red` победит, потому что слой `utilities` объявлен после `components`.

Важные нюансы:

- Необъявленные правила (вне `@layer`) имеют наивысший приоритет среди author-стилей.
- `@layer` влияет только на author-стили; он не может переопределить `!important` из user-стилей.
- Вложенные `@layer` работают: `@layer components.buttons { ... }`.

### Specificity

**Specificity** (специфичность) — это вес селектора. В современных браузерах она записывается как тройка `(A, B, C)`:

- **A** — количество ID-селекторов.
- **B** — количество селекторов классов, псевдоклассов и атрибутов.
- **C** — количество селекторов типа и псевдоэлементов.

`:where()` и служебные псевдоклассы внутри него не добавляют веса. `:is()` и `:not()` принимают вес наиболее специфичного аргумента.

```css
#nav .link { }           /* (1,1,0) */
nav ul li a { }          /* (0,0,4) */
.menu > .item:hover { }  /* (0,2,0) */
:where(.menu) a { }      /* (0,0,1) — :where обнуляет вес */
:is(#nav, a) { }         /* (1,0,0) — вес самого специфичного аргумента */
```

Inline-стили имеют специфичность `(1,0,0,0)` — выше, чем любой селектор, но ниже `!important`.

### Inheritance

**Inheritance** (наследование) — это механизм, при котором значения некоторых свойств передаются от родителя к потомкам. Наследуются в основном типографские и цветовые свойства: `color`, `font-family`, `font-size`, `line-height`, `text-align`, `visibility` и другие.

```css
body {
  color: #1f2937;
  font-family: system-ui, sans-serif;
}

/* color и font-family унаследуются всеми вложенными элементами */
```

Ключевые слова:

- `inherit` — явно наследовать значение от родителя.
- `initial` — сбросить к начальному значению свойства.
- `unset` — если свойство наследуется, вести себя как `inherit`; иначе как `initial`.
- `revert` — вернуть значение, которое было бы у элемента, если бы текущая декларация не применялась (учитывает origin и cascade).

```css
button {
  all: unset;       /* сбрасывает все свойства */
  display: inline-block;
}
```

### `!important`

`!important` поднимает декларацию в отдельный origin-уровень. Он побеждает любую обычную декларацию независимо от specificity, но уступает `!important` пользовательских стилей и animation/transition-уровням.

`!important` нарушает нормальный каскад и делает переопределения сложными. Его стоит использовать только тогда, когда это действительно необходимо:

- утилитарные классы вроде `.hidden { display: none !important; }`;
- переопределение inline-стилей, установленных сторонними библиотеками;
- accessibility-переопределения, которые пользователь должен иметь возможность изменить.

## Практические примеры

### Пример 1: `@layer` против specificity

```css
@layer base, overrides;

@layer base {
  #header .nav-link {
    color: blue; /* specificity (1,1,0) */
  }
}

@layer overrides {
  .nav-link.active {
    color: red; /* specificity (0,2,0) */
  }
}
```

```html
<header id="header">
  <a class="nav-link active">Link</a>
</header>
```

Несмотря на то что `#header .nav-link` специфичнее, `.nav-link.active` покрасит ссылку в красный, потому что слой `overrides` объявлен после `base`.

### Пример 2: `!important` и inline-стили

```html
<p id="text" style="color: green;">Text</p>
```

```css
#text {
  color: red !important;
}
```

Текст будет красным: `!important` в author-CSS побеждает inline-стили. Но если у пользователя в настройках браузера задано `p { color: black !important; }`, текст станет чёрным.

### Пример 3: наследование и `currentColor`

```css
.icon-button {
  color: #2563eb;
  border: 2px solid currentColor;
}

.icon-button svg {
  fill: currentColor;
}
```

```html
<button class="icon-button">
  <svg>...</svg>
  Save
</button>
```

`currentColor` — это ключевое слово, равное вычисленному значению `color` у текущего элемента. Граница и SVG-иконка унаследуют цвет текста кнопки. Это удобный способ строить темизируемые компоненты без лишних переменных.

### Пример 4: `revert` vs `unset`

```css
button.custom {
  all: unset;       /* button теряет display: inline-block, font, padding */
}

button.revert {
  all: revert;      /* возвращается к user agent стилям button */
}
```

`unset` сбрасывает всё к начальным или наследуемым значениям, игнорируя то, что было у `<button>` по умолчанию. `revert` возвращает стили, которые браузер назначает `<button>` изначально.

## Типичные ошибки и антипаттерны

- **Борьба со специфичностью через ещё большую специфичность.** Вместо `#app .header .nav .link` используйте `@layer` или более плоские селекторы.
- **`!important` ради быстрого фикса.** Каждый `!important` — это долг: потом его сложнее переопределить, чем обычное правило.
- **Игнорирование порядка `@layer`.** Если слои не объявлены в начале файла, порядок их первого появления определяет приоритет, что легко запутать.
- **Путаница `initial` и `unset`.** `initial` всегда сбрасывает к дефолту свойства, даже если оно наследуемое. `unset` учитывает наследование.
- **Использование `all: unset` на семантических элементах.** Кнопка после `all: unset` перестаёт быть доступной с клавиатуры. Для сброса используйте `all: revert` или целевые свойства.
- **Неучёт user-стилей и accessibility.** Пользовательские стили с `!important` могут переопределять ваши. Это не баг, а фича браузера.

## Ключевые тезисы для интервью

- Cascade разрешает конфликты через origin/importance, layer, specificity и порядок появления.
- Origin-уровни: user agent < user < author; `!important` меняет порядок, и author `!important` уступает user `!important`.
- `@layer` позволяет управлять приоритетом независимо от specificity: позже объявленный слой побеждает.
- Specificity — это `(A, B, C)`: ID, классы/атрибуты/псевдоклассы, типы/псевдоэлементы. `:where()` обнуляет вес, `:is()` берёт максимальный.
- Inline-стили имеют вес `(1,0,0,0)`, но уступают `!important` в author-CSS.
- Наследование распространяет типографику и цвет. Ключевые слова: `inherit`, `initial`, `unset`, `revert`.
- `!important` — инструмент для утилит и accessibility-переопределений, а не для повседневной борьбы со специфичностью.

## Полезные ссылки

- [Cascade and inheritance](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Cascade_and_inheritance)
- [Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity)
- [CSS Cascade Layers](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)
- [CSS Cascading and Inheritance Level 5](https://www.w3.org/TR/css-cascade-5/)
