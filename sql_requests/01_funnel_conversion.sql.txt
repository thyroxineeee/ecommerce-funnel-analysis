-- ============================================================================
-- Проект: Анализ воронки конверсии и выручки интернет-магазина
-- Файл: 01_funnel_conversion.sql.txt
-- Назначение:
--   SQL-запросы для расчёта воронки, конверсии и базовых срезов по сегментам.
--
-- Таблица:
--   events
--
-- Важные поля:
--   event_id        -- идентификатор события
--   event_ts        -- дата и время события
--   event_date      -- дата события
--   user_id         -- идентификатор пользователя
--   event_type      -- тип события: visit, view_item, add_to_cart, checkout, purchase
--   order_id        -- идентификатор заказа
--   revenue         -- выручка
--   traffic_source  -- источник трафика
--   device_type     -- тип устройства
--   category        -- категория товара
--   is_new_user     -- признак нового пользователя
--
-- Примечание:
--   В ноутбуке таблица создаётся из датафрейма df_clean:
--   df_clean.to_sql('events', conn, index=False, if_exists='replace')
-- ============================================================================


-- ============================================================================
-- 1. Базовая проверка таблицы
-- ============================================================================

SELECT
    COUNT(*) AS rows_count,
    COUNT(DISTINCT event_id) AS unique_events,
    COUNT(DISTINCT user_id) AS unique_users,
    MIN(event_date) AS first_date,
    MAX(event_date) AS last_date
FROM events;


-- ============================================================================
-- 2. Количество событий по типам
-- ============================================================================

SELECT
    event_type,
    COUNT(*) AS events_count,
    COUNT(DISTINCT user_id) AS users_count
FROM events
GROUP BY event_type
ORDER BY events_count DESC;


-- ============================================================================
-- 3. Основная воронка по уникальным пользователям
-- ============================================================================

WITH funnel AS (
    SELECT
        COUNT(DISTINCT CASE WHEN event_type = 'visit' THEN user_id END) AS visitors,
        COUNT(DISTINCT CASE WHEN event_type = 'view_item' THEN user_id END) AS viewers,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS cart_adders,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout' THEN user_id END) AS checkouts,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS buyers
    FROM events
)

SELECT
    visitors,
    viewers,
    cart_adders,
    checkouts,
    buyers,
    ROUND(viewers * 100.0 / visitors, 2) AS visit_to_view_conversion,
    ROUND(cart_adders * 100.0 / visitors, 2) AS visit_to_cart_conversion,
    ROUND(checkouts * 100.0 / visitors, 2) AS visit_to_checkout_conversion,
    ROUND(buyers * 100.0 / visitors, 2) AS visit_to_purchase_conversion
FROM funnel;


-- ============================================================================
-- 4. Воронка в вертикальном формате
-- Такой формат удобнее переносить в BI-системы и строить график воронки.
-- ============================================================================

WITH funnel_steps AS (
    SELECT
        1 AS step_number,
        'visit' AS step_name,
        COUNT(DISTINCT user_id) AS users_count
    FROM events
    WHERE event_type = 'visit'

    UNION ALL

    SELECT
        2 AS step_number,
        'view_item' AS step_name,
        COUNT(DISTINCT user_id) AS users_count
    FROM events
    WHERE event_type = 'view_item'

    UNION ALL

    SELECT
        3 AS step_number,
        'add_to_cart' AS step_name,
        COUNT(DISTINCT user_id) AS users_count
    FROM events
    WHERE event_type = 'add_to_cart'

    UNION ALL

    SELECT
        4 AS step_number,
        'checkout' AS step_name,
        COUNT(DISTINCT user_id) AS users_count
    FROM events
    WHERE event_type = 'checkout'

    UNION ALL

    SELECT
        5 AS step_number,
        'purchase' AS step_name,
        COUNT(DISTINCT user_id) AS users_count
    FROM events
    WHERE event_type = 'purchase'
),

funnel_with_previous_step AS (
    SELECT
        step_number,
        step_name,
        users_count,
        LAG(users_count) OVER (ORDER BY step_number) AS previous_step_users,
        FIRST_VALUE(users_count) OVER (ORDER BY step_number) AS first_step_users
    FROM funnel_steps
)

