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

```bash
npm install -D @lhci/cli
```

### Конфигурация

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

```bash
npx lhci autorun
```

### GitHub Actions

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

```bash
npm install -D size-limit
```

### Конфигурация

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

```bash
npx size-limit
```

### GitHub Actions

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

```bash
# macOS
brew install k6

# Linux
sudo apt install k6

# Windows
choco install k6
```

### Базовый скрипт

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

```bash
k6 run load-test.js
```

### Продвинутый сценарий

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

```bash
npx artillery run load-test.yml
```

---

## Performance budgets в CI

### Что такое performance budgets

Performance budgets — лимиты на метрики производительности. Если лимит превышен — CI падает.

### Vite — настройка лимитов

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

React DevTools Profiler показывает, сколько времени тратится на рендеринг каждого компонента.

### Использование в тестах

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

## Тестирование времени выполнения

### Тестирование утилит

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

### web-vitals в production

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
