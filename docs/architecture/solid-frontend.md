# SOLID в Frontend: Как запомнить и применить в React и Vue

## Мнемоника для запоминания

**SOLID** — это акроним. Запомни как "твёрдый код":

| Буква | Принцип | Аналогия |
|-------|---------|----------|
| **S** | Single Responsibility | **Один мастер — одно дело** |
| **O** | Open/Closed | **Стена с закладными: подключай что угодно** |
| **L** | Liskov Substitution | **Универсальный кирпич: любой блок подходит** |
| **I** | Interface Segregation | **Чертеж комнаты: не тащи лишнее** |
| **D** | Dependency Inversion | **Подрядчик через договор: не важен мастер** |

---

## S — Single Responsibility (Один мастер — одно дело)

**Суть:** Компонент/функция должны делать одну вещь. Если компонент растёт — раздели его.

**Аналогия:** На стройке у каждого мастера своя задача. Каменщик кладёт стены, электрик тянет проводку, маляр красит. Если один человек пытается делать всё — качество падает. Так и с компонентом: одна ответственность — одна причина менять.

### React

**Плохо:**
```tsx
// Компонент делает всё: fetch, форматирование, рендер
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => {
        setUsers(data.map(u => ({ ...u, name: u.name.toUpperCase() })));
        setLoading(false);
      });
  }, []);

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name} ({user.email})</li>
      ))}
    </ul>
  );
}
```

**Хорошо:**
```tsx
// Хук отвечает только за данные
function useUsers() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => setUsers(data));
  }, []);

  return { users, loading };
}

// Компонент отвечает только за рендер
function UserList() {
  const { users, loading } = useUsers();

  if (loading) return <div>Loading...</div>;

  return (
    <ul>
      {users.map(user => (
        <UserItem key={user.id} user={user} />
      ))}
    </ul>
  );
}

// Отдельный компонент для элемента
function UserItem({ user }) {
  return <li>{user.name.toUpperCase()} ({user.email})</li>;
}
```

### Vue

**Плохо:**
```vue
<script setup>
import { ref, onMounted } from 'vue';

const users = ref([]);
const loading = ref(true);

onMounted(async () => {
  const res = await fetch('/api/users');
  const data = await res.json();
  users.value = data.map(u => ({ ...u, name: u.name.toUpperCase() }));
  loading.value = false;
});
</script>

<template>
  <ul v-if="!loading">
    <li v-for="user in users" :key="user.id">
      {{ user.name }} ({{ user.email }})
    </li>
  </ul>
  <div v-else>Loading...</div>
</template>
```

**Хорошо:**
```vue
<!-- composables/useUsers.js -->
<script setup>
import { ref, onMounted } from 'vue';

export function useUsers() {
  const users = ref([]);
  const loading = ref(true);

  onMounted(async () => {
    const res = await fetch('/api/users');
    users.value = await res.json();
    loading.value = false;
  });

  return { users, loading };
}
</script>
```

```vue
<!-- UserList.vue -->
<script setup>
import { useUsers } from './composables/useUsers';
import UserItem from './UserItem.vue';

const { users, loading } = useUsers();
</script>

<template>
  <div v-if="loading">Loading...</div>
  <ul v-else>
    <UserItem v-for="user in users" :key="user.id" :user="user" />
  </ul>
</template>
```

```vue
<!-- UserItem.vue -->
<script setup>
defineProps(['user']);
</script>

<template>
  <li>{{ user.name.toUpperCase() }} ({{ user.email }})</li>
</template>
```

---

## O — Open/Closed (Стена с закладными: подключай что угодно)

**Суть:** Код должен быть открыт для расширения, но закрыт для изменения. Добавляй новое, не ломая старое.

**Аналогия:** При строительстве стены ты закладываешь крепёжные элементы. Потом на эту стену можно повесить полку, телевизор или картину — без разрушения конструкции. Стена открыта для расширения, но закрыта для переделки.

### React

**Плохо:**
```tsx
// При добавлении типа кнопки нужно менять компонент
function Button({ type, onClick, children }) {
  if (type === 'primary') {
    return <button className="btn-primary" onClick={onClick}>{children}</button>;
  }
  if (type === 'secondary') {
    return <button className="btn-secondary" onClick={onClick}>{children}</button>;
  }
  // Добавляешь новый тип — меняешь код
}
```

