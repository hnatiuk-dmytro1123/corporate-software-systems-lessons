# ЛР3. Інтеграція модулів — Orders

**Студент:** Василь Гордашевський. **Група:** не вказана у вихідних матеріалах. **Дата:** 2026-11-22  
**ШІ-асистент:** ChatGPT (GPT-6). **Діалог:** перед поданням додати посилання або експорт.

> Проєктний документ. Код не реалізовувався; контракти та сервіси автоматично не перевірялися.

## 0. Вихідні дані

Основа — кейс «КавоХаб», матеріали ЛР1 та [ЛР2](lab-02-orders-hordashevskyi.md). Orders володіє Quote, Order, цінами й погодженнями; Inventory — товарами, залишками та резервами; CRM — клієнтами й угодами; Billing — рахунками й оплатами, Odoo є бухгалтерською системою запису.

ЛР2 визначила окремі агрегати Quote й Order, незмінний ExchangeRateSnapshot, команди переходів станів, Inbox/Outbox та підтвердження лише після резервування. У цій роботі автомат уточнено для знижки понад 10%: Orders створює тимчасовий резерв на час погодження; подія inventory.stock.reserved фіксує його наявність, але Order лишається pending_approval. Після погодження іншою особою Orders переходить у confirmed і публікує orders.order.confirmed із тим самим reservationId; Inventory закріплює наявний резерв без дублювання.

Базовий каталог подій і конверт взято з README.md, 2.6; стики й прогін — з lab-03.md, розділи 5–6.

## 1. Контракти модуля

Повні специфікації: [orders.openapi.yaml](orders.openapi.yaml) та [orders.asyncapi.yaml](orders.asyncapi.yaml).

### 1.1 Загальні правила

- OpenAPI 3.1 описує HTTP API; AsyncAPI 3.0 — повідомлення й канали RabbitMQ.
- Запити захищені Bearer JWT від Keycloak. sales_manager працює зі своїми клієнтами; sales_head або делегований director погоджує знижку; svc-billing читає фінансовий snapshot. Автор замовлення не може його погодити.
- X-Correlation-Id необов’язковий у вхідному запиті (Orders генерує UUID, якщо відсутній), обов’язковий у відповідях і вихідних викликах.
- Усі POST-команди вимагають Idempotency-Key. PUT /orders/{id}/lines повністю замінює рядки, ідемпотентний за семантикою; version/If-Match захищає конкурентне редагування.
- Помилки мають Content-Type application/problem+json та RFC 9457 поля type, title, status, detail, instance. Додаткові поля currentState, allowedActions, errors дозволені.
- Той самий key і тіло повертають збережений результат; той самий key з іншим тілом — 422 idempotency-key-reuse; паралельна незавершена операція — 409 idempotency-in-progress. Тимчасовий 503 до створення доменного об’єкта не фіксується як завершений результат: key звільняється для повтору.
- Правила Spectral: шляхи kebab-case; operationId обов’язковий і унікальний; 4xx/5xx — Problem; Idempotency-Key обов’язковий для POST; X-Correlation-Id у відповідях і вихідних викликах.
- Prism може мокати HTTP API за OpenAPI. Він не емулює RabbitMQ: події перевіряються за AsyncAPI та фікстурами.
- Order ID має формат ORD-YYYY-NNNNNN. Гроші й курс — Decimal, у JSON як десяткові рядки; quantity — до 3 десяткових знаків, ціла для шт. Час — ISO 8601 UTC.
- Read API: GET /quotes/{id}, GET /orders, GET /orders/{id}, GET /prices, GET /health. Команди: POST /quotes, POST /orders, PUT /orders/{id}/lines, POST /orders/{id}/confirm, /approve-discount, /reject-discount, /withdraw, /cancel. Пряме редагування status заборонене.
- GET /orders/{id} для svc-billing повертає незмінний snapshot: orderId, accountId, валюту розрахунку UAH, ExchangeRateSnapshot (quoteDate, rateDate, currency, rate, напрям UAH за 1 EUR, source/observedAt), рядки з SKU, qty, unitPrice, currency, discount, VAT і сумами, totals. Billing бере ЄДРПОУ з повної CRM AccountReplica.
- Події Orders: orders.order.confirmed (orderId, accountId, warehouse, reservationId, lines[{sku,qty}], confirmedAt) і orders.order.cancelled (orderId, nullable reservationId, reason, cancelledAt). Обидві використовують конверт проєкту: specversion, id, type, source, time, correlationId, dataschema, data.
- SemVer для event contracts: необов’язкове поле — сумісна зміна; видалення/перейменування обов’язкового або несумісна зміна типу — нова major-версія.

