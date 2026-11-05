# Лабораторна робота № 2. Проєктування модуля Orders

## 1. Мета та сценарій

Ця робота моделює B2B-компанію «ТехноПостач Сервіс», яка постачає промислові комплектувальні вироби й надає сервісне обслуговування. Менеджери спілкуються з клієнтами через Outlook і Telegram, готують документи у Word 365, а бухгалтерія використовує Odoo. Бізнес, інтерв’ю та перевірка є навчальною симуляцією, а не описом реальної компанії.

Рішення має мінімізувати зміни для працівників: Outlook Add-in стає точкою фіксації звернення, контакту й угоди, Word Add-in формує пропозиції та звіти за шаблоном. Telegram на першому етапі фіксується вручну. CRM/Orders — операційні модулі навчальної системи; Odoo лишається системою бухгалтерських документів. За сценарієм є API-ключ Odoo, орієнтовна квота 60 запитів/хв та щотижневе вікно обслуговування до двох годин. Реальні можливості API залежать від редакції й тарифу та мають бути перевірені.

## 2. Інтерв’ю та перевірка потреб

### Симульоване інтерв’ю

1. **Як ведеться база клієнтів?** Керівник: «Історія розкидана між листами, Telegram і таблицями; колега не завжди бачить домовленості, якщо менеджер відсутній».
2. **Де губляться замовлення?** «Після пропозиції не завжди є нагадування зателефонувати; частина угод лишається без наступної дії».
3. **Як готується звітність?** «Менеджери зводять суми вручну, показники рахують неоднаково, керівництву важко оцінити прогноз і причини програшу».
4. **Які зміни прийнятні?** «Не хочемо окремого порталу. Зручніше працювати з Outlook, а документи формувати у Word. Telegram спершу можна записувати вручну».

### Результати правдоподібної перевірки

У навчальній перевірці взяли участь керівник, два менеджери та бухгалтер. На умовних прикладах з’ясовано: історія Outlook відновлюється, але Telegram не пов’язаний із карткою клієнта; є пропозиції без наступного контакту; керівнику потрібні суми за етапами, зважений прогноз, прострочені пропозиції та причини програшу; менеджери приймають коротку форму в Outlook; бухгалтер наполягає, щоб рахунки й облік лишилися в Odoo. Учасники погодили ручне внесення Telegram на першому етапі. Бізнес не підтвердив курс, строк дії пропозиції й округлення — нижче вони позначені як припущення проєкту.

## 3. Межі та доменні сутності

| Сутність | Призначення |
|---|---|
| Account | Компанія-клієнт, реквізити, сегмент і власник |
| Contact | Контактна особа Account, посада й канали зв’язку |
| Lead | Потенційний попит до кваліфікації, джерело й наступна дія |
| Deal | Комерційна можливість, етап, сума, імовірність, дата закриття |
| Activity | Лист, дзвінок, зустріч, повідомлення або задача-нагадування |
| Order / OrderLine | Замовлення й його позиції, ціни, валюта та версія |
| Product | Посилання на товар Inventory; Orders не володіє залишками |
| Reservation | Відповідь Inventory про резерв, час створення й завершення |
| Approval | Рішення щодо знижки: погоджувач, час, коментар |
| Document | Шаблон, стан генерації й посилання на файл M365 |
| IntegrationOutbox | Надійні події синхронізації з Odoo |
| AuditEvent | Хто, коли й яку зміну виконав |

CRM володіє Account, Contact, Lead та історією взаємодій. Orders володіє комерційним замовленням. Inventory — каталогом, залишком, резервом і відвантаженням. Odoo — бухгалтерськими документами й оплатами; Orders зберігає лише їхні зовнішні ідентифікатори та стан інтеграції.

## 4. Звіти й документи

**Щотижневий звіт із продажів і замовлень:** щопонеділка за попередній календарний тиждень, також доступний ручний запуск. Одержувачі — керівник продажів і керівництво. Вхідні дані: Deal, Order, Account, Activity, історія станів, причини програшу, менеджер, сума та валютний знімок. Структура DOCX: титульний блок із періодом і часом створення; коротке резюме; нові й активні угоди за етапами; відкриті пропозиції та прострочені дії; фактичні й зважені суми за валютами; виграні/програні угоди; причини програшу; план дій; джерела та примітка про валютні правила. PDF — похідна копія для розсилання/архіву, DOCX — редагований оригінал.

**Комерційна пропозиція:** шаблон DOCX із реквізитами сторін, номером і датою, строком чинності, позиціями, кількістю, ціною, знижкою, валютою, податками, умовами оплати/доставки та менеджером. Це не бухгалтерський рахунок; після погодження замовлення дані передаються до процесу рахунку в Odoo.

