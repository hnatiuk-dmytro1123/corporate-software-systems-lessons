# lab-03.md

> Прочитай файл lab-03.md

<details><summary>2 previous messages</summary>

> Зараз знайду й прочитаю `lab-03.md` у проєкті.
>
> <details><summary>Explored a file, ran a command</summary>
>
> - Ran `rg --files -g 'lab-03.md' -g '!sources/**'`
>
> <details><summary>Explored a file</summary>
>
> - Read `./corporate-software-systems-lessons/lab-03.md`
>
> </details>
>
> </details>

</details>

Прочитав `lab-03.md`. Це завдання на проєктування інтеграції одного модуля: контракти OpenAPI та AsyncAPI, синхронні виклики й події з повторами та ідемпотентністю, зовнішню інтеграцію й уявний прогін Lead-to-Cash. Здається один проєктний документ із діаграмами, таблицями, ADR і сценаріями стійкості.

> Візмьи контексти попередніх чатів в даному проєкті й напиши, що мені потрібно зробити в даній ЛР (за md файлом)

<details><summary>2 previous messages</summary>

> Перевірю матеріали проєкту, щоб пов’язати вимоги ЛР3 із вашим модулем і напрацюваннями з попередніх лабораторних.
>
> <details><summary>Ran 3 commands</summary>
>
> - Ran `rg --files`
> - Ran `Get-Content -Raw 'corporate-software-systems-lessons\README.md'; Get-Content -Raw 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md'; Get-Content -Raw 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02.md'`
> - Ran `rg -n '^#|^##|^###|НБУ|ORD-R|Idempotency|NFR-PERF|endpoint|Endpoint|event|подію|події|курс' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' 'corporate-software-systems-lessons\lab-01.md' 'corporate-software-systems-lessons\lab-02.md'`
>
> </details>

</details>

За матеріалами проєкту ваш модуль — **Orders**. У попередніх напрацюваннях уже є основа для ЛР3: стани й правила замовлення, маршрути API, `Idempotency-Key`, валютна політика для EUR і НБУ, а також рішення про резервування та `on_hold`. Тож ЛР3 — це переважно **оформити й перевірити інтеграційні контракти Orders**, а не починати проєктування модуля з нуля.

Орієнтуюся на ваш документ ЛР2 [lab-02-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md), загальний каталог подій і правила проєкту з [README.md](./corporate-software-systems-lessons/README.md), а також вимоги [lab-03.md](./corporate-software-systems-lessons/lab-03.md).

## Що потрібно зробити

1. **Підготувати контракти Orders.**
   - Описати ваші REST-маршрути з ЛР2 в OpenAPI 3.1: створення й отримання замовлення, редагування, погодження знижки, підтвердження та скасування.
   - Для команд, які можуть повторитися, зафіксувати `Idempotency-Key`; для помилок — формат Problem Details; передавати й повертати `X-Correlation-Id`.
   - В AsyncAPI 3.0 описати події Orders: насамперед `orders.order.confirmed` і `orders.order.cancelled`, разом із форматом конверта з README.
   - Вказати версію контракту та правила сумісних змін. Пояснити, як споживачі зможуть працювати з mock-сервером до готовності реалізації.

2. **Описати всі стики Orders із сусідніми модулями.**
   У таблиці зазначити напрям, тайм-аути, повтори, ідемпотентність, `correlationId` і поведінку за недоступності партнера:
   - Orders → Inventory: `GET /availability`, `POST /reservations`, `POST /reservations/{id}/release`;
   - Orders → CRM: `GET /accounts/{id}`, якщо локальної репліки клієнта ще немає;
   - Orders споживає події CRM, Inventory та Billing, перелічені в таблиці взаємодії ЛР3.

   Для синхронних викликів узяти за основу тайм-аут з’єднання **1 с**, відповіді **2–3 с** і не більш як **3 повтори** лише для ідемпотентних запитів та тимчасових помилок. Урахувати NFR вашого модуля: підтвердження замовлення до 20 рядків має вкладатися в p95 ≤ 2 с і p99 ≤ 5 с. Це може вимагати обережного бюджету часу на послідовні виклики до Inventory.

3. **Зафіксувати поведінку підписок та черг.**
   Для кожної вхідної події зробити таблицю «подія → обробник → зміна замовлення → дублікат → подія не за порядком». Особливо розглянути `inventory.stock.reserved`, `inventory.stock.reservation_failed`, `inventory.shipment.dispatched`, `billing.invoice.issued` і `billing.payment.received`.

   Додати RabbitMQ-топологію Orders: основну чергу, retry-чергу, DLQ і прив’язки за routing key. Описати, хто розбирає DLQ і як виправлене повідомлення повертається в обробку. Для однієї події, наприклад `inventory.stock.reserved`, показати таблицю `processed_messages` і псевдокод: бізнес-зміна та запис ID повідомлення відбуваються в одній транзакції, а `ack` — після `COMMIT`.

