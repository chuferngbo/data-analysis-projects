# 巴西电商订单经营分析：SQL 代码注释与问题复盘

> 项目数据：Olist Brazilian E-Commerce Public Dataset  
> 数据库：SQLite / DB Browser for SQLite  
> 说明：所有分析基于已导入的 8 张核心表；金额单位沿用原始数据中的巴西雷亚尔（BRL）。

## 1. 数据表与关联关系

本项目使用以下 8 张表：

| 表名 | 作用 | 核心关联字段 |
|---|---|---|
| `olist_orders_dataset` | 订单主表，含订单状态和下单、配送时间 | `order_id`、`customer_id` |
| `olist_order_items_dataset` | 订单商品明细，含商品价格、运费、卖家 | `order_id`、`product_id`、`seller_id` |
| `olist_order_payments_dataset` | 支付明细 | `order_id` |
| `olist_order_reviews_dataset` | 订单评价 | `order_id` |
| `olist_customers_dataset` | 客户所在地及真实客户标识 | `customer_id`、`customer_unique_id` |
| `olist_products_dataset` | 商品信息及类目 | `product_id`、`product_category_name` |
| `olist_sellers_dataset` | 卖家信息 | `seller_id` |
| `product_category_name_translation` | 商品类目中英文翻译 | `product_category_name` |

表关系如下：

```text
customers -> orders -> order_items -> products
                    -> order_items -> sellers
                    -> payments
                    -> reviews
```

## 2. 查询性能优化

导入 CSV 后，先为高频连接字段建立索引。每条语句只需执行一次。

```sql
-- 为订单主键建立索引，加快订单关联查询
CREATE INDEX IF NOT EXISTS idx_orders_order_id
ON olist_orders_dataset(order_id);

-- 为商品明细中的订单字段建立索引，加快 orders 与 order_items 的 JOIN
CREATE INDEX IF NOT EXISTS idx_items_order_id
ON olist_order_items_dataset(order_id);

-- 为支付明细中的订单字段建立索引，加快 orders 与 payments 的 JOIN
CREATE INDEX IF NOT EXISTS idx_payments_order_id
ON olist_order_payments_dataset(order_id);
```

## 3. 数据规模检查

```sql
-- 逐表统计行数，确认 CSV 是否完整导入 SQLite
SELECT 'customers' AS table_name, COUNT(*) AS row_count
FROM olist_customers_dataset

UNION ALL
SELECT 'orders', COUNT(*)
FROM olist_orders_dataset

UNION ALL
SELECT 'order_items', COUNT(*)
FROM olist_order_items_dataset

UNION ALL
SELECT 'payments', COUNT(*)
FROM olist_order_payments_dataset

UNION ALL
SELECT 'reviews', COUNT(*)
FROM olist_order_reviews_dataset

UNION ALL
SELECT 'products', COUNT(*)
FROM olist_products_dataset

UNION ALL
SELECT 'sellers', COUNT(*)
FROM olist_sellers_dataset

UNION ALL
SELECT 'category_translation', COUNT(*)
FROM product_category_name_translation;
```

### 查询结果

| 表 | 行数 |
|---|---:|
| customers | 203,327 |
| orders | 99,441 |
| order_items | 112,650 |
| payments | 103,886 |
| reviews | 99,224 |
| products | 32,951 |
| sellers | 3,095 |
| category_translation | 71 |

说明：`order_items` 行数大于 `orders`，表示一笔订单可能包含多件商品。因此统计订单数时，通常需要使用 `COUNT(DISTINCT order_id)` 去重。

## 4. 各州订单量与送达率

```sql
-- 将订单表与客户表连接，统计各州订单量和已送达率
SELECT
    c.customer_state AS customer_state, -- 客户所在州
    COUNT(DISTINCT o.order_id) AS total_orders, -- 总订单数，按订单去重
    SUM(
        CASE
            WHEN o.order_status = 'delivered' THEN 1
            ELSE 0
        END
    ) AS delivered_orders, -- 已送达订单数
    ROUND(
        1.0 * SUM(
            CASE
                WHEN o.order_status = 'delivered' THEN 1
                ELSE 0
            END
        ) / COUNT(DISTINCT o.order_id),
        4
    ) AS delivered_rate -- 已送达率
FROM olist_orders_dataset AS o
INNER JOIN olist_customers_dataset AS c
    ON o.customer_id = c.customer_id -- 订单通过 customer_id 对应客户
GROUP BY c.customer_state
ORDER BY total_orders DESC;
```

