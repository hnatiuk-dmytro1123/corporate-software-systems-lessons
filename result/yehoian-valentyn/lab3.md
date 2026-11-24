# Сформуй завдання для Lab 3

> переходимо до lab3 проаналізуй результати описані в другій лабораторній та сформуй завдання, для мого випадку

<details><summary>4 previous messages</summary>

> Знайду матеріали другої лабораторної у робочій теці, розберу ваші результати й на їх основі сформую персоналізоване завдання для лабораторної 3.
>
> <details><summary>Ran 4 commands</summary>
>
> - Ran `rg --files`
> - Ran `Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab2_gpt_chat.md; Get-Content corporate-software-systems-lessons\lab-03.md; Get-Content corporate-software-systems-lessons\README.md`
> - Ran `Get-Content -Raw corporate-software-systems-lessons\lab-03.md; Get-Content -Raw corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md; Get-Content -Raw corporate-software-systems-lessons\result\yehoian-valentyn\ADR-0004-buy-vs-build-notifications-reporting.md`
> - Ran `rg -n '^## 6|^### 6\.|^## 7|^## 8|^## 9|^## 10|^### 10\.|## 0|Відкриті питання|Уточнення|TODO|частково' corporate-software-systems-lessons\lab-03.md corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md`
>
> </details>
>
> У ЛР2 уже є повний проєктний документ і окремий ADR. Вони визначають межу модуля доволі чітко: сервіс приймає події й надсилає листи/звіти, а Odoo Community лишається бухгалтерським джерелом лише для читання. Зараз звірю конкретні вимоги ЛР3 з цими рішеннями, включно з позначеними в самоперевірці прогалинами.
>
> <details><summary>Ran a command</summary>
>
> - Ran `Get-Content corporate-software-systems-lessons\lab-03.md | Select-Object -Skip 225 -First 125; Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 765 -First 50; Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 571 -First 45`
>
> </details>
</details>

## Аналіз ЛР2

У вашому звіті ЛР2 модуль **Notifications / Reporting** — окремий сервіс: він споживає події, веде журнал і надсилає повідомлення та звіти. **Odoo Community** залишається джерелом бухгалтерських даних і PDF рахунків; сервіс звертається до нього лише для читання. В ADR-0004 обрано саме таку архітектуру.

Для ЛР3 є хороша основа: у ЛР2 описані API, модель сповіщень і повторів, ролі, інтеграція з Odoo, а також рішення надсилати листи через сервіс. Водночас самоперевірка ЛР2 позначає кілька пунктів як частково виконані. Їх варто закрити під час проєктування контрактів і стійкості.

## Персональне завдання для ЛР3

1. **Підготувати OpenAPI 3.1 для Notifications / Reporting.** Включити ендпоінти з ЛР2: перегляд журналу й деталей сповіщень, звітів і шаблонів, а також admin-only `resend`. Зафіксувати схеми відповідей, ролі, помилки RFC 9457, `X-Correlation-Id` і правила використання `Idempotency-Key` для команд, де повтор може спричинити побічний ефект. Додати 3–5 правил стилю API й описати, як команди перевірятимуть контракт через mock-сервер.

2. **Описати AsyncAPI 3.0 і підписки.** У базовій таблиці ЛР3 Notifications споживає `crm.deal.won`, `orders.order.confirmed`, `inventory.shipment.dispatched`, `billing.invoice.issued` та алерти DLQ. Уточнити, чи публікує ваш модуль власні події; якщо ні — явно зазначити це. Для кожної підписки вказати обробник, зміну стану або даних, дедуплікацію, поведінку за події не за порядком і необхідні поля конверта, зокрема `correlationId`.

3. **Описати потік повідомлень К1 і К2.** Для листа з ТТН після відвантаження та листа з PDF рахунку показати джерело події, пошук даних, побудову листа, надсилання й запис результату. Пов’язати це з рішенням ADR-0004: лист надсилає ваш сервіс, а Odoo дає бухгалтерські дані/PDF. Уточнити відкритий пункт Q-001 — стабільний ключ, який пов’язує shipment, order і рахунок.

4. **Спроєктувати обробку подій і RabbitMQ.** Визначити основну чергу підписок, retry-чергу, DLQ, до п’яти спроб і процедуру ручного розбору DLQ. На прикладі `inventory.shipment.dispatched` описати ідемпотентного споживача: записати ID події та бізнес-зміну в одній транзакції, а ACK робити після commit. Окремо пояснити, як не надіслати клієнту два листи через дубль події.

5. **Узгодити синхронні виклики та пошту.** За ЛР3 для Notifications потрібен ADR щодо недоступності пошти. Задати тайм-аути, повтори й `X-Correlation-Id` для поштового провайдера; визначити поведінку, якщо він недоступний 10 хвилин. Розмежувати повтори інтеграційних повідомлень у RabbitMQ та три спроби доставки листа з ЛР2 — це різні механізми й ліміти.

6. **Підготувати сценарії стійкості у Given / When / Then.** Мінімум: повтор тієї самої події не створює другого листа; «отруйна» подія після п’яти спроб потрапляє в DLQ; повтор HTTP-запиту з тим самим `Idempotency-Key` не дублює `resend`. Додати сценарій тимчасової відмови пошти й випадок, коли подія для рахунку надійшла раніше за доступність PDF.

7. **Пройти уявний інтеграційний прогін Lead-to-Cash.** Для вашого модуля перевірити події від угоди до рахунку та платежу, дубль `inventory.stock.reserved` і недоступність зовнішньої системи. Підготувати діаграму послідовності й таблицю розбіжностей контрактів: поле/тип/ідентифікатор, відповідальна сторона, запропонована зміна та реакція команди.

8. **Оформити документ і зафіксувати незакриті питання.** Здається `lab-03-notifications-reporting-yehoian.md`. Перенесіть потрібні рішення з ЛР2 і ADR-0004, позначте зміни, додайте журнали рішень, припущення, відкриті питання, самоперевірку й відповіді на щонайменше п’ять контрольних питань.

Найважливіші пункти ЛР2, які переходять у ЛР3: звірити ключі кореляції для рахунку та відвантаження (Q-001), визначити поведінку для події без ТТН (Q-006), узгодити поштового провайдера й політику зберігання (Q-007), а також уточнити `role scopes`, ACK для дубльованої події та назви помилок RFC 9457. Неузгоджене можна лишити відкритим, але зафіксувати наслідки.

Методичка ЛР3 просить спершу запропонувати власний варіант контракту, а потім отримати рецензію. Тож почнімо з першого артефакту: накидайте тезово 3–5 ендпоінтів вашого модуля для OpenAPI та вкажіть, які з них потребують захисту від повторного виконання.

> Мій варіант ендпоінтів Notifications / Reporting для OpenAPI: GET /health; GET /api/v1/notifications (журнал, пагінація page/pageSize); GET /api/v1/notifications/{id} (запис і спроби доставки); POST /api/v1/notifications/{id}/resend (ручний повтор, лише admin, лише для final_failed); GET /api/v1/reports/daily?date=; GET /api/v1/reports/inactive-renters?weekOf=; GET /api/v1/templates (лише admin). Захист від повторного виконання потрібен лише для POST resend: заголовок Idempotency-Key обов'язковий; повтор із тим самим ключем і тим самим тілом повертає ту саму відповідь 202 без нового циклу доставки; повтор з тим самим ключем і іншим тілом — 422; новий ключ, коли повтор уже виконується, — 409. GET безпечні й ідемпотентні. Рецензуй: що пропущено, що суперечить, чи правильні коди.

Загалом варіант узгоджується з API з ЛР2, а правила для `Idempotency-Key` відповідають методичці ЛР3. Ендпоінти покривають журнал, повторну доставку, звіти й перегляд шаблонів.

### Що варто уточнити

- **Фільтри журналу.** У ЛР2 для `GET /api/v1/notifications` були ще `dateFrom`, `dateTo`, `type` і `state`. Якщо залишаєте лише пагінацію, це зміна відносно ЛР2 — зафіксуйте її в журналі рішень.
- **Область доступу.** Для журналу й деталей потрібні не лише ролі, а й обмеження за компанією, складом або командою згідно з ЛР2. Також зафіксуйте поведінку для чужого запису: у звіті ЛР2 для нього вказано `403`, для неіснуючого ID — `404`.
- **`GET /health`.** Уточніть, що це liveness без автентифікації. Якщо `503` означає, що сервіс не готовий приймати трафік, опишіть readiness окремо або чітко визначте значення відповіді.
- **Повторне використання ключа.** Для першого запиту збережіть ключ, хеш тіла й відповідь на 24 години, як у методичці. Тоді повтор з тим самим ключем і тілом повертає ту саму відповідь `202` і той самий `deliveryCycle`, не створюючи нового циклу.
- **`409` має два різні випадки.** Повтор із новим ключем, коли доставка вже виконується, — `409 request-in-progress`. Повтор для сповіщення, яке вже не має стану `final_failed`, — теж `409`, але з окремим RFC 9457 `type`, наприклад `notification-retry-not-allowed`.
- **Валідація запиту.** Порожня чи відсутня `reason` для `resend` має давати `422`, як передбачено в ЛР2. Вкажіть також формат і обов’язковість `Idempotency-Key` — методичка пропонує UUID.

### Коди відповідей

- `202` — повтор прийнято в чергу; воркер надсилає лист асинхронно.
- `409` — конфлікт стану або запит із таким ключем ще виконується; потрібен відповідний `type`.
- `422` — ключ повторно використано з іншим тілом або запит не пройшов валідацію.
- `401` / `403` — користувач не автентифікований / не має потрібної ролі чи області доступу.
- `404` — сповіщення або знімок звіту не знайдено.
- `503` — сервіс не готовий або тимчасово недоступний.

`GET`-запити справді безпечні й ідемпотентні. Для всіх операцій варто описати `X-Correlation-Id` і повертати помилки у форматі `application/problem+json`. Ще залишене з ЛР2 питання Q-002: чи створювати нове ІТ-сповіщення, якщо повторний admin-цикл знову завершується `final_failed`. Це не блокує OpenAPI, але позначте наслідок як відкрите питання.

> Мої правила стилю API: 1) шляхи в множині, префікс /api/v1, kebab-case; 2) JSON camelCase, час у ISO 8601 з часовим поясом; 3) усі помилки лише application/problem+json (RFC 9457) зі стабільними type у форматі urn:kavohub:problem:...; 4) пагінація page/pageSize, максимум pageSize 100; 5) кожен запит і відповідь має X-Correlation-Id (якщо клієнт не передав, сервіс генерує й повертає). Перевірка контракту: з openapi.yaml запускається mock-сервер (наприклад Prism), а контрактні тести перевіряють відповіді mock і справжнього сервісу за тими самими схемами. Рецензуй правила й опиши процес перевірки команд через mock.

Правила загалом послідовні й придатні для OpenAPI. Є кілька деталей, які варто додати, щоб їх можна було однозначно перевірити.

### Рецензія правил

1. **Шляхи:** правило `/api/v1` і kebab-case відповідає вашим ендпоінтам, але має винятки: `GET /health` не має префікса, а `resend` — дієслово в шляху. Зафіксуйте їх явно. Решта шляхів уже використовує множину, наприклад `/notifications` і `/reports`.
2. **Формати:** camelCase для JSON — добре. Для часу визначте формат як RFC 3339 (`date-time` в OpenAPI), завжди з часовим поясом; бажано нормалізувати час у відповідях до UTC.
3. **Помилки:** `application/problem+json` і стабільні `type` URI узгоджуються з RFC 9457. Переконайтеся, що кожна помилка має відповідний `type`, а для `409` розрізнено конфлікт стану й запит, що вже виконується. Успішні відповіді мають використовувати звичайний JSON, не Problem Details.
4. **Пагінація:** задайте межі явно: `page >= 1`, `1 <= pageSize <= 100`, а також значення за замовчуванням. Невалідні значення — `422` із відповідним `type`.
5. **Кореляція:** правило корисне для трасування. В OpenAPI опишіть `X-Correlation-Id` як загальний параметр/заголовок для всіх операцій і як заголовок у відповідях, зокрема помилкових. Уточніть формат ID. Якщо клієнт передав невалідний ID, контракт має визначити відповідь; наприклад, `422`.

### Перевірка через mock-сервер

1. Команда Notifications публікує версію `openapi.yaml` разом із прикладами успішних і помилкових відповідей. Контракт містить схеми, статуси, заголовки, ролі й помилки `application/problem+json`.
2. Команди-споживачі запускають mock-сервер, наприклад Prism, на основі цієї версії контракту. Вони можуть інтегрувати свої клієнти до готовності реального сервісу, перевіряючи шляхи, параметри, заголовки й формати відповідей.
3. Перед погодженням контракту споживачі перевіряють потрібні їм сценарії: успішний перегляд, `401`/`403`, відсутній запис (`404`), конфлікт повтору (`409`) і помилку валідації (`422`). Для неоднозначних сценаріїв у контракті мають бути приклади або окремі mock-відповіді.
4. Контрактні тести запускаються і проти mock, і проти реального сервісу з однаковими схемами OpenAPI. Вони перевіряють статус, `Content-Type`, обов’язкові поля й заголовок кореляції. Mock допомагає перевірити клієнтську інтеграцію, але сам по собі не підтверджує реальну бізнес-логіку чи поведінку повторної доставки.

**Результат рецензії:** правила можна прийняти після того, як ви зафіксуєте винятки для `/health` і `resend`, межі пагінації та формат/валідацію `X-Correlation-Id`.

> Склади OpenAPI 3.1 (YAML) для ендпоінтів Notifications / Reporting.\
> \
> Ендпоінти: GET /health (liveness, без автентифікації, без префікса /api/v1); GET /api/v1/notifications; GET /api/v1/notifications/{id}; POST /api/v1/notifications/{id}/resend; GET /api/v1/reports/daily?date=; GET /api/v1/reports/inactive-renters?weekOf=; GET /api/v1/templates.\
> \
> GET /api/v1/notifications має фільтри dateFrom, dateTo, type, state та пагінацію page/pageSize (page >= 1, pageSize від 1 до 100, за замовчуванням page=1, pageSize=20). Невалідні значення дають 422.\
> \
> Схеми: Notification, DeliveryAttempt, DailyReport (completeness; відсутні метрики null), InactiveRentersReport, Template (без секретів), Problem (RFC 9457).\
> \
> Безпека: bearer JWT. Ролі director, warehouse_operator, sales_manager, admin. Email у журналі маскується, admin не бачить змісту листів.\
> \
> Заголовки: X-Correlation-Id у всіх запитах і відповідях, зокрема помилкових. Невалідний або відсутній X-Correlation-Id сервіс замінює новим UUID і повертає його у відповіді. Idempotency-Key (UUID, обов'язковий) для POST resend; ключ, хеш тіла й відповідь зберігаються 24 години.\
> \
> POST resend: тіло {"reason": "..."} (обов'язкове непорожнє). Повтор із тим самим ключем і тілом повертає ту саму 202 без нового циклу доставки; той самий ключ з іншим тілом дає 422.\
> \
> Відповіді: 200, 202 та помилки 400, 401, 403 (type urn:kavohub:problem:access-denied), 404, 409 (request-in-progress і notification-retry-not-allowed як різні type), 422, 503 — усі як application/problem+json. Чужий запис дає 403, неіснуючий ID дає 404. Час у форматі RFC 3339, у відповідях у UTC.\
> \
> Додай приклади успішних і помилкових відповідей для mock-сервера.