### 1.2 Основні HTTP операції

| Метод і шлях | Призначення та відповіді | Доступ |
|---|---|---|
| GET /quotes/{id} | Деталі КП; 200, 404 | продажі, керівник |
| GET /orders | Фільтрований сторінковий список; 200 | продажі, керівник, accountant із обмеженням полів |
| GET /orders/{id} | Деталі або фінансовий snapshot для Billing; 200, 404 | продажі, керівник, accountant, svc-billing |
| GET /prices | Чинні базові/контрактні ціни; 200 | продажі, не warehouse_operator |
| GET /health | Стан без секретів; 200/503 | внутрішній health-check |
| POST /quotes | Створення КП; 201, без курсу EUR — 503 | sales_manager |
| POST /orders | Створення draft; 201, повторне quote — 409 | sales_manager |
| PUT /orders/{id}/lines | Повна заміна рядків у draft; 200, version conflict — 409 | автор/уповноважений |
| POST /orders/{id}/confirm | Запустити погодження/резерв; 202 | sales_manager |
| POST /orders/{id}/approve-discount | Погодити; 202/200, заборона автору — 403 | sales_head/delegated director |
| POST /orders/{id}/reject-discount | Відхилити; 202 | погоджувач, не автор |
| POST /orders/{id}/withdraw | Відкликати; 202 | автор до рішення |
| POST /orders/{id}/cancel | Скасувати до shipped; 202/200, після — 409 | продажі/керівник |

Усі POST мають Idempotency-Key; усі відповіді мають X-Correlation-Id. У OpenAPI-файлі наведено основні request/response schemas та security extensions. Детальні поля quote/order можуть уточнюватися без зміни прийнятих бізнес-правил.

## 2. Стики, підписки та ідемпотентність

### 2.1 Синхронні виклики

| Стик | Тайм-аут | Повтори | Поведінка за збоєм |
|---|---|---|---|
| Orders → Inventory GET /availability | connect 1 с, read 2 с | до 3 повторів, backoff від 200 мс із jitter для тимчасових помилок | Показати «невідомо»; підтвердження все одно чекає фактичного резерву |
| Orders → Inventory POST /reservations | connect 1 с, read 3 с | до 3 повторів на timeout/429/5xx; поважати Retry-After | Стабільний key; timeout — невідомий результат, лишити pending і повторити тим самим key |
| Orders → Inventory POST /reservations/{id}/release | connect 1 с, read 3 с | до 3 повторів на тимчасові помилки | Ідемпотентний release; release_pending і повтор з outbox |
| Orders → CRM GET /accounts/{id} | connect 1 с, read 2 с | до 3 повторів на тимчасові помилки | Лише за відсутньої репліки; без клієнтських даних створення блокується |
| Orders → НБУ exchange GET | connect 1 с, read 3 с | до 3 спроб із jitter для мережевих/5xx | Cache miss — fail-closed, 503 Problem; [] для майбутньої дати не є транспортною помилкою |
| Billing → Orders GET /orders/{id} | connect 1 с, read 3 с | GET за політикою Billing | svc-billing, X-Correlation-Id, незмінний snapshot |

