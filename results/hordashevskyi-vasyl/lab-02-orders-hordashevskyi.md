# Лабораторна робота 2. ERP- та CRM-компоненти: fit-gap аналіз Odoo і проєктування модуля КПС
## Модуль Orders — КПС «КавоХаб»

**Студент:** Василь Гордашевський  
**Група:** не вказана у вихідних матеріалах  
**Дата:** 2026-11-21  
**ШІ-асистент:** ChatGPT (GPT-6)  
**Діалог:** посилання або експорт додається під час подання

## 0. Вихідні дані

Основа звіту — кейс «КавоХаб», вимоги ЛР1 та [lab-01-orders-hordashevskyi.md](lab-01-orders-hordashevskyi.md). ЛР1 визначила Orders власником Quote, Order і цін; Inventory — власником товарів, залишків і резервів; CRM — клієнтів та угод; Billing/Odoo — рахунків і оплат.

За фактами методички ЛР2, партнер використовує Odoo 18 Community, розглядає перехід на Odoo 19 наступного року, може видати API-ключ окремому технічному користувачу, має обмеження близько 60 запитів/хв і недільне вікно оновлення до двох годин. Це вихідні дані кейсу, а не незалежно перевірені факти партнера.

### Зміни рішень порівняно з ЛР1

1. Order стає confirmed лише після успішного резервування Inventory. До відповіді використовується pending_reservation.
2. Якщо резервування не вдалося, Order повертається у draft; orders.order.confirmed не публікується.
3. Знижку погоджує sales_head або делегований director, але не автор замовлення.
4. Quote й Order є окремими агрегатами; одна Quote може породити максимум один Order.
5. Ціни, курс і фінансові суми фіксуються у Quote та копіюються в Order для Billing.
6. Контракт перетворення тимчасового резерву та фактична доступність функцій Odoo Community лишаються питаннями перевірки.

## 1. Lead-to-Cash в Odoo

| Крок | Застосунок / моделі Odoo | Сценарій | Відмінність або перевірка для «КавоХаб» |
|---|---|---|---|
| Lead і opportunity | CRM: crm.lead; клієнт: res.partner | Створюється лід «Кав'ярня “Зерно”», його кваліфікують і ведуть до Won. | У КПС CRM володіє Deal і публікує crm.deal.won; Orders споживає подію та створює Draft Order із quoteId. Це наш контракт інтеграції, не поведінка Odoo. |
| Пропозиція | Sales: sale.order, sale.order.line, product.pricelist | Менеджер додає 10 кг кави та кавомолку, вводить 12% знижки й надсилає quotation. | У Odoo quotation після підтвердження стає sales order у тому самому записі; у КПС Quote і Order розділені. [Odoo 18: quotations](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations.html) |
| Підтвердження | Sales: sale.order | Користувач із правами підтверджує quotation; вона стає офіційним sales order. | Порогове погодження 10% і заборону самопогодження треба перевірити саме в Odoo 18 Community. |
| Резерв і доставка | Inventory: stock.picking, stock.quant | Створюється delivery order; спосіб резерву задається типом операції. | Odoo має режими At Confirmation, Manually і Before scheduled date. Тому резерв не завжди безстроковий чи миттєвий. TTL 24 год — окрема вимога. [Odoo 18: reservation methods](https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/reservation_methods.html) |
| Рахунок | Invoicing/Accounting: account.move | Рахунок можна виставити за замовленими або доставленими кількостями залежно від налаштування. | Billing отримує фінансовий знімок після відвантаження та веде рахунок в Odoo як у бухгалтерській системі запису. |
| Оплата | Accounting: account.payment | Фіксуються часткова й повна оплата та зіставлення з рахунком. | Імпорт банківської виписки належить Billing, не Orders. |
| Кредит-нота | Accounting: account.move | Коригування на 1 кг оформлюється окремим документом. | Виставлений рахунок не редагується; Billing створює кредит-ноту. |

**Знижка 12%.** Не припускаємо, що Odoo 18 Community штатно погоджує знижку понад 10%. Це перевіряється в середовищі партнера. У КПС Orders блокує підтвердження і запускає погодження, якщо перевірка не доведе, що Odoo повністю покриває правило.

## 2. Налаштування та аналіз Odoo Sales

### Запропоновані налаштування