Нижче — чернетка контракту. Успішні відповіді мають `application/json` (а `/health` — `application/json`); `application/problem+json` використовується для помилок. Інакше опис `200` і `202` як Problem Details суперечив би їхнім схемам.

У контракті відсутній `X-Correlation-Id` описаний як допустимий: сервіс згенерує UUID і поверне його у відповіді. Для ролей використано `x-required-roles` як документаційне розширення OpenAPI: перевірка ролей і областей доступу виконується сервісом за claims JWT.

```yaml
openapi: 3.1.0
info:
  title: KavoHub Notifications / Reporting API
  version: 1.0.0
  description: |
    API журналу сповіщень, звітів і шаблонів.
    Успішні відповіді повертаються як application/json.
    Помилки повертаються як application/problem+json (RFC 9457).
    X-Correlation-Id у відповіді завжди містить UUID:
    передане валідне значення зберігається, відсутнє або невалідне замінюється новим UUID.
servers:
  - url: https://notifications.kavohub.example
    description: Production
  - url: http://localhost:8107
    description: Local development

security:
  - bearerAuth: []

tags:
  - name: Health
  - name: Notifications
  - name: Reports
  - name: Templates

paths:
  /health:
    get:
      tags: [Health]
      operationId: getLiveness
      summary: Перевірити liveness сервісу
      description: Liveness endpoint без автентифікації. Не перевіряє готовність залежностей.
      security: []
      parameters:
        - $ref: "#/components/parameters/CorrelationId"
      responses:
        "200":
          description: Процес сервісу працює
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/json:
              schema:
                type: object
                required: [status]
                properties:
                  status:
                    type: string
                    const: ok
              examples:
                alive:
                  value:
                    status: ok
        "503":
          $ref: "#/components/responses/ServiceUnavailable"

  /api/v1/notifications:
    get:
      tags: [Notifications]
      operationId: listNotifications
      summary: Переглянути журнал сповіщень
      description: |
        Email отримувача маскується. Вміст листа не повертається жодній ролі,
        включно з admin. Результати обмежуються доступною користувачеві областю.
      x-required-roles:
        - director
        - warehouse_operator
        - sales_manager
        - admin
      parameters:
        - $ref: "#/components/parameters/CorrelationId"
        - name: dateFrom
          in: query
          description: Початкова дата включно, ISO 8601 date.
          schema:
            type: string
            format: date
          example: "2026-11-01"
        - name: dateTo
          in: query
          description: Кінцева дата включно, ISO 8601 date.
          schema:
            type: string
            format: date
          example: "2026-11-24"
        - name: type
          in: query
          description: Тип сповіщення.
          schema:
            $ref: "#/components/schemas/NotificationType"
        - name: state
          in: query
          description: Поточний стан сповіщення.
          schema:
            $ref: "#/components/schemas/NotificationState"
        - name: page
          in: query
          description: Номер сторінки, починаючи з 1.
          schema:
            type: integer
            minimum: 1
            default: 1
          example: 1
        - name: pageSize
          in: query
          description: Кількість записів на сторінці.
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
          example: 20
      responses:
        "200":
          description: Сторінка журналу
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/NotificationPage"
              examples:
                populated:
                  value:
                    items:
                      - id: "a4b756a3-118a-4eae-8ef9-962f69fa7392"
                        notificationType: shipment_with_tracking
                        recipient:
                          id: "crm:partner:87"
                          emailMasked: "m***@example.com"
                        state: sent
                        attemptCount: 1
                        createdAt: "2026-11-24T15:00:00Z"
                        updatedAt: "2026-11-24T15:00:03Z"
                    page: 1
                    pageSize: 20
                    total: 1
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/AccessDenied"
        "422":
          description: Некоректні query-параметри, зокрема page або pageSize
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/Problem"
              examples:
                invalidPageSize:
                  value:
                    type: "urn:kavohub:problem:validation-error"
                    title: "Некоректний параметр"
                    status: 422
                    detail: "pageSize має бути цілим числом від 1 до 100."
                    instance: "/api/v1/notifications?pageSize=200"
        "503":
          $ref: "#/components/responses/ServiceUnavailable"

  /api/v1/notifications/{id}:
    get:
      tags: [Notifications]
      operationId: getNotification
      summary: Переглянути сповіщення та спроби доставки
      description: |
        Email отримувача маскується. Вміст листа не повертається жодній ролі,
        включно з admin. Для наявного, але недоступного користувачеві запису
        повертається 403; для неіснуючого ID — 404.
      x-required-roles:
        - director
        - warehouse_operator
        - sales_manager
        - admin
      parameters:
        - $ref: "#/components/parameters/CorrelationId"
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          description: Сповіщення та історія спроб доставки
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Notification"
              examples:
                sentNotification:
                  value:
                    id: "a4b756a3-118a-4eae-8ef9-962f69fa7392"
                    notificationType: shipment_with_tracking
                    recipient:
                      id: "crm:partner:87"
                      emailMasked: "m***@example.com"
                    state: sent
                    attemptCount: 1
                    deliveryCycle: 1
                    createdAt: "2026-11-24T15:00:00Z"
                    updatedAt: "2026-11-24T15:00:03Z"
                    attempts:
                      - attemptNumber: 1
                        outcome: sent
                        attemptedAt: "2026-11-24T15:00:02Z"
                        completedAt: "2026-11-24T15:00:03Z"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/AccessDenied"
        "404":
          description: Сповіщення з таким ID не існує
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/Problem"
              examples:
                notFound:
                  value:
                    type: "urn:kavohub:problem:notification-not-found"
                    title: "Сповіщення не знайдено"
                    status: 404
                    detail: "Сповіщення з указаним ID не існує."
                    instance: "/api/v1/notifications/a4b756a3-118a-4eae-8ef9-962f69fa7392"
        "503":
          $ref: "#/components/responses/ServiceUnavailable"

  /api/v1/notifications/{id}/resend:
    post:
      tags: [Notifications]
      operationId: resendNotification
      summary: Поставити повторну доставку в чергу
      description: |
        Дозволено лише admin і лише для сповіщення у стані final_failed.
        Запит асинхронно ставить доставку в чергу; сам лист надсилає worker.
        Idempotency-Key, хеш тіла та відповідь зберігаються 24 години.
        Той самий ключ із тим самим тілом повертає збережену відповідь 202
        і не створює нового циклу доставки. Той самий ключ з іншим тілом
        повертає 422.
      x-required-roles: [admin]
      parameters:
        - $ref: "#/components/parameters/CorrelationId"
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: Idempotency-Key
          in: header
          required: true
          description: UUID ключ ідемпотентності, чинний 24 години.
          schema:
            type: string
            format: uuid
          example: "de305d54-75b4-431b-adb2-eb6b9e546014"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ResendRequest"
            examples:
              operatorRetry:
                value:
                  reason: "Поштовий сервіс відновив роботу; повтор погоджено адміністратором."
      responses:
        "202":
          description: Повтор доставки поставлено в чергу
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ResendAccepted"
              examples:
                accepted:
                  value:
                    notificationId: "a4b756a3-118a-4eae-8ef9-962f69fa7392"
                    state: pending
                    deliveryCycle: 2
                    acceptedAt: "2026-11-24T15:10:00Z"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/AccessDenied"
        "404":
          description: Сповіщення з таким ID не існує
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/Problem"
              examples:
                notFound:
                  value:
                    type: "urn:kavohub:problem:notification-not-found"
                    title: "Сповіщення не знайдено"
                    status: 404
                    detail: "Сповіщення з указаним ID не існує."
                    instance: "/api/v1/notifications/a4b756a3-118a-4eae-8ef9-962f69fa7392"
        "409":
          description: Запит виконується або повтор для поточного стану заборонений
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/Problem"
              examples:
                requestInProgress:
                  value:
                    type: "urn:kavohub:problem:request-in-progress"
                    title: "Запит уже виконується"
                    status: 409
                    detail: "Для цього сповіщення вже виконується цикл повторної доставки."
                    instance: "/api/v1/notifications/a4b756a3-118a-4eae-8ef9-962f69fa7392/resend"
                retryNotAllowed:
                  value:
                    type: "urn:kavohub:problem:notification-retry-not-allowed"
                    title: "Повтор недоступний"
                    status: 409
                    detail: "Повтор дозволений лише для сповіщення у стані final_failed."
                    instance: "/api/v1/notifications/a4b756a3-118a-4eae-8ef9-962f69fa7392/resend"
        "422":
          description: Невалідний ключ, тіло або повторне використання ключа з іншим тілом
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/Problem"
              examples:
                keyReusedWithDifferentBody:
                  value:
                    type: "urn:kavohub:problem:idempotency-key-reuse"
                    title: "Ключ ідемпотентності вже використано"
                    status: 422
                    detail: "Цей Idempotency-Key уже використано з іншим тілом запиту."
                    instance: "/api/v1/notifications/a4b756a3-118a-4eae-8ef9-962f69fa7392/resend"
                emptyReason:
                  value:
                    type: "urn:kavohub:problem:validation-error"
                    title: "Некоректне тіло запиту"
                    status: 422
                    detail: "Поле reason обов'язкове й не може бути порожнім."
                    instance: "/api/v1/notifications/a4b756a3-118a-4eae-8ef9-962f69fa7392/resend"
        "503":
          $ref: "#/components/responses/ServiceUnavailable"

  /api/v1/reports/daily:
    get:
      tags: [Reports]
      operationId: getDailyReport
      summary: Отримати щоденний звіт
      x-required-roles: [director, admin]
      parameters:
        - $ref: "#/components/parameters/CorrelationId"
        - name: date
          in: query
          required: true
          description: Дата звіту, ISO 8601 date.
          schema:
            type: string
            format: date
          example: "2026-11-24"
      responses:
        "200":
          description: Щоденний знімок; відсутні метрики мають значення null
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/DailyReport"
              examples:
                incomplete:
                  value:
                    date: "2026-11-24"
                    generatedAt: "2026-11-24T18:00:00Z"
                    completeness:
                      status: incomplete
                      missingSources: [billing]
                      asOf:
                        inventory: "2026-11-24T17:55:00Z"
                    metrics:
                      shippedOrders: 18
                      overdueInvoices: null
                      overdueAmount: null
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/AccessDenied"
        "404":
          description: Знімок звіту для цієї дати відсутній
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/Problem"
              examples:
                reportNotFound:
                  value:
                    type: "urn:kavohub:problem:report-not-found"
                    title: "Звіт не знайдено"
                    status: 404
                    detail: "Щоденний знімок для цієї дати ще не сформовано."
                    instance: "/api/v1/reports/daily?date=2026-11-24"
        "422":
          $ref: "#/components/responses/InvalidQuery"
        "503":
          $ref: "#/components/responses/ServiceUnavailable"

  /api/v1/reports/inactive-renters:
    get:
      tags: [Reports]
      operationId: getInactiveRentersReport
      summary: Отримати звіт про неактивних орендарів
      x-required-roles: [director, admin, sales_manager]
      parameters:
        - $ref: "#/components/parameters/CorrelationId"
        - name: weekOf
          in: query
          required: true
          description: Дата понеділка звітного тижня, ISO 8601 date.
          schema:
            type: string
            format: date
          example: "2026-11-23"
      responses:
        "200":
          description: Знімок звіту про неактивних орендарів
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/InactiveRentersReport"
              examples:
                populated:
                  value:
                    weekOf: "2026-11-23"
                    generatedAt: "2026-11-24T18:00:00Z"
                    completeness:
                      status: complete
                      missingSources: []
                      asOf:
                        renters: "2026-11-24T17:55:00Z"
                    items:
                      - renterId: "renter-104"
                        displayName: "ТОВ Приклад"
                        lastActivityAt: "2026-10-30T09:15:00Z"
                    total: 1
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/AccessDenied"
        "404":
          description: Знімок звіту для цього тижня відсутній
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/Problem"
              examples:
                reportNotFound:
                  value:
                    type: "urn:kavohub:problem:report-not-found"
                    title: "Звіт не знайдено"
                    status: 404
                    detail: "Знімок звіту для цього тижня ще не сформовано."
                    instance: "/api/v1/reports/inactive-renters?weekOf=2026-11-23"
        "422":
          $ref: "#/components/responses/InvalidQuery"
        "503":
          $ref: "#/components/responses/ServiceUnavailable"

  /api/v1/templates:
    get:
      tags: [Templates]
      operationId: listTemplates
      summary: Переглянути метадані активних шаблонів
      description: Відповідь не містить тіла листа, SMTP credentials чи інших секретів.
      x-required-roles: [admin]
      parameters:
        - $ref: "#/components/parameters/CorrelationId"
      responses:
        "200":
          description: Метадані шаблонів без вмісту листів і секретів
          headers:
            X-Correlation-Id:
              $ref: "#/components/headers/CorrelationId"
          content:
            application/json:
              schema:
                type: object
                required: [items]
                properties:
                  items:
                    type: array
                    items:
                      $ref: "#/components/schemas/Template"
              examples:
                templates:
                  value:
                    items:
                      - id: "tpl-shipment-tracking-v1"
                        notificationType: shipment_with_tracking
                        name: "Відвантаження з номером ТТН"
                        locale: uk-UA
                        active: true
                        updatedAt: "2026-11-20T12:00:00Z"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/AccessDenied"
        "503":
          $ref: "#/components/responses/ServiceUnavailable"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  parameters:
    CorrelationId:
      name: X-Correlation-Id
      in: header
      required: false
      description: |
        UUID для кореляції запитів. Якщо значення відсутнє або не є UUID,
        сервіс згенерує новий UUID. Відповідь завжди містить X-Correlation-Id.
      schema:
        type: string
        format: uuid
      example: "7f2c8a90-1f66-4b79-9d15-2d20ad88f5b1"

  headers:
    CorrelationId:
      description: UUID кореляції; передане або згенероване сервісом значення.
      required: true
      schema:
        type: string
        format: uuid

  responses:
    Unauthorized:
      description: Відсутній або невалідний bearer JWT
      headers:
        X-Correlation-Id:
          $ref: "#/components/headers/CorrelationId"
      content:
        application/problem+json:
          schema:
            $ref: "#/components/schemas/Problem"
          examples:
            unauthorized:
              value:
                type: "urn:kavohub:problem:authentication-required"
                title: "Потрібна автентифікація"
                status: 401
                detail: "Надайте валідний bearer JWT."
                instance: "/api/v1/notifications"

    AccessDenied:
      description: Користувач не має ролі або доступу до цієї області даних
      headers:
        X-Correlation-Id:
          $ref: "#/components/headers/CorrelationId"
      content:
        application/problem+json:
          schema:
            $ref: "#/components/schemas/Problem"
          examples:
            forbidden:
              value:
                type: "urn:kavohub:problem:access-denied"
                title: "Доступ заборонено"
                status: 403
                detail: "Користувач не має доступу до цього ресурсу."
                instance: "/api/v1/notifications"

    InvalidQuery:
      description: Невалідне значення query-параметра
      headers:
        X-Correlation-Id:
          $ref: "#/components/headers/CorrelationId"
      content:
        application/problem+json:
          schema:
            $ref: "#/components/schemas/Problem"
          examples:
            invalidDate:
              value:
                type: "urn:kavohub:problem:validation-error"
                title: "Некоректний параметр"
                status: 422
                detail: "Параметр date має бути валідною датою у форматі YYYY-MM-DD."
                instance: "/api/v1/reports/daily?date=invalid"

    ServiceUnavailable:
      description: Сервіс тимчасово недоступний
      headers:
        X-Correlation-Id:
          $ref: "#/components/headers/CorrelationId"
        Retry-After:
          description: Кількість секунд до рекомендованого повтору, якщо відома.
          schema:
            type: integer
            minimum: 1
      content:
        application/problem+json:
          schema:
            $ref: "#/components/schemas/Problem"
          examples:
            unavailable:
              value:
                type: "urn:kavohub:problem:service-unavailable"
                title: "Сервіс тимчасово недоступний"
                status: 503
                detail: "Спробуйте повторити запит пізніше."
                instance: "/api/v1/notifications"

  schemas:
    NotificationType:
      type: string
      enum:
        - deal_won
        - order_confirmed
        - shipment_with_tracking
        - invoice_with_pdf
        - payment_received
        - daily_report
        - inactive_renters_report
        - it_delivery_failure

    NotificationState:
      type: string
      enum:
        - waiting
        - pending
        - sending
        - retry_wait
        - sent
        - permanent_failure
        - final_failed

    DeliveryAttemptOutcome:
      type: string
      enum:
        - sent
        - temporary_failure
        - permanent_failure

    NotificationRecipient:
      type: object
      additionalProperties: false
      required: [id, emailMasked]
      properties:
        id:
          type: string
          description: Зовнішній ідентифікатор отримувача.
          example: "crm:partner:87"
        emailMasked:
          type: string
          description: Маскована адреса; повна адреса не повертається.
          example: "m***@example.com"

    DeliveryAttempt:
      type: object
      additionalProperties: false
      required: [attemptNumber, outcome, attemptedAt]
      properties:
        attemptNumber:
          type: integer
          minimum: 1
        outcome:
          $ref: "#/components/schemas/DeliveryAttemptOutcome"
        attemptedAt:
          type: string
          format: date-time
          description: RFC 3339, UTC.
        completedAt:
          type: [string, "null"]
          format: date-time
          description: RFC 3339, UTC; null, якщо спроба ще триває.
        errorType:
          type: [string, "null"]
          description: Стабільний внутрішній код помилки без секретів.
        errorMessage:
          type: [string, "null"]
          description: Безпечний для журналу опис помилки; не містить вмісту листа.

    Notification:
      type: object
      additionalProperties: false
      required:
        - id
        - notificationType
        - recipient
        - state
        - attemptCount
        - deliveryCycle
        - createdAt
        - updatedAt
      properties:
        id:
          type: string
          format: uuid
        notificationType:
          $ref: "#/components/schemas/NotificationType"
        recipient:
          $ref: "#/components/schemas/NotificationRecipient"
        state:
          $ref: "#/components/schemas/NotificationState"
        attemptCount:
          type: integer
          minimum: 0
          description: Кількість спроб у поточному циклі доставки.
        deliveryCycle:
          type: integer
          minimum: 1
        nextActionAt:
          type: [string, "null"]
          format: date-time
          description: Наступна запланована дія у UTC або null.
        createdAt:
          type: string
          format: date-time
          description: RFC 3339, UTC.
        updatedAt:
          type: string
          format: date-time
          description: RFC 3339, UTC.
        attempts:
          type: array
          description: Історія спроб; присутня у відповіді GET за ID.
          items:
            $ref: "#/components/schemas/DeliveryAttempt"

    NotificationPage:
      type: object
      additionalProperties: false
      required: [items, page, pageSize, total]
      properties:
        items:
          type: array
          items:
            $ref: "#/components/schemas/Notification"
        page:
          type: integer
          minimum: 1
        pageSize:
          type: integer
          minimum: 1
          maximum: 100
        total:
          type: integer
          minimum: 0

    Completeness:
      type: object
      additionalProperties: false
      required: [status, missingSources, asOf]
      properties:
        status:
          type: string
          enum: [complete, incomplete]
        missingSources:
          type: array
          items:
            type: string
        asOf:
          type: object
          description: Час актуальності даних за джерелом, RFC 3339 UTC.
          additionalProperties:
            type: [string, "null"]
            format: date-time

    DailyReport:
      type: object
      additionalProperties: false
      required: [date, generatedAt, completeness, metrics]
      properties:
        date:
          type: string
          format: date
        generatedAt:
          type: string
          format: date-time
          description: RFC 3339, UTC.
        completeness:
          $ref: "#/components/schemas/Completeness"
        metrics:
          type: object
          description: Метрика, яку неможливо обчислити через відсутність даних, має null, а не 0.
          additionalProperties:
            type: [integer, number, "null"]

    InactiveRenter:
      type: object
      additionalProperties: false
      required: [renterId, displayName, lastActivityAt]
      properties:
        renterId:
          type: string
        displayName:
          type: string
        lastActivityAt:
          type: [string, "null"]
          format: date-time
          description: RFC 3339, UTC; null, якщо активність невідома.

    InactiveRentersReport:
      type: object
      additionalProperties: false
      required: [weekOf, generatedAt, completeness, items, total]
      properties:
        weekOf:
          type: string
          format: date
          description: Понеділок звітного тижня.
        generatedAt:
          type: string
          format: date-time
          description: RFC 3339, UTC.
        completeness:
          $ref: "#/components/schemas/Completeness"
        items:
          type: array
          items:
            $ref: "#/components/schemas/InactiveRenter"
        total:
          type: integer
          minimum: 0

    Template:
      type: object
      additionalProperties: false
      description: Метадані шаблону בלבד; тіло листа та секрети не надаються.
      required: [id, notificationType, name, locale, active, updatedAt]
      properties:
        id:
          type: string
        notificationType:
          $ref: "#/components/schemas/NotificationType"
        name:
          type: string
        locale:
          type: string
          example: uk-UA
        active:
          type: boolean
        updatedAt:
          type: string
          format: date-time
          description: RFC 3339, UTC.

    ResendRequest:
      type: object
      additionalProperties: false
      required: [reason]
      properties:
        reason:
          type: string
          minLength: 1
          pattern: ".*\\S.*"
          maxLength: 1000
          description: Непорожнє пояснення для аудиту.

    ResendAccepted:
      type: object
      additionalProperties: false
      required: [notificationId, state, deliveryCycle, acceptedAt]
      properties:
        notificationId:
          type: string
          format: uuid
        state:
          type: string
          const: pending
        deliveryCycle:
          type: integer
          minimum: 1
        acceptedAt:
          type: string
          format: date-time
          description: RFC 3339, UTC.

    Problem:
      type: object
      description: RFC 9457 Problem Details.
      required: [type, title, status]
      properties:
        type:
          type: string
          format: uri
          description: Стабільний URI типу помилки urn:kavohub:problem:...
        title:
          type: string
        status:
          type: integer
          minimum: 400
          maximum: 599
        detail:
          type: string
        instance:
          type: string
          format: uri-reference
        errors:
          type: array
          description: Необов'язкові деталі валідації.
          items:
            type: object
            required: [name, reason]
            properties:
              name:
                type: string
              reason:
                type: string
      examples:
        validation:
          type: "urn:kavohub:problem:validation-error"
          title: "Некоректний запит"
          status: 422
          detail: "Перевірте параметри запиту."
          instance: "/api/v1/notifications?page=0"
``

> Виправ openapi.yaml: 1) у схемі Problem поле examples має бути масивом (або перенеси приклади в components/examples), щоб специфікація проходила валідацію; 2) у прикладі GET /api/v1/notifications додай обов'язкове deliveryCycle; 3) в описі схеми Template заміни сторонні символи на слово «лише»; 4) NotificationState = waiting, pending, sending, retry_wait, sent, final_failed, suppressed (permanent_failure прибери); 5) DeliveryAttempt: поля attemptNo, startedAt, finishedAt, outcome (in_progress, success, transient_failure, permanent_failure), errorCode; 6) NotificationType узгодь із матрицею ЛР2: deal_won, order_confirmed, order_cancelled, reserve_failed_customer, reserve_failed_internal, reserve_expired_customer, reserve_expired_internal, shipment_with_tracking, delivery_status_changed, invoice_after_dispatch, payment_received, discount_approval_requested, discount_decision, daily_summary, inactive_renters_weekly, it_final_failure; 7) метрики DailyReport: ordersAccepted, shipmentsDispatched, outstandingDebt і currency; 8) GET /reports/daily лише director; для inactive-renters вкажи, що sales_manager бачить лише свою команду; 9) у Completeness додай staleSources (source, lastSyncAt). Після виправлень перевір, що YAML валідний за OpenAPI 3.1 і всі приклади відповідають схемам.

<details><summary>4 previous messages</summary>

> Знайду `openapi.yaml` у робочій теці, внесу ці зміни й перевірю синтаксис YAML та відповідність прикладів схемам OpenAPI.
>
> <details><summary>Created a file, ran 5 commands</summary>
>
> - Ran `rg --files -g 'openapi.yaml' -g '*.yaml' -g '*.yml' -g 'package.json' -g 'pyproject.toml' -g 'requirements*.txt'`
> - Wrote `./openapi.yaml` (+730 -0)
> - Ran `python -c "import importlib.util as u; print('yaml', bool(u.find_spec('yaml'))); print('openapi_spec_validator', bool(u.find_spec('openapi_spec_validator'))); print('jsonschema', bool(u.find_spec('jsonschema')))"`
> - Ran `node -e "for (const p of ['yaml','js-yaml','@apidevtools/swagger-parser','openapi-schema-validator']) { try { console.log(p,require.resolve(p)) } catch { console.log(p,'missing') } }"`
> - Ran ``$names = 'yq','ruby','go','npx','docker','yamllint'; foreach ($name in $names) { $cmd = Get-Command $name -ErrorAction SilentlyContinue; if ($cmd) { "$name`t$($cmd.Source)" } }``
> - Ran `Get-Module -ListAvailable | Where-Object { $_.Name -match 'Yaml|Powershell-Yaml' } | Select-Object -ExpandProperty Name`
>
> </details>
>
> Зміни внесені. Зараз перевіряю посилання в OpenAPI, перелічені поля й приклади; у середовищі не знайшов встановленого YAML/OpenAPI-валідатора, тож перевірю, чи доступний локальний глобальний пакет без завантаження залежностей.
>
> <details><summary>Ran 5 commands</summary>
>
> - Ran `npm root -g`
> - Ran `docker image ls --format "{{.Repository}}:{{.Tag}}"`
> - Ran `rg -n "permanent_failure|attemptNo|finishedAt|deliveryCycle|reserve_failed_customer|ordersAccepted|staleSources|x-required-roles|description: Метадані шаблону" openapi.yaml; git status --short`
> - Ran `python -m pip install --disable-pip-version-check --target C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation pyyaml openapi-spec-validator`
> - Ran `@'
> import sys
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> from openapi_spec_validator import validate
> from openapi_schema_validator import OAS31Validator
> from jsonschema import RefResolver
>
> path = 'openapi.yaml'
> with open(path, encoding='utf-8') as f:
>     doc = yaml.safe_load(f)
> validate(doc)
> resolver = RefResolver.from_schema(doc)
> def deref(obj):
>     while isinstance(obj, dict) and '$ref' in obj:
>         ref = obj['$ref']
>         if not ref.startswith('#/'):
>             raise ValueError('external ref: ' + ref)
>         obj = doc
>         for part in ref[2:].split('/'):
>             obj = obj[part.replace('~1','/').replace('~0','~')]
>     return obj
> checked = 0
> for route, path_item in doc['paths'].items():
>     for method, operation in path_item.items():
>         if method not in {'get','post','put','patch','delete','options','head','trace'}:
>             continue
>         for status, response_ref in operation.get('responses', {}).items():
>             response = deref(response_ref)
>             for media_type, media in response.get('content', {}).items():
>                 schema = media.get('schema')
>                 if not schema:
>                     continue
>                 for example_ref in media.get('examples', {}).values():
>                     example = deref(example_ref)
>                     value = example.get('value')
>                     if value is not None:
>                         OAS31Validator(schema, resolver=resolver).validate(value)
>                         checked += 1
> print(f'OpenAPI valid; validated {checked} response examples against their schemas.')
> '@ | python -`
>
> </details>
</details>

