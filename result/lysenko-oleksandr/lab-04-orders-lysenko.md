# ЛР 4. Контейнеризація, CI/CD та розгортання КПС — модуль Orders

Студент: Олександр Лисенко, група: не зазначена. Дата: 2026-11-21.  
ШІ-асистент: Codex. Діалог: посилання/експорт додається під час здачі.

> Усі конфігурації, команди та діаграми нижче є проєктними чернетками. Вони не збиралися й не запускалися. Адреси з доменом `example.invalid` — заповнювачі.

## 0. Вихідні дані

Модуль **Orders** системи «КавоХаб» володіє замовленнями, рядками, цінами та погодженням знижок. Сервіс слухає порт `8102`, API має базовий шлях `/api/v1`. Технологічний стек: Python 3.12, FastAPI, Alembic, Pydantic, `aio-pika`.

Передумови з ЛР1–ЛР3, надані для цієї роботи:

- Знижка понад 10% потребує погодження `sales_head`; `POST /orders/{id}/confirm` запускає процес погодження. Під час очікування Inventory утримує резерв до 24 годин. Після схвалення підтверджується наявний резерв `R1`, повторний резерв не створюється.
- Небезпечні POST-запити вимагають UUID `Idempotency-Key`, який зберігається у `orders.idempotency_records` з TTL 24 години. Помилки API мають формат RFC 9457 `application/problem+json`.
- НБУ є зовнішньою залежністю для курсу EUR. За його недоступності використовується кеш лише для запитаної дати; без такого кешу EUR-операція повертає `503 exchange-rate-unavailable`, а UAH-замовлення продовжують працювати.
- Orders споживає `orders.events` (quorum), використовує `orders.events.retry` із TTL 5000 мс і `orders.events.dlq` після п’яти спроб. Дедуплікація через `orders.processed_messages` і зміна замовлення виконуються в одній транзакції PostgreSQL; `ack` надсилається лише після `COMMIT`.
- PostgreSQL 17+, окрема схема `orders`, користувач `orders_user`. Серед ключових таблиць: `orders`, `order_lines`, `idempotency_records`, `processed_messages`.
- Офісний сервер має Ubuntu 22.04, Docker і близько 20 ГБ вільної RAM, але залежить від одного провайдера та електроживлення (ДБЖ — близько 20 хв, можливі відключення до 4 годин). Хмарний бюджет — до $100/міс; окремої DevOps-команди немає. Олег Савчук — ІТ-адміністратор на 2 дні на тиждень, без нічних чергувань.

## 1. Середовища й розгортання (С1, С3, С4)

### 1.1. C4 Deployment

Схема відображає межі довіри, вузли розгортання та зв’язки Orders. Для простоти не показано всі модулі «КавоХаб»; CRM та Inventory позначені як зовнішні для команди Orders, але внутрішні системні сервіси. Розміщення Keycloak у середовищах має бути узгоджене системно.

```mermaid
flowchart TB
  subgraph External[Зовнішня мережа та сторонні системи]
    Managers[Менеджери]
    NBU[API НБУ]
    Odoo[Odoo партнера]
    CRM[CRM]
    Inventory[Inventory]
    Keycloak[Keycloak realm kavohub]
  end

  subgraph Local[local — ноутбук розробника]
    Dev[Розробник]
    subgraph LocalTrust[Межа довіри: локальний Docker Compose]
      LP[Reverse proxy / локальний вхід]
      LO[Orders :8102]
      LDB[(PostgreSQL :5432)]
      LMQ[(RabbitMQ :5672)]
      LP --> LO
      LO --> LDB
      LO --> LMQ
    end
    Dev --> LP
  end

  subgraph Staging[staging — офісний сервер]
    Office[Адмін, SSH-ключ]
    subgraph StageTrust[Межа довіри: Docker Compose, приватна мережа]
      SP[Reverse proxy :443]
      SO[Orders :8102]
      SDB[(PostgreSQL :5432)]
      SMQ[(RabbitMQ :5672)]
      SP --> SO
      SO --> SDB
      SO --> SMQ
    end
    Office -->|SSH| StageTrust
  end

  subgraph Production[production — окрема хмарна VM, 4 vCPU / 8 GiB]
    CloudAdmin[Олег, SSH-ключ]
    subgraph ProdTrust[Межа довіри: Docker Compose, приватна мережа]
      PP[Reverse proxy HTTPS :443]
      PO[Orders :8102]
      PDB[(PostgreSQL :5432)]
      PMQ[(RabbitMQ :5672)]
      PP --> PO
      PO --> PDB
      PO --> PMQ
    end
    CloudAdmin -->|SSH| ProdTrust
  end

  Managers -->|HTTPS :443| PP
  Managers -->|HTTPS :443, тестування| SP
  LO -->|HTTPS| NBU
  SO -->|HTTPS| NBU
  PO -->|HTTPS| NBU
  LO -->|HTTPS API key| Odoo
  SO -->|HTTPS API key| Odoo
  PO -->|HTTPS API key| Odoo
  LO -. REST .-> CRM
  LO -. REST .-> Inventory
  SO -. REST .-> CRM
  SO -. REST .-> Inventory
  PO -. REST .-> CRM
  PO -. REST .-> Inventory
  LO -. OIDC/JWT .-> Keycloak
  SO -. OIDC/JWT .-> Keycloak
  PO -. OIDC/JWT .-> Keycloak
```