### 结论

- `SP` 州订单量最高，为 41,746 单，是平台核心市场。
- 各州已送达率整体较高，主要州集中在约 96% 至 98%。

## 5. 各州 GMV 与客单价

```sql
-- 连接订单、客户和商品明细，统计各州已送达订单的销售额
SELECT
    c.customer_state AS customer_state, -- 客户所在州
    COUNT(DISTINCT o.order_id) AS delivered_orders, -- 已送达订单数
    ROUND(SUM(oi.price), 2) AS product_amount, -- 商品金额，不含运费
    ROUND(SUM(oi.freight_value), 2) AS freight_amount, -- 运费金额
    ROUND(SUM(oi.price + oi.freight_value), 2) AS total_amount, -- GMV：商品金额加运费
    ROUND(
        SUM(oi.price + oi.freight_value)
        / COUNT(DISTINCT o.order_id),
        2
    ) AS avg_order_amount -- 平均每笔订单成交金额
FROM olist_orders_dataset AS o
INNER JOIN olist_customers_dataset AS c
    ON o.customer_id = c.customer_id
INNER JOIN olist_order_items_dataset AS oi
    ON o.order_id = oi.order_id
WHERE o.order_status = 'delivered' -- 仅保留已送达订单
GROUP BY c.customer_state
ORDER BY total_amount DESC;
```

### 结论

- `SP` 州在订单量和 GMV 上均显著领先。
- 一笔订单可能有多条商品明细，所以金额按商品明细求和，订单数必须按 `order_id` 去重。

## 6. 月度 GMV、订单量与客单价

```sql
-- 月度销售趋势：仅连接订单表和商品明细表，不会造成多表金额重复
SELECT
    SUBSTR(o.order_purchase_timestamp, 1, 7) AS purchase_month, -- 截取 YYYY-MM 作为下单月份
    COUNT(DISTINCT o.order_id) AS delivered_orders, -- 已送达订单数
    ROUND(
        SUM(oi.price + oi.freight_value),
        2
    ) AS order_gmv, -- 商品金额加运费
    ROUND(
        SUM(oi.price + oi.freight_value)
        / COUNT(DISTINCT o.order_id),
        2
    ) AS avg_order_amount -- 平均订单金额
FROM olist_orders_dataset AS o
INNER JOIN olist_order_items_dataset AS oi
    ON o.order_id = oi.order_id
WHERE o.order_status = 'delivered'
GROUP BY SUBSTR(o.order_purchase_timestamp, 1, 7)
ORDER BY purchase_month;
```

### 结论

- 订单量和 GMV 从 2017 年初逐步增长。
- 2017 年 11 月达到峰值：7,289 单、GMV 约 115.34 万。
- 客单价主要位于约 146 至 170 区间，增长主要来自订单量提升。
- 2016 年 12 月仅 1 单，属于数据早期的极小样本，不宜作为经营判断重点。

## 7. `LAG()` 计算月度 GMV 环比

```sql
-- 第一步：先计算每月 GMV
WITH monthly_sales AS (
    SELECT
        SUBSTR(o.order_purchase_timestamp, 1, 7) AS purchase_month,
        COUNT(DISTINCT o.order_id) AS delivered_orders,
        ROUND(SUM(oi.price + oi.freight_value), 2) AS order_gmv,
        ROUND(
            SUM(oi.price + oi.freight_value)
            / COUNT(DISTINCT o.order_id),
            2
        ) AS avg_order_amount
    FROM olist_orders_dataset AS o
    INNER JOIN olist_order_items_dataset AS oi
        ON o.order_id = oi.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY SUBSTR(o.order_purchase_timestamp, 1, 7)
)

-- 第二步：LAG() 取当前月份上一行，即上个月的 GMV
SELECT
    purchase_month,
    delivered_orders,
    order_gmv,
    avg_order_amount,
    LAG(order_gmv) OVER (
        ORDER BY purchase_month
    ) AS last_month_gmv, -- 上月 GMV
    ROUND(
        100.0 * (
            order_gmv - LAG(order_gmv) OVER (ORDER BY purchase_month)
        ) / NULLIF(
            LAG(order_gmv) OVER (ORDER BY purchase_month),
            0
        ),
        2
    ) AS gmv_mom_rate -- GMV 环比，单位为 %
FROM monthly_sales
WHERE purchase_month >= '2017-01'
ORDER BY purchase_month;
```

