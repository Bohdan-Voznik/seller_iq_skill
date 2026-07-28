# stock_snapshot

Знімок залишків товару (свого чи конкурента) в момент сканування. Ключ — `rozetka_id`. Дозволяє рахувати швидкість продажів і оцінювати конкурентів по залишках.

## Поля

`name`, `rozetka_id`, `rozetka_id_string`, `href`, `img`, `seller`, `top_label`, `wishlist_count`, `comments_amount`, `comments_mark`, `base_price`, `discount_price`, `promo_price`, `price`, `discount_percent`, `promo_percent`, `discount`, `schema_v`, `prev_stock_limit`, `stock_limit`, `sales_count`, `restock_count`, `scan_time`, `stock_change_type`, `scan_is_exact`, `scan_limit_reason`.

Останні п'ять полів (`schema_v`, `scan_time`, `stock_change_type`, `scan_is_exact`, `scan_limit_reason`) не завжди присутні в публічній документації властивостей — але реально відправляються в подію. Не дивуватись їм у даних.

## Логіка `sales_count` / `restock_count` / `stock_change_type`

Рахується порівнянням поточного залишку (`stock_limit`) з попереднім (`prev_stock_limit`) на момент сканування:

- залишок став `0` → `stock_change_type = out_of_stock`
- залишок не змінився → без події (`no_change` — така подія до Mixpanel не доходить)
- залишок виріс (`delta > 0`) → `stock_change_type = restock`, `restock_count = delta`
- залишок впав, але падіння **більше за реалістичний поріг продажів за одне сканування** (аномально великий стрибок вниз, схожий на ручне перезаливання складу, а не на продажі) → `stock_change_type = stock_decrease_adjustment`, `sales_count = 0` — це захист від хибного сплеску «продажів», який насправді є коригуванням залишку продавцем
- залишок впав у межах реалістичного порогу (`delta < 0`) → `stock_change_type = sale`, `sales_count = |delta|`
- попереднє або поточне значення не є числом → `stock_change_type = unknown` (теж не доходить до Mixpanel)

**Реальні значення `stock_change_type`, які трапляються в даних: `out_of_stock`, `restock`, `stock_decrease_adjustment`, `sale` — усі нижнім регістром.** Подія шлеться в Mixpanel лише коли залишок дійсно змінився — `no_change` і `unknown` до Mixpanel не потрапляють.

⚠️ Обережно з джерелами поза цим файлом: і документація `seller.iq`, і org-level business context у Mixpanel можуть посилатись на значення `RESTOCK` великими літерами — це помилка в тих джерелах, реальне значення завжди нижнім регістром.

## Тиша в даних не завжди поломка

Якщо `stock_snapshot` для товару, який раніше регулярно траплявся в даних, раптово зникає на тривалий період (кілька тижнів) — це не обов'язково зламаний сканер. Користувач міг сам зняти товар з відстеження чи він вибув з асортименту. **Не робити висновок «сканер зламався» самостійно і не подавати це в звіті як проблему** — прямо запитати користувача, чи товар досі мав би скануватись, перш ніж трактувати мовчання як збій. Те саме стосується тривалої відсутності `product_position_snapshot`/`extra_product_position_snapshot` для товару, доданого в позиційний сканер.

## Використання

- **Швидкість продажів конкурента** — `sales_count` за період / кількість сканувань дає оцінку темпу продажів без доступу до чужої CRM.
- **Пошук перспективних ніш** — високий `sales_count`, низький `stock_limit`, стабільний `wishlist_count` разом натякають на дефіцитний товар з попитом.
- **Поповнення складу (свого чи конкурента)** — `stock_change_type = restock` + `restock_count`.
- Товар без рядка в довіднику `products_rozetka` тут — швидше за все конкурент, а не помилка (див. `core.md`).
