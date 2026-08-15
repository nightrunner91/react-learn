# Продвинутая оптимизация бандла — от анализа до реальных результатов

## Содержание

1. [Почему размер бандла имеет значение](#почему-размер-бандла-имеет-значение)
2. [Анализ бандла — как найти проблемы](#анализ-бандла--как-найти-проблемы)
3. [Tree shaking глубже — что можно, что нельзя](#tree-shaking-глубже--что-можно-что-нельзя)
4. [Code splitting — стратегии и паттерны](#code-splitting--стратегии-и-паттерны)
5. [Оптимизация vendor chunks](#оптимизация-vendor-chunks)
6. [Борьба с дубликатами зависимостей](#борьба-с-дубликатами-зависимостей)
7. [Динамические импорты и lazy loading](#динамические-импорты-и-lazy-loading)
8. [Оптимизация изображений и ассетов](#оптимизация-изображений-и-ассетов)
9. [Compression — gzip, brotli, zstd](#compression--gzip-brotli-zstd)
10. [Бюджет бандла и автоматический контроль](#бюджет-бандла-и-автоматический-контроль)
11. [Реальные кейсы оптимизации](#реальные-кейсы-оптимизации)

---

## Почему размер бандла имеет значение

Размер бандла — это не просто «цифра в отчёте». Это прямое влияние на бизнес-метрики: конверсию, удержание пользователей, SEO.

### Влияние на пользовательский опыт

Когда пользователь открывает ваш сайт, браузер должен:
1. Загрузить HTML
2. Загрузить JavaScript-бандл
3. Распарсить JavaScript
4. Выполнить JavaScript
5. Отрендерить UI

Каждый килобайт JavaScript-бандла увеличивает время до интерактивности (Time to Interactive). Исследования Google показывают:
- Если время загрузки превышает 3 секунды, 53% пользователей уходят
- Каждые 100 КБ JavaScript уменьшают конверсию на 1-2%
- Мобильные пользователи особенно чувствительны к размеру (медленный интернет, слабые процессоры)

### Влияние на SEO

Google учитывает Core Web Vitals при ранжировании:
- **LCP (Largest Contentful Paint)** — время до отрисовки основного контента
- **FID (First Input Delay)** — задержка до первого взаимодействия
- **CLS (Cumulative Layout Shift)** — визуальная стабильность

Большой бандл увеличивает LCP и FID, что снижает позицию в поиске.

### Влияние на стоимость инфраструктуры

Каждый байт, загруженный пользователем, — это трафик, за который вы платите. Если у вас миллион пользователей в месяц, и каждый загружает 1 МБ вместо 500 КБ, это 500 ГБ дополнительного трафика.

### Реальные примеры

**Pinterest** уменьшил размер бандла на 50% и увидел:
- Увеличение конверсии на 44%
- Уменьшение времени до интерактивности на 40%

**BBC News** уменьшил бандл с 200 КБ до 100 КБ:
- Увеличение трафика на 15%
- Уменьшение bounce rate на 10%

**Walmart** уменьшил бандл на 35%:
- Увеличение конверсии на 2%
- Улучшение Core Web Vitals на 30%

Это не абстрактные цифры — это реальные деньги и пользователи.

---

## Анализ бандла — как найти проблемы

Прежде чем оптимизировать, нужно понять, **что** именно оптимизировать. Анализ бандла показывает, какие модули занимают больше всего места, где есть дубликаты, что можно удалить.

### webpack-bundle-analyzer

Самый популярный инструмент для анализа Webpack-бандлов. Создаёт интерактивную карту, где размер блока пропорционален его размеру в бандле.

```bash
# Установка
npm install -D webpack-bundle-analyzer

# Использование в webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static', // Генерирует HTML-файл
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
    }),
  ],
};

# Запуск сборки
npm run build

# Открыть bundle-report.html в браузере
```

Или через CLI:

```bash
# Анализ существующего бандла
npx webpack-bundle-analyzer stats.json
```

Что вы увидите:
- **Красные блоки** — большие модули (обычно библиотеки)
- **Жёлтые блоки** — ваш код
- **Синие блоки** — CSS
- **Зелёные блоки** — изображения и другие ассеты

Если вы видите огромный красный блок — это библиотека, которую можно оптимизировать. Если видите дубликаты одного модуля — это проблема, которую нужно решить.

### rollup-plugin-visualizer

Аналог для Vite и Rollup:

```bash
npm install -D rollup-plugin-visualizer
```

```js
// vite.config.js
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      filename: 'stats.html',
      gzipSize: true,
      brotliSize: true,
    }),
  ],
});
```

### Source map explorer

Анализирует source maps и показывает, какие модули попали в бандл:

```bash
npm install -D source-map-explorer

# Анализ
npx source-map-explorer 'dist/static/js/*.js'
```

### Что искать в анализе

**1. Большие библиотеки.** Moment.js (330 КБ), Lodash (530 КБ), RxJS (400 КБ). Часто можно заменить на более лёгкие альтернативы.

**2. Дубликаты.** Если вы видите два экземпляра одной библиотеки (например, два React), это значит, что разные пакеты используют разные версии. Нужно унифицировать версии.

**3. Мёртвый код.** Код, который импортируется, но не используется. Tree shaking должен его удалить, но если библиотека не поддерживает ESM, мёртвый код останется.

**4. Неоптимальные импорты.** Если вы импортируете `import _ from 'lodash'` вместо `import { get } from 'lodash'`, в бандл попадёт весь Lodash.

---

## Tree shaking глубже — что можно, что нельзя

Tree shaking — это процесс удаления неиспользуемого кода из бандла. Но он работает не всегда и не для всего.

### Как работает tree shaking

Tree shaking основан на **статическом анализе ES-модулей**. Бандлер анализирует импорты и экспорты, определяет, что используется, и удаляет остальное.

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

Бандлер видит, что импортируется только `add`, и удаляет `subtract` и `multiply` из бандла.

### Когда tree shaking не работает

**1. CommonJS модули.** CommonJS использует динамические импорты (`require()`), которые нельзя проанализировать статически:

```js
// CommonJS — tree shaking не работает
const utils = require('./utils');
utils.add(2, 3);
// В бандл попадёт весь utils.js, даже если используется только add
```

**2. Side effects.** Если модуль имеет побочные эффекты (изменяет глобальные объекты, добавляет polyfill'ы), бандлер не может его удалить, даже если он не используется явно:

```js
// polyfill.js
Array.prototype.customMethod = function() { /* ... */ };
// Этот файл имеет side effects — его нельзя удалить через tree shaking
```

Чтобы указать бандлеру, какие модули имеют side effects, используется поле `sideEffects` в `package.json`:

```json
{
  "name": "my-library",
  "sideEffects": [
    "./src/polyfill.js"
  ]
}
```

Если `sideEffects: false`, бандлер агрессивно удаляет неиспользуемый код.

**3. Реэкспорты.** Если модуль реэкспортирует всё из другого модуля, tree shaking может не сработать:

```js
// index.js
export * from './utils';
export * from './helpers';

// app.js
import { add } from './index';
// В бандл может попасть всё из utils и helpers
```

Решение: использовать именованные реэкспорты:

```js
// index.js
export { add, subtract } from './utils';
export { format } from './helpers';
```

### Практические советы

**1. Используйте именованные импорты:**

```js
// ❌ Плохо — импортирует весь Lodash
import _ from 'lodash';
_.get(obj, 'path');

// ✅ Хорошо — импортирует только get
import { get } from 'lodash';
get(obj, 'path');

// ✅ Ещё лучше — использует lodash-es (оптимизирован для tree shaking)
import { get } from 'lodash-es';
```

**2. Проверяйте, поддерживает ли библиотека tree shaking:**

Большинство современных библиотек поддерживают tree shaking, но не все. Проверьте `package.json`:

```json
{
  "name": "some-library",
  "module": "dist/index.esm.js", // ESM-версия — tree shaking работает
  "main": "dist/index.cjs.js"    // CommonJS — tree shaking не работает
}
```

Если библиотека имеет только `main` (CommonJS), tree shaking не будет работать. Ищите альтернативу с ESM-версией.

**3. Используйте `sideEffects: false` в своих библиотеках:**

Если вы публикуете библиотеку, добавьте `sideEffects: false` в `package.json`, чтобы бандлеры могли агрессивно удалять неиспользуемый код.

---

## Code splitting — стратегии и паттерны

Code splitting — это разделение бандла на меньшие части (чанки), которые загружаются по требованию. Это одна из самых эффективных оптимизаций.

### Стратегии code splitting

**1. Splitting по маршрутам.** Каждая страница — отдельный чанк. Пользователь загружает только ту страницу, которую смотрит.

```jsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

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

**2. Splitting по компонентам.** Тяжёлые компоненты (графики, редакторы, модалки) загружаются только когда нужны.

```jsx
import { lazy, Suspense, useState } from 'react';

const Chart = lazy(() => import('./components/Chart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      
      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <Chart />
        </Suspense>
      )}
    </div>
  );
}
```

**3. Splitting по условиям.** Код загружается только при определённых условиях (например, только для админов, только на мобильных устройствах).

```jsx
import { lazy, Suspense } from 'react';
import { useAuth } from './hooks/useAuth';

const AdminPanel = lazy(() => import('./components/AdminPanel'));

function App() {
  const { user } = useAuth();

  return (
    <div>
      {/* Основной контент для всех */}
      <MainContent />
      
      {/* Админ-панель загружается только для админов */}
      {user?.role === 'admin' && (
        <Suspense fallback={<div>Loading admin panel...</div>}>
          <AdminPanel />
        </Suspense>
      )}
    </div>
  );
}
```

### Предзагрузка (prefetching)

Если вы знаете, что пользователь скоро перейдёт на другую страницу, вы можете предзагрузить её чанк:

```jsx
import { useCallback } from 'react';

// Предзагрузка при наведении на ссылку
function NavLink({ to, children }) {
  const prefetch = useCallback(() => {
    if (to === '/dashboard') {
      import('./pages/Dashboard');
    }
  }, [to]);

  return (
    <a href={to} onMouseEnter={prefetch}>
      {children}
    </a>
  );
}
```

Или используйте `<link rel="prefetch">` в HTML:

```html
<link rel="prefetch" href="/static/js/dashboard.chunk.js" />
```

Браузер загрузит чанк в фоне, когда у него будет свободное время. Когда пользователь перейдёт на страницу, чанк уже будет в кэше.

### Предзагрузка в Next.js

Next.js автоматически предзагружает страницы, которые видны в viewport:

```jsx
import Link from 'next/link';

// При наведении на ссылку Next.js предзагружает страницу
<Link href="/dashboard">Dashboard</Link>
```

Это работает «из коробки» и не требует дополнительной настройки.

---

## Оптимизация vendor chunks

Vendor chunks — это чанки, которые содержат код из `node_modules` (библиотеки). Оптимизация vendor chunks может значительно уменьшить размер бандла.

### Разделение vendor chunks

По умолчанию Webpack создаёт один vendor chunk для всех библиотек. Но если вы измените одну библиотеку, весь vendor chunk станет недействительным, и пользователю придётся загружать его заново.

Лучше разделить vendor chunks по категориям:

```js
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // React и связанные библиотеки
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router)[\\/]/,
          name: 'react-vendor',
          chunks: 'all',
          priority: 20,
        },
        // UI-библиотеки
        ui: {
          test: /[\\/]node_modules[\\/](antd|@ant-design|@mui)[\\/]/,
          name: 'ui-vendor',
          chunks: 'all',
          priority: 15,
        },
        // Утилиты
        utils: {
          test: /[\\/]node_modules[\\/](lodash|date-fns|axios)[\\/]/,
          name: 'utils-vendor',
          chunks: 'all',
          priority: 10,
        },
        // Остальные библиотеки
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          chunks: 'all',
          priority: 0,
        },
      },
    },
  },
};
```

Теперь у вас четыре vendor chunks:
- `react-vendor` — React и роутинг (меняется редко)
- `ui-vendor` — UI-библиотеки (меняется иногда)
- `utils-vendor` — утилиты (меняется иногда)
- `vendor` — остальные библиотеки

Если вы обновите `lodash`, только `utils-vendor` станет недействительным. Остальные chunks останутся в кэше браузера.

### Долгосрочное кэширование

Используйте хэши в именах файлов для долгосрочного кэширования:

```js
// webpack.config.js
module.exports = {
  output: {
    filename: '[name].[contenthash:8].js',
    chunkFilename: '[name].[contenthash:8].chunk.js',
  },
};
```

Теперь файлы будут называться `react-vendor.a1b2c3d4.js`. Если содержимое не изменится, хэш останется тем же, и браузер будет использовать кэшированную версию.

Настройте сервер (nginx, CDN) для долгосрочного кэширования:

```nginx
location /static/ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}
```

Директива `immutable` говорит браузеру, что файл никогда не изменится (потому что хэш меняется при изменении содержимого), и не нужно проверять его актуальность.

---

## Борьба с дубликатами зависимостей

Дубликаты зависимостей — это когда одна и та же библиотека присутствует в бандле несколько раз. Это может значительно увеличить размер бандла.

### Как появляются дубликаты

**1. Разные версии.** Пакет A использует `lodash@4.17.21`, пакет B использует `lodash@4.17.20`. NPM установит обе версии.

**2. Разные форматы.** Некоторые библиотеки имеют разные версии для CommonJS и ESM. Если один пакет импортирует CommonJS-версию, а другой — ESM, в бандл попадут обе.

**3. Nested dependencies.** Пакет A зависит от пакета C, пакет B тоже зависит от пакета C, но с другой версией. NPM установит C в `node_modules` A и в `node_modules` B.

### Как обнаружить дубликаты

**webpack-bundle-analyzer** показывает дубликаты визуально — вы увидите два блока с одинаковым названием.

**dupfinder** — инструмент для поиска дубликатов:

```bash
npx dupfinder dist/
```

**npm ls** — показывает дерево зависимостей:

```bash
npm ls lodash
```

Если вы увидите несколько версий lodash, это дубликаты.

### Как устранить дубликаты

**1. Унифицируйте версии.** Обновите все пакеты до одной версии зависимости:

```bash
npm dedupe
```

Эта команда пытается уменьшить дубликаты, устанавливая одну версию для всех пакетов.

**2. Используйте overrides (npm) или resolutions (Yarn).** Принудительно укажите версию зависимости:

```json
// package.json (npm)
{
  "overrides": {
    "lodash": "4.17.21"
  }
}
```

```json
// package.json (Yarn)
{
  "resolutions": {
    "lodash": "4.17.21"
  }
}
```

**3. Используйте pnpm.** pnpm использует строгую структуру `node_modules`, которая предотвращает многие дубликаты.

**4. Замените библиотеки.** Если библиотека имеет много дубликатов, возможно, стоит заменить её на более лёгкую альтернативу.

---

## Динамические импорты и lazy loading

Динамические импорты (`import()`) позволяют загружать модули по требованию. Это основа code splitting.

### Синтаксис динамического импорта

```js
// Статический импорт — загружается сразу
import { add } from './utils';

// Динамический импорт — загружается по требованию
const loadUtils = () => import('./utils');

loadUtils().then((utils) => {
  console.log(utils.add(2, 3));
});
```

### Lazy loading в React

React предоставляет `React.lazy` для ленивой загрузки компонентов:

```jsx
import { lazy, Suspense } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

### Lazy loading в Next.js

Next.js поддерживает `next/dynamic`:

```jsx
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <div>Loading...</div>,
  ssr: false, // Отключаем SSR для этого компонента
});

function Page() {
  return <HeavyComponent />;
}
```

### Когда использовать lazy loading

**Используйте, если:**
- Компонент большой (> 50 КБ)
- Компонент используется редко (модалка, график, редактор)
- Компонент не нужен для первоначального рендера

**Не используйте, если:**
- Компонент маленький (< 10 КБ)
- Компонент нужен сразу (header, footer)
- Компонент используется на каждой странице

---

## Оптимизация изображений и ассетов

Изображения часто занимают больше места, чем JavaScript. Оптимизация изображений может уменьшить размер страницы в разы.

### Форматы изображений

**WebP** — современный формат с лучшим сжатием, чем JPEG и PNG. Поддерживается всеми современными браузерами.

**AVIF** — ещё более эффективный формат, но поддержка в браузерах ограниченна.

**SVG** — для иконок и логотипов. Масштабируется без потери качества.

### Оптимизация в Next.js

Next.js предоставляет компонент `Image` для автоматической оптимизации:

```jsx
import Image from 'next/image';

function Avatar() {
  return (
    <Image
      src="/avatar.jpg"
      alt="User avatar"
      width={200}
      height={200}
      quality={80} // Качество JPEG (по умолчанию 75)
      format="webp" // Автоматическое преобразование в WebP
    />
  );
}
```

Next.js автоматически:
- Конвертирует в WebP/AVIF
- Создаёт несколько размеров для responsive images
- Lazy loads изображения ниже the fold
- Оптимизирует при сборке

### Оптимизация в Vite

Vite автоматически инлайнит небольшие изображения (< 4 КБ) как base64:

```js
// vite.config.js
export default defineConfig({
  build: {
    assetsInlineLimit: 4096, // 4 КБ
  },
});
```

Для больших изображений используйте плагин `vite-plugin-image-optimizer`:

```bash
npm install -D vite-plugin-image-optimizer
```

```js
import { defineConfig } from 'vite';
import { imageOptimizer } from 'vite-plugin-image-optimizer';

export default defineConfig({
  plugins: [imageOptimizer()],
});
```

### SVG-оптимизация

Используйте SVGO для оптимизации SVG:

```bash
npm install -D svgo
```

```bash
npx svgo input.svg -o output.svg
```

Или используйте плагин для Vite/Webpack, который оптимизирует SVG при сборке.

---

## Compression — gzip, brotli, zstd

Сжатие — это последний этап оптимизации. После tree shaking, code splitting и минификации, сжатие уменьшает размер бандла ещё на 60-80%.

### Gzip

Gzip — самый распространённый алгоритм сжатия. Поддерживается всеми браузерами и серверами.

```nginx
# nginx.conf
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_min_length 1000; // Сжимать только файлы > 1 КБ
gzip_comp_level 6; // Уровень сжатия (1-9)
```

### Brotli

Brotli — более современный алгоритм от Google. Сжимает лучше, чем gzip (на 15-20%), но требует больше ресурсов для сжатия.

```nginx
# nginx.conf
brotli on;
brotli_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
brotli_comp_level 6;
```

Браузеры поддерживают Brotli через заголовок `Accept-Encoding`:

```
Accept-Encoding: gzip, deflate, br
```

### Предварительное сжатие

Вместо сжатия на лету (что нагружает сервер), можно предварительно сжать файлы при сборке:

```bash
# Gzip
gzip -k dist/static/js/*.js

# Brotli (нужен brotli CLI)
brotli -k dist/static/js/*.js
```

Теперь у вас есть файлы `app.js`, `app.js.gz`, `app.js.br`. Настройте сервер для отдачи предварительно сжатых файлов:

```nginx
location /static/ {
  gzip_static on;
  brotli_static on;
  
  # Если клиент поддерживает brotli — отдавать .br
  # Если поддерживает gzip — отдавать .gz
  # Иначе — отдавать оригинал
}
```

### Zstd

Zstd (Zstandard) — ещё более эффективный алгоритм от Facebook. Сжимает лучше brotli, но поддержка в браузерах ограничена.

Пока не рекомендуется для веба, но стоит следить за развитием.

---

## Бюджет бандла и автоматический контроль

Бюджет бандла (bundle budget) — это ограничение на размер бандла. Если бандл превышает бюджет, CI падает, и PR не может быть смержен.

### Настройка бюджета в Webpack

```js
// webpack.config.js
module.exports = {
  performance: {
    maxAssetSize: 250000, // 250 КБ на файл
    maxEntrypointSize: 400000, // 400 КБ на entry point
    hints: 'error', // Ошибка, если превышен бюджет
  },
};
```

### bundlesize

Инструмент для проверки размера бандла в CI:

```bash
npm install -D bundlesize
```

```json
// package.json
{
  "bundlesize": [
    {
      "path": "./dist/static/js/*.js",
      "maxSize": "150 kB"
    },
    {
      "path": "./dist/static/css/*.css",
      "maxSize": "20 kB"
    }
  ]
}
```

```bash
npx bundlesize
```

Если размер превышает `maxSize`, команда завершится с ошибкой.

### size-limit

Более продвинутый инструмент, который учитывает время загрузки:

```bash
npm install -D size-limit
```

```json
// .size-limit.json
[
  {
    "path": "dist/index.js",
    "limit": "10 KB",
    "gzip": true
  }
]
```

```bash
npx size-limit
```

Size-limit показывает не только размер, но и время загрузки на разных скоростях интернета.

### Интеграция в CI

```yaml
# .github/workflows/bundle-size.yml
name: Bundle Size

on: [pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run build
      - run: npx bundlesize
        env:
          BUNDLESIZE_GITHUB_TOKEN: ${{ secrets.BUNDLESIZE_GITHUB_TOKEN }}
```

`bundlesize` может автоматически добавлять комментарий в PR с сравнением размеров:

```
Bundle size: 145 KB (was 150 KB, -5 KB)
```

---

## Реальные кейсы оптимизации

### Кейс 1: Уменьшение бандла с 2 МБ до 500 КБ

**Проблема:** Приложение имело бандл 2 МБ. Время загрузки — 8 секунд на 3G.

**Анализ:**
- Moment.js — 330 КБ
- Lodash (полный) — 530 КБ
- Два экземпляра React — 200 КБ
- Много дубликатов

**Решение:**
1. Заменили Moment.js на date-fns (75 КБ → 20 КБ)
2. Заменили `import _ from 'lodash'` на `import { get } from 'lodash-es'` (530 КБ → 5 КБ)
3. Унифицировали версии React через `overrides` (убрали 200 КБ дубликатов)
4. Добавили code splitting для маршрутов
5. Настроили tree shaking

**Результат:** Бандл уменьшился с 2 МБ до 450 КБ. Время загрузки сократилось с 8 до 2 секунд.

### Кейс 2: Оптимизация vendor chunks

**Проблема:** Vendor chunk 800 КБ. При любом изменении кода пользователь загружал весь vendor chunk заново.

**Решение:**
1. Разделили vendor chunks по категориям (React, UI, utils)
2. Настроили долгосрочное кэширование с хэшами
3. Добавили prefetching для критичных chunks

**Результат:** Repeat visits загружают только изменённые chunks. Время повторной загрузки сократилось с 3 до 0.5 секунд.

### Кейс 3: Оптимизация изображений

**Проблема:** Страница загружала 5 МБ изображений.

**Решение:**
1. Конвертировали JPEG в WebP (уменьшение на 30%)
2. Использовали responsive images (разные размеры для разных устройств)
3. Добавили lazy loading для изображений ниже the fold
4. Оптимизировали SVG через SVGO

**Результат:** Размер изображений сократился с 5 МБ до 1.5 МБ. LCP улучшился на 40%.

---

## Заключение

Оптимизация бандла — это не разовая задача, а непрерывный процесс. Каждый новый код, каждая новая библиотека увеличивает размер бандла. Без контроля бандл разрастается до неприемлемых размеров.

**Ключевые выводы:**

1. **Анализируйте бандл.** Используйте webpack-bundle-analyzer или rollup-plugin-visualizer для понимания, что попадает в бандл.

2. **Tree shaking работает не всегда.** CommonJS, side effects, реэкспорты — всё это может предотвратить tree shaking.

3. **Code splitting — самая эффективная оптимизация.** Разделяйте бандл по маршрутам, компонентам, условиям.

4. **Оптимизируйте vendor chunks.** Разделяйте библиотеки по категориям для лучшего кэширования.

5. **Боритесь с дубликатами.** Используйте `npm dedupe`, `overrides`, pnpm.

6. **Сжатие обязательно.** Gzip или Brotli должны быть включены на сервере.

7. **Настройте бюджет бандла.** Используйте bundlesize или size-limit для автоматического контроля в CI.

Оптимизация бандла — это инвестиция в пользовательский опыт и бизнес-метрики. Каждые 100 КБ — это конверсия, удержание, SEO.