- Увімкнути Discounts і Pricelists.
- Створити базовий UAH прайс-лист, клієнтські контрактні прайс-листи з датами дії та окремий EUR прайс-лист обладнання.
- Налаштувати quotation template зі строком п'ять робочих днів.
- Розділити групи Sales і Warehouse; перевірити, що комірник не бачить ціни у формах, звітах і API.
- Odoo документує валютні прайс-листи та зазначає, що для роботи з кількома валютами у Sales має бути встановлений Accounting. [Foreign currencies in Sales](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/currencies.html)
- Строк дії пропозиції налаштовується; документація не доводить жорсткого блокування підтвердження після строку. [Quotation deadlines](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations/deadline.html)

| Питання | Висновок | Статус підтвердження |
|---|---|---|
| Погодження знижки >10% | Потрібен пороговий workflow із забороною самопогодження; поки плануємо його в Orders. | Перевірити в Odoo 18 Community. |
| Курси валют і НБУ | Odoo документує автооновлення курсів з вибраного джерела; НБУ, потрібна редакція й fail-closed не підтверджені. | [Multi-currency](https://www.odoo.com/documentation/18.0/applications/finance/accounting/get_started/multi_currency.html); джерело НБУ перевірити. |
| Фіксація ціни в рахунку | Odoo має Sales-to-Invoice flow. Не доведено, що EUR курс саме з дати КП переноситься в рахунок. | Перевірити сценарій у Community; не стверджувати, що курс обов'язково береться на дату рахунку. |
| Прострочена quotation | Строк налаштовується, жорстке блокування не доведене. | Перевірити Odoo або реалізувати розширення. |

## 3. Fit-gap для Orders

Зусилля попередні: S — малі, M — середні, L — великі. Неперевірене позначено зірочкою.

| Вимога | Клас | Покриття / прогалина | Підстава | Зусилля | Вплив оновлень | Рішення |
|---|---|---|---|---|---|---|
| US-ORD-02 Базові й контрактні прайси за SKU/періодом | Configure | Правила прайс-листа підтримують товар і дати; клієнту призначається прайс. | [Pricelists](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/pricing.html); Community перевірити. | S | Низький без коду. | Orders зберігає master-правила; Odoo імпортується за потреби. |
| US-ORD-09 Комірник не бачить цін | Configure* | Рольові групи й окремі представлення; перевірити звіти та API. | Вимога С5. | S | Низький/середній. | Не передавати фінансові поля до Inventory. |
| US-ORD-01 Quote 5 робочих днів, прострочену не перенести | Extend* | Строк quotation підтримується; блокування не підтверджене. | [Quotation deadlines](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations/deadline.html). | M | Середній. | Жорстке правило у Orders. |
| US-ORD-04 Погодження >10%, без самопогодження | Extend | Пороговий workflow, ролі й аудит. | ORD-R3; Community перевірити. | M | Середній/високий. | Погодження у Orders. |
| US-ORD-05 Тимчасовий резерв на 24 год | Extend | Потрібні TTL, погодження та release понад стандартний метод резерву. | [Reservation methods](https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/reservation_methods.html). | M | Середній/високий. | Inventory володіє резервом, Orders оркеструє. |
| US-ORD-06 Підтвердити лише після резерву | Configure + Extend | Резерв At Confirmation налаштовується; подієвий контракт і Outbox — розширення. | Документація Inventory і ADR ЛР1. | M | Середній. | У КПС confirmed лише після Inventory. |
| US-ORD-07 Щотижневі Draft Orders із шаблону | Extend* | Є recurring plans/subscriptions, але автоматичне створення щотижневих чернеток у Community цим не доведене. | [Subscriptions](https://www.odoo.com/documentation/18.0/applications/sales/subscriptions.html); перевірити редакцію/сценарій. | M | Середній/високий. | Генерувати draft у Orders, якщо штатна функція не покриє вимогу. |
| US-ORD-03 EUR, курс НБУ на дату КП, fail-closed | Extend | Валютні прайси/курси є; НБУ, snapshot та fail-closed не підтверджені як готове рішення. | [Foreign currencies](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/currencies.html), [Multi-currency](https://www.odoo.com/documentation/18.0/applications/finance/accounting/get_started/multi_currency.html). | L | Високий для кастомізації Odoo. | ExchangeRateProvider у Orders, без мовчазного кешу. |

## 4. Адаптер до Odoo API

### Протокол і межа

Для Odoo 18 обрано XML-RPC: common endpoint для автентифікації й object endpoint з execute_kw. API-ключ технічного користувача використовується замість пароля. Документація Odoo 18 описує execute_kw, search_read, пагінацію та ключі API. [Odoo 18 External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html)

Application layer звертається до OdooCatalogPort. Домен не залежить від uid, execute_kw, Odoo domain, моделей sale.order і статусів Odoo. При переході на Odoo 19 можна замінити транспортний адаптер на JSON-2, але повторно перевірити мапінг.

| Операція | Модель/метод | Мапінг |
|---|---|---|
| Імпорт прайс-листів | product.pricelist/search_read | id, name, currency_id → зовнішній ID, назва, валюта |
| Імпорт рядків прайсу | product.pricelist.item/search_read | pricelist_id, product_id/product_tmpl_id, fixed_price, compute_price, date_start/end |
| SKU рядка | product.product або product.template/search_read | default_code читається через зв'язаний товар, не як поле рядка прайсу |
| Відкриті КП | sale.order/search_read | state draft/sent; id, name, partner_id, validity_date, currency_id, order_line, totals, write_date |
| Рядки КП | sale.order.line/search_read | order_id, product_id, product_uom_qty, price_unit, discount, tax_id |

Прайс-фільтр враховує date_start порожній або вже чинний і date_end порожній або ще чинний. Читання обмежується потрібним прайсом і компанією. Для синхронізації застосовується write_date watermark, пагінація та upsert.

### ACL, ідентичність і відмови

- Зовнішній ключ: UNIQUE(instance_id, model, external_odoo_id); повторний імпорт є upsert.
- Технічний користувач має read-only доступ лише до потрібних моделей/полів.
- API-ключ у secret store, не в логах.
- Ліміт адаптера 40–50 запитів/хв при партнерському обмеженні ~60.
- Тимчасові мережеві/5xx помилки — обмежений retry з exponential backoff і jitter. Під час недільної недоступності завдання відкладаються в чергу та продовжуються з watermark. 4xx доступу/валідації не повторюються автоматично.
- Логуються correlationId, зовнішній ID, тип операції, тривалість і результат без секретів.

Приклад схематичного XML-RPC виклику:

~~~python
models.execute_kw(
    db_name,
    technical_user_id,
    api_key,
    'product.pricelist.item',
    'search_read',
    [[
        ['pricelist_id', '=', pricelist_id],
        '|', ['date_start', '=', False], ['date_start', '<=', as_of_date],
        '|', ['date_end', '=', False], ['date_end', '>=', as_of_date],
    ]],
    {
        'fields': ['id', 'pricelist_id', 'product_id', 'product_tmpl_id',
                   'fixed_price', 'compute_price', 'date_start', 'date_end'],
        'limit': 100,
        'offset': 0,
        'order': 'id asc',
    },
)
~~~

Це проєктна ілюстрація; точний domain і наявність полів перевіряються в Odoo 18 середовищі.

## 5. ADR-0004 — Buy vs build

| Варіант | Переваги | Витрати й ризики |
|---|---|---|
| A. Усе в Odoo Community, правила у власних модулях | Менше систем, готові CRM/Sales/Inventory/Accounting. | Потрібні розширення погодження, TTL, НБУ й інтеграцій; сильніша прив'язка до моделей Odoo. 18→19 не обов'язково ламає код, але сумісність потребує перевірки й адаптації. |
| B. Odoo Enterprise + Studio | Більше готових і low-code можливостей; менше власного коду для частини сценаріїв. | Не відповідає обмеженню бюджету на ліцензії; інтеграції та власність даних залишаються. |
| C. Власні сервіси КПС, Odoo для бухгалтерії Billing | Контроль bounded contexts і правил; незалежна еволюція модулів. | Найвищі початкові витрати розробки/підтримки; адаптер, черги, спостережуваність і відновлення після простою. |

**Рішення — C.** CRM, Orders, Inventory, Billing і Notifications — модулі/сервіси КПС. Orders є системою запису Quote, Order, цін і погоджень; Inventory — товарів, залишків і резервів; Billing веде процес рахунків/оплат, Odoo є бухгалтерською системою запису. Orders передає Billing незмінний фінансовий знімок після відвантаження. Адаптер Orders може читати каталог/прайси/відкриті КП для порівняння або початкового імпорту.

TCO порівнюється за ліцензіями, розробкою, підтримкою, оновленнями, інтеграціями, простоями й навчанням. Власні модулі не вважаються гарантовано зламаними після оновлення, але потребують regression/sandbox перевірки.

## 6. Доменна модель і автомат станів

~~~mermaid
classDiagram
  class Quote {
    +QuoteId id
    +AccountId accountId
    +QuoteStatus status
    +Date expiresAt
    +ExchangeRateSnapshot rateSnapshot
    +QuoteLine[] lines
  }
  class QuoteLine {
    +LineId id
    +SKU sku
    +Quantity quantity
    +Money unitPrice
    +Decimal discountPercent
  }
  class Order {
    +OrderId id
    +QuoteId quoteId
    +AccountId accountId
    +WarehouseId warehouseId
    +OrderStatus status
    +Integer version
    +DiscountApproval approval
    +ExchangeRateSnapshot rateSnapshot
    +OrderLine[] lines
    +Money net/tax/total
  }
  class OrderLine {
    +LineId id
    +SKU sku
    +Quantity quantity
    +Money unitPriceSnapshot
    +Decimal discountPercent
    +Decimal taxRate
  }
  class DiscountApproval {
    +UserId requestedBy
    +UserId decidedBy
    +Decision decision
    +Instant requestedAt
    +Instant decidedAt
    +Instant expiresAt
  }
  class PriceList
  class ContractPrice
  class RecurringOrderTemplate
  class Money {
    +Decimal amount
    +Currency currency
  }
  class Quantity {
    +Decimal value
    +Unit unit
  }
  class ExchangeRateSnapshot {
    +Currency baseCurrency
    +Currency quoteCurrency
    +Decimal rate
    +Date rateDate
    +String source
    +Instant fetchedAt
  }
  class AccountReplica {
    +AccountId id
    +String sourceSystem
    +String externalId
  }
  class ProductReplica {
    +SKU sku
    +String sourceSystem
    +String externalId
    +Unit unit
    +Boolean isActive
  }
  Quote "1" *-- "1..*" QuoteLine
  Order "1" *-- "1..*" OrderLine
  Order "1" *-- "0..1" DiscountApproval
  Order ..> AccountReplica
  OrderLine ..> ProductReplica
~~~

Order — корінь агрегату; рядки й погодження змінюються в його транзакції. Quote має окремий життєвий цикл; UNIQUE quote_id обмежує один Order на Quote. OrderLine — дочірня сутність із локальним ID. DiscountApproval вбудований у Order. Money, Quantity та ExchangeRateSnapshot — незмінні Value Objects. Money перевіряє сумісність валют; ПДВ HALF_UP обчислюється на рівні рядка. ExchangeRateSnapshot зберігає напрям, наприклад UAH за 1 EUR, rate, rateDate, NBU source, fetchedAt. AccountReplica походить із CRM, ProductReplica — з Inventory; Orders їх не змінює.

~~~mermaid
stateDiagram-v2
  [*] --> draft
  draft --> pending_reservation: confirm, discount <= 10%
  pending_reservation --> confirmed: inventory.stock.reserved
  pending_reservation --> draft: inventory.stock.reservation_failed
  draft --> pending_approval: submit, discount > 10%
  pending_approval --> pending_approval: temporary reservation acknowledged
  pending_approval --> pending_reservation: approve, confirm same reservationId
  pending_approval --> release_pending: reject / withdraw / timeout
  release_pending --> draft: release confirmed
  pending_reservation --> release_pending: cancel
  pending_approval --> release_pending: cancel
  confirmed --> release_pending: cancel before shipment
  release_pending --> cancelled: release confirmed
  draft --> cancelled: cancel
  confirmed --> shipped: inventory.shipment.dispatched
  shipped --> invoiced: billing.invoice.issued
  invoiced --> paid: billing.payment.received, fully paid
~~~

| Стан | Команда/подія | Умова/роль | Новий стан й ефекти |
|---|---|---|---|
| draft | confirm | ≤10%, sales_manager/sales_head | pending_reservation; POST звичайного reserve з Idempotency-Key |
| pending_reservation | inventory.stock.reserved | Коректні order/reservation IDs | confirmed; Inbox + зміна стану + Outbox orders.order.confirmed в одній транзакції |
| pending_reservation | reservation_failed | Резерв не створено | draft; не публікувати confirmed |
| draft | submit | >10%, Sales role | pending_approval; TEMPORARY reserve, TTL ≤24 год, редагування заблоковане |
| pending_approval | reserved TEMPORARY | Inventory повернув reservationId/expiry | Лишається pending_approval; дозволити рішення |
| pending_approval | approve-discount | До expiry; sales_head/delegated director, не автор | pending_reservation; підтвердити той самий reservationId |
| pending_approval | reject/withdraw/timeout | Погоджувач, автор або scheduler | release_pending; запросити release |
| release_pending | released | Inventory підтвердив release | draft або cancelled залежно від команди |
| confirmed | shipment.dispatched | Подія для цього Order | shipped |
| shipped | invoice.issued | Рахунок відповідає Order | invoiced |
| invoiced | payment.received | Повна оплата | paid |
| shipped/invoiced/paid | cancel | Заборонено | 409 invalid-transition |

Погодження й тайм-аут використовують optimistic locking: лише одна зміна стану/версії перемагає. RabbitMQ ACK надсилається після commit Inbox + Order + Outbox.

## 7. Бізнес-правила

| Правило | Місце перевірки | Помилка/результат |
|---|---|---|
| ORD-R1: ContractPrice чинна на дату замовлення, інакше базовий прайс | PricingService при Quote/рядку; періоди в БД | 422 для невалідної ціни; вибір за пріоритетом |
| ORD-R2: EUR→UAH за курсом Quote, snapshot | ExchangeRateProvider + application service | Помилка курсу 503/422 за каталогом; без непомітного кешу |
| ORD-R3: >10% погоджується; автор не може погодити своє | Order aggregate + application authorization | 409 approval-required; 403 forbidden |
| ORD-R4: ПДВ 20% по рядку HALF_UP; total=sum(lines) | OrderLine/Order у транзакції; NUMERIC/Decimal | 422 для некоректної суми |
| ORD-R5: редагувати тільки draft; qty>0, шт — ціле | Доменний метод + CHECK | 409 invalid-transition або 422 validation-error |
| ORD-R6: не скасовувати після shipped | State machine | 409 invalid-transition |
| ORD-R7: тимчасовий резерв максимум 24 год | Scheduler Orders і TTL Inventory як страховка | Release; draft лише після підтвердження release |

## 8. Схема БД, міграції та seed

| Таблиця | Ключі й основні поля |
|---|---|
| quotes | PK id; UNIQUE quote_number; account_id; status; expires_at DATE; currency; net/tax/total NUMERIC(14,2); rate snapshot; created_by; version |
| quote_lines | PK id; FK quote_id; sku; quantity NUMERIC(12,3); unit; price/currency/discount/tax/net/tax/total |
| orders | PK id; UNIQUE order_number; UNIQUE nullable quote_id; account_id; warehouse_code; status; approval_expires_at; reservation_id; totals/rate snapshot; version |
| order_lines | PK id; FK order_id; sku; quantity/unit; price snapshot/currency/discount/tax/net/tax/total; CHECK qty>0; ціле для шт |
| price_lists / price_list_items | валюта, SKU, NUMERIC(14,2), validity, FK |
| contract_prices | account_id, sku, price/currency, valid_from, valid_to_exclusive; EXCLUDE overlap |
| recurring_order_templates | account, cadence, next_run, active, version і рядки шаблону |
| account_replicas / product_replicas | UNIQUE(source_system, external_id); поля-репліки CRM/Inventory; без зовнішніх FK |
| outbox_messages | UNIQUE event_id; aggregate_id/type, event_type, JSONB payload, correlation_id, created/published, status, attempts, next_attempt_at, lock_until, last_error |
| processed_events | UNIQUE(consumer_name,event_id), processed_at, aggregate_id, type, correlation_id |
| idempotency_records | UNIQUE(scope,idempotency_key), request_hash, saved response/status, expiry |

Гроші — NUMERIC(14,2); кількість — NUMERIC(12,3); rate з визначеною точністю; timestamp — timestamptz/UTC; бізнес-дати — Europe/Kyiv. Для EXCLUDE потрібне btree_gist; контрактні періоди напіввідкриті [valid_from, valid_to_exclusive), порожній верхній край означає безстроковість. Звичайний CHECK не може перевірити суму дочірніх рядків — це гарантується доменом у транзакції. version захищає від конкурентного оновлення.

Індекси: orders(account_id,status); частковий індекс approval_expires_at для pending_approval; quotes.expires_at для відкритих КП; order_lines(order_id), quote_lines(quote_id); outbox(status,next_attempt_at); Inbox unique key; GiST/exclusion для дат прайсів.

Міграції:
1. V1: схема, репліки, прайс-листи й контрактні ціни.
2. V2: quotes/orders/lines/approvals, обмеження та індекси.
3. V3: outbox, processed_events, idempotency_records.
4. V4: seed: п'ять клієнтів, двадцять товарів, довідникові коди складів KYIV/LVIV, UAH/EUR базові прайси й приклади контрактних цін. Залишки/резерви належать Inventory й там seed-яться.

## 9. REST API та RFC 9457

Команди переходу приймають Idempotency-Key. Повтор із тим самим ключем і тілом повертає збережену відповідь; інше тіло з тим самим ключем — 409 conflict. Прямого оновлення status немає.

| Метод / шлях | Роль і призначення | Успіх | Основні помилки |
|---|---|---|---|
| POST /quotes | sales_manager створює Quote | 201 з snapshot/expiresAt | 422, 503 курс, 403 |
| GET /quotes/{id} | власник, керівники, svc-crm за ACL | 200 | 404, 403 |
| GET /quotes/{id}/document | дозволений користувач/CRM | 200 PDF/ref | 404, 403 |
| GET /prices?accountId=&sku=&date= | Sales читає розраховану ціну | 200 | 404, 422 |
| POST /orders | створити Draft із Quote | 201 | 404, 409 Quote already used, 422 expired |
| GET /orders; GET /orders/{id} | Sales; Billing/accountant лише читання | 200 | 403, 404 |
| PUT /orders/{id}/lines | змінити Draft | 200 з version | 409, 422, 403 |
| POST /orders/{id}/confirm | запустити резерв | 202 pending_reservation/pending_approval | 409, 422, 403 |
| POST /orders/{id}/approve-discount | sales_head/delegated director, не автор | 202 pending_reservation | 403, 409 |
| POST /orders/{id}/reject-discount | погоджувач | 202 release_pending | 403, 409 |
| POST /orders/{id}/withdraw | автор відкликає погодження | 202 release_pending | 403, 409 |
| POST /orders/{id}/cancel | Sales role до shipment | 202 під час release або 200/204 після | 409, 403 |
| GET /health | публічний | 200 {"status":"ok"} | — |

Приклад забороненого скасування:

~~~http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json
~~~

~~~json
{
  "type": "https://kavohub.example/problems/invalid-transition",
  "title": "Недопустимий перехід стану",
  "status": 409,
  "detail": "Замовлення ORD-2026-000123 у стані shipped не можна скасувати",
  "instance": "/orders/ORD-2026-000123/cancel",
  "currentState": "shipped",
  "allowedActions": []
}
~~~

Приклад самопогодження:

~~~json
{
  "type": "https://kavohub.example/problems/forbidden",
  "title": "Дію заборонено",
  "status": 403,
  "detail": "Автор замовлення не може погодити власну знижку",
  "instance": "/orders/ORD-2026-000123/approve-discount"
}
~~~

Коди: 422 validation-error; 404 not-found; 403 forbidden; 409 invalid-transition, conflict або approval-required; асинхронні операції повертають 202.

## 10. Тестові сценарії

Це специфікації Given/When/Then, не запущені тести.

| ID | Given | When | Then | Правило |
|---|---|---|---|---|
| TC-01 | ContractPrice чинна на дату Quote | Розрахувати SKU | Застосована договірна ціна | R1 |
| TC-02 | ContractPrice завершилася, базова чинна | Розрахувати SKU | Застосована базова ціна | R1 |
| TC-03 | EUR товар і доступний NBU rate | Створити Quote | Snapshot має напрям/rate/date/source | R2 |
| TC-04 | NBU недоступний і валідного rate немає | Створити Quote | Quote відхилено; старий кеш не використано мовчки | R2 |
| TC-05 | Draft зі знижкою 8%, товар доступний | Confirm і отримати reserved | pending_reservation → confirmed, одна Outbox подія | R3 |
| TC-06 | Draft зі знижкою 12% | Автор погоджує сам | 403, стан незмінний | R3 |
| TC-07 | Знижка 12%, тимчасовий резерв є | Інший погоджувач approve, Inventory підтверджує той самий ID | confirmed, рішення й актор записані | R3/R7 |
| TC-08 | Два запити створюють Order з однієї Quote | Конкурентні вставки | Один 201, інший 409 UNIQUE quote_id | Унікальність |
| TC-09 | ПДВ 20% на рядках | Перерахувати Order | HALF_UP по рядку; total дорівнює сумі рядків | R4 |
| TC-10 | Не draft або qty<=0/дробова шт | PUT lines | 409 для стану; 422 для кількості | R5 |
| TC-11 | Order shipped | Cancel | 409 invalid-transition, allowedActions порожній | R6 |
| TC-12 | Pending approval, менеджер withdraw | Inventory підтверджує release | Draft; повтор ідемпотентний | R7 |
| TC-13 | Дедлайн настав одночасно з approve | Scheduler і погоджувач змагаються | Один переходить, інший 409; резерв звільняється | R7 |
| TC-14 | RabbitMQ повторює те саме event.id | Inbox вставляє processed event | Немає повторної зміни/Outbox; ACK після commit | Ідемпотентність |
| TC-15 | Два оновлення однієї version | Обидва редагують Order | Один успіх, другий 409 із актуальною version | Optimistic locking |
| TC-16 | Для account/SKU вже є період ціни | Додати перекривний період | EXCLUDE відхиляє; суміжні [from,to) дозволяє | R1 |
| TC-17 | Quote прострочена або прайс змінився пізніше | Створити Order / перечитати snapshot | Expired Quote заблокована; чинні snapshots незмінні | US-ORD-01/R2 |
| TC-18 | Inventory повертає reservation_failed | Обробити подію | Draft; confirmed не публікується | Інваріант ЛР1 |
| TC-19 | Inventory недоступний при release | Timeout/reject запускає release | release_pending, повтор ідемпотентний; новий submit не створює дубль | R7 |

### Матриця заборонених переходів

| Стан | Заборонені команди | Результат |
|---|---|---|
| draft | approve, reject, withdraw | 409 invalid-transition |
| pending_reservation | редагування, approve, reject | 409 invalid-transition |
| pending_approval | confirm, редагування | 409 invalid-transition |
| confirmed | редагування, approve, reject | 409 invalid-transition |
| shipped | cancel, edit, повторний confirm | 409 invalid-transition |
| invoiced | cancel, edit | 409 invalid-transition |
| paid | cancel, edit, approve | 409 invalid-transition |
| cancelled | будь-яка команда переходу | 409 invalid-transition |

## 11. Журнал проєктних рішень

| ID | Рішення | Альтернативи | Підстава |
|---|---|---|---|
| ADR-0004 | Власна КПС; Odoo — бухгалтерська система Billing | Усе Community; Enterprise+Studio; власні сервіси + Odoo Accounting | Бюджет, власність даних, TCO |
| ORD-ADR-01 | Confirmed лише після inventory.stock.reserved | Підтверджувати до резерву; переводити у hold | Інваріант ЛР1 |
| ORD-ADR-02 | DiscountApproval всередині Order | Окремий агрегат | Рішення атомарне зі станом Order |
| ORD-ADR-03 | Quote/Order зберігають цінові й курсові snapshots | Mutable прайс/поточний курс | Незмінність умов для рахунку |
| ORD-ADR-04 | Без валідного NBU rate EUR Quote не створюється | Тихий кеш; непозначений ручний rate | Fail-closed |
| ORD-ADR-05 | Scheduler Orders плюс TTL Inventory як страховка | Лише один із механізмів | Workflow в Orders, резерв у Inventory |
| ORD-ADR-06 | Inbox/Outbox у транзакціях | Publisher без Outbox; дедуп у пам'яті | At-least-once, відновлення після рестарту |
| ORD-ADR-07 | Періоди [from,to), перекриття заборонене | Перекриття з пріоритетом | Детермінований прайс |

## 12. Припущення та відкриті питання

1. Перевірити в Odoo 18 Community блокування простроченої quotation, discount approval, доступ комірника, щотижневі recurring draft orders, API та поля.
2. Погодити з Inventory контракт створення/конвертації/release тимчасового резерву, семантику retry і TTL.
3. Узгодити з Billing джерело істини для Invoice/Payment та момент запису в Odoo після shipment.
4. Визначити календар робочих днів, часовий пояс expiresAt, валютне округлення та курси НБУ у вихідні/свята.
5. Перевірити доступність btree_gist у PostgreSQL середовищі.
6. Уточнити часткові оплати та повідомлення Billing.
7. Додати посилання/експорт діалогу і результати перевірки Odoo перед поданням.

| ID | Припущення | Джерело/статус | Вплив |
|---|---|---|---|
| A1 | Odoo 18 Community; план переходу на 19 | Факт методички | API, ADR |
| A2 | ~60 rpm і до 2 год недоступності в неділю | Факт методички | Rate limit/queue |
| A3 | Документація підтверджує валютні прайси/курси/expiry field, не доводить Community, НБУ, fail-closed | Odoo docs; середовище не перевірено | Fit-gap |
| A4 | Quote діє до кінця п'ятого робочого дня Europe/Kyiv | Вимога ЛР1; календар свят уточнити | ExpiresAt |
| A5 | Ціна визначається на дату Quote та фіксується | ORD-R1/P5 | Billing snapshot |
| A6 | Без валідного NBU курсу EUR Quote блокується | Рішення ЛР1 | Доступність продажу |
| A7 | valid_to_exclusive для контрактних цін | Проєктне рішення | EXCLUDE |
| A8 | Одна Quote породжує максимум один Order | UNIQUE quote_id | Ідемпотентність |
| A9 | Делегування director має бути чинним у момент погодження | ЛР1; технічні деталі уточнити | RBAC |
| A10 | Billing веде фінансовий процес, Odoo — бухсистема запису | Рішення ADR | Синхронізація |

## 13. Самоперевірка за критеріями ЛР2

| Критерій | Бали | Статус | Коментар |
|---|---:|---|---|
| Lead-to-Cash і аналіз Odoo Sales | 1,5 | Частково | Наскрізний сценарій є; неперевірені функції Community треба підтвердити. |
| Fit-gap і ADR-0004 | 1,5 | Частково | 8 вимог і 3 альтернативи є; потрібні остаточні докази для позначених перевірок. |
| Адаптер Odoo API | 0,5 | Виконано як проєкт | Операції, поля, ACL, відмови та приклад описані. |
| Доменна модель і автомат | 1,5 | Виконано як проєкт | Агрегати й переходи описані. |
| Бізнес-правила й тести | 2 | Виконано як специфікація | R1—R7, 19 сценаріїв, негативні й конкурентні випадки. |
| БД, міграції, seed | 1 | Виконано як проєкт | Типи, обмеження, індекси, план; btree_gist перевірити. |
| REST API / RFC 9457 | 1 | Виконано як проєкт | Ендпоінти, ролі, приклади помилок є. |
| **Разом** | **9** | **Офіційну оцінку не виставлено** | Статуси описують повноту проєкту. |

## 14. Відповіді на контрольні питання

### 1. Як запобігаємо дублю резерву та повторній обробці?

Резервом володіє Inventory. Orders надсилає стабільний Idempotency-Key. Вхідна подія обробляється Inbox processed_events з UNIQUE(consumer_name,event_id). Inbox-вставка, перевірка стану/version, перехід Order і запис orders.order.confirmed в Outbox виконуються в одній транзакції. RabbitMQ ACK надсилається після commit. Дублікат із тим самим event.id не застосовується вдруге.

### 2. Чому команди замість прямої зміни status?

Команди виражають бізнес-намір, перевіряють стан/роль/автора та запускають необхідні ефекти. PATCH status обходив би інваріанти й аудит. Cancel у shipped повертає 409 Conflict, invalid-transition, із currentState та allowedActions.

### 3. Сутності й Value Objects

Order — коренева сутність із життєвим циклом; OrderLine — дочірня сутність із локальним ID. Money, Quantity, ExchangeRateSnapshot — immutable Value Objects без власного ID. Money має валюту; HALF_UP ПДВ обчислюється в рядку. Snapshot має напрям курсу, дату, джерело та час отримання.

### 4. Що захищає БД, а що домен?

БД захищає конкурентну унікальність і локальну цілісність через UNIQUE, EXCLUDE, CHECK і NUMERIC. Домен перевіряє переходи, роль, автора й інваріанти агрегату. Звичайний CHECK не перевірить суму дочірніх рядків. Доменні перевірки доповнюються БД і optimistic locking, щоб витримати конкурентні запити.

### 5. Чому не float для грошей?

IEEE 754 наближено представляє частину десяткових дробів. У JavaScript 100.10 * 3 може бути 300.29999999999995, а не рівно 300.30, і точне порівняння провалиться. Це показує похибку представлення, але саме по собі не доводить різницю на копійку після правильного округлення. Використовуються NUMERIC/Decimal та явне округлення.

## 15. Робота з ШІ-асистентом

1. Уточнено, що expiry quotation можна налаштувати, але жорстке блокування простроченого підтвердження потрібно перевірити.
2. Непідтверджені твердження про Enterprise-only курси, НБУ та мовчазний кеш не прийняті як факти; документ відокремлює доступну документацію від питань перевірки.
3. Узагальнення про безстрокове резервування виправлено: Odoo має різні reservation methods, а TTL 24 год — окрема вимога.
4. reservation_failed узгоджено з ЛР1: Order повертається в draft без події confirmed.
5. Приклад float збережено як похибку представлення, але не як доказ однокопієчної помилки після коректного округлення.

## Джерела

- Методичка: corporate-software-systems-lessons/lab-02.md.
- [ЛР1 Orders](lab-01-orders-hordashevskyi.md)
- [Odoo 18 Sales quotations](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations.html)
- [Odoo 18 quotation deadlines](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations/deadline.html)
- [Odoo 18 pricelists](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/pricing.html)
- [Odoo 18 foreign currencies](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/currencies.html)
- [Odoo 18 multi-currency](https://www.odoo.com/documentation/18.0/applications/finance/accounting/get_started/multi_currency.html)
- [Odoo 18 reservation methods](https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/reservation_methods.html)
- [Odoo 18 External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html)
- [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457)