说明：第一行没有上个月数据，因此 `last_month_gmv` 和环比为空属于正常现象。`NULLIF(..., 0)` 用于避免分母为 0 时出现除零错误。

## 8. 商品类目 GMV Top10

```sql
-- 先汇总每个商品类目的已送达订单 GMV
WITH category_sales AS (
    SELECT
        COALESCE(
            ct.product_category_name_english,
            p.product_category_name,
            'unknown'
        ) AS category_name, -- 优先使用英文类目；缺失时保留原始类目或 unknown
        COUNT(DISTINCT o.order_id) AS delivered_orders,
        ROUND(SUM(oi.price + oi.freight_value), 2) AS order_gmv
    FROM olist_orders_dataset AS o
    INNER JOIN olist_order_items_dataset AS oi
        ON o.order_id = oi.order_id
    INNER JOIN olist_products_dataset AS p
        ON oi.product_id = p.product_id
    LEFT JOIN product_category_name_translation AS ct
        ON p.product_category_name = ct.product_category_name
    WHERE o.order_status = 'delivered'
    GROUP BY
        COALESCE(
            ct.product_category_name_english,
            p.product_category_name,
            'unknown'
        )
)

-- 使用 ROW_NUMBER() 为类目按 GMV 排名
SELECT
    ROW_NUMBER() OVER (
        ORDER BY order_gmv DESC
    ) AS gmv_rank,
    category_name,
    delivered_orders,
    order_gmv
FROM category_sales
ORDER BY gmv_rank
LIMIT 10;
```

### 结果摘要

| 排名 | 类目 | 已送达订单数 | GMV |
|---:|---|---:|---:|
| 1 | health_beauty | 8,647 | 1,412,089.53 |
| 2 | watches_gifts | 5,495 | 1,264,333.12 |
| 3 | bed_bath_table | 9,272 | 1,225,209.26 |

`health_beauty` 是 GMV 第一类目；`bed_bath_table` 的订单数更高，但 GMV 排第三；`watches_gifts` 订单数较少却位居 GMV 第二，客单价相对更高。

## 9. RFM 客户评分

### 9.1 先计算 R、F、M 指标与评分

```sql
-- 分析基准日设置为数据中最后一次下单日期的下一天
WITH analysis_date AS (
    SELECT DATE(MAX(order_purchase_timestamp), '+1 day') AS reference_date
    FROM olist_orders_dataset
),

-- 按真实客户 customer_unique_id 聚合，避免把同一人多次下单视为多个客户
customer_rfm AS (
    SELECT
        c.customer_unique_id,
        MAX(DATE(o.order_purchase_timestamp)) AS last_purchase_date,
        COUNT(DISTINCT o.order_id) AS frequency, -- F：购买订单数
        ROUND(SUM(oi.price + oi.freight_value), 2) AS monetary -- M：消费金额
    FROM olist_orders_dataset AS o
    INNER JOIN olist_customers_dataset AS c
        ON o.customer_id = c.customer_id
    INNER JOIN olist_order_items_dataset AS oi
        ON o.order_id = oi.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id
),

-- 计算 R：距最近一次购买的天数
rfm_base AS (
    SELECT
        r.*,
        CAST(
            JULIANDAY(a.reference_date) - JULIANDAY(r.last_purchase_date)
            AS INTEGER
        ) AS recency
    FROM customer_rfm AS r
    CROSS JOIN analysis_date AS a
),

-- R、F、M 评分
rfm_score AS (
    SELECT
        *,
        NTILE(5) OVER (ORDER BY recency DESC) AS r_score, -- 最近购买的客户位于后面的分组，得分更高
        CASE
            WHEN frequency >= 5 THEN 5
            WHEN frequency = 4 THEN 4
            WHEN frequency = 3 THEN 3
            WHEN frequency = 2 THEN 2
            ELSE 1
        END AS f_score, -- 同样购买次数必须得到同样的 F 分
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score -- 消费金额越高，M 分越高
    FROM rfm_base
)

SELECT
    customer_unique_id,
    recency,
    frequency,
    monetary,
    r_score,
    f_score,
    m_score
FROM rfm_score
ORDER BY monetary DESC
LIMIT 20;
```

