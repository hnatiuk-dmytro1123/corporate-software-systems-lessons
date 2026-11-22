# ЛР 4. Контейнеризація, CI/CD та розгортання — модуль Orders

**Студент:** Василь Гордашевський. **Група:** не вказана. **Дата:** 2026-11-22.  
**ШІ-асистент:** ChatGPT (GPT-6). **Діалог:** додати посилання або експорт перед поданням.

> Проєктний документ. Образи, pipeline і розгортання не запускалися; розмір і часи є оцінками.

## 0. Вихідні дані

- ADR-0004, наданий студентом: TypeScript, Node.js 22 LTS, NestJS, PostgreSQL 17+, Prisma ORM.
- NFR-ORD-DR-01: RPO ≤24 год, RTO ≤4 год.
- ЛР3: RabbitMQ, PostgreSQL, transactional outbox/inbox, ACK після COMMIT; Orders публікує orders.order.confirmed та orders.order.cancelled.
- GitHub/GHCR — організація на Free; Олег працює два дні на тиждень, DevOps-команди немає. Офісний сервер має один ISP і ДБЖ на ~20 хв при відключеннях до 4 год.
- Для релізної стратегії прийнято надане студентом навантаження до 4 замовлень/хв.
- Для приватного репозиторію GitHub Free не має protected environment approvals/secrets. Production deploy виконує Олег із сервера за перевіреним SHA; production SSH key не зберігається в GitHub.

## 1. Середовища та C4 Deployment

### Local

~~~mermaid
flowchart TB
  subgraph LAPTOP["Ноутбук розробника"]
    CLIENT["Браузер / API-клієнт"]
    subgraph COMPOSE["Docker Compose · kavohub-net"]
      RP["Локальний proxy / localhost"]
      ORD["Orders API · NestJS · :8102"]
      MIG["orders-migrate · Prisma CLI"]
      PG[("PostgreSQL 17 · orders schema")]
      RMQ[("RabbitMQ · kavohub.events")]
      KC["Keycloak"]
      MOCK["Prism / WireMock<br/>NBU, CRM, Inventory, payments"]
      RP --> ORD
      MIG --> PG
      ORD --> PG
      ORD --> RMQ
      ORD --> KC
      ORD --> MOCK
    end
    CLIENT --> RP
  end
~~~

Локальні credentials тестові; .env не комітиться. Зовнішні API підміняються mocks, коли можливо.

### Staging

~~~mermaid
flowchart TB
  USERS["Тестувальники / бізнес"]
  GH["GitHub Actions · image SHA"]
  DNS["staging.<домен>"]
  HTTPS["HTTPS :443"]
  subgraph VPS["Окремий staging VPS"]
    FW["Firewall · SSH за ключами"]
    RP["Caddy / Nginx · TLS"]
    subgraph NET["Внутрішня Docker network kavohub-net"]
      ORD["Orders API :8102"]
      MIG["orders-migrate"]
      PG[("PostgreSQL 17 · staging")]
      RMQ[("RabbitMQ")]
      KC["Keycloak"]
      OTHER["CRM :8101 · Inventory :8103"]
      MOCK["Prism / WireMock або sandbox"]
      RP --> ORD
      MIG --> PG
      ORD --> PG
      ORD --> RMQ
      ORD --> KC
      ORD --> OTHER
      ORD --> MOCK
    end
    FW --> RP
  end
  USERS --> DNS --> HTTPS --> FW
  GH -->|"deploy після publish"| FW
~~~

Окремі staging БД, черги, тестові дані та credentials. Зовнішній вхід лише через HTTPS proxy; SSH — за ключем. PR jobs не мають deploy credentials.

### Production

