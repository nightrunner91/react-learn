# defaultProps и propTypes в React

## propTypes

`propTypes` — статическое свойство компонента для декларирования типов и валидации props. Работает только в runtime (не в compile-time).

```jsx
import PropTypes from 'prop-types';

function Button({ label, onClick, size }) {
  return <button onClick={onClick} className={size}>{label}</button>;
}

Button.propTypes = {
  label: PropTypes.string.isRequired,
  onClick: PropTypes.func,
  size: PropTypes.oneOf(['sm', 'md', 'lg']),
};
```

### Встроенные валидаторы

| Валидатор | Описание |
|-----------|----------|
| `PropTypes.string` | Строка |
| `PropTypes.number` | Число |
| `PropTypes.bool` | Булевое значение |
| `PropTypes.func` | Функция |
| `PropTypes.object` | Объект |
| `PropTypes.array` | Массив |
| `PropTypes.node` | Любой рендеримый узел |
| `PropTypes.element` | React-элемент |
| `PropTypes.instanceOf(Class)` | Экземпляр класса |
| `PropTypes.oneOf(['a', 'b'])` | Одно из значений (enum) |
| `PropTypes.oneOfType([...])` | Один из типов |
| `PropTypes.arrayOf(...)` | Массив определённого типа |
| `PropTypes.shape({...})` | Объект с конкретной формой |
| `PropTypes.exact({...})` | Объект без лишних ключей |

### `.isRequired`

Добавление `.isRequired` к любому валидатору делает prop обязательным. В консоли появится предупреждение, если prop не передан.

```jsx
Button.propTypes = {
  label: PropTypes.string.isRequired,
  onClick: PropTypes.func.isRequired,
};
```

> **Примечание:** `propTypes` игнорируется в production-сборке. Валидация работает только в development.

---

## defaultProps

`defaultProps` — статическое свойство, задающее значения по умолчанию для props, если они не были переданы.

```jsx
function Button({ label, size, disabled }) {
  return <button disabled={disabled} className={size}>{label}</button>;
}

Button.defaultProps = {
  label: 'Click me',
  size: 'md',
  disabled: false,
};
```

### Порядок разрешения props

1. Значение, переданное в JSX (`<Button label="Submit" />`)
2. Значение из `defaultProps`
3. `undefined`

### defaultProps с функциональными компонентами

```jsx
function Greeting({ name = 'World', greeting = 'Hello' }) {
  return <h1>{greeting}, {name}!</h1>;
}
```

Parameters with defaults (деструктуризация с дефолтами) — современный и рекомендуемый способ. `defaultProps` считается устаревшим подходом для функциональных компонентов.

### defaultProps с class-компонентами

```jsx
class Button extends React.Component {
  static defaultProps = {
    size: 'md',
    disabled: false,
  };

  render() {
    const { size, disabled, label } = this.props;
    return <button disabled={disabled} className={size}>{label}</button>;
  }
}
```

---

## Современная альтернатива: TypeScript

React-сообщество постепенно отказывается от `propTypes` в пользу TypeScript, который даёт compile-time проверку:

```tsx
interface ButtonProps {
  label: string;
  onClick?: () => void;
  size?: 'sm' | 'md' | 'lg';
}

function Button({ label, onClick, size = 'md' }: ButtonProps) {
  return <button onClick={onClick} className={size}>{label}</button>;
}
```

Преимущества TypeScript перед propTypes:
- Проверка на этапе компиляции, а не runtime
- Автодополнение в IDE
- Не нужен дополнительный пакет `prop-types`
- Более выразительная система типов (generics, union types, utility types)

---

## Сравнение с Vue

### Vue: `defineProps` + `defineEmits` (Composition API)

```vue
<script setup>
const props = defineProps({
  label: {
    type: String,
    required: true,
  },
  size: {
    type: String,
    default: 'md',
    validator: (value) => ['sm', 'md', 'lg'].includes(value),
  },
  disabled: {
    type: Boolean,
    default: false,
  },
});
</script>
```

### Vue: TypeScript с `defineProps`

```vue
<script setup lang="ts">
interface Props {
  label: string;
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  size: 'md',
  disabled: false,
});
</script>
```

### Таблица соответствий

| Концепция | React | Vue 3 |
|-----------|-------|-------|
| Объявление типов props | `Component.propTypes = {...}` | `defineProps({...})` или `defineProps<T>()` |
| Значения по умолчанию | `Component.defaultProps = {...}` или деструктуризация | `withDefaults()` или `defineProps` с `default` |
| Обязательные props | `PropTypes.string.isRequired` | `required: true` в options API |
| Валидация значений | `PropTypes.oneOf([...])` | `validator: (val) => ...` |
| Runtime-валидация | `prop-types` (отдельный пакет) | Встроена в Vue (dev-only) |
| Compile-time проверка | TypeScript | TypeScript (`lang="ts"`) |
| Вложенная валидация | `PropTypes.shape({...})` | Нет встроенного аналога |
| Массив определённого типа | `PropTypes.arrayOf(...)` | `type: Array` (без параметризации) |

### Ключевые различия

**1. Где объявляются**

```jsx
// React — ВНЕ компонента (или как static property)
Button.propTypes = { ... };
Button.defaultProps = { ... };
```

```vue
<!-- Vue — ВНУТРИ <script setup> -->
<script setup>
const props = defineProps({ ... });
</script>
```

**2. Объединение типов и дефолтов**

```jsx
// React — раздельно
Button.propTypes = { size: PropTypes.string };
Button.defaultProps = { size: 'md' };
// Или через деструктуризацию
function Button({ size = 'md' }) { ... }
```

```vue
<!-- Vue — вместе -->
<script setup>
defineProps({
  size: { type: String, default: 'md' }
});
</script>
```

**3. Валидация**

```jsx
// React — через prop-types (нужна установка)
import PropTypes from 'prop-types';
Button.propTypes = {
  size: PropTypes.oneOf(['sm', 'md', 'lg']),
};
```

```vue
<!-- Vue — встроено, без дополнительных пакетов -->
<script setup>
defineProps({
  size: {
    type: String,
    validator: (v) => ['sm', 'md', 'lg'].includes(v),
  },
});
</script>
```

**4. Предупреждения**

Оба фреймворка выводят предупреждения в консоль только в development-режиме. В production валидация отключена.

**5. React 19+ — defaultProps для функциональных компонентов**

Начиная с React 19, `defaultProps` для функциональных компонентов **deprecated**. Рекомендуется использовать значения по умолчанию через деструктуризацию:

```jsx
// React 19+ (рекомендуется)
function Button({ size = 'md', label = 'Click' }) { ... }

// React 19+ (deprecated для функциональных компонентов)
Button.defaultProps = { size: 'md' };
```

Для class-компонентов `defaultProps` остаётся поддерживаемым.

---

## Итог

| Аспект | React | Vue 3 |
|--------|-------|-------|
| Валидация runtime | `prop-types` (deprecated) | `defineProps` (встроено) |
| Дефолты | Деструктуризация / `defaultProps` | `withDefaults` / `default` в options |
| Типизация | TypeScript (рекомендуется) | TypeScript (`lang="ts"`) |
| Направление | Движение к полной типизации через TS | Валидация runtime + TS |
