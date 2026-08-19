# React

Раздел охватывает всё, что нужно для уверенной работы с React: фазы рендеринга, хуки, события, мемоизацию, Suspense, конкурентный режим и внутреннее устройство Fiber.

## Начни здесь

1. **[Порядок рендеринга и вызова хуков](./react_rendering_order.md)** — фазы render/commit, порядок хуков, Strict Mode.
2. **[Хуки React](./react_hooks.md)** — `useRef`, `useState`, `useReducer`, `useEffect`, `useMemo`, `useCallback`, правила хуков.
3. **[useEffect](./react_useeffect.md)** — жизненный цикл эффектов, cleanup, зависимости.
4. **[useContext](./react_usecontext.md)** — передача данных через дерево компонентов.
5. **[Синтетические события](./react_synthetic_events.md)** — event pooling, делегирование, доступ к нативным событиям.
6. **[Error Boundaries](./react_errorboundary.md)** — перехват ошибок рендеринга и fallback UI.

## Продвинутые темы

- **[Мемоизация](./react_memorization.md)** — `useMemo`, `useCallback`, `React.memo`, React Compiler.
- **[Suspense](./react_suspense.md)** — lazy loading, потоковый рендеринг, Error Boundaries.
- **[HOC](./react_hoc.md)** — композиция, проблемы, сравнение с хуками.
- **[Конкурентные хуки](./react_concurrent_hooks.md)** — `useTransition`, `useDeferredValue`, приоритеты обновлений.

## Устаревшее и специализированное

- **[defaultProps и propTypes](./react_defaultprops_proptypes.md)** — legacy-типизация и современная альтернатива.
- **[React Fiber](./react_fiber.md)** — внутреннее устройство reconciler для глубокого понимания.
- **[use()](./react_use.md)** — новый хук для Promise и Context.
- **[useImperativeHandle](./react_use_imperative_handle.md)** — управление императивным API дочерних компонентов.

## Как пользоваться

1. Начни с порядка рендеринга и основных хуков.
2. Реши реальные задачи с `useEffect`, `useContext` и событиями.
3. Переходи к мемоизации и Suspense, когда столкнёшься с производительностью.
4. Fiber и конкурентные хуки — для senior-уровня и собеседований.