У production назовні доступний HTTPS `443` через reverse proxy. Orders працює на `8102` у внутрішній мережі. PostgreSQL `5432` і RabbitMQ `5672` не публікуються на хост/інтернет. Staging розміщений в офісі й може бути недоступний під час офісного збою, але не зупиняє production.

### 1.2. ADR-04-01 «Середовище production»

**Статус:** прийнято. **Рішення:** окрема хмарна VM, Docker Compose, 4 vCPU і 8 GiB RAM; оцінка VM — близько $48/міс, резервних копій — близько $10/міс. Сумарний орієнтир $58/міс вкладається у ліміт $100/міс. Це бюджетне припущення, яке треба перевірити за обраним провайдером, регіоном, податками та фактичним кошторисом.

| Альтернатива | Вартість/операційна складність | Переваги | Недоліки та відповідність |
|---|---|---|---|
| Офісний сервер | Низька пряма вартість, знайомий Олегу Docker | Наявні 20 ГБ вільної RAM, локальний контроль | Одне живлення й один провайдер; ДБЖ близько 20 хв при можливих відключеннях до 4 год. Неприйнятний ризик для production продажів |
| **Окрема хмарна VM — обрано** | Орієнтир $48 VM + $10 бекапи на місяць; низька/середня складність Compose | Не залежить від офісного живлення/каналу; просте розгортання кнопкою | Одна VM — SPOF; для HA потрібно більше коштів і операційної роботи |
| Керований Kubernetes | Вища вартість і складність, потрібні навички експлуатації | Оркестрація, масштабування, розширені можливості відновлення | Немає DevOps-команди, бюджет і навантаження не обґрунтовують складність на цьому етапі |

**Критерії:** зменшити ризик простою продажів; не перевищити $100/міс; зберегти просту експлуатацію; мати перевірені резервні копії та відкат. Одна VM є єдиною точкою відмови: її збій зупинить систему до відновлення або перенесення. Обрано цей компроміс, бо друга VM/кластер збільшили б витрати та складність. Рішення переглядається зі зростанням навантаження, скороченням допустимого простою або появою бюджету/команди для HA.

### 1.3. Таблиця середовищ

| Середовище | Адреса/порти | Доступ | Зберігання секретів | Отримання й ротація |
|---|---|---|---|---|
| `local` | `http://localhost:8102`; PostgreSQL `5432`, RabbitMQ `5672` лише локально | Розробник Олександр | Не комітований `.env` з тестовими значеннями | Створюється з `.env.example`; ротація заміною локальних тестових значень |
| `staging` | `https://staging.example.invalid`; Orders `8102` у приватній мережі; SSH до офісної VM; DB/MQ не публічні | Олег через SSH-ключ, deploy job, тестувальники через HTTPS | GitHub Environment secrets; захищений файл конфігурації на VM з обмеженими правами | CI використовує середовищні secrets; Олег має доступ за процедурою. Ротація при зміні доступу/підозрі компрометації та за графіком |
| `production` | `https://orders.example.invalid` через `443`; Orders `8102` внутрішньо; SSH-ключі; DB `5432`, MQ `5672` внутрішні | Менеджери через HTTPS; Олег через SSH; deploy job після approval | GitHub Environment secrets; захищена конфігурація на VM, не в образі/репозиторії | Approval Олега для deploy; секрети змінюються контрольовано, перевіряється підключення, старі credentials відкликаються |

Фактичні DNS-адреси, регіон провайдера, відповідальні за доступ і графік планової ротації — відкрите уточнення до реального розгортання.

## 2. Образ та Healthcheck (М1, М2)

### 2.1. Dockerfile

Чернетка припускає, що `requirements.txt` містить зафіксовані production-залежності. Для максимальної відтворюваності базовий образ у релізі фіксується за digest. За наявності нативних залежностей потрібно додати лише потрібні runtime-бібліотеки.