~~~mermaid
flowchart TB
  SALES["Менеджери з телефонів"]
  HOOK["Платіжні webhooks"]
  OLEG["Олег · deploy за перевіреним SHA"]
  S3[("Зашифровані резервні копії")]
  EXT["Odoo · НБУ · Нова Пошта"]
  DNS["orders.<домен>"]
  HTTPS["HTTPS :443"]
  subgraph PROD["Окремий європейський production VPS"]
    FW["Firewall · 443 публічно · SSH за ключами"]
    RP["Caddy / Nginx · TLS"]
    subgraph NET["Внутрішня Docker network kavohub-net"]
      ORD["Orders API · NestJS · Node.js 22"]
      MIG["Одноразовий orders-migrate"]
      PG[("PostgreSQL 17 · Orders schema/user")]
      RMQ[("RabbitMQ · durable queues + volume")]
      KC["Keycloak"]
      OTHER["CRM, Inventory, Billing, інші модулі"]
      RP --> ORD
      RP --> OTHER
      MIG --> PG
      ORD --> PG
      ORD --> RMQ
      ORD --> KC
      ORD --> OTHER
    end
    FW --> RP
    BK["Нічний PostgreSQL backup · контроль + alert"]
    PG --> BK --> S3
  end
  SALES --> DNS --> HTTPS --> FW
  HOOK --> DNS
  OLEG -->|"SSH / deploy.sh <sha>"| FW
  ORD -->|"HTTPS egress"| EXT
  OTHER -->|"HTTPS egress"| EXT
~~~

PostgreSQL, RabbitMQ та внутрішні API не мають публічних портів. Для RPO контролюється кожна копія; для RTO треба відпрацювати відновлення БД, конфігурації, секретів і черг/повторної публікації.

## 2. Production ADR і середовища

### ADR-04-01 — Середовище production

**Статус:** обрано умовно; кошторис і план відновлення підтвердити до запуску.

| Варіант | Переваги | Ризики | Висновок |
|---|---|---|---|
| Офісний сервер | Без нової щомісячної оренди; Docker уже є | Відключення довші за ДБЖ, один ISP, зовнішній доступ; ризик порушення RTO | Відхилено |
| Managed Kubernetes / PaaS | Керовані функції та масштабування | Складніше підтримувати без DevOps; ризик перевищити $100/міс | Відхилено на цьому етапі |
| Європейський VPS + Compose, окремий staging VPS | Звичний Compose; незалежність від офісу; середовища ізольовані | VPS один — SPOF; кошторис і RTO треба підтвердити | Обрано умовно |

Оцінка близько $60/міс — попередня, не затверджена ціна провайдера. Перевірити тариф обох VPS, object storage, IP, податки й додаткові послуги в межах $100/міс. Один VPS не має автоматичного failover; рішення покладається на відновлення в межах RTO ≤4 год.

### Таблиця середовищ

| Середовище | Адреси / вузли | Доступ | Секрети |
|---|---|---|---|
| Local | Orders localhost:8102; PostgreSQL localhost:5432; RabbitMQ localhost:5672; Compose DNS | Розробник | Тестовий .env поза Git; .env.example без секретів |
| Staging | Окремий VPS; https://staging.<домен> (призначити); proxy :443 | Тестувальники HTTPS; SSH deploy-користувач за ключем | Окремі credentials; SSH key як repository secret лише staging job; runtime secrets на хості |
| Production | Окремий VPS; https://orders.<домен> (призначити); лише proxy :443 | Менеджери/webhooks HTTPS; deploy SSH — Олег | Runtime .env на хості; production SSH key у Bitwarden, не GitHub; GHCR pull credential read-only |

### Compose домовленості

| Елемент | Домовленість |
|---|---|
| Network | kavohub-net внутрішня; БД і RabbitMQ без host ports |
| RabbitMQ | Exchange kavohub.events; orders.events, orders.events.retry, orders.events.dlq |
| Сусідні модулі | http://inventory:8103, http://crm:8101, внутрішній DNS |
| Keycloak | Внутрішній http://keycloak:8080; KEYCLOAK_ISSUER збігається з iss JWT; JWKS URL окремий |
| PostgreSQL | Окремі схема й користувач Orders; доступ до чужих схем заборонено |
| Orders | :8102; на VPS host port відсутній, reverse proxy у kavohub-net |
| Міграції | Окремий одноразовий orders-migrate перед новим API |