### 9.2 客户分层汇总

```sql
WITH analysis_date AS (
    SELECT DATE(MAX(order_purchase_timestamp), '+1 day') AS reference_date
    FROM olist_orders_dataset
),
customer_rfm AS (
    SELECT
        c.customer_unique_id,
        MAX(DATE(o.order_purchase_timestamp)) AS last_purchase_date,
        COUNT(DISTINCT o.order_id) AS frequency,
        SUM(oi.price + oi.freight_value) AS monetary
    FROM olist_orders_dataset AS o
    INNER JOIN olist_customers_dataset AS c
        ON o.customer_id = c.customer_id
    INNER JOIN olist_order_items_dataset AS oi
        ON o.order_id = oi.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id
),
rfm_base AS (
    SELECT
        r.*,
        CAST(
            JULIANDAY(a.reference_date) - JULIANDAY(r.last_purchase_date)
            AS INTEGER
        ) AS recency
    FROM customer_rfm AS r
    CROSS JOIN analysis_date AS a
),
rfm_score AS (
    SELECT
        *,
        NTILE(5) OVER (ORDER BY recency DESC) AS r_score,
        CASE
            WHEN frequency >= 5 THEN 5
            WHEN frequency = 4 THEN 4
            WHEN frequency = 3 THEN 3
            WHEN frequency = 2 THEN 2
            ELSE 1
        END AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM rfm_base
),

-- 将 RFM 分数转化为业务标签
customer_segment AS (
    SELECT
        *,
        CASE
            WHEN r_score >= 4 AND f_score >= 3 AND m_score >= 4
                THEN '高价值客户'
            WHEN r_score >= 4 AND (f_score >= 2 OR m_score >= 4)
                THEN '潜力客户'
            WHEN r_score <= 2 AND (f_score >= 2 OR m_score >= 4)
                THEN '流失风险客户'
            WHEN r_score <= 2 AND f_score = 1
                THEN '低价值客户'
            ELSE '一般客户'
        END AS customer_type
    FROM rfm_score
)

SELECT
    customer_type AS 客户类型,
    COUNT(*) AS 客户数,
    ROUND(AVG(recency), 1) AS 平均未购买天数,
    ROUND(AVG(frequency), 2) AS 平均购买次数,
    ROUND(AVG(monetary), 2) AS 平均消费金额
FROM customer_segment
GROUP BY customer_type
ORDER BY 客户数 DESC;
```

### RFM 分层结果

| 客户类型 | 客户数 | 平均未购买天数 | 平均购买次数 | 平均消费金额 |
|---|---:|---:|---:|---:|
| 一般客户 | 40,388 | 199.4 | 1.02 | 111.58 |
| 低价值客户 | 22,594 | 445.0 | 1.00 | 72.69 |
| 潜力客户 | 15,509 | 140.7 | 1.07 | 302.23 |
| 流失风险客户 | 14,750 | 442.7 | 1.07 | 306.20 |
| 高价值客户 | 117 | 140.9 | 3.55 | 572.05 |

业务建议：优先召回流失风险客户；通过优惠券、关联推荐推动潜力客户复购；为高价值客户提供会员权益与专属服务。

## 10. 各州配送时长与评价

```sql
-- 先把评价表处理为每个订单一行，避免重复连接
WITH order_reviews AS (
    SELECT
        order_id,
        AVG(review_score) AS avg_review_score
    FROM olist_order_reviews_dataset
    GROUP BY order_id
)

SELECT
    c.customer_state AS customer_state,
    COUNT(DISTINCT o.order_id) AS delivered_orders,
    ROUND(
        AVG(
            JULIANDAY(o.order_delivered_customer_date)
            - JULIANDAY(o.order_purchase_timestamp)
        ),
        2
    ) AS avg_delivery_days, -- 平均配送时长
    ROUND(
        AVG(
            JULIANDAY(o.order_estimated_delivery_date)
            - JULIANDAY(o.order_delivered_customer_date)
        ),
        2
    ) AS avg_early_delivery_days, -- 正数表示平均提前送达
    ROUND(AVG(r.avg_review_score), 2) AS avg_review_score -- 平均评价分
FROM olist_orders_dataset AS o
INNER JOIN olist_customers_dataset AS c
    ON o.customer_id = c.customer_id
LEFT JOIN order_reviews AS r
    ON o.order_id = r.order_id
WHERE o.order_status = 'delivered'
  AND o.order_delivered_customer_date IS NOT NULL
  AND o.order_estimated_delivery_date IS NOT NULL
GROUP BY c.customer_state
ORDER BY avg_delivery_days DESC;
```