```dockerfile
# ---- build: інструменти збірки й wheels ----
FROM python:3.12-slim AS build
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_NO_CACHE_DIR=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /build
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip wheel --wheel-dir=/wheels -r requirements.txt

# ---- runtime: без компілятора ----
FROM python:3.12-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PIP_DISABLE_PIP_VERSION_CHECK=1
WORKDIR /app
RUN groupadd --system app && useradd --system --gid app --create-home app
COPY --from=build /wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels
COPY --chown=app:app app/ ./app/
COPY --chown=app:app alembic.ini ./
COPY --chown=app:app migrations/ ./migrations/
USER app
EXPOSE 8102
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8102", "--access-log"]
```

Build і runtime розділені: компілятор не потрапляє у фінальний образ; `requirements.txt` копіюється окремо, щоб кешувати шар залежностей. `USER app` прибирає root-привілеї. `PYTHONUNBUFFERED=1` і access log забезпечують вивід у stdout/stderr для збору журналів. Орієнтир образу — **150–300 МБ**, залежить від пакетів і системних бібліотек; зменшують slim-база, відсутність build tools у runtime та встановлення лише потрібних залежностей.

### 2.2. `.dockerignore`

```dockerignore
.git
.gitignore
.env
.env.*
!.env.example
*.pem
*.key
secrets/
.venv/
venv/
__pycache__/
*.py[cod]
.pytest_cache/
.mypy_cache/
.ruff_cache/
dist/
build/
*.egg-info/
.coverage
htmlcov/
tests/
```

Git-історія, локальні середовища, кеші, тести й артефакти не потрібні в runtime та збільшують build context. `.env`, ключі й `secrets/` виключені, щоб секрети не потрапили у build context або шари образу. Тести запускає CI до публікації релізного образу.

### 2.3. `/health` та `/ready`

| Endpoint | Сенс і перевірки | Код | Тайм-аут/реакція |
|---|---|---|---|
| `/health` | Liveness: процес і event loop відповідають; не перевіряє залежності | `200`; відсутність відповіді означає, що процес не живий | Легка локальна перевірка, орієнтир 1 с; оркестратор може перезапустити контейнер |
| `/ready` | Перевірка з’єднання з PostgreSQL і доступності схеми `orders`, а також доступності RabbitMQ та налаштованої черги `orders.events` | `200` лише коли обидві залежності доступні; інакше `503` | Загалом орієнтир до 2 с; обмежені timeout кожної перевірки |

`/health` не має перевіряти БД: короткий збій залежності не повинен спричинити нескінченні перезапуски живого процесу. `/ready` перевіряє критичні для прийому операцій залежності. Перевірка PostgreSQL підтверджує доступ саме до схеми `orders`, а перевірка RabbitMQ — доступність потрібної черги; health endpoint не створює черги.

НБУ, CRM, Inventory й Odoo не входять у readiness: Orders має деградований режим, кеш курсу на запитану дату, а відмова НБУ блокує лише відповідну EUR-операцію без кешу (`503 exchange-rate-unavailable`), не UAH-замовлення. Недоступність партнерів не повинна робити весь Orders неготовим, якщо запит можна відкласти, повторити або обробити деградовано.

## 3. Конфігурація та Compose (М3, М4, С2)

### 3.1. Змінні середовища

| Назва | Призначення | Local приклад | Джерело staging/production | Секрет |
|---|---|---|---|---|
| `SERVICE_NAME` | Ім’я у логах/метриках | `orders` | Compose | Ні |
| `APP_ENV` | Середовище | `local` | Environment config | Ні |
| `LOG_LEVEL` | Рівень логування | `INFO` | Environment config | Ні |
| `PORT` | HTTP-порт | `8102` | Compose | Ні |
| `DATABASE_URL` | PostgreSQL і схема Orders | `postgresql://orders_user:local-only@postgres:5432/kavohub` | GitHub Environment secret/захищений файл VM | Так |
| `RABBITMQ_URL` | Підключення до брокера | `amqp://orders_user:local-only@rabbitmq:5672/` | GitHub Environment secret/захищений файл VM | Так |
| `ORDERS_QUEUE` | Основна черга | `orders.events` | Compose | Ні |
| `ORDERS_RETRY_QUEUE` | Retry queue | `orders.events.retry` | Compose | Ні |
| `ORDERS_DLQ` | Dead-letter queue | `orders.events.dlq` | Compose | Ні |
| `KEYCLOAK_ISSUER` | JWT issuer realm `kavohub` | `http://keycloak:8080/realms/kavohub` | Environment config | Ні |
| `NBU_BASE_URL` | API курсу НБУ | `https://bank.gov.ua/NBUStatService/v1` | Environment config | Ні |
| `ODOO_URL` | Odoo партнера | `https://odoo.example.invalid` | Environment config | Ні |
| `ODOO_API_KEY` | Доступ до Odoo | `local-placeholder` | Secret середовища, доступний Orders | Так |

### 3.2. Чернетка `.env.example`