## 5. Odoo fit-gap і зв’язок із ЛР1

Оцінка орієнтується на документацію Odoo 18.0. Доступність зовнішнього API залежить від редакції, тарифу й розгортання. Odoo документує ліди [CRM](https://www.odoo.com/documentation/18.0/applications/sales/crm/acquire_leads/email_manual.html), аналіз виграних/програних угод [win/loss](https://www.odoo.com/documentation/18.0/applications/sales/crm/performance/win_loss.html), [пропозиції](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations.html), [знижки](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/discounts.html), [прайс-листи](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/pricing.html), [валюти](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/currencies.html), [резервування](https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/reservation_methods/at_confirmation.html), [рахунки](https://www.odoo.com/documentation/18.0/applications/sales/sales/invoicing/invoicing_policy.html), [кредит-ноти](https://www.odoo.com/documentation/18.0/applications/finance/accounting/customer_invoices/credit_notes.html) та [зовнішній API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html).

| Потреба | Fit-gap висновок |
|---|---|
| Ліди та воронка | Базові CRM-функції є; потрібне узгодження полів і швидке Outlook-внесення |
| Пропозиції | Базові Sales-пропозиції є; потрібен корпоративний Word-шаблон і звіт |
| Погодження знижок понад 12% | Правило навчального бізнесу треба налаштувати або реалізувати; не вважаємо його автоматичним стандартом Odoo |
| Ціни й валюти | Механізми є; нашій системі потрібен незмінний знімок курсу конкретної версії пропозиції |
| Запаси | У межах цієї архітектури ними володіє Inventory, не Odoo Orders |
| Рахунки й оплати | Залишаються в Odoo; бухгалтерський ledger не дублюється |
| Outlook/Word та звіт | Потрібні Office Add-ins, шаблони й зберігання у M365 |
| Інтеграція | Потрібні адаптер, квотування, повтори та перевірка API цільового тарифу |

У доступному результаті ЛР1 описано межі Orders/Inventory, атомарне резервування, чергу очікування, строк резерву 24 години та роль Odoo, але не знайдено формального беклогу з ID US-ORD. Тому наступні ID — **нові ідентифікатори простежуваності ЛР2**, а не приписані заднім числом вимоги ЛР1.

| ID ЛР2 | Основа в ЛР1 / gap | Реалізація | Пріоритет |
|---|---|---|---|
| US-ORD-01 | Orders не володіє запасами | product_id і запит до Inventory | Must |
| US-ORD-02 | Потрібен атомарний резерв | Резерв усіх позицій або відмова | Must |
| US-ORD-03 | Inventory може бути недоступний | on_hold без активного резерву; повтор після відновлення | Must |
| US-ORD-04 | Резерв обмежено 24 годинами | Таймер від фактичного reserved_at | Must |
| US-ORD-05 | Odoo — облік | Передавання погодженого замовлення через адаптер | Must |
| US-ORD-06 | API має квоту/перерви | Outbox, обмеження частоти й повтори | Must |
| US-ORD-07 | Office-процесу бракує | Outlook/Word Add-ins через REST API | Should |
| US-ORD-08 | Ціна має бути відтворювана | Знімок валюти, курсу, джерела й часу | Must |
| US-ORD-09 | Потрібна звітність | Щотижневий звіт за шаблоном Word | Should |

## 6. Архітектура

1. **Outlook Web Add-in:** Task Pane на Office.js знаходить Account/Contact, створює Lead або Activity та викликає CRM REST API. Див. [документацію Outlook Add-ins](https://learn.microsoft.com/en-us/office/dev/add-ins/outlook/apis).
2. **Word Web Add-in:** отримує структуровані дані звіту/пропозиції з API й формує документ через Word JavaScript API. Див. [Word Add-ins](https://learn.microsoft.com/en-us/office/dev/add-ins/word/).
3. **Microsoft Graph:** працює з файлами в OneDrive for Business/SharePoint. Готові DOCX/PDF зберігаються у бібліотеці M365; приклад — [завантаження DriveItem](https://learn.microsoft.com/en-us/graph/api/driveitem-put-content). Graph **не** з’єднується з реляційною БД і не замінює Backend.
4. **CRM Backend REST API / Orders Service:** автентифікує користувача, виконує правила, зберігає CRM/Orders, викликає Inventory і адаптер Odoo.
5. **Реляційна БД:** наприклад PostgreSQL, зберігає CRM/Orders, аудит і outbox.
6. **Автентифікація:** корпоративний API приймає OIDC-токен (наприклад, Keycloak у навчальному середовищі). Для Graph Add-in отримує окремий делегований токен через MSAL/NAA, за [документацією Microsoft](https://learn.microsoft.com/en-us/office/dev/add-ins/develop/enable-nested-app-authentication-in-your-add-in). Токен Graph не є токеном Orders API.

~~~mermaid
flowchart LR
  U[Менеджер] --> O[Outlook Client]
  O --> OP[Task Pane Office.js]
  OP -->|OIDC токен, HTTPS REST| API[CRM Backend REST API]
  U --> W[Word 365]
  W --> WP[Word Add-in Office.js]
  WP -->|дані звіту або пропозиції| API
  WP -->|Office.js: змінити документ| W
  API --> S[CRM та Orders Service]
  S --> DB[(Реляційна БД)]
  API -->|резерв| INV[Inventory API]
  API --> AD[Odoo Adapter]
  AD -->|зовнішній API| OD[Odoo Accounting]
  WP -->|делегований Graph токен: файл| G[Microsoft Graph API]
  G -->|DriveItem| SP[OneDrive for Business / SharePoint]
  WP -.->|шаблон DOCX| SP
~~~

Файл і операційні дані зберігаються окремо: БД містить сутності CRM/Orders, а M365 — шаблони та сформовані документи. Graph використовується для файлів, не як канал доступу до БД.

## 7. Адаптер Odoo

Адаптер працює тільки на сервері; ключ не потрапляє в надбудови. Він відображає Account на res.partner, Order на sale.order, OrderLine на sale.order.line, а зовнішній ID account.move записує як посилання на рахунок. Синхронізація CRM Deal із crm.lead вмикається лише за погодженим контрактом. Секрети зберігаються в сховищі секретів.

Події пишуться в IntegrationOutbox з унікальним idempotency key, correlation_id, числом спроб і часом наступної спроби. Адаптер обмежує частоту нижче сценарної квоти, повторює тимчасові помилки з експоненційною затримкою та випадковим розкидом, після ліміту переводить подію в dead-letter для ручного розбору. Під час технічного вікна події накопичуються. Помилка Odoo не губить підтверджене локальне замовлення. Доступність API треба перевірити на фактичному тарифі Odoo.

## 8. ADR-0004: розміщення функцій

| Варіант | Переваги | Недоліки |
|---|---|---|
| A. Перенести CRM/Orders/звіти в Odoo | Багато стандартних Sales-функцій | Не відповідає меті спроєктувати Orders; потрібна адаптація Office; залежність від редакції |
| B. Створити власну систему, включно з бухгалтерією | Максимальний контроль | Дублювання ledger, рахунків та оплат; найбільший обсяг і ризик |
| C. Власні CRM/Orders та Office Add-ins, Odoo для обліку | Мінімальні зміни робочих звичок, виконання цілі ЛР, чіткий поділ власників даних | Потрібні адаптер, моніторинг і підтримка інтеграції |

**Рішення:** варіант C. Він узгоджується з межами ЛР1: Inventory є власником залишку, Orders — замовлення, Odoo — бухгалтерських документів. Наслідки: потрібні outbox, контроль доступу, аудит і повтори інтеграції; Telegram лишається ручним процесом першого релізу.

## 9. Валюта, стани й правила

### Валютна політика

Для навчального проєкту встановлюємо конкретне правило: базова валюта обліку — UAH; документ може бути в UAH або EUR; EUR конвертується за офіційним курсом НБУ, зафіксованим на дату створення версії пропозиції. Зберігаються валюта, курс, дата, джерело та час отримання. Пропозиція чинна 7 календарних днів — це проєктне припущення, не підтверджене бізнесом. Після прийняття клієнтом валютний знімок незмінний. Прострочення вимагає нової версії з новим курсом; попередній DOCX не переписується. Суми документа округлюються до двох десяткових знаків; спосіб округлення та правила податків треба затвердити з бухгалтером до реального запуску.

### Розділені виміри стану

Order.status: draft, pending_approval, on_hold, confirmed, partially_reserved, reserved, cancelled, shipped, invoiced, closed. Окремо approval_status: not_required, pending, approved, rejected; reservation_status: not_requested, pending, on_hold, partially_reserved, reserved, expired, released, failed. Навчальний поріг знижки понад 12% вимагає погодження керівника.

Послідовність: команда confirm перевіряє права, версію, рядки, ціну й чинність пропозиції. За знижки понад 12% система встановлює pending_approval і повертає **202 Accepted** з посиланням на погодження; очікування погодження — не 409. Після погодження Backend запитує Inventory про атомарний повний резерв. Тимчасова недоступність Inventory дає on_hold без активного резерву й таймера. Після успішного резерву встановлюються reserved_at та expires_at=reserved_at+24 години. Після цього створюється outbox-подія для Odoo. Відмова Odoo не скасовує резерв. Відхилене погодження не резервує товар.

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
    ('draft','pending_approval','on_hold','confirmed','partially_reserved',
     'reserved','cancelled','shipped','invoiced','closed')),
  approval_status text NOT NULL,
  reservation_status text NOT NULL,
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

Основні маршрути: POST/GET/PATCH /api/v1/orders; POST /{id}/submit-approval, /approve, /reject, /confirm, /cancel; GET /api/v1/reports/weekly-sales; POST /api/v1/documents/quote. Outlook Add-in використовує CRM-маршрути, Word Add-in — звітні й документні. Graph-токен не надає доступу до Orders API.

## 12. Тестові сценарії

Це заплановані перевірки; код не реалізовано, тож результати запуску не заявляються.

1. Створення чернетки з чинними Account і рядками; відмова для порожнього замовлення.
2. Відмова для кількості ≤ 0 та знижки поза межами 0–100%.
3. Знижка 12% не потребує керівника; 12,01% створює pending_approval і повертає 202.
4. Відхилене погодження не викликає резервування; approve продовжує процес.
5. Повний атомарний резерв створює expires_at через 24 години від reserved_at.
6. Недоступність Inventory дає on_hold без активного таймера; нестача не створює частковий резерв за базовим правилом.
7. Повторний Idempotency-Key не створює дубль; застарілий If-Match дає 409 Problem Details.
8. Оновлення курсу не змінює раніше створену версію; прострочену пропозицію не підтверджують без нової версії.
9. Збій Odoo залишає retryable outbox-подію; відвантажене/раховане замовлення не скасовується звичайною командою.
10. Звіт обмежений заданим періодом і показує валюту; користувач без ролі не може погодити знижку.

## 13. Питання перед реальною реалізацією

- Перевірити редакцію, тариф і доступність API Odoo.
- Затвердити курс, строк пропозиції, податки та округлення з бізнесом і бухгалтером.
- Вирішити, чи потрібні частковий резерв і часткове відвантаження.
- Погодити ролі та пороги знижок.
- Перевірити політики Microsoft 365, дозволи Graph, SharePoint-сайт і правила зберігання документів.
- Визначити строки зберігання, аудит, доступ і правила персональних даних.
- Telegram інтегрувати окремим етапом після перевірки API та потреби.

## 14. Самоперевірка

| Вимога | Розділ |
|---|---|
| Бізнес, інтерв’ю та перевірка | 1–2 |
| Сутності та формат звіту | 3–4 |
| Fit-gap та зв’язок із результатами ЛР1 | 5 |
| Outlook, Word, Backend, БД, Graph і OneDrive/SharePoint | 6 |
| Адаптер і ADR з трьома варіантами | 7–8 |
| Валюта, переходи, правила, схема БД, API й тести | 9–12 |

## Офіційні джерела

- Microsoft Learn: [Outlook Add-ins](https://learn.microsoft.com/en-us/office/dev/add-ins/outlook/apis), [Word Add-ins](https://learn.microsoft.com/en-us/office/dev/add-ins/word/), [Nested App Authentication](https://learn.microsoft.com/en-us/office/dev/add-ins/develop/enable-nested-app-authentication-in-your-add-in), [Graph: завантаження DriveItem](https://learn.microsoft.com/en-us/graph/api/driveitem-put-content).
- Odoo 18.0: [CRM leads](https://www.odoo.com/documentation/18.0/applications/sales/crm/acquire_leads/email_manual.html), [win/loss](https://www.odoo.com/documentation/18.0/applications/sales/crm/performance/win_loss.html), [quotations](https://www.odoo.com/documentation/18.0/applications/sales/sales/sales_quotations.html), [discounts](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/discounts.html), [pricelists](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/pricing.html), [currencies](https://www.odoo.com/documentation/18.0/applications/sales/sales/products_prices/prices/currencies.html), [reservation](https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/reservation_methods/at_confirmation.html), [invoicing](https://www.odoo.com/documentation/18.0/applications/sales/sales/invoicing/invoicing_policy.html), [credit notes](https://www.odoo.com/documentation/18.0/applications/finance/accounting/customer_invoices/credit_notes.html), [External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html).
- IETF: [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457).