**Хорошо:**
```tsx
// Базовый компонент принимает className как проп
function Button({ className = '', onClick, children, ...props }) {
  return (
    <button className={`btn ${className}`} onClick={onClick} {...props}>
      {children}
    </button>
  );
}

// Расширяешь через композицию, не меняя базовый компонент
function PrimaryButton(props) {
  return <Button className="btn-primary" {...props} />;
}

function SecondaryButton(props) {
  return <Button className="btn-secondary" {...props} />;
}

// Или через render-пропсы для ещё большей гибкости
function Button({ render, ...props }) {
  return render(props);
}

<Button render={(props) => <button className="btn-custom" {...props} />} />
```

### Vue

**Плохо:**
```vue
<script setup>
const props = defineProps(['type']);
</script>

<template>
  <button v-if="type === 'primary'" class="btn-primary">
    <slot />
  </button>
  <button v-else-if="type === 'secondary'" class="btn-secondary">
    <slot />
  </button>
</template>
```

**Хорошо:**
```vue
<!-- BaseButton.vue -->
<script setup>
defineProps(['class']);
</script>

<template>
  <button :class="['btn', class]">
    <slot />
  </button>
</template>
```

```vue
<!-- PrimaryButton.vue -->
<script setup>
import BaseButton from './BaseButton.vue';
</script>

<template>
  <BaseButton class="btn-primary">
    <slot />
  </BaseButton>
</template>
```

```vue
<!-- Используешь через динамический компонент -->
<script setup>
import PrimaryButton from './PrimaryButton.vue';
import SecondaryButton from './SecondaryButton.vue';

const buttonType = 'primary';
</script>

<template>
  <component :is="buttonType === 'primary' ? PrimaryButton : SecondaryButton">
    Click me
  </component>
</template>
```

---

## L — Liskov Substitution (Универсальный кирпич: любой блок подходит)

**Суть:** Если S — подтип T, то объекты T можно заменить на S без изменения поведения.

**Аналогия:** Кирпичи в стене. Ты можешь заменить один блок на другой — стена стоит. Главное, чтобы размер и стандарт совпадали. Если кирпич подходит по ГОСТу — кладка не рушится. Так и с типами: если объект реализует базовый интерфейс, он работает везде.

### React

**Плохо:**
```tsx
// Компонент ожидает специфичные пропсы, но получает другие
function UserCard({ user }) {
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      <button onClick={() => user.delete()}>Delete</button>
    </div>
  );
}

// Guest не имеет метода delete — ошибка!
<UserCard user={{ name: 'Guest', email: 'guest@example.com' }} />
```

**Хорошо:**
```tsx
// Базовый интерфейс для всех пользователей
interface User {
  name: string;
  email: string;
}

// Компонент работает с базовым интерфейсом
function UserCard({ user }: { user: User }) {
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}

// Расширения могут иметь дополнительные методы
interface AdminUser extends User {
  delete: () => void;
}

// AdminCard расширяет UserCard, но UserCard всё ещё работает с любым User
function AdminCard({ user }: { user: AdminUser }) {
  return (
    <UserCard user={user} />
  );
}

// Теперь любой User подходит
<UserCard user={{ name: 'Guest', email: 'guest@example.com' }} />
<AdminCard user={{ name: 'Admin', email: 'admin@example.com', delete: () => {} }} />
```

### Vue

**Плохо:**
```vue
<!-- UserCard.vue -->
<script setup>
const props = defineProps({
  user: {
    type: Object,
    required: true,
    validator: (user) => {
      // Ожидаем метод delete — но Guest его не имеет
      return typeof user.delete === 'function';
    }
  }
});
</script>

<template>
  <div>
    <h3>{{ user.name }}</h3>
    <button @click="user.delete()">Delete</button>
  </div>
</template>
```

**Хорошо:**
```vue
<!-- BaseUserCard.vue -->
<script setup>
defineProps({
  user: {
    type: Object,
    required: true,
    validator: (user) => user.name && user.email
  }
});
</script>

<template>
  <div class="user-card">
    <h3>{{ user.name }}</h3>
    <p>{{ user.email }}</p>
    <slot name="actions" />
  </div>
</template>
```