```dotenv
# Лише безпечні локальні приклади; не використовувати у staging/production.
SERVICE_NAME=orders
APP_ENV=local
LOG_LEVEL=INFO
PORT=8102
DATABASE_URL=postgresql://orders_user:local-only@postgres:5432/kavohub
RABBITMQ_URL=amqp://orders_user:local-only@rabbitmq:5672/
ORDERS_QUEUE=orders.events
ORDERS_RETRY_QUEUE=orders.events.retry
ORDERS_DLQ=orders.events.dlq
KEYCLOAK_ISSUER=http://keycloak:8080/realms/kavohub
NBU_BASE_URL=https://bank.gov.ua/NBUStatService/v1
ODOO_URL=https://odoo.example.invalid
ODOO_API_KEY=local-placeholder
```

Реальний `.env` виключений із Git. Секрети не вбудовуються в образ, Dockerfile, аргументи збірки або `.env.example`.

### 3.3. Фрагмент `docker-compose.yml`

Припускається, що спільні сервіси `postgres` і `rabbitmq` оголошені в системному Compose, під’єднані до `kavohub-network` і мають healthcheck. Образ для застосунку й міграцій один, його релізний тег — SHA.

```yaml
services:
  orders-migrate:
    image: ${ORDERS_IMAGE:-kavohub-orders:local}
    command: ["alembic", "upgrade", "head"]
    env_file: [.env]
    environment:
      DATABASE_URL: ${ORDERS_DATABASE_URL:?set ORDERS_DATABASE_URL}
    depends_on:
      postgres:
        condition: service_healthy
    networks: [kavohub-network]
    restart: "no"
    profiles: [app]

  orders:
    image: ${ORDERS_IMAGE:-kavohub-orders:local}
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8102", "--access-log"]
    env_file: [.env]
    environment:
      SERVICE_NAME: orders
      APP_ENV: ${APP_ENV:-local}
      LOG_LEVEL: ${LOG_LEVEL:-INFO}
      PORT: "8102"
      DATABASE_URL: ${ORDERS_DATABASE_URL:?set ORDERS_DATABASE_URL}
      RABBITMQ_URL: ${ORDERS_RABBITMQ_URL:?set ORDERS_RABBITMQ_URL}
      ORDERS_QUEUE: orders.events
      ORDERS_RETRY_QUEUE: orders.events.retry
      ORDERS_DLQ: orders.events.dlq
      KEYCLOAK_ISSUER: ${KEYCLOAK_ISSUER:?set KEYCLOAK_ISSUER}
      NBU_BASE_URL: ${NBU_BASE_URL}
      ODOO_URL: ${ODOO_URL}
      ODOO_API_KEY: ${ODOO_API_KEY}
    expose: ["8102"]
    # Local override може додати "8102:8102"; staging/prod порт хоста не публікують.
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      orders-migrate:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8102/ready', timeout=2)"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 768M
    restart: unless-stopped
    networks: [kavohub-network]
    profiles: [app]

networks:
  kavohub-network:
    external: true
```

Сервісне DNS-ім’я Orders — `orders`, контейнерний порт — `8102`. Модулі звертаються один до одного через сервісні імена в спільній мережі. Порти PostgreSQL `5432` і RabbitMQ `5672` лишаються внутрішніми; назовні виставляється reverse proxy на `443`. Користувач БД Orders має доступ лише до схеми `orders`. Для local потрібен override з `ports: ["8102:8102"]`; production не публікує цей порт на хості. У staging/prod секрети мають бути доступні міграційному контейнеру й Orders з одного захищеного джерела.

## 4. Конвеєр CI/CD (М5)

### 4.1. Схема

```mermaid
flowchart LR
  PR["PR: services/orders/** або contracts/**"] --> Q["make lint · make test · make contract-check"]
  Q --> PRBuild["Build перевірки + Trivy"]
  PRBuild --> Merge["Гейти зелені, merge у main"]
  Merge --> Build["Build один раз, тег повного SHA"]
  Build --> Scan["Trivy: CRITICAL gate"]
  Scan --> GHCR["Publish GHCR"]
  GHCR --> Stage["Deploy staging, міграції"]
  Stage --> Smoke["/health · /ready · smoke"]
  Smoke -->|успіх| Approval["Олег: ручний approval"]
  Smoke -->|помилка| RollbackS["Rollback staging до previous SHA"]
  Approval --> Prod["Deploy production тим самим образом"]
  Prod --> Verify["Health + smoke"]
  Verify -->|помилка| RollbackP["Rollback production до previous SHA"]
```

### 4.2. Етапи, гейти й артефакти

