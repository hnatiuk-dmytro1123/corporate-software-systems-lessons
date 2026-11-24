# Лабораторна робота 3 — Billing

> Проєктний документ ЛР3 для модуля Billing. Події для Orders і snapshot endpoint іншого модуля позначені як контрактні пропозиції/залежності, а не як API Billing.

Студент: Голубєв Богдан Володимирович, група М-26. Дата: 2026-11-23.  
ШІ-асистент: Codex, GPT-6. Діалог: додати посилання або експорт перед здачею.

## 0. Вихідні дані й рішення з попередніх робіт

- Модуль: Billing, проєкт «КавоХаб».
- Рахунок створюється за фактично відвантаженою поставкою. Для початкової моделі — один рахунок на один `shipmentId`.
- `invoiceId` і `paymentId` — UUID Billing. `orderId`, `shipmentId`, `accountId` — непрозорі рядки, бо ними володіють інші модулі.
- Billing видає `billingInvoiceNumber`; Odoo після проведення видає окремий `odooJournalNumber`.
- `inventory.shipment.dispatched` обробляється через inbox за `eventId`; унікальний `shipmentId` не дозволяє виставити другий рахунок за ту саму поставку.
- Billing публікує через outbox. Споживачі мають приймати доставку щонайменше один раз і дедуплікувати за ID події.
- Нерозподілений платіж не змінює залишок рахунку. Allocation підтверджує лише `accountant`.
- Для контрактів використано каталог подій і envelope з [README.md](../../README.md), API-таблицю та ролі з [lab-02-billing-golubiev.md](lab-02-billing-golubiev.md), а також ескізи API з [lab-02.md](lab-02.md). Попередні матеріали містили вихідні дані й таблиці; YAML-контракти створено в цій ЛР.

## 1. Контракти Billing

- [OpenAPI 3.1 — billing.openapi.yaml](billing.openapi.yaml) формалізує API-таблиці ЛР2 та додає читання рахунку/PDF, Odoo sync status і майбутній webhook як специфікацію без реалізації.
- [AsyncAPI 3.0 — billing.asyncapi.yaml](billing.asyncapi.yaml) формалізує envelope README та публікації Billing через RabbitMQ.
- `billing.invoice.payment_status_changed` і `billing.credit_note.issued` позначені як пропозиції, які має погодити Orders. `billing.invoice.issued` у цьому рішенні означає локальне issue у Billing; Odoo синхронізується асинхронно.
- Для подій зі станом рахунку використовується `invoiceVersion` — ціле число, монотонно зростаюче в межах одного рахунку. Orders зберігає останню застосовану версію на кожен `invoiceId` і відкидає версію `<=` збереженої. Пропуски версій Orders звіряє через власний `GET /orders/{orderId}/invoices`; цей endpoint належить Orders і не входить до OpenAPI Billing. `changedAt`/`issuedAt` — для аудиту, не для сортування.
- Версійний маркер для ціноутворення проходить від `orders.order.confirmed` через Inventory до `inventory.shipment.dispatched`: Billing порівнює переданий `orderVersion` зі `pricingVersion` у shipment-scoped знімку Orders. `pricingVersion` змінюється лише при зміні цін, знижки, курсу чи умов оплати; зміни інших полів замовлення не роблять snapshot застарілим для виставлення рахунку.
- Пропозиція Orders щодо агрегації: замовлення стає `paid`, лише коли всі його рахунки оплачені, усі рядки відвантажені та для них виставлені рахунки. Після кредит-ноти вже повністю оплаченого рахунку invoice лишається `paid`, а статус замовлення не змінюється; факт і сума кредит-ноти передаються окремою `billing.credit_note.issued`.
- Усі `orderId`, `shipmentId`, `accountId` лишаються непрозорими рядками; власні ID Billing — UUID. Суми в контрактах — десяткові рядки. Платіжна подія публікується після підтвердженого allocation.
- Типи звірено з ЛР2: `dueAt` — календарна дата (`YYYY-MM-DD`, OpenAPI `format: date`); суми — decimal strings, що відповідають `NUMERIC(14,2)` (до 12 цифр до крапки та 2 після); кількість у кілограмах — `NUMERIC(12,3)`. VAT округлюється до копійок на рівні рядка.
- `/internal/orders/{orderId}/billing-snapshot` не включено до OpenAPI Billing: це залежність від контракту Orders.

## 2. Синхронні стики та обмін подіями

### 2.1 Вихідні виклики Billing

