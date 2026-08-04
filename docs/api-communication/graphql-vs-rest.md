# GraphQL vs REST: Полное руководство для Frontend-разработчика

## Введение

При построении API разработчики выбирают между REST и GraphQL. В этой статье разберём обе технологии, их преимущества, недостатки и сценарии использования. Особое внимание уделим GraphQL.

## Что такое REST

REST (Representational State Transfer) — архитектурный стиль для распределённых систем. Основные принципы:

- **Ресурсы** — объекты с URI (`/users/123`)
- **HTTP-методы** — GET, POST, PUT, DELETE
- **Stateless** — каждый запрос независим
- **Стандартные коды ответов** — 200, 404, 500

### Пример REST API

```
GET    /api/users          # Получить всех пользователей
GET    /api/users/123      # Получить пользователя по ID
POST   /api/users          # Создать пользователя
PUT    /api/users/123      # Обновить пользователя
DELETE /api/users/123      # Удалить пользователя
```

### Проблемы REST

**Over-fetching** — получаешь больше данных, чем нужно:

```js
// Запрос: GET /api/users/123
// Ответ содержит ВСЕ поля, даже те, что не нужны:
{
  "id": 123,
  "name": "John",
  "email": "john@example.com",
  "password": "***",        // Не нужен на фронте
  "createdAt": "...",       // Не нужен на фронте
  "internalData": "..."     // Не нужен на фронте
}
```

**Under-fetching** — нужно делать несколько запросов:

```js
// Получить пользователя и его посты
const user = await fetch('/api/users/123');
const posts = await fetch('/api/users/123/posts');
const comments = await fetch('/api/posts/456/comments');
// 3 запроса вместо одного
```

## Что такое GraphQL

GraphQL — язык запросов для API, созданный Facebook в 2012 году (открыт в 2015). Позволяет клиенту запрашивать именно те данные, которые нужны.

### Основные концепции

#### 1. Схема (Schema)

Определяет структуру данных и доступные операции:

```graphql
# Типы данных
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
}

type Comment {
  id: ID!
  text: String!
  author: User!
}

# Доступные запросы
type Query {
  users: [User!]!
  user(id: ID!): User
  posts: [Post!]!
}

# Доступные мутации (изменения)
type Mutation {
  createUser(name: String!, email: String!): User!
  createPost(title: String!, content: String!, authorId: ID!): Post!
}

# Подписки (real-time)
type Subscription {
  postCreated: Post!
}
```

#### 2. Запросы (Queries)

Клиент сам определяет структуру ответа:

```graphql
# Запрос
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}

# Ответ — ТОЛЬКО запрошенные поля
{
  "data": {
    "user": {
      "name": "John",
      "email": "john@example.com",
      "posts": [
        { "title": "First Post" },
        { "title": "Second Post" }
      ]
    }
  }
}
```

#### 3. Мутации (Mutations)

Операции для изменения данных:

```graphql
# Мутация с переменными
mutation CreateUser($name: String!, $email: String!) {
  createUser(name: $name, email: $email) {
    id
    name
    email
  }
}

# Переменные
{
  "name": "John",
  "email": "john@example.com"
}
```

#### 4. Подписки (Subscriptions)

Real-time обновления через WebSocket:

```graphql
subscription {
  postCreated {
    id
    title
    author {
      name
    }
  }
}
```

## Примеры кода

### Backend (Node.js + Apollo Server)

```js
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';

// Схема
const typeDefs = `#graphql
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
  }

  type Query {
    users: [User!]!
    user(id: ID!): User
  }

  type Mutation {
    createUser(name: String!, email: String!): User!
  }

  type Subscription {
    userCreated: User!
  }
`;

// Резолверы — как получать данные
const resolvers = {
  Query: {
    users: async () => {
      return await db.users.findAll();
    },
    user: async (_, { id }) => {
      return await db.users.findById(id);
    },
  },

  Mutation: {
    createUser: async (_, { name, email }) => {
      const user = await db.users.create({ name, email });
      // Публикуем событие для подписок
      pubsub.publish('USER_CREATED', { userCreated: user });
      return user;
    },
  },

  // Резолверы для вложенных полей
  User: {
    posts: async (user) => {
      return await db.posts.findByAuthor(user.id);
    },
  },

  Subscription: {
    userCreated: {
      subscribe: () => pubsub.asyncIterator(['USER_CREATED']),
    },
  },
};

const server = new ApolloServer({ typeDefs, resolvers });

const { url } = await startStandaloneServer(server, { listen: { port: 4000 } });
console.log(`Server ready at ${url}`);
```

