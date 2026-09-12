# Способы применения CSS в клиентских приложениях

На современном фронтенде стили можно писать десятком способов — от архаичного `style={}` до zero-runtime компиляции. Каждый подход решает свою задачу: инкапсуляцию, динамичность, производительность, DX. На собеседовании важно не просто перечислить варианты, а объяснить trade-offs: что каждый подход даёт и что отнимает, как влияет на бандл, SSR и поддерживаемость.

## Глубокий разбор

### 1. Inline styles

Стили напрямую в атрибуте `style` элемента.

```jsx
function Badge({ count }) {
  return (
    <span style={{
      backgroundColor: count > 0 ? 'red' : 'gray',
      color: 'white',
      borderRadius: 12,
      padding: '2px 8px',
      fontSize: 12,
    }}>
      {count}
    </span>
  );
}
```

**Как работает.** Браузер преобразует объект в CSS-строку и записывает в `style` элемента. Каждый рендер генерирует новый объект — React не может сравнить ссылки и обновляет DOM-атрибут.

**Плюсы:**
- Нулевая настройка — работает из коробки.
- Полный доступ к JS-переменным и props без посредников.
- Нет проблем с именами классов.

**Минусы:**
- Нельзя использовать псевдоклассы (`:hover`, `:focus`) и псевдоэлементы (`::before`).
- Нельзя использовать медиа-запросы (без JS-костылей).
- Высокая специфичность inline-атрибута — сложно переопределить.
- Нет кеширования: стили пересоздаются при каждом рендере.
- Префиксы (vendor) не добавляются автоматически.
- Нельзя анимировать через `@keyframes` (только через JS).
- Увеличение HTML-размера: стили дублируются в каждом элементе.

**Когда использовать:** Прототипы, единичные динамические значения (позиция tooltip, ширина прогресс-бара), conditional rendering простых стилей. Не подходит для системного стилирования.

---

### 2. Глобальный CSS

Классический подход: один или несколько `.css`-файлов, подключаемых через `import './styles.css'` или `<link>`.

```css
/* global.css */
.button {
  background: #3b82f6;
  color: white;
  padding: 8px 16px;
  border-radius: 6px;
}

.button:hover {
  background: #2563eb;
}
```

```jsx
import './global.css';

function Button({ children }) {
  return <button className="button">{children}</button>;
}
```

**Как работает.** CSS-файл парсится, все правила попадают в единый CSSOM. Классы глобальны — любой элемент с классом `.button` получит эти стили.

**Плюсы:**
- Максимальная простота, нет зависимостей.
- Полная мощность CSS: псевдоклассы, медиа-запросы, анимации, `@layer`.
- Кеширование браузером — файл загружается один раз.
- Минимальный размер бандла — нет runtime-кода.

**Минусы:**
- Глобальное пространство имён — конфликты классов на крупных проектах.
- Нет tree shaking: весь CSS загружается, даже неиспользуемые классы.
- Сложность отслеживания, какой компонент использует какой класс.
- Удаление класса из компонента не гарантирует, что он не используется где-то ещё.
- Порядок импортов влияет на результат.

**Когда использовать:** Малые проекты, лендинги, глобальные стили (reset, typography), утилитарные классы. На больших проектах — только в комбинации с методологией (BEM) или `@layer`.

---

### 3. CSS Modules

Файлы `.module.css`, в которых все имена классов автоматически локализуются.

```css
/* Button.module.css */
.button {
  background: #3b82f6;
  color: white;
  padding: 8px 16px;
  border-radius: 6px;
}

.primary {
  background: #3b82f6;
}

.secondary {
  background: transparent;
  border: 1px solid #3b82f6;
  color: #3b82f6;
}

.button:hover {
  background: #2563eb;
}
```

```jsx
import styles from './Button.module.css';

function Button({ variant = 'primary', children }) {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      {children}
    </button>
  );
}
```

**Как работает.** Бандлер (Vite, Webpack, Next.js) при сборке переименовывает классы в уникальные идентификаторы вида `Button_button__x7K2p`. CSS-файл остаётся обычным CSS, генерируется объект-словарь для импорта в JS.