| Етап | Коли | Інструменти/дія | Гейт | Артефакт |
|---|---|---|---|---|
| Lint | PR і `main`, тільки `services/orders/**` або `contracts/**` | `make lint` (наприклад Ruff) | Будь-яка помилка | Звіт lint |
| Unit tests | PR і `main` | `make test` | Невдалий тест | Звіт тестів |
| Contract | PR і `main` | `make contract-check`; Spectral для OpenAPI lint; Prism за потреби для контрактної перевірки | Невалідний/несумісний контракт | Звіт контракту |
| Build + scan | PR — без push; main — образ релізу | Docker build, Trivy | Збій збірки; CRITICAL vulnerability за прийнятою політикою | Для main образ `ghcr.io/<org>/kavohub-orders:<повний-SHA>` і digest |
| Publish | Лише main після гейтів | GHCR push | Попередні гейти пройдено | Образ у GHCR |
| Deploy staging | Після publish | SSH deploy, міграції, очікування readiness | Помилка міграцій/старту | Запис релізу |
| Smoke | Після staging deploy | `/health`, `/ready`, ключові API-перевірки | Будь-яка помилка; тоді зупинити просування й відкотити staging | Smoke report |
| Production approval/deploy | Після зеленого staging | GitHub Environment із ручним reviewer Олегом; той самий SHA | Немає погодження або помилка production smoke | Журнал production релізу |

### 4.3. Права й секрети

Базове право `GITHUB_TOKEN` — `contents: read`; лише publish job додає `packages: write`. Deploy secrets доступні лише середовищному job; production має окремий GitHub Environment з обов’язковим approval Олега. PR з форків секретів не отримує. Сторонні Actions закріплюються за повним SHA. SSH-ключі зберігаються у GitHub Environment secrets та на відповідному сервері з обмеженими правами; паролі/ключі застосунку не вбудовуються в образ. Тег `latest` заборонений для розгортання: використовується незмінний SHA образу й фіксується digest.

## 5. Реліз, відкат і міграція схеми (М6, М7, М9)

### 5.1. Стратегія релізу та черги

**ADR-рішення:** Recreate із graceful shutdown на хмарній VM 8 GiB. Blue-Green потребує двох одночасних екземплярів і перемикання трафіку, що збільшує пікове споживання пам’яті поряд із PostgreSQL, RabbitMQ та іншими сервісами. Recreate простіший для команди без чергового DevOps, але створює коротке вікно недоступності Orders. Перед заміною фіксується `previous` SHA, споживач перестає брати нові повідомлення, поточна обробка завершується в межах timeout, далі старий контейнер замінюється новим. Міграції мають бути сумісними з попереднім кодом протягом вікна відкату.

Під час зупинки контейнера повідомлення `orders.events`, на які ще не було `ack`, залишаються unacked і можуть бути повторно доставлені брокером. Orders виконує `ack` лише після `COMMIT`; `processed_messages` забезпечує дедуплікацію. Quorum queue реплікує дані між вузлами кластера, але сама по собі не є абсолютною гарантією нульової втрати: важливі персистентність повідомлень, publisher confirms, кількість доступних реплік і відсутність спільної відмови. Якщо RabbitMQ стоїть на тій самій одиночній VM, VM лишається SPOF. Тому нульову втрату не заявляємо без конфігураційного підтвердження.

**Критерії відкату:** `/ready` не дає `200` протягом 60 с; smoke не проходить; контейнер перезапускається або ключовий запит стабільно помилковий. Конвеєр зупиняє просування й відновлює попередній SHA. Міграцію не відкочують руйнівною down-операцією без аналізу сумісності; для expand/contract безпечний відкат коду залежить від фази.

### 5.2. Expand / Contract для `discount_pct`

До backfill бізнес має погодити алгоритм розподілу загальної знижки між рядками, правила округлення й інваріант збереження загальної суми: просте копіювання відсотка на кожен рядок може змінити фінансовий результат.

| Фаза | Схема/код | `pending_approval` і резерв 24 год | Безпечність відкату |
|---|---|---|---|
| **1. Expand** | Додати `order_lines.discount_pct` nullable; стару `orders.discount_pct` не видаляти. Стара версія продовжує працювати | Статус і таймер не змінюються; резерв Inventory не скасовується, керівник погоджує за старою моделлю | Безпечний: стара версія не залежить від нової колонки |
| **2. Dual Write / Dual Read — v1.1.0** | Записувати сумісні значення в обидва представлення; читати нові рядки з fallback на старе поле. Запис атомарний у межах транзакції | Замовлення лишається pending, резерв `R1` утримується до 24 год; після схвалення підтверджується саме `R1`, без дублювання. Екран/подія погодження мають відтворювати узгоджену знижку | Безпечний за умови перевіреного dual write; v1.0.0 читає старе поле |
| **3. Data Backfill** | Ідемпотентний пакетний скрипт заповнює рядки за погодженим алгоритмом; звірити суми, пропуски й помилки | Не змінювати статус, reserve ID чи таймер. In-flight замовлення мають коректно показувати знижку для рішення sales_head | Код можна повернути до v1.0.0, доки старе поле актуальне; backfill можна повторити |
| **4. Contract — v2.0.0** | Переконатися у повноті даних і сумісності споживачів; код більше не читає/пише старе поле; окремою міграцією видалити `orders.discount_pct` | Погодження читає рядкові знижки; резерв не скасовується, після схвалення підтверджується `R1`. Не контрактити, доки in-flight workflow не може вимагати стару форму | Відкат на v1.1.0 після видалення колонки небезпечний; потрібен forward-fix або відновлення старого поля з рядків |

