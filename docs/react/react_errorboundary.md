### Error Boundary

Перехватывает ошибки рендеринга и показывает fallback UI.

### Нативный способ — только через класс

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    logError(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) return <FallbackUI />;
    return this.props.children;
  }
}
```

Хуки не могут заменить класс — они выполняются внутри рендера, и если рендер падает, хук не успевает перехватить ошибку.

### Библиотека `react-error-boundary`

Готовый компонент + хук `useErrorBoundary`:

```jsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onError={(error, info) => {
    Sentry.captureException(error, { extra: info });
  }}
>
  <MyComponent />
</ErrorBoundary>
```

### Что ловит / не ловит

| Ловит | Не ловит |
|-------|----------|
| Ошибки рендеринга | Обработчики событий (`onClick`) |
| Ошибки `useLayoutEffect` | Асинхронный код (`setTimeout`, `Promise`) |
| | SSR |
| | Ошибки в самом boundary |

### Обходные пути для event handlers и async

**Через `useErrorBoundary`:**

```jsx
const { showBoundary } = useErrorBoundary();

try { await fetchData(); } catch (e) { showBoundary(e); }
```

**Через `useTransition` (React 19):**

```jsx
const [isPending, startTransition] = useTransition();

startTransition(async () => {
  await addComment(); // ошибки автоматически ловятся boundary
});
```

### Практические паттерны

**Вложенные boundaries для изоляции:**

```jsx
<ErrorBoundary FallbackComponent={AppError}>
  <Header />
  <ErrorBoundary FallbackComponent={WidgetError}>
    <ExpensiveWidget />
  </ErrorBoundary>
</ErrorBoundary>
```

**Сброс по изменению ключей:**

```jsx
<ErrorBoundary FallbackComponent={ErrorFallback} resetKeys={[userId]}>
  <UserProfile userId={userId} />
</ErrorBoundary>
```