```vue
<!-- AdminUserCard.vue -->
<script setup>
import BaseUserCard from './BaseUserCard.vue';

defineProps({
  user: {
    type: Object,
    required: true
  }
});
</script>

<template>
  <BaseUserCard :user="user">
    <template #actions>
      <button @click="user.delete()">Delete</button>
    </template>
  </BaseUserCard>
</template>
```

```vue
<!-- Используешь с любым типом пользователя -->
<BaseUserCard :user="{ name: 'Guest', email: 'guest@example.com' }" />
<AdminUserCard :user="{ name: 'Admin', email: 'admin@example.com', delete: () => {} }" />
```

---

## I — Interface Segregation (Чертеж комнаты: не тащи лишнее)

**Суть:** Не заставляй клиент зависеть от методов, которые он не использует. Разделяй интерфейсы.

**Аналогия:** Чертеж дома. Для каждой комнаты — свой план. В санузел не тащи кухонную разводку, а в спальню — вентиляцию из гаража. Каждая комната получает только то, что ей нужно. Так и с интерфейсами: не заставляй компонент зависеть от методов, которые он не использует.

**Важно понимать:** Backend может отдавать огромные DTO, но в компонент нужно передавать только то, что он реально использует. Это не значит, что нужно искусственно дробить — если компоненту нужно 15 полей, передавай 15 полей.

### React

**Плохо (передаём весь DTO, когда нужно 2 поля):**
```tsx
// Backend отдаёт огромный User объект
interface UserDTO {
  id: number;
  name: string;
  email: string;
  avatar: string;
  phone: string;
  address: string;
  createdAt: Date;
  updatedAt: Date;
  lastLogin: Date;
  role: string;
  // ... ещё 10 полей
}

// Компонент Avatar использует только avatarUrl и name
function Avatar({ user }: { user: UserDTO }) {
  return (
    <div className="avatar">
      <img src={user.avatar} alt={user.name} />
      <span>{user.name}</span>
    </div>
  );
}

// Передаём весь объект, хотя нужно только 2 поля
<Avatar user={userDTO} />
```

**Хорошо (передаём только то, что нужно):**
```tsx
// Компонент Avatar требует только нужные поля
interface AvatarProps {
  avatarUrl: string;
  name: string;
}

function Avatar({ avatarUrl, name }: AvatarProps) {
  return (
    <div className="avatar">
      <img src={avatarUrl} alt={name} />
      <span>{name}</span>
    </div>
  );
}

// В родительском компоненте извлекаем только нужное
function UserProfile({ userDTO }: { userDTO: UserDTO }) {
  return (
    <div>
      <Avatar avatarUrl={userDTO.avatar} name={userDTO.name} />
      <div>{userDTO.email}</div>
      <div>{userDTO.phone}</div>
    </div>
  );
}
```

**Нормально (если компонент реально использует много полей):**
```tsx
// Если компонент реально использует 10 полей — передавай 10 полей
interface UserFormProps {
  name: string;
  email: string;
  phone: string;
  address: string;
  role: string;
  avatar: string;
  // ... все поля, которые реально используются
}

function UserForm({ name, email, phone, address, role, avatar }: UserFormProps) {
  // Форма использует все эти поля
}
```

### Vue

**Плохо (передаём весь DTO, когда нужно 2 поля):**
```vue
<!-- Avatar.vue -->
<script setup>
const props = defineProps({
  user: {
    type: Object,
    required: true
    // Принимает весь UserDTO, хотя использует только avatar и name
  }
});
</script>

<template>
  <div class="avatar">
    <img :src="user.avatar" :alt="user.name" />
    <span>{{ user.name }}</span>
  </div>
</template>
```

```vue
<!-- Использование -->
<template>
  <Avatar :user="userDTO" />
</template>
```

**Хорошо (передаём только то, что нужно):**
```vue
<!-- Avatar.vue -->
<script setup>
const props = defineProps({
  avatarUrl: { type: String, required: true },
  name: { type: String, required: true }
});
</script>

<template>
  <div class="avatar">
    <img :src="avatarUrl" :alt="name" />
    <span>{{ name }}</span>
  </div>
</template>
```