Оновив [openapi.yaml](./openapi.yaml): привів стани, типи сповіщень, поля спроб доставки, метрики звіту й `staleSources` до ваших вимог. Виправив опис `Template`, додав `deliveryCycle` у приклад журналу, обмежив щоденний звіт роллю `director` і вказав, що `sales_manager` бачить звіт орендарів лише своєї команди.

**Перевірка пройшла:** OpenAPI 3.1 валідний; 43 приклади відповідей перевірені на відповідність схемам.

> Мої підписки Notifications / Reporting для AsyncAPI 3.0 (RabbitMQ, exchange kavohub.events): crm.deal.won (повідомити менеджера); orders.order.confirmed (підтвердження клієнтці, якщо резерв чинний); orders.order.cancelled; inventory.stock.reserved (лише оновити репліку резерву); inventory.stock.reservation_failed; inventory.shipment.dispatched (лист з ТТН; розблокувати рахунок, що чекає); billing.invoice.issued (лист з PDF після відвантаження); billing.payment.received; алерти DLQ (повідомити ІТ). Власні події на спільну шину модуль не публікує; внутрішні події notifications.\* лишаються в журналі й метриках. Для кожної підписки опиши: обробник, зміну стану або даних, дедуплікацію (processed_events за (source, event_id), ключ сповіщення triggerOccurrenceId + recipientId + notificationType), поведінку при події не за порядком (наприклад, рахунок раніше відвантаження), обов'язкові поля конверта (id, type, source, time, correlationId, data). Рецензуй: що пропущено чи суперечить.