SELECT
    step_number,
    step_name,
    users_count,
    previous_step_users,
    users_count - previous_step_users AS users_change_from_previous_step,
    ROUND(users_count * 100.0 / first_step_users, 2) AS conversion_from_start,
    ROUND(users_count * 100.0 / previous_step_users, 2) AS conversion_from_previous_step
FROM funnel_with_previous_step
ORDER BY step_number;


-- ============================================================================
-- 5. Ключевые метрики проекта
-- ============================================================================

WITH main_metrics AS (
    SELECT
        COUNT(DISTINCT user_id) AS visitors,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS buyers,
        COUNT(DISTINCT order_id) AS orders,
        SUM(revenue) AS revenue
    FROM events
)

SELECT
    visitors,
    buyers,
    orders,
    revenue,
    ROUND(revenue * 1.0 / orders, 2) AS average_order_value,
    ROUND(buyers * 100.0 / visitors, 2) AS purchase_conversion
FROM main_metrics;


-- ============================================================================
-- 6. Источники трафика: пользователи, покупатели, заказы, выручка, конверсия
-- ============================================================================

WITH source_metrics AS (
    SELECT
        traffic_source,
        COUNT(DISTINCT user_id) AS users,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS buyers,
        COUNT(DISTINCT order_id) AS orders,
        SUM(revenue) AS revenue
    FROM events
    GROUP BY traffic_source
)

SELECT
    traffic_source,
    users,
    buyers,
    orders,
    revenue,
    ROUND(buyers * 100.0 / users, 2) AS purchase_conversion,
    ROUND(revenue * 1.0 / orders, 2) AS average_order_value
FROM source_metrics
ORDER BY revenue DESC;


-- ============================================================================
-- 7. Типы устройств: где пользователи лучше доходят до покупки
-- ============================================================================

WITH device_metrics AS (
    SELECT
        device_type,
        COUNT(DISTINCT user_id) AS users,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS buyers,
        COUNT(DISTINCT order_id) AS orders,
        SUM(revenue) AS revenue
    FROM events
    GROUP BY device_type
)

SELECT
    device_type,
    users,
    buyers,
    orders,
    revenue,
    ROUND(buyers * 100.0 / users, 2) AS purchase_conversion,
    ROUND(revenue * 1.0 / orders, 2) AS average_order_value
FROM device_metrics
ORDER BY purchase_conversion DESC;


-- ============================================================================
-- 8. Категории товаров: вклад в выручку
-- ============================================================================

SELECT
    category,
    COUNT(*) AS events_count,
    COUNT(DISTINCT user_id) AS users,
    COUNT(DISTINCT order_id) AS orders,
    SUM(revenue) AS revenue,
    ROUND(SUM(revenue) * 1.0 / COUNT(DISTINCT order_id), 2) AS average_order_value
FROM events
WHERE category IS NOT NULL
GROUP BY category
ORDER BY revenue DESC;


-- ============================================================================
-- 9. Новые и старые пользователи
-- ============================================================================

WITH user_type_metrics AS (
    SELECT
        CASE
            WHEN is_new_user = 1 THEN 'new_user'
            WHEN is_new_user = 0 THEN 'returning_user'
            ELSE 'unknown'
        END AS user_type,
        COUNT(DISTINCT user_id) AS users,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS buyers,
        COUNT(DISTINCT order_id) AS orders,
        SUM(revenue) AS revenue
    FROM events
    GROUP BY is_new_user
)

SELECT
    user_type,
    users,
    buyers,
    orders,
    revenue,
    ROUND(buyers * 100.0 / users, 2) AS purchase_conversion,
    ROUND(revenue * 1.0 / orders, 2) AS average_order_value
FROM user_type_metrics
ORDER BY revenue DESC;


-- ============================================================================
-- 10. Динамика заказов и выручки по дням
-- ============================================================================

SELECT
    event_date,
    COUNT(DISTINCT user_id) AS users,
    COUNT(DISTINCT order_id) AS orders,
    SUM(revenue) AS revenue
FROM events
GROUP BY event_date
ORDER BY event_date;
