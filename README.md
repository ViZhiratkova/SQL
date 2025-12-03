# 🐬 MySQL

Вот запросы к MySQL, которые я использовала для выполнения задач по тестированию интернет-магазина [https://intern.demoshopping.ru/](https://intern.demoshopping.ru/) (задачи внутри ссылочек):

- [SELECT - запросы](https://docs.google.com/spreadsheets/d/1tPCrDBt4BpKhUI10iq2ZqrK_pJyymaNaXjHhi1t7M04/edit?gid=0#gid=0)  
- [JOIN - запросы](https://docs.google.com/spreadsheets/d/1shVvbY5ZcAPTCOjE9aRcfhl4Ki3YTXWTqDfuIZnGtco/edit?gid=0#gid=0)

---

# 🦫 DBeaver + PostgreSQL: Поиск аномальных заказов (мошенничество)

Реальная бизнес задача. Нужно было реализовать запрос из БД, чтобы найти максимально аномальные заказы в интернет-магазине, которые являются частью мошеннической схемы. Был реализован следующий запрос с последующей доработкой под особенности данных:

```sql
WITH filtered AS (
  SELECT *
  FROM params
  WHERE score >= 100 AND param3 >= 1000
),
normalized AS (
  SELECT
    (param1 - AVG(param1) OVER ()) / STDDEV_SAMP(param1) OVER () AS norm_p1,
    (param2 - AVG(param2) OVER ()) / STDDEV_SAMP(param2) OVER () AS norm_p2,
    (param3 - AVG(param3) OVER ()) / STDDEV_SAMP(param3) OVER () AS norm_p3,
    *
  FROM filtered
),
stats AS (
  SELECT
    percentile_disc(0.25) WITHIN GROUP (ORDER BY norm_p1) AS q1_p1,
    percentile_disc(0.75) WITHIN GROUP (ORDER BY norm_p1) AS q3_p1,
    percentile_disc(0.25) WITHIN GROUP (ORDER BY norm_p2) AS q1_p2,
    percentile_disc(0.75) WITHIN GROUP (ORDER BY norm_p2) AS q3_p2,
    percentile_disc(0.25) WITHIN GROUP (ORDER BY norm_p3) AS q1_p3,
    percentile_disc(0.75) WITHIN GROUP (ORDER BY norm_p3) AS q3_p3
  FROM normalized
),
bounds AS (
  SELECT
    q3_p1 + 1.5 * (q3_p1 - q1_p1) AS high_p1,
    q3_p2 + 1.5 * (q3_p2 - q1_p2) AS high_p2,
    q3_p3 + 1.5 * (q3_p3 - q1_p3) AS high_p3
  FROM stats
)
SELECT n.*
FROM normalized n
CROSS JOIN bounds b
WHERE
  (
    n.norm_p1 > b.high_p1
    OR
    n.norm_p2 > b.high_p2
    OR
    n.norm_p3 > b.high_p3
  )
ORDER BY n.param1 DESC;
