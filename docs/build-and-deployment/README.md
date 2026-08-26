# Сборка, CI/CD и деплой

Раздел про весь путь от исходного кода до production: бандлеры, оптимизация бандла, CI/CD, стратегии деплоя, рендеринг, CDN, Edge и платформы хостинга.

## Начни здесь

1. **[Сборка и CI/CD](./build-tools-ci-cd.md)** — эволюция бандлеров, Webpack vs Vite, code splitting, tree shaking, GitHub Actions, Docker.
2. **[Стратегии рендеринга, SSG, CDN и Edge](./deployment-rendering-strategies.md)** — CSR, SSR, SSG, ISR, CDN, кэширование и Edge Functions. База, без которой непонятны остальные статьи.

## Сборка и оптимизация

- **[Продвинутая оптимизация бандла](./advanced-bundle-optimization.md)** — анализ, chunks, lazy loading, compression, bundle budget.
- **[Монорепозитории](./monorepos.md)** — workspaces, Turborepo, Nx, CI/CD в монорепо.

## Стратегии деплоя

- **[Стратегии деплоя](./deployment-strategies.md)** — blue-green, canary, feature flags, rollback, zero-downtime.

## Деплой фреймворков

- **[Деплой Next.js](./deployment-nextjs.md)** — `output: 'export'`, SSR/SSG/ISR, Edge Runtime, Image Optimization, Vercel, Docker.
- **[Деплой Nuxt 3](./deployment-nuxt.md)** — Nitro, prerender, SSR, edge-пресеты, `routeRules`, хостинги.

## Платформы

- **[Платформы деплоя](./deployment-platforms.md)** — Vercel, Netlify, Railway, Render, Fly.io, Cloudflare Pages/Workers, AWS/GCP/Azure. Сравнение, цены, ограничения.

## Как пользоваться

1. Пойми, как работает бандлер в твоём проекте.
2. Настрой CI/CD для проверок и деплоя.
3. Разберись со стратегиями рендеринга: CSR, SSR, SSG, ISR, CDN, Edge.
4. Изучи специфику деплоя своего фреймворка: Next.js или Nuxt.
5. Оптимизируй бандл и осваивай монорепозитории при росте проекта.
6. На собеседовании чаще всего спрашивают про Vercel, но знание альтернатив выделяет кандидата.
