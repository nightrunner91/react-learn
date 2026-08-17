# Тестирование производительности

## Содержание

1. [Что такое тестирование производительности](#что-такое-тестирование-производительности)
2. [Lighthouse CI](#lighthouse-ci)
3. [Web Vitals в тестах](#web-vitals-в-тестах)
4. [Тестирование bundle size](#тестирование-bundle-size)
5. [Нагрузочное тестирование API](#нагрузочное-тестирование-api)
6. [Performance budgets в CI](#performance-budgets-в-ci)
7. [Профилирование React-компонентов](#профилирование-react-компонентов)
8. [Тестирование времени выполнения](#тестирование-времени-выполнения)
9. [Мониторинг производительности в production](#мониторинг-производительности-в-production)
10. [Лучшие практики](#лучшие-практики)
11. [Антипаттерны](#антипаттерны)

---

## Что такое тестирование производительности

Тестирование производительности проверяет, что приложение работает быстро и эффективно. Оно включает:

| Вид | Что проверяет | Инструменты |
|---|---|---|
| **Lighthouse** | Общая оценка (Performance, Accessibility, SEO) | Lighthouse CI |
| **Web Vitals** | Core Web Vitals (LCP, CLS, INP) | web-vitals, Playwright |
| **Bundle size** | Размер JavaScript/CSS бандла | size-limit, bundlesize |
| **Нагрузочное** | API под нагрузкой | k6, Artillery |
| **Профилирование** | Рендеринг компонентов | React DevTools Profiler |

### Когда использовать

**Использовать:**
- Критичные страницы (landing, checkout)
- PWA и мобильные приложения
- Высоконагруженные API
- Дизайн-системы

**Не использовать:**
- Внутренние админки (если не критично)
- Прототипы
- Одноразовые скрипты

---

## Lighthouse CI

### Что такое Lighthouse CI

Lighthouse CI автоматизирует запуск Lighthouse в CI/CD. Проверяет Performance, Accessibility, Best Practices, SEO.

### Установка

Lighthouse CI запускается как CLI-инструмент. Он поднимает сервер, открывает страницы через headless Chrome и собирает метрики. Установка проста — один пакет без дополнительных зависимостей:

```bash
npm install -D @lhci/cli
```

### Конфигурация

Конфигурация определяет три вещи: какие URL проверять (`collect`), куда сохранять результаты (`upload`) и какие пороги считать провалом (`assert`). Без `assert` Lighthouse CI просто собирает данные без блокировки CI — это удобно для первого запуска, чтобы посмотреть текущие метрики.

Начальная конфигурация для типичного SPA:

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ["http://localhost:3000/", "http://localhost:3000/about"],
      startServerCommand: "npm run build && npm run start",
    },
    upload: {
      target: "temporary-public-storage",
    },
    assert: {
      assertions: {
        "categories:performance": ["error", { minScore: 0.9 }],
        "categories:accessibility": ["error", { minScore: 0.9 }],
        "categories:best-practices": ["error", { minScore: 0.9 }],
        "categories:seo": ["error", { minScore: 0.9 }],
      },
    },
  },
};
```

### Запуск

После конфигурации запуск — одна команда. `autorun` автоматически находит `lighthouserc.js`, запускает сервер, проверяет URL и сравнивает с порогами:

```bash
npx lhci autorun
```

### GitHub Actions

Для интеграции с GitHub нужен токен `LHCI_GITHUB_APP_TOKEN` — он позволяет Lighthouse CI оставлять комментарии в PR с результатами. Без токена тесты всё равно работают, но результаты видны только в логах CI. Токен получается через [GitHub App Lighthouse CI](https://github.com/apps/lighthouse-ci):

```yaml
name: Lighthouse CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - run: npm ci
      
      - name: Run Lighthouse CI
        run: npx lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

### Assertions

Assertions позволяют задавать пороги как для категорий (Performance, Accessibility), так и для отдельных метрик. Разница между `error` и `warn`: `error` ломает CI, `warn` только предупреждает. Начинайте с `warn`, чтобы не блокировать разработку, и переводите в `error` после стабилизации метрик.

Пример комбинирования категорий и конкретных метрик:

```javascript
// lighthouserc.js
assert: {
  assertions: {
    // Категории
    "categories:performance": ["error", { minScore: 0.9 }],
    "categories:accessibility": ["warn", { minScore: 0.8 }],
    
    // Метрики
    "first-contentful-paint": ["error", { maxNumericValue: 2000 }],
    "largest-contentful-paint": ["error", { maxNumericValue: 2500 }],
    "interactive": ["error", { maxNumericValue: 3800 }],
    "cumulative-layout-shift": ["error", { maxNumericValue: 0.1 }],
    
    // Аудиты
    "unused-javascript": ["warn", { maxLength: 100000 }],
    "uses-responsive-images": ["warn", {}],
  },
}
```

---

## Web Vitals в тестах

### Что такое Web Vitals

Web Vitals — метрики, измеряющие пользовательский опыт:

| Метрика | Что измеряет | Целевое значение |
|---|---|---|
| **LCP** (Largest Contentful Paint) | Время загрузки основного контента | < 2.5с |
| **INP** (Interaction to Next Paint) | Отзывчивость интерфейса | < 200мс |
| **CLS** (Cumulative Layout Shift) | Визуальная стабильность | < 0.1 |
| **FCP** (First Contentful Paint) | Время до первого контента | < 1.8с |
| **TTFB** (Time to First Byte) | Время до первого байта | < 800мс |

### Измерение в тестах Playwright

Измерять Web Vitals в E2E-тестах имеет смысл только для критичных страниц — лендингов, checkout, dashboard. Для внутренних страниц это избыточно. Важно: метрики в headless-браузере всегда лучше, чем на реальном устройстве. Используйте результаты как baseline для отслеживания регрессий, а не как абсолютные значения.

Playwright предоставляет доступ к `PerformanceObserver` через `page.evaluate`. Код ниже собирает LCP и CLS вручную через браузерные API:

```ts
import { test, expect } from "@playwright/test";

test("measures Web Vitals", async ({ page }) => {
  await page.goto("/");
  
  // Ждём полную загрузку
  await page.waitForLoadState("networkidle");
  
  // Получаем метрики
  const metrics = await page.evaluate(() => {
    return new Promise((resolve) => {
      const metrics: Record<string, number> = {};
      
      // LCP
      new PerformanceObserver((entryList) => {
        const entries = entryList.getEntries();
        metrics.lcp = entries[entries.length - 1].startTime;
      }).observe({ type: "largest-contentful-paint", buffered: true });
      
      // CLS
      new PerformanceObserver((entryList) => {
        let cls = 0;
        for (const entry of entryList.getEntries()) {
          if (!(entry as any).hadRecentInput) {
            cls += (entry as any).value;
          }
        }
        metrics.cls = cls;
      }).observe({ type: "layout-shift", buffered: true });
      
      setTimeout(() => resolve(metrics), 1000);
    });
  });
  
  console.log("Web Vitals:", metrics);
  
  // Проверки
  expect(metrics.lcp).toBeLessThan(2500);
  expect(metrics.cls).toBeLessThan(0.1);
});
```

### Использование web-vitals библиотеки

Библиотека `web-vitals` от Google инкапсулирует работу с `PerformanceObserver` и даёт более точные результаты, чем ручная реализация. Она корректно обрабатывает edge cases (например, FID заменили на INP в 2024). Однако в headless-среде `web-vitals` требует таймаут для сбора данных, потому что некоторые метрики (LCP, CLS) продолжают обновляться после загрузки страницы.

Подключение библиотеки внутри `page.evaluate`:

```ts
import { onLCP, onINP, onCLS } from "web-vitals";

test("measures Core Web Vitals", async ({ page }) => {
  await page.goto("/");
  
  const metrics = await page.evaluate(() => {
    return new Promise((resolve) => {
      const metrics: Record<string, number> = {};
      
      onLCP((metric) => { metrics.lcp = metric.value; });
      onINP((metric) => { metrics.inp = metric.value; });
      onCLS((metric) => { metrics.cls = metric.value; });
      
      setTimeout(() => resolve(metrics), 2000);
    });
  });
  
  expect(metrics.lcp).toBeLessThan(2500);
  expect(metrics.inp).toBeLessThan(200);
  expect(metrics.cls).toBeLessThan(0.1);
});
```

---

## Тестирование bundle size

### Что такое size-limit

size-limit — инструмент для измерения размера JavaScript бандла и предотвращения регрессий.

### Установка

size-limit работает поверх bundler'а (webpack, esbuild, rollup) и считает реальный размер после tree-shaking и минификации. Это точнее, чем просто `du -sk dist/`, потому что учитывает gzip/brotli сжатие. Устанавливается как dev-зависимость:

```bash
npm install -D size-limit
```

### Конфигурация

Конфигурация задаёт лимиты для конкретных файлов. Важно проверять не только общий бандл, но и vendor-чанк (React, lodash и т.д.) и CSS отдельно. Лимиты задаются строкой с единицей измерения — `size-limit` парсит `"50 KB"`, `"1 MB"` и т.д. Если лимит превышен — `size-limit` завершается с ошибкой, что ломает CI.

Пример конфигурации в `package.json`:

```json
// package.json
{
  "size-limit": [
    {
      "path": "dist/index.js",
      "limit": "50 KB"
    },
    {
      "path": "dist/vendor.js",
      "limit": "100 KB"
    },
    {
      "path": "dist/**/*.css",
      "limit": "10 KB"
    }
  ]
}
```

### Запуск

Запуск показывает текущие размеры и сравнивает с лимитами. Если лимит не задан — `size-limit` просто покажет размеры без ошибки:

```bash
npx size-limit
```

### GitHub Actions

GitHub Action `andresz1/size-limit-action` автоматически сравнивает размеры бандла в PR с базовой веткой и оставляет комментарий с таблицей изменений. Параметр `skip_step: build` нужен, потому что предыдущий шаг уже выполнил сборку:

```yaml
name: Bundle Size
on:
  pull_request:
    branches: [main]

jobs:
  size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - run: npm ci
      
      - run: npm run build
      
      - uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          skip_step: build
```

### Альтернативы

Выбор инструмента зависит от задачи. `size-limit` — самый точный для библиотек и приложений с tree-shaking. `bundlesize` — проще, но считает только gzip без анализа зависимостей. `webpack-bundle-analyzer` и `source-map-explorer` — для визуального анализа, а не для CI-проверок:

| Инструмент | Особенности |
|---|---|
| **size-limit** | Точный, поддерживает tree-shaking |
| **bundlesize** | Простой, интеграция с CI |
| **webpack-bundle-analyzer** | Визуализация бандла |
| **source-map-explorer** | Анализ source maps |

---

## Нагрузочное тестирование API

### Что такое k6

k6 — инструмент нагрузочного тестирования от Grafana. Позволяет тестировать API под нагрузкой.

### Установка

k6 написан на Go, поэтому установка — через системный пакетный менеджер. Альтернатива — Docker-образ `grafana/k6`. На Windows k6 доступен через Chocolatey или как standalone binary с GitHub Releases:

```bash
# macOS
brew install k6

# Linux
sudo apt install k6

# Windows
choco install k6
```

### Базовый скрипт

Базовый скрипт k6 определяет два параметра: `vus` (виртуальные пользователи) и `duration`. Каждый VU — это отдельная goroutine, выполняющая `default function` параллельно. `check` — это assertion без остановки теста (в отличие от `threshold`). `sleep` имитирует реальное поведение пользователя — без пауз все запросы уйдут одновременно, что не отражает реальную нагрузку.

Простой сценарий для smoke-теста API:

```javascript
// load-test.js
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  vus: 100, // Количество виртуальных пользователей
  duration: "30s", // Длительность теста
};

export default function () {
  const res = http.get("http://localhost:3000/api/users");
  
  check(res, {
    "status is 200": (r) => r.status === 200,
    "response time < 500ms": (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

### Запуск

Запуск выводит таблицу с агрегированными результатами: min/avg/max/p90/p95 для времени ответа, количество итераций, ошибки. Если `threshold` не пройден — exit code будет ненулевым:

```bash
k6 run load-test.js
```

### Продвинутый сценарий

Продвинутый сценарий использует `stages` для постепенного увеличения нагрузки (разминка → пик → остывание) и `thresholds` для автоматического провала теста. `Rate` — кастомная метрика для отслеживания процента ошибок конкретного чека. Такой подход ближе к реальности: сервер проходит разминку, работает под пиковой нагрузкой и корректно завершает соединения.

Сценарий с stages и thresholds для production-like нагрузки:

```javascript
// load-test.js
import http from "k6/http";
import { check, sleep } from "k6";
import { Rate } from "k6/metrics";

const errorRate = new Rate("errors");

export const options = {
  stages: [
    { duration: "1m", target: 50 }, // Разминка
    { duration: "3m", target: 100 }, // Нагрузка
    { duration: "1m", target: 0 },  // Остывание
  ],
  thresholds: {
    http_req_duration: ["p(95)<500"], // 95% запросов < 500мс
    errors: ["rate<0.01"], // Менее 1% ошибок
  },
};

export default function () {
  const res = http.post("http://localhost:3000/api/login", {
    email: "user@example.com",
    password: "password123",
  });
  
  const success = check(res, {
    "logged in successfully": (r) => r.json().token !== undefined,
  });
  
  errorRate.add(!success);
  
  sleep(1);
}
```

### Artillery — альтернатива

Artillery — YAML-конфигурируемый инструмент, проще k6 для базовых сценариев. Подходит, если команда не хочет писать JavaScript для тестов. `think` — пауза между шагами сценария, аналог `sleep` в k6. Artillery лучше подходит для quick smoke tests, k6 — для серьёзных нагрузочных тестов с кастомными метриками.

Конфигурация Artillery в YAML:

```yaml
# load-test.yml
config:
  target: "http://localhost:3000"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Load test"

scenarios:
  - name: "User login"
    flow:
      - post:
          url: "/api/login"
          json:
            email: "user@example.com"
            password: "password123"
      - think: 1
```

Запуск Artillery через npx не требует глобальной установки:

```bash
npx artillery run load-test.yml
```

---

## Performance budgets в CI

### Что такое performance budgets

Performance budgets — лимиты на метрики производительности. Если лимит превышен — CI падает.

### Vite — настройка лимитов

`chunkSizeWarningLimit` заставляет Vite показывать предупреждение при сборке, если чанк превышает лимит. Это первая линия обороны — разработчик видит проблему сразу после `npm run build`. `manualChunks` позволяет явно выделить vendor-зависимости в отдельный чанк, чтобы они кешировались браузером отдельно от бизнес-кода. При изменении только бизнес-кода vendor-чанк остаётся в кеше.

Конфигурация Vite для контроля размера бандла:

```ts
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  build: {
    chunkSizeWarningLimit: 500, // Предупреждение для чанков > 500KB
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
        },
      },
    },
  },
});
```

### Next.js — настройка лимитов

`optimizePackageImports` указывает Next.js загружать только используемые функции из тяжёлых пакетов (tree-shaking для runtime). Без этого `import { debounce } from "lodash"` тянет всю библиотеку. `removeConsole` удаляет `console.log` из production-бандла — это и уменьшает размер, и убирает отладочный вывод. Компромисс: `console.error` и `console.warn` тоже удаляются, что может затруднить диагностику.

Конфигурация Next.js для оптимизации бандла:

```javascript
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ["@heroicons/react", "lodash"],
  },
  compiler: {
    removeConsole: process.env.NODE_ENV === "production",
  },
};
```

### CI проверка

Скрипт ниже — минимальная проверка размера бандла в CI. Для серьёзных проектов лучше использовать `size-limit` (см. выше), потому что `du` считает размер на диске, а не размер после gzip. Однако для быстрого прототипа или monorepo с нестандартной структурой shell-скрипт может быть проще.

Проверка размера бандла в GitHub Actions:

```yaml
name: Performance Budget
on:
  pull_request:
    branches: [main]

jobs:
  performance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - run: npm ci
      
      - run: npm run build
      
      - name: Check bundle size
        run: |
          # Проверка размера бандла
          BUNDLE_SIZE=$(du -sk .next/static/chunks | cut -f1)
          MAX_SIZE=1024 # 1MB в KB
          
          if [ $BUNDLE_SIZE -gt $MAX_SIZE ]; then
            echo "Bundle size exceeds limit: ${BUNDLE_SIZE}KB > ${MAX_SIZE}KB"
            exit 1
          fi
```

---

## Профилирование React-компонентов

### React DevTools Profiler

React DevTools Profiler — основной инструмент для поиска медленных компонентов. Он показывает `actualDuration` (время рендера компонента) и `baseDuration` (время без мемоизации). В тестах `<Profiler>` даёт те же данные программно, что позволяет автоматизировать проверки.

Компромисс профилирования в тестах: `Profiler` добавляет overhead (~5-10% к времени рендера), поэтому абсолютные значения будут отличаться от production. Используйте тесты для обнаружения грубых регрессий (рендер стал в 2 раза медленнее), а не для точных бенчмарков.

### Использование в тестах

`<Profiler>` оборачивает компонент и вызывает `onRender` после каждого рендера. Аргументы `onRender`: `id`, `phase` ("mount" или "update"), `actualDuration`, `baseDuration`, `startTime`, `commitTime`. Для проверки производительности достаточно `actualDuration` — реального времени рендера.

Тест, проверяющий что рендер списка из 1000 элементов укладывается в 100мс:

```tsx
import { render, screen } from "@testing-library/react";
import { Profiler } from "react";
import { ExpensiveList } from "./ExpensiveList";

it("renders list efficiently", () => {
  const onRender = vi.fn();
  
  render(
    <Profiler id="ExpensiveList" onRender={onRender}>
      <ExpensiveList items={Array.from({ length: 1000 })} />
    </Profiler>
  );
  
  // Проверяем время рендеринга
  const renderCall = onRender.mock.calls[0];
  const actualDuration = renderCall[2]; // actualDuration в мс
  
  expect(actualDuration).toBeLessThan(100); // Менее 100мс
});
```

### Тестирование мемоизации

Мемоизация (`React.memo`, `useMemo`, `useCallback`) предотвращает ререндер при неизменных пропсах. Тест ниже проверяет, что `onRender` вызывается только один раз — значит, компонент не ререндерится при тех же пропсах. Если `onRender` вызывается дважды — мемоизация не работает (пропсы создаются заново, или компонент не обёрнут в `memo`).

Важно: `Profiler` оборачивается вокруг компонента при каждом ререндере, чтобы `onRender` фиксировал каждый рендер. Без повторного оборачивания `Profiler` не увидит ререндер.

Тест корректности мемоизации:

```tsx
import { render } from "@testing-library/react";
import { Profiler, memo } from "react";
import { ExpensiveComponent } from "./ExpensiveComponent";

it("memoizes component correctly", () => {
  const onRender = vi.fn();
  
  const { rerender } = render(
    <Profiler id="Expensive" onRender={onRender}>
      <ExpensiveComponent value="test" />
    </Profiler>
  );
  
  // Первый рендер
  expect(onRender).toHaveBeenCalledTimes(1);
  
  // Ререндер с теми же пропсами
  rerender(
    <Profiler id="Expensive" onRender={onRender}>
      <ExpensiveComponent value="test" />
    </Profiler>
  );
  
  // Не должно быть нового рендера (если компонент мемоизирован)
  expect(onRender).toHaveBeenCalledTimes(1);
});
```

---

## Профилирование Vue-компонентов

### Vue DevTools Profiler

Vue DevTools — основной инструмент для анализа производительности Vue-приложений. Он показывает время рендера компонентов, количество ререндеров и зависимости реактивных данных. В тестах DevTools недоступен, но можно измерить время рендера программно через `performance.now` или через хук `onRenderTracked`.

### Измерение времени рендера

Для автоматизации проверок можно обернуть рендер компонента в `performance.now()`. Это даёт грубую оценку времени рендера — достаточную для обнаружения регрессий.

```ts
import { mount } from "@vue/test-utils";
import ExpensiveList from "./ExpensiveList.vue";

it("renders list efficiently", () => {
  const items = Array.from({ length: 1000 }, (_, i) => ({ id: i, name: `Item ${i}` }));

  const start = performance.now();
  const wrapper = mount(ExpensiveList, {
    props: { items },
  });
  const duration = performance.now() - start;

  expect(wrapper.findAll("li")).toHaveLength(1000);
  expect(duration).toBeLessThan(100); // Менее 100мс
});
```

### Тестирование мемоизации

Vue предоставляет `computed`, `watch` и `shallowRef` для оптимизации реактивности. Для проверки того, что компонент не ререндерится лишний раз, можно отслеживать количество вызовов `onUpdated`.

```vue
<!-- MemoizedCounter.vue -->
<template>
  <div>{{ count }}</div>
</template>

<script setup>
import { ref, onUpdated } from "vue";

const props = defineProps({ value: Number });
const count = ref(0);

onUpdated(() => {
  count.value++;
});
</script>
```

```ts
it("does not rerender when props do not change", async () => {
  const wrapper = mount(MemoizedCounter, {
    props: { value: 1 },
  });

  // Первый рендер
  expect(wrapper.vm.count).toBe(0);

  // Ререндер с теми же пропсами
  await wrapper.setProps({ value: 1 });

  expect(wrapper.vm.count).toBe(0);
});
```

> ⚠️ Тестирование через `wrapper.vm` допустимо при проверке производительности, но избегайте завязывать тесты на внутреннее состояние в обычных сценариях.

---

## Тестирование времени выполнения

Тестирование времени выполнения (timing tests) проверяет, что функции укладываются в разумные сроки. Это не замена профилированию — timing тесты ловят грубые регрессии (O(n^2) вместо O(n)), а не микрооптимизации. Порог должен быть щедрым (100мс вместо реальных 5мс), иначе тесты будут flaky на медленных CI-машинах.

### Тестирование утилит

`performance.now()` точнее `Date.now()` — работает с субмиллисекундной точностью и не зависит от системных часов. Для синхронных функций оборачиваем вызов в `performance.now()` до и после. Для асинхронных — см. следующий пример.

Тест синхронной утилиты с проверкой времени:

```ts
import { heavyComputation } from "./utils";

it("computes result within time limit", () => {
  const start = performance.now();
  
  const result = heavyComputation(1000000);
  
  const duration = performance.now() - start;
  
  expect(result).toBeDefined();
  expect(duration).toBeLessThan(100); // Менее 100мс
});
```

### Тестирование асинхронных операций

Для асинхронных операций `performance.now()` оборачивает `await`. Важно: время включает не только вычисления, но и сетевую задержку. Если тестируете с реальной API — используйте моки, иначе тест будет flaky. Порог для асинхронных операций обычно выше (1-3 секунды), потому что сеть непредсказуема.

Тест асинхронной операции:

```ts
import { fetchAndProcessData } from "./api";

it("fetches and processes data within time limit", async () => {
  const start = performance.now();
  
  const data = await fetchAndProcessData();
  
  const duration = performance.now() - start;
  
  expect(data).toBeDefined();
  expect(duration).toBeLessThan(1000); // Менее 1 секунды
});
```

---

## Мониторинг производительности в production

Lighthouse и E2E-тесты измеряют производительность в контролируемой среде. Реальные пользователи работают на разных устройствах, в разных сетях, с разными расширениями браузера. Production-мониторинг собирает метрики от настоящих пользователей (Real User Monitoring, RUM) и показывает, что происходит на самом деле. Без RUM вы видите только синтетические данные, которые могут сильно отличаться от реальности.

### web-vitals в production

Код ниже отправляет метрики на сервер аналитики. `navigator.sendBeacon` — специальный API для отправки данных при выгрузке страницы (когда обычный `fetch` может не успеть). `keepalive: true` в `fetch` — fallback для браузеров без `sendBeacon`.

Важно: не отправляйте метрики на каждый визит — это создаёт нагрузку на сервер. Используйте сэмплирование (например, отправляйте только 10% визитов) или батчинг (накапливайте метрики и отправляйте пачкой).

Модуль отправки Web Vitals в аналитику:

```tsx
// src/analytics/web-vitals.ts
import { onLCP, onFID, onCLS, onFCP, onTTFB } from "web-vitals";

function sendToAnalytics(metric: any) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
    navigationType: metric.navigationType,
  });
  
  // Отправка в аналитику
  if (navigator.sendBeacon) {
    navigator.sendBeacon("/analytics", body);
  } else {
    fetch("/analytics", {
      body,
      method: "POST",
      keepalive: true,
    });
  }
}

