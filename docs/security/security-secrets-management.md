# Управление секретами во фронтенде: от .env до Vault

## Содержание

1. [Что такое секрет и почему фронтенд — не надёжное хранилище](#что-такое-секрет-и-почему-фронтенд--не-надёжное-хранилище)
2. [Как секреты утекают в клиентский бандл](#как-секреты-утекают-в-клиентский-бандл)
3. [Next.js: NEXT_PUBLIC_ и серверные переменные](#nextjs-next_public_-и-серверные-переменные)
4. [Nuxt: runtimeConfig и appConfig](#nuxt-runtimeconfig-и-appconfig)
5. [Vue и React без метафреймворка](#vue-и-react-без-метафреймворка)
6. [Архитектура: серверный прокси](#архитектура-серверный-прокси)
7. [Хранилища секретов](#хранилища-секретов)
8. [Ротация секретов](#ротация-секретов)
9. [Защита от утечек в git](#защита-от-утечек-в-git)
10. [Чек-лист управления секретами](#чек-лист-управления-секретами)
11. [Терминология](#терминология)

---

## Что такое секрет и почему фронтенд — не надёжное хранилище

**Секрет** — это любая конфиденциальная информация, которая даёт доступ к ресурсам, данным или сервисам. Примеры:

- API-ключи (Stripe, SendGrid, Maps, OpenAI);
- пароли к базам данных;
- приватные ключи JWT;
- OAuth client_secret;
- токены CI/CD.

Фундаментальное правило веба: **всё, что отправляется в браузер, может быть прочитано**. JavaScript-бандл не является чёрным ящиком. Любой пользователь может:

- открыть DevTools → Sources;
- посмотреть Network-запросы;
- найти строки в бандле через поиск.

Поэтому секреты нельзя хранить в клиентском коде. Это не «плохая практика» — это архитектурная ошибка.

Аналогия: нельзя написать пин-код от банковской карты на самой карте. Даже если карта у вас в кошельке, любой, кто её увидит, узнает пин-код.

---

## Как секреты утекают в клиентский бандл

### Через переменные окружения

В Next.js любая переменная с префиксом `NEXT_PUBLIC_` встраивается в клиентский бандл на этапе сборки.

```env
# .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_STRIPE_PUBLIC_KEY=pk_live_...
STRIPE_SECRET_KEY=sk_live_...
```

`STRIPE_SECRET_KEY` доступен только на сервере. `NEXT_PUBLIC_STRIPE_PUBLIC_KEY` попадёт в бандл.

### Через импорт серверного кода в клиентский компонент

```tsx
// ❌ Опасно: серверная константа попадает в клиент
'use client';
import { SECRET_KEY } from '@/config/server';
```

Даже если файл называется `server.ts`, при импорте в клиентский компонент его содержимое может попасть в бандл.

### Через логи и ошибки

```ts
// ❌ Не делайте так
try {
  await api.call();
} catch (e) {
  console.error(e); // может содержать секреты
  Sentry.captureException(e); // может отправить секреты в Sentry
}
```

### Через source maps

Если source maps загружаются в production, злоумышленник может легче понять структуру приложения и найти чувствительные данные.

---

## Next.js: NEXT_PUBLIC_ и серверные переменные

### Правила

- `NEXT_PUBLIC_*` — доступны на клиенте и сервере, встраиваются в бандл.
- Без префикса — только на сервере.

### Пример правильного разделения

```env
# .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_GA_ID=G-1234567890
STRIPE_SECRET_KEY=sk_live_...
DATABASE_URL=postgresql://...
```

### Пример неправильного использования

```tsx
'use client';

export function PaymentForm() {
  // ❌ Секретный ключ в клиентском компоненте
  const stripe = new Stripe(process.env.NEXT_PUBLIC_STRIPE_SECRET_KEY!);
}
```

### Правильная архитектура

```ts
// app/api/create-payment-intent/route.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: Request) {
  const { amount } = await req.json();
  const paymentIntent = await stripe.paymentIntents.create({ amount, currency: 'usd' });
  return Response.json({ clientSecret: paymentIntent.client_secret });
}
```

```tsx
// components/PaymentForm.tsx
'use client';
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);
```

---

## Nuxt: runtimeConfig и appConfig

Nuxt 3 предоставляет `runtimeConfig` для серверных секретов и `appConfig` для публичной конфигурации.

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // Серверные секреты
    apiSecret: process.env.API_SECRET,
    stripeSecretKey: process.env.STRIPE_SECRET_KEY,
    public: {
      // Публичные значения
      apiBase: process.env.NUXT_PUBLIC_API_BASE,
      stripePublicKey: process.env.NUXT_PUBLIC_STRIPE_PUBLIC_KEY,
    },
  },
});
```

### Использование

```vue
<script setup>
const config = useRuntimeConfig();
// config.public.apiBase — доступен на клиенте
// config.apiSecret — только на сервере
</script>
```

### Серверный API route

```ts
// server/api/payment.post.ts
import Stripe from 'stripe';

export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig(event);
  const stripe = new Stripe(config.stripeSecretKey);

  const body = await readBody(event);
  const paymentIntent = await stripe.paymentIntents.create(body);

  return { clientSecret: paymentIntent.client_secret };
});
```

---

## Vue и React без метафреймворка

Если вы используете чистый Vue или React (Vite, CRA, Webpack), важно понимать, как работает подстановка переменных окружения.

### Vite

```env
VITE_API_URL=https://api.example.com
```

```js
const apiUrl = import.meta.env.VITE_API_URL;
```

Любая переменная с префиксом `VITE_` попадает в клиентский бандл.

### Create React App

```env
REACT_APP_API_URL=https://api.example.com
```

Только переменные с `REACT_APP_` доступны в клиентском коде.

### Webpack DefinePlugin

```js
new webpack.DefinePlugin({
  'process.env.API_URL': JSON.stringify(process.env.API_URL),
});
```

Будьте осторожны: всё, что определено здесь, попадает в бандл.

---

## Архитектура: серверный прокси

Если фронтенду нужен доступ к API с секретным ключом, правильный путь — серверный прокси.

```
Клиент → Ваш сервер (Next.js / Nuxt / Express) → Внешний API с секретом
```

### Пример: отправка email через SendGrid

```ts
// app/api/send-email/route.ts
import sgMail from '@sendgrid/mail';

sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

export async function POST(req: Request) {
  const { to, subject, text } = await req.json();

  await sgMail.send({ to, from: 'noreply@example.com', subject, text });
  return Response.json({ ok: true });
}
```

Клиент отправляет запрос на `/api/send-email`, а сервер уже общается с SendGrid, используя секретный ключ.

---

## Хранилища секретов

Для production рекомендуется использовать специализированные хранилища:

| Сервис | Особенности |
|--------|-------------|
| **Vercel Environment Variables** | Встроено в Vercel, удобно для Next.js |
| **AWS Secrets Manager** | Интеграция с IAM, автоматическая ротация |
| **AWS Systems Manager Parameter Store** | Более лёгкая альтернатива Secrets Manager |
| **Azure Key Vault** | Для Azure-инфраструктуры |
| **Google Secret Manager** | Для GCP |
| **HashiCorp Vault** | Self-hosted, enterprise-функции |
| **Doppler** | Удобный интерфейс, много интеграций |
| **Infisical** | Open-source альтернатива |

### Преимущества хранилищ

- Централизованное управление.
- Аудит доступа (кто и когда просматривал секрет).
- Автоматическая ротация.
- Разделение окружений (dev/staging/prod).

---

## Ротация секретов

**Ротация** — это регулярная замена секретов. Почему это важно:

- Если секрет утёк, его время жизни ограничено.
- Снижается риск от уволенных сотрудников.
- Требование многих compliance-стандартов (SOC2, PCI DSS).

### Практика

- API-ключи: раз в 90 дней.
- Пароли к базам данных: раз в 90–180 дней.
- JWT-секреты: раз в год или при компрометации.

### Zero-downtime ротация

1. Генерируем новый секрет.
2. Поддерживаем старый и новый секрет одновременно в течение переходного периода.
3. Обновляем всех потребителей.
4. Отключаем старый секрет.

---

## Защита от утечек в git

### .gitignore

```gitignore
.env
.env.local
.env.*.local
.env.development
.env.production
secrets/
*.pem
*.key
```

### pre-commit hooks

Используйте инструменты для сканирования секретов перед коммитом:

- **git-secrets** (AWS);
- **truffleHog**;
- **gitleaks**;
- **GitGuardian**.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

### Если секрет всё-таки попал в git

1. Немедленно отзовите/смените секрет.
2. Удалите коммит из истории (`git filter-repo` или BFG Repo-Cleaner).
3. Убедитесь, что секрет больше не используется нигде.
4. Проанализируйте логи на предмет злоупотреблений.

**Никогда не думайте, что достаточно просто удалить файл в следующем коммите.** Git сохраняет историю.

---

## Чек-лист управления секретами

### Разработка

- [ ] Разделить переменные на публичные (`NEXT_PUBLIC_`, `VITE_`, `NUXT_PUBLIC_`) и серверные.
- [ ] Никогда не использовать серверные переменные в клиентских компонентах.
- [ ] Использовать серверный прокси для внешних API с секретами.
- [ ] Не логировать секреты и ошибки, которые их могут содержать.

### Инфраструктура

- [ ] Хранить секреты в специализированном хранилище, а не в `.env` на сервере.
- [ ] Использовать разные секреты для dev/staging/prod.
- [ ] Включить ротацию секретов.
- [ ] Настроить аудит доступа к секретам.

### Процессы

- [ ] Добавить `.env` файлы в `.gitignore`.
- [ ] Настроить pre-commit hooks для поиска секретов.
- [ ] Проводить регулярный скан репозитория на утечки.
- [ ] Иметь план реагирования на компрометацию секрета.

---

## Терминология

| Термин | Значение |
|--------|----------|
| **Secret** | Конфиденциальные данные, дающие доступ к ресурсам |
| **Environment variable** | Переменная окружения, доступная приложению |
| **Runtime config** | Конфигурация, доступная во время выполнения |
| **Vault** | Хранилище секретов с аудитом и ротацией |
| **Secret rotation** | Регулярная замена секретов |
| **Pre-commit hook** | Скрипт, выполняемый перед коммитом |
| **Source map** | Файл, связывающий минифицированный код с исходным |
| **Least privilege** | Принцип минимальных привилегий |
| **Server proxy** | Промежуточный сервер, скрывающий секреты от клиента |

---

## Что важно понимать

Управление секретами — это не только про `.env`. Это архитектурный вопрос: какие данные попадают в браузер, как они передаются, где хранятся и как часто меняются.

Для SOC2 и CASA важно показать:

- разделение секретов по окружениям;
- отсутствие секретов в клиентском коде;
- использование хранилищ и ротации;
- сканирование на утечки в git.

Фронтенд-разработчик должен уверенно объяснять, почему `NEXT_PUBLIC_` не может содержать секреты, и уметь спроектировать серверный прокси.