4. **Окремо розв’язати складні місця саме вашого модуля.**
   - У ЛР2 описано резервування через Inventory й стан `on_hold` за його недоступності; у ЛР3 потрібно доповнити це ідемпотентним запитом, повторами й гарантованим зняттям резерву за потреби.
   - Перевірити реакцію автомата на події не за порядком і дублікати. Orders не має припускати глобального порядку подій між різними замовленнями.
   - Спроєктувати роботу з НБУ: кеш курсу за датою, курс останнього робочого дня для вихідних і свят, а для майбутньої дати — окрема обробка порожньої відповіді `[]`.
   - Для недоступності НБУ протягом 10 хвилин оформити ADR з альтернативами та наслідками. Урахувати ваше рішення з ЛР2 про незмінний знімок курсу й нову версію пропозиції після її прострочення.

5. **Додати специфікації стійкості у форматі Given / When / Then.**
   Мінімум три сценарії: дубль події змінює стан лише один раз; «отруйна» подія після п’яти невдалих спроб потрапляє в DLQ; повтор HTTP-команди з тим самим ключем не дублює дію. Для HTTP також описати випадок, коли той самий ключ надіслали з іншим тілом.

6. **Пройти уявний прогін Lead-to-Cash як команда Orders.**
   За сценарієм ЛР3 пройти угоду `won`, створення чернетки, рядки замовлення, погодження знижки, резерв, відвантаження, рахунок і оплату. Перевірити поля, типи, ID, помилки й `correlationId`; змоделювати дубль події про резерв та недоступність НБУ. Результат — Mermaid-діаграма послідовності й таблиця розбіжностей із рішенням або відкритим питанням.

## Важливе питання для узгодження

У ваших напрацюваннях є потенційна розбіжність: у ЛР3 для інтеграційного прогону задано знижку **12% із переходом на погодження**, а в документі ЛР2 написано, що **12% проходить без керівника**, а погодження починається вище 12%. Це потрібно явно узгодити в документі ЛР3, перш ніж контрактувати сценарій `confirm`.

## Що здати

Один документ `lab-03-orders-lysenko.md` зі вступними даними з ЛР1–ЛР2, OpenAPI й AsyncAPI, таблицями стиків і підписок, RabbitMQ та ідемпотентним споживачем, сценаріями стійкості, інтеграцією з НБУ та ADR, прогоном Lead-to-Cash, відкритими питаннями й самоперевіркою. Контракти можна додати окремими файлами `orders.openapi.yaml` і `orders.asyncapi.yaml`. Код писати не потрібно.

**Рекомендований порядок:** спершу уточнити правило знижки, потім погодити контракти з командами-споживачами, описати стійкість і інтеграцію з НБУ, а тоді провести прогін і зібрати документ.

> Чи можна визначити суть даної роботи, так само як це робив мій виклдач, коли я уточнював завдання для лабораторної?
>
> Для крщого розуміння:&#x20;
>
> - ЛР№2 він описав суть таким чином: "нам потрібно "поспілкуватися" з бізнесом, зібрати основні дані бізнесу, який звіт він буде робити та спроектувати CRM-систему. Завдання потрібно виконати найменшими змінами у вже існуючій системі бізнесу. Потім можна розробити програмний модуль для вирішення даної задачі й створити архітектурну схему даного рішення. Для створеного модуля/плагіна варто користуватися існуючою документацією, наприклад: Outlook, Word365 тощо. Те що ми спроеткуємо має відповідати документації обраної CRM."
> - ЛР№1 він описав: "потрібно ознайомитися з бізнесом, визначити основних сетйкхолдерів, визначити їхні болі, вибрати 1 проблему та розкрити її: задавати питання, розглянути потенційні рішення та провести самоаналіз, адже жодні рішення не ідеальні"

Так. **Суть ЛР3** можна сформулювати так:

> Потрібно спроєктувати, як ваш модуль **Orders взаємодіє з іншими модулями та зовнішніми сервісами**, щоб разом вони працювали як одна система. Для цього треба домовитися про формати запитів, відповідей і подій, визначити, що станеться при повторах, затримках і збоях, а потім пройти наскрізний сценарій замовлення — від виграної угоди до оплати — і знайти неузгодженості до початку реалізації.

У цій роботі ви не пишете код. Ви **перетворюєте рішення з ЛР2 на конкретні інтеграційні домовленості**: що Orders надсилає й отримує, як реагує на помилки, як не обробляє ту саму дію двічі та що робить, коли недоступний НБУ чи інший партнер. Потім перевіряєте ці домовленості на повному сценарії Lead-to-Cash.

Якщо зовсім коротко: **ЛР1 — розібратися в бізнес-проблемі; ЛР2 — спроєктувати свій модуль; ЛР3 — спроєктувати, як цей модуль надійно працює разом з рештою системи.**

> Так як суть вже визначена, який план виконання даної роботи?

Для вашого модуля **Orders** план ЛР3 можна побудувати так:

1. **Зібрати вихідні рішення з ЛР1–ЛР2.** Зафіксувати призначення Orders, його стани, API, події, правила резервування й валютну політику. Окремо узгодити суперечність щодо знижки 12%: у сценарії ЛР3 вона потребує погодження, а в документі ЛР2 — ні.

2. **Описати контракти модуля.** Підготувати OpenAPI для REST-команд Orders і AsyncAPI для подій `orders.order.confirmed` та `orders.order.cancelled`. Визначити формати помилок, ключі ідемпотентності, передавання `correlationId` і правила версіонування.

3. **Розписати взаємодію з іншими модулями.** Для кожного вихідного виклику й вхідної події вказати тайм-аути, повтори, ідемпотентність і поведінку за недоступності партнера. Перевірити, як Orders реагує на дублікати та події не за порядком.