Загальний напрям узгоджується з ЛР2: Notifications споживає події, надсилає листи та веде локальний журнал, але **не публікує власні події на спільну шину**. Внутрішні `notifications.*` можна лишити як типи записів журналу/метрик — тільки не описувати їх як повідомлення AsyncAPI для `kavohub.events`.

| Підписка | Обробник і зміна даних | Подія не за порядком |
|---|---|---|
| `crm.deal.won` | Записати подію; створити повідомлення менеджеру про виграну угоду. Отримувача визначити за відповідальним із CRM або його локальної репліки. | Якщо даних менеджера ще немає, відкласти обробку/лист до появи репліки; не позначати доставку успішною. |
| `orders.order.confirmed` | Записати підтвердження й оновити локальну репліку замовлення. Створити клієнтське підтвердження лише коли відомо, що резерв чинний. | Якщо `inventory.stock.reserved` ще не оброблено, зберегти підтвердження як очікуване й переоцінити після оновлення резерву. Не виводити чинність резерву лише з факту підтвердження замовлення. |
| `orders.order.cancelled` | Оновити репліку замовлення; скасувати очікувані повідомлення, які вже не мають надсилатися, та за потреби створити повідомлення клієнту про скасування. | Якщо скасування прийшло перед підтвердженням, зберегти фінальний стан із версією/часом джерела, щоб пізніше підтвердження не відновило скасоване замовлення. Уточнити, що робити зі скасуванням після відвантаження. |
| `inventory.stock.reserved` | Оновити репліку резерву. Не надсилати лист лише через сам факт резервування, якщо це не окрема бізнес-вимога. | Пізніша застаріла подія не повинна перезаписати новіший стан резерву. Потрібне поле версії або надійний порядок для одного замовлення. |
| `inventory.stock.reservation_failed` | Оновити репліку невдалого резерву й створити відповідне повідомлення клієнту або внутрішньому адресату за причиною/типом помилки. | Якщо після невдачі резервування вже прийшов успішний резерв, перевірити версію/порядок і не надсилати застаріле повідомлення про відмову. |
| `inventory.shipment.dispatched` | Зберегти відвантаження і ТТН; створити лист клієнту з ТТН. Позначити shipment як виконану передумову для надсилання рахунку. | Якщо подія рахунку вже є, розблокувати відкладений лист із PDF. Переконатися, що подія має стабільні `orderId`/ключ кореляції та shipment ID. |
| `billing.invoice.issued` | Зберегти рахунок і посилання/ідентифікатор PDF. Надсилати лист після підтвердження відвантаження; до того — зберігати рахунок в очікуванні. | Якщо рахунок прийшов раніше за `inventory.shipment.dispatched`, не надсилати завчасно; після події відвантаження повторно перевірити умови й надіслати один раз. |
| `billing.payment.received` | Оновити локальну фінансову репліку/знімки звіту; за визначеним правилом створити квитанцію клієнту або повідомлення відповідальним. | Пізня подія про оплату не повинна повернути рахунок у стан боргу. Потрібні версія/час або правила пріоритету статусів. |
| DLQ-алерти | Обробник алертів створює одне внутрішнє ІТ-сповіщення з назвою модуля, routing key, ID повідомлення, причиною та кількістю спроб. | Повторний alert для тієї самої DLQ-записи не має породжувати новий лист; повторне надходження після ручного requeue треба розрізняти за ID/циклом обробки. |

### Що треба узгодити перед AsyncAPI

- **DLQ не належить до `kavohub.events`.** За методичкою кожен модуль має власну `<module>.events.dlq`. Щоб Notifications читав DLQ інших модулів, потрібні окремі права й погоджені черги/маршрути. Також зафіксуйте, хто створює alert: модуль-власник DLQ чи Notifications.
- **Розблокування рахунку — це термін процесу.** Подія `inventory.shipment.dispatched` не має змінювати рахунок у Billing. Вона лише робить можливим надсилання листа з уже випущеним рахунком. Таке розділення збереже межі відповідальності Billing і Notifications.
- **Не вистачає ідентифікаторів для кореляції.** У `billing.invoice.issued` та `inventory.shipment.dispatched` має бути спільний стабільний ключ, щоб знайти потрібний рахунок для відвантаження. Це відкрите Q-001 із ЛР2. Переконайтеся також, що `invoice.issued` дає PDF або спосіб отримати його з Odoo.
- **Загальні поля конверта ще не є повним контрактом.** Для кожного типу події потрібні типізовані поля `data`, формати й обов’язковість. Крім `id`, `type`, `source`, `time`, `correlationId`, додайте `specversion`, як у конверті з README, а для повідомлень із кількома записами — ключ агрегату/версію, необхідну для визначення застарілих подій.
- **Дедуплікація має два рівні.** `processed_events(source, event_id)` зупиняє повторну бізнес-обробку події. Ключ `triggerOccurrenceId + recipientId + notificationType` зупиняє дубль створення конкретного повідомлення. Вкажіть, як саме `triggerOccurrenceId` стабільно формується з `event_id`, і як вести нові цикли доставки після ручного resend.
- **Транзакція не гарантує одноразове надсилання листа.** Збережіть факт обробки, зміну репліки й запис про заплановане повідомлення транзакційно (наприклад, через outbox/таблицю черги). Окремо опишіть невизначений результат, коли поштовий провайдер прийняв лист, але підтвердження загубилося.
- **Політика для резерву потребує точного правила.** `orders.order.confirmed` може надійти до `inventory.stock.reserved`; визначте, скільки чекати на резерв, що робити після тайм-ауту й чи потрібне клієнтське повідомлення, якщо резерв так і не підтверджено.
- **Для скасування й оплати бракує бізнес-умов.** Вкажіть, кому повідомляти про скасування, що робити після відвантаження/виставлення рахунку, і чи `billing.payment.received` надсилає клієнтську квитанцію, чи лише оновлює звіт.

Ці уточнення можна позначити відкритими рішеннями у документі ЛР3, але їхні наслідки для відкладених повідомлень, повторів і кореляції слід описати до завершення AsyncAPI-контракту.

> Прийми зауваження й зафіксуй рішення: 1) DLQ-алерт створює модуль-власник DLQ, Notifications лише споживає окремий канал алертів і шле лист ІТ, права на читання чужих DLQ не потрібні; 2) спільний ключ рахунку й відвантаження — orderId у data обох подій, shipmentId необов'язковий; 3) після orders.order.confirmed чекати inventory.stock.reserved до 15 хвилин; без резерву клієнту нічого не йде, ІТ отримує внутрішній алерт; 4) скасування сповіщає клієнтку, якщо вона вже отримала підтвердження; після відвантаження лист про скасування не йде, сповіщається менеджер; payment.received лише оновлює звіт і шле лист менеджеру, квитанції клієнтці немає; 5) для замовлення й резерву в data є version, подія з меншою версією не перезаписує новішу; 6) triggerOccurrenceId = event.id, для кількох відвантажень event.id + shipmentId, resend збільшує deliveryCycle, але не змінює triggerOccurrenceId; 7) якщо провайдер прийняв лист, а підтвердження загубилося, спроба вважається transient_failure, ризик дубля записати як припущення; 8) конверт: specversion, id, type, source, time, correlationId, data. Дай оновлену таблицю підписок і перелік відкритих питань.

## Оновлена таблиця підписок

Для всіх подій Notifications спершу дедуплікує обробку за `(source, event_id)` в `processed_events`. Створення повідомлення дедуплікується окремо за `triggerOccurrenceId + recipientId + notificationType`. Вхідний конверт має `specversion`, `id`, `type`, `source`, `time`, `correlationId`, `data`.

| Подія | Обробник і зміна даних | Повідомлення | Дублікати й події не за порядком |
|---|---|---|---|
| `crm.deal.won` | Обробник угоди зберігає подію та оновлює локальний стан угоди. | Повідомлення відповідальному менеджеру. | Дублікат пропускається. Для визначення актуального стану угоди потрібна версія; подія зі старішою версією не перезаписує новішу. |
| `orders.order.confirmed` | Обробник замовлення зберігає версію та ставить очікування резерву на 15 хвилин. | До підтвердження резерву клієнтці нічого не надсилається. Якщо резерв не надійшов за 15 хв, створюється внутрішній ІТ-алерт. | Дублікат не створює нового таймера/повідомлення. Подія зі старішою версією замовлення ігнорується. |
| `inventory.stock.reserved` | Оновлює репліку резерву. Якщо відповідне підтверджене замовлення ще чекає, знімає очікування. | Після підтвердження резерву можна надіслати клієнтці підтвердження замовлення. | Дублікат не змінює повторно дані. Подія зі старішою версією резерву ігнорується. |
| `inventory.stock.reservation_failed` | Оновлює репліку невдалого резерву й пов’язаний стан замовлення. | Повідомлення клієнтці або внутрішній адресат визначаються типом помилки та правилами відповідної категорії. | Застаріла подія не перезаписує новіший стан резерву. Потрібно визначити, як вона співвідноситься з 15-хвилинним очікуванням. |
| `orders.order.cancelled` | Оновлює стан замовлення. Перевіряє, чи підтвердження вже було доставлене, і чи відвантаження вже відбулося. | Клієнтці — якщо вона вже отримала підтвердження. Якщо замовлення вже відвантажене — лист про скасування не надсилається, повідомляється менеджер. | Подія з меншою версією замовлення не перезаписує новішу. Дублікати не створюють повторного повідомлення. |
| `inventory.shipment.dispatched` | Зберігає відвантаження, прив’язане до `orderId`; `shipmentId` може бути в `data`. Перевіряє наявність відкладеного рахунку. | Лист клієнтці з ТТН. Якщо рахунок уже надійшов, розблоковує надсилання листа з PDF. | Подія рахунку може прийти раніше: її зберігають в очікуванні до відвантаження. Ключ для кількох відвантажень — `event.id + shipmentId`. |
| `billing.invoice.issued` | Зберігає рахунок і PDF/посилання на нього за `orderId`. | Лист із PDF надсилається лише після події відвантаження. | Якщо рахунок надійшов першим, він залишається очікувати відвантаження. Дублікати не створюють другого листа. |
| `billing.payment.received` | Оновлює дані для звіту про заборгованість. | Лист менеджеру; квитанція клієнтці не надсилається. | Дублікат не повторює оновлення й лист. Потрібні версія або правило пріоритету, щоб запізнілий статус не повернув оплачений рахунок до боргу. |
| Окремий канал DLQ-алертів | Notifications читає алерт, створений модулем-власником DLQ; не читає чужі DLQ. | Внутрішній лист ІТ. | Ідемпотентність — за стабільним ID алерту/початкового повідомлення і типом алерту. Повторний алерт не має породжувати повторний лист без нового циклу інциденту. |

Модуль **не публікує власних подій на `kavohub.events`**. Внутрішні `notifications.*` залишаються записами журналу та метрик.

### Виявлена невідповідність

У матриці типів ЛР2 є `delivery_status_changed`, але серед підписок немає події про зміну статусу доставки. Треба або додати відповідний тип події й обробник, або вилучити тип сповіщення/вимогу, якщо клієнтів не потрібно повідомляти про зміну статусу.

## Відкриті питання

1. **15 хвилин очікування резерву:** від якого моменту рахується таймер — `time` події чи момент її фактичної обробки? Що робити, якщо `stock.reserved` прийде після тайм-ауту: лише оновити репліку чи ще й дозволити клієнтське підтвердження?
2. **Умова ІТ-алерту:** чи має алерт при відсутності резерву після 15 хвилин створюватися один раз на замовлення й версію? Як його дедуплікувати з `reservation_failed`?
3. **Кілька відвантажень:** `shipmentId` названо необов’язковим, але він потрібен для розрізнення кількох відвантажень одного замовлення. Чи може для замовлення бути кілька shipment-подій, і коли `shipmentId` стає обов’язковим?
4. **Формат `triggerOccurrenceId`:** `event.id + shipmentId` може вже не бути UUID, тоді як у ЛР2 `trigger_occurrence_id` має тип UUID. Потрібно вирішити, чи зберігати складений ключ як рядок, чи детерміновано перетворювати його на UUID.
5. **Менеджер-отримувач:** у яких полях подій/репліки знаходити менеджера для `deal.won`, `order.cancelled` після відвантаження та `payment.received`? Яка поведінка, якщо призначеного менеджера не знайдено?
6. **Відкладений рахунок:** що робити, якщо PDF ще недоступний у Odoo після появи обох подій? Потрібні ліміт і інтервали повторів, строк зберігання очікуваного листа та фінальна дія після невдачі.
7. **Версії та рівні агрегування:** підтвердити, що `version` монотонно зростає окремо для замовлення й резерву. Як обробляти однакову версію з різним вмістом?
8. **Невизначений результат надсилання:** прийнято припущення, що втрата підтвердження від провайдера означає `transient_failure`, отже можливий дубль листа. Чи є поштовий провайдер із ключем ідемпотентності або механізмом перевірки результату?
9. **DLQ-канал:** визначити назву exchange/routing key, схему алерту, стабільний ID та правила повторного створення алерту після ручного requeue. Власники DLQ мають погодити цей контракт.
10. **Статус доставки:** чи входить сповіщення `delivery_status_changed` у межі ЛР3? Якщо так, потрібні джерело події, одержувач і правило дедуплікації.

> Прийми рішення й оновлюй таблицю. 1) crm.deal.won: не веде стан угоди, лише зберігає подію й отримувача. 2) ІТ-алерт про відсутність резерву дедуплікується за orderId + version. 3) Таймер 15 хв рахується від time події order.confirmed; запізнілий stock.reserved лише оновлює репліку, клієнтське підтвердження не надсилається; reservation_failed скасовує таймер і алерт не створюється. 4) shipmentId обов'язковий у inventory.shipment.dispatched; для billing.invoice.issued ключ — orderId. 5) triggerOccurrenceId = UUIDv5 від рядка event.id:shipmentId. 6) Менеджер береться з ownerId у crm.deal.won і з репліки замовлення; якщо його немає, лист іде директору й це фіксується в журналі. 7) Якщо PDF недоступний: 3 повтори з інтервалом 5 хв, потім final_failed та ІТ-алерт. 8) version монотонний окремо для замовлення й резерву; однакова версія з іншим вмістом іде в DLQ як порушення контракту. 9) Алерти DLQ: routing key dlq.alert, стабільний ID з ID початкового повідомлення, ручний requeue збільшує incidentCycle. 10) delivery_status_changed лишається поза межами ЛР3 і вноситься у відкриті питання. Поштовий провайдер і його ідемпотентність лишаються відкритим питанням. Після цього згенеруй asyncapi.yaml версії 3.0.0: канали й операції receive для кожної підписки, окрема схема data для кожної події, спільний конверт зі specversion, id, type, source, time, correlationId, data; приклади повідомлень; без жодної операції send. Перевір, що YAML валідний.

