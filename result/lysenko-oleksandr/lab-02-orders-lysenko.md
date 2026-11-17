# Лабораторна робота № 2. Проєктування модуля Orders

## 1. Мета та сценарій

Ця робота продовжує наскрізний проєкт «КавоХаб» — B2B-дистриб’ютора кави, чаю та кавового обладнання для HoReCa. Orders відповідає за замовлення, контрактні ціни, знижки та оркестрацію виконання. CRM володіє клієнтами й угодами, Inventory — товарами, залишками та резервами, Billing/Odoo — рахунками й обліком. Рішення є навчальним проєктом, а не описом реальної компанії.

Orders інтегрується з іншими модулями через REST і події RabbitMQ. Зовнішній стик Orders — API НБУ для курсу EUR; Billing інтегрується з Odoo для бухгалтерських документів. Модулі мають окремі межі даних і PostgreSQL-схеми, а спільний RabbitMQ використовується для доменних подій. Для Odoo передбачаються квота близько 60 запитів/хв та щотижневе вікно обслуговування; фактичні можливості API залежать від редакції й розгортання.

## 2. Інтерв’ю та перевірка потреб

### Симульоване інтерв’ю

1. **Як створюється замовлення?** CRM публікує `crm.deal.won`; Orders використовує цю подію для створення чернетки замовлення.
2. **Як застосовуються ціни та знижки?** Для клієнтів можуть діяти контрактні ціни; знижка понад 10% потребує погодження керівника продажів.
3. **Як перевіряється наявність товару?** Inventory є власником залишків; Orders запитує доступність і просить атомарно зарезервувати всі рядки.
4. **Як обробляються валюта й рахунки?** Курс EUR отримується з НБУ та фіксується у версії пропозиції; бухгалтерські рахунки й облік залишаються в Odoo через Billing.

### Результати правдоподібної перевірки

Вихідні бізнес-факти взято зі сценарію «КавоХаб» і матеріалів ЛР1: індивідуальні ціни та знижки, погодження знижки понад 10%, резерв товару на час погодження не довше 24 годин, а також окремі власники замовлень, запасів і бухгалтерських документів. Для проєктування прийнято, що курс EUR фіксується у версії пропозиції й після погодження не змінюється. Строк дії пропозиції та правила округлення позначені як проєктні припущення, що потребують підтвердження бізнесу.

## 3. Межі та доменні сутності

| Сутність | Призначення |
|---|---|
| AccountReference | Репліка потрібних Orders реквізитів клієнта; джерело істини — CRM |
| Order / OrderLine | Замовлення й його позиції, ціни, валюта та версія |
| Product | Посилання на товар Inventory; Orders не володіє залишками |
| Reservation | Відповідь Inventory про резерв, час створення й завершення |
| Approval | Рішення щодо знижки: погоджувач, час, коментар |
| IntegrationOutbox | Надійна публікація доменних подій і синхронізація із зовнішніми системами |
| AuditEvent | Хто, коли й яку зміну виконав |

CRM володіє Account, Contact і Deal. Orders володіє Order, рядками, цінами та погодженням знижки. Inventory — каталогом, залишком, резервом і відвантаженням. Billing володіє рахунками й оплатами та інтегрується з Odoo; Orders отримує потрібні зміни стану через події.

## 4. Звіти й документи

**Операційний звіт Orders:** за вибраний період показує кількість і суми замовлень за станом, клієнтом, складом і валютою, замовлення на погодженні, замовлення `on_hold`, прострочені резерви та час проходження етапів. Дані про воронку й причини програшу належать CRM і не дублюються в Orders.

**Комерційна пропозиція:** дані Order і OrderLine містять клієнта, позиції, кількість, контрактну ціну, знижку, валюту, курс, дату курсу та строк чинності. Пропозиція не є бухгалтерським рахунком; рахунок створює Billing і передає до Odoo.

## 5. Odoo fit-gap і зв’язок із ЛР1