### Frontend (React + Apollo Client)

#### Настройка клиента

```js
import { ApolloClient, InMemoryCache, ApolloProvider } from '@apollo/client';

const client = new ApolloClient({
  uri: 'http://localhost:4000/graphql',
  cache: new InMemoryCache(),
});

function App() {
  return (
    <ApolloProvider client={client}>
      <Users />
    </ApolloProvider>
  );
}
```

#### Запросы (useQuery)

```js
import { gql, useQuery } from '@apollo/client';

const GET_USERS = gql`
  query GetUsers {
    users {
      id
      name
      email
      posts {
        id
        title
      }
    }
  }
`;

function Users() {
  const { loading, error, data } = useQuery(GET_USERS);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {data.users.map(user => (
        <li key={user.id}>
          <h3>{user.name}</h3>
          <p>{user.email}</p>
          <h4>Posts:</h4>
          <ul>
            {user.posts.map(post => (
              <li key={post.id}>{post.title}</li>
            ))}
          </ul>
        </li>
      ))}
    </ul>
  );
}
```

#### Запросы с переменными

```js
const GET_USER = gql`
  query GetUser($userId: ID!) {
    user(id: $userId) {
      id
      name
      email
    }
  }
`;

function UserProfile({ userId }) {
  const { loading, error, data } = useQuery(GET_USER, {
    variables: { userId },
  });

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <div>
      <h2>{data.user.name}</h2>
      <p>{data.user.email}</p>
    </div>
  );
}
```

#### Мутации (useMutation)

```js
import { gql, useMutation } from '@apollo/client';

const CREATE_USER = gql`
  mutation CreateUser($name: String!, $email: String!) {
    createUser(name: $name, email: $email) {
      id
      name
      email
    }
  }
`;

function CreateUserForm() {
  const [createUser, { loading, error }] = useMutation(CREATE_USER);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);

    try {
      const { data } = await createUser({
        variables: {
          name: formData.get('name'),
          email: formData.get('email'),
        },
      });
      console.log('User created:', data.createUser);
    } catch (err) {
      console.error('Error:', err);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" placeholder="Name" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create User'}
      </button>
      {error && <p>Error: {error.message}</p>}
    </form>
  );
}
```

#### Подписки (useSubscription)

```js
import { gql, useSubscription } from '@apollo/client';

const USER_CREATED = gql`
  subscription UserCreated {
    userCreated {
      id
      name
      email
    }
  }
`;

function NewUsersNotification() {
  const { data, loading } = useSubscription(USER_CREATED);

  if (loading) return <p>Listening for new users...</p>;

  return (
    <div>
      <h3>New user registered!</h3>
      <p>{data.userCreated.name} ({data.userCreated.email})</p>
    </div>
  );
}
```

#### Ленивые запросы (useLazyQuery)

```js
import { gql, useLazyQuery } from '@apollo/client';

const SEARCH_USERS = gql`
  query SearchUsers($query: String!) {
    searchUsers(query: $query) {
      id
      name
    }
  }
`;

function SearchUsers() {
  const [searchUsers, { data, loading }] = useLazyQuery(SEARCH_USERS);

  const handleSearch = (query) => {
    searchUsers({ variables: { query } });
  };

  return (
    <div>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {loading && <p>Searching...</p>}
      {data?.searchUsers.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
    </div>
  );
}
```

## Сравнение GraphQL и REST

| Критерий | GraphQL | REST |
|----------|---------|------|
| **Endpoints** | Один (`/graphql`) | Много (`/users`, `/posts`) |
| **Получение данных** | Клиент выбирает структуру | Сервер определяет структуру |
| **Over-fetching** | Нет | Да |
| **Under-fetching** | Нет | Да (N запросов) |
| **Кэширование** | Сложнее (нужен Apollo Client) | Из коробки (HTTP-кэш) |
| **Версионирование** | Не нужно | `/v1/users`, `/v2/users` |
| **Документация** | Автосгенерированная из схемы | Swagger/OpenAPI вручную |
| **Real-time** | Subscriptions из коробки | WebSocket отдельно |
| **Файлы** | Не поддерживает (нужен multipart) | Поддерживает |
| **Порог входа** | Выше | Ниже |
| **Типизация** | Встроена в схему | Вручную (TypeScript) |
| **Отладка** | GraphiQL/Playground | DevTools, URL |

## Плюсы и минусы

### GraphQL