<details><summary>4 previous messages</summary>

> Зафіксую ці рішення в AsyncAPI-контракті: окремі `receive`-операції для бізнес-подій і DLQ-алертів, типізовані `data`-схеми та приклади конвертів. Потім перевірю YAML і відповідність AsyncAPI 3.0.0.
>
> <details><summary>Created a file, edited a file, ran 5 commands, searched the web</summary>
>
> Searched the web for `site:asyncapi.com/docs/reference/specification/v3.0.0 channels operations receive message examples AsyncAPI 3.0 ...`
>
> Searched the web for `site:asyncapi.com/docs/reference/bindings/amqp channel binding exchange queue is routingKey 0-9-1`
>
> - Wrote `./asyncapi.yaml` (+510 -0)
>
> - Ran `@'
> import sys
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> from jsonschema import Draft202012Validator, RefResolver
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     doc = yaml.safe_load(f)
> assert doc['asyncapi'] == '3.0.0'
> assert all(op['action'] == 'receive' for op in doc['operations'].values())
> assert len(doc['operations']) == 9
> resolver = RefResolver.from_schema(doc)
> checked = 0
> for message_name, message in doc['components']['messages'].items():
>     schema = message['payload']
>     for example in message.get('examples', []):
>         Draft202012Validator(schema, resolver=resolver).validate(example['payload'])
>         checked += 1
> print(f'YAML parsed; {len(doc["operations"])} receive operations; no send operations; {checked} message examples validated against payload schemas.')
> '@ | python -`
>
> - Ran `Get-Content asyncapi.yaml | Select-Object -First 28`
>
> - Ran `rg -n 'description:.*:' asyncapi.yaml`
>
> - Wrote `./asyncapi.yaml` (+3 -3)
>
> - Ran `@'
> import sys
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> from jsonschema import Draft202012Validator, RefResolver
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     doc = yaml.safe_load(f)
> assert doc['asyncapi'] == '3.0.0'
> assert all(op['action'] == 'receive' for op in doc['operations'].values())
> assert len(doc['operations']) == 9
> resolver = RefResolver.from_schema(doc)
> checked = 0
> for message_name, message in doc['components']['messages'].items():
>     schema = message['payload']
>     for example in message.get('examples', []):
>         Draft202012Validator(schema, resolver=resolver).validate(example['payload'])
>         checked += 1
> print(f'YAML parsed; {len(doc["operations"])} receive operations; no send operations; {checked} message examples validated against payload schemas.')
> '@ | python -`
>
> Searched the web for `https://raw.githubusercontent.com/asyncapi/spec-json-schemas/master/schemas/3.0.0.json`
>
> - Ran `@'
> import json, sys, urllib.request
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> from jsonschema import Draft7Validator, RefResolver
> schema_url = 'https://raw.githubusercontent.com/asyncapi/spec-json-schemas/master/schemas/3.0.0.json'
> with urllib.request.urlopen(schema_url, timeout=20) as response:
>     spec_schema = json.load(response)
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     document = yaml.safe_load(f)
> store = {key: value for key, value in spec_schema.get('definitions', {}).items()}
> for value in store.values():
>     if isinstance(value, dict) and '$id' in value:
>         store[value['$id']] = value
> resolver = RefResolver.from_schema(spec_schema, store=store)
> errors = sorted(Draft7Validator(spec_schema, resolver=resolver).iter_errors(document), key=lambda e: list(e.absolute_path))
> if errors:
>     for error in errors[:30]:
>         print('/'.join(map(str, error.absolute_path)), error.message)
>     sys.exit(1)
> print('Valid AsyncAPI 3.0.0 document against the official AsyncAPI JSON Schema.')
> '@ | python -`
>
> </details>
>
> YAML уже розбирається, усі 9 прикладів відповідають схемам, а документ проходить офіційну JSON Schema AsyncAPI 3.0.0. Доповню контракт ще кількома прийнятими правилами дедуплікації, версій і відкритих питань, щоб рішення були зафіксовані не лише в діалозі.
>
> <details><summary>Edited a file, ran 2 commands</summary>
>
> - Wrote `./asyncapi.yaml` (+14 -6)
> - Ran `@'
> import json, sys, urllib.request, yaml
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> from jsonschema import Draft7Validator, Draft202012Validator, RefResolver
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     document = yaml.safe_load(f)
> assert document['asyncapi'] == '3.0.0'
> assert all(operation['action'] == 'receive' for operation in document['operations'].values())
> assert len(document['operations']) == 9
> checked = 0
> example_resolver = RefResolver.from_schema(document)
> for message in document['components']['messages'].values():
>     for example in message.get('examples', []):
>         Draft202012Validator(message['payload'], resolver=example_resolver).validate(example['payload'])
>         checked += 1
> schema_url = 'https://raw.githubusercontent.com/asyncapi/spec-json-schemas/master/schemas/3.0.0.json'
> with urllib.request.urlopen(schema_url, timeout=20) as response:
>     schema = json.load(response)
> store = dict(schema.get('definitions', {}))
> for value in list(store.values()):
>     if isinstance(value, dict) and '$id' in value:
>         store[value['$id']] = value
> resolver = RefResolver.from_schema(schema, store=store)
> errors = sorted(Draft7Validator(schema, resolver=resolver).iter_errors(document), key=lambda error: list(error.absolute_path))
> if errors:
>     for error in errors[:30]:
>         print('/'.join(map(str, error.absolute_path)), error.message)
>     sys.exit(1)
> print(f'Valid AsyncAPI 3.0.0; {len(document["operations"])} receive operations and {checked} examples validated; no send operations.')
> '@ | python -`
> - Ran `@'
> import json, sys, urllib.request
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> from jsonschema import Draft7Validator, Draft202012Validator, RefResolver
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     document = yaml.safe_load(f)
> assert document['asyncapi'] == '3.0.0'
> assert all(operation['action'] == 'receive' for operation in document['operations'].values())
> assert len(document['operations']) == 9
> checked = 0
> example_resolver = RefResolver.from_schema(document)
> for message in document['components']['messages'].values():
>     for example in message.get('examples', []):
>         Draft202012Validator(message['payload'], resolver=example_resolver).validate(example['payload'])
>         checked += 1
> schema_url = 'https://raw.githubusercontent.com/asyncapi/spec-json-schemas/master/schemas/3.0.0.json'
> with urllib.request.urlopen(schema_url, timeout=20) as response:
>     schema = json.load(response)
> store = dict(schema.get('definitions', {}))
> for value in list(store.values()):
>     if isinstance(value, dict) and '$id' in value:
>         store[value['$id']] = value
> resolver = RefResolver.from_schema(schema, store=store)
> errors = sorted(Draft7Validator(schema, resolver=resolver).iter_errors(document), key=lambda error: list(error.absolute_path))
> if errors:
>     for error in errors[:30]:
>         print('/'.join(map(str, error.absolute_path)), error.message)
>     sys.exit(1)
> print(f'Valid AsyncAPI 3.0.0; {len(document["operations"])} receive operations and {checked} examples validated; no send operations.')
> '@ | python -`
>
> </details>
</details>

Створив [asyncapi.yaml](./asyncapi.yaml) з 9 `receive`-операціями: вісім бізнес-підписок і окремий канал `dlq.alert`. Операцій `send` немає. У контракті зафіксував ваші рішення щодо 15-хвилинного очікування резерву, версій, дедуплікації, `UUIDv5`, fallback на директора, повторів отримання PDF та циклів DLQ-інцидентів.

**Перевірка пройшла:** YAML валідний за офіційною JSON Schema AsyncAPI 3.0.0; усі 9 прикладів перевірені на відповідність payload-схемам. Структура використовує `receive`-операції й AMQP routing-key bindings згідно зі [специфікацією AsyncAPI 3.0](https://www.asyncapi.com/docs/reference/specification/v3.0.0) та [AMQP bindings](https://www.asyncapi.com/docs/reference/bindings/amqp).

В описі також залишив відкритими питання про поштового провайдера та його ідемпотентність, а `delivery_status_changed` позначив поза межами цієї версії.

> Виправ asyncapi.yaml: 1) додай в info.description опис черг: notifications.events (основна), notifications.events.retry, notifications.events.dlq, notifications.dlq-alerts; 2) в прикладі ReservationFailed заміни orderId на ORD-2026-000105; 3) в описі version для OrderConfirmed/OrderCancelled уточни «версія агрегату замовлення», для StockReserved/ReservationFailed «версія агрегату резерву»; 4) в DlqAlert поясни, що correlationId беруть з початкового повідомлення, а ключ дедуплікації це alertId + incidentCycle; 5) в info.description додай відкриті питання: поштовий провайдер та його ідемпотентність, узгодження формату DLQ-алерту з власниками DLQ. Не змінюй решту й перевір валідність.

<details><summary>2 previous messages</summary>

> Внесу лише запитані уточнення в `asyncapi.yaml`, потім перевірю YAML і приклади повідомлень.
>
> <details><summary>Edited a file, ran 2 commands</summary>
>
> - Wrote `./asyncapi.yaml` (+13 -6)
> - Ran `@'
> import sys
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> from jsonschema import Draft202012Validator, RefResolver
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     doc = yaml.safe_load(f)
> assert doc['asyncapi'] == '3.0.0'
> assert all(op['action'] == 'receive' for op in doc['operations'].values())
> assert len(doc['operations']) == 9
> resolver = RefResolver.from_schema(doc)
> count = 0
> for msg in doc['components']['messages'].values():
>     for example in msg.get('examples', []):
>         Draft202012Validator(msg['payload'], resolver=resolver).validate(example['payload'])
>         count += 1
> print(f'YAML parsed; {count} message examples conform to their schemas; only receive operations are present.')
> '@ | python -`
> - Ran `@'
> import json, sys, urllib.request
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> from jsonschema import Draft7Validator, RefResolver
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     doc = yaml.safe_load(f)
> with urllib.request.urlopen('https://raw.githubusercontent.com/asyncapi/spec-json-schemas/master/schemas/3.0.0.json', timeout=20) as response:
>     schema = json.load(response)
> store = dict(schema.get('definitions', {}))
> for value in list(store.values()):
>     if isinstance(value, dict) and '$id' in value:
>         store[value['$id']] = value
> resolver = RefResolver.from_schema(schema, store=store)
> errors = list(Draft7Validator(schema, resolver=resolver).iter_errors(doc))
> if errors:
>     for error in errors[:20]:
>         print('/'.join(map(str, error.absolute_path)), error.message)
>     sys.exit(1)
> print('Valid against official AsyncAPI 3.0.0 JSON Schema.')
> '@ | python -`
>
> </details>
</details>

Оновив [asyncapi.yaml](./asyncapi.yaml) лише за вказаними пунктами: додав описи черг і відкритих питань, уточнив джерело `correlationId` та ключ дедуплікації DLQ-алерту, конкретизував значення `version` і замінив `orderId` у прикладі `ReservationFailed`.

Перевірка пройшла: YAML розбирається, усі 9 прикладів відповідають схемам повідомлень, документ валідний за офіційною JSON Schema AsyncAPI 3.0.0.

> Моя пропозиція для RabbitMQ у Notifications. Exchange kavohub.events. Черги: notifications.events (основна), notifications.events.retry (затримка через TTL, потім назад в основну), notifications.events.dlq (після 5 невдалих спроб). Споживач ідемпотентний: в одній транзакції БД пишемо (source, event_id) у processed_events і бізнес-зміну, ACK лише після commit; дубль просто ACK. Тимчасова помилка йде в retry, нерозбірлива подія або порушення схеми одразу в DLQ. Повтори RabbitMQ (до 5) і спроби листа (3 по 5 хв) окремі. Рецензуй: що пропущено чи суперечить, особливо між commit і ACK, затримки, лічильник спроб, prefetch, порядок.

Загальна схема правильна, зокрема для випадку, коли процес упаде після commit, але до ACK. `processed_events` дає змогу безпечно обробити повторну доставку, якщо запис про подію та бізнес-зміну зафіксовані **в одній транзакції**.

### Між commit і ACK

Можливий сценарій: БД зробила commit, процес упав до ACK, RabbitMQ доставив повідомлення повторно. Споживач знаходить `(source, event_id)` у `processed_events`, не повторює зміну й ACK-ає повідомлення. Це правильна поведінка.

Але не надсилайте лист напряму в межах цієї обробки: відправка поштою не може бути атомарною з транзакцією БД. У тій самій транзакції зберігайте бізнес-зміну та запис у таблицю outbox/чергу повідомлень, а окремий worker надсилає лист. Так commit-before-ACK не губить намір надіслати лист. Збій після прийняття листа поштовим провайдером усе одно може призвести до дубля — це окремий ризик доставки.

### Retry і DLQ

- Задайте TTL retry-черги явно. У зразку ЛР3 це **5 секунд** для повторної обробки події; затримка листа **5 хвилин** належить окремому механізму доставки листа.
- Визначте лічильник як **п’ять загальних спроб обробки**, включно з першою. Після п’ятої тимчасової помилки — DLQ. Назва заголовка й момент інкременту мають бути сталими, наприклад `x-retry-count`.
- Не використовуйте `nack(requeue=true)` для тимчасових помилок: повідомлення може зациклитися без затримки й без коректного лічильника. Опублікуйте його в retry-чергу зі збільшеним лічильником і ACK-айте оригінал лише після підтвердження публікації.
- Якщо retry-публікація або DLQ-публікація не підтверджена, не ACK-айте оригінал: інакше повідомлення можна втратити. Застосуйте publisher confirms.
- Нерозбірливе повідомлення чи порушення схеми справді можна одразу спрямувати в DLQ. Додайте до запису DLQ первинне тіло, routing key, headers і причину відмови; обмежте доступ до чутливих даних.
- Уточніть, що означає TTL, якщо в черзі є повідомлення з різними затримками. Для однієї фіксованої затримки спільна TTL-черга прийнятна; для різних затримок краще розділити черги за інтервалами.

### Prefetch і порядок