Три повтори означають три повторні спроби після початкової. Backoff має верхню межу. POST повторюється лише з тим самим Idempotency-Key.

### 2.2 Підписки Orders

| Подія | Дія | Дублікат/порядок |
|---|---|---|
| crm.account.upserted | Оновити AccountReplica | version/updatedAt; застарілу версію ігнорувати |
| crm.deal.won | Створити чернетку за dealId | Унікальність dealId; data: dealId, accountId, salesManagerId required, quoteId optional |
| inventory.product.upserted | Оновити ProductReplica | version/updatedAt |
| inventory.stock.reserved | Позначити резерв активним | У pending_approval лишитися в цьому стані; пізній резерв після cancel компенсувати release |
| inventory.stock.reservation_failed | Повернути до draft або завершити узгоджене скасування | Не публікувати confirmed |
| inventory.shipment.dispatched | confirmed → shipped | Обов’язковий orderId |
| billing.invoice.issued | shipped → invoiced, зберегти invoice number | Рання подія — pending-events |
| billing.payment.received | Звірити суму/валюту; invoiced → paid | Ранню оплату зберегти до invoice |

Ранні події життєвого циклу зберігаються з причиною й correlationId та застосовуються після потрібного переходу. Версія агрегата перевіряється транзакційно.

### 2.3 Idempotency-Key на стороні Orders

Таблиця idempotency_records: scope, key, request_hash, state (processing/completed), response_status/body, created_at, expires_at, UNIQUE(scope,key). Ключ захоплюється атомарно. Той самий key/body відтворює відповідь; інше тіло — 422; паралельний запит — 409. Для асинхронної команди зберігається 202 і operation reference. Тимчасовий 503 до створення Quote не зберігається як завершений результат.

## 3. RabbitMQ, Inbox/Outbox і DLQ

Спільний topic exchange — kavohub.events; routing key — event type. Orders публікує confirmed/cancelled через transactional outbox. Quorum queue orders.events підписана на події з таблиці 2.2; orders.events.retry відтерміновує повтор (початково 5 с, далі backoff із jitter); після 5 тимчасових невдалих спроб подія йде в orders.events.dlq. Невиправні schema errors не зациклюються в retry.

processed_messages має UNIQUE(consumer_name,message_id), event_type та processed_at. У тій самій транзакції фіксуються бізнес-зміна та outbox/pending event. ACK надсилається після COMMIT. Падіння після COMMIT і до ACK дає повторну доставку; Inbox робить її no-op. Якщо транзакція не закомітилася — подія не ACK-иться.

~~~text
BEGIN
  INSERT processed_messages(consumer_name,message_id,event_type)
    ON CONFLICT DO NOTHING
  if duplicate: COMMIT; ACK; return
  validate envelope and payload
  lock Order; validate version and transition
  if event is early: persist pending_event
  else apply domain change
  write outgoing event to outbox when needed
COMMIT
ACK
~~~

**DLQ-процедура:** черговий інженер отримує alert, аналізує routing key, message id, correlationId, redacted payload і причину; виправляє конфігурацію/дані/код; уповноважена особа повторно публікує з audit записом. Poison messages автоматично не replay-яться. Секрети й персональні дані маскуються.

## 4. Тестові сценарії стійкості