## 3. Образ, health/readiness, конфігурація, Compose

### Dockerfile

Використовується Node 22 LTS із сумісною Debian/OpenSSL-основою в build/runtime. Для релізу закріпити актуальний patch і digest базового образу. Prisma потребує OpenSSL, zlib і libc, тому slim образ явно встановлює OpenSSL та CA certificates. [Вимоги Prisma до Linux](https://www.prisma.io/docs/orm/v6/reference/system-requirements)

~~~dockerfile
# syntax=docker/dockerfile:1
# У CI базовий Node 22 patch треба закріпити за digest.
FROM node:22-bookworm-slim AS base
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends openssl ca-certificates && rm -rf /var/lib/apt/lists/*

FROM base AS build
COPY package.json package-lock.json ./
RUN npm ci
COPY prisma ./prisma
RUN npx prisma generate --schema=prisma/schema.prisma
COPY . .
RUN npm run build
RUN npm prune --omit=dev

FROM base AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --chown=node:node --from=build /app/package.json ./package.json
COPY --chown=node:node --from=build /app/node_modules ./node_modules
COPY --chown=node:node --from=build /app/dist ./dist
USER node
EXPOSE 8102
HEALTHCHECK --interval=10s --timeout=3s --start-period=20s --retries=3 CMD node -e "fetch('http://127.0.0.1:8102/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/main.js"]

# Окремий migration image із Prisma CLI.
FROM base AS migrate
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY prisma ./prisma
USER node
CMD ["npx", "prisma", "migrate", "deploy", "--schema=prisma/schema.prisma"]
~~~

Build генерує Prisma Client і dist, потім прибирає dev dependencies; runtime копіює dist і production node_modules. CLI лишається в migration target. Шлях до schema/config узгодити з Prisma у lockfile. 180–220 МБ — непідтверджена оцінка. .dockerignore скорочує build context, а не безпосередньо фінальний образ.

### .dockerignore

~~~text
.git
.gitignore
node_modules
dist
coverage
.nyc_output
test
tests
*.spec.ts
*.e2e-spec.ts
*.log
npm-debug.log*
.env*
!.env.example
.npmrc
*.pem
*.key
.DS_Store
~~~

Секрети не потрапляють у build context. .env.example не копіюється в образ.

### Перевірки /health і /ready

| Перевірка | Що перевіряє | Тайм-аут / результат |
|---|---|---|
| /health | Живий процес Node і event loop, без залежностей | 200, якщо процес відповідає; без JWT |
| /ready — PostgreSQL | DB connection і SELECT 1 | До 2 с; 503 при недоступності |
| /ready — RabbitMQ | Стан наявного connection/channel; не відкривати новий канал на probe | До 2 с; 503 при закритому channel |
| /ready — JWKS | Придатний локальний кеш ключів JWT | Локально, без запиту Keycloak на кожен probe; 503, якщо JWT неможливо перевірити |
| CRM, Inventory, НБУ | Не включені в загальну readiness | Збій обробляється на рівні операції за ЛР3 |

Без НБУ можливі частина UAH-сценаріїв; EUR Quote без курсу — fail-closed. Відсутність CRM replica може блокувати конкретну операцію, не весь API. Health endpoints не публікувати напряму в Інтернет.

### Змінні середовища

| Змінна | Призначення | Local | Staging/production | Secret |
|---|---|---|---|---|
| NODE_ENV, PORT, LOG_LEVEL | Режим, порт, JSON logs | development, 8102, debug | production, 8102, info | Ні |
| DATABASE_URL | Prisma/PostgreSQL URL | Тестовий URL | Окреме значення середовища | Так |
| RABBITMQ_URL | Broker URL/credentials | Локальні тестові | Окремі credentials | Так |
| RABBITMQ_EXCHANGE / QUEUE | Exchange і черга | kavohub.events / orders.events | Ті самі імена | Ні |
| CRM_BASE_URL | CRM endpoint | http://crm:8101 | Внутрішній DNS | Ні |
| INVENTORY_BASE_URL | Inventory endpoint | http://inventory:8103 | Внутрішній DNS | Ні |
| NBU_BASE_URL | API курсу | WireMock або НБУ | HTTPS НБУ | Ні |
| NBU_CONNECT_TIMEOUT_MS / NBU_RESPONSE_TIMEOUT_MS | Тайм-аути | 1000 / 3000 | 1000 / 3000 | Ні |
| KEYCLOAK_ISSUER / KEYCLOAK_JWKS_URL | Перевірка JWT | Локальний realm | Issuer токена / окремий JWKS URL | Ні |
| KEYCLOAK_CLIENT_ID | Client svc-orders | svc-orders-local | Окремий client | Ні |
| KEYCLOAK_CLIENT_SECRET | Client credentials | Локальна заглушка | Secret середовища | Так |
| HTTP_CLIENT_TIMEOUT_MS | Ліміт HTTP-виклику | 3000 | За стиками ЛР3 | Ні |

### .env.example

~~~dotenv
NODE_ENV=development
PORT=8102
LOG_LEVEL=debug
DATABASE_URL=postgresql://orders_app:local-only@postgres:5432/kavohub?schema=orders
RABBITMQ_URL=amqp://orders_app:local-only@rabbitmq:5672/kavohub
RABBITMQ_EXCHANGE=kavohub.events
RABBITMQ_QUEUE=orders.events
CRM_BASE_URL=http://crm:8101
INVENTORY_BASE_URL=http://inventory:8103
NBU_BASE_URL=https://bank.gov.ua/NBUStatService/v1/statdirectory/exchange
NBU_CONNECT_TIMEOUT_MS=1000
NBU_RESPONSE_TIMEOUT_MS=3000
KEYCLOAK_ISSUER=http://keycloak:8080/realms/kavohub
KEYCLOAK_JWKS_URL=http://keycloak:8080/realms/kavohub/protocol/openid-connect/certs
KEYCLOAK_CLIENT_ID=svc-orders-local
KEYCLOAK_CLIENT_SECRET=local-only-not-a-real-secret
HTTP_CLIENT_TIMEOUT_MS=3000
~~~

Усі значення лише локальні; реальний .env не комітити.

### Docker Compose фрагмент

postgres і rabbitmq визначені у спільному Compose з healthchecks і тією самою мережею. На local можна додати localhost mapping; на VPS порт Orders напряму не публікується.

~~~yaml
services:
  orders-migrate:
    image: ${REGISTRY:-ghcr.io/example}/kavohub-orders-migrate:${ORDERS_TAG:-local}
    build:
      context: ./services/orders
      dockerfile: Dockerfile
      target: migrate
    env_file: [.env]
    environment:
      DATABASE_URL: ${DATABASE_URL:?Set DATABASE_URL}
    depends_on:
      postgres:
        condition: service_healthy
    networks: [kavohub-net]
    restart: "no"
    profiles: ["app"]

  orders:
    image: ${REGISTRY:-ghcr.io/example}/kavohub-orders:${ORDERS_TAG:-local}
    build:
      context: ./services/orders
      dockerfile: Dockerfile
      target: runtime
    env_file: [.env]
    environment:
      NODE_ENV: ${NODE_ENV:-production}
      PORT: ${PORT:-8102}
      DATABASE_URL: ${DATABASE_URL:?Set DATABASE_URL}
      RABBITMQ_URL: ${RABBITMQ_URL:?Set RABBITMQ_URL}
      RABBITMQ_EXCHANGE: ${RABBITMQ_EXCHANGE:-kavohub.events}
      RABBITMQ_QUEUE: ${RABBITMQ_QUEUE:-orders.events}
      CRM_BASE_URL: ${CRM_BASE_URL:-http://crm:8101}
      INVENTORY_BASE_URL: ${INVENTORY_BASE_URL:-http://inventory:8103}
      KEYCLOAK_ISSUER: ${KEYCLOAK_ISSUER:?Set KEYCLOAK_ISSUER}
      KEYCLOAK_JWKS_URL: ${KEYCLOAK_JWKS_URL:?Set KEYCLOAK_JWKS_URL}
      KEYCLOAK_CLIENT_ID: ${KEYCLOAK_CLIENT_ID:?Set KEYCLOAK_CLIENT_ID}
      KEYCLOAK_CLIENT_SECRET: ${KEYCLOAK_CLIENT_SECRET:?Set KEYCLOAK_CLIENT_SECRET}
      NBU_BASE_URL: ${NBU_BASE_URL:-https://bank.gov.ua/NBUStatService/v1/statdirectory/exchange}
    depends_on:
      orders-migrate:
        condition: service_completed_successfully
      rabbitmq:
        condition: service_healthy
    expose: ["8102"]
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:8102/ready').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 10s
      timeout: 3s
      start_period: 20s
      retries: 5
    networks: [kavohub-net]
    restart: unless-stopped
    profiles: ["app"]

networks:
  kavohub-net:
    name: kavohub-net
~~~

## 4. CI/CD pipeline

### Схема

~~~mermaid
flowchart LR
  PR["PR<br/>services/orders/** або contracts/**"]
  MAIN["Push у main<br/>ті самі path filters"]
  CHECK["Lint + unit tests"]
  CONTRACT["OpenAPI / AsyncAPI<br/>contract-check"]
  BUILD["Build runtime + migrate<br/>images :SHA"]
  SCAN["Trivy scan обох образів"]
  PUB["GHCR publish<br/>лише main"]
  STAGE["Staging deploy<br/>migration → recreate → ready → smoke"]
  SROLL["Staging failure<br/>auto rollback"]
  OLEG["Олег запускає production deploy<br/>на сервері за SHA"]
  PROD["Ті самі digest-и<br/>без rebuild"]
  PROLL["Production failure<br/>deploy --previous"]
  PR --> CHECK --> CONTRACT --> BUILD --> SCAN
  MAIN --> CHECK
  SCAN --> PUB --> STAGE
  STAGE -->|"failed"| SROLL
  STAGE -->|"passed"| OLEG --> PROD
  PROD -->|"failed"| PROLL
~~~

### Етапи та гейти

| Етап | Тригер | Дія | Гейт |
|---|---|---|---|
| Path filter | PR/push | services/orders/** або contracts/** | Інші зміни не запускають Orders pipeline |
| Lint + unit | PR і main | Стиль, типи, unit tests | Будь-яка помилка блокує |
| Contract-check | PR і main | OpenAPI/AsyncAPI lint і сумісність з ЛР3 | Помилка/несумісність без погодження блокує |
| Build | PR і main | Runtime та migrate images із SHA | Помилка будь-якої збірки блокує |
| Trivy | Після build | Обидва образи; CRITICAL/HIGH, ignore-unfixed | Виправні вразливості блокують publish |
| Publish | Лише main | GHCR images із commit SHA | Після всіх гейтів |
| Staging deploy | Після publish | SSH, migration, Recreate, /ready, smoke | Migration error, readiness не 200 за 60с, smoke failure → rollback |
| Staging monitor | Перші 5 хв | Перевірка 5xx і health | ≥3 unexpected 5xx/5 хв або >5% при ≥20 запитах → rollback |
| Production deploy | Ручно Олегом | Ті самі image digest-и, без rebuild | SHA пройшов staging |
| Production rollback | За потреби | Попередній digest | Не виконувати DB rollback автоматично |

GITHUB_TOKEN короткоживучий: contents:read для перевірок, packages:write лише publish job, packages:read для staging pull. Actions закріплені повними commit SHA. Staging SSH key — repository secret лише staging job. Production SSH key — поза GitHub. Якщо лишити workflow_dispatch, обмежити github.actor, ref main і SHA, що пройшов staging; самих branch rules недостатньо для обмеження запуску.

## 5. Реліз і міграція

### ADR-04-02 — Recreate

**Статус:** прийнято. Recreate через Docker Compose обрано замість Blue-Green: на одному VPS Blue-Green потребує додаткової RAM і складнішої маршрутизації. За цільових 4 замовлень/хв перерва API приблизно 10–15 секунд прийнятна; при зміні NFR переглянути ADR.

RabbitMQ durable queues, persistent messages і volume зберігають чергу. Publisher використовує confirms; consumer робить ACK після COMMIT. Повторна доставка можлива, Inbox не дає повторно застосувати бізнес-зміну. Не видаляти volumes і не запускати down -v.

**Критерії невдачі:** migration exit code не нуль; /ready не 200 за 60с; smoke failure; ≥3 unexpected 5xx за 5 хв або >5% при ≥20 запитах. Staging rollback автоматичний, production rollback запускає Олег. Схему автоматично не відкочувати.

### Expand / contract: перехід до знижки на рядок

Backfill не змінює status, version, reservationId чи 24-годинний дедлайн резерву. Нове поле має зберігати чинну точність.

| Крок | Реліз | Зміна | Безпечний відкат |
|---|---|---|---|
| Expand | v1.1.0 | Nullable order_line.discount_pct; старий код працює з order.discount_pct | Так, стара версія не використовує нову колонку |
| Dual-write/backfill | v1.2.0 | Писати однакове значення в обидва поля, читати старе; пакетно заповнити історичні рядки | На v1.1.0, старе поле — source of truth |
| Switch read | v1.3.0 | Читати рядкове поле, продовжити dual-write; знижки рядків ще однакові | На v1.2.0, поки рівність збережена |
| Line-only compatibility | v1.4.0 | Код не читає/пише стару колонку; колонку лишити до кінця rollback-вікна | На v1.3.0 лише поки старе поле актуальне |
| Різні знижки | v1.5.0 або feature flag | Дозволити відмінності; v1.4.0 стає rollback baseline і має читати line discounts | На v1.4.0; не на v1.3.0 і старіші |
| Contract | v1.6.0 | Після rollback-вікна видалити order.discount_pct | На v1.5/v1.4, що не залежать від старої колонки |

Після різних знижок одне order-level поле не зберігає семантику; старі версії не є допустимими для rollback. orders.order.confirmed не змінюється: AsyncAPI ЛР3 має sku/qty; Billing бере snapshot з Orders API.

## 6. Runbook production deploy / rollback

### Передумови

1. Планувати поза піком 09:00–13:00.
2. Перевірити staging-approved SHA і digest обох образів; не використовувати latest.
3. Перевірити успішний зашифрований PostgreSQL backup не старший за 24 год та alerting.
4. Перевірити критичні завислі операції, DLQ та backlog.
5. Переконатися в сумісності migration з поточним кодом і rollback baseline.
6. Підключитися до VPS із перевіреним SSH host key під deploy user; перевірити GHCR read-only credential.
7. Записати виконавця, час, чинний і цільовий SHA у журнал.

### Deploy

1. Запустити ./ops/deploy.sh production orders <verified-sha>.
2. Скрипт перевіряє SHA/digest, записує чинний образ в orders.previous і завантажує runtime та migration images.
3. Запустити одноразовий orders-migrate: prisma migrate deploy.
4. Якщо migration впала — зупинити deploy до заміни API; не робити автоматичну зворотну migration. Зберегти лог і перевірити Prisma migration state та БД.
5. Після успіху виконати Compose Recreate Orders. Не перезапускати RabbitMQ без потреби й не видаляти volumes.
6. Чекати /ready до 60с; очікуваний код — 200.
7. Запустити ./ops/smoke-test.sh production orders: /health, /ready, безпечний read-only запит і RabbitMQ channel.
8. Спостерігати 5 хв; readiness/smoke failure або поріг 5xx означає rollback.
9. Після успіху оновити orders.current і журнал: час, виконавець, SHA/digest, результати.

### Rollback

1. Запустити ./ops/deploy.sh production orders --previous.
2. Переконатися, що повернувся попередній digest; БД автоматично не відкочувати.
3. Перевірити /ready, smoke і backlog черг.
4. Записати невдалий та відновлений SHA, причину, виконавця і час.
5. Якщо попередній код несумісний зі схемою, припинити повторні спроби й перейти до процедури інциденту; це означає порушення expand/contract gate.

## 7. Навчання з відкату

Сценарій: помилка конфігурації після merge робить staging /ready=503.

| Час від merge | Подія |
|---:|---|
| T+0 | Merge у main запускає workflow |
| T+0–5 хв | Lint, unit tests, contract-check проходять |
| T+5–12 хв | Build двох образів і Trivy scan |
| T+12–14 хв | GHCR publish за SHA |
| T+14–18 хв | Staging migration успішна, Recreate |
| T+18–19 хв | /ready лишається 503; таймаут 60с фіксує збій |
| T+19–21 хв | Auto rollback повертає попередній SHA |
| T+21–23 хв | /ready та smoke старої версії проходять |

Орієнтир: 21 хв від merge до rollback і 23 хв до підтвердженого smoke. Якщо збій видно лише після повного 5-хвилинного моніторингу — близько 25–26 хв. Це оцінки, не виміряний SLA.

## Журнал проєктних рішень

| № | Рішення | Альтернативи | Підстава |
|---|---|---|---|
| ADR-04-01 | Європейський VPS + Compose, окремий staging | Офісний сервер; PaaS/Kubernetes | Доступ звідусіль, офісні ризики; ціна умовна |
| ADR-04-02 | Recreate | Blue-Green / rolling | Один VPS, простота, короткий downtime прийнятний |
| ADR-04-03 | SHA/digest, build once | latest; rebuild у production | Відтворюваність та rollback |
| ADR-04-04 | Окремий Prisma migration target | Migration при старті API | Runtime без CLI, контрольований deploy |
| ADR-04-05 | Production deploy Олегом із сервера | GitHub Environment approval | Free/private обмеження GitHub |
| ADR-04-06 | Відкласти різні discounts до line-only baseline | Одразу після backfill | Старе поле не моделює різні значення |

## Уточнення та припущення

| Уточнення | Джерело | Вплив |
|---|---|---|
| Node 22/NestJS/PostgreSQL 17+/Prisma | ADR-0004, студент | Образ і конфіг |
| RPO/RTO | NFR-ORD-DR-01, студент | Backup/restore |
| До 4 замовлень/хв | Пропозиція студента | Recreate |
| Production deploy за SHA — Олег із сервера | Рішення студента | Production key поза GitHub |
| 180–220 МБ | Оцінка студента | Не підтверджено збіркою |
| VPS/S3 близько $60/міс | Попередня оцінка | Потрібен кошторис |
| Bitwarden shared vault/MFA/резервний доступ | Пропозиція студента | Отримання секретів при інциденті |
| RabbitMQ може повторно доставити повідомлення | ЛР3 | Inbox дедуплікує ефект, не доставку |
| confirmed event має sku/qty, не discount | AsyncAPI ЛР3 | Подія не змінюється |

## Відкриті питання та ризики

- Підтвердити тарифи обох VPS, object storage, IP і податків у межах $100/міс.
- Зафіксувати Node 22 patch/digest і версію Prisma/schema config.
- Відпрацювати restore до RTO ≤4 год із БД, конфігурацією, секретами, чергами/повторною публікацією.
- Якщо лишається workflow_dispatch — обмежити actor, main і staging-approved SHA.
- Визначити зберігання/ротацію GHCR read-only credential на VPS.
- Образ, CI duration та rollback times поки оцінені, не виміряні.
- Додати навчальну групу й посилання/експорт діалогу перед поданням.

## Самоперевірка за критеріями розділу 10

| Критерій | Статус | Орієнтир |
|---|---|---:|
| Образ і health/readiness | Multi-stage, non-root, Prisma/OpenSSL; digest/size не підтверджені | ~1,1/1,5 |
| Середовища, Compose, конфігурація | Три C4, ADR, secrets; кошторис умовний | ~1,1/1,5 |
| CI/CD | Path filters, checks, scans, publish/staging; реалізація й auth потребують уточнення | ~1,5/2 |
| Реліз, rollback, migration | Recreate, gates, expand/contract визначені | ~1,2/1,5 |
| Runbook і навчання | Кроки, перевірки, таймлайн є; часи не виміряні | ~1,1/1,5 |
| **Разом** | **Попередній самоаналіз, не оцінка викладача** | **~6,0/8** |

## Відповіді на контрольні питання

### 1. Build і runtime

Build встановлює dev dependencies, генерує Prisma Client та компілює NestJS у dist. Runtime отримує dist і production node_modules та працює без root. Prisma CLI — у migration target. 180–220 МБ є оцінкою; .dockerignore скорочує build context, не розмір образу напряму.

### 2. Health і readiness

/health перевіряє процес; /ready — БД, RabbitMQ і локальний JWKS cache. Якщо БД включити у liveness, короткий збій може спричинити restart loop. CRM, Inventory і НБУ не блокують весь Orders, бо деякі операції мають деградований режим.

### 3. SHA/digest

SHA/digest прив’язує середовища до одного перевіреного артефакту. latest змінюється; previous зберігає попередній образ для rollback без rebuild. Образ має залишатися доступним у реєстрі чи локальному кеші.

### 4. Окремі migration та rollback floor

Окремий контейнер зупиняє deploy до заміни API, якщо migration впала, і залишає Prisma CLI поза runtime. До різних line discounts відкат можливий, поки старе поле відображає дані; після їх увімкнення baseline — line-aware v1.4. Після видалення старої колонки дозволені лише версії, що її не використовують.

### 5. Pin GitHub Actions

Теги Actions можна перемістити; повний SHA фіксує commit. У березні 2026 Trivy повідомив про перепризначення тегів trivy-action на шкідливі commits для викрадення credentials. Actions піняться за SHA, образи — за digest. [Офіційний advisory Trivy](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)

### 6. Секрети

Runtime secrets — DATABASE_URL, RABBITMQ_URL, KEYCLOAK_CLIENT_SECRET і GHCR pull credential. GITHUB_TOKEN тимчасовий і має мінімальні job-scoped права. Staging SSH key — repository secret тільки staging job; production SSH key — Bitwarden; .env — на хості з обмеженим доступом. Bitwarden має MFA й аварійний доступ.

### 7. Виявлення збою й час rollback

Збій: migration exit code не нуль; /ready не 200 за 60с; smoke failure; ≥3 неочікуваних 5xx за 5 хв чи >5% при ≥20 запитах. Staging сценарій дає 21 хв від merge до rollback і 23 хв до підтвердженого smoke; якщо збій видно лише після моніторингу — 25–26 хв. Це оцінки.

## Робота з ШІ-асистентом

1. Production обрано на VPS через ризики офісного живлення/ISP; ціну лишено умовною.
2. Production deploy перенесено на контроль Олега через GitHub Free/private обмеження.
3. Prisma CLI відокремлено в migration target, щоб prune dev deps не зламав runtime.
4. Expand/contract уточнено: line-only baseline, окреме ввімкнення різних discounts і видалення старої колонки.
5. AsyncAPI ЛР3 перевірено: event lines містять sku/qty, а не discount; Billing читає snapshot з API.