Задайте обмежений `prefetch`, а не залишайте його необмеженим. Початкове значення на кшталт **10 повідомлень на consumer** можна перевірити з урахуванням тривалості обробника й кількості worker-ів. Це обмежить кількість непідтверджених повідомлень, якщо БД чи зовнішні залежності сповільняться.

RabbitMQ не гарантує бізнес-порядок, якщо повідомлення обробляють кілька consumer-ів або попереднє повідомлення пішло на retry. Для `orders` і резервів використовуйте вже визначені версії агрегатів: старішу версію ігнорувати, однакову версію з іншим вмістом відправляти в DLQ. Очікування резерву 15 хвилин зберігайте як таймер у БД/планувальнику й ACK-айте підтвердження після запису таймера — не тримайте RabbitMQ-повідомлення непідтвердженим усі 15 хвилин.

Для фінального опису лишається зафіксувати конкретний TTL retry, значення `prefetch` і формат лічильника спроб. DLQ-алерт має створювати власник DLQ та передавати його через узгоджений `dlq.alert` канал.

> Прийми рішення: TTL retry 5 с; лічильник x-retry-count, 5 загальних спроб включно з першою, інкремент при публікації в retry, після п'ятої невдачі DLQ; prefetch 10; retry і DLQ публікуємо з publisher confirms, оригінал ACK лише після підтвердження; у DLQ зберігаємо тіло, routing key, заголовки й причину, email маскуємо; лист іде через outbox і окремий worker; порядок між споживачами не гарантований, захист через version. Дай підсумкову таблицю: сценарій, дія, чергa, ACK/NACK. Сценарії: успіх, дубль, тимчасова помилка, 5-та невдача, невалідна схема, збій між commit і ACK, недоступний брокер при retry-публікації.

Зафіксував: retry TTL — 5 с, `x-retry-count` починається з `0`, публікації в retry/DLQ підтверджуються publisher confirms, `prefetch=10`. П’ята спроба — остання; після її невдачі повідомлення йде в DLQ. У DLQ зберігаються тіло, routing key, заголовки й причина, email маскується. Бізнес-зміна та запис в outbox фіксуються однією транзакцією; лист надсилає окремий worker.

| Сценарій | Дія | Черга | ACK/NACK оригіналу |
|---|---|---|---|
| Успіх | У транзакції зберегти `(source, event_id)`, бізнес-зміну й outbox-запис; після commit підтвердити повідомлення. | `notifications.events` | ACK лише після commit. Лист обробляє окремий worker. |
| Дубль | Знайти `(source, event_id)` у `processed_events`; повторну бізнес-зміну не виконувати. | `notifications.events` | ACK; нового outbox-запису не створювати. |
| Тимчасова помилка, спроби 1–4 | Опублікувати копію в retry з інкрементованим `x-retry-count`; чекати publisher confirm. Retry повертає повідомлення в основну чергу після TTL 5 с. | `notifications.events.retry` → `notifications.events` | ACK оригіналу лише після confirm публікації в retry. До confirm — не ACK. |
| Невдача п’ятої спроби | Після п’ятої невдалої обробки опублікувати повідомлення в DLQ з тілом, routing key, заголовками й причиною; email у збережених даних замаскувати. Дочекатися publisher confirm. | `notifications.events.dlq` | ACK оригіналу лише після confirm публікації в DLQ. |
| Невалідна схема | Не запускати бізнес-обробку; опублікувати повідомлення в DLQ з причиною валідації й повними діагностичними даними, з маскуванням email. | `notifications.events.dlq` | ACK лише після confirm DLQ-публікації. |
| Збій між commit і ACK | RabbitMQ доставляє подію повторно. Споживач знаходить її в `processed_events`, не повторює зміну чи outbox-запис. | `notifications.events` | Після повторної перевірки — ACK. |
| Брокер недоступний під час retry-публікації | Не підтверджувати оригінал і не вважати retry запланованим. Після відновлення з’єднання повторити публікацію; якщо канал закрився — дочекатися redelivery. | Оригінал лишається в `notifications.events`; retry ще не підтверджений | Не ACK-ати. Не NACK-ати з циклом негайного requeue. |
| Брокер недоступний під час DLQ-публікації | Так само не підтверджувати оригінал; повторити DLQ-публікацію після відновлення або обробити redelivery. | Оригінал лишається в основній черзі | Не ACK-ати до confirm DLQ-публікації. |

Для тимчасової помилки `x-retry-count=0` означає першу обробку. Після її невдачі retry-повідомлення публікується зі значенням `1`; після четвертої невдалої обробки — зі значенням `4`. Якщо невдалою була п’ята обробка, повідомлення переходить у DLQ без нової retry-публікації.

Повідомлення обробляються з `prefetch=10`. Порядок між споживачами не гарантується; старіші версії агрегатів не перезаписують новіші. **ACK після publisher confirm означає, що брокер підтвердив копію**, а не що її вже оброблено; повторна доставка можлива й надалі, тому дедуплікація залишається обов’язковою.

> Пропозиція для ADR «Недоступна пошта». Лист надсилає worker з outbox. Тайм-аут з'єднання 5 с, відправки 30 с. Тимчасова помилка (тайм-аут, SMTP 4xx): до 3 спроб через 5 хв, стан retry_wait. Постійна (SMTP 5xx, невалідна адреса): одразу final_failed. Після final_failed ІТ отримує лист (ключ final-failure + notificationId), далі admin робить resend вручну. Повтори RabbitMQ (до 5) і спроби листа (3) різні. Сценарій: пошта недоступна 10 хв. Рецензуй: що пропущено, чи вистачає 3 спроб, що з чергою після відновлення.

<details><summary>2 previous messages</summary>

> Зіставлю ADR з уже прийнятим outbox-процесом і перевірю семантику SMTP-кодів, а тоді відзначу ризики трьох спроб та поведінку черги під час відновлення пошти.
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `site:rfc-editor.org RFC 5321 4yz transient negative completion 5yz permanent negative completion retry mail`
>
> </details>
</details>

Схема узгоджується з outbox: RabbitMQ-повідомлення обробляється й ACK-ується після транзакції БД, а повтори листа виконує окремий worker. Тому недоступність пошти не повинна повертати подію до RabbitMQ.

**Трьох спроб може вистачити для короткого збою, але це не гарантія доставки після 10-хвилинної недоступності.** Якщо це три спроби загалом із паузами 5 хвилин після невдалих спроб, наступні спроби будуть приблизно на 5-й і 10-й хвилині, плюс час тайм-аутів. Уточніть, чи пауза рахується від завершення спроби. Для 10-хвилинного збою третя спроба може вже потрапити на відновлення пошти; якщо й вона завершиться невдачею, повідомлення перейде у `final_failed`. Для важливих листів розгляньте довше вікно повторів, backoff із jitter та межу максимальної затримки листа.

### Що уточнити в ADR

- **Класифікувати SMTP 5xx за причиною.** Невалідна адреса — постійна помилка для цього отримувача. Але помилка автентифікації чи конфігурації поштового сервісу може зачепити багато листів і потребує системної реакції, а не масового `final_failed`. RFC розрізняє 4yz як тимчасові відмови, які слід повторити, і 5yz як постійні відмови, для яких не слід повторювати той самий запит без змін. [RFC 5321](https://www.rfc-editor.org/rfc/rfc5321.html)
- **Визначити момент старту затримки** — після завершення невдалої спроби чи від запланованого часу. Додайте невеликий jitter, щоб після відновлення worker не намагався надіслати весь накопичений outbox одночасно.
- **Визначити поведінку для застарілих листів.** Наприклад, лист із ТТН, який надсилається через кілька годин після відвантаження, може вже втратити цінність. Потрібні строк актуальності та дія після його спливу.
- **Продумати ІТ-алерт під час збою пошти.** Якщо той самий SMTP недоступний, ІТ-лист про `final_failed` теж застрягне в outbox. Не запускайте нескінченний ланцюжок алертів про невдачу самого алерту; передбачте метрику або інший канал ескалації.
- **Уточнити дедуплікацію після ручного resend.** Ключ `final-failure + notificationId` дає лише один ІТ-алерт за весь час життя сповіщення. Якщо повторний admin-цикл теж завершиться невдачею, нового листа ІТ не буде. Уточніть, чи це бажано, чи ключ має враховувати `deliveryCycle`.

### Пошта після відновлення

Outbox-записи мають залишатися з `retry_wait` і `nextAttemptAt`. Після відновлення пошти worker обробляє ті записи, для яких настав час наступної спроби; він не повинен скидати лічильники чи миттєво повторювати всі листи. Для системного збою корисні circuit breaker, обмеження швидкості відправлення й метрика віку найстарішого запису.

Зафіксуйте в ADR, що три спроби — це **три загалом**, включно з першою. Це відповідає попередньому правилу для листів і залишається окремим від п’яти спроб доставки події через RabbitMQ.

> Прийми: 3 спроби загалом, ризик 10-хв збою в наслідки, circuit breaker і строк актуальності листів у відкриті питання. SMTP 5xx: невалідна адреса — permanent_failure; помилка автентифікації/конфігурації — спроба не рахується, пауза worker, одне ІТ-сповіщення. Пауза від завершення спроби + jitter ±10 %. ІТ-алерт дублюється метрикою й логом. Ключ: final-failure + notificationId + deliveryCycle. Оформи ADR-0005 (контекст, рішення, альтернативи, наслідки, статус «Запропоновано»).

<details><summary>2 previous messages</summary>

> Оформлю ADR-0005 у теці результатів вашого модуля. Збережу статус «Запропоновано», внесу узгоджені правила спроб і класифікації SMTP-помилок, а circuit breaker та строк актуальності листів залишу відкритими питаннями.
>
> <details><summary>Created a file</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/ADR-0005-email-unavailable-notifications.md` (+96 -0)
>
> </details>
</details>

Створив [ADR-0005 «Недоступна пошта»](./corporate-software-systems-lessons/result/yehoian-valentyn/ADR-0005-email-unavailable-notifications.md) зі статусом **«Запропоновано»**.

Зафіксував три спроби загалом, паузу 5 хвилин після завершення спроби з jitter ±10 %, розрізнення адресних і системних SMTP-помилок, паузу worker для помилок автентифікації/конфігурації та ключ ІТ-алерту `final-failure + notificationId + deliveryCycle`. Circuit breaker і строк актуальності листів залишив відкритими питаннями.

> Допиши в ADR-0005: 1) таблицю сценарію «пошта недоступна 10 хв» (час спроби, результат, стан); 2) для системної помилки auth/config основний канал — метрика й лог, ІТ-лист за можливості; 3) якщо не вдалося надіслати сам it_final_failure, він стає final_failed без нового алерту. Решту не змінюй.

<details><summary>2 previous messages</summary>

> Додам до ADR-0005 лише три запитані уточнення: часовий сценарій 10-хвилинного збою, резервний канал для системних помилок і правило, що невдалий `it_final_failure` не породжує нового алерту.
>
> <details><summary>Edited a file, ran a command</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/ADR-0005-email-unavailable-notifications.md` (+14 -1)
> - Ran `rg -n -C 2 'основний канал сповіщення|it_final_failure|Сценарій: пошта недоступна|Час спроби|T\+09:00' corporate-software-systems-lessons\result\yehoian-valentyn\ADR-0005-email-unavailable-notifications.md`
>
> </details>
</details>

Доповнив [ADR-0005](./corporate-software-systems-lessons/result/yehoian-valentyn/ADR-0005-email-unavailable-notifications.md): додав таблицю 10-хвилинного збою, уточнив метрику й лог як основний канал для системної помилки та зафіксував, що невдале `it_final_failure` не створює нового алерту. Решту документа не змінював.

> Мої сценарії стійкості Notifications у форматі Given/When/Then: 1) пошта недоступна 10 хв; 2) Odoo недоступний, PDF рахунку немає; 3) RabbitMQ недоступний під час retry-публікації; 4) БД недоступна під час обробки події; 5) дубль події; 6) рахунок раніше відвантаження; 7) збій після commit до ACK. Для кожного: очікувана поведінка, стан сповіщення, що бачить користувач. Рецензуй: що пропущено чи суперечить ЛР2 та ADR-0005.

<details><summary>2 previous messages</summary>

> Зіставлю сценарії з правилами outbox/RabbitMQ та станами сповіщень у ЛР2 і ADR-0005. Особливо перевірю межу між повтором читання PDF з Odoo та повтором SMTP-відправлення, бо для них можуть діяти різні ліміти.
>
> <details><summary>Ran 4 commands</summary>
>
> - Ran `rg -n -C 4 'retry_wait|waiting|final_failed|Odoo|PDF|5 спроб|3 спроб|deliveryCycle|outbox|processed_events|період|відвантаж' corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md corporate-software-systems-lessons\result\yehoian-valentyn\ADR-0004-buy-vs-build-notifications-reporting.md corporate-software-systems-lessons\result\yehoian-valentyn\ADR-0005-email-unavailable-notifications.md`
> - Ran `Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 321 -First 55; Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 420 -First 28`
> - Ran `rg -n 'Odoo|pdfRef|PDF|5 тимчас|5 спроб|адаптер|invoice_after_dispatch|рахунок.*відвантаж|24 год|тайм-аут' corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -First 45`
> - Ran `Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 375 -First 18; Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 390 -First 20; Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 95 -First 20`
>
> </details>
</details>

Сценарії охоплюють ключові збої й узгоджуються з основною схемою `processed_events` + транзакційний outbox. Для формулювань G/W/T додайте очікувані стани та видимий результат, як у таблиці нижче.