```
Исходный CSS:  .button { background: blue; }
Скомпилированный: .Button_button__x7K2p { background: blue; }
```

**Плюсы:**
- Локальность классов — нет глобальных конфликтов.
- Нулевой runtime — на выходе обычный CSS-файл.
- Tree shaking: неиспользуемые классы удаляются бандлером.
- Полная поддержка CSS: псевдоклассы, медиа-запросы, анимации, `@layer`.
- Кеширование браузером.
- Предсказуемая специфичность — один класс = одна специфичность.
- Отличная поддержка в Next.js, Vite, Create React App.

**Минусы:**
- Динамические стили через props неудобны — нужны CSS-переменные или условные классы.
- Нет автоматической генерации стилей из props (в отличие от CSS-in-JS).
- `composes` работает, но менее мощён, чем наследование в styled-components.
- Имена классов в DevTools нечитаемы (хэши).

**Композиция классов:**

```css
/* base.module.css */
.base {
  padding: 8px 16px;
  border-radius: 6px;
  font-weight: 500;
}

/* Button.module.css */
.button {
  composes: base from './base.module.css';
  background: #3b82f6;
  color: white;
}
```

`composes` — аналог наследования: класс `.button` получит все свойства `.base` без дублирования CSS.

**Глобальные классы внутри CSS Modules:**

```css
/* Button.module.css */
.button { /* локальный */ }

:global(.active) .button { /* глобальный .active */ }
```

**Динамические стили через CSS-переменные:**

```css
/* Card.module.css */
.card {
  width: var(--card-width, 300px);
  background: var(--card-bg, white);
}
```

```jsx
function Card({ width, bg, children }) {
  return (
    <div
      className={styles.card}
      style={{ '--card-width': `${width}px`, '--card-bg': bg }}
    >
      {children}
    </div>
  );
}
```

**Когда использовать:** Основной выбор для React/Next.js проектов среднего и крупного размера. Оптимален по соотношению DX, производительности и инкапсуляции.

---

### 4. CSS-in-JS (runtime): styled-components, Emotion

Стили описываются как JavaScript-выражения, генерируются и внедряются в DOM в runtime.

#### styled-components

```jsx
import styled from 'styled-components';

const Button = styled.button`
  background: ${p => p.$primary ? '#3b82f6' : 'transparent'};
  color: ${p => p.$primary ? 'white' : '#3b82f6'};
  padding: 8px 16px;
  border-radius: 6px;
  border: 1px solid ${p => p.$primary ? 'transparent' : '#3b82f6'};

  &:hover {
    background: ${p => p.$primary ? '#2563eb' : 'rgba(59, 130, 246, 0.1)'};
  }

  @media (min-width: 768px) {
    padding: 12px 24px;
  }
`;

function App() {
  return (
    <>
      <Button $primary>Primary</Button>
      <Button>Secondary</Button>
    </>
  );
}
```

#### Emotion (object syntax)

```jsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

const buttonStyles = (primary) => css`
  background: ${primary ? '#3b82f6' : 'transparent'};
  color: ${primary ? 'white' : '#3b82f6'};
  padding: 8px 16px;
  border-radius: 6px;
`;

function Button({ primary, children }) {
  return <button css={buttonStyles(primary)}>{children}</button>;
}
```

**Как работает.** При первом рендере компонента библиотека:
1. Парсит шаблонный литерал (или объект).
2. Генерирует уникальное имя класса (хэш).
3. Создаёт `<style>` элемент в `<head>` и вставляет CSS-правило.
4. Назначает класс элементу.

Последующие рендеры с теми же props переиспользуют кешированный класс. Новые комбинации props — новые правила.

**Transient props (`$`-префикс):**

В styled-components props с `$` не попадают в DOM — это предотвращает предупреждения React о неизвестных атрибутах.

```jsx
const Box = styled.div`
  color: ${p => p.$color};
`;

<Box $color="red" color="blue" />
// $color — только для стилизации, color — DOM-атрибут
```

