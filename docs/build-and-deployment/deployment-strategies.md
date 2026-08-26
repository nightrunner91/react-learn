# Стратегии деплоя — blue-green, canary releases, feature flags

## Содержание

1. [Почему «просто задеплоить» больше не работает](#почему-просто-задеплоить-больше-не-работает)
2. [Эволюция стратегий деплоя](#эволюция-стратегий-деплоя)
3. [Blue-Green Deployment — мгновенное переключение](#blue-green-deployment--мгновенное-переключение)
4. [Canary Releases — постепенный rollout](#canary-releases--постепенный-rollout)
5. [Feature Flags — контроль без деплоя](#feature-flags--контроль-без-деплоя)
6. [Rollback strategies — как быстро откатиться](#rollback-strategies--как-быстро-откатиться)
7. [Database migrations при деплое](#database-migrations-при-деплое)
8. [Zero-downtime deployment](#zero-downtime-deployment)
9. [Инструменты и платформы](#инструменты-и-платформы)
10. [Практические рекомендации](#практические-рекомендации)

---

## Почему «просто задеплоить» больше не работает

Раньше деплой был простым: остановили сервер, залили новый код, запустили сервер. Пользователи видели downtime в несколько минут, но это было приемлемо.

В современном вебе это не работает по нескольким причинам:

**1. Пользователи ожидают 24/7 доступности.** Если ваш сервис недоступен даже 5 минут, пользователи уходят к конкурентам. Исследования показывают, что 70% пользователей не вернутся, если сайт был недоступен.

**2. Релизы стали чаще.** Раньше релизы происходили раз в месяц или квартал. Сейчас компании деплоят десятки раз в день. Amazon деплоит каждые 11.7 секунд. Если каждый деплой — это downtime, бизнес не сможет работать.

**3. Риски выросли.** Один баг в production может стоить миллионы. В 2017 году Knight Capital потерял $440 миллионов за 45 минут из-за бага в деплое. В 2021 году Facebook был недоступен 6 часов из-за ошибки в конфигурации BGP.

**4. Масштаб усложнился.** Микросервисы, контейнеры, облака — всё это делает деплой сложнее. Нельзя просто «остановить сервер», когда у вас 100 серверов в разных регионах.

Поэтому появились стратегии деплоя, которые решают эти проблемы: blue-green, canary releases, feature flags. Они позволяют деплоить часто, безопасно и без downtime.

---

## Эволюция стратегий деплоя

### Эра ручных деплоев (до 2010)

**Как работало:**
1. Разработчик создаёт релизную ветку
2. QA тестирует в staging-окружении
3. В назначенное время (обычно ночью) ops-инженер останавливает production-сервер
4. Заливает новый код
5. Запускает сервер
6. Проверяет, что всё работает
7. Если что-то не так — откатывает назад

**Проблемы:**
- Downtime 10-30 минут
- Релизы редкие (раз в месяц) из-за риска
- «Deploy Fridays» запрещены (никто не хочет чинить production в выходные)
- Откат медленный (нужно заново останавливать сервер и заливать старый код)

### Эра автоматизированных деплоев (2010–2015)

**Как работало:**
1. CI/CD автоматически собирает и тестирует код
2. Деплой запускается одной командой
3. Load balancer переключает трафик на новые серверы
4. Если что-то не так — автоматический rollback

**Улучшения:**
- Меньше downtime (но всё ещё есть)
- Релизы чаще (раз в неделю)
- Меньше человеческого фактора

**Проблемы:**
- Всё ещё есть downtime при переключении
- Если баг попал в production — он влияет на всех пользователей
- Откат всё ещё медленный

### Эра непрерывных деплоев (2015–настоящее время)

**Как работает:**
1. Каждый push в main автоматически деплоится в production
2. Новые версии запускаются параллельно со старыми
3. Трафик переключается постепенно (canary) или мгновенно (blue-green)
4. Feature flags позволяют включать функции без деплоя
5. Автоматический rollback при проблемах

**Преимущества:**
- Zero downtime
- Релизы десятки раз в день
- Мгновенный откат
- Контроль над rollout (можно включить функцию только для 1% пользователей)

Именно в эту эру мы живём сейчас. И стратегии blue-green, canary releases, feature flags — это не «продвинутые техники», а **стандарт индустрии**.

---

## Blue-Green Deployment — мгновенное переключение

### Концепция

Blue-Green Deployment — это стратегия, при которой у вас есть два идентичных окружения: **Blue** (текущая версия) и **Green** (новая версия).

```
                    ┌─────────────┐
                    │ Load        │
                    │ Balancer    │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       ┌──────▼──────┐          ┌──────▼──────┐
       │   Blue      │          │   Green     │
       │  (v1.0)     │          │  (v2.0)     │
       │  Production │          │  Staging    │
       └─────────────┘          └─────────────┘
              │                         │
              └────────────┬────────────┘
                           │
                    ┌──────▼──────┐
                    │  Database   │
                    └─────────────┘
```

**Как это работает:**

1. **Blue** — активное окружение, обслуживает всех пользователей (версия 1.0)
2. **Green** — неактивное окружение, куда вы деплоите новую версию (версия 2.0)
3. Вы тестируете Green (можно дать доступ QA или ограниченному числу пользователей)
4. Когда всё готово, load balancer переключает весь трафик с Blue на Green
5. Blue остаётся доступным (на случай отката)
6. Если всё хорошо, через некоторое время Blue можно отключить или использовать для следующего деплоя

### Преимущества

**1. Zero downtime.** Переключение происходит мгновенно на уровне load balancer. Пользователи не замечают перехода.

**2. Мгновенный rollback.** Если что-то пошло не так, вы переключаете load balancer обратно на Blue. Это занимает секунды.

**3. Простота тестирования.** Green — это полноценное production-окружение. Вы можете тестировать новую версию в реальных условиях перед переключением трафика.

**4. Предсказуемость.** Вы точно знаете, какая версия активна. Нет «серой зоны», где работают обе версии одновременно.

### Недостатки

**1. Двойные ресурсы.** Вам нужно два полноценных окружения. Это дорого (в 2 раза больше серверов, баз данных и т.д.).

**2. Сложность с базой данных.** Если новая версия требует изменения схемы БД, нужно обеспечить обратную совместимость. Old version и new version должны работать с одной БД.

**3. Сессии пользователей.** Если пользователи авторизованы, их сессии должны быть доступны обоим окружениям. Обычно используется общее хранилище сессий (Redis, Memcached).

### Пример с nginx

```nginx
# nginx.conf
upstream blue {
    server blue-server:3000;
}

upstream green {
    server green-server:3000;
}

# Активное окружение — blue
upstream production {
    server blue-server:3000;
    # server green-server:3000; # Закомментировано
}

server {
    listen 80;
    
    location / {
        proxy_pass http://production;
    }
}

# Для переключения на green — раскомментируйте green и закомментируйте blue
```

Для автоматизации переключения используются инструменты вроде Kubernetes, AWS CodeDeploy, Vercel.

### Пример с Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Переключите на green для деплоя
  ports:
  - port: 80
    targetPort: 3000
```

Для переключения измените `version: blue` на `version: green` в Service и примените:

```bash
kubectl apply -f deployment.yaml
```

---

## Canary Releases — постепенный rollout

### Концепция

Canary Release — это стратегия, при которой новая версия разворачивается для небольшой группы пользователей (canary group), а затем постепенно rollout'ится на всех.

Название происходит от практики использования канареек в шахтах: если канарейка умирала, шахтёры знали, что в воздухе есть опасный газ. Так и здесь: если canary-группа сталкивается с проблемами, rollout останавливается.

```
                    ┌─────────────┐
                    │ Load        │
                    │ Balancer    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼───┐ ┌─────▼─────┐
       │  Stable     │ │Canary│ │  Stable   │
       │  (v1.0)     │ │(v2.0)│ │  (v1.0)   │
       │  95%        │ │ 5%   │ │  95%      │
       └─────────────┘ └──────┘ └───────────┘
              │            │            │
              └────────────┴────────────┘
                           │
                    ┌──────▼──────┐
                    │  Database   │
                    └─────────────┘
```

**Как это работает:**

1. **Шаг 1:** Деплоите новую версию для 1% пользователей (canary)
2. **Шаг 2:** Мониторите метрики (ошибки, latency, бизнес-метрики) в течение нескольких часов/дней
3. **Шаг 3:** Если всё хорошо — увеличиваете до 5%, затем 10%, 25%, 50%, 100%
4. **Шаг 4:** Если на любом шаге что-то идёт не так — автоматический rollback

### Преимущества

**1. Минимальный риск.** Если баг попал в production, он влияет только на 1% пользователей, а не на всех.

**2. Реальное тестирование.** Canary-группа использует новую версию в реальных условиях, а не в staging. Это выявляет проблемы, которые не видны в тестах.

**3. A/B тестирование.** Можно сравнить поведение пользователей на старой и новой версиях.

**4. Постепенный rollout.** Вы можете rollout'ить новую версию в течение нескольких дней, что снижает нагрузку на поддержку.

### Недостатки

**1. Сложность.** Нужно управлять двумя версиями одновременно, маршрутизацией трафика, мониторингом.

**2. Длительность.** Canary release может занять дни или недели, в отличие от blue-green, который происходит мгновенно.

**3. Консистентность данных.** Если пользователи взаимодействуют друг с другом (социальная сеть, чат), они могут видеть разные версии.

### Пример с Kubernetes и Istio

```yaml
# VirtualService для canary routing
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 95  # 95% трафика на стабильную версию
    - destination:
        host: myapp
        subset: canary
      weight: 5   # 5% трафика на canary

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: stable
    labels:
      version: v1.0
  - name: canary
    labels:
      version: v2.0
```

Для увеличения canary-процента измените `weight` и примените:

```bash
kubectl apply -f virtualservice.yaml
```

### Автоматический canary analysis

Современные инструменты (Flagger, Argo Rollouts) могут автоматически анализировать метрики и принимать решение о rollout или rollback:

```yaml
# Flagger Canary
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  progressDeadlineSeconds: 60
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 5
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
```

Flagger автоматически:
1. Увеличивает canary-процент на 5% каждую минуту
2. Проверяет success rate (должен быть > 99%)
3. Проверяет latency (должен быть < 500ms)
4. Если метрики в норме — продолжает rollout
5. Если метрики плохие — делает rollback

---

## Feature Flags — контроль без деплоя

### Концепция

Feature Flags (также известные как Feature Toggles) — это техника, при которой функции включаются/выключаются через конфигурацию, без необходимости деплоя.

Вместо того чтобы деплоить новую версию и надеяться, что всё работает, вы деплоите код с feature flag'ами и включаете функции постепенно.

```jsx
// Без feature flags
function Checkout() {
  return <NewCheckout />; // Новая версия для всех
}

// С feature flags
function Checkout() {
  const { isEnabled } = useFeatureFlag('new-checkout');
  
  if (isEnabled) {
    return <NewCheckout />;
  }
  
  return <OldCheckout />;
}
```

### Типы feature flags

**1. Release Toggles.** Включают новые функции для определённых пользователей. Используются для постепенного rollout.

```jsx
const { isEnabled } = useFeatureFlag('new-dashboard');

if (isEnabled) {
  return <NewDashboard />;
}
```

**2. Experiment Toggles.** Используются для A/B тестирования. Разные пользователи видят разные версии.

```jsx
const { variant } = useFeatureFlag('checkout-button');

if (variant === 'A') {
  return <Button color="blue">Buy Now</Button>;
} else {
  return <Button color="green">Purchase</Button>;
}
```

**3. Ops Toggles.** Используются для отключения функций при проблемах (например, если новая функция вызывает высокую нагрузку).

```jsx
const { isEnabled } = useFeatureFlag('expensive-analytics');

if (isEnabled) {
  trackExpensiveAnalytics();
}
```

**4. Permission Toggles.** Включают функции для определённых групп пользователей (бета-тестеры, премиум-пользователи, сотрудники).

```jsx
const { isEnabled } = useFeatureFlag('beta-feature');

if (isEnabled && user.isBetaTester) {
  return <BetaFeature />;
}
```

### Преимущества

**1. Разделение деплоя и релиза.** Вы можете деплоить код в production, но не включать функцию. Это снижает риск — код уже в production, протестирован, но функция ещё не видна пользователям.

**2. Мгновенный rollback.** Если функция вызывает проблемы, вы выключаете её через feature flag. Не нужно деплоить старую версию.

**3. A/B тестирование.** Вы можете тестировать разные версии функции на разных группах пользователей.

**4. Gradual rollout.** Вы можете включить функцию для 1% пользователей, затем 5%, 10%, 100%.

**5. Тестирование в production.** Вы можете тестировать новую функцию в реальном production-окружении, но только для определённых пользователей.

### Реализация

#### Простая реализация через localStorage

```jsx
import { useState, useEffect } from 'react';

function useFeatureFlag(flagName) {
  const [isEnabled, setIsEnabled] = useState(false);

  useEffect(() => {
    const value = localStorage.getItem(`feature_${flagName}`);
    setIsEnabled(value === 'true');
  }, [flagName]);

  return { isEnabled };
}

// Использование
function App() {
  const { isEnabled } = useFeatureFlag('new-checkout');
  
  return isEnabled ? <NewCheckout /> : <OldCheckout />;
}
```

#### Реализация через API

```jsx
import { useState, useEffect } from 'react';

function useFeatureFlag(flagName) {
  const [isEnabled, setIsEnabled] = useState(false);

  useEffect(() => {
    fetch(`/api/feature-flags/${flagName}`)
      .then(res => res.json())
      .then(data => setIsEnabled(data.enabled));
  }, [flagName]);

  return { isEnabled };
}
```

#### Использование LaunchDarkly (популярный сервис)

```bash
npm install launchdarkly-js-client-sdk
```

```jsx
import { useState, useEffect } from 'react';
import LDClient from 'launchdarkly-js-client-sdk';

const client = LDClient.initialize('your-client-side-id', {
  key: 'user-123',
  name: 'John Doe',
});

function useFeatureFlag(flagName) {
  const [isEnabled, setIsEnabled] = useState(false);

  useEffect(() => {
    client.on('ready', () => {
      setIsEnabled(client.variation(flagName, false));
    });

    client.on(`change:${flagName}`, (value) => {
      setIsEnabled(value);
    });
  }, [flagName]);

  return { isEnabled };
}
```

### Best practices

**1. Удаляйте старые flags.** Feature flags — не навсегда. После того как функция rollout'ится на 100%, удалите flag и старый код.

**2. Документируйте flags.** Ведите список всех feature flags, их назначения, владельцев.

**3. Ограничьте количество flags.** Слишком много flags делают код сложным. Используйте только для критичных функций.

**4. Тестируйте с flags и без.** Убедитесь, что код работает как с включённым, так и с выключенным flag.

---

## Rollback strategies — как быстро откатиться

### Почему rollback важен

Даже с лучшими стратегиями деплоя, баги попадают в production. Вопрос не «если», а «когда». И когда это происходит, скорость отката критична.

**Правило:** Если production сломан, каждая минута стоит денег. Быстрый rollback важнее, чем понимание, что именно сломалось. Сначала откатитесь, потом разбирайтесь.

### Стратегии rollback

**1. Blue-Green rollback.** Переключите load balancer обратно на Blue. Это занимает секунды.

**2. Canary rollback.** Уберите canary-версию и направьте весь трафик на стабильную версию.

**3. Feature flag rollback.** Выключите feature flag. Новая функция исчезнет, но код останется.

**4. Database rollback.** Если изменения в БД сломали приложение, откатите миграцию. Это сложнее и требует планирования.

### Автоматический rollback

Современные системы могут автоматически откатываться при обнаружении проблем:

```yaml
# GitHub Actions с автоматическим rollback
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - run: ./deploy.sh
      
      - name: Health check
        run: |
          sleep 30
          curl -f https://mysite.com/health || exit 1
      
      - name: Rollback on failure
        if: failure()
        run: ./rollback.sh
```

Если health check не проходит, workflow автоматически откатывается.

---

## Database migrations при деплое

### Проблема

Изменение схемы базы данных — одна из самых сложных частей деплоя. Если новая версия кода требует новой колонки в таблице, а старая версия не знает об этой колонке, что делать?

### Стратегии

**1. Expand-Contract pattern.**

**Expand:** Добавьте новую колонку, но не удаляйте старую. Новая версия пишет в обе колонки, старая версия продолжает работать со старой.

```sql
-- Шаг 1: Expand
ALTER TABLE users ADD COLUMN new_email VARCHAR(255);
```

**Migrate:** Перенесите данные из старой колонки в новую.

```sql
-- Шаг 2: Migrate
UPDATE users SET new_email = old_email WHERE new_email IS NULL;
```

**Contract:** После того как все версии используют новую колонку, удалите старую.

```sql
-- Шаг 3: Contract (после полного rollout)
ALTER TABLE users DROP COLUMN old_email;
```

**2. Backward-compatible migrations.**

Делайте миграции обратно совместимыми. Новая версия должна работать со старой схемой БД, старая версия должна работать с новой схемой.

```sql
-- Новая колонка должна быть nullable или иметь default value
ALTER TABLE users ADD COLUMN preferences JSONB DEFAULT '{}';
```

**3. Feature flags для миграций.**

Используйте feature flags для контроля над миграцией:

```jsx
const { isEnabled } = useFeatureFlag('new-schema');

if (isEnabled) {
  // Используйте новую схему
  const prefs = user.preferences;
} else {
  // Используйте старую схему
  const prefs = JSON.parse(user.old_preferences);
}
```

---

## Zero-downtime deployment

### Концепция

Zero-downtime deployment — это деплой без прерывания обслуживания. Пользователи не замечают, что происходит деплой.

### Как это работает

**1. Graceful shutdown.** Сервер перестаёт принимать новые запросы, но завершает текущие перед остановкой.

```js
// Node.js graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Shutting down gracefully...');
  
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
  
  // Force shutdown after 30 seconds
  setTimeout(() => {
    console.error('Could not close connections in time, forcefully shutting down');
    process.exit(1);
  }, 30000);
});
```

**2. Health checks.** Load balancer проверяет, что сервер готов принимать запросы, прежде чем направлять трафик.

```js
// Health check endpoint
app.get('/health', (req, res) => {
  if (isReady) {
    res.status(200).json({ status: 'ok' });
  } else {
    res.status(503).json({ status: 'not ready' });
  }
});
```

**3. Rolling updates.** Серверы обновляются по одному, а не все сразу. Load balancer направляет трафик только на готовые серверы.

```yaml
# Kubernetes rolling update
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1  # Максимум 1 дополнительный под
      maxUnavailable: 0  # Ни один под не должен быть недоступен
```

---

## Инструменты и платформы

### Vercel

Vercel автоматически использует blue-green deployment для всех проектов:

1. Каждый push создаёт новый deployment
2. Vercel тестирует новый deployment
3. Если всё хорошо, трафик переключается на новый deployment
4. Старый deployment остаётся доступным (для rollback)

```bash
# Деплой через Vercel CLI
vercel --prod
```

Vercel также поддерживает preview deployments для каждого PR.

### Netlify

Netlify работает аналогично Vercel:

1. Каждый push создаёт новый deploy
2. Netlify тестирует новый deploy
3. Atomic deploy гарантирует, что все файлы обновляются одновременно
4. Старый deploy остаётся доступным

```bash
# Деплой через Netlify CLI
netlify deploy --prod
```

### AWS CodeDeploy

AWS CodeDeploy поддерживает blue-green и in-place deployment для EC2, ECS, Lambda:

```yaml
# appspec.yml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "myapp"
          ContainerPort: 80
Hooks:
  - BeforeInstall: "scripts/before_install.sh"
  - AfterInstall: "scripts/after_install.sh"
  - AfterAllowTestTraffic: "scripts/run_tests.sh"
  - BeforeAllowTraffic: "scripts/before_allow_traffic.sh"
  - AfterAllowTraffic: "scripts/after_allow_traffic.sh"
```

### Kubernetes

Kubernetes поддерживает rolling updates, blue-green, canary через различные инструменты:

- **Kubernetes Deployments** — rolling updates из коробки
- **Istio** — canary releases с traffic splitting
- **Flagger** — автоматический canary analysis
- **Argo Rollouts** — продвинутые стратегии rollout

---

## Практические рекомендации

### Когда что использовать

**Blue-Green:**
- Когда вам нужен мгновенный rollback
- Когда у вас достаточно ресурсов для двух окружений
- Когда изменения не требуют сложных миграций БД

**Canary Releases:**
- Когда вы хотите минимизировать риск
- Когда вы хотите тестировать в production
- Когда у вас есть инструменты для автоматического анализа метрик

**Feature Flags:**
- Когда вы хотите разделить деплой и релиз
- Когда вы хотите делать A/B тестирование
- Когда вы хотите постепенный rollout

### Комбинирование стратегий

Лучшие результаты даёт комбинация стратегий:

1. **Blue-Green** для infrastructure-level изменений
2. **Canary** для постепенного rollout новых функций
3. **Feature Flags** для контроля над функциями

Пример:
1. Деплоите новую версию с feature flags (все flags выключены)
2. Canary rollout на 5% пользователей
3. Включаете feature flags для canary-группы
4. Мониторите метрики
5. Увеличиваете canary до 100%
6. Включаете feature flags для всех

### Checklist перед деплоем

- [ ] Все тесты проходят в CI
- [ ] Database migrations обратно совместимы
- [ ] Feature flags настроены и протестированы
- [ ] Health checks работают
- [ ] Rollback strategy готова
- [ ] Мониторинг настроен (ошибки, latency, бизнес-метрики)
- [ ] Уведомления настроены (Slack, email, SMS)
- [ ] Документация обновлена
- [ ] Команда знает о деплое и готова к поддержке

### Мониторинг после деплоя

После деплоя критически важно мониторить:

**1. Ошибки.** Используйте Sentry, LogRocket для отслеживания ошибок в реальном времени.

**2. Performance.** Используйте Lighthouse CI, Web Vitals для отслеживания производительности.

**3. Бизнес-метрики.** Конверсия, bounce rate, время на сайте — всё это может упасть после деплоя.

**4. Infrastructure.** CPU, memory, disk, network — убедитесь, что новая версия не потребляет больше ресурсов.

Если что-то из этого ухудшилось — rollback.

---

## Заключение

Стратегии деплоя — это не «продвинутые техники» для крупных компаний. Это **необходимость** для любого современного веб-приложения. Пользователи ожидают 24/7 доступности, бизнес требует частых релизов, и единственный способ совместить эти требования — использовать blue-green, canary releases и feature flags.

**Ключевые выводы:**

1. **Blue-Green Deployment** — мгновенное переключение между двумя окружениями. Простой и надёжный, но требует двойных ресурсов.

2. **Canary Releases** — постепенный rollout на небольшую группу пользователей. Минимизирует риск, но сложнее в реализации.

3. **Feature Flags** — включение/выключение функций без деплоя. Разделяет деплой и релиз, позволяет делать A/B тестирование.

4. **Rollback strategy** — обязательна. Если production сломан, скорость отката критична.

5. **Database migrations** — требуют планирования. Используйте expand-contract pattern и backward-compatible migrations.

6. **Zero-downtime** — стандарт индустрии. Пользователи не должны замечать деплой.

7. **Мониторинг** — критически важен после деплоя. Ошибки, performance, бизнес-метрики — всё должно отслеживаться.

Практикуйтесь: настройте preview deployments в Vercel или Netlify, добавьте feature flags в своё приложение, настройте автоматический rollback в CI. Эти навыки выделят вас среди других кандидатов на собеседовании.