| Сценарій | Очікувана поведінка | Стан сповіщення | Що бачить користувач |
|---|---|---|---|
| **Пошта недоступна 10 хв** | Worker робить до 3 спроб загалом; затримка між ними — 5 хв після завершення попередньої, jitter ±10%. Якщо третя спроба припадає після відновлення — лист надсилається; якщо вона теж невдала — `final_failed`, ІТ-алерт для цього `deliveryCycle`. | `retry_wait` між спробами; далі `sent` або `final_failed`. | Клієнт не отримує листа, доки SMTP не прийме його. У журналі доступний статус; після остаточної невдачі — ІТ-алерт. Якщо SMTP підтвердив прийняття, але відповідь загубилася, можливий дублікат. Це прийнятий ризик ADR-0005. |
| **Odoo недоступний, PDF рахунку відсутній** | Не надсилати рахунок без PDF. Повторювати отримання PDF за окремою політикою; після її вичерпання — `final_failed` та ІТ-алерт. Не рахувати спроби читання PDF як SMTP-спроби. | До готовності PDF — стан очікування/повтору; після успішного отримання PDF — звичайний цикл доставки; після вичерпання повторів — `final_failed`. | Клієнт не отримує лист із неповним вкладенням. Оператор бачить, що PDF недоступний, а не помилку відправлення пошти. |
| **RabbitMQ недоступний під час retry-публікації** | Не ACK-ати оригінал, поки retry-публікація не підтверджена publisher confirm. Після відновлення брокера повторити публікацію; якщо з’єднання втрачено — очікувати redelivery. | Якщо обробку не зафіксовано — нового сповіщення ще немає; наявний стан не змінюється. | Листа поки немає. Подія лишається в обробці/черзі, а не губиться. |
| **БД недоступна під час обробки події** | Транзакція не commit-иться, отже споживач не ACK-ає подію. Якщо RabbitMQ доступний, можна опублікувати її в retry й ACK-нути оригінал лише після confirm. Після п’ятої загальної невдачі — DLQ. | Сповіщення не створене, якщо транзакція відкотилась. Якщо запис уже існував, він зберігає останній commit-нутий стан. | Листа немає, доки БД не відновиться й подію не буде оброблено. |
| **Дубль події** | Перевірити `(source, event_id)` у `processed_events`; повторно не змінювати бізнес-дані й не створювати outbox-запис. ACK-нути дубль. | Без змін. | Одне логічне сповіщення. Фізичний дубль листа все ще можливий, якщо провайдер прийняв попередній лист, але підтвердження загубилося. |
| **Рахунок надійшов раніше за відвантаження** | Зберегти рахунок в очікуванні за `orderId`. Не надсилати лист до `inventory.shipment.dispatched`; після відвантаження отримати PDF і розпочати доставку. Якщо відвантаження не надійшло за 24 години — `suppressed`, записати тайм-аут у журнал. | `waiting` до події відвантаження; потім `pending`/цикл доставки. Після 24 год без умови — `suppressed`. | Клієнт не отримує передчасного рахунку. Після відвантаження отримує лист із PDF; після тайм-ауту листа немає. |
| **Збій після commit, але до ACK** | RabbitMQ повторно доставляє подію. `processed_events` уже містить її ID, тому consumer не повторює зміну чи outbox-запис і ACK-ає повідомлення. Outbox-worker продовжує окрему доставку. | Залишається останній стан, зафіксований у БД; дубль події не створює нового циклу. | Звичайний результат обробки; повторної логічної нотифікації немає. Ризик SMTP-дубля залишається окремим і не вирішується ACK-ом RabbitMQ. |

### Що виправити або уточнити

1. **Odoo, PDF і SMTP мають окремі лічильники.** У ЛР2 для тимчасових read-помилок адаптера Odoo зазначено до п’яти спроб; у попередньому рішенні для недоступного PDF — три повтори через 5 хв, а в ADR-0005 — три спроби SMTP загалом. Зафіксуйте, чи політика PDF є винятком/зміною ЛР2, і що її лічильник не змінює `DeliveryAttempt` SMTP.
2. **Уточніть значення «три повтори» для PDF.** Це три повтори після першого запиту чи три спроби загалом? Для SMTP ADR-0005 чітко задає три спроби загалом.
3. **У сценаріях 3–4 визначте межу п’яти спроб.** За retry TTL 5 с п’ять загальних спроб завершуються швидко. Тривала недоступність БД/брокера може довести повідомлення до DLQ ще до відновлення; потрібен опис ручного requeue/відновлення.
4. **Для сценарію 6 вкажіть умову 24-годинного тайм-ауту.** За ЛР2 рахунок, що чекає відвантаження, після 24 год переходить у `suppressed` і лише журналює тайм-аут; не плутайте це з повтором отримання PDF після фактичного відвантаження.
5. **ІТ-сповіщення також залежить від пошти.** Якщо SMTP недоступний, користуйтеся узгодженим для системних проблем основним каналом — метрикою й логом; ІТ-лист надсилайте за можливості. Невдалий `it_final_failure` переходить у `final_failed` без нового алерту, як визначено в ADR-0005.
6. **Розділіть видимість клієнта й оператора.** Клієнт бачить факт отримання листа або його відсутність; внутрішній користувач бачить стан у журналі, причину й спроби. Email у журналі маскується, а вміст листа не показується навіть admin.

Отже, головна невизначеність — **повторне отримання PDF з Odoo**: його політика має бути відокремлена від SMTP-повторів і явно зіставлена з лімітом адаптера з ЛР2. Також сценарії з БД і брокером мають показувати, що відбувається, коли п’ять спроб вичерпалися раніше за відновлення залежності. [Звіт ЛР2](./corporate-software-systems-lessons/result/yehoian-valentyn/lab-02-notifications-reporting-template.md), [ADR-0005](./corporate-software-systems-lessons/result/yehoian-valentyn/ADR-0005-email-unavailable-notifications.md).

> Перепиши сценарії у форматі Given/When/Then, кожен з ID (RES-01…RES-07) і полями «Стан», «Бачить клієнт», «Бачить оператор». Врахуй: PDF з Odoo — 3 спроби загалом, лічильник окремий від SMTP, це виняток із ліміту адаптера ЛР2, записати у журнал рішень. У сценаріях БД і брокера: 5 спроб RabbitMQ вичерпуються за \~25 с, тоді DLQ, після відновлення ІТ робить ручний requeue, incidentCycle збільшується. 24-годинне очікування відвантаження → suppressed окремо від PDF. Метрика й лог — основний канал при мертвій пошті.

<details><summary>2 previous messages</summary>

> Оформлю сім сценаріїв у заданому форматі й внесу в журнал рішень окремий запис про ліміт отримання PDF з Odoo. Для брокерного збою окремо покажу межу: поки RabbitMQ не підтверджує публікацію, повідомлення не можна помістити в DLQ чи ACK-нути.
>
> <details><summary>Edited a file, ran a command</summary>
>
> - Ran `Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 752 -First 20`
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-02-notifications-reporting-template.md` (+1 -0)
>
> </details>
</details>

Нижче — сім сценаріїв у Given/When/Then. Для PDF стан `retry_wait` стосується повторів **отримання PDF**; його лічильник окремий від SMTP-спроб. Рішення про три спроби отримання PDF зафіксував як DEC-010 у [журналі рішень ЛР2](./corporate-software-systems-lessons/result/yehoian-valentyn/lab-02-notifications-reporting-template.md).

### RES-01 — Пошта недоступна 10 хвилин

- **Given:** сповіщення готове до надсилання, SMTP тимчасово недоступний.
- **When:** worker робить до трьох спроб загалом; між ними — 5 хвилин від завершення попередньої спроби з jitter ±10%.
- **Then:** якщо SMTP відновився до успішної спроби, лист надсилається; якщо всі три спроби невдалі — сповіщення переходить у `final_failed`, створюється ІТ-алерт із ключем `final-failure + notificationId + deliveryCycle`. Для мертвої пошти основними каналами ІТ є метрика й лог.
- **Стан:** `retry_wait` між спробами, потім `sent` або `final_failed`.
- **Бачить клієнт:** не бачить листа до успішного надсилання.
- **Бачить оператор:** стан і спроби в журналі; email отримувача замаскований. Для системної помилки автентифікації/конфігурації — метрика й лог, worker на паузі, спроби не витрачаються.

### RES-02 — Odoo недоступний, PDF рахунку немає

- **Given:** відвантаження вже підтверджене, але PDF рахунку з Odoo недоступний.
- **When:** адаптер Notifications робить три спроби отримати PDF загалом із п’ятихвилинним інтервалом; лічильник PDF не змінює лічильник SMTP.
- **Then:** якщо PDF отримано — починається окремий SMTP-цикл. Якщо всі три спроби отримання PDF невдалі — сповіщення переходить у `final_failed` та створюється ІТ-алерт; лист без PDF не надсилається.
- **Стан:** `retry_wait` під час повторів отримання PDF; потім `pending` перед SMTP-відправленням або `final_failed`.
- **Бачить клієнт:** не отримує листа без вкладення.
- **Бачить оператор:** причину недоступності PDF і окремий лічильник спроб Odoo; SMTP-спроби починаються лише після отримання PDF.

Цей ліміт у три спроби — виняток для PDF-потоку Notifications щодо загального ліміту адаптера Odoo в п’ять спроб. Щоб ліміти не змішалися, їх потрібно зберігати й показувати окремо.

### RES-03 — RabbitMQ недоступний під час retry-публікації

- **Given:** обробка події завершилася тимчасовою помилкою, а retry-публікація не отримала publisher confirm.
- **When:** RabbitMQ недоступний, а згодом відновлюється.
- **Then:** consumer не ACK-ає оригінал; після відновлення брокера повідомлення доставляється повторно. П’ять спроб і перехід у DLQ відбуваються лише якщо broker доступний для retry/DLQ-публікацій; після confirm DLQ-публікації оригінал ACK-ується. Після ручного requeue від ІТ збільшується `incidentCycle`.
- **Стан:** до commit нового сповіщення немає, попередній стан лишається без змін; якщо п’ять обробок вичерпано — подія в DLQ.
- **Бачить клієнт:** листа ще немає.
- **Бачить оператор:** спершу помилку/відсутність confirm; якщо після відновлення брокера подія дійшла до DLQ — алерт, причина та `incidentCycle`.

**Уточнення:** ~25 секунд — це орієнтовний час вичерпання п’яти спроб при доступному RabbitMQ та TTL 5 с. Під час повної недоступності брокера неможливо підтвердити ані retry-, ані DLQ-публікацію. Отже, таймер ~25 с не може гарантувати потрапляння в DLQ, поки брокер не відновився.

### RES-04 — БД недоступна під час обробки події

- **Given:** consumer отримав подію, але БД недоступна.
- **When:** транзакція не може зафіксувати `processed_events`, бізнес-зміну й outbox-запис.
- **Then:** транзакція відкочується; за доступного RabbitMQ подія проходить до п’яти загальних спроб із TTL 5 с і після п’ятої невдачі переходить у DLQ. Після відновлення БД ІТ вручну робить requeue, збільшуючи `incidentCycle`.
- **Стан:** нового сповіщення немає, оскільки commit не відбувся; якщо запис існував раніше, він зберігає останній commit-нутий стан.
- **Бачить клієнт:** листа немає до успішної обробки події та наступного надсилання.
- **Бачить оператор:** помилки БД у метриці/логу, а після DLQ — алерт і цикл інциденту. Якщо недоступність БД довша за ~25 секунд, подія може потрапити в DLQ до відновлення.

### RES-05 — Дублікат події

- **Given:** подія з тим самим `(source, event_id)` уже оброблена й зафіксована.
- **When:** consumer отримує її повторно.
- **Then:** бізнес-зміна й outbox-запис повторно не створюються; дубль ACK-ується. Ключ сповіщення додатково захищає від повтору за `triggerOccurrenceId + recipientId + notificationType`.
- **Стан:** стан наявного сповіщення не змінюється.
- **Бачить клієнт:** одне логічне сповіщення. Фізичний дубль усе ще можливий, якщо поштовий провайдер прийняв попередній лист, але підтвердження загубилося.
- **Бачить оператор:** наявний запис і спроби без нового сповіщення; повтор події можна зафіксувати в журналі/метриці.

### RES-06 — Рахунок надійшов раніше за відвантаження

- **Given:** `billing.invoice.issued` прийшла до `inventory.shipment.dispatched`.
- **When:** Notifications зберігає рахунок в очікуванні за `orderId`.
- **Then:** якщо відвантаження надійде в межах 24 годин, умова очікування виконується й починається отримання PDF; якщо ні — сповіщення переходить у `suppressed`, тайм-аут записується в журнал. Очікування відвантаження не витрачає ліміт спроб PDF.
- **Стан:** `waiting` до відвантаження, потім `pending`/`retry_wait` для PDF або `suppressed` після 24 годин.
- **Бачить клієнт:** не отримує листа до відвантаження; після нього отримує лист лише з доступним PDF. Якщо 24 години минули без відвантаження — листа немає.
- **Бачить оператор:** запис `waiting`; далі — стан отримання PDF або `suppressed` із причиною тайм-ауту.

### RES-07 — Збій після commit, але до ACK

- **Given:** транзакція вже зафіксувала подію в `processed_events`, бізнес-зміну й outbox-запис.
- **When:** consumer падає до ACK, а RabbitMQ доставляє ту саму подію повторно.
- **Then:** повторна обробка знаходить `(source, event_id)`, не створює другу бізнес-зміну/outbox-запис і ACK-ає дублікат. Outbox-worker продовжує доставку окремо.
- **Стан:** залишається останній стан, зафіксований у БД; він може перейти далі через outbox-worker.
- **Бачить клієнт:** одне логічне сповіщення після успішної доставки; невизначене підтвердження SMTP може спричинити фізичний дубль.
- **Бачить оператор:** один запис сповіщення й один outbox-запис; у журналі видно повторно доставлену подію без повторної бізнес-зміни.

### Що ще варто додати

Для повного набору бракує окремих сценаріїв **SMTP auth/config помилки** — worker на паузі, лічильник не змінюється, метрика/лог основні, ІТ-лист за можливості — та **невалідної email-адреси**, яка одразу дає `permanent_failure`. Також варто окремо перевірити втрату підтвердження після прийняття листа провайдером, бо саме тут залишається ризик фізичного дубля.

> Додай RES-08 (SMTP auth/config: пауза worker, спроби не витрачаються, метрика/лог основні), RES-09 (невалідна адреса → permanent_failure → final_failed без повторів), RES-10 (провайдер прийняв лист, підтвердження загубилося → transient_failure, можливий дубль). DEC-010 про 3 спроби PDF записуй у журнал ЛР3, а не ЛР2. RES-03: «Then» розбий на кроки 1–3.

<details><summary>2 previous messages</summary>

> Перенесу DEC-010 з журналу ЛР2 до журналу ЛР3 і зберу RES-01…RES-10 в окремому документі ЛР3. У RES-03 розіб’ю `Then` на три нумеровані кроки, як ви просите.
>
> <details><summary>Created a file, edited a file, ran a command</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-02-notifications-reporting-template.md` (+0 -1)
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+106 -0)
> - Ran `rg -n 'DEC-010|RES-0[1-9]|RES-10|^### RES-03|^  1\.' corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md`
>
> </details>
</details>

Додав RES-08, RES-09 і RES-10 до документа ЛР3. У RES-03 розбив `Then` на три кроки й уточнив, що ~25 секунд до DLQ — це орієнтир за доступного RabbitMQ.

Переніс DEC-010 з журналу ЛР2 до [журналу рішень ЛР3](./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md). RES-01…RES-10 зібрані в цьому ж документі.

> Мій прогін Lead-to-Cash для Notifications: deal.won → order.confirmed → stock.reserved → shipment.dispatched → invoice.issued → payment.received. Дай Mermaid sequence-діаграму та таблицю розбіжностей контрактів (поле, що очікує Notifications, що може дати інший модуль, ризик, рішення). Ключі зв'язку: dealId, orderId. Рецензуй, що пропущено.