1. **Дублікат резерву.** Given pending_reservation; When двічі приходить inventory.stock.reserved із тим самим id; Then перехід/outbox один раз, дублікат ACK-иться після Inbox-перевірки.
2. **Poison message.** Given подія з невалідною схемою; When consumer валідовує її; Then Order не змінюється, повідомлення не зациклюється й потрапляє в DLQ з alert.
3. **HTTP retry.** Given POST confirm із key K і тілом H; When повторити K/H; Then повертається збережена відповідь без другого резерву; K з іншим тілом дає 422.
4. **Crash між COMMIT і ACK.** Given COMMIT бізнес-зміни/processed_messages/outbox; When процес падає до ACK; Then RabbitMQ доставляє повторно, consumer робить no-op і ACK.
5. **Подія не за порядком.** Given Order shipped; When payment приходить раніше invoice; Then подія чекає; після invoice застосовується і стан стає paid.
6. **Відмова НБУ.** Given дата 2026-11-23 без кешу; When НБУ недоступний після спроб; Then 503 Problem, Quote не створено, key звільнений; повтор після відновлення безпечний.
7. **Race cancel/reserve.** Given pending_reservation; When cancel конкурує з reservation event; Then optimistic version дозволяє узгоджений перехід; пізній резерв скасованого замовлення знімається через outbox.

## 5. Зовнішня інтеграція: НБУ та ADR

Запит: GET https://bank.gov.ua/NBUStatService/v1/statdirectory/exchange?valcode=EUR&date=YYYYMMDD&json.

| Поле Orders | Поле НБУ | Правило |
|---|---|---|
| currency | cc | Перевірити EUR |
| rate | rate | Decimal, UAH за 1 EUR |
| rateDate | exchangedate | Фактична дата відповіді |
| quoteDate | query date КП | Зберігати окремо |
| source/observedAt | endpoint/час отримання | Аудит |

На вихідні та свята діє курс останнього робочого дня; quoteDate може відрізнятися від rateDate. На майбутню дату API повертає []; курс на наступний день публікується приблизно о 15:30–16:00 за фактами методички, не як SLA.

~~~mermaid
sequenceDiagram
  actor M as Менеджер
  participant O as Orders
  participant C as Cache/DB
  participant N as API НБУ
  M->>O: POST /quotes + key + correlation
  O->>C: Пошук snapshot за датою КП
  alt cache hit
    C-->>O: ExchangeRateSnapshot
  else cache miss
    O->>N: GET EUR/date (connect 1s, read 3s)
    N-->>O: rate, cc, exchangedate або []
    O->>O: Валідація та Decimal
    O->>C: Кеш і незмінний snapshot
  end
  alt Валідний snapshot
    O-->>M: 201 Quote
  else Недоступність або []
    O-->>M: 503 Problem; Quote не створено; key звільнено
  end
~~~

### ADR-ORD-03 — fail-closed для EUR

**Статус:** прийнято. При cache miss Orders робить запит із тайм-аутами 1/3 с і до трьох повторів з jitter. Перевіряє cc=EUR, Decimal і exchangedate. Без придатного snapshot повертає 503 Problem і не створює КП. Snapshot після фіксації незмінний.

| Альтернатива | Перевага | Ризик |
|---|---|---|
| Fail-closed — обрано | Точність та аудитовність | Тимчасово блокує EUR-КП |
| Курс попереднього робочого дня | Вища доступність | Може порушити правило курсу на дату КП |
| Ручний курс sales_head | Не залежить від API | Помилка/зловживання; потрібні джерело, погодження й аудит |

~~~mermaid
sequenceDiagram
  participant CRM
  participant O as Orders
  participant N as НБУ
  participant I as Inventory
  participant NP as Нова Пошта
  participant B as Billing
  participant OD as Odoo
  participant P as Платіжний провайдер
  CRM-->>O: crm.deal.won
  O->>N: GET EUR rate (за потреби)
  N-->>O: rate 43.2500, exchangedate 2026-11-20
  O->>I: POST /reservations (temporary, TTL 24h, key)
  I-->>O: inventory.stock.reserved
  Note over O: pending_approval; резерв активний
  O->>O: sales_head approve; автор не погоджує себе
  O-->>I: orders.order.confirmed з reservationId
  Note over I: існуючий резерв закріплюється
  I->>NP: InternetDocument/save
  NP-->>I: HTTP 200; перевірити success=true; ТТН
  I-->>O: inventory.shipment.dispatched(orderId, shipmentId, ttnNumber, dispatchedAt)
  I-->>B: inventory.shipment.dispatched
  B->>O: GET /orders/{id} (svc-billing)
  O-->>B: незмінний UAH snapshot
  B->>OD: створити рахунок, action_post
  OD-->>B: номер рахунку та Odoo ID
  B-->>O: billing.invoice.issued
  P->>B: webhook success + HMAC-SHA256
  B-->>P: 202 Accepted до 10 с
  B-->>O: billing.payment.received