export function initWebVitals() {
  onLCP(sendToAnalytics);
  onFID(sendToAnalytics);
  onCLS(sendToAnalytics);
  onFCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
}
```

### Использование в приложении

Инициализация должна происходить как можно раньше — до рендера приложения, чтобы не пропустить метрики загрузки. Проверка `typeof window !== "undefined"` нужна для SSR (Next.js, Remix), где первый рендер происходит на сервере без `window`.

Подключение мониторинга в корневом компоненте:

```tsx
// src/app/layout.tsx
import { initWebVitals } from "@/analytics/web-vitals";

if (typeof window !== "undefined") {
  initWebVitals();
}
```

---

## Лучшие практики

### 1. Устанавливайте реалистичные бюджеты

```javascript
// lighthouserc.js
assert: {
  assertions: {
    "categories:performance": ["error", { minScore: 0.9 }],
    "largest-contentful-paint": ["error", { maxNumericValue: 2500 }],
  },
}
```

### 2. Тестируйте на реальных устройствах

```ts
// Playwright — эмуляция мобильных устройств
test.use({
  ...devices["iPhone 12"],
});
```

### 3. Используйте thresholds в k6

```javascript
export const options = {
  thresholds: {
    http_req_duration: ["p(95)<500"],
    errors: ["rate<0.01"],
  },
};
```

### 4. Мониторьте в production

```tsx
import { onLCP, onCLS } from "web-vitals";

