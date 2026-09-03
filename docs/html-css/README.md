# HTML, CSS и браузерный фундамент

> **Статус:** в разработке. Статьи добавляются постепенно — по 1–2 за сессию.

Этот раздел не заменяет учебник для новичков. Он предполагает, что базовый синтаксис, селекторы, блочная модель и базовые вещи вроде разницы между `div` и `span` уже известны. Здесь собраны темы, которые реально спрашивают на Middle+/Senior-собеседованиях: как браузер строит страницу, почему CSS ведёт себя именно так, современные механизмы раскладки и доступность.

## Принципы раздела

- **Без очевидностей.** Не объясняем, что такое HTML, как писать классы или что делает `display: flex`.
- **Глубина важнее ширины.** Лучше разобрать механизм до конца, чем перечислить 20 свойств.
- **Связь с интервью.** Каждая статья заканчивается вопросами, которые могут задать на собеседовании.
- **Практика.** Примеры кода, типичные ошибки и антипаттерны.

## Карта статей

| # | Файл | Тема | Статус |
|---|------|------|--------|
| 1 | `html-rendering-pipeline.md` | Парсинг HTML, критический путь рендеринга, DOM/CSSOM/Render Tree, Layout/Paint/Composite | 🚧 запланировано |
| 2 | `html-semantics-accessibility.md` | Семантический HTML, landmarks, default ARIA roles, доступность | 🚧 запланировано |
| 3 | `html-forms-validation.md` | Формы, Constraint Validation API, `ElementInternals`, custom elements | 🚧 запланировано |
| 4 | `html-web-components.md` | Custom elements, Shadow DOM, slots, lifecycle callbacks | 🚧 запланировано |
| 5 | `css-cascade-specificity.md` | Cascade, origin, `@layer`, specificity, inheritance, `!important` | 🚧 запланировано |
| 6 | `css-layout-formatting-contexts.md` | BFC/IFC/FFC/GFC, containing block, margin collapse, box-sizing | 🚧 запланировано |
| 7 | `css-flexbox.md` | Flexbox в глубину: оси, `flex-basis`, grow/shrink, alignment | 🚧 запланировано |
| 8 | `css-grid.md` | Grid: explicit/implicit, `fr`, `minmax`, `auto-fit`/`auto-fill`, subgrid | 🚧 запланировано |
| 9 | `css-positioning-stacking.md` | Positioning, stacking context, z-index, paint order | 🚧 запланировано |
| 10 | `css-responsive-container-queries.md` | Media queries, container queries, viewport units, `prefers-*` | 🚧 запланировано |
| 11 | `css-animations-performance.md` | Transitions/animations, composite-only свойства, `will-change`, `contain` | 🚧 запланировано |
| 12 | `css-variables-architecture.md` | Custom properties, theming, BEM/CUBE/layers, CSS Modules vs CSS-in-JS | 🚧 запланировано |
| 13 | `css-modern-selectors.md` | `:is`/`:where`/`:has`/`:not`, nesting, logical properties, color spaces | 🚧 запланировано |

## Рекомендуемый порядок написания

Необязательно идти строго по номерам, но логично двигаться от фундамента к специфике:

### Волна 1 — Core
1. `html-rendering-pipeline.md`
2. `css-cascade-specificity.md`
3. `css-layout-formatting-contexts.md`
4. `css-flexbox.md`
5. `css-grid.md`
6. `css-positioning-stacking.md`

### Волна 2 — Accessibility & Forms
7. `html-semantics-accessibility.md`
8. `html-forms-validation.md`

### Волна 3 — Advanced / Modern CSS
9. `html-web-components.md`
10. `css-responsive-container-queries.md`
11. `css-animations-performance.md`
12. `css-variables-architecture.md`
13. `css-modern-selectors.md`

## Чеклист вопросов

Полная версия чеклиста живёт в корневом [README.md](../README.md#2-html-css-и-браузерный-фундамент). Кратко:

- Как браузер парсит HTML и что такое preload scanner?
- Что такое критический путь рендеринга?
- Как семантика влияет на accessibility и SEO?
- Как работает Constraint Validation API?
- Что такое Shadow DOM и slots?
- Как устроен cascade и что такое `@layer`?
- Что создаёт новый stacking context?
- Почему схлопываются вертикальные margin’ы?
- Что такое containing block?
- `flex-basis` vs `width` — в чём разница?
- `auto-fit` vs `auto-fill` в Grid?
- Как работают container queries?
- Какие свойства безопасны для анимации с точки зрения производительности?
- Как работают CSS custom properties и их область видимости?

## Шаблон статьи

Чтобы все статьи были в едином стиле, используй следующую структуру:

```markdown
# Название темы

Краткое вступление в 2–3 предложения: зачем эта тема и что читатель вынесет.

## Вопросы для самопроверки

Перед чтением попробуй ответить:

- Вопрос 1?
- Вопрос 2?

## Глубокий разбор

Основное содержание. Разбивай на подзаголовки H2/H3.

## Практические примеры

```html
<!-- HTML -->
```

```css
/* CSS */
```

## Типичные ошибки и антипаттерны

- Ошибка 1: почему плохо и как правильно.
- Ошибка 2.

## Ключевые тезисы для интервью

- Короткий тезис 1.
- Короткий тезис 2.
- Короткий тезис 3.

## Полезные ссылки

- [Ссылка 1](url)
- [Ссылка 2](url)
```

## Правила оформления

- Код в блоках ` ```html `, ` ```css `, ` ```js `.
- Свойства CSS пиши в нижнем регистре.
- Термины на английском оставляй как есть (`stacking context`, `containing block`), но рядом давай русский перевод при первом упоминании.
- Каждая статья должна заканчиваться блоком «Ключевые тезисы для интервью».