**Плюсы:**
- Запрашиваешь ровно то, что нужно
- Один запрос вместо нескольких
- Автодокументация и автокомплит (GraphiQL)
- Изменения без версионирования
- Строгая типизация из коробки
- Real-time подписки
- Меньше round-trips к серверу

**Минусы:**
- Сложность кэширования
- Проблема N+1 (нужен DataLoader)
- Высокий порог входа
- Не поддерживает загрузку файлов нативно
- Сложные запросы могут нагружать сервер
- Меньше готовых инструментов для мониторинга

### REST

**Плюсы:**
- Простота и зрелость
- HTTP-кэширование из коробки
- Поддержка файлов
- Низкий порог входа
- Широкая экосистема инструментов
- Легко масштабировать

**Минусы:**
- Over-fetching и under-fetching
- Множество endpoints
- Нужна версионность
- Документация вручную
- Больше round-trips для сложных данных

## Сложность для разработчиков

### Frontend-разработчик

| Задача | REST | GraphQL |
|--------|------|---------|
| Получение данных | Несколько fetch/axios вызовов | Один запрос с нужными полями |
| Изменение данных | Ждать новый endpoint | Сам добавляешь поля в запрос |
| Типизация | Вручную (TypeScript) | Автогенерация из схемы |
| Кэширование | Простое (HTTP) | Нужен Apollo Client / URQL |
| Отладка | DevTools, URL | GraphiQL/Playground |
| Порог входа | Низкий | Средний |

**Пример:** Frontend-разработчику нужны имя пользователя и количество его постов.

**REST:**
```js
// Нужно запросить весь объект пользователя
const user = await fetch('/api/users/123');
// Получить все посты
const posts = await fetch('/api/users/123/posts');
// Посчитать количество на клиенте
const postCount = posts.length;
```

**GraphQL:**
```graphql
query {
  user(id: "123") {
    name
    postsCount
  }
}
```

### Backend-разработчик

| Задача | REST | GraphQL |
|--------|------|---------|
| Создание endpoint | Простой CRUD | Нужна схема + резолверы |
| Защита данных | На уровне endpoint | На уровне полей (directive) |
| Оптимизация | Простая | Проблема N+1 (нужен DataLoader) |
| Валидация | Ручная или middleware | Встроена в схему |
| Версионирование | `/v1/`, `/v2/` | Не нужно |
| Документация | Swagger вручную | Авто из схемы |
| Порог входа | Низкий | Высокий |

**Пример:** Backend-разработчику нужно добавить новое поле `avatar` для пользователя.

**REST:**
```js
// Нужно изменить endpoint, возможно создать новую версию
// GET /api/v2/users/123
```

**GraphQL:**
```graphql
# Просто добавить поле в схему
type User {
  id: ID!
  name: String!
  avatar: String  # Новое поле
}

# Резолвер
User: {
  avatar: (user) => getAvatarUrl(user.id)
}
```

## Когда что использовать

### Выбирай REST, если:

- Простой CRUD API
- Публичное API для третьих сторон
- Нужна загрузка файлов
- Команда не знакома с GraphQL
- Простые сценарии использования
- Важно HTTP-кэширование

### Выбирай GraphQL, если:

- Сложный UI с вложенными данными
- Мобильные приложения (экономия трафика)
- Нужна быстрая итерация без ожидания бэкенда
- Real-time функциональность
- Несколько клиентов с разными потребностями (web, mobile, tablet)
- Команда готова к learning curve

## Компромиссные решения

### BFF (Backend for Frontend)

Отдельный слой между frontend и backend, который агрегирует данные:

```
[Mobile App] --> [Mobile BFF] --> [Backend Services]
[Web App]    --> [Web BFF]    --> [Backend Services]
```

### REST + GraphQL

Использовать REST для простых операций, GraphQL для сложных:

```js
// Простые операции через REST
POST /api/auth/login
GET  /api/files/download

// Сложные данные через GraphQL
query {
  dashboard {
    user { name }
    recentPosts { title }
    notifications { message }
  }
}
```

## Заключение

**GraphQL** — мощный инструмент для сложных приложений с вложенными данными и необходимостью гибкости. Требует инвестиций в обучение, но окупается скоростью разработки frontend.

**REST** — проверенное временем решение для простых API с широкой экосистемой инструментов.

Выбор зависит от:
- Сложности приложения
- Размера команды
- Требований к производительности
- Готовности к learning curve

Для большинства современных web-приложений с сложным UI GraphQL становится предпочтительным выбором, особенно если команда готова инвестировать в обучение.
