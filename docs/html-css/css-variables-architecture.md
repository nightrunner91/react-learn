# CSS custom properties, архитектура стилей и подходы к инкапсуляции

CSS на больших проектах перестаёт быть просто набором правил: появляется необходимость в системе токенов, темизации, масштабируемой методологии именования и способе изолировать стили компонентов. CSS custom properties решают часть этих задач на уровне самого языка, но важно понимать их область видимости, ограничения и место среди методологий вроде BEM, CUBE, ITCSS и инструментов вроде CSS Modules и CSS-in-JS.

## Глубокий разбор

### CSS custom properties

**Custom properties** (пользовательские свойства, часто называемые CSS-переменными) — это именованные значения, объявляемые с префиксом `--` и используемые через `var()`.

```css
:root {
  --color-primary: #3b82f6;
  --space-md: 1rem;
  --radius-base: 0.5rem;
}

.button {
  background: var(--color-primary);
  padding: var(--space-md);
  border-radius: var(--radius-base);
}
```

В отличие от препроцессорных переменных (Sass, Less), custom properties живут в runtime браузера: их можно переопределять в медиа-запросах, псевдоклассах, через JavaScript и наследовать по DOM.

### Область видимости и наследование

Custom properties наследуются, как и большинство CSS-свойств. Область видимости определяется селектором, в котором они объявлены.

```css
:root {
  --accent: blue;
}

.card {
  --accent: green;
}

.card .title {
  color: var(--accent); /* green, если .title внутри .card */
}
```

Это позволяет создавать локальные переопределения: один и тот же компонент выглядит по-разному в разных контекстах без изменения HTML или классов.

### Резервные значения

`var()` поддерживает второй аргумент — значение по умолчанию, которое используется, если переменная не определена или недействительна для данного свойства.

```css
.button {
  color: var(--button-color, white);
}
```

Важный нюанс: резервное значение подставляется только если переменная не задана. Если переменная задана, но содержит невалидное для свойства значение, резервное не сработает — свойство получит initial value.

### Типизация и вычисления

CSS custom properties — это строки. Браузер не знает, что внутри `--size: 16px`, пока не подставит значение в свойство. Поэтому нельзя просто сложить `var(--a) + var(--b)`; для вычислений используется `calc()`.

```css
:root {
  --base: 1rem;
  --scale: 2;
}

.box {
  padding: calc(var(--base) * var(--scale));
}
```

Если переменная используется в месте, где ожидается число без единиц, иногда применяют трюк с умножением на единицу: `calc(var(--value) * 1px)`.

### Темизация и токены

Custom properties — стандартный способ реализации **design tokens**: именованных значений цветов, отступов, типографики, теней и т.д.

```css
:root {
  --color-bg: #ffffff;
  --color-text: #111827;
  --color-border: #e5e7eb;
  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --font-base: system-ui, sans-serif;
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-bg: #111827;
    --color-text: #f9fafb;
    --color-border: #374151;
  }
}
```

Токены обычно делят на уровни:

- **Primitive tokens** — низкоуровневые значения: `--blue-500`, `--space-4`.
- **Semantic tokens** — значения смысла: `--color-surface`, `--color-text-default`.
- **Component tokens** — значения для конкретного компонента: `--button-bg`, `--input-border`.

Семантический слой позволяет менять тему, не трогая примитивы: достаточно переназначить `--color-surface` в зависимости от контекста.

### Методологии именования

#### BEM

**BEM** (Block, Element, Modifier) — классическая методология для именования классов:

```html
<button class="button button--primary button--large">Save</button>
```

```css
.button { }
.button__icon { }
.button--primary { }
.button--large { }
```

BEM решает проблему специфичности и делает структуру классов предсказуемой. Недостаток: длинные имена и жёсткая привязка к компоненту.

#### CUBE CSS

**CUBE CSS** — подход, в котором стили разделяются на слои:

- **Composition** — раскладка и пространство (flex/grid-утилиты, контейнеры).
- **Utility** — мелкие одноцелевые классы.
- **Block** — компоненты в духе BEM.
- **Exception** — состояния и модификаторы, часто через `data-*` атрибуты.

