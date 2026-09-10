# HTML и CSS

Раздел углубляет понимание браузерного фундамента: как HTML и CSS работают под капотом, как строится страница, как устроены раскладки и современные механизмы стилизации. Материалы ориентированы на Middle+/Senior-собеседования и уверенную разработку.

## Начни здесь

Если хочешь закрыть пробелы в фундаменте — иди по порядку:

1. **[Парсинг HTML и критический путь рендеринга](./html-rendering-pipeline.md)** — DOM/CSSOM/Render Tree, Layout/Paint/Composite, preload scanner.
2. **[Каскад и специфичность](./css-cascade-specificity.md)** — cascade, origin, `@layer`, specificity, inheritance, `!important`.
3. **[Formatting контексты и блочная модель](./css-layout-formatting-contexts.md)** — BFC/IFC/FFC/GFC, containing block, margin collapse, box-sizing.
4. **[Flexbox](./css-flexbox.md)** — оси, `flex-basis`, grow/shrink, alignment.
5. **[Grid](./css-grid.md)** — explicit/implicit, `fr`, `minmax`, `auto-fit`/`auto-fill`, subgrid.
6. **[Позиционирование и stacking context](./css-positioning-stacking.md)** — positioning, stacking context, z-index, paint order.

## Продвинутые темы

После базы переходи к глубоким и прикладным темам:

- **[Семантический HTML и доступность](./html-semantics-accessibility.md)** — landmarks, default ARIA roles, доступность.
- **[Формы и валидация](./html-forms-validation.md)** — Constraint Validation API, `ElementInternals`, custom elements.
- **[Web Components](./html-web-components.md)** — custom elements, Shadow DOM, slots, lifecycle callbacks.
- **[Адаптивность и container queries](./css-responsive-container-queries.md)** — media queries, container queries, viewport units, `prefers-*`.
- **[Анимации и производительность](./css-animations-performance.md)** — transitions/animations, composite-only свойства, `will-change`, `contain`.
- **[CSS-переменные и архитектура](./css-variables-architecture.md)** — custom properties, theming, BEM/CUBE/layers, CSS Modules vs CSS-in-JS.

## Специализированные темы

- **[Современные селекторы и возможности CSS](./css-modern-selectors.md)** — `:is`/`:where`/`:has`/`:not`, nesting, logical properties, color spaces.

## Как пользоваться

1. Пройди раздел «Начни здесь» по порядку — темы логически связаны.
2. Изучи семантику, формы и доступность параллельно с практикой вёрстки.
3. Переходи к Web Components, container queries и анимациям, когда база усвоена.
4. Возвращайся к каскаду, специфичности и stacking context перед собеседованиями.