4. **Спроєктувати обробку подій.** Описати черги RabbitMQ для Orders, повторні спроби та DLQ. На прикладі однієї події показати, як споживач не допускає повторної бізнес-зміни й підтверджує повідомлення брокеру лише після збереження змін.

5. **Продумати інтеграцію з НБУ.** Визначити, як Orders отримує й кешує курс EUR, обробляє вихідні дні, майбутні дати й тимчасову недоступність сервісу. Оформити це рішення як ADR з альтернативами та наслідками.

6. **Описати сценарії стійкості.** Скласти Given / When / Then для дубліката події, повідомлення, що після повторів потрапляє в DLQ, і повтору HTTP-запиту з тим самим ключем ідемпотентності.

7. **Провести уявний прогін Lead-to-Cash.** Пройти сценарій від виграної угоди до оплати, де Orders взаємодіє з CRM, Inventory, Billing і НБУ. Записати розбіжності контрактів, запропоновані зміни та рішення команд.

8. **Зібрати й перевірити документ.** Оформити `lab-03-orders-lysenko.md` із контрактами, таблицями, діаграмами, ADR, сценаріями, відкритими питаннями та самоперевіркою. Додати посилання на діалог або його експорт.

**Послідовність роботи:** спочатку узгодити суперечливі бізнес-правила, потім підготувати контракти й політики взаємодії, а вже на них провести інтеграційний прогін. Код для ЛР3 не потрібен.

> Давай почнемо виконання покроково з пункту 1. Зразу кажу -- в результаті роботи в другій лабораторній роботі знижка більше 10% потребує погодження. Це видно в твоїй останній відповіді в розділі 7. Доменна модель.

<details><summary>2 previous messages</summary>

> Починаю з вихідних рішень Orders і враховую ваше уточнення: **знижка понад 10% потребує погодження**. Звірю це з документом ЛР2 і зберу коротку базу рішень, на які спиратимемося далі.
>
> <details><summary>Ran a command</summary>
>
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' | Select-Object -First 150; Get-Content 'corporate-software-systems-lessons\lab-03.md' | Select-Object -Skip 150 -First 115`
>
> </details>

</details>

Пункт 1 — собрать исходные решения — почти готов. Я звірив доступні файли проєкту й врахував ваше уточнення про поріг знижки.

### Робоча база для ЛР3

- **Ваш модуль — Orders.** Він володіє замовленнями, позиціями, цінами та погодженням знижок. Залишками й резервами володіє Inventory, бухгалтерськими рахунками й оплатами — Odoo/Billing.
- **Знижка понад 10% потребує погодження керівника.** Беру це за чинне рішення з вашої фінальної ЛР2. Тоді знижка 12% у сценарії ЛР3 має перейти на погодження.
- **Курс EUR:** отримується з НБУ й фіксується у версії пропозиції разом із датою та джерелом. Погоджений валютний знімок не змінюється; для нової версії пропозиції отримується новий курс.
- **Резервування:** Inventory — джерело істини щодо залишків. Резерв має бути повним; якщо Inventory недоступний, Orders переходить у `on_hold` без активного резерву й повторює спробу після відновлення. Строк резерву — 24 години від фактичного резервування.
- **Події Orders:** публікує `orders.order.confirmed` і `orders.order.cancelled`. Споживає події CRM, Inventory та Billing для створення чернетки, оновлення реплік і переходів замовлення.
- **Надійність:** небезпечні POST-команди захищені `Idempotency-Key`; HTTP-виклики мають обмежені тайм-аути й повтори лише для тимчасових помилок; події обробляються ідемпотентно, з retry та DLQ.
- **Зовнішня інтеграція ЛР3:** для Orders обов’язково спроєктувати роботу з API НБУ, зокрема кеш курсу за датою й поведінку за недоступності.

### Знайшлися дві розбіжності в матеріалах

1. Доступний мені файл [lab-02-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md) містить поріг **понад 12%**, а ви вказуєте, що фінальне рішення — **понад 10%**. Для нашої роботи застосовую ваше уточнення **понад 10%**.
2. README описує компанію «КавоХаб» — дистриб’ютора кави, а доступний документ ЛР2 починається зі сценарію «ТехноПостач Сервіс» і містить CRM/Office-рішення. Це впливає на бізнес-контекст і сумісність із базовими контрактами ЛР3.

Тому пункт 1 можна зафіксувати з правилом знижки **понад 10%**, але перед контрактами треба визначити, який бізнес-контекст вважати основним: **КавоХаб із README** чи **ТехноПостач Сервіс із документа ЛР2**.

> Це одруківка в самому фінальному результаті, я із самого почтку працював з "КавоХаб". Можеш виправити файл [lab-02-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md)? Бо здається тут сталося непорозуміння в минулому чаті

<details><summary>2 previous messages</summary>