Оцінка орієнтується на документацію Odoo 18.0. Доступність зовнішнього API залежить від редакції, тарифу й розгортання. Odoo документує ліди [CRM](https://www.odoo.com/documentation/18.0/applications/sales/crm/acquire_leads/email_manual.html), аналіз виграних/програних угод [win/loss](https://www.odoo.com/documentation/18.0/applications/sales/crm/performance/win_loss.html), [пропозиції](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations.html), [знижки](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/discounts.html), [прайс-листи](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/pricing.html), [валюти](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/currencies.html), [резервування](https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/reservation_methods/at_confirmation.html), [рахунки](https://www.odoo.com/documentation/18.0/applications/sales/sales/invoicing/invoicing_policy.html), [кредит-ноти](https://www.odoo.com/documentation/18.0/applications/finance/accounting/customer_invoices/credit_notes.html) та [зовнішній API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html).

| Потреба | Fit-gap висновок |
|---|---|
| Ліди та воронка | Належить CRM і не дублюється в Orders |
| Замовлення й контрактні ціни | Orders зберігає замовлення та версії погоджених цін |
| Погодження знижок понад 10% | Правило бізнесу реалізується в Orders; не вважаємо його стандартною поведінкою Odoo |
| Ціни й валюти | Механізми є; нашій системі потрібен незмінний знімок курсу конкретної версії пропозиції |
| Запаси | У межах цієї архітектури ними володіє Inventory, не Odoo Orders |
| Рахунки й оплати | Власник — Billing; бухгалтерський облік лишається в Odoo |
| Інтеграція | Потрібні адаптер, квотування, повтори та перевірка API цільового тарифу |

У доступному результаті ЛР1 описано межі Orders/Inventory, атомарне резервування, чергу очікування, строк резерву 24 години та роль Odoo, але не знайдено формального беклогу з ID US-ORD. Тому наступні ID — **нові ідентифікатори простежуваності ЛР2**, а не приписані заднім числом вимоги ЛР1.

| ID ЛР2 | Основа в ЛР1 / gap | Реалізація | Пріоритет |
|---|---|---|---|
| US-ORD-01 | Orders не володіє запасами | product_id і запит до Inventory | Must |
| US-ORD-02 | Потрібен атомарний резерв | Резерв усіх позицій або відмова | Must |
| US-ORD-03 | Inventory може бути недоступний | on_hold без активного резерву; повтор після відновлення | Must |
| US-ORD-04 | Резерв обмежено 24 годинами | Таймер від фактичного reserved_at | Must |
| US-ORD-05 | Odoo — система бухгалтерського обліку | Orders публікує події; Billing передає рахунки й оплати в Odoo | Must |
| US-ORD-06 | Зовнішня інтеграція має квоту/перерви | Outbox у відповідальному модулі, обмеження частоти й повтори | Must |
| US-ORD-07 | Потрібно бачити стан виконання замовлень | Операційний звіт Orders за станами, сумами й резервами | Should |
| US-ORD-08 | Ціна має бути відтворювана | Знімок валюти, курсу, джерела й часу для кожної версії | Must |
| US-ORD-09 | Команди можуть повторюватися після тайм-ауту | Ідемпотентні POST-команди збережені ключем і відповіддю | Must |

## 6. Архітектура

1. **Orders API:** приймає команди менеджерів і сервісних клієнтів, перевіряє ролі та виконує бізнес-правила замовлень.
2. **Orders PostgreSQL:** зберігає замовлення, позиції, валютні знімки, погодження, резерви, аудит та outbox.
3. **CRM:** є власником клієнтів і угод; публікує `crm.account.upserted` та `crm.deal.won`, які Orders споживає.
4. **Inventory:** є власником товарів, залишків, резервів і відвантажень; Orders синхронно перевіряє доступність/створює резерв і споживає події виконання.
5. **Billing/Odoo:** Billing володіє рахунками й оплатами та інтегрується з Odoo. Orders отримує події про рахунок і оплату; бухгалтерський ledger у Orders не дублюється.
6. **RabbitMQ та зовнішні сервіси:** доменні події проходять через `kavohub.events`; Orders отримує курс EUR з API НБУ. Сервіси автентифікуються через OIDC та міжмодульні облікові дані.

~~~mermaid
flowchart LR
  M[Менеджер продажів] -->|HTTPS / OIDC| ORD[Orders API]
  ORD --> DB[(Orders PostgreSQL)]
  ORD -->|GET availability / POST reservation| INV[Inventory API]
  INV --> MQ[(RabbitMQ kavohub.events)]
  CRM[CRM] -->|crm.deal.won / account.upserted| MQ
  MQ --> ORD
  MQ -->|orders.*| INV
  MQ --> BILL[Billing]
  BILL -->|invoice.issued / payment.received| MQ
  ORD -->|EUR rate by date| NBU[API НБУ]
  BILL --> ODOO[Odoo accounting]
~~~

Кожен модуль змінює лише власні дані. Orders не читає чужі таблиці: клієнтські та товарні репліки оновлює з подій, а доступність запасу запитує в Inventory.

## 7. Межа інтеграції з Odoo

Orders не викликає Odoo напряму й не створює бухгалтерські рахунки. Після потрібного етапу Orders публікує доменну подію через transactional outbox; Billing отримує замовлення й відповідає за створення/проведення рахунку та синхронізацію з Odoo. Це відповідає власності даних у «КавоХаб» і не дублює бухгалтерський ledger в Orders. Відповідність полів і механізм API Odoo проєктуються на боці Billing. Тимчасова помилка Billing/Odoo не має губити подію чи скасовувати замовлення; потрібні обмежені повтори, моніторинг і DLQ у відповідальному модулі.

## 8. ADR-0004: розміщення функцій

| Варіант | Переваги | Недоліки |
|---|---|---|
| A. Реалізувати Orders у межах Odoo | Використання готових модулів продажів | Обмежена відповідність специфічним правилам і цільовій модульній архітектурі; залежність від редакції |
| B. Створити власні Orders та власний бухгалтерський облік | Максимальний контроль над процесом | Дублювання бухгалтерського ledger, рахунків та оплат; високі витрати й ризик |
| C. Створити власний Orders, лишивши бухгалтерію в Odoo через Billing | Відповідає межам модулів і зберігає Odoo як облік | Потрібні надійні події, адаптер Billing та моніторинг інтеграції |

**Рішення:** варіант C. Він узгоджується з цільовою архітектурою «КавоХаб»: Inventory є власником залишків, Orders — замовлень, Billing — рахунків та оплат, Odoo — бухгалтерських проведень. Наслідки: потрібні outbox, контроль доступу, аудит і надійна інтеграція між модулями.

## 9. Валюта, стани й правила

### Валютна політика

Для навчального проєкту встановлюємо конкретне правило: базова валюта обліку — UAH; документ може бути в UAH або EUR; EUR конвертується за офіційним курсом НБУ, зафіксованим на дату створення версії пропозиції. Зберігаються валюта, курс, дата, джерело та час отримання. Пропозиція чинна 7 календарних днів — це проєктне припущення, не підтверджене бізнесом. Після прийняття клієнтом валютний знімок незмінний. Прострочення вимагає нової версії з новим курсом; попередній DOCX не переписується. Суми документа округлюються до двох десяткових знаків; спосіб округлення та правила податків треба затвердити з бухгалтером до реального запуску.

### Розділені виміри стану

Order.status: draft, pending_approval, on_hold, confirmed, reserved, cancelled, shipped, invoiced, paid, closed. Окремо approval_status: not_required, pending, approved, rejected; reservation_status: not_requested, pending, on_hold, reserved, expired, released, failed. Знижка **понад 10%** вимагає погодження керівника продажів. Часткового резерву немає: Inventory резервує всі рядки або публікує `inventory.stock.reservation_failed`.

Послідовність: команда confirm перевіряє права, версію, рядки, ціну й чинність пропозиції. За знижки понад 10% система встановлює pending_approval і повертає **202 Accepted** з посиланням на погодження; очікування погодження — не 409. На цей час Orders просить Inventory повністю зарезервувати рядки з ключем ідемпотентності; резерв діє не довше 24 годин. Після схвалення Orders публікує `orders.order.confirmed`, а Inventory підтверджує вже наявний резерв цього замовлення без створення другого. Якщо резерв ще не був створений, Inventory виконує повний атомарний резерв. У разі відхилення або спливу строку Orders просить звільнити резерв; тимчасова недоступність Inventory переводить замовлення в `on_hold` без активного резерву й таймера. Після успішного резерву зберігаються `reserved_at` та `expires_at=reserved_at+24 години`. Події для Billing публікуються через outbox; збій інтеграції не губить локальне замовлення.

Правила цілісності: замовлення має Account і відповідального; порожнє не підтверджується; кількість > 0, знижка 0–100%; затверджена ціна/курс незмінні, зміна створює нову версію; Inventory — авторитетне джерело залишку; expected_version захищає від конкурентного редагування; повтор Idempotency-Key не створює дубль; після відвантаження чи рахунку звичайне скасування заборонене — потрібен процес повернення або кредит-ноти в Odoo.

### Таблиця заборонених переходів

| Стан | Команда | Результат |
|---|---|---|
| draft | mark_shipped | Заборонено: немає підтвердження й резерву |
| pending_approval | confirm_reservation | Заборонено до рішення погоджувача |
| on_hold | mark_shipped | Заборонено: резерв відсутній |
| reserved | cancel | Дозволено лише після успішного звільнення резерву й без рахунку |
| shipped / invoiced | cancel | Заборонено; застосувати процес повернення/коригування |
| closed | edit_lines | Заборонено; потрібне нове замовлення або коригування |
| cancelled | confirm | Заборонено; створити новий Order |

## 10. Реляційна БД і міграції

Скорочений ескіз PostgreSQL (пояснення українською; це не повний production DDL):

~~~sql
CREATE TABLE accounts (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  external_ref text,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE orders (
  id uuid PRIMARY KEY,
  account_id uuid NOT NULL REFERENCES accounts(id),
  status text NOT NULL CHECK (status IN
    ('draft','pending_approval','on_hold','confirmed','reserved',
     'cancelled','shipped','invoiced','paid','closed')),
  approval_status text NOT NULL,
  reservation_status text NOT NULL CHECK (reservation_status IN
    ('not_requested','pending','on_hold','reserved','expired','released','failed')),
  currency char(3) NOT NULL CHECK (currency IN ('UAH','EUR')),
  exchange_rate numeric(18,8) NOT NULL CHECK (exchange_rate > 0),
  rate_date date NOT NULL,
  rate_source text NOT NULL,
  valid_until date NOT NULL,
  version integer NOT NULL DEFAULT 1,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE order_lines (
  id uuid PRIMARY KEY,
  order_id uuid NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
  product_id uuid NOT NULL,
  quantity numeric(14,3) NOT NULL CHECK (quantity > 0),
  unit_price numeric(14,2) NOT NULL CHECK (unit_price >= 0),
  discount_percent numeric(5,2) NOT NULL DEFAULT 0
    CHECK (discount_percent BETWEEN 0 AND 100)
);
CREATE TABLE reservations (
  id uuid PRIMARY KEY,
  order_id uuid NOT NULL REFERENCES orders(id),
  inventory_ref text NOT NULL UNIQUE,
  status text NOT NULL,
  reserved_at timestamptz,
  expires_at timestamptz
);
CREATE TABLE integration_outbox (
  id uuid PRIMARY KEY,
  aggregate_id uuid NOT NULL,
  event_type text NOT NULL,
  idempotency_key text NOT NULL UNIQUE,
  payload jsonb NOT NULL,
  status text NOT NULL DEFAULT 'pending',
  attempts integer NOT NULL DEFAULT 0,
  next_attempt_at timestamptz NOT NULL DEFAULT now()
);
~~~

Міграції: 001 — accounts/orders/order_lines та обмеження; 002 — окремі стани погодження/резерву й reservations; 003 — outbox, idempotency, аудит та індекси. Міграції версіонуються, застосовуються послідовно й мають план відкату або сумісного виправлення. Seed навчального середовища: тестова компанія, контакт, товари з Inventory, UAH/EUR та замовлення в різних допустимих станах. Секрети не зберігаються у seed чи відкритому тексті.

## 11. REST API та помилки

API працює через HTTPS, використовує OIDC bearer-токен і перевіряє ролі. Помилки відповідають [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457).

~~~http
POST /api/v1/orders
Authorization: Bearer <access-token>
Idempotency-Key: 56fe...
Content-Type: application/json

{"accountId":"9d9f...","currency":"EUR","lines":[{"productId":"3b1a...","quantity":4,"unitPrice":1250,"discountPercent":5}]}

POST /api/v1/orders/{orderId}/confirm
If-Match: "7"
Idempotency-Key: 6a04...
~~~

Погодження повертає 202 із pending_approval та посиланням на рішення. Тимчасово недоступний Inventory може повернути 202 із on_hold. Застарілий If-Match повертає 409:

~~~json
{"type":"https://example.invalid/problems/concurrency-conflict","title":"Версію замовлення вже змінено","status":409,"detail":"Оновіть замовлення та повторіть команду.","correlationId":"b41c..."}
~~~

Основні маршрути: POST/GET/PATCH /api/v1/orders; POST /{id}/submit-approval, /approve, /reject, /confirm, /cancel; GET /api/v1/reports/orders. CRM-маршрути належать CRM; Orders не реалізує API для лідів, активностей чи Office Add-ins.

## 12. Тестові сценарії

Це заплановані перевірки; код не реалізовано, тож результати запуску не заявляються.

1. Створення чернетки з чинними Account і рядками; відмова для порожнього замовлення.
2. Відмова для кількості ≤ 0 та знижки поза межами 0–100%.
3. Знижка 10% не потребує погодження; знижка понад 10% (наприклад, 10,01% або 12%) створює pending_approval і повертає 202.
4. Відхилене погодження не викликає резервування; approve продовжує процес.
5. Повний атомарний резерв створює expires_at через 24 години від reserved_at.
6. Недоступність Inventory дає on_hold без активного таймера; нестача не створює частковий резерв за базовим правилом.
7. Повторний Idempotency-Key не створює дубль; застарілий If-Match дає 409 Problem Details.
8. Оновлення курсу не змінює раніше створену версію; прострочену пропозицію не підтверджують без нової версії.
9. Подія `billing.payment.received` переводить відповідне замовлення в `paid` один раз; відвантажене/раховане замовлення не скасовується звичайною командою.
10. Звіт обмежений заданим періодом і показує валюту; користувач без ролі не може погодити знижку.

## 13. Питання перед реальною реалізацією

- Перевірити редакцію, тариф і доступність API Odoo.
- Затвердити курс, строк пропозиції, податки та округлення з бізнесом і бухгалтером.
- Вирішити, чи потрібні частковий резерв і часткове відвантаження.
- Погодити ролі та пороги знижок.
- Визначити строки зберігання, аудит, доступ і правила персональних даних.
- Узгодити поведінку Orders для майбутньої дати НБУ, коли API повертає порожній масив.

## 14. Самоперевірка

| Вимога | Розділ |
|---|---|
| Бізнес, інтерв’ю та перевірка | 1–2 |
| Сутності та формат звіту | 3–4 |
| Fit-gap та зв’язок із результатами ЛР1 | 5 |
| Межі Orders та інтеграція з CRM, Inventory, Billing і Odoo | 1–8 |
| Адаптер і ADR з трьома варіантами | 7–8 |
| Валюта, переходи, правила, схема БД, API й тести | 9–12 |

## Офіційні джерела

- НБУ: [API курсів валют](https://bank.gov.ua/NBUStatService/v1/statdirectory/exchange).
- Odoo 18.0: [CRM leads](https://www.odoo.com/documentation/18.0/applications/sales/crm/acquire_leads/email_manual.html), [win/loss](https://www.odoo.com/documentation/18.0/applications/sales/crm/performance/win_loss.html), [quotations](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations.html), [discounts](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/discounts.html), [pricelists](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/pricing.html), [currencies](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/currencies.html), [reservation](https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/reservation_methods/at_confirmation.html), [invoicing](https://www.odoo.com/documentation/18.0/applications/sales/sales/invoicing/invoicing_policy.html), [credit notes](https://www.odoo.com/documentation/18.0/applications/finance/accounting/customer_invoices/credit_notes.html), [External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html).
- IETF: [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457).