onLCP(sendToAnalytics);
onCLS(sendToAnalytics);
```

### 5. Автоматизируйте в CI

```yaml
- name: Run Lighthouse CI
  run: npx lhci autorun
```

---

## Антипаттерны

### 1. Игнорирование производительности

```ts
// ❌ Нет тестов производительности
it("renders page", async () => {
  await page.goto("/");
  // Нет проверок метрик
});

// ✅ Проверка метрик
it("renders page within time limit", async () => {
  const start = Date.now();
  await page.goto("/");
  const duration = Date.now() - start;
  expect(duration).toBeLessThan(3000);
});
```

### 2. Нереалистичные бюджеты

```javascript
// ❌ Слишком строгий бюджет
"categories:performance": ["error", { minScore: 1.0 }]

// ✅ Реалистичный бюджет
"categories:performance": ["error", { minScore: 0.9 }]
```

### 3. Тестирование только в идеальных условиях

```ts
// ❌ Тестирование только на десктопе
test("desktop performance", async ({ page }) => { ... });

// ✅ Тестирование на разных устройствах
test("mobile performance", async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 667 });
  // ...
});
```

### 4. Игнорирование метрик в production

```ts
// ❌ Нет мониторинга в production
export default function App() {
  return <Router />;
}

// ✅ Мониторинг Web Vitals
import { initWebVitals } from "./analytics";

if (typeof window !== "undefined") {
  initWebVitals();
}
```

### 5. Оптимизация без измерений

```ts
// ❌ Оптимизация без данных
React.memo(ExpensiveComponent); // Без профилирования

// ✅ Оптимизация на основе данных
// 1. Профилирование в DevTools
// 2. Выявление узких мест
// 3. Целевая оптимизация
```