**Плюсы:**
- Полная мощность JS для генерации стилей: условия, циклы, функции.
- Доступ к props, теме, контексту без посредников.
- Автоматическая инкапсуляция — уникальные имена классов.
- Автоматический vendor prefixing (через stylis в Emotion/styled-components).
- Удаление мёртвого CSS — стили удаляются при unmount компонента.
- Темизация из коробки: `<ThemeProvider>`.
- TypeScript-интеграция: типизация props и темы.

**Минусы:**
- Runtime-накладные расходы: парсинг, хэширование, DOM-манипуляции при первом рендере.
- Увеличение JS-бандла: ~10-15 KB (styled-components) или ~8 KB (Emotion).
- Дублирование CSS-правил в `<style>` при множестве уникальных комбинаций props.
- Сложности с SSR: нужно извлечь стили на сервере и передать клиенту (`ServerStyleSheet`).
- Нет кеширования CSS-файла браузером — стили живут в JS.
- Проблема «specificity war» при микшировании с глобальным CSS.
- Потенциальные проблемы с Content Security Policy (inline `<style>`).

**Производительность:** На небольших проектах незаметна. На больших (100+ компонентов с динамическими стилями) — может быть ощутима при первом рендере (FCP/LCP). Библиотеки кешируют результаты, но первый проход по-прежнему дорог.

**Когда использовать:** Проекты с высокой динамичностью стилей (theming, user-customizable UI), дизайн-системы с runtime-темизацией, команды, которым важен colocation (стили рядом с компонентом в одном файле).

---

### 5. CSS-in-JS (zero-runtime): Vanilla Extract, Linaria, Panda CSS

Стили пишутся в JS/TS, но компилируются в статические CSS-файлы на этапе сборки. Runtime-кода нет.

#### Vanilla Extract

```ts
// Button.css.ts
import { style, styleVariants } from '@vanilla-extract/css';

export const button = style({
  padding: '8px 16px',
  borderRadius: '6px',
  fontWeight: 500,
});

export const variant = styleVariants({
  primary: {
    background: '#3b82f6',
    color: 'white',
  },
  secondary: {
    background: 'transparent',
    border: '1px solid #3b82f6',
    color: '#3b82f6',
  },
});
```

```tsx
// Button.tsx
import * as styles from './Button.css';

interface ButtonProps {
  variant?: keyof typeof styles.variant;
  children: React.ReactNode;
}

export function Button({ variant = 'primary', children }: ButtonProps) {
  return (
    <button className={`${styles.button} ${styles.variant[variant]}`}>
      {children}
    </button>
  );
}
```

**Как работает.** Плагин для бандлера (Vite, Webpack, Next.js) на этапе сборки:
1. Находит `.css.ts` файлы.
2. Выполняет их в Node.js (не в браузере).
3. Генерирует статические `.css` файлы с хэшированными именами классов.
4. Заменяет экспорты в JS на строковые идентификаторы классов.

На выходе — обычный CSS-файл и JS без runtime-кода для стилей.

**Динамические стили через CSS-переменные:**

```ts
// Card.css.ts
import { style } from '@vanilla-extract/css';

export const card = style({
  width: 'var(--card-width)',
  background: 'var(--card-bg)',
});
```

```tsx
function Card({ width, bg, children }) {
  return (
    <div
      className={styles.card}
      style={{ '--card-width': `${width}px`, '--card-bg': bg }}
    >
      {children}
    </div>
  );
}
```

#### Linaria

```tsx
import { styled } from 'linaria/react';
import { css } from 'linaria';

const Button = styled.button<{ $primary?: boolean }>`
  background: ${p => p.$primary ? '#3b82f6' : 'transparent'};
  color: ${p => p.$primary ? 'white' : '#3b82f6'};
  padding: 8px 16px;
  border-radius: 6px;
`;
```

Linaria анализирует шаблонные литералы на этапе сборки и извлекает статические CSS-правила. Динамические значения через props компилируются в CSS-переменные.

#### Panda CSS

```tsx
// panda.config.ts
import { defineConfig } from '@pandacss/dev';

export default defineConfig({
  theme: {
    extend: {
      tokens: {
        colors: {
          primary: { value: '#3b82f6' },
        },
      },
    },
  },
});
```