**Споживачі подій:** до Фази 4 CRM, Inventory, Billing та інші споживачі погоджують версію схеми. Якщо подія містить загальну знижку, v1.1.0 тимчасово зберігає сумісний формат або застосовує версіоновану подію/dual publish. `order_id`, дедуплікація та семантика погодження лишаються сталими. Contract дозволений після підтвердження споживачів, відсутності старих екземплярів і завершення вікна відкату.

### 5.3. Навчання з відкату на папері

Сценарій: помилкова конфігурація пулу PostgreSQL у новій версії спричиняє вичерпання з’єднань; `/health` відповідає, `/ready` повертає `503`. Оцінки нижче — проєктні, а не виміряні.

| Хвилина від merge | Подія |
|---|---|
| 00 | PR злитий у `main`; стартують гейти lint, tests, contract |
| 01–03 | Гейти проходять, збирається образ із повним SHA |
| 04–05 | Trivy не знаходить блокуючих CRITICAL; образ публікується в GHCR |
| 06 | Deploy staging зберігає попередній SHA, виконує міграцію та запускає контейнер |
| 07 | `/health` дає 200, `/ready` дає 503 через пул PostgreSQL; smoke не проходить |
| 08 | Просування зупинене; production approval/deploy недоступний; стартує staging rollback |
| 09–10 | Попередній SHA запущений, `/ready` і smoke проходять; журнал фіксує невдалий та відновлений SHA |

Орієнтир від злиття до відновлення staging — **близько 10 хвилин**. Production не зачеплений. Фактичний MTTR треба уточнити контрольованим навчальним прогоном.

## 6. Runbook розгортання Orders для Олега (М8)

Команди — шаблон для Compose-проєкту `/opt/kavohub`; фактичні назви файлів, URL і спосіб подачі секретів треба звірити на цільовій VM. Значення токена/секрету в командному рядку не вводити.

### 6.1. Передумови та збереження попередньої версії

- Переконатися, що CI зелений, образ має точний SHA, staging smoke пройшов і є production approval.
- Перевірити диск, контейнери, логи Orders, PostgreSQL/RabbitMQ health, `/health`, `/ready`.
- Перевірити свіжість і доступність резервної копії БД; для міграції — її сумісність із попереднім кодом.
- Зберегти `PREVIOUS_SHA` до будь-якої зміни.

```bash
cd /opt/kavohub
docker compose ps
docker compose logs --since=10m orders
curl --fail https://orders.example.invalid/health
curl --fail https://orders.example.invalid/ready
PREVIOUS_SHA="$(cat releases/orders.current)"
printf '%s\n' "$PREVIOUS_SHA" > releases/orders.previous
```

### 6.2. Deploy

```bash
cd /opt/kavohub
RELEASE_SHA="<повний SHA коміту>"
export ORDERS_IMAGE="ghcr.io/<org>/kavohub-orders:${RELEASE_SHA}"
docker compose pull orders-migrate orders
docker compose run --rm orders-migrate
docker compose up -d orders
```

Міграція повинна завершитися з кодом `0`, інакше зупинитися й не оновлювати Orders. Після запуску:

```bash
docker compose ps orders
curl --fail https://orders.example.invalid/health
curl --fail https://orders.example.invalid/ready
curl --fail -H "Authorization: Bearer <тестовий-токен>" \
  https://orders.example.invalid/api/v1/orders/<контрольний-id>
printf '%s\n' "$RELEASE_SHA" > releases/orders.current
```

Smoke API-запит має бути безпечним для тестових даних; не створювати реальне замовлення без спеціального тестового сценарію та idempotency key.

### 6.3. Критерії і процедура відкату

Відкотити, якщо міграція помилкова, `/ready` не став `200` за 60 с, smoke не проходить, контейнер перезапускається або ключова операція систематично помиляється. Ціль повернення контейнера — близько хвилини від рішення; повний MTTR може бути довшим. Перевірити сумісність старого коду зі схемою; не запускати руйнівний downgrade автоматично.

```bash
cd /opt/kavohub
PREVIOUS_SHA="$(cat releases/orders.previous)"
export ORDERS_IMAGE="ghcr.io/<org>/kavohub-orders:${PREVIOUS_SHA}"
docker compose pull orders
docker compose up -d --no-deps orders
curl --fail https://orders.example.invalid/health
curl --fail https://orders.example.invalid/ready
```