```html
<div class="cluster | card" data-state="highlighted">
```

CUBE акцентирует внимание на глобальных утилитах и раскладке, а компоненты оставляет минималистичными.

#### ITCSS

**ITCSS** (Inverted Triangle CSS) — архитектурная система, которая организует стили от глобальных к локальным:

1. Settings — переменные и конфигурация.
2. Tools — миксины и функции.
3. Generic — сбросы и нормализация.
4. Elements — стили тегов.
5. Objects — раскладочные паттерны.
6. Components — компоненты.
7. Utilities — вспомогательные классы.
8. Trumps — переопределения с высшим приоритетом.

ITCSS хорошо сочетается с `@layer`: каждый уровень треугольника может быть отдельным слоем.

### `@layer` в архитектуре

CSS Layers позволяют явно управлять приоритетом стилей, не повышая специфичность. Это особенно полезно в архитектуре, где есть reset, base, components и utilities.

```css
@layer reset, base, components, utilities;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
}

@layer components {
  .button { background: blue; }
}

@layer utilities {
  .bg-red { background: red !important; }
}
```

Слои объявлены один раз, и их порядок определяет приоритет. Правила вне `@layer` имеют наивысший приоритет среди author-стилей.

### CSS Modules

**CSS Modules** — подход, при котором имена классов автоматически становятся локальными для файла. Это предотвращает глобальные конфликты без необходимости вручную писать BEM.

```css
/* Button.module.css */
.button { }
.primary { }
```

```js
import styles from './Button.module.css';

function Button() {
  return <button className={styles.button + ' ' + styles.primary}>Save</button>;
}
```

На выходе классы превращаются в уникальные хэши: `Button_button__3x7a2`. CSS Modules хорошо работают с custom properties, но сами по себе не решают задачу токенизации.

### CSS-in-JS

**CSS-in-JS** — семейство решений, при которых стили пишутся внутри JavaScript и часто генерируются динамически. Примеры: styled-components, Emotion, Linaria, Vanilla Extract.

Преимущества:

- Инкапсуляция стилей на уровне компонента.
- Доступ к runtime-данным и props для динамических стилей.
- Удаление неиспользуемого CSS.

Недостатки:

- Дополнительный runtime (кроме zero-runtime решений).
- Сложности с SSR и кешированием.
- Потенциальная потеря производительности при большом количестве динамических правил.

Современный тренд — возврат к «CSS-в-CSS» с использованием CSS Modules, `@layer`, custom properties и zero-runtime инструментов вроде Vanilla Extract или Lightning CSS.

### Композиция подходов

На практике редко используется что-то одно. Типичный современный стек:

- Custom properties для токенов и темизации.
- `@layer` для управления каскадом.
- BEM или CSS Modules для именования компонентных классов.
- Утилитарные классы для раскладки и мелких переопределений.

## Практические примеры

### Пример 1: базовая система токенов

```css
:root {
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-surface: #ffffff;
  --color-text: #111827;

  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 1rem;
  --space-4: 1.5rem;

  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;

  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
}
```

### Пример 2: темизация компонента через локальные переменные

```html
<div class="theme-dark">
  <button class="button">Dark button</button>
</div>
```

```css
.button {
  --button-bg: var(--color-primary);
  --button-color: white;

  background: var(--button-bg);
  color: var(--button-color);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-md);
}

.theme-dark .button {
  --button-bg: #60a5fa;
  --button-color: #0f172a;
}
```

Компонент не знает о тёмной теме напрямую; он просто использует свои локальные переменные, которые переопределяются в контексте.

### Пример 3: адаптивная типографика через custom properties

```css
:root {
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
}

@media (min-width: 768px) {
  :root {
    --text-base: 1.125rem;
    --text-lg: 1.25rem;
    --text-xl: 1.5rem;
  }
}

body {
  font-size: var(--text-base);
}
```

### Пример 4: CUBE-подход