~~~

## 6. Інтеграційний прогін

Усі події одного сценарію мають спільний correlationId; кожне повідомлення має власний id.

| Крок | Розбіжність | Рішення |
|---|---|---|
| CRM deal.won | Базовий каталог не визначав data | CRM погодила dealId, accountId, salesManagerId required; quoteId optional; рядки додаються в Orders |
| Резерв під discount | Невідомо, чи stock.reserved — тимчасовий резерв і як його закріпити | Подія фіксує активний тимчасовий резерв; Orders лишається pending_approval; після approve публікує confirmed; Inventory закріплює той самий id |
| Shipment | Не було orderId у payload | Inventory погодила orderId (ORD-YYYY-NNNNNN), shipmentId, ttnNumber, dispatchedAt UTC |
| Invoice | Payload не визначено | Billing погодила orderId, invoiceId, invoiceNumber, currency, totalAmount, issuedAt required; odooMoveId optional |
| Payment | Payload не визначено | Billing погодила orderId, invoiceId, paymentId, amount, currency, receivedAt required; providerEventId optional |
| Дублікат reservation | Доставка того самого message id після переходу Order у paid | Inbox відкидає дублікат; стан і outbox не змінюються |
| НБУ cache miss | Для дати успішного сценарію курс уже закешований | Тест використовує 2026-11-23; 503 не завершує Idempotency-Key |

Billing бере ЄДРПОУ з повної CRM replica й snapshot з Orders. Odoo створює/оновлює контрагента за ЄДРПОУ; номер рахунку призначається під час action_post. Платіжний webhook перевіряє HMAC-SHA256 і дедуплікується за ID події провайдера; Billing відповідає 2xx до 10 с і обробляє подію після відповіді.

Нова Пошта завжди відповідає HTTP 200; бізнес-успіх визначає поле success. Повтор InternetDocument/save може створити другу ТТН, тому потрібні локальний operation record і reconciliation, без сліпого повтору після невизначеного timeout. Оновлення статусу доставки може затриматися до 15 хвилин.

## Журнал проєктних рішень

| № | Рішення | Альтернативи | Обґрунтування |
|---|---|---|---|
| ADR-03-01 | OpenAPI 3.1 та AsyncAPI 3.0 | Неформальний опис | Контракти придатні для mock і ранньої інтеграції |
| ADR-03-02 | RFC 9457, correlation ID, idempotency для POST | Неуніфіковані помилки/повтори | Трасування та безпечні повтори |
| ADR-03-03 | SemVer для event contracts | Нескоординовані зміни | Передбачувана сумісність |
| ADR-03-04 | orders.order.confirmed закріплює наявний тимчасовий резерв | Окремий endpoint/подія-відповідь | Зберігає базовий контракт Inventory; уточнює автомат ЛР2 |
| ADR-03-05 | Fail-closed для НБУ | Попередній кеш/ручний курс | Точність та відтворюваність фінансового snapshot |
| ADR-03-06 | Inbox/Outbox, ACK після COMMIT | ACK до транзакції | Надійність без втрати й подвійного застосування |

## Уточнення та припущення

