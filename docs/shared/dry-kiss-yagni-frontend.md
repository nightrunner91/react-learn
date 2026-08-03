# DRY, KISS, YAGNI: Три кита здорового кода во Frontend

## Мнемоника для запоминания

Эти три принципа — как правила техники безопасности. Короткие, простые, спасают от проблем.

| Принцип | Расшифровка | Аналогия |
|---------|-------------|----------|
| **DRY** | Don't Repeat Yourself | **Копирка: один раз написал — используй везде** |
| **KISS** | Keep It Simple, Stupid | **Молоток для гвоздя: не бери отбойный** |
| **YAGNI** | You Aren't Gonna Need It | **Не покупай продукты на неделю, если не знаешь меню** |

---

## DRY — Don't Repeat Yourself (Копирка: один раз написал — используй везде)

**Суть:** Каждая часть знаний должна иметь единственное, авторитетное и непротиворечивое представление в системе. Если код повторяется — вынеси в одно место.

**Аналогия:** Копирка. Ты один раз создаёшь шаблон, а потом используешь его многократно. Если нужно изменить — меняешь в одном месте, а не в десяти копиях. Дублирование — это баг, который ждёт своего часа.

### React

**Плохо (дублирование логики):**
```tsx
// Одинаковая логика в двух компонентах
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => {
        setUsers(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}

function ProductList() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

**Хорошо (выносим общую логику в хук):**
```tsx
// Хук инкапсулирует всю логику загрузки
function useFetch(url) {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, [url]);

  return { data, loading, error };
}

// Компоненты становятся чистыми
function UserList() {
  const { data: users, loading, error } = useFetch('/api/users');

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}

function ProductList() {
  const { data: products, loading, error } = useFetch('/api/products');

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

### Vue

**Плохо (дублирование логики):**
```vue
<!-- UserList.vue -->
<script setup>
import { ref, onMounted } from 'vue';

const users = ref([]);
const loading = ref(true);
const error = ref(null);

onMounted(async () => {
  try {
    const res = await fetch('/api/users');
    users.value = await res.json();
  } catch (err) {
    error.value = err.message;
  } finally {
    loading.value = false;
  }
});
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error }}</div>
  <ul v-else>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

```vue
<!-- ProductList.vue — та же логика -->
<script setup>
import { ref, onMounted } from 'vue';

const products = ref([]);
const loading = ref(true);
const error = ref(null);

onMounted(async () => {
  try {
    const res = await fetch('/api/products');
    products.value = await res.json();
  } catch (err) {
    error.value = err.message;
  } finally {
    loading.value = false;
  }
});
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error }}</div>
  <ul v-else>
    <li v-for="product in products" :key="product.id">{{ product.name }}</li>
  </ul>
</template>
```

**Хорошо (выносим общую логику в composable):**
```js
// composables/useFetch.js
import { ref, onMounted } from 'vue';

export function useFetch(url) {
  const data = ref([]);
  const loading = ref(true);
  const error = ref(null);

  onMounted(async () => {
    try {
      const res = await fetch(url);
      data.value = await res.json();
    } catch (err) {
      error.value = err.message;
    } finally {
      loading.value = false;
    }
  });

  return { data, loading, error };
}
```

```vue
<!-- UserList.vue -->
<script setup>
import { useFetch } from './composables/useFetch';

const { data: users, loading, error } = useFetch('/api/users');
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error }}</div>
  <ul v-else>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

```vue
<!-- ProductList.vue -->
<script setup>
import { useFetch } from './composables/useFetch';

const { data: products, loading, error } = useFetch('/api/products');
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error }}</div>
  <ul v-else>
    <li v-for="product in products" :key="product.id">{{ product.name }}</li>
  </ul>
</template>
```

---

## KISS — Keep It Simple, Stupid (Молоток для гвоздя: не бери отбойный)

**Суть:** Самое простое решение часто оказывается лучшим. Не усложняй код без необходимости.

**Аналогия:** Если нужно забить гвоздь — бери молоток, а не отбойный молоток. Отбойный мощнее, но для гвоздя это оверкилл. Так и в коде: не используй сложную архитектуру, если задача решается проще.

### React

**Плохо (излишнее усложнение):**
```tsx
// Используем Redux для локального состояния формы
import { useSelector, useDispatch } from 'react-redux';
import { setUserName, setUserEmail } from './store/userSlice';

function UserForm() {
  const dispatch = useDispatch();
  const name = useSelector(state => state.user.name);
  const email = useSelector(state => state.user.email);

  return (
    <form>
      <input
        value={name}
        onChange={e => dispatch(setUserName(e.target.value))}
      />
      <input
        value={email}
        onChange={e => dispatch(setUserEmail(e.target.value))}
      />
    </form>
  );
}
```

**Хорошо (простое решение для простой задачи):**
```tsx
// Локальное состояние — проще и понятнее
function UserForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  return (
    <form>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input value={email} onChange={e => setEmail(e.target.value)} />
    </form>
  );
}
```

