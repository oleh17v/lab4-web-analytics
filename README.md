US-04. Cross-Product Purchase Patterns**

Цю задачу я розбирав як **research / analytics-only**. Тобто не як імплементацію, а як перевірку: чи можна побудувати story на поточному стеку, на яких джерелах, з якими обмеженнями, і якою має бути правильна модель даних для цього кейсу.

**Що саме я перевіряв**

Я дивився три шари.

1. **dbt-моделі**
- silver database:
  - [stg_database_sales_order_header.sql](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/dbt/models/silver/database/stg_database_sales_order_header.sql)
  - [stg_database_sales_order_detail.sql](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/dbt/models/silver/database/stg_database_sales_order_detail.sql)
- gold core:
  - [fct_purchase.sql](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/dbt/models/gold/core/facts/fct_purchase.sql)
  - [fct_view.sql](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/dbt/models/gold/core/facts/fct_view.sql)
  - [dim_product.sql](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/dbt/models/gold/core/dimensions/dim_product.sql)
- source definitions:
  - [dailyfiles sources.yml](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/dbt/models/silver/dailyfiles/sources.yml)
  - [realtime sources.yml](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/dbt/models/silver/realtime/sources.yml)

2. **pipeline / docs**
- [docs/normalization_dailyfiles_realtime.md](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/docs/normalization_dailyfiles_realtime.md)
- [docs/database_business_metrics.md](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/docs/database_business_metrics.md)
- [docs/db_silver_analytics_pipeline.md](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/docs/db_silver_analytics_pipeline.md)
- cleaning / normalization logic around purchase items:
  - [transformations.py](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/src/utils/cleaning/transformations.py)
  - [validation.py](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/src/utils/cleaning/validation.py)
  - [field_cleaners.py](/abs/path/C:/Users/LEGION/ZedCharmProjects/global-de/src/utils/cleaning/field_cleaners.py)

3. **живі дані**
- через Athena я перевіряв:
  - схеми таблиць
  - grain
  - row counts
  - distinct orders / distinct products
  - чи є `event_purchase_item`
  - чи є overlap між dailyfiles і realtime
  - часовий діапазон online item data

**Що я встановив по джерелах**

Основний висновок: story не треба будувати з `fct_purchase` напряму.

Чому:
- `fct_purchase` має grain: **1 row = 1 purchase/order**
- а для `Cross-Product Purchase Patterns` потрібен grain: **1 row = 1 product within basket/order**

Тобто для цієї задачі критичні не order-level facts, а **line items**.

Джерела, які реально підходять:

- **DB**
  - `stg_database_sales_order_header`
  - `stg_database_sales_order_detail`
  - `dim_product`

- **Online**
  - `db_dailyfiles.event_purchase`
  - `db_dailyfiles.event_purchase_item`
  - `db_realtime.event_purchase`
  - `db_realtime.event_purchase_item`

Це важливий момент: спочатку по `event_purchase` могло здатися, що online не придатний, бо там order-level shape. Але потім я підтвердив, що в normalized шарі є окрема таблиця **`event_purchase_item`**, і саме вона потрібна для basket/pair analysis.

**Що я побачив у даних**

По DB:
- `stg_database_sales_order_header`: **251,340 orders**
- `stg_database_sales_order_detail`: **752,985 line items**
- `295 distinct purchased products`

По online:
- `db_dailyfiles.event_purchase_item`: **196,974,095 rows**
- `db_realtime.event_purchase_item`: **5,818,673 rows**
- в обох потоках: **304 distinct products**

По online history:
- `event_purchase_item.created_at` починається приблизно з **2026-06-09**
- тобто **повної historical online basket history немає** до цієї дати

По overlap:
- між dailyfiles і realtime по `order_id` я не побачив прямого перетину:
  - `overlapping_orders = 0`

По source mapping:
- `source_sk = 1` -> realtime
- `source_sk = 2` -> dailyfiles
- `source_sk = 3` -> database

**Субтаска 1. General approach**

Story треба будувати так:

- спочатку зібрати **online purchase lines**
  - `event_purchase_item` + join до `event_purchase` для `order_id`, `event_date`, `source_sk`
- окремо зібрати **DB purchase lines**
  - `sales_order_detail` + `sales_order_header`
- привести обидва джерела до спільного grain:
  - `basket_id`
  - `source_sk`
  - `event_date`
  - `product_id`
  - `product_name`
  - `quantity`
  - `line_revenue`