```tsx
import { css } from '../styled-system/css';

function Button({ primary, children }) {
  return (
    <button className={css({
      bg: primary ? 'primary' : 'transparent',
      color: primary ? 'white' : 'primary',
      px: 4,
      py: 2,
      rounded: 'md',
    })}>
      {children}
    </button>
  );
}
```

Panda CSS генерирует утилитарные CSS-классы на этапе сборки — подход, сочетающий DX CSS-in-JS с производительностью utility-first.

**Плюсы:**
- Нулевой runtime — на выходе статический CSS.
- Полная типизация (TypeScript-first).
- Инкапсуляция через хэшированные имена.
- Tree shaking: неиспользуемые стили не попадают в бандл.
- Кеширование CSS-файла браузером.
- Нет проблем с SSR — обычный CSS.
- Нет проблем с CSP — нет inline `<style>`.

**Минусы:**
- Динамические стили ограничены CSS-переменными (нельзя генерировать новые классы в runtime).
- Более сложная настройка (плагины для бандлера).
- Меньше библиотек и экосистемы по сравнению с runtime-решениями.
- Долгий initial build из-за компиляции.

**Когда использовать:** Когда нужен DX CSS-in-JS, но критична производительность. Оптимальный выбор для новых проектов на Next.js/Vite.

---

### 6. Utility-first CSS: Tailwind CSS

Стили собираются из атомарных утилитарных классов прямо в JSX.

```jsx
function Card({ title, description }) {
  return (
    <div className="rounded-lg bg-white p-6 shadow-md hover:shadow-lg transition-shadow">
      <h2 className="text-xl font-bold text-gray-900">{title}</h2>
      <p className="mt-2 text-gray-600">{description}</p>
    </div>
  );
}
```

**Как работает.** Tailwind сканирует исходники, находит используемые утилиты и генерирует минимальный CSS-файл только с нужными классами. На этапе сборки — не runtime.

```css
/* Сгенерированный output.css */
.rounded-lg { border-radius: 0.5rem; }
.bg-white { background-color: #fff; }
.p-6 { padding: 1.5rem; }
.shadow-md { box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1); }
/* ... только используемые классы */
```

**Плюсы:**
- Минимальный CSS-бандл — только используемые утилиты.
- Нет придумывания имён классов.
- Быстрый прототипинг — стили не покидают JSX.
- Консистентность: ограничения дизайн-системы в конфиге (цвета, отступы, шрифты).
- Responsive и state-модификаторы прямо в классе: `md:flex`, `hover:bg-blue-700`.
- Тёмная тема: `dark:bg-gray-900`.
- Нет runtime — чистый CSS на выходе.

**Минусы:**
- Длинные className — ухудшение читаемости JSX.
- Нельзя выразить сложные стили (многослойные тени, кастомные анимации) без `@apply` или arbitrary values.
- CSS-файл всё ещё глобален — нет инкапсуляции на уровне компонента.
- Переключение контекста между JSX и CSS-мышлением.
- Legacy-код с Tailwind тяжело рефакторить.

**Arbitrary values:**

```jsx
<div className="w-[350px] bg-[#1da1f2] grid-cols-[repeat(3,minmax(0,1fr))]">
```

**Компонентный подход с `cva` (class-variance-authority):**

```tsx
import { cva } from 'class-variance-authority';
import { cn } from './utils';

const button = cva(
  'rounded-md font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: 'bg-blue-500 text-white hover:bg-blue-600',
        secondary: 'border border-blue-500 text-blue-500 hover:bg-blue-50',
        ghost: 'text-blue-500 hover:bg-blue-50',
      },
      size: {
        sm: 'px-3 py-1.5 text-sm',
        md: 'px-4 py-2 text-base',
        lg: 'px-6 py-3 text-lg',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

function Button({ variant, size, className, children }) {
  return (
    <button className={cn(button({ variant, size }), className)}>
      {children}
    </button>
  );
}
```

**Когда использовать:** Команды, ценящие скорость разработки и консистентность; дизайн-системы с чёткими токенами; проекты, где важна минимальность CSS-бандла.

---

### 7. CSS `@scope`

Новый CSS-механизм для ограничения области видимости стилей без CSS Modules.

