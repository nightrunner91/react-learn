# Безопасность

Раздел объясняет, как защищать веб-приложения на уровне фронтенда: от классических XSS и CSRF до Server Actions, CSP, секретов, авторизации, безопасности зависимостей и подготовки к security аудитам SOC2 и CASA.

## Начни здесь

1. **[Безопасность веб-приложений](./security.md)** — комплексный обзор угроз и защит.
2. **[Подготовка к SOC2 и CASA](./security-soc2-casa-workflows.md)** — как фронтендеру участвовать в security аудите.

## Углублённые темы

### Атаки и защита

- **[XSS: анатомия атаки](./security-xss-deep-dive.md)** — виды XSS, экранирование, санитизация, защита в React/Vue/Nuxt/Next.js, Trusted Types.
- **[CSRF: как браузер становится оружием](./security-csrf-deep-dive.md)** — механика атаки, SameSite cookies, CSRF-токены, double submit cookie.
- **[CSP: Content Security Policy](./security-csp-deep-dive.md)** — директивы, nonce, strict-dynamic, Report-Only, настройка в Next.js и Nuxt.
- **[HTTP-заголовки безопасности](./security-http-headers.md)** — HSTS, X-Frame-Options, COOP, COEP, CORP, Permissions-Policy и другие.

### Аутентификация, авторизация и секреты

- **[Аутентификация и авторизация](./security-authn-authz.md)** — сессии, JWT, OAuth 2.0, OIDC, PKCE, RBAC/ABAC, хранение токенов.
- **[Управление секретами](./security-secrets-management.md)** — `NEXT_PUBLIC_`, `runtimeConfig`, vaults, ротация, защита от утечек в git.

### Фреймворки и зависимости

- **[Безопасность Next.js](./security-nextjs.md)** — Server Components, Server Actions, middleware, Route Handlers, CSP, Open Redirect.
- **[Безопасность Vue и Nuxt](./security-vue-nuxt.md)** — `v-html`, refs, Nitro, `nuxt-security`, `runtimeConfig`, CSRF, CSP.
- **[Безопасность зависимостей и supply chain](./security-dependency-supply-chain.md)** — npm audit, lock-файлы, Snyk, Socket, SBOM, provenance.

## Пройди по порядку

1. XSS — виды, защита, экранирование.
2. CSRF — механизм атаки и токены.
3. Content Security Policy.
4. HTTP-заголовки безопасности.
5. Аутентификация и авторизация.
6. Управление секретами.
7. Безопасность Server Actions в Next.js / server routes в Nuxt.
8. Безопасность зависимостей.
9. Подготовка к SOC2 и CASA.

## Как пользоваться

1. Пойми базовые угрозы: XSS и CSRF.
2. Изучи CSP и security-заголовки — внедри в проект.
3. Проверь, где хранятся токены и секреты.
4. Регулярно аудируй зависимости на уязвимости.
5. Используй чек-листы перед релизом и аудитом.