说明：偏远州的配送时长更长，但其中部分州订单量小，不能只根据均值直接做强结论。

## 11. 配送时长是否影响评分

```sql
-- 每笔订单的评价聚合为一行
WITH order_reviews AS (
    SELECT
        order_id,
        AVG(review_score) AS avg_review_score
    FROM olist_order_reviews_dataset
    GROUP BY order_id
),

-- 计算每笔已送达订单的配送天数
delivery_detail AS (
    SELECT
        o.order_id,
        JULIANDAY(o.order_delivered_customer_date)
        - JULIANDAY(o.order_purchase_timestamp) AS delivery_days,
        r.avg_review_score
    FROM olist_orders_dataset AS o
    LEFT JOIN order_reviews AS r
        ON o.order_id = r.order_id
    WHERE o.order_status = 'delivered'
      AND o.order_delivered_customer_date IS NOT NULL
),

-- 用 CASE WHEN 将连续的配送天数划分为业务可读的区间
delivery_segment AS (
    SELECT
        *,
        CASE
            WHEN delivery_days <= 7 THEN '7天及以内'
            WHEN delivery_days <= 14 THEN '8-14天'
            WHEN delivery_days <= 21 THEN '15-21天'
            ELSE '超过21天'
        END AS delivery_type
    FROM delivery_detail
)

SELECT
    delivery_type AS 配送时长分组,
    COUNT(*) AS 订单数,
    ROUND(AVG(delivery_days), 2) AS 平均配送天数,
    ROUND(AVG(avg_review_score), 2) AS 平均评价分
FROM delivery_segment
GROUP BY delivery_type
ORDER BY
    CASE delivery_type
        WHEN '7天及以内' THEN 1
        WHEN '8-14天' THEN 2
        WHEN '15-21天' THEN 3
        ELSE 4
    END;
```

### 结果与结论

| 配送时长分组 | 订单数 | 平均配送天数 | 平均评价分 |
|---|---:|---:|---:|
| 7天及以内 | 26,046 | 4.66 | 4.42 |
| 8-14天 | 40,212 | 10.09 | 4.31 |
| 15-21天 | 17,713 | 16.91 | 4.14 |
| 超过21天 | 12,499 | 30.79 | 3.12 |

配送越慢，评价越低。超过 21 天的订单平均评分为 3.12，较 7 天及以内订单低 1.30 分，是物流体验优化的重点。

## 12. 每月 GMV Top3 商品类目

```sql
-- 统计每个月、每个商品类目的 GMV
WITH monthly_category_gmv AS (
    SELECT
        SUBSTR(o.order_purchase_timestamp, 1, 7) AS purchase_month,
        COALESCE(
            ct.product_category_name_english,
            p.product_category_name,
            'unknown'
        ) AS category_name,
        ROUND(SUM(oi.price + oi.freight_value), 2) AS order_gmv
    FROM olist_orders_dataset AS o
    INNER JOIN olist_order_items_dataset AS oi
        ON o.order_id = oi.order_id
    INNER JOIN olist_products_dataset AS p
        ON oi.product_id = p.product_id
    LEFT JOIN product_category_name_translation AS ct
        ON p.product_category_name = ct.product_category_name
    WHERE o.order_status = 'delivered'
    GROUP BY
        SUBSTR(o.order_purchase_timestamp, 1, 7),
        COALESCE(
            ct.product_category_name_english,
            p.product_category_name,
            'unknown'
        )
),

-- PARTITION BY 表示每个月内部重新排名
category_rank AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY purchase_month
            ORDER BY order_gmv DESC
        ) AS category_rank
    FROM monthly_category_gmv
)

SELECT
    purchase_month,
    category_rank AS 类目排名,
    category_name AS 商品类目,
    order_gmv AS GMV
FROM category_rank
WHERE category_rank <= 3
ORDER BY purchase_month, category_rank;
```