```css
@scope (.card) to (.card__content) {
  :scope {
    padding: 16px;
    background: white;
  }

  a {
    color: blue;
  }

  img {
    border-radius: 8px;
  }
}
```

**Как работает.** `@scope` определяет корень (`.card`) и границу (`.card__content`). Стили применяются только к элементам внутри этой области, не выходя за границу.

**Плюсы:**
- Нативный CSS — нет зависимостей и сборки.
- Инкапсуляция без хэширования имён.
- Можно применять к существующему CSS без миграции.

**Минусы:**
- Ограниченная поддержка браузеров (на 2026 — Chromium, частично Firefox).
- Не решает проблему именования — классы всё ещё глобальны вне scope.
- Не интегрируется с JS-модулями напрямую.

**Когда использовать:** Прогрессивное улучшение, виджеты на чистом HTML/CSS, изоляция legacy-стилей.

---

### 8. Shadow DOM и стили в Web Components

Shadow DOM обеспечивает полную инкапсуляцию: стили внутри не влияют на внешний мир и наоборот.

```js
class MyButton extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        button {
          background: #3b82f6;
          color: white;
          padding: 8px 16px;
          border-radius: 6px;
        }
        button:hover {
          background: #2563eb;
        }
      </style>
      <button><slot></slot></button>
    `;
  }
}

customElements.define('my-button', MyButton);
```

**Плюсы:**
- Полная изоляция — стили не протекают ни в одну сторону.
- Идеально для переиспользуемых виджетов.
- Нативный браузерный механизм.

**Минусы:**
- Нельзя стилизовать извне (кроме custom properties, которые проникают через Shadow DOM).
- Сложности с глобальными темами и дизайн-системами.
- Не все CSS-фреймворки работают с Shadow DOM.

**Когда использовать:** Web Components, виджеты для сторонних сайтов, микрофронтенды.

---

## Сравнительная таблица

| Подход | Runtime | Инкапсуляция | Динамические стили | CSS-бандл | SSR | Типизация |
|---|---|---|---|---|---|---|
| Inline | Нет | Полная | Да | Нет CSS | Да | Частичная |
| Глобальный CSS | Нет | Нет | Через CSS-переменные | Полный | Да | Нет |
| CSS Modules | Нет | Локальные классы | Через CSS-переменные | Tree-shaken | Да | Через `.d.ts` |
| CSS-in-JS (runtime) | Да | Хэш-классы | Полная | Нет CSS | Сложнее | Да |
| CSS-in-JS (zero-runtime) | Нет | Хэш-классы | CSS-переменные | Tree-shaken | Да | Да |
| Tailwind CSS | Нет | Нет (утилиты) | Через arbitrary values | Минимальный | Да | Через плагины |
| `@scope` | Нет | Через scope | Через CSS-переменные | Полный | Да | Нет |
| Shadow DOM | Нет | Полная | Да | Нет CSS | Да | Нет |

---

## Тренд 2025–2026

Индустрия движется от runtime CSS-in-JS к **compile-time решениям**. Причины:

1. **Performance.** Runtime CSS-in-JS добавляет задержку к FCP/LCP. На мобильных устройствах с низким CPU это заметно.
2. **React Server Components.** Серверные компоненты не могут использовать runtime CSS-in-JS — нет JS-рантайма на клиенте для генерации стилей. Zero-runtime решения и CSS Modules работают с RSC.
3. **Bundle size.** Каждый KB JS-бандла — это время загрузки и парсинга. CSS-файл кешируется отдельно и не блокирует JS-выполнение.
4. **CSP.** Политики безопасности всё чаще запрещают inline `<style>`, что ломает runtime CSS-in-JS.

**Рекомендуемый стек для нового React/Next.js проекта:**

- **CSS Modules** — для компонентных стилей (простота, нулевой runtime).
- **CSS-переменные** — для токенов и темизации.
- **`@layer`** — для управления каскадом.
- **Tailwind CSS** — для утилитарных стилей и быстрого прототипирования.
- **Vanilla Extract / Panda CSS** — если нужен типизированный CSS-in-JS DX без runtime.

---

## Практические примеры

### Пример 1: CSS Modules + темизация через CSS-переменные

```css
/* Button.module.css */
.button {
  --btn-bg: var(--color-primary);
  --btn-color: var(--color-text-inverted);
  --btn-border: transparent;

  background: var(--btn-bg);
  color: var(--btn-color);
  border: 1px solid var(--btn-border);
  padding: var(--space-2) var(--space-4);
  border-radius: var(--radius-md);
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s ease;
}