> Так, виправлю фінальний документ ЛР2: поверну його до контексту «КавоХаб» і узгоджу поріг погодження знижки з вашим рішенням — понад 10%. Спершу знайду всі згадки чужої компанії та пов’язаних із нею CRM/Office-рішень, щоб виправити зміст послідовно, а не лише назву.
>
> <details><summary>Edited a file, ran 8 commands</summary>
>
> - Ran `rg -n 'ТехноПостач|Outlook|Word|Office Add|Microsoft|M365|Graph|CRM|знижк|12%|понад 12|10%|12\b|Telegram|лідер|угод|proposal|DOCX|PDF|Orders|KavoHub|КавоХаб' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md'`
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' | Select-Object -Skip 145 -First 130`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+7 -7)
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' | Select-Object -Skip 24 -First 55`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+10 -12)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+18 -22)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+6 -6)
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' | Select-Object -Skip 220 -First 55`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+5 -6)
> - Ran `rg -n 'ТехноПостач|Outlook|Word|Office Add|Microsoft|M365|Graph|12%|понад 12|12,01|понад 10|Telegram|Activity|Document|ліди|воронк|програш|weekly-sales|documents/quote' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md'`
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' | Select-Object -Skip 95 -First 38; rg -n 'Odoo|CRM Backend|CRM/Orders|Account|Lead|Deal|Office|M365|частков|partially_reserved|знижк' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md'`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+9 -11)
> - Ran `rg -n 'ТехноПостач|Outlook|Word|Office Add|Microsoft|M365|Graph|понад 12|12% не|12,01|Telegram|щотижневий звіт|Account, Contact, Lead|адаптер Odoo|CRM Backend|partially_reserved|надбудови|Odoo — облік|понад 10|знижка 10%' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md'; git diff -- 'corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md'`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+2 -5)
> - Ran `rg -n 'ТехноПостач|Outlook|Word|Office Add|Microsoft|M365|Graph|понад 12|12% не|12,01|Telegram|Activity|Document|щотижневий звіт із продажів|CRM/Orders|partially_reserved|AccountReference|Погодження знижок понад|Операційний звіт|НБУ|pending_approval|orders.order.confirmed' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md'; Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' | Select-Object -First 135`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md` (+3 -3)
>
> </details>

</details>

Виправив [lab-02-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md).

Повернув документ до контексту **«КавоХаб» і модуля Orders**: прибрав сторонні сценарії з Outlook, Word і Microsoft 365, уточнив межі Orders, CRM, Inventory та Billing/Odoo. Поріг погодження тепер **понад 10%**. Також узгодив опис резервування з цільовою схемою: під час погодження резерв утримується до 24 годин; після схвалення Inventory підтверджує наявний резерв, не створюючи дубль.

Звірив документ на залишкові згадки чужого сценарію й старого порога знижки. Кодових перевірок не проводив — це виправлення проєктного документа.

> Додай в кінець логів чату lab-02.md (в папці result) запит на правку [lab-02-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md)

<details><summary>2 previous messages</summary>

> Знайду файл логу ЛР2 у `result` і додам наприкінці короткий запис із запитом на виправлення фінального документа.
>
> <details><summary>Edited a file, ran 3 commands</summary>
>
> - Ran `rg --files 'corporate-software-systems-lessons\result'`
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02.md' -Tail 55`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02.md` (+10 -0)
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02.md' -Tail 18`
>
> </details>

</details>

Додав запис про ваш запит і виправлення фінального документа в кінець [lab-02.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02.md).

> Тепер можемо працювати над ЛА№3. Виконуємо крок 1:

<details><summary>2 previous messages</summary>

> Так, переходимо до кроку 1. Зафіксую вихідні рішення Orders із «КавоХаб», вашої виправленої ЛР2 та спільних правил проєкту; окремо позначу питання, які ще треба вирішити, щоб не закладати суперечності в контракти.
>
> <details><summary>Ran a command</summary>
>
> - Ran `Get-Content 'corporate-software-systems-lessons\lab-01.md' | Select-Object -Skip 99 -First 70; Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-02-orders-lysenko.md' | Select-Object -Skip 115 -First 40; Get-Content 'corporate-software-systems-lessons\README.md' | Select-Object -Skip 55 -First 85`
>
> </details>

</details>

## Крок 1 завершено: вихідна база для ЛР3

Основний контекст — **«КавоХаб», модуль Orders**. Надалі спиратимемося на загальні правила проєкту, вашу виправлену ЛР2 та вимоги ЛР3: [README.md](./corporate-software-systems-lessons/README.md), [lab-01.md](./corporate-software-systems-lessons/lab-01.md), [lab-02-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-02-orders-lysenko.md) і [lab-03.md](./corporate-software-systems-lessons/lab-03.md).

**Рішення, які беремо в роботу:**

- Orders володіє замовленнями, позиціями, контрактними цінами та погодженнями знижок. CRM володіє клієнтами й угодами; Inventory — товарами, залишками, резервами й відвантаженнями; Billing — рахунками й оплатами, Odoo — бухгалтерськими проведеннями.
- **Знижка понад 10% потребує погодження керівника продажів.** Отже, знижка 12% у сценарії інтеграційного прогону переходить на погодження.
- На час погодження знижки товар резервується не довше ніж на **24 години**. Після відхилення або спливу строку резерв треба звільнити. Після схвалення Inventory підтверджує вже створений резерв, не створюючи його вдруге.
- Inventory резервує всі рядки замовлення або відхиляє резерв повністю; часткового резервування немає. Orders не є джерелом істини щодо залишків.
- Для EUR Orders отримує курс із НБУ та зберігає його у версії пропозиції. Затверджений валютний знімок не змінюється; нова версія пропозиції отримує власний знімок курсу.
- Orders публікує `orders.order.confirmed` і `orders.order.cancelled`. Він споживає події CRM, Inventory та Billing, зокрема `crm.deal.won`, `inventory.stock.reserved`, `inventory.stock.reservation_failed`, `inventory.shipment.dispatched`, `billing.invoice.issued` і `billing.payment.received`.
- Небезпечні POST-команди захищаються `Idempotency-Key`; через HTTP і події проходить `correlationId`. Дублікати не повинні повторно змінювати стан замовлення.
- Підтвердження замовлення до 20 рядків має відповідати NFR ЛР1: p95 ≤ 2 с, p99 ≤ 5 с, частка помилок ≤ 0,5%. Це обмежує час, який можна витратити на синхронні виклики під час підтвердження.