```vue
<!-- UserProfile.vue -->
<script setup>
import Avatar from './Avatar.vue';

const props = defineProps({
  userDTO: { type: Object, required: true }
});
</script>

<template>
  <div>
    <Avatar :avatar-url="userDTO.avatar" :name="userDTO.name" />
    <div>{{ userDTO.email }}</div>
    <div>{{ userDTO.phone }}</div>
  </div>
</template>
```

**Нормально (если компонент реально использует много полей):**
```vue
<!-- UserForm.vue -->
<script setup>
const props = defineProps({
  name: { type: String, required: true },
  email: { type: String, required: true },
  phone: { type: String, required: true },
  address: { type: String, required: true },
  role: { type: String, required: true }
  // Все поля, которые реально используются
});
</script>

<template>
  <!-- Форма использует все эти поля -->
</template>
```

---

## D — Dependency Inversion (Подрядчик через договор: не важен мастер)

**Суть:** Зависеть от абстракций, а не от конкретики. Высокоуровневые модули не должны зависеть от низкоуровневых.

**Аналогия:** Заказчик нанимает подрядчика через договор (стандарт работы), а не конкретного мастера. Не важно, кто будет класть плитку — Иванов или Петров. Главное, чтобы работа соответствовала ГОСТу. Так и в коде: завись от интерфейса (договора), а не от конкретной реализации (мастера).

**Когда это нужно на практике:**
- Тестирование без реального API (мок-сервис)
- Переключение между окружениями (dev/staging/prod)
- Миграция на другой API без переписывания компонентов
- Несколько источников данных (REST + GraphQL + локальный кэш)

**Когда НЕ нужно:**
- Один API, который не меняется
- Прототип или MVP
- Нет планов тестировать компонент изолированно

### React

**Плохо (когда DI нужен, но не используется):**
```tsx
// Компонент жёстко зависит от конкретного API
function UserList() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    // Захардкожено: URL, метод, формат ответа
    fetch('https://api.example.com/users')
      .then(res => res.json())
      .then(data => setUsers(data.users)); // Формат data.users тоже зашит
  }, []);

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

**Хорошо (когда DI оправдан):**
```tsx
// Абстракция — сервис данных
interface UserService {
  getUsers(): Promise<User[]>;
}

// Конкретная реализация для production
class HttpUserService implements UserService {
  async getUsers(): Promise<User[]> {
    const res = await fetch('https://api.example.com/users');
    const data = await res.json();
    return data.users; // Преобразование формата внутри сервиса
  }
}

// Мок для тестов
class MockUserService implements UserService {
  async getUsers(): Promise<User[]> {
    return [
      { id: 1, name: 'Test User' }
    ];
  }
}

// Компонент зависит от абстракции
function UserList({ userService }: { userService: UserService }) {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    userService.getUsers().then(setUsers);
  }, [userService]);

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}

// В приложении — реальная реализация
<UserList userService={new HttpUserService()} />

// В тестах — мок
<UserList userService={new MockUserService()} />
```

**Нормально (когда DI не нужен):**
```tsx
// Если API стабилен и тесты не важны — можно проще
function UserList() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(setUsers);
  }, []);

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### Vue

**Плохо (когда DI нужен, но не используется):**
```vue
<script setup>
import { ref, onMounted } from 'vue';

const users = ref([]);

onMounted(async () => {
  // Захардкожено: URL, формат ответа
  const res = await fetch('https://api.example.com/users');
  const data = await res.json();
  users.value = data.users;
});
</script>

<template>
  <ul>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

**Хорошо (когда DI оправдан):**
```ts
// services/userService.ts
export interface UserService {
  getUsers(): Promise<User[]>;
}

export class HttpUserService implements UserService {
  async getUsers(): Promise<User[]> {
    const res = await fetch('https://api.example.com/users');
    const data = await res.json();
    return data.users;
  }
}

export class MockUserService implements UserService {
  async getUsers(): Promise<User[]> {
    return [{ id: 1, name: 'Test User' }];
  }
}
```

```vue
<!-- UserList.vue -->
<script setup>
import { ref, onMounted } from 'vue';
import type { UserService } from './services/userService';

const props = defineProps({
  userService: {
    type: Object as () => UserService,
    required: true
  }
});

const users = ref([]);