说明：月度头部类目会变化，例如 2017 年 1 月为 `furniture_decor`，2 月为 `health_beauty`，3 月为 `computers_accessories`，4 月为 `bed_bath_table`。因此类目运营应结合月度变化动态调整。

## 13. 卖家 GMV 集中度与二八分析

### 13.1 卖家 GMV 累计贡献

```sql
-- 先计算每个卖家的已送达订单 GMV
WITH seller_gmv AS (
    SELECT
        oi.seller_id,
        COUNT(DISTINCT o.order_id) AS delivered_orders,
        ROUND(SUM(oi.price + oi.freight_value), 2) AS order_gmv
    FROM olist_orders_dataset AS o
    INNER JOIN olist_order_items_dataset AS oi
        ON o.order_id = oi.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY oi.seller_id
),

-- 计算卖家排名和从第一名开始累积的 GMV 占比
seller_contribution AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            ORDER BY order_gmv DESC
        ) AS seller_rank,
        ROUND(
            100.0 * SUM(order_gmv) OVER (
                ORDER BY order_gmv DESC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            )
            / SUM(order_gmv) OVER (),
            2
        ) AS cumulative_gmv_pct
    FROM seller_gmv
)

SELECT
    seller_rank AS 卖家排名,
    seller_id AS 卖家ID,
    delivered_orders AS 已送达订单数,
    order_gmv AS GMV,
    cumulative_gmv_pct AS 累计GMV贡献占比
FROM seller_contribution
ORDER BY seller_rank
LIMIT 20;
```

### 13.2 达到 80% GMV 所需的卖家数

```sql
WITH seller_gmv AS (
    SELECT
        oi.seller_id,
        SUM(oi.price + oi.freight_value) AS order_gmv
    FROM olist_orders_dataset AS o
    INNER JOIN olist_order_items_dataset AS oi
        ON o.order_id = oi.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY oi.seller_id
),
seller_contribution AS (
    SELECT
        seller_id,
        order_gmv,
        ROW_NUMBER() OVER (
            ORDER BY order_gmv DESC
        ) AS seller_rank,
        100.0 * SUM(order_gmv) OVER (
            ORDER BY order_gmv DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) / SUM(order_gmv) OVER () AS cumulative_gmv_pct
    FROM seller_gmv
)
SELECT
    MIN(seller_rank) AS 达到80百分比GMV所需卖家数
FROM seller_contribution
WHERE cumulative_gmv_pct >= 80;
```

### 结果与结论

排名前 551 的卖家贡献了约 80% 的已送达订单 GMV，平台存在明显的头部卖家集中现象。建议重点维护头部卖家，并通过流量扶持、营销资源和运营培训提升腰部卖家贡献。

## 14. 中间遇到的问题、原因与解决方法

### 问题 1：批量导入 CSV 时提示“已存在同名表”

**现象**：一次选中多个 CSV 导入时，系统提示已存在名为 `olist_customers_dataset` 的表。

**原因**：DB Browser 的该导入窗口会将当前输入的“表名称”应用给所选文件；第一张客户表已创建，下一张文件仍尝试写入同名表，因而冲突。

**解决方法**：每次只导入一个 CSV，并让每个文件使用独立表名。第一张客户表已经导入成功，保留即可，不需要重复导入。

### 问题 2：CSV 导入时“列名在首行”未勾选

**风险**：如果不勾选“列名在首行”，字段会被命名为 `field1`、`field2` 等，后续 SQL 无法使用 `order_id`、`customer_id` 等业务字段。

**解决方法**：导入每张 CSV 时均勾选“列名在首行”，确认预览区第一行显示真实字段名后再导入。

### 问题 3：只选中了 SQL 的 `WHERE / GROUP BY / ORDER BY` 片段

**现象**：执行 SQL 后没有得到预期结果，或报语法错误。

**原因**：DB Browser 会优先执行被选中的文本；单独执行 `WHERE`、`GROUP BY`、`ORDER BY` 不包含完整的 `SELECT` 和 `FROM`，不是一条完整 SQL。

**解决方法**：选中完整 SQL 语句再运行，或取消选中并将光标放在需要运行的完整语句内。不要只选中半段代码。

### 问题 4：多层 `WITH` 查询运行缓慢甚至界面卡住