**Плохо (излишняя абстракция):**
```tsx
// Создаём фабрику компонентов для одной кнопки
function createButton(className, icon) {
  return function Button({ onClick, children }) {
    return (
      <button className={className} onClick={onClick}>
        {icon && <span className="icon">{icon}</span>}
        {children}
      </button>
    );
  };
}

const PrimaryButton = createButton('btn-primary', '★');
const SecondaryButton = createButton('btn-secondary', null);
```

**Хорошо (прямое решение):**
```tsx
// Просто два компонента
function PrimaryButton({ onClick, children }) {
  return (
    <button className="btn-primary" onClick={onClick}>
      <span className="icon">★</span>
      {children}
    </button>
  );
}

function SecondaryButton({ onClick, children }) {
  return (
    <button className="btn-secondary" onClick={onClick}>
      {children}
    </button>
  );
}
```

### Vue

**Плохо (излишнее усложнение):**
```vue
<!-- Используем Vuex для локального состояния -->
<script setup>
import { useStore } from 'vuex';

const store = useStore();
const name = computed(() => store.state.user.name);
const email = computed(() => store.state.user.email);

const updateName = (e) => store.commit('setUserName', e.target.value);
const updateEmail = (e) => store.commit('setUserEmail', e.target.value);
</script>

<template>
  <form>
    <input :value="name" @input="updateName" />
    <input :value="email" @input="updateEmail" />
  </form>
</template>
```

**Хорошо (простое решение):**
```vue
<script setup>
import { ref } from 'vue';

const name = ref('');
const email = ref('');
</script>

<template>
  <form>
    <input v-model="name" />
    <input v-model="email" />
  </form>
</template>
```

**Плохо (излишняя абстракция):**
```vue
<!-- Динамическое создание компонентов через render-функции -->
<script setup>
import { h } from 'vue';

const createField = (type, label) => {
  return {
    render() {
      return h('div', { class: 'field' }, [
        h('label', label),
        h('input', { type, value: this.modelValue, onInput: this.$emit('update:modelValue', $event.target.value) })
      ]);
    }
  };
};

const NameField = createField('text', 'Name');
const EmailField = createField('email', 'Email');
</script>
```

**Хорошо (прямое решение):**
```vue
<!-- Просто два компонента -->
<script setup>
import { ref } from 'vue';

const name = ref('');
const email = ref('');
</script>

<template>
  <div class="field">
    <label>Name</label>
    <input v-model="name" type="text" />
  </div>
  <div class="field">
    <label>Email</label>
    <input v-model="email" type="email" />
  </div>
</template>
```

---

## YAGNI — You Aren't Gonna Need It (Не покупай продукты на неделю, если не знаешь меню)

**Суть:** Не добавляй функциональность, пока она действительно не нужна. Пиши код для текущих требований, а не для гипотетических будущих.

**Аналогия:** Поход в магазин. Если ты не знаешь, что будешь готовить всю неделю — не покупай продукты наперёд. Они испортятся. Так и в коде: не пиши функционал "на всякий случай", "может пригодиться". Если он действительно понадобится — добавишь позже.

### React

**Плохо (пишем "на будущее"):**
```tsx
// Добавляем поддержку 5 типов аутентификации, хотя нужна только одна
interface AuthConfig {
  type: 'oauth' | 'jwt' | 'session' | 'saml' | 'custom';
  provider?: string;
  tokenEndpoint?: string;
  refreshEndpoint?: string;
  // ... ещё 10 полей для будущих типов
}

function AuthProvider({ config }: { config: AuthConfig }) {
  // Реализация только для JWT, но код готов для всех типов
  if (config.type === 'jwt') {
    // ...
  }
  // Остальные типы — заглушки
}
```

**Хорошо (пишем для текущих требований):**
```tsx
// Реализуем только то, что нужно сейчас
interface AuthConfig {
  type: 'jwt';
  tokenEndpoint: string;
}

function AuthProvider({ config }: { config: AuthConfig }) {
  // Простая реализация для JWT
}

// Когда понадобится другой тип — добавим
```

**Плохо (создаём абстракцию "на всякий случай"):**
```tsx
// Создаём универсальный хук для всех возможных случаев
function useData<T>(
  url: string,
  options?: {
    transform?: (data: any) => T;
    cache?: boolean;
    retry?: number;
    timeout?: number;
    fallback?: T;
    // ... ещё 10 опций
  }
) {
  // Сложная логика со всеми опциями
}

// Используем только 10% возможностей
const { data } = useData('/api/users');
```

**Хорошо (простой хук для реальной задачи):**
```tsx
// Минимальный хук, который решает текущую задачу
function useFetch(url: string) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading };
}

// Когда понадобятся дополнительные опции — добавим конкретные
```

### Vue