onMounted(async () => {
  users.value = await props.userService.getUsers();
});
</script>

<template>
  <ul>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

```vue
<!-- App.vue -->
<script setup>
import UserList from './UserList.vue';
import { HttpUserService } from './services/userService';

const userService = new HttpUserService();
</script>

<template>
  <UserList :user-service="userService" />
</template>
```

**Нормально (когда DI не нужен):**
```vue
<script setup>
import { ref, onMounted } from 'vue';

const users = ref([]);

onMounted(async () => {
  const res = await fetch('/api/users');
  users.value = await res.json();
});
</script>

<template>
  <ul>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

---

## Шпаргалка для собеседования

| Принцип | Вопрос на собесе | Короткий ответ |
|---------|------------------|----------------|
| **S** | "Как понять, что компонент слишком большой?" | "Если он делает больше одной вещи: fetch + рендер + валидация. Раздели на хуки/компоненты." |
| **O** | "Как добавить новый тип кнопки без изменения кода?" | "Через композицию: базовый компонент + расширение через пропсы или render-пропсы." |
| **L** | "Можно ли заменить Admin на Guest?" | "Да, если оба реализуют базовый интерфейс User. Компонент не должен ожидать специфичных методов." |
| **I** | "Зачем разделять пропсы?" | "Backend может отдавать огромный DTO, но в компонент передаём только то, что он реально использует. Не тащим лишнее." |
| **D** | "Как тестировать компонент с API?" | "Зависеть от абстракции (сервис), передавать реализацию снаружи. В тестах — мок. Но если API стабилен и тестов нет — можно без DI." |

**Важный вопрос на собесе:** "Когда НЕ применять SOLID?"
- Прототипы и MVP
- Маленькие проекты с одним разработчиком
- Когда абстракция не упрощает код, а усложняет
- Когда нет конкретной проблемы, которую решает принцип

---

## Когда НЕ применять SOLID

SOLID — это инструмент, а не религия. Вот ситуации, когда можно (и нужно) отступать от правил:

### 1. Прототипы и MVP
Когда нужно быстро проверить гипотезу — пиши как проще. SOLID добавишь позже, если проект выживет.

```tsx
// Нормально для прототипа
function UserList() {
  const [users, setUsers] = useState([]);
  
  useEffect(() => {
    fetch('/api/users').then(r => r.json()).then(setUsers);
  }, []);

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### 2. Маленькие проекты
Если проект на 5-10 компонентов и один разработчик — оверинжиниринг вреден. Пиши просто.

### 3. Backend стабилен и не меняется
Если API не меняется годами, нет смысла делать DI "на всякий случай". Добавишь, когда появится вторая реализация.

### 4. Компонент реально использует все поля
Если форма редактирования пользователя использует 15 полей — передавай 15 полей. Не нужно искусственно дробить на 3 интерфейса.

### 5. Нет планов тестировать изолированно
Если компонент всегда будет работать с реальным API и ты не пишешь unit-тесты — DI не нужен.

### Признаки оверинжиниринга:
- Создаёшь абстракцию "на будущее", но нет конкретной причины
- Интерфейс из 3 методов, где нужен 1
- Компонент разбит на 5 частей, хотя каждая часть используется только здесь
- Код стал сложнее читать из-за "правильной архитектуры"

**Правило:** Если абстракция не упрощает код сейчас и не решает конкретную проблему — она не нужна.

---

## Главные мысли

1. **SOLID — это инструмент, а не догма.** Применяй там, где это упрощает код и решает конкретные проблемы.
2. **S и O — самые полезные.** Single Responsibility и Open/Closed легко соблюдаются и реально облегчают жизнь.
3. **D и I — ситуативные.** Dependency Inversion нужен для тестов и гибкости, Interface Segregation — когда компонент не использует все поля.
4. **Не занимайся оверинжинирингом.** Если абстракция не упрощает код сейчас — она не нужна.
5. **В React** SOLID проявляется через хуки, композицию пропсов и разделение ответственности.
6. **В Vue** — через composables, слоты и dependency injection.
7. **Запоминай через аналогии:** мастер, закладные, кирпич, чертёж, договор.
8. **На собесах** важнее показать применение и понимание, когда правило НЕ нужно применять.