**现象**：同时对订单商品明细、支付明细进行聚合并连接时，DB Browser 长时间无响应。

**原因**：查询需要扫描多张十万级明细表并按 `order_id` 分组；刚导入的数据没有索引，连接和聚合成本较高。编辑区中存在多段 SQL 时，误执行全部语句也会增加等待时间。

**解决方法**：

1. 使用工具栏红色停止按钮终止长时间执行的查询；若界面无响应，可关闭并重新打开数据库。
2. 为 `order_id` 建立索引，见本文第 2 节。
3. 先运行轻量的 `orders + order_items` 月度 GMV 查询，再逐步增加支付表或其他明细表。
4. 每次仅执行当前需要的完整 SQL，避免误跑编辑区中的历史查询。

### 问题 5：R 分数方向写反

**错误写法**：

```sql
-- 错误：会让距离最近购买越久的客户得到越高 R 分
6 - NTILE(5) OVER (ORDER BY recency DESC) AS r_score
```

**原因**：当按 `recency DESC` 排序时，最久未购买的客户位于前面的分组，已经得到较低分；再用 `6 -` 反转，会把最久未购买的客户变成高分。

**正确写法**：

```sql
-- 正确：越近期购买的客户位于后面的分组，得分越高
NTILE(5) OVER (ORDER BY recency DESC) AS r_score
```

### 问题 6：购买 1 次的客户得到不同 F 分

**现象**：多位 `frequency = 1` 的客户，`f_score` 却出现了 1 到 5 的不同分数。

**原因**：`NTILE(5)` 按行平均切分数据。当购买 1 次的客户数量很大时，它会把相同的频次硬分散到不同分组，因此不符合业务含义。

**解决方法**：频次是离散值，改用 `CASE WHEN` 固定规则评分。相同购买次数就会获得相同 F 分。

```sql
CASE
    WHEN frequency >= 5 THEN 5
    WHEN frequency = 4 THEN 4
    WHEN frequency = 3 THEN 3
    WHEN frequency = 2 THEN 2
    ELSE 1
END AS f_score
```

### 问题 7：直接连接商品明细表和支付明细表会重复计算金额

**原因**：一笔订单可以有多条商品明细，也可能有多条支付记录。若直接连接 `order_items` 与 `payments`，会形成“商品行数 × 支付行数”的组合，导致金额被放大。

**解决思路**：先分别按 `order_id` 汇总为“一单一行”，再连接订单表。

```sql
-- 商品明细先按订单汇总
WITH order_item_amount AS (
    SELECT
        order_id,
        SUM(price + freight_value) AS order_gmv
    FROM olist_order_items_dataset
    GROUP BY order_id
),

-- 支付明细也先按订单汇总
order_payment_amount AS (
    SELECT
        order_id,
        SUM(payment_value) AS payment_amount
    FROM olist_order_payments_dataset
    GROUP BY order_id
)

SELECT
    SUBSTR(o.order_purchase_timestamp, 1, 7) AS purchase_month,
    COUNT(DISTINCT o.order_id) AS delivered_orders,
    ROUND(SUM(ia.order_gmv), 2) AS order_gmv,
    ROUND(SUM(pa.payment_amount), 2) AS payment_amount
FROM olist_orders_dataset AS o
INNER JOIN order_item_amount AS ia
    ON o.order_id = ia.order_id
INNER JOIN order_payment_amount AS pa
    ON o.order_id = pa.order_id
WHERE o.order_status = 'delivered'
GROUP BY SUBSTR(o.order_purchase_timestamp, 1, 7)
ORDER BY purchase_month;
```

## 15. 本项目掌握的 SQL 技能

- 多表 `INNER JOIN`、`LEFT JOIN` 与关联粒度判断。
- `GROUP BY`、`COUNT(DISTINCT ...)`、`SUM()`、`AVG()` 等聚合分析。
- `CASE WHEN` 实现状态统计、客户分层和业务分组。
- `WITH`（CTE）拆分复杂计算，并防止多对多连接导致的金额重复。
- `JULIANDAY()`、`DATE()`、`SUBSTR()` 等 SQLite 日期与文本函数。
- 窗口函数：`LAG()`、`ROW_NUMBER()`、`NTILE()`、`SUM() OVER()`。
- `PARTITION BY` 实现每月内部排名。
- `CREATE INDEX` 优化多表连接查询性能。