**Що ще треба спроєктувати в наступних кроках:** точні схеми запитів і подій, тайм-аути й повтори для викликів Inventory та CRM, правила Orders для подій не за порядком, черги retry/DLQ, обробку недоступності НБУ та ADR. Це вже частина контрактів і рішень ЛР3, а не невирішена плутанина щодо модуля чи бізнесу.

> Тепер перейдемо до кроку 2

<details><summary>2 previous messages</summary>

> Для кроку 2 оформлю чернетки контрактів Orders окремими YAML-файлами. Включу команди з ЛР2, Problem Details, `Idempotency-Key`, `X-Correlation-Id`, ролі та події Orders з конвертом README. Якщо для `orders.order.cancelled` не задано точний payload, позначу поля як проєктне узгодження, а не як уже затверджений контракт.
>
> <details><summary>Created 2 files, edited 2 files, ran 3 commands</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+474 -0)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.asyncapi.yaml` (+154 -0)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+11 -3)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.asyncapi.yaml` (+6 -1)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+5 -0)
> - Ran `rg -n -B5 -A4 'IdempotencyKey|IfMatch|operationId' 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml'`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+1 -1)
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml' | Select-Object -Skip 25 -First 14; Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml' | Select-Object -Skip 258 -First 17`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.asyncapi.yaml` (+3 -2)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+6 -0)
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml' | Select-Object -Skip 250 -First 30; Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.asyncapi.yaml' | Select-Object -First 55; rg -n 'x-required-roles|submit-approval|confirmOrder|Idempotency|dataschema|orderVersion' 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml' 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.asyncapi.yaml'`
>
> </details>

</details>

## Крок 2: чернетки контрактів Orders

Створив два YAML-файли:

- [orders.openapi.yaml](./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml) — маршрути замовлень, погодження знижок, звіт; ролі, `If-Match`, `Idempotency-Key`, `X-Correlation-Id` і помилки Problem Details.
- [orders.asyncapi.yaml](./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.asyncapi.yaml) — події `orders.order.confirmed` та `orders.order.cancelled` у конверті з README.

Попередня перевірка як споживачами: Inventory отримує в події підтвердження потрібні `orderId`, склад і рядки `sku/qty`; CRM — `accountId`; Billing може отримати рядки й ціни через `GET /orders/{orderId}`. У події скасування передбачено причину та ID резерву, якщо він відомий.

У контрактах запропонував додати `orderVersion`, щоб споживачі могли відкидати застарілі події. Це пропозиція до узгодження з Inventory, CRM і Billing. Для лінтингу пропоную вимагати унікальний `operationId`, Problem Details для 4xx, `Idempotency-Key` на командах, заголовок кореляції та обов’язкові поля конверта подій. Для роботи до реалізації OpenAPI-контракт можна під’єднати до Prism mock-сервера.

Є один момент, який варто узгодити перед фіналізацією: у ЛР2 є і `POST /submit-approval`, і `POST /confirm`; обидва можуть привести замовлення до погодження. У чернетці вони лишені, але їхні ролі в процесі треба остаточно визначити. **Пропоную:** `confirm` автоматично запускає погодження, якщо знижка понад 10%, а `submit-approval` лишається явною командою для окремого етапу подання на погодження.

> Так, давай так і зробимо і перейдемо до кроку 3

<details><summary>1 previous message</summary>

> <details><summary>Created a file, edited a file, ran a command</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+70 -0)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+6 -2)
> - Ran ``rg -n '^#|^##|^###|^\| Orders →|^\| `|openapi:|asyncapi:|approval|cancellationStatus|IfMatch' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md' 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml' 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.asyncapi.yaml'``
>
> </details>

</details>

Зафіксував погоджену семантику: `confirm` автоматично запускає погодження для знижки понад 10%, а `submit-approval` — явна команда; обидва шляхи ведуть до того самого переходу без дублювання заявки.

**Крок 3 опрацьовано** у робочому документі [lab-03-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md). Там є таблиця вихідних викликів Orders → Inventory, CRM і НБУ, а також підписок на події CRM, Inventory та Billing із правилами для дублікатів і подій не за порядком.

Політики враховують NFR підтвердження: перевірка доступності не вважається резервом, а довге очікування Inventory переходить в асинхронну обробку з відповіддю 202. Також описано передачу `X-Correlation-Id`, ідемпотентні повтори та поведінку під час недоступності партнерів.

Зафіксував відкриті контрактні залежності: версію Account від CRM; поля подій Inventory для прив’язки до замовлення й спроби резерву; `orderId` та `invoiceId` у подіях Billing. Пропозиції до власних контрактів Orders — `orderVersion` і стан очікування звільнення резерву — теж позначені для погодження.