```html
<div class="stack | card">
  <h2 class="card__title">Title</h2>
  <p class="card__text">Description</p>
</div>
```

```css
.stack {
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
}

.card {
  padding: var(--space-4);
  background: var(--color-surface);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-md);
}
```

`.stack` отвечает за раскладку, `.card` — за внешний вид. Вертикальная черта в классах — соглашение CUBE для разделения слоёв.

### Пример 5: CSS Modules + custom properties

```css
/* Button.module.css */
.button {
  background: var(--color-primary);
  color: var(--color-text-inverted);
  padding: var(--space-2) var(--space-3);
}

.primary {
  background: var(--color-primary);
}

.secondary {
  background: var(--color-secondary);
}
```

```jsx
import styles from './Button.module.css';

export function Button({ variant = 'primary', children }) {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      {children}
    </button>
  );
}
```

### Пример 6: `@layer` с ITCSS

```css
@layer settings, generic, elements, objects, components, utilities;

@layer settings {
  :root {
    --color-primary: #3b82f6;
  }
}

@layer generic {
  *, *::before, *::after { box-sizing: border-box; }
}

@layer components {
  .button { background: var(--color-primary); }
}

@layer utilities {
  .hidden { display: none !important; }
}
```

## Типичные ошибки и антипаттерны

- **Хранение в custom properties значений, которые нельзя интерполировать или вычислить.** Например, `--radius: 4` без единицы требует `calc(var(--radius) * 1px)`.
- **Путаница наследования и каскада.** Custom properties наследуются, но не каскадируют как обычные свойства: переопределение в дочернем элементе не влияет на родителя.
- **Смешение примитивных и семантических токенов в одном слое.** Примитивы должны быть стабильными, семантические — адаптироваться под тему.
- **Переусложнённый BEM:** `header__nav__list__item__link`. Вложенность элементов в BEM не отражается в имени; достаточно `header__link`.
- **CSS-in-JS для статичных проектов без необходимости в runtime-стилях.** Zero-runtime решения или CSS Modules часто предпочтительнее.
- **Игнорирование `@layer` в больших системах.** Без слоёв утилиты и компоненты начинают бороться через `!important` и высокую специфичность.
- **Дублирование токенов в JS и CSS.** Если цвета хранятся и в теме styled-components, и в CSS-переменных, появляется риск рассинхронизации.
- **Использование custom properties как замены всему препроцессору.** Sass/Less всё ещё удобны для циклов, математики сложнее `calc()`, и миксинов.

## Ключевые тезисы для интервью

- Custom properties живут в runtime браузера, наследуются по DOM и поддерживают области видимости через селекторы.
- `var(--name, fallback)` задаёт резервное значение, но оно не сработает, если переменная объявлена с невалидным для свойства значением.
- Для вычислений с custom properties используется `calc()`; сами переменные — строки.
- Токены делят на примитивные, семантические и компонентные уровни.
- BEM — Block__Element--Modifier; CUBE — Composition, Utility, Block, Exception; ITCSS — иерархия от глобального к локальному.
- `@layer` позволяет управлять каскадом без повышения специфичности; порядок объявления слоёв определяет приоритет.
- CSS Modules дают локальную инкапсуляцию классов через хэширование имён.
- CSS-in-JS удобен для динамических стилей, но может иметь runtime-накладные расходы; zero-runtime решения возвращают производительность CSS.
- Современный стек обычно комбинирует custom properties, `@layer`, CSS Modules/BEM и утилитарные классы.

## Полезные ссылки

- [Using CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
- [var()](https://developer.mozilla.org/en-US/docs/Web/CSS/var)
- [CSS Cascade Layers](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)
- [BEM Methodology](https://en.bem.info/methodology/)
- [CUBE CSS](https://cube.fyi/)
- [ITCSS](https://itcss.io/)
- [CSS Modules](https://github.com/css-modules/css-modules)
- [styled-components](https://styled-components.com/)
- [Vanilla Extract](https://vanilla-extract.style/)
- [Design Tokens Community Group](https://www.w3.org/community/designtokens/)