Після перевірок зафіксувати фактичний результат/час. Якщо попередня версія несумісна зі зміненою схемою, не продовжувати сліпий відкат — залучити відповідального за БД і застосувати погоджений план відновлення.

### 6.4. Журнал релізів

| Час UTC | Середовище | Модуль | Новий SHA | Попередній SHA | Хто погодив/виконав | Міграція | Health/smoke | Результат/причина відкату | Тривалість |
|---|---|---|---|---|---|---|---|---|---|
| `<дата-час>` | `<stage/prod>` | Orders | `<SHA>` | `<SHA>` | `<ім’я>` | `<успіх/не запускалась/помилка>` | `<результат>` | `<опис>` | `<хв>` |

## 7. Журнал проєктних рішень

| № | Рішення | Альтернативи | Чому обрано | Підстава |
|---|---|---|---|---|
| ADR-04-01 | Production на окремій хмарній VM, Compose | Офісний сервер; керований Kubernetes | Офісна інфраструктура має ризик тривалого збою; VM вкладається у бюджет і не вимагає кластерної команди | Бюджет до $100/міс, простої продажів дорогі, DevOps-команди немає |
| ADR-04-02 | Recreate з graceful shutdown | Blue-Green | Менше пікових ресурсів та експлуатаційної складності на 8 GiB VM | Один невеликий сервіс, ручний approval Олега, попередній SHA збережено |
| ADR-04-03 | Readiness перевіряє PostgreSQL і RabbitMQ, не НБУ/партнерів | Перевіряти всі залежності | Деградований режим зберігає UAH і роботу при недоступності зовнішніх сервісів | ADR-ORD-03, локальні retry/cache та розділення критичних залежностей |
| ADR-04-04 | Expand/contract для знижки | Одноразове перейменування колонки | Дозволяє сумісні релізи й контрольований відкат | In-flight `pending_approval`, 24-годинний резерв і сумісність подій |

## 8. Уточнення стейкхолдерів і припущення

| № | Уточнення / припущення | Джерело | На що впливає |
|---|---|---|---|
| 1 | Production VM близько $48/міс, бекапи $10/міс | Рішення студента за оцінкою вартості | ADR/бюджет; перевірити перед реальним закупленням |
| 2 | Олег погоджує production кнопкою й не має нічних чергувань | Контекст ІТ-адміністратора | Manual approval, автоматичні health gates/rollback |
| 3 | Orders має працювати деградовано без НБУ й партнерів | ЛР3 / ADR-ORD-03 | Readiness і зовнішні інтеграції |
| 4 | Точне правило розподілу загальної знижки між рядками ще не задане | Виявлене відкрите доменне рішення | Backfill та збереження суми; блокує Фазу 3 до погодження |
| 5 | Quorum-черга захищає від втрати лише за відповідної кластерної/персистентної конфігурації | Архітектурне уточнення | Не можна заявляти абсолютну гарантію без перевірки RabbitMQ |
| 6 | Адреси `example.invalid`, назви секретного сховища і точні порти інших модулів — заповнювачі | Проєктне припущення | Потрібні фактичні значення в реалізаційному проєкті |

## 9. Відкриті питання та ризики

1. Який провайдер, регіон, фактична ціна VM/резервних копій і цільовий RPO/RTO?
2. Чи розміщуються production PostgreSQL і RabbitMQ на тій самій VM; як буде забезпечено відновлення при втраті VM?
3. Яка фактична політика RabbitMQ: кількість quorum-реплік, durable/persistent публікації, publisher confirms, резервування даних?
4. Яке бізнес-правило розподіляє історичну знижку на рядки без зміни загальної суми й округлення?
5. Яка версія подій/схема потрібна CRM, Inventory, Billing; чи погоджують вони період сумісності?
6. Які реальні DNS-адреси, середовищні reviewers, строк зберігання бекапів і графік ротації секретів?
7. Точні команди smoke API, механізм безпечного тестового замовлення та критерії rate/error для post-deploy ще треба узгодити.

## 10. Самоперевірка за критеріями (максимум 8 балів)

Самооцінка за проєктними артефактами; критерій формально не підтверджує реалізацію чи фактичні випробування, оскільки робота цього не вимагає.