<details><summary>2 previous messages</summary>

> Побудую прогін на чинних AsyncAPI-контрактах і внесу діаграму з таблицею розбіжностей у документ ЛР3. Перевірю окремо наскрізне зіставлення `dealId` → `orderId`, адресата клієнтського листа та джерело менеджера для оплат.
>
> <details><summary>Edited a file, ran a command</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+105 -0)
> - Ran `Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md | Select-Object -Skip 100 -First 125`
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+1 -8)
>
> </details>
</details>

Додав діаграму й таблицю розбіжностей до [документа ЛР3](./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md).

```mermaid
sequenceDiagram
    autonumber
    participant CRM
    participant MQ as RabbitMQ kavohub.events
    participant ORD as Orders
    participant INV as Inventory
    participant NP as Нова Пошта
    participant BILL as Billing
    participant ODOO as Odoo Community
    participant N as Notifications / Reporting
    participant W as Outbox worker
    participant SMTP as Поштовий провайдер
    participant PAY as Платіжний провайдер

    CRM->>MQ: crm.deal.won (dealId, accountId, ownerId, correlationId)
    MQ-->>ORD: crm.deal.won
    MQ-->>N: crm.deal.won
    N->>N: Дедуплікація, запис події та outbox для менеджера
    W->>SMTP: Лист менеджеру
    SMTP-->>W: Прийнято / помилка
    ORD->>ORD: Створити чернетку замовлення

    ORD->>MQ: orders.order.confirmed (orderId, version, correlationId)
    MQ-->>INV: orders.order.confirmed
    MQ-->>N: orders.order.confirmed
    N->>N: Запустити 15-хвилинне очікування резерву від event.time
    INV->>INV: Зарезервувати всі рядки
    INV->>MQ: inventory.stock.reserved (orderId, reservationId, version)
    MQ-->>N: inventory.stock.reserved
    N->>N: Перевірити версію резерву й умову замовлення
    N->>W: Outbox: підтвердження клієнтці
    W->>SMTP: Лист-підтвердження
    SMTP-->>W: Прийнято / помилка
    Note over INV,N: reservation_failed скасовує таймер без timeout-алерту; без резерву за 15 хв клієнтського листа немає, ІТ-алерт дедуплікується за orderId + version

    INV->>NP: Створити ТТН
    NP-->>INV: trackingNumber
    INV->>MQ: inventory.shipment.dispatched (orderId, shipmentId, trackingNumber)
    MQ-->>N: inventory.shipment.dispatched
    N->>N: Зберегти shipment і перевірити рахунок за orderId
    N->>W: Outbox: лист з ТТН
    W->>SMTP: Лист клієнтці з ТТН
    SMTP-->>W: Прийнято / помилка

    MQ-->>BILL: inventory.shipment.dispatched
    BILL->>ORD: GET даних замовлення
    ORD-->>BILL: Рядки й суми
    BILL->>ODOO: Створити й провести рахунок
    ODOO-->>BILL: invoiceId, invoiceNumber, PDF reference
    BILL->>MQ: billing.invoice.issued (orderId, invoiceId, invoiceNumber)
    MQ-->>N: billing.invoice.issued
    N->>N: Корелювати за orderId; отримати PDF, якщо його немає в події
    Note over N,ODOO: PDF — до 3 спроб загалом; окремий від SMTP лічильник
    N->>W: Outbox: лист із PDF після відвантаження
    W->>SMTP: Лист клієнтці з рахунком
    SMTP-->>W: Прийнято / помилка

    PAY->>BILL: Payment webhook
    BILL->>ODOO: Зареєструвати / звірити оплату
    ODOO-->>BILL: Стан оплати
    BILL->>MQ: billing.payment.received (orderId, invoiceId, paymentId, amount, currency)
    MQ-->>N: billing.payment.received
    N->>N: Оновити звіт і знайти менеджера в репліці замовлення
    N->>W: Outbox: лист менеджеру
    W->>SMTP: Повідомлення менеджеру
    SMTP-->>W: Прийнято / помилка
```

### Розбіжності контрактів

| Поле | Що очікує Notifications | Що може дати інший модуль | Ризик | Рішення |
|---|---|---|---|---|
| `dealId` ↔ `orderId` | Наскрізний зв’язок угоди й замовлення | CRM дає `dealId`, Orders дає `orderId`, але `orders.order.confirmed` не має `dealId` | Неможливо зіставити угоду із замовленням за подіями | Додати `dealId` до події Orders або погодити стабільний mapping |
| `accountId` / `recipientId` | Визначити клієнта й адресу для листів | CRM `deal.won` має `accountId`, але Orders-події не мають `accountId`/`recipientId`; Notifications не підписаний на `crm.account.upserted` | Немає гарантованого способу знайти адресу клієнта за `orderId` | Додати `accountId`/`recipientId` до Orders-контракту й погодити джерело email |
| `ownerId` | Адресат листів менеджеру, зокрема про оплату | `crm.deal.won` має `ownerId`, але Orders не передає його і немає зв’язку `dealId`–`orderId` | Fallback на директора може спрацьовувати зайво | Передавати `ownerId` у події Orders або вести погоджену репліку `orderId → ownerId` |
| `version` | Окремо відстежувати версії замовлення й резерву | Orders передає версію замовлення; Inventory — версію резерву | Порівняння версій різних агрегатів може перезаписати актуальний стан | Зберігати й перевіряти версії окремо; однакова версія з іншим payload — у DLQ |
| `shipmentId`, `trackingNumber` | Розрізняти відвантаження й сформувати лист із ТТН | Inventory передає обидва поля | Без стабільного `shipmentId` можливі дублікати для кількох відвантажень | Залишити `shipmentId` обов’язковим; для дедуплікації кількох відвантажень використовувати `UUIDv5(event.id:shipmentId)` |
| `orderId`, `invoiceId`, `pdfRef` | Знайти рахунок і отримати PDF; надсилати після відвантаження | Billing передає `orderId`, `invoiceId`, номер; `pdfRef` може бути відсутнім, PDF доступний через Odoo | Рахунок може прийти раніше за shipment або PDF може бути недоступний | Очікувати shipment до 24 годин; PDF отримувати окремим лічильником — до 3 спроб |
| `paymentId`, сума, валюта | Оновити звіт і повідомити менеджера | Billing передає ці поля та `orderId` | Без репліки замовлення може бракувати адресата; повтор платежу може дублювати ефект | Дедуплікувати подію та `paymentId`; менеджера шукати в репліці, квитанцію клієнтці не надсилати |
| `correlationId` | Простежити шлях від угоди до платежу | Усі події мають поле, але це не гарантує незмінного перенесення | Втрачається трасування бізнес-процесу | Погодити передачу початкового `correlationId` через усі модулі |

Найсуттєвіші прогалини — **відсутність зв’язку `dealId`–`orderId`** і **відсутність стабільного клієнтського адресата в Orders-подіях**. Також для оплати Notifications розраховує на репліку замовлення з `ownerId`, але поточний контракт не визначає, як ця репліка формується. Це варто погодити зі споживачами до фіксації контрактів.

> Прийми: Orders має додати в orders.order.confirmed поля dealId, accountId, ownerId (nullable); це вимога до Orders, записати у відкриті питання. Notifications веде репліку orderId → dealId, accountId, ownerId. Email клієнта береться за accountId з CRM/Odoo при відправленні (припущення). У діаграмі заміни «Odoo Community» на «Odoo». Онови asyncapi.yaml (OrderConfirmedData і приклад), розбіжності й журнал рішень ЛР3.

<details><summary>2 previous messages</summary>

> Оновлю `OrderConfirmedData` та приклад події, а в документі ЛР3 позначу додавання полів як вимогу до Orders, яку ще треба погодити. Також зафіксую локальну репліку Notifications і припущення про пошук email за `accountId`; у схемі діаграми зміню назву на «Odoo».
>
> <details><summary>Edited 2 files, ran a command</summary>
>
> - Wrote `./asyncapi.yaml` (+5 -2)
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+14 -6)
> - Ran `@'
> import sys
> sys.path.insert(0, r'C:\Users\yehoi\AppData\Local\Temp\kavohub-openapi-validation')
> import yaml
> with open('asyncapi.yaml', encoding='utf-8') as f:
>     doc = yaml.safe_load(f)
> data_schema = doc['components']['schemas']['OrderConfirmedData']
> example = doc['components']['messages']['OrderConfirmed']['examples'][0]['payload']['data']
> assert doc['asyncapi'] == '3.0.0'
> assert set(data_schema['required']) == {'orderId', 'dealId', 'accountId', 'ownerId', 'version'}
> assert {'orderId', 'dealId', 'accountId', 'ownerId', 'version'} <= set(example)
> assert example['ownerId'] is None or isinstance(example['ownerId'], str)
> print('YAML parses; OrderConfirmedData and its example contain all required fields, with nullable ownerId.')
> '@ | python -`
>
> </details>
</details>

Оновив [asyncapi.yaml](./asyncapi.yaml): `OrderConfirmedData` тепер вимагає `dealId`, `accountId` і `ownerId` (nullable); приклад події містить ці поля.

У [документі ЛР3](./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md) оновив діаграму (підписав систему як «Odoo»), таблицю розбіжностей, журнал рішень і відкриті питання. Додав вимогу до Orders передавати нові поля та рішення про репліку `orderId → dealId, accountId, ownerId`. Отримання email за `accountId` позначив припущенням, яке ще треба уточнити.

YAML розбирається; перевірив, що приклад `OrderConfirmed` містить усі обов’язкові поля й допускає `ownerId: null`.

> Збери підсумковий lab-03-notifications-reporting-yehoian.md: огляд артефактів (openapi.yaml, asyncapi.yaml, ADR-0005); RabbitMQ (черги, TTL 5 с, 5 спроб, prefetch 10, таблиця сценаріїв); RES-01…10; Lead-to-Cash; журнал усіх рішень; припущення; єдиний список відкритих питань; самоперевірка; ≥5 контрольних питань з відповідями. У рядку про dealId заміни «чинний AsyncAPI Orders має додати» на «Orders має додати до своєї події».

<details><summary>4 previous messages</summary>

> Зберу наявні сценарії, діаграму, таблицю контрактних розбіжностей і рішення в один підсумковий документ. Додам RabbitMQ-топологію, припущення, зведений список відкритих питань, самоперевірку та відповіді на контрольні питання; формулювання про `dealId` виправлю на вказане.
>
> <details><summary>Edited a file, ran 5 commands</summary>
>
> - Ran `Get-Content corporate-software-systems-lessons\lab-03.md | Select-Object -Skip 225 -First 125; Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md | Select-Object -Last 35; Get-Content asyncapi.yaml | Select-Object -First 30`
> - Ran `rg --files | rg '(^|/|\\)(asyncapi\.yaml|openapi\.yaml|ADR-0005|lab-03-notifications-reporting-yehoian\.md)$'; Get-Content corporate-software-systems-lessons\lab-03.md | Select-Object -Skip 344 -First 80`
> - Ran `Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-02-notifications-reporting-template.md | Select-Object -Skip 80 -First 42; Get-Content corporate-software-systems-lessons\lab-03.md | Select-Object -Skip 344 -First 18`
> - Ran `rg --files corporate-software-systems-lessons\result\yehoian-valentyn`
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+86 -0)
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+33 -5)
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+34 -0)
> - Ran `rg -n '^## |^### |RES-10|Orders має додати до своєї події|^\| DEC-|Q-INT-' corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md; rg --files corporate-software-systems-lessons\result\yehoian-valentyn`
>
> </details>
>
> Додаю ще дві частини, потрібні для повного звіту ЛР3: явну таблицю підписок і таблицю синхронних стиків із Odoo та поштою. Це зробить документ самодостатнім поруч із YAML-контрактами й ADR.
>
> <details><summary>Edited a file, ran 2 commands</summary>
>
> - Ran `Get-Content corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md | Select-Object -Skip 26 -First 35`
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+24 -1)
> - Ran `rg -n '^## |^### |Orders має додати до своєї події|^\| Q-INT-|^\| RES-' corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md; Get-Item corporate-software-systems-lessons\result\yehoian-valentyn\openapi.yaml, corporate-software-systems-lessons\result\yehoian-valentyn\asyncapi.yaml, corporate-software-systems-lessons\result\yehoian-valentyn\ADR-0005-email-unavailable-notifications.md | Select-Object Name,Length`
>
> </details>
</details>

Зібрав підсумковий [lab-03-notifications-reporting-yehoian.md](./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md).

Документ містить огляд [OpenAPI](./corporate-software-systems-lessons/result/yehoian-valentyn/openapi.yaml), [AsyncAPI](./corporate-software-systems-lessons/result/yehoian-valentyn/asyncapi.yaml) і [ADR-0005](./corporate-software-systems-lessons/result/yehoian-valentyn/ADR-0005-email-unavailable-notifications.md), RabbitMQ-топологію та сценарії, RES-01…10, Lead-to-Cash, журнал рішень, припущення, єдиний список відкритих питань, самоперевірку й 8 контрольних питань із відповідями.

У розбіжності про `dealId` використано формулювання «Orders має додати до своєї події». Самоперевірка позначає частковими ті пункти, що залежать від погодження Orders і уточнення джерела email та Odoo.

> Виправ у lab-03: 1) у Q-INT-08 заміни «Odoo Community збірці» на «інсталяції Odoo»; 2) у розділі RabbitMQ додай: алерт про власну notifications.events.dlq Notifications створює сам як внутрішнє ІТ-сповіщення без публікації на шину. Решту не змінюй.

<details><summary>2 previous messages</summary>

> Внесу лише дві правки в підсумковий lab-03: уточню формулювання Q-INT-08 і додам правило для власної DLQ Notifications.
>
> <details><summary>Edited a file, ran 3 commands</summary>
>
> - Ran `rg -n -C 2 'Q-INT-08|notifications.events.dlq|DLQ-алерт' corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md`
> - Ran `rg -n 'Q-INT-08|notifications.events.dlq' corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md`
> - Wrote `./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md` (+3 -1)
> - Ran ``rg -n 'власної `notifications.events.dlq`|Q-INT-08' corporate-software-systems-lessons\result\yehoian-valentyn\lab-03-notifications-reporting-yehoian.md``
>
> </details>
</details>

Вніс лише дві правки в [lab-03](./corporate-software-systems-lessons/result/yehoian-valentyn/lab-03-notifications-reporting-yehoian.md): оновив формулювання Q-INT-08 і додав, що для власної `notifications.events.dlq` Notifications створює внутрішнє ІТ-сповіщення без публікації на шину.