.button:hover {
  --btn-bg: var(--color-primary-hover);
}

.outline {
  --btn-bg: transparent;
  --btn-color: var(--color-primary);
  --btn-border: var(--color-primary);
}

.ghost {
  --btn-bg: transparent;
  --btn-color: var(--color-primary);
  --btn-border: transparent;
}

.ghost:hover {
  --btn-bg: var(--color-primary-alpha-10);
}

.disabled {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
}
```

```tsx
import styles from './Button.module.css';
import { cn } from './utils';

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'solid' | 'outline' | 'ghost';
}

export function Button({ variant = 'solid', className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        styles.button,
        variant === 'outline' && styles.outline,
        variant === 'ghost' && styles.ghost,
        props.disabled && styles.disabled,
        className
      )}
      {...props}
    />
  );
}
```

### Пример 2: Vanilla Extract с рецептами (sprinkles)

```ts
// sprinkles.css.ts
import { defineProperties, createSprinkles } from '@vanilla-extract/sprinkles';

const responsiveProperties = defineProperties({
  conditions: {
    mobile: {},
    tablet: { '@media': '(min-width: 768px)' },
    desktop: { '@media': '(min-width: 1024px)' },
  },
  defaultCondition: 'mobile',
  properties: {
    display: ['none', 'flex', 'block', 'grid'],
    flexDirection: ['row', 'column'],
    padding: { small: '8px', medium: '16px', large: '24px' },
    gap: { small: '8px', medium: '16px', large: '24px' },
  },
  shorthands: {
    p: ['padding'],
    fd: ['flexDirection'],
  },
});

const colorProperties = defineProperties({
  properties: {
    background: {
      primary: '#3b82f6',
      surface: '#ffffff',
      danger: '#ef4444',
    },
    color: {
      default: '#111827',
      inverted: '#ffffff',
      muted: '#6b7280',
    },
  },
});

export const sprinkles = createSprinkles(responsiveProperties, colorProperties);
```

```tsx
import { sprinkles } from './sprinkles.css';

function Card() {
  return (
    <div className={sprinkles({
      display: 'flex',
      fd: 'column',
      p: 'medium',
      background: 'surface',
      color: 'default',
      gap: { mobile: 'small', desktop: 'medium' },
    })}>
      <h2>Card title</h2>
      <p>Card content</p>
    </div>
  );
}
```

### Пример 3: Tailwind + cva — компонентный подход

```tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from './utils';

const badge = cva(
  'inline-flex items-center rounded-full font-medium',
  {
    variants: {
      variant: {
        success: 'bg-green-100 text-green-800',
        warning: 'bg-yellow-100 text-yellow-800',
        error: 'bg-red-100 text-red-800',
        info: 'bg-blue-100 text-blue-800',
      },
      size: {
        sm: 'px-2 py-0.5 text-xs',
        md: 'px-2.5 py-0.5 text-sm',
        lg: 'px-3 py-1 text-base',
      },
    },
    defaultVariants: {
      variant: 'info',
      size: 'md',
    },
  }
);

type BadgeProps = VariantProps<typeof badge> & {
  children: React.ReactNode;
};

