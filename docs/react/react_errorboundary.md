# Error Boundary в React

Error Boundary — это механизм перехвата ошибок JavaScript в любом месте дерева компонентов, журналирования этих ошибок и отображения fallback UI вместо упавшего дерева.

## Зачем нужны Error Boundaries

В React ошибка в рендере одного компонента по умолчанию ломает всё приложение — пользователь видит белый экран. Error Boundary изолирует сбой: если упал виджет, падает только виджет, а остальная страница продолжает работать.

Типичные сценарии:

- Защита всего приложения от падения из-за одного «багованного» компонента.
- Изоляция независимых UI-блоков (sidebar, карточки товаров, виджеты).
- Логирование ошибок в Sentry / LogRocket / собственный сервис.

## Нативный способ — только через класс

Error Boundary реализуется как классовый компонент с двумя методами жизненного цикла: `static getDerivedStateFromError` и `componentDidCatch`.

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    // Обновляем state, чтобы следующий рендер показал fallback UI
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    // Логируем ошибку
    logError(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return <FallbackUI error={this.state.error} />;
    }

    return this.props.children;
  }
}
```

### Почему не хук

Хуки выполняются внутри рендера. Если рендер падает, хук не успевает перехватить ошибку — выполнение прерывается раньше. Поэтому Error Boundary остаётся классовым компонентом даже в эпоху хуков.

## Библиотека `react-error-boundary`

Готовый компонент `ErrorBoundary` и хук `useErrorBoundary` упрощают работу:

```jsx
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <p>Что-то пошло не так:</p>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Попробовать снова</button>
    </div>
  );
}

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onError={(error, info) => {
    Sentry.captureException(error, { extra: info });
  }}
  onReset={() => {
    // Очистить состояние, перезапросить данные и т.д.
  }}
>
  <MyComponent />
</ErrorBoundary>
```

### Сброс ошибки

`resetErrorBoundary` позволяет попытаться перерендерить упавшее дерево. Часто его вызывают при изменении ключевых пропсов или маршрута:

```jsx
<ErrorBoundary
  FallbackComponent={ErrorFallback}
  resetKeys={[userId]}
>
  <UserProfile userId={userId} />
</ErrorBoundary>
```

При изменении `userId` boundary автоматически сбросится и попробует отрендерить компонент снова.

## Что ловит / не ловит

| Ловит | Не ловит |
|-------|----------|
| Ошибки рендеринга | Обработчики событий (`onClick`) |
| Ошибки `useLayoutEffect` | Асинхронный код (`setTimeout`, `Promise`) |
| Ошибки в конструкторе дочерних компонентов | SSR |
| | Ошибки внутри самого Error Boundary |
| | Ошибки в обработчиках ошибок (например, в `componentDidCatch`) |

## Обходные пути для event handlers и async

### Через `useErrorBoundary`

```jsx
const { showBoundary } = useErrorBoundary();

try {
  await fetchData();
} catch (e) {
  showBoundary(e);
}
```

### Через `useTransition` (React 19)

```jsx
const [isPending, startTransition] = useTransition();

startTransition(async () => {
  await addComment(); // ошибки автоматически ловятся ближайшим boundary
});
```

## Практические паттерны

### Вложенные boundaries для изоляции

```jsx
<ErrorBoundary FallbackComponent={AppError}>
  <Header />
  <ErrorBoundary FallbackComponent={WidgetError}>
    <ExpensiveWidget />
  </ErrorBoundary>
  <ErrorBoundary FallbackComponent={WidgetError}>
    <ReviewsWidget />
  </ErrorBoundary>
</ErrorBoundary>
```

### Boundary + Suspense

```jsx
<ErrorBoundary FallbackComponent={DataError}>
  <Suspense fallback={<Spinner />}>
    <UserData />
  </Suspense>
</ErrorBoundary>
```

`Suspense` обрабатывает состояние загрузки, а Error Boundary — состояние ошибки.

## TypeScript

```tsx
import { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(_: Error): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    console.error('ErrorBoundary caught:', error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }

    return this.props.children;
  }
}
```

## Лучшие практики

1. **Всегда имейте корневой boundary** — защита от белого экрана.
2. **Используйте вложенные boundaries** — изолируйте независимые части UI.
3. **Логируйте ошибки** — без логирования boundary бесполезен.
4. **Давайте пользователю кнопку «Повторить»** — не оставляйте его один на один с ошибкой.
5. **Не злоупотребляйте** — boundary не заменяет валидацию и обработку ошибок API.
6. **Тестируйте** — убедитесь, что fallback UI действительно отображается.

## Итог

Error Boundary — обязательный инструмент для надёжного React-приложения. Он не ловит всё, но там, где работает, спасает пользовательский опыт и помогает команде быстрее находить проблемы.