**Плохо (пишем "на будущее"):**
```vue
<!-- Компонент с поддержкой 10 темизаций, хотя нужна одна -->
<script setup>
const props = defineProps({
  theme: {
    type: String,
    validator: (v) => ['light', 'dark', 'blue', 'green', 'red', 'custom'].includes(v)
  },
  size: {
    type: String,
    validator: (v) => ['xs', 'sm', 'md', 'lg', 'xl', 'xxl'].includes(v)
  },
  variant: {
    type: String,
    validator: (v) => ['primary', 'secondary', 'outline', 'ghost', 'link'].includes(v)
  }
  // ... ещё 5 пропсов для будущих фич
});
</script>

<template>
  <button :class="[theme, size, variant]">
    <slot />
  </button>
</template>
```

**Хорошо (пишем для текущих требований):**
```vue
<!-- Минимальный компонент, который решает задачу -->
<script setup>
const props = defineProps({
  theme: {
    type: String,
    default: 'light'
  }
});
</script>

<template>
  <button :class="theme">
    <slot />
  </button>
</template>

<!-- Когда понадобятся другие пропсы — добавим конкретные -->
```

**Плохо (создаём систему "на всякий случай"):**
```js
// plugins/analytics.js
// Создаём универсальную систему аналитики с поддержкой 10 сервисов
export function createAnalytics(config) {
  const providers = {
    google: config.google ? new GoogleAnalytics(config.google) : null,
    yandex: config.yandex ? new YandexMetrika(config.yandex) : null,
    mixpanel: config.mixpanel ? new Mixpanel(config.mixpanel) : null,
    // ... ещё 7 провайдеров
  };

  return {
    track(event, data) {
      Object.values(providers).forEach(p => p?.track(event, data));
    }
  };
}

// Используем только Google Analytics
const analytics = createAnalytics({ google: { id: 'UA-XXX' } });
```

**Хорошо (простое решение для реальной задачи):**
```js
// Простая обёртка для конкретного сервиса
export function trackEvent(event, data) {
  gtag('event', event, data);
}

// Когда понадобится второй сервис — добавим конкретный код
```

---

## Шпаргалка для собеседования

| Принцип | Вопрос на собесе | Короткий ответ |
|---------|------------------|----------------|
| **DRY** | "Что делать, если код повторяется в двух компонентах?" | "Вынести общую логику в хук (React) или composable (Vue). Дублирование — это баг." |
| **KISS** | "Как выбрать между простым и сложным решением?" | "Всегда начинать с простого. Усложнять только когда простое решение перестало работать." |
| **YAGNI** | "Нужно ли добавлять функционал 'на всякий случай'?" | "Нет. Добавлять только когда есть конкретная потребность. Код для гипотетического будущего — это мусор." |

**Важный вопрос на собесе:** "Когда НЕ применять эти принципы?"

---

## Когда НЕ применять DRY, KISS, YAGNI

Эти принципы — инструмент, а не догма. Вот ситуации, когда можно отступать:

### 1. DRY: Когда абстракция усложняет код
Если вынос общей логики делает код сложнее для понимания — оставь дублирование. Иногда два похожих, но простых куска кода лучше, чем одна сложная абстракция.

```tsx
// Иногда это лучше, чем сложная абстракция
function UserList() {
  // Простая логика специфичная для пользователей
}

function ProductList() {
  // Похожая, но специфичная для продуктов
}
```

### 2. KISS: Когда задача действительно сложная
Не все задачи простые. Если проблема требует сложного решения — не упрощай искусственно. Но даже сложное решение должно быть настолько простым, насколько возможно.

### 3. YAGNI: Когда есть чёткие требования
Если продукт-менеджер сказал "нам нужна поддержка OAuth, SAML и JWT через месяц" — можно подготовить абстракцию. Но только если это реальные планы, а не "может быть".

### Признаки нарушения принципов:

**Нарушение DRY:**
- Один и тот же код в 3+ местах
- Исправление бага требует изменения в нескольких файлах
- Копипаст с минимальными изменениями

**Нарушение KISS:**
- Код сложнее, чем сама задача
- Используются паттерны "потому что модно"
- Непонятно, что делает код, без комментариев

**Нарушение YAGNI:**
- Функционал добавлен "на всякий случай"
- Абстракции созданы без конкретной причины
- Код содержит мёртвые ветки "для будущего"

---

## Главные мысли

1. **DRY, KISS, YAGNI — это здравый смысл.** Не догма, а инструменты для принятия решений.
2. **DRY — про дублирование.** Если код повторяется — вынеси в одно место. Но не создавай абстракций ради абстракций.
3. **KISS — про простоту.** Начинай с простого решения. Усложняй только когда простое перестало работать.
4. **YAGNI — про будущее.** Не пиши код "на всякий случай". Добавляй только когда есть конкретная потребность.
5. **В React** принципы проявляются через хуки (DRY), локальное состояние вместо глобального (KISS), и отсутствие лишних абстракций (YAGNI).
6. **В Vue** — через composables (DRY), ref/reactive вместо store (KISS), и минимальные компоненты (YAGNI).
7. **На собесах** важнее показать понимание баланса: когда применять принципы, а когда отступать.
8. **Главное правило:** Код должен решать текущую задачу максимально просто. Всё остальное — опционально.
