# Сборка и CI/CD — от бандлеров до автоматизации деплоя

## Содержание

1. [Почему фронтендеру нужно понимать сборку](#почему-фронтендеру-нужно-понимать-сборку)
2. [Эволюция: от скриптов в HTML до бандлеров](#эволюция-от-скриптов-в-html-до-бандлеров)
3. [Что такое бандлер и зачем он нужен](#что-такое-бандлер-и-зачем-он-нужен)
4. [Webpack — как работает и почему стал стандартом](#webpack--как-работает-и-почему-стал-стандартом)
5. [Vite — почему появился и чем отличается](#vite--почему-появился-и-чем-отличается)
6. [Code splitting и lazy loading](#code-splitting-и-lazy-loading)
7. [Tree shaking — удаление мёртвого кода](#tree-shaking--удаление-мёртвого-кода)
8. [CI/CD — что это и зачем нужно фронтендеру](#cicd--что-это-и-зачем-нужно-фронтендеру)
9. [GitHub Actions — автоматизация для фронтенда](#github-actions--автоматизация-для-фронтенда)
10. [Docker для фронтенда](#docker-для-фронтенда)
11. [Preview deployments — деплой каждого PR](#preview-deployments--деплой-каждого-pr)
12. [Лучшие практики и оптимизация](#лучшие-практики-и-оптимизация)

---

## Почему фронтендеру нужно понимать сборку

Многие фронтенд-разработчики работают с React, Vue или другими фреймворками, не задумываясь о том, что происходит между написанием кода и его запуском в браузере. Вы пишете JSX, импортируете модули, используете TypeScript — но браузер не понимает ни JSX, ни ES-модули (в старых браузерах), ни TypeScript. Кто-то должен это всё превратить в обычный JavaScript, CSS и HTML.

Этот «кто-то» — **бандлер** (bundle — связка, пучок). Он берёт ваш код со всеми зависимостями, трансформирует его в понятный браузеру формат и оптимизирует для production.

Но на этом работа не заканчивается. Код нужно протестировать, проверить, собрать на сервере и доставить на production. Этот процесс называется **CI/CD** (Continuous Integration / Continuous Deployment). Если вы понимаете, как работает сборка и деплой, вы можете:

- Оптимизировать размер бандла и скорость загрузки
- Настраивать автоматические тесты в CI
- Понимать, почему ваш код работает в dev, но ломается в prod
- Эффективно работать в команде с другими разработчиками и DevOps-инженерами

На собеседовании вопросы про сборку и CI/CD задают всё чаще, особенно для middle+ позиций. Понимание этих процессов показывает, что вы видите картину целиком, а не только свой кусочек кода.

---

## Эволюция: от скриптов в HTML до бандлеров

Чтобы понять, зачем нужны бандлеры, давайте посмотрим, как развивалась фронтенд-разработка.

### Эра простых скриптов (до 2010)

В начале был хаос. JavaScript-код подключался через теги `<script>` в HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <script src="jquery.js"></script>
  <script src="underscore.js"></script>
  <script src="app.js"></script>
</head>
<body>
  <!-- ... -->
</body>
</html>
```

Проблемы этого подхода:

**1. Глобальная область видимости.** Все скрипты работают в одном глобальном пространстве имён. Если две библиотеки определяют функцию с одинаковым именем — конфликт.

```js
// jquery.js
function $(selector) { /* ... */ }

// app.js
function $(selector) { /* конфликт! */ }
```

**2. Порядок загрузки критичен.** Скрипты выполняются в порядке появления в HTML. Если `app.js` зависит от `jquery.js`, но загружается первым — ошибка.

**3. Нет модульности.** Нет способа сказать «этот файл зависит от того файла». Все зависимости неявные — вы должны помнить, что `app.js` использует jQuery, и загружать jQuery первым.

**4. Нет минификации и оптимизации.** Код загружается как есть, со всеми комментариями и пробелами.

### Появление модульных систем (2010–2015)

Чтобы решить проблему глобальной области видимости и неявных зависимостей, появились модульные системы.

**CommonJS** (используется в Node.js):

```js
// math.js
module.exports = {
  add: function(a, b) { return a + b; }
};

// app.js
const math = require('./math');
console.log(math.add(2, 3)); // 5
```

**AMD (Asynchronous Module Definition)** (использовался в браузерах через RequireJS):

```js
// math.js
define(function() {
  return {
    add: function(a, b) { return a + b; }
  };
});

// app.js
define(['./math'], function(math) {
  console.log(math.add(2, 3)); // 5
});
```

Обе системы решали проблему модульности, но требовали либо серверной среды (CommonJS), либо специальной библиотеки для загрузки модулей (AMD). Ни одна не работала нативно в браузере без дополнительных инструментов.

### Эра бандлеров (2014–настоящее время)

Бандлеры решили все проблемы сразу: они берут модульный код (CommonJS, AMD, ES-модули), разрешают все зависимости, объединяют их в один или несколько файлов (бандлов) и оптимизируют для production.

Первым популярным бандлером стал **Browserify** (2011) — он позволял использовать CommonJS-модули в браузере. Но настоящий прорыв случился с появлением **Webpack** (2014) — инструмента, который стал стандартом индустрии.

---

## Что такое бандлер и зачем он нужен

**Бандлер** — это инструмент, который берёт ваш код со всеми зависимостями (модулями), анализирует граф зависимостей, трансформирует код и объединяет его в один или несколько файлов (бандлов), оптимизированных для браузера.

### Что делает бандлер

**1. Разрешает зависимости.** Когда вы пишете `import React from 'react'`, бандлер находит модуль `react` в `node_modules`, анализирует его зависимости (и зависимости его зависимостей), и строит полный граф зависимостей.

**2. Трансформирует код.** Бандлер использует **лоадеры** (loaders) или **плагины** (plugins) для трансформации кода:
- JSX → JavaScript
- TypeScript → JavaScript
- Sass/Less → CSS
- Современные синтаксические конструкции → старый синтаксис (для совместимости)

**3. Объединяет модули.** Все модули объединяются в один или несколько файлов. Это уменьшает количество HTTP-запросов (что важно для производительности).

**4. Оптимизирует код.** Бандлер удаляет мёртвый код (tree shaking), минифицирует результат, разделяет код на части (code splitting) для ленивой загрузки.

**5. Добавляет source maps.** Source maps позволяют браузеру показать исходный код (до трансформации) в DevTools, что упрощает отладку.

### Аналогия

Представьте, что вы пишете книгу. У вас есть:
- Главы (ваши модули)
- Ссылки на другие книги (импорты)
- Черновики с пометками (TypeScript, JSX)
- Повторяющиеся фрагменты (дублирующийся код)

Бандлер — это редактор, который:
- Находит все ссылки и проверяет, что они корректны
- Превращает черновики в чистый текст
- Удаляет повторяющиеся фрагменты
- Объединяет всё в одну книгу (или несколько томов)
- Создаёт указатель (source maps), чтобы вы могли найти исходные черновики

Без бандлера вам пришлось бы вручную собирать все фрагменты, проверять ссылки и оптимизировать текст. С бандлером вы просто пишете код, а он делает всю остальную работу.

---

## Webpack — как работает и почему стал стандартом

**Webpack** — это самый популярный бандлер для JavaScript-приложений. Он появился в 2014 году и с тех пор стал стандартом индустрии. Даже если вы используете Vite (о котором мы поговорим позже), понимание Webpack важно, потому что многие проекты всё ещё используют его, и многие концепции бандлеров пришли из Webpack.

### Как работает Webpack

Webpack строит **граф зависимостей** (dependency graph). Он начинает с «точки входа» (entry point) — обычно это ваш главный файл (например, `src/index.js`) — и рекурсивно обходит все импорты, строя дерево зависимостей.

```
src/index.js
  ├── import React from 'react'
  │   └── ... (зависимости React)
  ├── import App from './App'
  │   ├── import Header from './Header'
  │   │   └── import './header.css'
  │   └── import Footer from './Footer'
  └── import './styles.css'
```

Webpack обрабатывает каждый файл в этом графе, применяя соответствующие лоадеры (например, `babel-loader` для JavaScript, `css-loader` для CSS), и объединяет всё в один или несколько бандлов.

### Концепция лоадеров

Webpack сам по себе понимает только JavaScript и JSON. Для всего остального нужны **лоадеры** — плагины, которые трансформируют файлы.

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.jsx?$/, // для файлов .js и .jsx
        use: 'babel-loader', // трансформировать через Babel
        exclude: /node_modules/
      },
      {
        test: /\.css$/, // для файлов .css
        use: ['style-loader', 'css-loader'] // сначала css-loader, потом style-loader
      },
      {
        test: /\.(png|jpg|gif)$/, // для изображений
        type: 'asset/resource' // встроить как base64 или скопировать
      }
    ]
  }
};
```

Лоадеры применяются **справа налево** (или снизу вверх). В примере с CSS сначала `css-loader` обрабатывает CSS-файл (разрешает `@import` и `url()`), затем `style-loader` вставляет результат в `<style>` тег в HTML.

### Концепция плагинов

Если лоадеры трансформируют отдельные файлы, то **плагины** выполняют более широкие задачи: оптимизацию бандла, генерацию HTML-файлов, копирование статических файлов и т.д.

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html' // генерирует HTML на основе шаблона
    }),
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css' // извлекает CSS в отдельные файлы
    })
  ]
};
```

### Почему Webpack стал стандартом

**1. Универсальность.** Webpack может обрабатывать любые типы файлов (JavaScript, CSS, изображения, шрифты) благодаря системе лоадеров.

**2. Экосистема.** Тысячи плагинов и лоадеров для любых задач: минификация, оптимизация изображений, извлечение CSS, генерация service workers и т.д.

**3. Code splitting.** Webpack может автоматически разделять бандл на части, которые загружаются по требованию (ленивая загрузка).

**4. Hot Module Replacement (HMR).** Изменения в коде применяются без полной перезагрузки страницы, что ускоряет разработку.

**5. Оптимизация для production.** Tree shaking, минификация, оптимизация изображений — всё из коробки.

### Проблемы Webpack

Несмотря на популярность, Webpack имеет серьёзные недостатки:

**1. Медленный старт.** При запуске dev-сервера Webpack должен обработать все файлы проекта, прежде чем вы сможете открыть страницу. Для больших проектов это может занять десятки секунд.

**2. Медленный HMR.** При изменении файла Webpack пересобирает часть бандла, что может быть медленным для больших проектов.

**3. Сложная конфигурация.** Webpack требует много ручной настройки. Даже простой проект может потребовать сотни строк конфигурации.

Эти проблемы привели к появлению новых инструментов — в частности, Vite.

---

## Vite — почему появился и чем отличается

**Vite** (произносится как «вит», что означает «быстрый» по-французски) — это инструмент сборки, созданный Эваном Ю (автором Vue.js) в 2020 году. Он решает проблемы Webpack, используя современные возможности браузеров.

### Ключевая идея: нативные ES-модули

Современные браузеры поддерживают **нативные ES-модули** (Native ESM). Это означает, что браузер может сам загружать модули через `import` без необходимости бандлера:

```html
<script type="module">
  import { add } from './math.js';
  console.log(add(2, 3));
</script>
```

Vite использует эту возможность. В режиме разработки он **не создаёт бандл**. Вместо этого он отдаёт файлы по одному, и браузер сам загружает нужные модули через `import`.

### Как работает Vite

**Режим разработки:**

1. Вы запускаете `vite` (dev-сервер)
2. Vite не обрабатывает весь проект — он ждёт запросы от браузера
3. Браузер загружает `index.html`, который содержит `<script type="module" src="/src/main.js">`
4. Браузер запрашивает `main.js`, Vite трансформирует его на лету (например, удаляет TypeScript)
5. Браузер видит `import React from 'react'` и запрашивает `react`
6. Vite находит модуль `react` в `node_modules`, трансформирует его и отдаёт
7. Процесс повторяется для всех зависимостей

**Преимущества:**
- **Мгновенный старт.** Не нужно обрабатывать весь проект — только запрашиваемые файлы
- **Быстрый HMR.** Изменения применяются за миллисекунды, потому что нужно обновить только один модуль
- **Простая конфигурация.** Vite работает из коробки с минимальной настройкой

**Production-сборка:**

Для production Vite использует **Rollup** (другой бандлер) для создания оптимизированного бандла. Rollup лучше подходит для production, потому что:
- Создаёт более эффективный бандл (меньше накладных расходов)
- Лучше поддерживает tree shaking
- Поддерживает несколько форматов (ESM, CommonJS, UMD)

### Сравнение Webpack и Vite

| Характеристика | Webpack | Vite |
|---|---|---|
| Режим разработки | Создаёт бандл при запуске | Использует нативные ES-модули |
| Время старта | Медленно (секунды-минуты) | Мгновенно |
| HMR | Медленно для больших проектов | Очень быстро |
| Production-сборка | Встроенный бандлер | Использует Rollup |
| Конфигурация | Сложная, много ручной настройки | Простая, минимум настройки |
| Экосистема плагинов | Огромная | Растущая, но меньше |
| Поддержка legacy-браузеров | Отличная | Требует дополнительной настройки |

### Когда использовать что

**Используйте Vite, если:**
- Вы начинаете новый проект
- Вам важна скорость разработки
- Вы не поддерживаете старые браузеры (IE11)
- Вы используете современный фреймворк (React, Vue, Svelte)

**Используйте Webpack, если:**
- У вас уже есть проект на Webpack (миграция может быть дорогой)
- Вам нужна поддержка старых браузеров
- У вас сложная кастомная конфигурация, которую сложно воспроизвести в Vite
- Вы используете специфические плагины, которых нет в Vite

В 2026 году Vite стал де-факто стандартом для новых проектов. Create React App (который использовал Webpack) официально устарел, и даже официальная документация React рекомендует использовать Vite или Next.js.

---

## Code splitting и lazy loading

**Code splitting** — это техника разделения бандла на меньшие части (чанки), которые загружаются по требованию. Вместо одного большого файла `bundle.js` вы получаете несколько меньших файлов, которые загружаются только когда нужны.

### Зачем это нужно

Представьте, что у вас есть приложение с тремя страницами: `Home`, `Dashboard` и `Settings`. Если все три страницы включены в один бандл, пользователь загружает весь код при первом посещении, даже если он собирается смотреть только `Home`.

С code splitting каждая страница загружается отдельно:
- Пользователь заходит на `Home` → загружается только код `Home`
- Пользователь переходит на `Dashboard` → загружается код `Dashboard`
- Пользователь переходит на `Settings` → загружается код `Settings`

Это уменьшает время начальной загрузки и улучшает производительность.

### Как это работает в React

В React code splitting реализуется через `React.lazy` и `Suspense`:

```jsx
import { lazy, Suspense } from 'react';

// Ленивая загрузка компонента
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <div>
      <Home /> {/* Загружается сразу */}
      
      <Suspense fallback={<div>Loading...</div>}>
        <Dashboard /> {/* Загружается по требованию */}
      </Suspense>
      
      <Suspense fallback={<div>Loading...</div>}>
        <Settings /> {/* Загружается по требованию */}
      </Suspense>
    </div>
  );
}
```

Когда React встречает `lazy(() => import('./Dashboard'))`, он не загружает компонент сразу. Вместо этого он создаёт «ленивый» компонент, который загружается только при первом рендере. `Suspense` показывает fallback (например, спиннер), пока компонент загружается.

### Маршрутизация и code splitting

Code splitting особенно эффективен в сочетании с маршрутизацией. Каждая страница загружается только при переходе на неё:

```jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { lazy, Suspense } from 'react';

const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

Теперь каждая страница — это отдельный чанк, который загружается только при переходе на соответствующий маршрут.

### Автоматическое разделение в Vite и Webpack

Современные бандлеры могут автоматически разделять код:

**Vite** автоматически разделяет код из `node_modules` на отдельные чанки (vendor chunks). Это означает, что библиотеки (например, React) загружаются отдельно от вашего кода и кэшируются браузером.

**Webpack** использует `SplitChunksPlugin` для автоматического разделения:

```js
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all', // разделять все чанки
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/, // модули из node_modules
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};
```

Это создаёт отдельный чанк `vendors.js` для всех библиотек из `node_modules`, что улучшает кэширование.

---

## Tree shaking — удаление мёртвого кода

**Tree shaking** — это процесс удаления неиспользуемого кода из бандла. Если вы импортируете функцию из модуля, но не используете её, tree shaking удалит её из финального бандла.

### Как это работает

Tree shaking работает только с **ES-модулями** (статическими импортами/экспортами), потому что они позволяют статически анализировать код (без выполнения).

```js
// utils.js
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}

export function multiply(a, b) {
  return a * b;
}

// app.js
import { add } from './utils';

console.log(add(2, 3));
```

В этом примере импортируется только `add`. Функции `subtract` и `multiply` не используются, поэтому tree shaking удалит их из бандла.

### Почему это важно для ES-модулей

CommonJS (используется в Node.js) не поддерживает tree shaking, потому что импорты динамические:

```js
// CommonJS
const utils = require('./utils'); // загружает ВСЁ, даже если нужно только add
```

ES-модули используют статические импорты, которые можно проанализировать без выполнения кода:

```js
// ES-модули
import { add } from './utils'; // можно statically определить, что нужно только add
```

### Side effects и package.json

Некоторые модули имеют **побочные эффекты** (side effects) — например,.polyfill'ы, которые добавляют методы в глобальные объекты. Такие модули нельзя удалять через tree shaking, даже если они не используются явно.

Чтобы бандлер знал, какие модули имеют побочные эффекты, используется поле `sideEffects` в `package.json`:

```json
{
  "name": "my-library",
  "sideEffects": [
    "./src/polyfill.js" // этот файл имеет побочные эффекты
  ]
}
```

Если `sideEffects` не указан, бандлер предполагает, что все модули могут иметь побочные эффекты, и не удаляет их. Если `sideEffects: false`, бандлер агрессивно удаляет неиспользуемый код.

### Практические советы

**1. Используйте именованные импорты вместо импорта всего модуля:**

```js
// ❌ Плохо — импортирует всё
import _ from 'lodash';
_.get(obj, 'path.to.value');

// ✅ Хорошо — импортирует только нужное
import { get } from 'lodash';
get(obj, 'path.to.value');
```

**2. Используйте библиотеки, оптимизированные для tree shaking:**

```js
// ❌ Плохо — lodash не оптимизирован для tree shaking
import { get } from 'lodash';

// ✅ Хорошо — lodash-es оптимизирован
import { get } from 'lodash-es';

// ✅ Ещё лучше — используйте отдельные пакеты
import get from 'lodash.get';
```

**3. Проверяйте размер бандла.** Используйте инструменты вроде `webpack-bundle-analyzer` или `rollup-plugin-visualizer` для анализа того, что попало в бандл.

---

## CI/CD — что это и зачем нужно фронтендеру

**CI/CD** (Continuous Integration / Continuous Deployment) — это набор практик для автоматизации сборки, тестирования и деплоя кода. Если вы работаете в команде (или даже в одиночку над серьёзным проектом), CI/CD критически важен.

### Что такое Continuous Integration (CI)

**Continuous Integration** — это практика автоматической сборки и тестирования кода при каждом изменении. Когда вы делаете push в репозиторий, CI-сервер:

1. Забирает последнюю версию кода
2. Устанавливает зависимости (`npm install`)
3. Запускает линтеры (`npm run lint`)
4. Запускает тесты (`npm test`)
5. Собирает проект (`npm run build`)

Если что-то падает — вы получаете уведомление. Это позволяет обнаруживать проблемы рано, до того как они попадут в production.

### Что такое Continuous Deployment (CD)

**Continuous Deployment** — это практика автоматического деплоя кода в production после успешного прохождения CI. Когда CI проходит успешно, CD-сервер:

1. Собирает production-версию (`npm run build`)
2. Загружает артефакты на сервер (или в CDN)
3. Обновляет конфигурацию (если нужно)
4. Перезапускает сервис (если нужно)

В результате код попадает в production автоматически, без ручного вмешательства.

### Зачем это фронтендеру

Многие фронтенд-разработчики думают, что CI/CD — это забота DevOps-инженеров. Но это не так. Понимание CI/CD помогает вам:

**1. Понимать, почему код работает в dev, но ломается в prod.** Различия в окружении (Node.js версии, переменные окружения, оптимизации бандлера) могут вызывать странные баги.

**2. Настраивать автоматические тесты.** Если вы не настроите тесты в CI, они будут запускаться только локально. Это означает, что баги могут попасть в production незамеченными.

**3. Оптимизировать время сборки.** Если сборка занимает 10 минут, разработчики будут ждать по 10 минут после каждого push. Понимание того, как работает сборка, помогает её ускорить.

**4. Работать в команде эффективно.** Если каждый разработчик использует своё окружение, результаты могут отличаться. CI/CD гарантирует, что все работают в одинаковых условиях.

### Популярные CI/CD-системы

**GitHub Actions** — встроенная CI/CD-система GitHub. Идеально подходит для проектов, размещённых на GitHub. Конфигурация через YAML-файлы в `.github/workflows/`.

**GitLab CI/CD** — встроенная CI/CD-система GitLab. Аналогична GitHub Actions, но для GitLab.

**Jenkins** — старый, но мощный инструмент. Требует больше настройки, но очень гибкий.

**CircleCI** — облачная CI/CD-система с хорошей интеграцией с GitHub и GitLab.

**Vercel / Netlify** — специализированные платформы для фронтенда. Автоматически деплоят при каждом push, поддерживают preview deployments.

В этой статье мы сосредоточимся на **GitHub Actions**, потому что это самый популярный выбор для фронтенд-проектов в 2026 году.

---

## GitHub Actions — автоматизация для фронтенда

**GitHub Actions** — это система автоматизации, встроенная в GitHub. Она позволяет запускать задачи (workflows) при определённых событиях: push, pull request, schedule и т.д.

### Основные концепции

**Workflow** — это автоматизированный процесс, определённый в YAML-файле. Workflow состоит из одного или нескольких jobs.

**Job** — это набор шагов (steps), которые выполняются на одном runner'е (виртуальной машине). Jobs могут выполняться параллельно или последовательно.

**Step** — это отдельная задача в job. Step может быть командой shell или action'ом (переиспользуемым блоком).

**Action** — это переиспользуемый блок кода, который выполняет определённую задачу. GitHub имеет marketplace с тысячами готовых actions.

### Пример workflow для React-приложения

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest # операционная система runner'а
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4 # готовый action для checkout кода
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20' # версия Node.js
          cache: 'npm' # кэширование node_modules
      
      - name: Install dependencies
        run: npm ci # ci вместо install для CI/CD
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/
```

Разберём этот workflow по частям:

**`on:`** — определяет, при каких событиях запускается workflow. В этом примере workflow запускается при push в ветки `main` и `develop`, и при создании pull request в ветку `main`.

**`jobs:`** — определяет jobs, которые будут выполнены. В этом примере один job `build-and-test`.

**`runs-on: ubuntu-latest`** — указывает, на какой операционной системе выполнять job. GitHub предоставляет runner'ы с Ubuntu, Windows и macOS.

**`steps:`** — список шагов, которые выполняются последовательно. Каждый step может быть:
- Командой shell (`run: npm test`)
- Action'ом (`uses: actions/checkout@v4`)

**`actions/checkout@v4`** — готовый action, который загружает код из репозитория на runner.

**`actions/setup-node@v4`** — готовый action, который устанавливает Node.js указанной версии. Параметр `cache: 'npm'` включает кэширование `node_modules`, что ускоряет последующие запуски.

**`npm ci`** — команда для установки зависимостей в CI. Она отличается от `npm install` тем, что:
- Удаляет `node_modules` перед установкой
- Устанавливает версии точно по `package-lock.json`
- Быстрее, чем `npm install`
- Падает, если `package-lock.json` не синхронизирован с `package.json`

### Матрица тестирования

Если вы хотите тестировать на разных версиях Node.js или операционных системах, используйте матрицу:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node-version: [18, 20, 22]
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

Это создаст 6 jobs (2 ОС × 3 версии Node.js), каждая из которых тестирует код в своём окружении.

### Кэширование зависимостей

Установка зависимостей — одна из самых медленных частей CI. Кэширование может значительно ускорить workflow:

```yaml
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

Или используйте встроенное кэширование `actions/setup-node`:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm' # автоматически кэширует ~/.npm
```

### Деплой на Vercel / Netlify

Если вы используете Vercel или Netlify, вы можете автоматизировать деплой:

```yaml
- name: Deploy to Vercel
  uses: amondnet/vercel-action@v25
  with:
    vercel-token: ${{ secrets.VERCEL_TOKEN }}
    vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
    vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
    vercel-args: '--prod'
```

Секреты (`VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID`) хранятся в Settings → Secrets вашего репозитория на GitHub.

---

## Docker для фронтенда

**Docker** — это платформа для контейнеризации приложений. Контейнер — это изолированная среда, которая содержит всё необходимое для запуска приложения: код, зависимости, переменные окружения, конфигурационные файлы.

### Зачем фронтенду Docker

Многие фронтенд-разработчики думают, что Docker нужен только для бэкенда. Но Docker полезен и для фронтенда:

**1. Консистентность окружения.** Если приложение работает на вашем компьютере, оно будет работать и на CI-сервере, и в production. Нет проблем «у меня работает».

**2. Упрощение CI/CD.** Вместо настройки Node.js, зависимостей и конфигурации на каждом сервере, вы просто запускаете контейнер.

**3. Локальная разработка.** Вы можете запустить всё приложение (фронтенд + бэкенд + база данных) в Docker Compose одной командой.

**4. Микросервисы.** Если ваш фронтенд состоит из нескольких микросервисов (например, через Module Federation), каждый микросервис может быть в своём контейнере.

### Dockerfile для React-приложения

```dockerfile
# Этап 1: Сборка
FROM node:20-alpine AS builder

WORKDIR /app

# Копируем package.json и package-lock.json
COPY package*.json ./

# Устанавливаем зависимости
RUN npm ci

# Копируем исходный код
COPY . .

# Собираем production-версию
RUN npm run build

# Этап 2: Production-образ
FROM nginx:alpine

# Копируем собранные файлы из builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Копируем конфигурацию nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Открываем порт 80
EXPOSE 80

# Запускаем nginx
CMD ["nginx", "-g", "daemon off;"]
```

Этот Dockerfile использует **multi-stage build** (многоэтапную сборку):

**Этап 1 (builder):** Устанавливает Node.js, копирует зависимости, собирает приложение. Результат — папка `dist/` с production-версией.

**Этап 2 (production):** Использует nginx для раздачи статических файлов. Копирует только `dist/` из builder, без исходного кода и `node_modules`.

Это уменьшает размер финального образа: вместо ~500 МБ (Node.js + зависимости) получаем ~20 МБ (nginx + статические файлы).

### Конфигурация nginx

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Для SPA: все маршруты перенаправляются на index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Кэширование статических файлов
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Gzip-сжатие
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
}
```

Ключевой момент для SPA (Single Page Application) — это `try_files $uri $uri/ /index.html`. Это означает, что если пользователь запрашивает `/dashboard`, nginx отдаст `index.html`, а маршрутизация обработается на клиенте.

### Docker Compose для локальной разработки

Если ваше приложение состоит из фронтенда и бэкенда, вы можете использовать Docker Compose для запуска всего стека:

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
  
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
  
  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Теперь одна команда `docker-compose up` запускает фронтенд, бэкенд и базу данных в изолированных контейнерах.

---

## Preview deployments — деплой каждого PR

**Preview deployments** — это практика автоматического деплоя каждой ветки или pull request на отдельный URL. Это позволяет тестировать изменения до мержа в основную ветку.

### Как это работает

Когда вы создаёте pull request, CI/CD-система:

1. Собирает проект из ветки PR
2. Деплоит его на уникальный URL (например, `pr-123.myapp.com`)
3. Добавляет комментарий в PR с ссылкой на preview

Ревьюеры могут открыть preview, протестировать изменения и оставить feedback до мержа.

### Пример с Vercel

Vercel автоматически создаёт preview deployments для каждого PR:

```yaml
# .github/workflows/preview.yml
name: Preview Deployment

on:
  pull_request:
    branches: [main]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          # Без --prod, это preview deployment
```

Vercel автоматически создаст уникальный URL для этого PR и добавит комментарий с ссылкой.

### Пример с Netlify

Netlify также поддерживает preview deployments из коробки. Добавьте файл `netlify.toml`:

```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

Netlify автоматически создаст preview deployment для каждого PR и добавит комментарий с ссылкой.

### Преимущества preview deployments

**1. Раннее обнаружение проблем.** Ревьюеры могут протестировать изменения в реальной среде, а не только читать код.

**2. Лучшая коммуникация.** Дизайнеры, продукт-менеджеры и QA могут увидеть изменения своими глазами, а не представлять их по описанию.

**3. Безопасность.** Изменения тестируются до мержа, что снижает риск поломки production.

**4. Быстрая итерация.** Разработчик может вносить изменения, push'ить и сразу видеть результат на preview URL.

---

## Лучшие практики и оптимизация

### Оптимизация размера бандла

**1. Анализируйте бандл.** Используйте `webpack-bundle-analyzer` или `rollup-plugin-visualizer` для понимания того, что попало в бандл:

```bash
# Для Webpack
npx webpack --profile --json > stats.json
npx webpack-bundle-analyzer stats.json

# Для Vite
npm install -D rollup-plugin-visualizer
```

**2. Используйте dynamic imports для больших библиотек.** Если библиотека нужна только на определённых страницах, загружайте её лениво:

```jsx
// Вместо
import moment from 'moment';

// Используйте
const formatDate = async (date) => {
  const moment = await import('moment');
  return moment(date).format('DD.MM.YYYY');
};
```

**3. Заменяйте тяжёлые библиотеки на лёгкие альтернативы.**

| Тяжёлая | Лёгкая | Экономия |
|---|---|---|
| moment (330 КБ) | date-fns (75 КБ) или dayjs (2 КБ) | ~90% |
| lodash (530 КБ) | lodash-es (только используемые функции) | ~80% |
| axios (55 КБ) | fetch (нативный API) | 100% |

**4. Включите gzip/brotli сжатие.** Настройте nginx или CDN для сжатия статических файлов:

```nginx
# nginx.conf
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_min_length 1000;
```

**5. Используйте кэширование.** Настройте длинные сроки кэширования для статических файлов с хэшами в именах:

```nginx
# nginx.conf
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

### Оптимизация CI/CD

**1. Кэшируйте зависимости.** Используйте `actions/cache` или встроенное кэширование `actions/setup-node`:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
```

**2. Параллельте jobs.** Если у вас есть независимые задачи (lint, test, build), запускайте их параллельно:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
  
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
  
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```

Все три jobs запустятся параллельно, что сократит общее время выполнения.

**3. Используйте матрицу для тестирования на разных версиях.** Если вы поддерживаете несколько версий Node.js, тестируйте на всех них:

```yaml
strategy:
  matrix:
    node-version: [18, 20, 22]
```

**4. Пропускайте ненужные запуски.** Если вы изменили только README, не нужно запускать тесты:

```yaml
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

**5. Используйте артефакты для передачи данных между jobs.** Если один job собирает проект, а другой деплоит, используйте артефакты:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/
  
  deploy:
    needs: build # ждёт завершения build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      - name: Deploy
        run: ./deploy.sh
```

### Мониторинг и аналитика

**1. Отслеживайте размер бандла.** Используйте GitHub Actions для проверки размера бандла при каждом PR:

```yaml
- name: Check bundle size
  uses: actions/checkout@v4
- run: npm ci
- run: npm run build
- name: Analyze bundle
  run: npx bundlesize
  env:
    BUNDLESIZE_GITHUB_TOKEN: ${{ secrets.BUNDLESIZE_GITHUB_TOKEN }}
```

**2. Используйте Lighthouse CI для проверки производительности.**

```yaml
- name: Lighthouse CI
  uses: treosh/lighthouse-ci-action@v10
  with:
    urls: |
      http://localhost:3000
    uploadArtifacts: true
```

**3. Настройте уведомления о падении CI.** Используйте Slack, Discord или email для уведомлений:

```yaml
- name: Notify on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: 'CI failed on ${{ github.ref }}'
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Заключение

Сборка и CI/CD — это не «забота DevOps», а важная часть работы фронтенд-разработчика. Понимание того, как работает бандлер, как оптимизировать размер бандла, как настроить CI/CD и как использовать Docker, делает вас более эффективным разработчиком и повышает вашу ценность на рынке.

Ключевые выводы:

1. **Бандлеры** (Webpack, Vite) превращают ваш код в оптимизированный бандл для браузера. Vite стал стандартом для новых проектов благодаря скорости и простоте.

2. **Code splitting и tree shaking** — ключевые техники оптимизации размера бандла. Используйте их для ускорения загрузки.

3. **CI/CD** автоматизирует сборку, тестирование и деплой. GitHub Actions — самый популярный выбор для фронтенд-проектов.

4. **Docker** обеспечивает консистентность окружения и упрощает деплой. Multi-stage builds позволяют создавать лёгкие production-образы.

5. **Preview deployments** позволяют тестировать изменения до мержа, что улучшает качество кода и коммуникацию в команде.

Практикуйтесь: настройте CI/CD для своего проекта, попробуйте Docker, оптимизируйте размер бандла. Эти навыки выделят вас среди других кандидатов на собеседовании.