Контракти лежать окремо: [OpenAPI](./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml) та [AsyncAPI](./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.asyncapi.yaml).

> Виконаємо крок 4

<details><summary>2 previous messages</summary>

> Для кроку 4 зафіксую топологію Orders у RabbitMQ, обробку `inventory.stock.reserved` в одній транзакції з дедуплікацією та процедуру розбору DLQ. Додам це до робочого документа ЛР3, щоб артефакти накопичувалися в одному місці.
>
> <details><summary>Edited a file</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+73 -0)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+2 -2)
>
> </details>

</details>

Крок 4 додано до [робочого документа ЛР3](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md).

У документі описано топологію RabbitMQ для Orders: durable topic exchange, quorum-чергу споживача, retry-чергу із затримкою 5 секунд і DLQ після п’яти невдалих спроб. Для `inventory.stock.reserved` додано схему `processed_messages` і псевдокод: запис дедуплікації та зміна замовлення виконуються в одній транзакції, `ack` надсилається після `COMMIT`.

Також зафіксував ручний розбір DLQ: точне повідомлення можна повторити після виправлення тимчасової причини; якщо помилка в самому payload, його виправляє власник події та публікує нову подію з новим ID. Код і тести не запускав — це проєктний опис.

> Давай зразу виконаємо кроки 5-6

<details><summary>2 previous messages</summary>

> Виконаю обидва кроки: додам сценарії стійкості Given/When/Then і спроєктую інтеграцію Orders із НБУ — поле-мапінг, помилки, поведінку при 10-хвилинній недоступності та ADR з альтернативами. Перевірю формат відповіді за офіційною документацією НБУ, щоб контракт не спирався на здогад.
>
> <details><summary>Edited 2 files, ran 3 commands, searched the web</summary>
>
> Searched the web for `site:bank.gov.ua/NBUStatService/v1/statdirectory/exchange json valcode date r030 txt rate exchangedate`
>
> Searched the web for `https://bank.gov.ua/NBUStatService/v1/statdirectory/exchange?valcode=EUR&date=20261116&json`
>
> Searched the web for `site:bank.gov.ua "NBUStatService/v1/statdirectory/exchange?valcode" ...`
>
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+1 -0)
> - Ran `rg -n -B4 -A8 'rateRequestedDate|exchangeRate|rateDate' 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml'`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+99 -0)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+7 -5)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+18 -1)
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml' | Select-Object -First 125; Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md' | Select-Object -Skip 85 -First 140`
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md' | Select-Object -Skip 150 -First 80; Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\orders.openapi.yaml' | Select-Object -Skip 98 -First 30`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+12 -0)
>
> </details>

</details>

Кроки 5–6 додані до [робочого документа ЛР3](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md).

- Додав Given/When/Then для дубліката події, отруйного повідомлення й DLQ, повторів HTTP-команд, падіння після `COMMIT` до `ack`, а також двох сценаріїв недоступності НБУ — з точним курсом у кеші й без нього.
- Для НБУ описав мапінг полів, кеш за запитаною датою, обробку вихідних і майбутніх дат, Mermaid-діаграму та ADR з трьома альтернативами. Рішення: брати з кешу лише курс для потрібної дати; без нього не фіксувати нову EUR-ціну. Офіційна інструкція НБУ підтверджує формат дати й поля JSON-відповіді, зокрема `rate`, `cc` та `exchangedate` ([інструкція НБУ](https://finliteracy.bank.gov.ua/admin_uploads/article/Instr_API_KURS_VAL_Full_eng.pdf)).
- Оновив [OpenAPI-контракт](./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml): додав запитану дату курсу та відповіді `503` для недоступності НБУ без точного кешу.

Це специфікації сценаріїв, а не результати запуску тестів чи лінтера.

> З урахуванням попередніх кроків, виконаємо пункт 7

<details><summary>2 previous messages</summary>

> Проведу прогін за сценарієм ЛР3 від `crm.deal.won` до `billing.payment.received`, включно з дублем резерву та недоступністю НБУ. У фіналі зафіксую, які поля й рішення сумісні, а що потрібно погодити з командами CRM, Inventory та Billing до реалізації.
>
> <details><summary>Edited 2 files, ran a command</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+3 -3)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+75 -0)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml` (+8 -0)
> - Ran `rg -n 'currency=EUR|Джерельна валюта|\| 2\. Додати' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md'`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+1 -1)
>
> </details>

</details>

Крок 7 доданий до [робочого документа ЛР3](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md): там є діаграма Lead-to-Cash і таблиця розбіжностей із запропонованими змінами та симульованими реакціями команд.

У прогоні знижка 12% запускає погодження й тимчасовий резерв. Після схвалення Inventory підтверджує наявний резерв, замовлення відвантажується, Billing виставляє рахунок, а після повної оплати Orders переходить у `paid`. Повторна доставка події резерву не змінює стан. Коли НБУ недоступний 10 хвилин, Orders використовує кеш курсу для тієї самої дати.

Зафіксував потребу погодити payload-поля між модулями: ключі угоди від CRM; ID замовлення й резерву від Inventory; зв’язок рахунку та повної/часткової оплати із замовленням від Billing. Також оновив [OpenAPI-контракт](./corporate-software-systems-lessons/result/lysenko-oleksandr/orders.openapi.yaml), додавши до відповіді позиції джерельну суму й валюту для аудиту конвертації.