function Badge({ variant, size, children }: BadgeProps) {
  return <span className={badge({ variant, size })}>{children}</span>;
}
```

### Пример 4: Миграция с runtime CSS-in-JS на CSS Modules

**До (styled-components):**

```tsx
const Card = styled.div<{ $highlighted?: boolean }>`
  padding: 16px;
  border-radius: 8px;
  background: ${p => p.$highlighted ? '#eff6ff' : '#ffffff'};
  border: 1px solid ${p => p.$highlighted ? '#3b82f6' : '#e5e7eb'};
  box-shadow: 0 1px 3px rgb(0 0 0 / 0.1);
`;
```

**После (CSS Modules):**

```css
/* Card.module.css */
.card {
  padding: 16px;
  border-radius: 8px;
  background: var(--card-bg, #ffffff);
  border: 1px solid var(--card-border, #e5e7eb);
  box-shadow: 0 1px 3px rgb(0 0 0 / 0.1);
}

.highlighted {
  --card-bg: #eff6ff;
  --card-border: #3b82f6;
}
```

```tsx
import styles from './Card.module.css';
import { cn } from './utils';

function Card({ highlighted, children }) {
  return (
    <div className={cn(styles.card, highlighted && styles.highlighted)}>
      {children}
    </div>
  );
}
```

---

## Типичные ошибки и антипаттерны

- **Inline styles для всего.** Потеря псевдоклассов, медиа-запросов, кеширования. Inline — только для truly динамических значений.
- **Смешение подходов без стратегии.** CSS Modules + styled-components + Tailwind в одном проекте — хаос. Выберите один основной подход.
- **Динамические классы через интерполяцию в CSS Modules.** `styles[\`btn-\${variant}\`]` — хрупкий и непроверяемый. Используйте явные маппинги.
- **Глобальный CSS без `@layer` или методологии.** На проекте с 50+ компонентами без системы именования — неизбежные конфликты.
- **Runtime CSS-in-JS для SSR-проектов без настройки.** Забытый `ServerStyleSheet` = стили не попадают в первый HTML.
- **Tailwind без `cva`/`tv`.** Длинный `className` с условной логикой в JSX — нечитаем. Вынесите варианты в `cva`.
- **Хранение стилей в JS-объектах.** `{ color: 'red' }` вместо CSS — потеря кеширования, нет псевдоклассов, нет DevTools.
- **Игнорирование CSS-переменных для динамических значений.** Вместо `style={{ width: value }}` — `style={{ '--width': value }}` + `width: var(--width)` в CSS.

---

## Ключевые тезисы для интервью

- Inline styles: полный доступ к JS, но нет псевдоклассов, медиа-запросов и кеширования. Только для единичных динамических значений.
- Глобальный CSS: прост и быстр, но требует дисциплины (BEM, `@layer`) на крупных проектах.
- CSS Modules: локальные классы через хэширование, нулевой runtime, полная поддержка CSS. Оптимальный выбор для React/Next.js.
- Runtime CSS-in-JS (styled-components, Emotion): максимальная гибкость и DX, но runtime-накладные расходы, увеличение JS-бандла, сложности с SSR и CSP.
- Zero-runtime CSS-in-JS (Vanilla Extract, Panda CSS): DX CSS-in-JS + производительность чистого CSS. Компиляция на этапе сборки.
- Tailwind CSS: utility-first, минимальный CSS-бандл, быстрота разработки. Комбинируется с `cva` для компонентного подхода.
- `@scope`: нативная инкапсуляция CSS без инструментов сборки. Ограниченная поддержка браузеров.
- Shadow DOM: полная изоляция стилей для Web Components. Custom properties проникают через границу.
- Тренд 2025–2026: переход от runtime CSS-in-JS к compile-time решениям из-за RSC, performance и CSP.
- Рекомендуемый стек: CSS Modules + CSS-переменные + `@layer` + Tailwind (или Vanilla Extract для типизированного DX).

---

## Полезные ссылки

- [CSS Modules spec](https://github.com/css-modules/css-modules)
- [styled-components](https://styled-components.com/)
- [Emotion](https://emotion.sh/)
- [Vanilla Extract](https://vanilla-extract.style/)
- [Linaria](https://linaria.dev/)
- [Panda CSS](https://panda-css.com/)
- [Tailwind CSS](https://tailwindcss.com/)
- [class-variance-authority](https://cva.style/)
- [CSS @scope](https://developer.mozilla.org/en-US/docs/Web/CSS/@scope)
- [Shadow DOM and styling](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM)
- [Sprinkles (Vanilla Extract)](https://github.com/vanilla-extract-css/vanilla-extract/tree/master/packages/sprinkles)
- [CSS-in-JS performance comparison](https://github.com/nicolo-ribaudo/css-in-js-perf)