| Критерій | Статус | Бали | Посилання на розділи | Обґрунтування |
|---|---|---:|---|---|
| Проєкт образу й health/readiness | Виконано повністю на рівні проєкту | 1,5 / 1,5 | 2 | Багатоетапний Python образ, non-root, кеш залежностей, оцінка розміру, `.dockerignore`; чітко відокремлено liveness/readiness та пояснено зовнішні залежності |
| Середовища, Compose і конфігурація | Виконано повністю на рівні проєкту | 1,5 / 1,5 | 1, 3 | Три середовища й trust boundaries, ADR альтернатив, Compose з міграціями/health/dependencies, 12-factor і розділення секретів |
| Конвеєр CI/CD | Виконано повністю на рівні проєкту | 2 / 2 | 4 | Path triggers, lint/test/contract, Trivy, SHA-тег, GHCR, staging, approval, least privilege і секрети описані |
| Реліз, відкат і міграція схеми | Виконано повністю на рівні проєкту | 1,5 / 1,5 | 5.1–5.2 | Recreate обґрунтовано ресурсами; критерії відкату, черги, 4 фази expand/contract та in-flight резерви розглянуті. Backfill лишається заблокованим до правила розподілу знижки |
| Runbook і навчання з відкату | Виконано повністю на рівні проєкту | 1,5 / 1,5 | 5.3, 6 | Є покроковий deploy/rollback, перевірки, журнал і хронологія з оцінкою часу. Час оцінний, запуск не проводився |
| **Разом** |  | **8 / 8** |  |  |

## 11. Відповіді на контрольні питання

### 1. Чим відрізняються `build` і `runtime`; який розмір очікується?

`build` містить інструменти для створення Python wheels і може містити компілятор. `runtime` має лише Python slim, встановлені залежності й застосунок; запускається як користувач `app`. Орієнтовний розмір — 150–300 МБ, залежно від залежностей. Образ зменшують slim-база, відсутність компілятора у фінальному шарі, кешований окремий шар залежностей і виключення тестів/кешів/секретів із контексту.

### 2. Чому `/health` не перевіряє БД, а `/ready` перевіряє? Які залежності виключені?

`/health` відповідає на питання, чи живий процес; якщо включити туди БД, короткий збій БД може викликати перезапуски живого сервісу. `/ready` визначає, чи може Orders обслуговувати операції, тому перевіряє PostgreSQL у схемі `orders` та потрібну чергу RabbitMQ. НБУ й партнери виключені: курс можна взяти з кешу на запитану дату, UAH-процес працює під час збою НБУ, а зовнішня інтеграція не має блокувати весь Orders. Без курсу на дату конкретний EUR-запит отримує контрольований `503`.

### 4. Чому міграція окремим кроком і де відкат стає небезпечним?

Окремий `orders-migrate` робить зміну схеми видимим контрольованим кроком і не запускає міграцію конкурентно в кожній репліці застосунку. Expand додає nullable колонку, стара версія продовжує працювати; далі v1.1.0 пише в обидва поля, backfill переносить дані, а v2.0.0 завершує перехід. Відкат стає небезпечним після Contract, коли стара колонка видалена: старий код її очікує. Також до backfill потрібне узгоджене правило розподілу знижки, яке зберігає фінансовий підсумок.

### 7. Як конвеєр визначає невдалий реліз і скільки триває відкат?

Невдачу визначають readiness, smoke-запити та стабільність контейнера. Якщо `/ready` не став `200` за 60 секунд або smoke не пройшов, просування зупиняється й staging автоматично повертається до попереднього SHA. Проєктна оцінка від merge до staging-відновлення у сценарії — близько 10 хвилин; production додатково вимагає approval Олега. Фактичний MTTR невідомий до навчального прогону.

### 10. Що стається з повідомленнями під час заміни версії?

Graceful shutdown припиняє брати нові повідомлення і дає завершитися поточним. Якщо процес завершується до `ack`, брокер може повернути unacked повідомлення в чергу для повторної доставки. Оскільки Orders підтверджує повідомлення після транзакційного `COMMIT` і дедуплікує їх у `processed_messages`, повторна доставка не повинна повторно застосувати зміну. Quorum черга реплікує дані в кластері, але абсолютну гарантію нульової втрати можна заявляти лише після перевірки конфігурації реплік, persistence і publisher confirms.

## 12. Робота з ШІ-асистентом

1. Початкове припущення про нульову втрату повідомлень у quorum queue було уточнене: quorum дає реплікацію, але гарантії залежать від конфігурації брокера, persistence і publisher confirms.
2. Для 8 GiB VM обрано Recreate замість Blue-Green, щоб не вимагати одночасних копій Orders і додаткової пам’яті; компроміс — коротке вікно недоступності.
3. Для readiness виключено НБУ та партнерів, бо деградований режим має зберігати роботу не залежних від них операцій.
4. Під час плану backfill виявлено, що копіювання загальної знижки в кожен рядок може змінити суму. Додано вимогу погодити алгоритм розподілу до виконання міграції.
5. Оцінки часу відкату й ціни позначені як проєктні припущення, а адреси й деталі секретного сховища — як значення для уточнення, а не перевірені факти.