Формат і поля відповіді НБУ звірив з [офіційною інструкцією НБУ](https://finliteracy.bank.gov.ua/admin_uploads/article/Instr_API_KURS_VAL_Full_eng.pdf). Узгодження команд у таблиці є симуляцією для лабораторної; фактичні контракти інших модулів залишаються пропозиціями до погодження.

> \`\`\`mermaid\
> sequenceDiagram\
> &#x20; actor M as Менеджер продажів\
> &#x20; actor H as Керівник продажів\
> &#x20; participant CRM as CRM\
> &#x20; participant MQ as RabbitMQ\
> &#x20; participant ORD as Orders\
> &#x20; participant CACHE as Rate cache\
> &#x20; participant NBU as API НБУ\
> &#x20; participant INV as Inventory\
> &#x20; participant BILL as Billing\
> &#x20; participant ODOO as Odoo\
> &#x20; participant PAY as Платіжний провайдер\
> \
> &#x20; CRM->>MQ: crm.deal.won {dealId, accountId, ownerId}\
> &#x20; MQ->>ORD: crm.deal.won\
> &#x20; ORD->>ORD: Ідемпотентно створити чернетку для ACC-000587\
> &#x20; M->>ORD: Додати рядки; знижка 12%; POST confirm (Idempotency-Key, If-Match)\
> &#x20; ORD->>NBU: GET EUR rate за rateRequestedDate\
> &#x20; NBU--xORD: Timeout / сервіс недоступний\
> &#x20; ORD->>CACHE: Знайти курс EUR для тієї самої запитаної дати\
> &#x20; CACHE-->>ORD: Кешований rate, rateDate, fetchedAt\
> &#x20; ORD->>ORD: Зафіксувати ціновий знімок; відкрити pending_approval\
> &#x20; ORD->>INV: POST /reservations (той самий ключ для повторів)\
> &#x20; INV-->>ORD: Повний тимчасовий резерв R1, expiresAt ≤ 24 год\
> &#x20; INV->>MQ: inventory.stock.reserved (event id E1)\
> &#x20; MQ->>ORD: inventory.stock.reserved E1\
> &#x20; MQ->>ORD: Повторна доставка inventory.stock.reserved E1\
> &#x20; ORD->>ORD: Перша доставка зберігає R1; дубль відсікає processed_messages\
> &#x20; ORD-->>M: 202 Pending approval; резерв утримується до 24 год\
> &#x20; H->>ORD: POST /approve (If-Match, Idempotency-Key)\
> &#x20; ORD->>ORD: approvalStatus=approved; наявний резерв лишається прив’язаним\
> &#x20; ORD->>MQ: orders.order.confirmed {orderId, accountId, warehouse, lines, orderVersion, reservationId}\
> &#x20; MQ->>INV: orders.order.confirmed\
> &#x20; INV->>INV: Підтвердити R1 для замовлення; нового резерву не створювати\
> &#x20; INV->>MQ: inventory.shipment.dispatched {orderId, shipmentId, trackingNumber, dispatchedAt}\
> &#x20; par Споживачі відвантаження\
> &#x20; MQ->>ORD: inventory.shipment.dispatched\
> &#x20; ORD->>ORD: Зафіксувати shipped і дані ТТН\
> &#x20; and\
> &#x20; MQ->>BILL: inventory.shipment.dispatched\
> &#x20; BILL->>ORD: GET /orders/ORD-2026-004213\
> &#x20; ORD-->>BILL: Рядки, клієнт, ціни, валюта та знімок курсу\
> &#x20; BILL->>ODOO: Створити й провести рахунок\
> &#x20; ODOO-->>BILL: Проведений рахунок і номер\
> &#x20; BILL->>MQ: billing.invoice.issued {orderId, invoiceId, invoiceNumber, amount, currency}\
> &#x20; end\
> &#x20; MQ->>ORD: billing.invoice.issued\
> &#x20; ORD->>ORD: Зафіксувати invoiceId та стан invoiced\
> &#x20; M->>PAY: Оплатити рахунок\
> &#x20; PAY->>BILL: Webhook платежу\
> &#x20; BILL-->>PAY: 2xx після перевірки підпису й прийняття події\
> &#x20; BILL->>MQ: billing.payment.received {orderId, invoiceId, paymentId, invoiceStatus=paid}\
> &#x20; MQ->>ORD: billing.payment.received\
> &#x20; ORD->>ORD: Зафіксувати платіж; статус paid\
> \`\`\`
>
> Проблема з рендерингом діаграми:
> Parse error on line 17:\
> ...ти рядки; знижка 12%; POST confirm (Idem\
> \-----------------------^\
> Expecting '()', 'SOLID_OPEN_ARROW', 'DOTTED_OPEN_ARROW', 'SOLID_ARROW', 'SOLID_ARROW_TOP', 'SOLID_ARROW_BOTTOM', 'STICK_ARROW_TOP', 'STICK_ARROW_BOTTOM', 'SOLID_ARROW_TOP_DOTTED', 'SOLID_ARROW_BOTTOM_DOTTED', 'STICK_ARROW_TOP_DOTTED', 'STICK_ARROW_BOTTOM_DOTTED', 'SOLID_ARROW_TOP_REVERSE', 'SOLID_ARROW_BOTTOM_REVERSE', 'STICK_ARROW_TOP_REVERSE', 'STICK_ARROW_BOTTOM_REVERSE', 'SOLID_ARROW_TOP_REVERSE_DOTTED', 'SOLID_ARROW_BOTTOM_REVERSE_DOTTED', 'STICK_ARROW_TOP_REVERSE_DOTTED', 'STICK_ARROW_BOTTOM_REVERSE_DOTTED', 'BIDIRECTIONAL_SOLID_ARROW', 'DOTTED_ARROW', 'BIDIRECTIONAL_DOTTED_ARROW', 'SOLID_CROSS', 'DOTTED_CROSS', 'SOLID_POINT', 'DOTTED_POINT', got 'NEWLINE'

<details><summary>2 previous messages</summary>

> Знайшов причину в Mermaid-блоці: у підписах стрілок є крапки з комою, які Mermaid трактує як розділювачі команд. Заміню їх на звичайні розділові знаки, залишивши зміст повідомлень незмінним.
>
> <details><summary>Edited a file, ran 2 commands</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+5 -5)
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md' | Select-Object -Skip 278 -First 55; rg -n '^```mermaid|^  .*;|^```$' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md'`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+4 -4)
> - Ran `rg -n '^  .*;|^    .*;' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md'`
>
> </details>

