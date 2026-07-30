# Серверные и клиентские компоненты: правила композиции

Главное правило Next.js App Router — **серверный компонент нельзя рендерить внутри клиентского**, но **клиентский компонент можно передать в серверный через `children`**. Этот документ разбирает все допустимые и недопустимые комбинации с примерами.

## Содержание

- [Базовые определения](#базовые-определения)
- [Правило №1: children — это лазейка](#правило-1-children--это-лазейка)
- [Правило №2: импорт клиентского в серверном](#правило-2-импорт-клиентского-в-серверном)
- [Правило №3: импорт серверного в клиентском — ЗАПРЕЩЁН](#правило-3-импорт-серверного-в-клиентском--запрещён)
- [Правило №4: серверный как children в клиентский](#правило-4-серверный-как-children-в-клиентский)
- [Сводная таблица](#сводная-таблица)
- [Паттерн «Interleaving» — чередование компонентов](#паттерн-interleaving--чередование-компонентов)
- [Почему так работает: модель сериализации RSC](#почему-так-работает-модель-сериализации-rsc)
- [Практические рецепты](#практические-рецепты)
- [Антипаттерны](#антипаттерны)

## Базовые определения

**Серверный компонент (Server Component)** — компонент, который рендерится только на сервере. Не имеет доступа к браузерным API (`window`, `useState`, `onClick`). В Next.js App Router **все компоненты по умолчанию серверные**.

```tsx
// app/page.tsx — серверный по умолчанию
export default async function Page() {
  const data = await db.query('SELECT * FROM posts');
  return <ul>{data.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

**Клиентский компонент (Client Component)** — компонент, который рендерится на сервере (SSR) и гидрируется на клиенте. Имеет доступ к хукам состояния и событиям. Помечается директивой `"use client"`.

```tsx
// components/Counter.tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

## Правило №1: children — это лазейка

Серверный компонент может передать **любой JSX** (включая другие серверные компоненты) как `children` в клиентский компонент. Клиентский компонент не знает и не заботится о том, что находится в `children` — он просто рендерит их как «чёрный ящик».

```tsx
// app/layout.tsx — серверный компонент
import Modal from "@/components/Modal"; // клиентский

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <Modal>
      {/* children — это серверный компонент, переданный в клиентский */}
      {children}
    </Modal>
  );
}
```

```tsx
// components/Modal.tsx — клиентский компонент
"use client";

import { useState } from "react";

export default function Modal({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(true);
  if (!isOpen) return null;
  return (
    <div className="modal-overlay">
      <div className="modal-content">
        <button onClick={() => setIsOpen(false)}>Close</button>
        {children} {/* Рендерится без проблем — это просто React-элементы */}
      </div>
    </div>
  );
}
```

**Почему это работает:** `children` передаются как уже сериализованный RSC-поток. Клиентский компонент получает их как готовые элементы, а не как импорты. Он не пытается их «выполнить» — он просто вставляет их в DOM.

## Правило №2: импорт клиентского в серверном

Серверный компонент **может** импортировать и рендерить клиентский компонент напрямую. Это стандартный паттерн.

```tsx
// app/page.tsx — серверный компонент
import Counter from "@/components/Counter"; // клиентский — ОК

export default async function Page() {
  const posts = await getPosts();
  return (
    <div>
      <h1>Posts</h1>
      <Counter /> {/* Клиентский компонент внутри серверного — работает */}
      <ul>
        {posts.map(p => <li key={p.id}>{p.title}</li>)}
      </ul>
    </div>
  );
}
```

**Что происходит при сборке:**
1. Сервер рендерит `<Page />` и встречает `<Counter />`
2. Вместо HTML-разметки Counter в RSC-поток записывается **Reference** — указатель на клиентский бандл
3. На клиенте Next.js загружает JS-бандл Counter и гидрирует его

## Правило №3: импорт серверного в клиентском — ЗАПРЕЩЁН

Клиентский компонент **не может** импортировать серверный компонент напрямую. Это приведёт к ошибке сборки.

```tsx
// components/ClientWrapper.tsx — клиентский компонент
"use client";

// ❌ ОШИБКА: нельзя импортировать серверный компонент в клиентский
import ServerData from "./ServerData";

export default function ClientWrapper() {
  return (
    <div>
      <ServerData /> {/* Build error! */}
    </div>
  );
}
```

```tsx
// components/ServerData.tsx — серверный компонент (без "use client")
export default async function ServerData() {
  const data = await fetch("https://api.example.com/data").then(r => r.json());
  return <pre>{JSON.stringify(data)}</pre>;
}
```

**Ошибка сборки:**
```
Error: Cannot import server component "ServerData" into client component "ClientWrapper".
```

**Почему:** серверный компонент может использовать Node.js API, читать файлы, обращаться к БД напрямую. Клиентский компонент выполняется в браузере — у него нет доступа к серверному рантайму. Если бы это было разрешено, серверный код «утёк» бы в браузер.

## Правило №4: серверный как children в клиентский

Это **разрешено** и является ключевым паттерном композиции. Родительский серверный компонент создаёт серверный дочерний компонент и передаёт его как `children` в клиентский.

```tsx
// app/page.tsx — серверный компонент
import ClientShell from "@/components/ClientShell";
import ServerChart from "@/components/ServerChart";

export default async function Page() {
  const data = await fetchAnalytics();
  return (
    <ClientShell>
      {/* ServerChart — серверный, передаётся как children в ClientShell — клиентский */}
      <ServerChart data={data} />
    </ClientShell>
  );
}
```

```tsx
// components/ClientShell.tsx — клиентский
"use client";

import { useState } from "react";

export default function ClientShell({ children }: { children: React.ReactNode }) {
  const [collapsed, setCollapsed] = useState(false);
  return (
    <div>
      <button onClick={() => setCollapsed(c => !c)}>Toggle</button>
      {!collapsed && children}
    </div>
  );
}
```

```tsx
// components/ServerChart.tsx — серверный
export default async function ServerChart({ data }: { data: AnalyticsData }) {
  // Тяжёлые вычисления на сервере, доступ к БД, и т.д.
  const processed = await processChartData(data);
  return <div className="chart">{/* SVG-разметка */}</div>;
}
```

**Почему работает:** Серверный компонент `<Page />` рендерит `<ServerChart />` на сервере, получая готовый HTML. Этот HTML передаётся в `<ClientShell />` как часть `children`. Клиентский компонент просто вставляет готовый HTML — ему не нужно «знать» о серверной природе Chart.

## Сводная таблица

| Действие | Разрешено? | Пример |
|---|---|---|
| Серверный импортирует клиентский | ✅ Да | `import Counter from "./Counter"` в серверном |
| Серверный рендерит клиентский | ✅ Да | `<Counter />` внутри серверного |
| Клиентский импортирует серверный | ❌ Нет | `import ServerData from "./ServerData"` в `"use client"` |
| Клиентский рендерит серверный напрямую | ❌ Нет | `<ServerData />` внутри `"use client"` |
| Серверный передаёт серверный как `children` в клиентский | ✅ Да | `<Modal><ServerChild /></Modal>` в серверном |
| Клиентский передаёт клиентский как `children` в серверный | ✅ Да | `<Server><ClientChild /></Server>` в серверном |
| Клиентский принимает `children` от серверного | ✅ Да | `({ children }) => <div>{children}</div>` в `"use client"` |

## Паттерн «Interleaving» — чередование компонентов

На практике серверные и клиентские компоненты чередуются. Стратегия: **клиентские компоненты как можно «ниже» в дереве**, серверные — как можно «выше».

```
app/page.tsx (Server)
├── Header (Server)
│   └── SearchBar (Client) ← интерактивность
├── Sidebar (Server)
│   └── NavLinks (Client) ← интерактивность
└── Content (Server)
    ├── ArticleBody (Server) ← тяжёлый контент, остаётся серверным
    └── CommentSection (Client) ← интерактивность
        └── CommentList (Client)
```

### Правильный подход — «поднять серверную часть наверх»

```tsx
// app/page.tsx — серверный
import InteractiveShell from "@/components/InteractiveShell";
import HeavyServerContent from "@/components/HeavyServerContent";

export default async function Page() {
  return (
    <InteractiveShell>
      <HeavyServerContent />
    </InteractiveShell>
  );
}
```

### Неправильный подход — попытка импорта серверного в клиентском

```tsx
// ❌ НЕПРАВИЛЬНО
// components/InteractiveShell.tsx
"use client";

import HeavyServerContent from "./HeavyServerContent"; // Ошибка сборки!

export default function InteractiveShell() {
  return (
    <div>
      <button>Toggle</button>
      <HeavyServerContent />
    </div>
  );
}
```

## Почему так работает: модель сериализации RSC

React Server Components используют протокол сериализации, который разделяет компоненты на две категории:

1. **Server Module References** — указатели на серверные модули. Рендерятся на сервере, в клиентский бандл не попадают.
2. **Client Module References** — указатели на клиентские модули. На сервере создаётся «заглушка», на клиенте загружается реальный JS.

Когда серверный компонент передаёт `children` в клиентский:
- `children` — это уже **сериализованный RSC-поток** (готовые React-элементы)
- Клиентский компонент получает их как «пропсы» — обычные данные
- Ему не нужно знать, серверные они или клиентские — они уже отрендерены

Когда клиентский компонент пытается импортировать серверный:
- Next.js пытается разрешить импорт на этапе сборки
- Обнаруживает, что целевой модуль — серверный (не имеет `"use client"`)
- Выдаёт ошибку, потому что серверный модуль может содержать Node.js API

## Практические рецепты

### Рецепт 1: Клиентская обёртка с серверным содержимым

```tsx
// app/page.tsx
import Tabs from "@/components/Tabs";
import ServerContent from "@/components/ServerContent";

export default async function Page() {
  return (
    <Tabs
      items={[
        { label: "Overview", content: <ServerContent type="overview" /> },
        { label: "Details", content: <ServerContent type="details" /> },
      ]}
    />
  );
}
```

```tsx
// components/Tabs.tsx
"use client";

import { useState } from "react";

interface TabItem {
  label: string;
  content: React.ReactNode;
}

export default function Tabs({ items }: { items: TabItem[] }) {
  const [active, setActive] = useState(0);
  return (
    <div>
      <div className="tab-headers">
        {items.map((item, i) => (
          <button key={i} onClick={() => setActive(i)}>{item.label}</button>
        ))}
      </div>
      <div className="tab-content">{items[active].content}</div>
    </div>
  );
}
```

### Рецепт 2: Серверный компонент передаёт callback в клиентский

Если клиентскому компоненту нужно вызвать серверную логику, используйте **Server Actions** вместо импорта серверного компонента:

```tsx
// app/page.tsx — серверный
import DeleteButton from "@/components/DeleteButton";
import { deleteItem } from "@/app/actions";

export default function Page() {
  return <DeleteButton action={deleteItem} />;
}
```

```tsx
// components/DeleteButton.tsx — клиентский
"use client";

export default function DeleteButton({ action }: { action: (id: string) => Promise<void> }) {
  return (
    <button onClick={() => action("item-1")}>
      Delete
    </button>
  );
}
```

```tsx
// app/actions.ts — серверная функция
"use server";

export async function deleteItem(id: string) {
  await db.items.delete({ where: { id } });
}
```

### Рецепт 3: Делегирование через пропсы (без children)

Если нельзя использовать `children`, передавайте серверный контент через обычные пропсы:

```tsx
// app/page.tsx — серверный
import ClientCard from "@/components/ClientCard";
import ServerAvatar from "@/components/ServerAvatar";

export default async function Page() {
  const user = await getUser();
  return (
    <ClientCard
      avatar={<ServerAvatar userId={user.id} />}
      title={user.name}
    />
  );
}
```

```tsx
// components/ClientCard.tsx — клиентский
"use client";

import { useState } from "react";

export default function ClientCard({
  avatar,
  title,
}: {
  avatar: React.ReactNode;
  title: string;
}) {
  const [expanded, setExpanded] = useState(false);
  return (
    <div onClick={() => setExpanded(e => !e)}>
      {avatar}
      <h2>{title}</h2>
    </div>
  );
}
```

## Антипаттерны

### Антипаттерн 1: «use client» на весь файл, когда нужен только один хук

```tsx
// ❌ Плохо: весь файл стал клиентским из-за одного useState
"use client";

export default async function Page() {
  const data = await fetch("..."); // Это вообще не сработает — "use client"
  // ...
}
```

Решение: вынести интерактивную часть в отдельный клиентский компонент.

### Антипаттерн 2: Попытка обойти запрет через динамический импорт

```tsx
// ❌ Не сработает
"use client";

import dynamic from "next/dynamic";
const ServerComp = dynamic(() => import("./ServerComp")); // Ошибка!
```

`dynamic` не позволяет загружать серверные компоненты в клиентские.

### Антипаттерн 3: Передача несериализуемых данных через children

```tsx
// ❌ Плохо: передача функций или классов через children
<ClientWrapper>
  <ServerComponent callback={() => console.log("hi")} />
</ClientWrapper>
```

RSC-поток сериализуется в JSON. Функции, классы, Date-объекты и Symbols не передаются. Используйте Server Actions для callback'ов.
