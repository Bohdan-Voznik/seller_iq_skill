# SellerIQ skill

Скіл для Claude і ChatGPT (формат Agent Skills, `agentskills.io`), який розуміє схему даних SellerIQ у Mixpanel (аналітика для продавців Rozetka) і дозволяє будувати довільні звіти й розрізи: продажі, юніт-економіка, позиції у видачі, реклама, залишки, конкуренти — а не лише один фіксований звіт.

## Встановлення

Один і той самий файл — [`skill/SKILL.md`](./skill/SKILL.md) — ставиться і в Claude, і в ChatGPT: обидва клієнти читають формат Agent Skills однаково. Уся семантика даних, онбординг і хардблоки (`reference/*.md`) підтягуються автоматично з `raw.githubusercontent.com`, коли справді потрібно — окремого файлу під ChatGPT не потрібно.

1. Завантаж `skill/SKILL.md`.
2. Постав як skill у своєму клієнті:
   - **Claude (Claude.ai / Claude Desktop):** аватар/ім'я користувача → **Settings** → **Skills** → **Add** → **Upload a skill** → обрати завантажений файл.
   - **ChatGPT:** аватар/ім'я користувача → **Settings** → у бічній панелі **Plugins** → у Plugin Directory вкладка **Skills** → **Create** → **Upload from your computer** → обрати завантажений файл.
3. Переконайся, що інструмент виконання коду в чаті має доступ до мережі на `raw.githubusercontent.com`/`github.com` — саме через нього скіл тягне довідкові файли (`curl`), не через `web_fetch`. Без цього довідкові файли не підтягнуться, і скіл не запрацює (`SKILL.md` не тримає резервної копії правил). Перевірено вручну робочим на обох платформах.
4. Підключи Mixpanel MCP-конектор до проєкту SellerIQ — покрокова інструкція під конкретний клієнт видається самим скілом при першому запуску (`reference/connector-setup.md`); регіон конектора повинен збігатись з регіоном проєкту (`mcp.mixpanel.com` US / `mcp-eu.mixpanel.com` EU).

Перевстановлення `SKILL.md` потрібне лише коли змінюється сам цей файл. Оновлення довідкових файлів (`reference/*.md`) підхоплюються самі — кеш `raw.githubusercontent.com` близько п'яти хвилин.

## Структура репозиторію

```
seller_iq_skill/
  README.md                              ← цей файл
  skill/
    SKILL.md                             ← єдина точка входу, спільна для Claude і ChatGPT
  reference/
    index.json                           ← маніфест: версія, дата, список файлів і коли який підвантажувати
    core.md                              ← базові пастки трактування (завжди)
    connector-setup.md                   ← підключення Mixpanel MCP-конектора (усі клієнти)
    order-item-snapshot.md               ← продажі, юніт-економіка, статуси замовлень
    product-position-snapshot.md         ← позиції у видачі, SERP
    extra-product-position-snapshot.md   ← конкуренти з 1-ї сторінки видачі
    ad-item-snapshot.md                  ← реклама, ДРР, CPO
    stock-snapshot.md                    ← залишки, швидкість продажів
    lookup-tables.md                     ← довідники products_crm / products_rozetka
    onboarding.md                        ← перше підключення
    recipes.md                           ← готові приклади розрізів і звітів
```

## Публічність

Репозиторій відкритий: семантика подій SellerIQ і логіка роботи скіла видимі всім. Це свідомий вибір — документація й так публічна, а оновлення розповсюджуються миттєво без участі користувача.