</details>

Виправив Mermaid-діаграми в [lab-03-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md): прибрав крапки з комою з підписів стрілок — Mermaid сприймав їх як розділювачі команд. Перевірив, що в підписах діаграм більше не залишилося таких розділювачів.

> Модифікуй діаграму для кращого перегляду. Через її великий розмір її важко переглядати. За можливості розділи її на менші діаграми і залиш велику для кращого розуміння

<details><summary>2 previous messages</summary>

> Залишу повну діаграму як огляд наскрізного процесу й додам три компактні діаграми: погодження та резерв, відвантаження й рахунок, оплата й повторна доставка події. Так буде простіше роздивитися кожен етап окремо, не втрачаючи загальної картини.
>
> <details><summary>Edited a file, ran a command</summary>
>
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+87 -1)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+2 -2)
> - Ran `rg -n '^### 7\.|^#### Етапи|^```mermaid|^```$|^  .*;' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md'`
>
> </details>

</details>

Оновив [діаграму в документі ЛР3](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md): залишив повну схему Lead-to-Cash і додав три менші діаграми за етапами:

- чернетка, курс, погодження та резерв;
- відвантаження й виставлення рахунку;
- оплата й повторна доставка події.

Так можна переглядати деталі кожного етапу окремо, а повну схему використовувати для огляду всього процесу.

> Тепер виконаємо крок 8 та проведемо самоаналіз

<details><summary>2 previous messages</summary>

> Зберу попередні кроки в цілісний документ ЛР3, звірю його з переліком здачі та критеріями оцінювання, додам журнал рішень, припущення, відкриті питання й самооцінку. Наприкінці окремо позначу, що вже покрито і що ще потребує підтвердження інших команд.
>
> <details><summary>Edited a file, ran 4 commands</summary>
>
> - Ran `rg -n '3\.5|6\.2|самоперевір|журнал рішень|Що здати|Критерії оцінювання|Контрольні питання|робота з ШІ' 'corporate-software-systems-lessons\README.md' 'corporate-software-systems-lessons\lab-03.md'; Get-Content 'corporate-software-systems-lessons\lab-03.md' | Select-Object -Skip 235 -First 85; Get-Content 'corporate-software-systems-lessons\README.md' | Select-Object -Skip 275 -First 95`
> - Ran `Get-Content 'corporate-software-systems-lessons\README.md' | Select-Object -Skip 230 -First 20; Get-Content 'corporate-software-systems-lessons\lab-03.md' | Select-Object -Skip 330 -First 37; Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md' | Select-Object -First 75`
> - Ran `Get-Content 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md' | Select-Object -First 25; Get-Content 'corporate-software-systems-lessons\lab-03.md' | Select-Object -Skip 329 -First 28`
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+43 -1)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+4 -1)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+67 -0)
> - Wrote `./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md` (+9 -6)
> - Ran `rg -n '^#|^##|^###|^####|^\| [A-ZА-Я]|^- ' 'corporate-software-systems-lessons\result\lysenko-oleksandr\lab-03-orders-lysenko.md'`
>
> </details>

</details>

Зібрав крок 8 у [lab-03-orders-lysenko.md](./corporate-software-systems-lessons/result/lysenko-oleksandr/lab-03-orders-lysenko.md). Додав схему зберігання `Idempotency-Key`, журнал рішень, припущення, відкриті питання, самооцінку за критеріями й чотири приклади того, як ваші уточнення змінили роботу.

У самооцінці синхронна взаємодія та контракти позначені як **частково виконані**: дизайн є, але треба погодити окремі поля й семантику з іншими командами та перевірити YAML лінтером. Події, стійкість і інтеграція з НБУ покриті проєктно; фактичного запуску не було. Офіційну оцінку не виставляв.

За вимогою захисту ставлю контрольні питання по одному. **Питання 1:** Чому Orders надсилає `ack` лише після `COMMIT`? Що станеться, якщо процес упаде між `COMMIT` і `ack`?
