# GitLab CI/CD для фронтенд-тестов

GitLab CI/CD — один из самых распространённых инструментов автоматизации в компаниях, особенно в enterprise. Правильно настроенный pipeline ускоряет обратную связь, надёжно запускает тесты и даёт понятные отчёты.

Эта статья — про особенности GitLab CI/CD именно в контексте фронтенд-тестирования: unit, интеграционные и E2E.

---

## Содержание

1. [Структура pipeline](#структура-pipeline)
2. [Stages: lint → typecheck → unit → build → e2e](#stages-lint--typecheck--unit--build--e2e)
3. [Cache vs artifacts](#cache-vs-artifacts)
4. [Параллельный запуск unit-тестов](#параллельный-запуск-unit-тестов)
5. [E2E с Playwright в Docker](#e2e-с-playwright-в-docker)
6. [Шардинг E2E в GitLab CI](#шардинг-e2e-в-gitlab-ci)
7. [Services для баз данных и моков](#services-для-баз-данных-и-моков)
8. [needs и DAG-зависимости](#needs-и-dag-зависимости)
9. [Rules: когда запускать тесты](#rules-когда-запускать-тесты)
10. [JUnit-отчёты и coverage](#junit-отчёты-и-coverage)
11. [Безопасность: переменные и secrets](#безопасность-переменные-и-secrets)
12. [Полный пример .gitlab-ci.yml](#полный-пример-gitlab-ci-yml)
13. [Чеклист](#чеклист)

---

## Структура pipeline

Классический фронтенд-pipeline выглядит так:

```
lint → typecheck → unit → build → e2e
```

| Stage | Что происходит | Время |
|---|---|---|
| `lint` | ESLint, Prettier, commitlint | секунды |
| `typecheck` | `tsc --noEmit` | секунды–минуты |
| `unit` | Vitest/Jest + React Testing Library | минуты |
| `build` | Сборка production-бандла | минуты |
| `e2e` | Playwright/Cypress против сборки | минуты–десятки минут |

Стадии выполняются последовательно, но job'ы внутри одной стадии — параллельно.

---

## Stages: lint → typecheck → unit → build → e2e

```yaml
stages:
  - lint
  - typecheck
  - unit
  - build
  - e2e

lint:
  stage: lint
  image: node:20-alpine
  script:
    - npm ci
    - npm run lint

-typecheck:
  stage: typecheck
  image: node:20-alpine
  script:
    - npm ci
    - npm run typecheck

unit:
  stage: unit
  image: node:20-alpine
  script:
    - npm ci
    - npm run test:unit
  artifacts:
    reports:
      junit: reports/unit-tests.xml
    paths:
      - coverage/

build:
  stage: build
  image: node:20-alpine
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
```

---

## Cache vs artifacts

| | Cache | Artifacts |
|---|---|---|
| Назначение | Ускорение установки зависимостей | Передача файлов между job'ами |
| Пример | `node_modules/`, `.npm/` | `dist/`, отчёты тестов |
| Жизнь | Между pipeline'ами | Внутри одного pipeline |
| Управление | `cache:` | `artifacts:` |

### Пример cache

```yaml
default:
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
      - .npm/
    policy: pull-push
```

### Пример artifacts

```yaml
build:
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 day
```

> 💡 `cache` — для `node_modules`. `artifacts` — для `dist/` и отчётов. Не путай.

---

## Параллельный запуск unit-тестов

Vitest и Jest умеют запускать тесты в несколько потоков. В GitLab CI важно правильно выделить ресурсы.

```yaml
unit:
  stage: unit
  image: node:20-alpine
  variables:
    NODE_OPTIONS: "--max-old-space-size=4096"
  script:
    - npm ci
    - npm run test:unit -- --reporter=junit --outputFile=reports/unit-tests.xml
  parallel:
    matrix:
      - SHARD_INDEX: [1, 2, 3, 4]
        SHARD_TOTAL: [4]
  artifacts:
    reports:
      junit: reports/unit-tests.xml
```

Для Vitest шардинг настраивается через `--shard=${SHARD_INDEX}/${SHARD_TOTAL}`.

---

## E2E с Playwright в Docker

Playwright предоставляет официальные образы со всеми браузерами и зависимостями.

```yaml
e2e:
  stage: e2e
  image: mcr.microsoft.com/playwright:v1.46.0-jammy
  script:
    - npm ci
    - npx playwright install-deps
    - npm run build
    - npm run test:e2e
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    expire_in: 3 days
```

> ⚠️ Образ `playwright:vX.Y.Z-jammy` уже содержит браузеры. `npx playwright install` обычно не нужен, но `install-deps` может потребоваться для системных библиотек.

---

## Шардинг E2E в GitLab CI

```yaml
e2e:
  stage: e2e
  image: mcr.microsoft.com/playwright:v1.46.0-jammy
  parallel:
    matrix:
      - SHARD_INDEX: [1, 2, 3]
        SHARD_TOTAL: [3]
  script:
    - npm ci
    - npm run build
    - npx playwright test --shard=$SHARD_INDEX/$SHARD_TOTAL
  artifacts:
    when: always
    paths:
      - blob-reports/
    expire_in: 1 day

merge-reports:
  stage: e2e
  image: mcr.microsoft.com/playwright:v1.46.0-jammy
  needs:
    - job: e2e
      artifacts: true
  script:
    - npx playwright merge-reports --reporter html ./blob-reports
  artifacts:
    paths:
      - playwright-report/
    expire_in: 3 days
```

---

## Services для баз данных и моков

Если E2E используют реальную БД, подключи её как service:

```yaml
e2e-with-db:
  stage: e2e
  image: node:20-alpine
  services:
    - name: postgres:15-alpine
      alias: postgres
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: "postgresql://test:test@postgres:5432/test"
  script:
    - npm ci
    - npm run db:migrate
    - npm run db:seed
    - npm run test:e2e
```

Services доступны по alias. Внутри job'ы хост `postgres` резолвится в контейнер PostgreSQL.

---

## needs и DAG-зависимости

По умолчанию job'ы одной стадии ждут завершения всех job'ов предыдущей стадии. `needs` позволяет строить DAG и запускать job'ы раньше.

```yaml
build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/

e2e-smoke:
  stage: e2e
  needs:
    - job: build
      artifacts: true
  script:
    - npx playwright test --grep @smoke

e2e-regression:
  stage: e2e
  needs:
    - job: build
      artifacts: true
  script:
    - npx playwright test --grep-invert @slow
```

`build` выполнится один раз, а оба E2E-job'а запустятся параллельно, как только сборка готова.

---

## Rules: когда запускать тесты

Не все тесты нужны на каждое изменение.

```yaml
lint:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

unit:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

e2e:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_LABELS =~ /run-e2e/'
  when: always
```

### Варианты условий

| Условие | Когда запускать |
|---|---|
| `CI_PIPELINE_SOURCE == "merge_request_event"` | На каждый MR |
| `CI_COMMIT_BRANCH == "main"` | После merge в main |
| `CI_MERGE_REQUEST_LABELS =~ /run-e2e/` | Только если на MR повесили label |
| `CI_COMMIT_MESSAGE =~ /\[skip ci\]/` | Не запускать (стандартное соглашение) |

---

## JUnit-отчёты и coverage

GitLab умеет парсить JUnit XML и показывать упавшие тесты прямо в MR.

### Vitest → JUnit

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    reporters: ["default", "junit"],
    outputFile: {
      junit: "./reports/unit-tests.xml",
    },
  },
});
```

### Playwright → JUnit

```ts
// playwright.config.ts
export default defineConfig({
  reporter: [
    ["html"],
    ["junit", { outputFile: "reports/e2e-results.xml" }],
  ],
});
```

### Coverage

```yaml
unit:
  script:
    - npm run test:unit -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    paths:
      - coverage/
```

> 💡 Регулярка для coverage зависит от формата вывода Vitest. Проверь вывод `npm run test:unit -- --coverage`.

---

## Безопасность: переменные и secrets

- Храни secrets в **CI/CD Variables** (Settings → CI/CD → Variables).
- Для production используй **Protected variables**.
- Никогда не коммить `.env` файлы с реальными ключами.
- Для E2E используй отдельные тестовые аккаунты, не продовые.

```yaml
variables:
  TEST_USER_EMAIL: $TEST_USER_EMAIL
  TEST_USER_PASSWORD: $TEST_USER_PASSWORD
```

---

## Полный пример .gitlab-ci.yml

```yaml
stages:
  - lint
  - typecheck
  - unit
  - build
  - e2e

default:
  image: node:20-alpine
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
      - .npm/

variables:
  NODE_OPTIONS: "--max-old-space-size=4096"
  FF_USE_FASTZIP: "true"

lint:
  stage: lint
  script:
    - npm ci --prefer-offline
    - npm run lint
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

typecheck:
  stage: typecheck
  script:
    - npm ci --prefer-offline
    - npm run typecheck
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

unit:
  stage: unit
  script:
    - npm ci --prefer-offline
    - npm run test:unit -- --reporter=junit --outputFile=reports/unit-tests.xml
  artifacts:
    when: always
    reports:
      junit: reports/unit-tests.xml
    paths:
      - coverage/
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

build:
  stage: build
  script:
    - npm ci --prefer-offline
    - npm run build
  artifacts:
    paths:
      - dist/
      - package.json
    expire_in: 1 hour
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

e2e-smoke:
  stage: e2e
  image: mcr.microsoft.com/playwright:v1.46.0-jammy
  needs:
    - job: build
      artifacts: true
  script:
    - npm ci --prefer-offline
    - npx playwright install-deps
    - npx playwright test --grep @smoke
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    reports:
      junit: reports/e2e-results.xml
    expire_in: 3 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

e2e-regression:
  stage: e2e
  image: mcr.microsoft.com/playwright:v1.46.0-jammy
  needs:
    - job: build
      artifacts: true
  parallel:
    matrix:
      - SHARD_INDEX: [1, 2, 3]
        SHARD_TOTAL: [3]
  script:
    - npm ci --prefer-offline
    - npx playwright install-deps
    - npx playwright test --shard=$SHARD_INDEX/$SHARD_TOTAL --grep-invert @slow
  artifacts:
    when: always
    paths:
      - blob-reports/
    expire_in: 1 day
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_LABELS =~ /run-e2e/'
```

---

## Чеклист

- [ ] Pipeline разбит на stages: lint → typecheck → unit → build → e2e.
- [ ] `node_modules` кешируется между pipeline'ами.
- [ ] Production-сборка передаётся в E2E через artifacts.
- [ ] Unit-тесты генерируют JUnit-отчёты.
- [ ] E2E запускаются в официальном Docker-образе Playwright.
- [ ] E2E шардируются для ускорения.
- [ ] Используются `needs` для параллельного запуска независимых job'ов.
- [ ] `rules` запускают тяжёлые E2E не на каждый MR, а по необходимости.
- [ ] Secrets хранятся в CI/CD Variables, не в репозитории.
- [ ] Coverage и отчёты о тестах видны в MR.