| Стик | Протокол / тайм-аути | Повтори | Кореляція | Ідемпотентність | Недоступність партнера |
|---|---|---|---|---|---|
| Billing → Orders `GET /internal/orders/{orderId}/billing-snapshot?shipmentId=...` | HTTPS/REST; connect 1 с, response 3 с | До 3 спроб для timeout, `429`, `502`, `503`, `504`; 200 мс × 2ⁿ + jitter. 4xx не повторювати | Передати `X-Correlation-Id` із події; згенерувати, якщо відсутній | GET безпечний для повтору. Перевіряти `status`, snapshot `pricingVersion` проти переданого `orderVersion`, shipment-scoped кількості та відповідність події | Зберегти подію в довговічному retry-процесі; після 5 доставок у вікні ≥10 хвилин — durable DLQ і алерт. Застаріла версія або відсутній маркер — без автоматичного retry, ручна звірка |
| Billing adapter → Odoo пошук партнера/рахунку | XML-RPC Odoo 18; connect 2 с, response 10 с | До 3 спроб на тимчасові помилки; rate limit спільний — близько 60 запитів/хв | Correlation ID зберігається в `odoo_sync` і логах адаптера; Odoo ID не стає correlation ID | Read-only пошук безпечний; пошук рахунку за `x_billing_invoice_id` | Задача залишається в довговічній черзі. У неділю 02:00–04:00 адаптер не відправляє запити або ставить їх у паузу; після вікна — контрольоване відновлення, див. [ADR-03-01](#adr-03-01) |
| Billing adapter → Odoo `create` / `action_post` | XML-RPC; connect 2 с, response 15 с для create, 20 с для post | Не повторювати write наосліп. Після timeout спершу reconciliation-пошук; повторити лише після підтвердження, що операція не відбулася | Той самий correlation ID з job; один in-flight job на invoice | `odoo_sync.invoice_id` UNIQUE; пошук за `x_billing_invoice_id`. 0 збігів — контрольоване створення; 1 — використати запис; >1 — зупинити й `manual_review` | Odoo job не губиться й не протерміновується після 10 хвилин; зберігається через планове вікно до 2 годин. Після неоднозначного результату — ручна звірка, не повторне створення |

Знімок Orders має бути прив’язаний до конкретного `shipmentId` і містити тільки його фактичні рядки/кількості, разом із цінами й умовами. `orders.order.confirmed` може нести необов’язковий `orderVersion`; Inventory зберігає передане значення разом із резервом і повертає його в `inventory.shipment.dispatched`. Billing зіставляє його зі snapshot `pricingVersion`. Розбіжність означає застарілі ціни/умови: issue зупиняється й переходить на ручну звірку. Якщо `orderVersion` відсутній, автоматично довести відповідність цінового контексту неможливо; пропозиція Billing — зупинити issue й направити випадок на ручну звірку. Ця поведінка лишається на погодження з Orders та Inventory, зокрема для legacy-повідомлень. Це версіонування цінового контексту, а не повної сутності Orders.

Усі HTTP-виклики Billing передають `X-Correlation-Id`. Для XML-RPC Odoo той самий ID зберігається в job і структурованих логах адаптера. Повтори обмежені ідемпотентними GET або write-операціями, для яких адаптер спочатку встановив результат reconciliation. 4xx (крім 429) не повторюються. Circuit breaker після 5 послідовних transient-помилок на 30 секунд — додатковий захист; чергу він не очищує.

Для `odoo_sync` пропоную розклад повторів 5 с, 30 с, 2 хв, 10 хв, 30 хв, далі щогодини з jitter; записи не видаляються після 10 хв або 2 годин. У планове недільне вікно job можна відкласти до 04:00 локального часу партнера й потім обробляти чергу з rate limit. Якщо після кількох reconciliation-циклів результат лишається неоднозначним, статус переходить у `manual_review`, але дані та історія зберігаються.

Платіжний провайдер — майбутній вхідний стик, не реалізація цієї лабораторної. Якщо контракт буде активований, webhook перевіряє HMAC-SHA256, дедуплікує за `providerEventId`, довговічно зберігає повідомлення й повертає 2xx не пізніше 10 с; бізнес-обробка відбувається асинхронно. Події провайдера можуть бути не за порядком, тож старий статус не перезаписує новіший.

### 2.2 Підписки Billing

| Подія | Обробник і зміна | Дублікат / порядок | Помилка й відновлення |
|---|---|---|---|
| `crm.account.upserted` | Upsert повної репліки клієнта за зовнішнім `accountId` | Inbox за `eventId`; порівняти `source_updated_at`, старішу версію відкинути | Тимчасова помилка — retry; payload, який не проходить схему, — DLQ та виправлення власником CRM/Billing |
| `inventory.shipment.dispatched` | Перевірити `orderId`, `shipmentId`, рядки й фактичні кількості цієї поставки; отримати Orders snapshot; створити/видати один рахунок на shipment | Inbox за `eventId` + `UNIQUE(shipment_id)`. На подіях, що несуть стан invoice, використовувати `invoiceVersion`. Застарілий Orders snapshot — stop/manual review | Тимчасова недоступність Orders — retry протягом щонайменше 10 хвилин; потім durable DLQ, без втрати події. Невідповідні рядки — manual review |
| `orders.order.cancelled` | Draft рахунки цього замовлення скасувати без номера; issued/partially-paid/paid не змінювати, створити задачу accountant | Inbox за `eventId`; зміна стану ідемпотентна. Якщо cancellation випередив shipment event, snapshot має показати `cancelled`, і Billing не issue рахунок | DB/transient failure — retry. Суперечність станів або вже проведений рахунок — review task, без автоматичного редагування |

Порядок різних routing key/подій не вважається гарантованим. Для account використовується `source_updated_at`; для рахунку — монотонний `invoiceVersion`; для shipment — `shipmentId` та звірка переданого `orderVersion` зі snapshot `pricingVersion`. `changedAt`/`issuedAt` лишаються для аудиту, не для сортування. Orders тримає останній `invoiceVersion` окремо для кожного рахунку. Якщо версія пропущена, Orders може відновити свою проєкцію через власний `GET /orders/{orderId}/invoices`.

### 2.3 RabbitMQ: топологія Billing

| Об’єкт | Налаштування / призначення |
|---|---|
| Exchange `kavohub.events` | Durable topic exchange; routing key дорівнює event `type` |
| `billing.events` | Durable quorum queue; bindings: `crm.account.upserted`, `inventory.shipment.dispatched`, `orders.order.cancelled`; persistent messages |
| Retry queues | Durable queues `billing.events.retry.5s`, `.30s`, `.2m`, `.8m`; TTL 5 000 / 30 000 / 120 000 / 480 000 мс; dead-letter exchange `""`, dead-letter routing key `billing.events` |
| `billing.events.dlq` | Durable quarantine queue; отримує події після п’ятої невдалої обробки або не-відновлюваної помилки; повідомлення не видаляється автоматично |
| Outbox | `outbox_event` у БД Billing; записується в тій самій транзакції, що й invoice/payment change; publisher надсилає persistent event і чекає publisher confirm |

Спроби входу в `billing.events` плануються на t=0, t=5 с, t=35 с, t=2 хв 35 с і t=10 хв 35 с. Після п’ятої невдачі повідомлення переходить у DLQ. Це прибирає проблему DLQ через ≈25 секунд і зберігає можливість повторної обробки після недоступності до 10 хв за NFR-REL-01. DLQ durable; оператор бачить алерт і може replay після усунення причини. Для планового Odoo вікна до 2 годин використовується окрема довговічна `odoo_sync` черга/таблиця, а не коротка Rabbit retry-черга.

### 2.4 Inbox, outbox та ack

Inbox-таблиця Billing має щонайменше `event_id UUID PRIMARY KEY`, `event_type`, `correlation_id`, `received_at`, `processed_at`, `status` та `last_error`. Outbox має `event_id`, `event_type`, aggregate/invoice ID, payload, `created_at`, `published_at` і attempt metadata.

```text
on delivery m:
  перевірити envelope та схему
  якщо це shipment: отримати Orders snapshot поза DB-транзакцією
  BEGIN
    insert inbox(m.id); якщо вже існує як processed -> COMMIT; ACK; return
    перевірити shipmentId/version або команду cancel
    застосувати бізнес-зміну
    збільшити invoiceVersion для consumer-visible зміни invoice
    записати outbox event(и) з незмінним eventId та correlationId
    позначити inbox processed
  COMMIT
  ACK

on transient failure:
  зберегти/опублікувати retry copy з тим самим eventId, retryCount+1
  дочекатися publisher confirm
  ACK original лише після підтвердження retry copy

outbox publisher:
  опублікувати незаданий event із persistent delivery та publisher confirm
  після confirm позначити published
  якщо впав між publish і позначенням — перевидати той самий eventId;
  споживач відкине копію через inbox
```

Inbox/outbox commit-яться разом із відповідною бізнес-зміною. `ack` успішного повідомлення надсилається лише після `COMMIT`; якщо процес падає між commit та ack, повторна доставка стає no-op за `eventId`.

### 2.5 Idempotency-Key для HTTP-команд Billing

Для `POST /invoices`, `/issue`, `/cancel`, `/credit-notes`, `/payment-imports`, `/payments`, `/proposals` і `/allocate` Billing зберігає `principal + operation + key`, хеш тіла, `in_progress/completed`, відповідь і час завершення. Запис ключа й бізнес-ефект однієї команди атомарні; ключі зберігаються щонайменше 24 години. Той самий key/body повертає попередню відповідь; той самий key з іншим body — `422 idempotency-key-reuse`; конкурентний запит — `409 request-in-progress`. Для allocation один key ідемпотентний на всю команду з усім набором allocations.

### 2.6 Процедура DLQ

On-call власник Billing переглядає DLQ та correlation trail. Помилки контракту/даних виправляє команда-власник payload разом із Billing; помилки Odoo обробляє власник адаптера. Перед replay перевірити причину, статус inbox, invoice/shipment у БД і зовнішній стан. Якщо бізнес-транзакція вже committed, не виконувати її повторно; відновлювати лише невиконану роботу або outbox. Replay аудіюється, початковий ID події зберігається.

### 2.7 Сценарії стійкості (специфікація)

1. **Дубль поставки.** Given одна `inventory.shipment.dispatched` вже створила invoice і commit. When той самий `eventId` або новий `eventId` із тим самим `shipmentId` доставляється повторно. Then перший випадок відкидає inbox; другий зупиняє `UNIQUE(shipment_id)`; додаткового номера чи outbox invoice event немає.
2. **Недоступний Orders.** Given Orders snapshot endpoint недоступний 10 хв. When shipment event обробляється. Then повідомлення/робота зберігається, повтори тривають понад 10 хв, після п’ятої спроби на t≈10:35 воно перебуває в durable DLQ; після відновлення Orders оператор replay-ить подію, а Billing створює не більше одного рахунку.
3. **Poison event.** Given shipment payload постійно не проходить бізнес-перевірку. When п’ять обробок завершуються помилкою. Then після останньої спроби event у DLQ з причиною, correlationId і retry history; invoice не створений.
4. **Повтор HTTP issue.** Given `POST /invoices/{id}/issue` завершився й зберіг Idempotency-Key та відповідь. When клієнт повторив запит із тим самим ключем і тілом після timeout. Then повертається початкова відповідь, номер Billing не видається вдруге, outbox не отримує другу бізнес-подію.
5. **Odoo недоступний 10 хв.** Given invoice issue committed у Billing, але Odoo не відповідає. When адаптер повторює створення/проведення. Then `billing.invoice.issued` і PDF доступні, `odooSyncStatus=pending`; довговічна задача не втрачається, після відновлення адаптер проводить reconciliation перш ніж повторювати write.

## 3. Інтеграція Billing з Odoo

### 3.1 Рішення пошуку та відображення номерів

Основний ключ пошуку рахунку в Odoo — `x_billing_invoice_id` із унікальним обмеженням, якщо партнер дозволить поле/модуль. `ref` зберігає `billingInvoiceNumber`; ці два значення не використовуються як взаємозамінні ключі.

Якщо унікальне поле додати неможливо, погоджений fallback — зберігати в `ref` саме `invoiceId` як ключ пошуку. У такому варіанті номер Billing має зберігатися в окремому підтвердженому полі Odoo. Не можна одночасно використовувати `ref` і для `invoiceId`, і для номера Billing.

| Billing | Odoo | Примітка |
|---|---|---|
| `invoiceId` | `x_billing_invoice_id` (основний варіант) | Унікальний зовнішній ключ для пошуку й reconciliation |
| `billingInvoiceNumber` | `ref` | Номер рахунку КавоХаб; не є `odooJournalNumber` |
| `odoo_record_id` | `account.move.id` | Зберігається в адаптерній таблиці `odoo_sync` |
| `odooJournalNumber` | `account.move.name` після `action_post` | Може бути відсутній до проведення |
| `accountId`, ЄДРПОУ, назва | `res.partner` | Пошук за стабільним CRM reference/ЄДРПОУ; неоднозначність не вирішувати автоматично |
| дата рахунку, строк оплати | `invoice_date`, `invoice_date_due` | `dueAt` — дата без часу, `YYYY-MM-DD`, відповідає ЛР2 і схемі `date` |
| валюта й рядки рахунку | `currency_id`, `invoice_line_ids` | Точний мапінг податків, доходних рахунків та одиниць перевірити на базі партнера |

### 3.2 Поведінка пошуку та повторів

Перед `create` адаптер шукає рахунок за одним обраним зовнішнім ключем:

- **0 збігів:** створити чернетку, якщо попередня операція точно не завершилась; після невідомого timeout спершу виконати reconciliation-пошук.
- **1 збіг:** перевірити `invoiceId`, клієнта, суму, валюту та стан; використати цей запис, не створювати інший.
- **Більше 1 збігу:** припинити автоматичну обробку, позначити `manual_review`, записати всі знайдені Odoo ID та створити задачу звірки. Не створювати й не проводити ще один запис.

Адаптер має довговічну таблицю `odoo_sync` з унікальним `invoice_id`, `odoo_record_id`, `status`, `attempt_count`, `next_attempt_at`, `last_error` і журналом операцій. Зовнішній `operation_key` стабільний для рахунку, наприклад `billing-invoice:<invoiceId>`. Ключ сам по собі не гарантує ідемпотентності Odoo; він потрібен для відновлення локальної роботи.

### 3.3 Timeout і недоступність

Стартові налаштовувані значення: connect timeout 2 с; read timeout 10 с для пошуку, 15 с для `create`, 20 с для `action_post`. Значення уточнити за вимірюванням часу відповіді партнера. Після timeout команди запису не повторювати навмання: спершу прочитати стан Odoo за зовнішнім ключем. Якщо результат лишається неоднозначним — ручна звірка.

Odoo-операції зберігаються в довговічній черзі синхронізації, не видаляються після короткої серії повторів і автоматично поновлюються після відновлення Odoo. Це покриває NFR-REL-01 (події не губляться під час недоступності до 10 хв) і планове вікно Odoo до двох годин щонеділі. Після вичерпання автоматичних спроб запис лишається в `manual_review`/DLQ; DLQ є збереженим станом, а не видаленням.

Рейт-лімітер адаптера обмежує потік до приблизно 60 запитів на хвилину (не більше одного запиту за секунду), з поступовим відновленням черги після простою. Неідентифіковані помилки й помилки прав/валідації не повторюються як тимчасові.

### 3.4 Номер у PDF і момент події

PDF клієнту формується Billing і містить `billingInvoiceNumber`. Надсилання PDF не очікує на доступність Odoo; `odooJournalNumber` — внутрішній бухгалтерський номер і не є блокером для документа клієнта за прийнятою моделлю BIL-R1. API Billing показує `odooSyncStatus` окремо (`pending`, `synced`, `manual_review`) і повертає `odooJournalNumber`, коли той уже відомий.

`billing.invoice.issued` записується в outbox у транзакції з проведенням рахунку в Billing. Отже, подія означає «рахунок виданий у Billing», а не «Odoo вже успішно провів запис». Це потребує уточнення базового каталогу README, де для події вказано «і передано в Odoo». Якщо споживачеві потрібне підтвердження Odoo, окремо додати `billing.invoice.accounting_synced` або читати `odooSyncStatus` через Billing API.

### ADR-03-01 — Billing окремо, Odoo через адаптер

<a id="adr-03-01"></a>

**Статус:** рекомендовано; продовжує ADR-0004 ЛР2.

| Варіант | Переваги | Наслідки / ризики |
|---|---|---|
| Увесь облік рахунків і оплат в Odoo | Один обліковий контур та інтерфейс | Перенесення BIL-R1–R6 і matching у модулі Odoo; більша залежність від оновлень і конфігурації партнера |
| Billing володіє доменом, Odoo через адаптер (**обрано**) | Billing контролює рахунки, номери й allocations; протокольні зміни ізольовані | Потрібні довговічна синхронізація, reconciliation і узгодження двох номерів; недоступність Odoo не губить локальний рахунок |
| Гібрид: Odoo створює рахунки, Billing зіставляє платежі | Частина бухгалтерського процесу лишається у звичному Odoo | Розділене володіння станами й номерами; складніше визначити момент події та безпечно повторювати workflow |

**Наслідки:** `billing.invoice.issued` і PDF не очікують Odoo; `odooSyncStatus` окремо відображає стан бухгалтерської синхронізації. Черга зберігає роботу до відновлення Odoo.

## 4. Уявний інтеграційний прогін Lead-to-Cash

### 4.1 Сценарій

Клієнт «Кав’ярня “Зерно”» замовляє 10 кг кави й кавомолку зі знижкою 12%. Orders погоджує знижку, Inventory резервує та відвантажує товари. Billing виставляє рахунок на відвантажену поставку. Odoo недоступний 10 хвилин. Після відновлення надходить платіж, бухгалтерка підтверджує allocation; після повної оплати створюється кредит-нота на 1 кг.

### 4.2 Послідовність

```mermaid
sequenceDiagram
    participant CRM
    participant ORD as Orders
    participant INV as Inventory
    participant BILL as Billing
    participant ODOO as Odoo
    participant N as Notifications
    participant PAY as Payment source (simulation)

    CRM->>ORD: crm.deal.won (correlationId)
    ORD->>ORD: Створити замовлення, рядки, знижка 12%
    ORD->>INV: confirm / reserve (після погодження знижки)
    INV-->>ORD: inventory.stock.reserved
    INV->>INV: Відвантажити та створити ТТН
    INV-->>BILL: inventory.shipment.dispatched (eventId, orderId, shipmentId, shipment lines + qty, correlationId)
    BILL->>ORD: GET billing-snapshot (X-Correlation-Id)
    ORD-->>BILL: status, pricingVersion, shipmentLines, ціни/знижка, валюта/курс, умови, одержувач, доставка
    BILL->>BILL: Перевірити status і orderVersion проти pricingVersion; inbox + UNIQUE(shipmentId)
    BILL->>BILL: Транзакція: issue, billingInvoiceNumber, outbox, odoo_sync
    BILL-->>N: billing.invoice.issued (через outbox, at-least-once)
    N->>BILL: GET /invoices/{invoiceId}/document
    BILL-->>N: PDF з billingInvoiceNumber
    BILL->>ODOO: Створити/провести через адаптер
    Note over BILL,ODOO: Odoo недоступний 10 хв; задача збережена, повтори тривають
    ODOO-->>BILL: account.move id, odooJournalNumber після відновлення
    PAY-->>BILL: Майбутній webhook або імпорт CSV (джерело платежу)
    BILL->>BILL: Зберегти платіж як unmatched; не публікувати matched event
    BILL->>BILL: Accountant підтверджує allocate
    BILL-->>ORD: billing.payment.received (після commit allocation)
    BILL-->>ORD: billing.invoice.payment_status_changed (стан конкретного рахунку)
    BILL->>ODOO: Передати бухгалтерське подання оплати/звірки, якщо погоджено
    BILL->>BILL: Accountant проводить кредит-ноту на 1 кг
    BILL-->>ORD: billing.invoice.payment_status_changed з оновленими балансами/кредитом
```

### 4.3 Перевірка кроків і меж модулів

| Крок | Подія/виклик | Результат перевірки для Billing |
|---|---|---|
| CRM виграла угоду | `crm.deal.won` → Orders | Billing не підписаний на цю подію; участі Billing немає |
| Знижка 12% і підтвердження | Orders підтверджує після approval; курс EUR — за процесом Orders | Billing не затверджує знижку й не викликає НБУ. Він використовує підтверджені ціни/курс із знімка Orders для відвантаженої поставки |
| Резерв | `inventory.stock.reserved` → Orders | Billing не підписаний; повтор цього кроку має бути перевірений Inventory/Orders. У прогоні Billing окремо перевіряє дубль `shipment.dispatched` |
| Відвантаження | `inventory.shipment.dispatched` → Billing | Перевірити `orderId`, `shipmentId`, рядки/кількості конкретної поставки; якщо `orderVersion` переданий — звірити зі snapshot `pricingVersion`, якщо відсутній — зупинити issue для ручної звірки. Звірити рядки зі shipment-scoped `shipmentLines`. Inbox за `eventId`; `UNIQUE(shipmentId)` |
| Видання рахунку | Локальний issue + outbox `billing.invoice.issued` | Billing видає свій номер; Odoo sync незалежний і може бути `pending` |
| PDF | Notifications → `GET /invoices/{invoiceId}/document` | PDF доступний після локального issue і містить `billingInvoiceNumber`; Odoo номер не блокує надсилання за прийнятим рішенням |
| Платіж | Майбутній webhook — лише симульований контракт; поточний банківський процес — CSV | Після приймання платіж спершу `unmatched`; `billing.payment.received` виходить тільки після підтвердженого allocation |
| Зміна оплати | `billing.invoice.payment_status_changed` → Orders | Подія описує стан окремого рахунку й має `invoiceVersion`; Orders зберігає останню версію для кожного `invoiceId`, старі й дубльовані відкидає. Пропуски відновлює власним invoices read endpoint. Замовлення стає `paid`, коли всі рахунки оплачені, усі рядки відвантажені й виставлені |
| Кредит-нота на 1 кг | Команда accountant у Billing | Рахунок лишається `paid`, а статус замовлення не змінюється. Billing публікує `billing.credit_note.issued` з новим `invoiceVersion` і результатом кредит-ноти; окрема зміна payment status не потрібна, якщо статус не змінився |

### 4.4 Збій і дублікати

1. **Крок 7 (крок 3): дубль `inventory.stock.reserved`:** це дубль із базового сценарію; Billing на цю подію не підписаний, її обробляють Inventory/Orders. Додаткова перевірка Billing — дубль `inventory.shipment.dispatched`: той самий `eventId` дає no-op, а новий ID із тим самим `shipmentId` не створює другий рахунок.
2. **Orders недоступний під час знімка:** подія та незавершена робота зберігаються довговічно; retry працює довше за 10 хвилин і не переводить подію у втрату. Після вичерпання автоматичного вікна — збережена DLQ/ручний розгляд.
3. **Знімок Orders застарілий:** це не transient timeout. Billing зупиняє issue, не робить сліпий retry та створює задачу ручної звірки.
4. **Крок 7, друга частина — Odoo недоступний:** issue у Billing завершується, `billing.invoice.issued` публікується з outbox, PDF доступний, `odooSyncStatus=pending`. Адаптер зберігає роботу й поновлює її з лімітом ≈60 запитів/хв після простою, враховуючи вікно обслуговування в неділю 02:00–04:00. Деталі — [ADR-03-01](#adr-03-01).
5. **Кредит-нота після повної оплати:** рахунок лишається `paid`; Billing публікує `billing.credit_note.issued` з інкрементованим `invoiceVersion`. За прийнятою пропозицією Orders статус замовлення не змінюється.

## 5. Таблиця розбіжностей і запропонованих змін

| Крок | Розбіжність | Сторона | Запропонована зміна контракту/рішення | Реакція команди (модель прогону) |
|---|---|---|---|---|
| 5, рахунок і Odoo | README описує `billing.invoice.issued` як «виставлено і передано в Odoo», а прийнятий дизайн публікує після локального issue; Odoo може бути недоступний | Billing / загальний каталог | Уточнити семантику: подія означає issue у Billing. Додати `billing.invoice.accounting_synced` або використовувати `odooSyncStatus` для окремого підтвердження | Billing приймає зміну; споживачам повідомити, що PDF має `billingInvoiceNumber`, а не обов’язково `odooJournalNumber` |
| 5, номер PDF | Сценарій ЛР3 згадує номер рахунку з Odoo, тоді як BIL-R1 і ADR мають окремі номери | Billing / умова сценарію, підтвердження бізнесу | Зафіксувати клієнтський номер як `billingInvoiceNumber`; `odooJournalNumber` — бухгалтерський. Якщо вимога інша, PDF/Notifications мають чекати `accounting_synced` | Попередньо прийнято Billing number згідно BIL-R1; підтвердження стейкхолдера лишається відкритим |
| 6, оплата | `billing.payment.received` у каталозі означає «отримано та зіставлено»; raw payment може бути unmatched | Billing / Orders | Публікувати `billing.payment.received` лише після commit підтвердженого allocation; окремо передавати стан рахунку в `billing.invoice.payment_status_changed` | Billing приймає семантику; Orders має підтвердити споживання подій і агрегацію замовлення |
| 6, статус замовлення | Orders має агрегувати кілька рахунків і часткові відвантаження | Orders | `paid` лише якщо всі рахунки оплачені, всі рядки відвантажені та виставлені. Інакше лишається відповідний неповний статус | Пропозицію правила сформульовано; потрібне підтвердження команди Orders |
| 6, кредит-нота | Кредит-нота після повної оплати не змінює settlement status рахунку | Billing → Orders | `billing.credit_note.issued` передає факт кредит-ноти й новий `invoiceVersion`; рахунок лишається `paid`, статус замовлення не змінюється | Рішення для цієї моделі зафіксовано; назва/схема події ще потребують контрактного погодження |
| 6, подія після кредит-ноти | Якщо рахунок лишається `paid`, `payment_status_changed` не описує сам факт кредит-ноти | Billing / Orders | Публікувати запропоновану `billing.credit_note.issued` з `invoiceVersion`, поточним invoice status, залишком і customer credit. `payment_status_changed` надсилати лише коли змінився settlement status | Orders має підтвердити, чи цього достатньо для його проєкції |
| 7 (крок 3), дубль резерву | У сценарії ЛР3 двічі доставляється подія резерву, але Billing на неї не підписаний | Orders / Inventory | Залишити цей дубль як перевірку Inventory→Orders; для Billing окремо перевірити дубль `inventory.shipment.dispatched` | Billing поза межами дубля резерву; дубль поставки обробляє ідемпотентно |
| 7, недоступність зовнішньої системи | Odoo недоступний, короткий retry не покриває NFR-REL-01 і планове вікно обслуговування | Billing / Odoo adapter | Завершити issue локально, опублікувати подію, лишити `odooSyncStatus=pending`, зберігати й поновлювати синхронізацію з rate limit і reconciliation | Прийнято в Billing; див. [ADR-03-01](#adr-03-01) |
| 4, склад події відвантаження | Для рахунку на часткову поставку потрібні `orderId`, `shipmentId` і фактичні рядки/кількості саме цієї поставки; повний payload Inventory в базовому каталозі не задано | Inventory; Billing як споживач | Запропонувати required `orderId`, `shipmentId`, `lines[{sku, qty}]`; Orders snapshot має віддавати відповідні `shipmentLines` із цінами та умовами | Пропозиція до контракту Inventory/Orders; без цих даних Billing не формує рахунок для часткової поставки |
| 4, знімок Orders | Billing потребує саме цінову версію знімка для поставки | Orders / Inventory | `orders.order.confirmed` може передати `orderVersion`, Inventory зберігає його з резервом і включає у `inventory.shipment.dispatched`; Billing порівнює його з `pricingVersion` snapshot. `pricingVersion` змінюється лише при зміні ціни, знижки, курсу чи умов | Пропозицію узгоджено як модель; поля треба синхронно внести до контрактів Orders та Inventory їхніми власниками |
| 4, знімок Orders | Таблиця ЛР3 має загальний `GET /orders/{id}`, а Billing використовує shipment-scoped `/internal/orders/{orderId}/billing-snapshot` | Orders | Описати snapshot endpoint як залежність Billing і послатися на контракт Orders; не включати endpoint до OpenAPI Billing | Зафіксовано межу володіння: API належить команді Orders |
| 6, пропуск `invoiceVersion` | Orders може отримати події не за порядком і пропустити одну з них | Billing / Orders | Orders зберігає останню версію кожного `invoiceId`, відкидає `<=` і відновлює пропуски через власний `GET /orders/{orderId}/invoices` | Пропозицію для механізму відновлення зафіксовано; endpoint є власністю Orders |
| 6, порядок подій рахунку | Різні події рахунку можуть прийти не за порядком; часові мітки ненадійні через розбіжність годинників | Billing / Orders | Додати монотонний `invoiceVersion` до `billing.invoice.issued`, `billing.invoice.payment_status_changed` і `billing.credit_note.issued`; Orders порівнює версії, а час використовує лише для аудиту | Прийнято в Billing; Orders має підтвердити застосування версії в проєкції |
| 6, платіжний провайдер | ЛР3 пропонує webhook, але поточний проєктний процес імпортує CSV; endpoint позначений як майбутній | Billing / умова ЛР3 | У прогоні webhook симулюється як майбутній контракт; для поточного процесу використовувати CSV. Реалізацію webhook не включати без окремої вимоги | Billing приймає симуляцію; обсяг реалізації не змінюється |

## 6. Відкриті питання після прогону

1. Чи погоджує Orders назви/схеми `billing.invoice.payment_status_changed` і `billing.credit_note.issued`, зокрема перелік полів кредит-ноти, потрібних Orders?
2. Чи підтверджують власники Orders та Inventory наскрізне передавання `orderVersion` (ціновий контекст) та його рівність `pricingVersion` у snapshot?
3. Чи має клієнтський PDF показувати лише `billingInvoiceNumber` (рекомендовано за BIL-R1), чи бізнес вимагає чекати на `odooJournalNumber`?
4. Чи дозволяє партнер додати унікальне поле `x_billing_invoice_id`; якщо ні, яке окреме поле зберігатиме `billingInvoiceNumber`, коли `ref` використано для `invoiceId`?
5. Які поля та права доступні фактично в Odoo Community партнера для партнера, податків, рахунку, проведення та кредит-ноти?

## Журнал проєктних рішень

| № | Рішення | Альтернативи | Обґрунтування | Підстава |
|---|---|---|---|---|
| D1 | Один рахунок на один відвантажений `shipmentId`; унікальність shipment + inbox | Один рахунок на все замовлення; рахунок на кожну позицію | Підтримує часткові поставки та повтори доставки | Власне рішення на основі `inventory.shipment.dispatched` |
| D2 | Billing є джерелом істини рахунку й публікує через outbox; Odoo — асинхронний облік | Увесь життєвий цикл рахунку в Odoo; гібридне володіння | Доступність Billing не залежить від Odoo; немає сліпого повтору XML-RPC write | ADR-03-01, NFR-REL-01 |
| D3 | Перевіряти `orderVersion` події проти `pricingVersion` snapshot | Повний `order.version`; timestamp | Лише ціноутворювальні зміни мають блокувати виставлення; годинники не використовуються для порядку | Узгоджена модель Orders → Inventory → Billing |
| D4 | Orders відкидає `invoiceVersion <= lastApplied` окремо на рахунок; пропуски відновлює через власний invoices endpoint | Впорядкована доставка; сортування за часом | Черги не гарантують порядок між типами подій, час може мати clock skew | Контрактна пропозиція Billing → Orders |
| D5 | Після credit note повністю оплаченого рахунку invoice лишається `paid`; статус замовлення не змінюється; публікується `credit_note.issued` | Перевести invoice/order у credited/refunded | Кредит-нота є окремим фінансовим фактом; замовлення відображає виконання та оплату поставленого обсягу | Рішення для прогону; потребує погодження контракту Orders |

## Уточнення стейкхолдерів і припущення

| № | Уточнення / припущення | Джерело | Вплив |
|---|---|---|---|
| A1 | `orders.order.confirmed.orderVersion` — необов’язковий ціновий маркер; Inventory зберігає й повертає його при відвантаженні; Billing зіставляє з `pricingVersion` snapshot | Рішення користувача для ЛР3 | Виявлення зміни ціни/знижки/курсу/умов |
| A2 | Замовлення стає `paid`, коли оплачені всі рахунки та всі рядки замовлення відвантажені й виставлені | Пропозиція користувача до Orders | Правило агрегації статусу Orders; не реалізується в Billing |
| A3 | Credit note після повної оплати не змінює `paid` у рахунку чи замовленні; Billing повідомляє факт окремою подією | Пропозиція користувача | Споживачам потрібна схема `billing.credit_note.issued` |
| A4 | `GET /orders/{orderId}/invoices` належить Orders і використовується лише для відновлення пропущених версій/звірки | Рішення користувача та межі володіння модулями | Endpoint має бути у контракті Orders, не Billing |
| A5 | Інтерфейс Orders snapshot є shipment-scoped; доставка й податкові/цінові умови для часткової поставки підтверджуються Orders | Вихідна проєктна передумова | Потрібен окремий контракт Orders для snapshot |

## Відкриті питання та ризики

- Orders та Inventory мають внести передавання необов’язкового `orderVersion`/`pricingVersion` до своїх контрактів. Окремо очікує погодження з ними пропозиція Billing: якщо маркер відсутній, зупинити issue й відправити випадок на ручну звірку. Billing не може самостійно гарантувати це поле на вході.
- Orders має підтвердити запропоновані події та правило агрегації; межа робіт Billing — опублікувати точний стан рахунку й credit note.
- Партнер Odoo має підтвердити унікальне поле `x_billing_invoice_id`, фактичні права/поля Community та обробку кредит-нот.
- `dueAt` і точність сум звірені з ЛР2: дата без часу, `NUMERIC(14,2)`; кількість кг — `NUMERIC(12,3)`. VAT округлюється на рівні рядка до копійок.
- Бізнес має підтвердити, що PDF може використовувати `billingInvoiceNumber`, навіть коли Odoo sync ще `pending`.

## Самоперевірка за критеріями

| Критерій | Статус | Де в документі |
|---|---|---|
| OpenAPI 3.1 та AsyncAPI 3.0, помилки, ролі, ідемпотентність, кореляція й версіонування | `dueAt` звірено як `date`, money — як `NUMERIC(14,2)`; Redocly і AsyncAPI CLI: 0 помилок. Redocly залишає 2 попередження; AsyncAPI CLI — інформаційне нагадування про версію 3.1, тоді як ЛР3 вимагає 3.0 | `billing.openapi.yaml`, `billing.asyncapi.yaml`, розділ 1 |
| Синхронна взаємодія | Описано для Orders/Odoo, тайм-аути, безпечні повтори, кореляцію та недоступність | 2.1 |
| Стійкість подій | Inbox/outbox, RabbitMQ retry/DLQ, порядок/дедуплікація, ack після commit і DLQ-процедура описані | 2.2–2.7 |
| Зовнішня інтеграція | Odoo-мепінг, тайм-аути, reconciliation та ADR з альтернативами описані | 3.1–3.4, ADR-03-01 |
| Інтеграційний прогін | Є послідовність Lead-to-Cash і таблиця розбіжностей; залежності інших модулів винесено окремо | 4–5 |
| Журнал рішень, припущення та ризики | Заповнено; відкриті міжмодульні узгодження збережено | розділи вище |
| Контрольні відповіді та самоаналіз роботи з ШІ | Відповіді сформовані; самоаналіз структурований за п’ятьма етапами ЛР3. Автору слід перечитати й уточнити формулювання відповідно до власного досвіду | розділи нижче |
| Посилання/експорт діалогу | Не додано; автору потрібно прикріпити перед здачею | метадані документа |

## Відповіді на контрольні питання (чернетка для усного захисту)

1. **Навіщо контракти до коду?** Вони дають Billing, Orders, Inventory та Notifications змогу погодити поля, семантику й помилки та паралельно будувати mock-и. Провайдер не може мовчки змінити тип або значення поля без версійного узгодження.
2. **OpenAPI й AsyncAPI?** OpenAPI описує синхронні HTTP paths, запити, відповіді й безпеку. AsyncAPI описує message channels, операції публікації/споживання та payload подій.
3. **Чому ACK після COMMIT?** Якщо ACK виконати до commit, а процес впаде, брокер уже видалить повідомлення, хоча бізнес-зміна не збережена. Після commit повторна доставка безпечна завдяки inbox-дедуплікації.
4. **Як захищається споживач від дубля?** `eventId` записується у inbox в тій самій транзакції, що й зміна стану та outbox. Повтор із тим самим ID стає no-op; для поставок додатково діє `UNIQUE(shipment_id)`.
5. **Що повторювати?** Тимчасові збої мережі/5xx/429 — з backoff та jitter, лише для безпечного GET або ідемпотентно захищеної операції. Валідаційні 4xx не повторюються. Після невідомого результату Odoo write спершу потрібен reconciliation, а не сліпе створення вдруге.
6. **Той самий Idempotency-Key з іншим body?** Billing повертає `422 idempotency-key-reuse`; ключ прив’язаний до операції, суб’єкта та hash тіла.
7. **Odoo недоступний 10 хвилин?** Локальне issue і outbox commit проходять; `odooSyncStatus=pending`, job залишається довговічним і поновлюється з rate limit та вікном maintenance за ADR-03-01.
8. **Навіщо snapshot Orders?** Він є джерелом підтверджених цін/знижки/курсу/умов для конкретної часткової поставки. Альтернатива — передавати повний незмінний ціновий контекст у shipment event; це зменшує синхронну залежність, але розширює event і відповідальність Inventory за snapshot даних Orders.
9. **Як захистити webhook?** Майбутній endpoint перевіряє HMAC над сирим тілом із constant-time compare, дедуплікує `providerEventId`, зберігає webhook довговічно та швидко повертає 2xx; бізнес-обробка асинхронна.
10. **Які розбіжності в прогоні?** Odoo може бути недоступний після локального issue, тому PDF не може обіцяти Odoo journal number; shipment event потрібен з рядками поставки та версією цін; Orders потрібна версія рахунку й окремий credit-note сигнал.
11. **Події не за порядком?** Orders зберігає `invoiceVersion` по `invoiceId`, відкидає стару/повторну версію та звіряє пропуски через власний invoices endpoint. `changedAt` не використовується для сортування. Billing аналогічно зіставляє pricing version до створення рахунку.

## Самоаналіз роботи за етапами ЛР3

### Етап 1. Контракти модуля

Я використав API й доменні рішення ЛР2 як основу OpenAPI, а каталог подій README — для AsyncAPI. Під час уточнення контрактів я відділив факт отримання платежу від підтвердженого allocation: `billing.payment.received` виходить тільки після commit allocation, а зміна стану рахунку передається окремою подією. Також виявилося, що `GET /orders/{orderId}/invoices` належить Orders; спершу він помилково потрапив до OpenAPI Billing, після чого я прибрав його й залишив як залежність для reconciliation. Контракти пройшли лінтинг без помилок, але назви/семантика кількох подій і міжмодульні зміни ще потребують підтвердження Orders та Inventory.

### Етап 2. Синхронні виклики й обмін подіями

Я перевірив повтори проти вимоги недоступності до 10 хвилин і помітив, що п’ять спроб кожні п’ять секунд відправили б подію до DLQ приблизно за 25 секунд. Тому спроєктував довше вікно повторів і довговічну DLQ. Для доставки закріпив inbox, `UNIQUE(shipment_id)`, outbox та ACK після commit. У частині порядку подій повний `order.version` замінив на `invoiceVersion` для стану рахунку й окрему цінову перевірку `orderVersion` проти `pricingVersion`. Відкритою лишив пропозицію, що робити, коли необов’язкового `orderVersion` немає: її мають погодити Orders та Inventory.

### Етап 3. Зовнішня інтеграція

Для Odoo я обрав адаптер і асинхронну довговічну синхронізацію, щоб локальне виставлення Billing та публікація події не залежали від доступності Odoo. Я врахував неоднозначний результат XML-RPC timeout: `create` та `action_post` не повторюються наосліп, спершу виконується пошук/reconciliation. У мапінгу я розділив `x_billing_invoice_id` як ключ пошуку й `billingInvoiceNumber` як номер у `ref`; якщо знайдено кілька записів, автоматична обробка зупиняється на `manual_review`. Це зменшує ризик дубльованого проведення, але фактичну наявність унікального поля та дозволи партнера ще треба перевірити в його Odoo.

### Етап 4. Уявний інтеграційний прогін

Прогін допоміг виявити, що для рахунку на часткову поставку потрібні її власні рядки/кількості, а snapshot має бути shipment-scoped. Я також зафіксував розбіжність щодо номера в PDF, затриманої синхронізації Odoo, порядку invoice-подій і кредит-ноти після повної оплати. Для Orders сформулював правило агрегації `paid` і версійний механізм із `invoiceVersion`, але не видаю їх за погоджений контракт іншої команди. Це важливий результат прогону: Billing може запропонувати необхідні поля й семантику, а зміни в Orders/Inventory залишаються їхнім рішенням.

### Етап 5. Самоперевірка, захист і оформлення

Я звірив тип `dueAt` з ЛР2 як календарну дату, грошову точність — як `NUMERIC(14,2)`, вагу — як `NUMERIC(12,3)`, а округлення ПДВ — до копійок на рівні рядка. Redocly і AsyncAPI CLI показали нуль помилок. Два Redocly warnings стосуються відсутньої ліцензії та локального `localhost` URL; AsyncAPI CLI лише інформує про новішу версію 3.1, але я залишив потрібну за завданням 3.0. Перевірка підтвердила узгодженість схем на рівні лінтерів, але не замінює погодження контрактів споживачами чи перевірки на Odoo партнера. Перед здачею треба додати посилання/експорт діалогу й особисто перечитати відповіді для захисту.

**Діалог із ШІ:** додати посилання на чат або експортувати його й долучити до здачі відповідно до вимог курсу.

## 8. Статус лабораторної

Основні артефакти ЛР3 сформовані, рішення та відкриті міжмодульні залежності розмежовані. Контракти перевірені лінтерами; `dueAt` і точність сум звірені з ЛР2. Перед здачею автору лишається розглянути два Redocly попередження, перенести погоджені Billing-рішення до контрактів, отримати/зафіксувати контрактні відповіді Orders та Inventory і додати посилання або експорт цього діалогу. Неузгоджені контракти інших модулів лишаються відкритими пунктами, а не блокують оформлення Billing-частини.