Після цього вже рахувати co-purchase metrics:
- `orders_with_pair`
- `support`
- `confidence`
- `lift`
- `pair_revenue`
- `pair_units`

Технічно це означає, що в dbt треба робити не просто один gold fact, а новий ланцюжок:
- `int_online_purchase_items_enriched`
- `int_purchase_basket_lines`
- `mart_cross_product_pairs`
- `mart_cross_product_pairs_monthly`

**Субтаска 2. Points to pay attention to**

Тут основні ризики такі.

- **`fct_purchase` недостатній**
  - він не містить basket composition
  - для pair analysis цього мало

- **online history деградована по часу**
  - online item-level картина є не за весь історичний період
  - це обмежує full-history analysis по digital source

- **product taxonomy для online не ідеальна**
  - `dim_product` побудований з DB product source
  - для online category/subcategory canonical mapping не виглядає повністю оформленим
  - у `fct_view` category/subcategory є, але це вже поведінковий event layer, а не продуктова dimension

- **pair logic не можна рахувати “в лоб”**
  - якщо в одному order є 3 однакові SKU lines або quantity > 1, pair counts можуть роздуватися
  - для affinity-метрик треба рахувати **distinct product presence per order**

- **low-support outliers**
  - pair може мати високий `lift`, але бути статистично шумним
  - тому треба ставити minimum threshold, наприклад minimum order count per pair

- **DB order status**
  - треба явно вирішити scope:
  - чи виключати `Rejected` і `Cancelled`
  - я б рекомендував виключати `4` і `6`

**Субтаска 3. Story relevance**

Якщо оцінювати по суті story, а не по тексту AC, то задача **реалізовна**.

Що реально можна закрити зараз:
- які товари купують разом
- top product pairs
- cross-sell affinity
- pair support / confidence / lift
- порівняння DB vs online
- динаміка по часу

Що може не закритися повністю:
- якщо story вимагає **повний historical online scope**
- якщо story вимагає **строгу category/subcategory аналітику для online**
- якщо очікується, що все це вже є в поточному gold без нових dbt-моделей

Окремо: **точних acceptance criteria у репозиторії я не знайшов**. Тобто я можу оцінити feasibility дуже предметно, але не можу сказати “AC-3 закривається, AC-4 не закривається” без самого тексту AC.

**Субтаска 4. Views / tables / data needed**

Що потрібно використовувати:

- `stg_database_sales_order_header`
  - `sales_order_id`
  - `order_ts`
  - `order_status`
  - `customer_id`
- `stg_database_sales_order_detail`
  - `sales_order_id`
  - `product_id`
  - `order_qty`
  - `unit_price`
  - `line_total`
- `dim_product`
  - `product_id`
  - `name`
  - `product_subcategory_id`
  - `product_line`
  - `product_class`

Online:
- `event_purchase`
  - `purchase_sk`
  - `order_id`
  - `created_at`
- `event_purchase_item`
  - `purchase_sk`
  - `product_id_cleaned`
  - `product_name`
  - `quantity`
  - `unit_price`
  - `created_at`

Supporting:
- `fct_view`
  - якщо знадобиться enrichment по `category/subcategory/brand`

**Субтаска 5. Visualization format**

Найкращий формат для story:

- **Horizontal bar chart**
  - top 10/20 product pairs by `orders_with_pair`

- **Heatmap**
  - осі: продукти
  - cell: `lift` або `orders_with_pair`

- **Bubble chart**
  - `x = support`
  - `y = lift`
  - `size = pair_revenue` або `orders_with_pair`

- **Line chart**
  - trend selected pairs by month / week

- **Detail table**
  - pair-level metrics для analyst drilldown

Що я не рекомендую як основний формат:
- network graph
  - гарно виглядає, але швидко стає шумним і погано читається як основний artifact

**Підсумок по задачі**

Як research ця задача закрита таким висновком:

- data foundation для story **є**
- поточний **gold шар недостатній**
- story треба будувати **через нові dbt intermediate + mart моделі**
- DB частина виглядає сильно і стабільно
- online частина придатна, але має **historical gap** і потенційний taxonomy gap
- для production-quality story я б будував unified basket-lines mart, а не намагався витиснути це з `fct_purchase`

Якщо хочеш, я можу наступним повідомленням перетворити це в:
1. **готовий research note для Jira**, або
2. **готову Confluence-сторінку**, або
3. **implementation plan по dbt моделях** з grain, joins і колонками.