| № | Уточнення | Джерело | Вплив |
|---|---|---|---|
| 1 | Група та посилання/експорт чату ще не додані | Вихідні матеріали | Титул і подання |
| 2 | crm.deal.won: dealId/accountId/salesManagerId required; quoteId optional | Погоджено CRM | Авторство й створення draft |
| 3 | stock.reserved означає тимчасовий резерв до погодження | Погоджено Inventory | Автомат станів |
| 4 | AccountReplica Billing має повний стан та ЄДРПОУ | Базовий контракт CRM | Мапінг Odoo |
| 5 | Orders GET надає податковий snapshot, підсумок у UAH | Погоджено Billing | Створення рахунку |
| 6 | 503 НБУ звільняє key, якщо Quote не створено | Узгоджено у прогоні | Безпечний повтор |
| 7 | Payload інших модулів оновлюється за погодженими версіями | Погоджено в діалозі | Синхронізація контрактів |

## Відкриті питання та ризики

- Додати групу студента та посилання/експорт цього діалогу.
- YAML контракти потребують запуску Spectral/валідації перед здачею; у цій проєктній роботі автоматична перевірка не виконувалась.
- Відобразити погоджені зміни payload в специфікаціях CRM, Inventory і Billing.
- Уточнити пагінацію/фільтри GET /orders, точні Problem type URI, operation status для 202 та backoff cap за NFR.
- Узгодити RabbitMQ bindings, DLX/TTL, retention і quorum queue arguments із платформною командою.
- Перевірити права сервісних акаунтів і податкові поля CRM replica.
- Odoo, Keycloak, RabbitMQ та партнерські середовища не перевірялися.

## Самоперевірка

| Критерій | Статус | Розділ |
|---|---|---|
| OpenAPI й AsyncAPI — 2 | YAML sidecars підготовлено; потрібна зовнішня lint/валідація | 1 |
| Синхронна взаємодія — 1,5 | Тайм-аути, retries, idempotency, correlation і збої описані; backoff cap відкритий | 2 |
| Події та стійкість — 2 | Підписки, Inbox/Outbox, ACK, retry/DLQ, порядок і сценарії описані | 2–4 |
| Зовнішня інтеграція — 1,5 | Mapping, sequence і ADR з трьома альтернативами наведені | 5 |
| Інтеграційний прогін — 1 | Happy path завершено; розбіжності вирішені; два збої змодельовані | 6 |

## Відповіді на контрольні питання

1. **Навіщо контракт до коду?** Він узгоджує поля, типи, помилки й доступ до реалізації; споживач працює з mock до готовності провайдера.
2. **OpenAPI й AsyncAPI?** OpenAPI описує HTTP request/response. AsyncAPI описує повідомлення, канали та операції send/receive. Канал — місце/адреса обміну; операція — дія застосунку через канал. Надійність забезпечує архітектура, а не сама специфікація.
3. **Чому ACK після COMMIT?** ACK до COMMIT може видалити подію до збереження зміни. COMMIT перед ACK дає безпечну повторну доставку з дедуплікацією.
4. **Той самий key з іншим тілом?** Зіставляємо хеш, повертаємо 422 Problem типу idempotency-key-reuse і не виконуємо команду.
5. **Події не за порядком?** Старі репліки ігноруються за версією; передчасні lifecycle-події буферизуються. Payment до invoice чекає, а потім переводить Order у paid; пізній резерв після cancel компенсується.
6. **НБУ недоступний?** Без придатного кешу fail-closed повертає 503, не створює Quote й звільняє key для повтору. ADR фіксує компроміс точності та доступності.

## Робота з ШІ-асистентом

1. Виявили відсутню схему crm.deal.won і погодили мінімальні поля з CRM замість припущень.
2. Не додавали Inventory endpoint підтвердження резерву; уточнили використання orders.order.confirmed для закріплення існуючого reservation.
3. Додали orderId до inventory.shipment.dispatched для пошуку замовлення Billing.
4. Виявили відсутні payload invoice/payment events і погодили їх зі споживачем Orders.
5. Уточнили різницю quoteDate/rateDate, cache miss для тесту НБУ та повтор ключа після 503.

## Перед поданням

Заповнити навчальну групу та додати посилання на діалог або його експорт.
