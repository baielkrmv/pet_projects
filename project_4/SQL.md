
## Симулятор SQL Karpov.Courses

[Построение дашбордов](#построение-дашбордов)

[Анализ продуктовых метрик](#анализ-продуктовых-метрик)

[Маркетинговые метрики](#маркетинговые-метрики)

---
### Построение дашбордов
### Задача 1
Для каждого дня, представленного в таблицах user_actions и courier_actions, рассчитайте следующие показатели:   
Число новых пользователей.  
Число новых курьеров.  
Общее число пользователей на текущий день.  
Общее число курьеров на текущий день.  
Колонки с показателями назовите соответственно new_users, new_couriers, total_users, total_couriers. Колонку с датами назовите date.    
Проследите за тем, чтобы показатели были выражены целыми числами. Результат должен быть отсортирован по возрастанию даты.    
Поля в результирующей таблице: date, new_users, new_couriers, total_users, total_couriers.    

```sql
WITH user_first_actions as (
SELECT 
    user_id,
    min(time::date) as first_action_date
FROM user_actions
GROUP BY user_id),
  
courier_first_actions as (
SELECT 
  courier_id,
  min(time::date) as first_action_date
FROM courier_actions
GROUP BY courier_id), 
  
all_dates as (
SELECT first_action_date as date
FROM user_first_actions
UNION
SELECT first_action_date
FROM courier_first_actions), 
  
daily_counts as (
SELECT 
  all_dates.date,
  coalesce(count(distinct user_first_actions.user_id), 0)::integer as new_users,
  coalesce(count(distinct courier_first_actions.courier_id), 0)::integer as new_couriers
FROM all_dates
LEFT JOIN user_first_actions
  ON all_dates.date = user_first_actions.first_action_date
LEFT JOIN courier_first_actions
  ON all_dates.date = courier_first_actions.first_action_date
GROUP BY all_dates.date)

SELECT 
  date,
  new_users,
  new_couriers,
  SUM(new_users) OVER (ORDER BY date)::INT as total_users,
  SUM(new_couriers) OVER (ORDER BY date)::INT as total_couriers
FROM daily_counts
ORDER BY date;
```
![image](https://github.com/user-attachments/assets/75ee9766-ae94-481f-a0da-f1ea279ddfb3)


### Задача 2
Дополните запрос из предыдущего задания и теперь для каждого дня, представленного в таблицах user_actions и courier_actions, дополнительно рассчитайте следующие показатели:  
Прирост числа новых пользователей.  
Прирост числа новых курьеров.  
Прирост общего числа пользователей.  
Прирост общего числа курьеров.  
Показатели, рассчитанные на предыдущем шаге, также включите в результирующую таблицу.  
Колонки с новыми показателями назовите соответственно new_users_change, new_couriers_change, total_users_growth, total_couriers_growth. Колонку с датами назовите date.  
Все показатели прироста считайте в процентах относительно значений в предыдущий день.   
При расчёте показателей округляйте значения до двух знаков после запятой.    
Результирующая таблица должна быть отсортирована по возрастанию даты.   
Поля в результирующей таблице:   
date, new_users, new_couriers, total_users, total_couriers,   
new_users_change, new_couriers_change, total_users_growth, total_couriers_growth  

```sql
SELECT
  date,
  new_users,
  new_users_change,
  total_users,
  ROUND((((total_users - LAG(total_users) OVER(ORDER BY date))::DECIMAL
      / LAG(total_users) OVER(ORDER BY date))*100), 2) as total_users_growth,
  new_couriers,
  new_couriers_change,
  total_couriers,
  ROUND((((total_couriers - LAG(total_couriers) OVER(ORDER BY date))::DECIMAL
      / LAG(total_couriers) OVER(ORDER BY date))*100), 2) as total_couriers_growth
FROM (
  SELECT
    min_date as date,
    new_users,
    ROUND((((new_users - LAG(new_users) OVER(ORDER BY min_date))::DECIMAL / LAG(new_users) OVER(ORDER BY min_date))*100), 2) as new_users_change,
    SUM(new_users) OVER(ORDER BY min_date)::integer as total_users,
    new_couriers,
    ROUND((((new_couriers - LAG(new_couriers) OVER(ORDER BY min_date))::DECIMAL / LAG(new_couriers) OVER(ORDER BY min_date))*100), 2) as new_couriers_change,
    SUM(new_couriers) OVER (ORDER BY min_date)::INTEGER as total_couriers
  FROM (
    SELECT
      min_date,
      COUNT(distinct courier_id) as new_couriers
    FROM (
      SELECT
        courier_id,
        MIN(time::date) as min_date
      FROM courier_actions
      GROUP BY courier_id) as t1
    GROUP BY min_date) as t2
    LEFT JOIN (
      SELECT
        min_date,
        COUNT(distinct user_id) as new_users
      FROM (
        SELECT
          user_id,
          MIN(time::date) as min_date
        FROM user_actions
        GROUP BY user_id) as t3
      GROUP BY min_date) as t4 USING(min_date)) as t5
ORDER BY date
```
![image](https://github.com/user-attachments/assets/20cebae0-1cfb-47e0-b8c9-f3b493cc9e69)


### Задача 3
Для каждого дня, представленного в таблицах user_actions и courier_actions, рассчитайте следующие показатели:  
Число платящих пользователей.  
Число активных курьеров.  
Долю платящих пользователей в общем числе пользователей на текущий день.  
Долю активных курьеров в общем числе курьеров на текущий день.  
Колонки с показателями назовите соответственно paying_users, active_couriers, paying_users_share, active_couriers_share. Колонку с датами назовите date.   
Проследите за тем, чтобы абсолютные показатели были выражены целыми числами.   
Все показатели долей необходимо выразить в процентах. При их расчёте округляйте значения до двух знаков после запятой.  
Результат должен быть отсортирован по возрастанию даты.   
Поля в результирующей таблице: date, paying_users, active_couriers, paying_users_share, active_couriers_share  

```sql
WITH all_active_couriers AS(
SELECT
    time::DATE AS date,
    COUNT(DISTINCT courier_id) AS active_couriers
FROM courier_actions  
WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
GROUP BY date),

all_paying_users AS(
SELECT
    time::DATE AS date,
    COUNT(DISTINCT user_id) AS paying_users
FROM user_actions
WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
GROUP BY date),

start_date_users AS(
SELECT
    start_date,
    COUNT(user_id) AS new_users
FROM
    (SELECT
        user_id,
        MIN(time::DATE) AS start_date
    FROM user_actions
    GROUP BY user_id) AS s1
GROUP BY start_date),

start_date_couriers AS(
SELECT
    start_date,
    COUNT(courier_id) AS new_couriers
FROM
    (SELECT
        courier_id,
        MIN(time::DATE) AS start_date
    FROM courier_actions
    GROUP BY courier_id) AS s2
GROUP BY start_date)


SELECT
  date,
  paying_users,
  active_couriers,
  ROUND(100 * paying_users::DECIMAL / total_users, 2) AS paying_users_share,
  ROUND(100 * active_couriers::DECIMAL / total_couriers, 2) AS active_couriers_share
FROM
    (SELECT
        date,
        active_couriers,
        paying_users
    FROM all_active_couriers LEFT JOIN all_paying_users USING(date)) AS t1
LEFT JOIN 
    (SELECT
        start_date AS date,
        new_users,
        new_couriers,
        (SUM(new_users) OVER (ORDER BY start_date))::INT AS total_users,
        (SUM(new_couriers) OVER (ORDER BY start_date))::INT AS total_couriers
    FROM
    start_date_users LEFT JOIN start_date_couriers USING(start_date)) AS t2
USING(date)
```


### Задача 4
Для каждого дня, представленного в таблице user_actions, рассчитайте следующие показатели:  
Долю пользователей, сделавших в этот день всего один заказ, в общем количестве платящих пользователей.  
Долю пользователей, сделавших в этот день несколько заказов, в общем количестве платящих пользователей.  
Колонки с показателями назовите соответственно single_order_users_share, several_orders_users_share.   
Колонку с датами назовите date. Все показатели с долями необходимо выразить в процентах. При расчёте долей округляйте значения до двух знаков после запятой.  
Результат должен быть отсортирован по возрастанию даты.  
Поля в результирующей таблице: date, single_order_users_share, several_orders_users_share  

```sql
WITH orders_perday_peruser AS(
SELECT 
    user_id,
    time::DATE AS date,
    COUNT(order_id) AS orders_per_day
FROM user_actions
WHERE 
    action = 'create_order' AND
    order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
GROUP BY user_id, date)

SELECT 
    date,
    ROUND(((single_order_users::DECIMAL / all_paying_users) * 100), 2) AS single_order_users_share,
    ROUND(((several_order_users::DECIMAL / all_paying_users) * 100), 2) AS several_orders_users_share
FROM
    (SELECT
        u.date,
        COUNT(DISTINCT u.user_id) AS all_paying_users, 
        COUNT(DISTINCT u.user_id) FILTER (WHERE orders_per_day = 1) AS single_order_users,
        COUNT(DISTINCT u.user_id) FILTER (WHERE orders_per_day >= 2) AS several_order_users
    FROM orders_perday_peruser AS u
    GROUP BY u.date) AS t1
ORDER BY date
```
![image](https://github.com/user-attachments/assets/35e88aa4-b54f-41e8-bb43-51e257ed0d2a)


### Задача 6
На основе данных в таблицах user_actions, courier_actions и orders для каждого дня рассчитайте следующие показатели:  
Число платящих пользователей на одного активного курьера.  
Число заказов на одного активного курьера.  
Колонки с показателями назовите соответственно users_per_courier и orders_per_courier. Колонку с датами назовите date.   
При расчёте показателей округляйте значения до двух знаков после запятой.  
Результирующая таблица должна быть отсортирована по возрастанию даты.  
Поля в результирующей таблице: date, users_per_courier, orders_per_courier  

```sql
WITH all_active_couriers AS(
SELECT
    time::DATE AS date,
    COUNT(DISTINCT courier_id) AS active_couriers
FROM courier_actions
WHERE 
    (action = 'accept_order' OR action ='deliver_order') AND 
    order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
GROUP BY date),

all_paying_users AS(
SELECT
    time::DATE AS date,
    COUNT(DISTINCT user_id) AS paying_users
FROM user_actions
WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
GROUP BY date),

all_delivered_orders AS(
SELECT 
    creation_time::DATE AS date,
    COUNT(DISTINCT order_id) AS all_orders
FROM orders
WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
GROUP BY date)

SELECT
    date,
    ROUND((paying_users / active_couriers::DECIMAL), 2) AS users_per_courier,
    ROUND((all_orders / active_couriers::DECIMAL), 2) AS orders_per_courier
FROM 
    all_active_couriers 
    LEFT JOIN all_paying_users USING(date)
    LEFT JOIN all_delivered_orders USING(date)
ORDER BY date
```


### Задача 7
На основе данных в таблице courier_actions для каждого дня рассчитайте, за сколько минут в среднем курьеры доставляли свои заказы.  
Колонку с показателем назовите minutes_to_deliver. Колонку с датами назовите date.   
При расчёте среднего времени доставки округляйте количество минут до целых значений.   
Учитывайте только доставленные заказы, отменённые заказы не учитывайте.  
Результирующая таблица должна быть отсортирована по возрастанию даты.  
Поля в результирующей таблице: date, minutes_to_deliver  

```sql
WITH acceptance_time AS(
SELECT
    order_id,
    time
FROM courier_actions
WHERE 
    order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order') AND
    action = 'accept_order'),
    
delivery_time AS(
SELECT
    order_id,
    time
FROM courier_actions
WHERE 
    order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order') AND
    action = 'deliver_order')

SELECT
    date, 
    ROUND(AVG(delivery_time))::INTEGER AS minutes_to_deliver
FROM
    (SELECT
        order_id,
        acceptance_time.time::DATE AS date,
        EXTRACT(EPOCH FROM (delivery_time.time - acceptance_time.time)) / 60 AS delivery_time
    FROM 
        acceptance_time LEFT JOIN
        delivery_time USING (order_id)) AS t1
GROUP BY date
ORDER BY date
```

### Задача 8
На основе данных в таблице orders для каждого часа в сутках рассчитайте следующие показатели:  
Число успешных (доставленных) заказов.  
Число отменённых заказов.  
Долю отменённых заказов в общем числе заказов (cancel rate).  
Колонки с показателями назовите соответственно successful_orders, canceled_orders, cancel_rate.   
Колонку с часом оформления заказа назовите hour. При расчёте доли отменённых заказов округляйте значения до трёх знаков после запятой.  
Результирующая таблица должна быть отсортирована по возрастанию колонки с часом оформления заказа.  
Поля в результирующей таблице: hour, successful_orders, canceled_orders, cancel_rate  

```sql
SELECT 
    hour,
    successful_orders,
    canceled_orders,
    ROUND((canceled_orders::DECIMAL/(successful_orders+canceled_orders)), 3) AS cancel_rate
FROM
    (SELECT 
        EXTRACT(HOUR FROM creation_time)::INTEGER AS hour,
        COUNT(order_id) AS canceled_orders
    FROM orders
    WHERE order_id IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
    GROUP BY hour) AS t1
LEFT JOIN 
    (SELECT 
        EXTRACT(HOUR FROM creation_time)::INTEGER AS hour,
        COUNT(*) AS successful_orders
    FROM orders
    WHERE 
        order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order') AND
        order_id IN (SELECT order_id FROM courier_actions WHERE action = 'deliver_order')
    GROUP BY hour) AS t2
USING(hour)
ORDER BY hour
```
![image](https://github.com/user-attachments/assets/dccc03dd-507e-4da7-864a-c818c836d101)

---
### Анализ продуктовых метрик
### Экономика продукта

### Задача 1
Для каждого дня в таблице orders рассчитайте следующие показатели:  
Выручку, полученную в этот день.  
Суммарную выручку на текущий день.  
Прирост выручки, полученной в этот день, относительно значения выручки за предыдущий день.  
Колонки с показателями назовите соответственно revenue, total_revenue, revenue_change. Колонку с датами назовите date.  
Прирост выручки рассчитайте в процентах и округлите значения до двух знаков после запятой.  
Результат должен быть отсортирован по возрастанию даты.  
Поля в результирующей таблице: date, revenue, total_revenue, revenue_change  

```sql
SELECT 
    date,
    revenue,
    total_revenue,
    ROUND((((revenue*100) / LAG(revenue) OVER()) - 100), 2) AS revenue_change
FROM
    (SELECT
        date,
        revenue,
        SUM(revenue) OVER(ORDER BY date) AS total_revenue
    FROM
        (SELECT
            creation_time::DATE AS date,
            SUM(price) AS revenue
        FROM
            (SELECT
                creation_time,
                order_id,
                UNNEST(product_ids) AS product_id
            FROM orders) all_products
        LEFT JOIN products USING(product_id)
        WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')
        GROUP BY date) AS revenue) AS total
```

### Задача 2
Для каждого дня в таблицах orders и user_actions рассчитайте следующие показатели:  
Выручку на пользователя (ARPU) за текущий день.  
Выручку на платящего пользователя (ARPPU) за текущий день.  
Выручку с заказа, или средний чек (AOV) за текущий день.  
Колонки с показателями назовите соответственно arpu, arppu, aov. Колонку с датами назовите date.   
При расчёте всех показателей округляйте значения до двух знаков после запятой.  
Результат должен быть отсортирован по возрастанию даты.   
Поля в результирующей таблице: date, arpu, arppu, aov  

```sql
WITH unnested AS(
SELECT
    DATE(creation_time) AS date,
    order_id,
    UNNEST(product_ids) AS product_id
FROM orders
WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action ='cancel_order')),

paying_users_quantity AS (
  SELECT
    DATE(time) AS date,
    COUNT(DISTINCT user_id) FILTER (WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action = 'cancel_order')) AS paying_users,
    COUNT(DISTINCT user_id) AS all_users
  FROM user_actions
  GROUP BY date)

SELECT
    date,
    ROUND(revenue::DECIMAL / all_users, 2) AS ARPU,
    ROUND(revenue::DECIMAL / paying_users, 2) AS ARPPU,
    ROUND(revenue::DECIMAL / orders_quantity, 2) AS AOV
FROM 
    (SELECT
        date,
        SUM(price) AS revenue,
        COUNT(DISTINCT order_id) AS orders_quantity
    FROM unnested LEFT JOIN  products USING(product_id)
    GROUP BY date)AS t1
LEFT JOIN paying_users_quantity USING(date)
ORDER BY date
```

### Задача 6
Для каждого товара, представленного в таблице products, за весь период времени в таблице orders рассчитайте следующие показатели:  
Суммарную выручку, полученную от продажи этого товара за весь период.  
Долю выручки от продажи этого товара в общей выручке, полученной за весь период.  
Колонки с показателями назовите соответственно revenue и share_in_revenue. Колонку с наименованиями товаров назовите product_name.  
Долю выручки с каждого товара необходимо выразить в процентах. При её расчёте округляйте значения до двух знаков после запятой.  
Товары, округлённая доля которых в выручке составляет менее 0.5%, объедините в общую группу с названием «ДРУГОЕ» (без кавычек), просуммировав округлённые доли этих товаров.  
Результат должен быть отсортирован по убыванию выручки от продажи товара.  
Поля в результирующей таблице: product_name, revenue, share_in_revenue  

```sql
WITH unnested AS(
SELECT 
    creation_time, 
    order_id,
    UNNEST(product_ids) AS product_id
FROM orders),

joined AS(
SELECT 
    creation_time, 
    order_id, 
    unnested.product_id,
    name AS product_name, 
    price
FROM unnested
JOIN products USING(product_id)
WHERE order_id NOT IN (SELECT order_id FROM user_actions WHERE action ='cancel_order')),

base AS(
SELECT
    product_name,
    SUM(price) AS revenue
FROM joined
GROUP BY product_name),

total AS(
SELECT
    SUM(revenue) AS total_revenue
FROM base),

drugoe AS (
SELECT
    revenue,
    ROUND(100 * revenue / total.total_revenue, 2) AS share_in_revenue,
    CASE
        WHEN revenue / total_revenue < 0.005 THEN 'ДРУГОЕ'
        ELSE product_name
    END AS product_name
FROM base CROSS JOIN total)

SELECT
    product_name,
    SUM(revenue) AS revenue,
    SUM(share_in_revenue) AS share_in_revenue
FROM drugoe
GROUP BY product_name
ORDER BY revenue DESC
```
![image](https://github.com/user-attachments/assets/f37dc676-7c7b-41d8-b824-e1afece54eab)


### Задача 7
Для каждого дня в таблицах orders и courier_actions рассчитайте следующие показатели:
Выручку, полученную в этот день.  
Затраты, образовавшиеся в этот день.  
Сумму НДС с продажи товаров в этот день.  
Валовую прибыль в этот день (выручка за вычетом затрат и НДС).  
Суммарную выручку на текущий день.  
Суммарные затраты на текущий день.  
Суммарный НДС на текущий день.  
Суммарную валовую прибыль на текущий день.  
Долю валовой прибыли в выручке за этот день (долю п.4 в п.1).  
Долю суммарной валовой прибыли в суммарной выручке на текущий день (долю п.8 в п.5).  

```sql
SELECT date,
       revenue,
       costs,
       tax,
       gross_profit,
       total_revenue,
       total_costs,
       total_tax,
       total_gross_profit,
       round(gross_profit / revenue * 100, 2) as gross_profit_ratio,
       round(total_gross_profit / total_revenue * 100, 2) as total_gross_profit_ratio
FROM   (SELECT date,
               revenue,
               costs,
               tax,
               revenue - costs - tax as gross_profit,
               sum(revenue) OVER (ORDER BY date) as total_revenue,
               sum(costs) OVER (ORDER BY date) as total_costs,
               sum(tax) OVER (ORDER BY date) as total_tax,
               sum(revenue - costs - tax) OVER (ORDER BY date) as total_gross_profit
        FROM   (SELECT date,
                       orders_packed,
                       orders_delivered,
                       couriers_count,
                       revenue,
                       case when date_part('month',
                                                                                                                                                                      date) = 8 then 120000.0 + 140 * coalesce(orders_packed, 0) + 150 * coalesce(orders_delivered, 0) + 400 * coalesce(couriers_count, 0)
                            when date_part('month',
                                                                                                                                                                      date) = 9 then 150000.0 + 115 * coalesce(orders_packed, 0) + 150 * coalesce(orders_delivered, 0) + 500 * coalesce(couriers_count, 0) end as costs,
                       tax
                FROM   (SELECT creation_time::date as date,
                               count(distinct order_id) as orders_packed,
                               sum(price) as revenue,
                               sum(tax) as tax
                        FROM   (SELECT order_id,
                                       creation_time,
                                       product_id,
                                       name,
                                       price,
                                       case when name in ('сахар', 'сухарики', 'сушки', 'семечки', 'масло льняное', 'виноград', 'масло оливковое', 'арбуз', 'батон', 'йогурт', 'сливки', 'гречка', 'овсянка', 'макароны', 'баранина', 'апельсины', 'бублики', 'хлеб', 'горох', 'сметана', 'рыба копченая', 'мука', 'шпроты', 'сосиски', 'свинина', 'рис', 'масло кунжутное', 'сгущенка', 'ананас', 'говядина', 'соль', 'рыба вяленая', 'масло подсолнечное', 'яблоки', 'груши', 'лепешка', 'молоко', 'курица', 'лаваш', 'вафли', 'мандарины') then round(price/110*10,
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         2)
                                            else round(price/120*20, 2) end as tax
                                FROM   (SELECT order_id,
                                               creation_time,
                                               unnest(product_ids) as product_id
                                        FROM   orders
                                        WHERE  order_id not in (SELECT order_id
                                                                FROM   user_actions
                                                                WHERE  action = 'cancel_order')) t1
                                    LEFT JOIN products using (product_id)) t2
                        GROUP BY date) t3
                    LEFT JOIN (SELECT time::date as date,
                                      count(distinct order_id) as orders_delivered
                               FROM   courier_actions
                               WHERE  order_id not in (SELECT order_id
                                                       FROM   user_actions
                                                       WHERE  action = 'cancel_order')
                                  and action = 'deliver_order'
                               GROUP BY date) t4 using (date)
                    LEFT JOIN (SELECT date,
                                      count(courier_id) as couriers_count
                               FROM   (SELECT time::date as date,
                                              courier_id,
                                              count(distinct order_id) as orders_delivered
                                       FROM   courier_actions
                                       WHERE  order_id not in (SELECT order_id
                                                               FROM   user_actions
                                                               WHERE  action = 'cancel_order')
                                          and action = 'deliver_order'
                                       GROUP BY date, courier_id having count(distinct order_id) >= 5) t5
                               GROUP BY date) t6 using (date)) t7) t8
```


### Маркетинговые-метрики
### Задача 1

На основе таблицы user_actions рассчитайте метрику CAC для двух рекламных кампаний.

```sql
SELECT concat('Кампания № ', ads_campaign) as ads_campaign,
       round(250000.0 / count(distinct user_id), 2) as cac
FROM   (SELECT user_id,
               order_id,
               action,
               case when user_id in (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                     8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788, 8791,
                                     8804, 8810, 8815, 8828, 8830, 8845, 8853, 8859, 8867,
                                     8869, 8876, 8879, 8883, 8896, 8909, 8911, 8933, 8940,
                                     8972, 8976, 8988, 8990, 9002, 9004, 9009, 9019, 9020,
                                     9035, 9036, 9061, 9069, 9071, 9075, 9081, 9085, 9089,
                                     9108, 9113, 9144, 9145, 9146, 9162, 9165, 9167, 9175,
                                     9180, 9182, 9197, 9198, 9210, 9223, 9251, 9257, 9278,
                                     9287, 9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                     9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472, 9476,
                                     9478, 9491, 9494, 9505, 9512, 9518, 9524, 9526, 9528,
                                     9531, 9535, 9550, 9559, 9561, 9562, 9599, 9603, 9605,
                                     9611, 9612, 9615, 9625, 9633, 9652, 9654, 9655, 9660,
                                     9662, 9667, 9677, 9679, 9689, 9695, 9720, 9726, 9739,
                                     9740, 9762, 9778, 9786, 9794, 9804, 9810, 9813, 9818,
                                     9828, 9831, 9836, 9838, 9845, 9871, 9887, 9891, 9896,
                                     9897, 9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                     9998, 9999, 10001, 10013, 10016, 10023, 10030, 10051,
                                     10057, 10064, 10082, 10103, 10105, 10122, 10134, 10135) then 1
                    when user_id in (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                     8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700, 8704,
                                     8712, 8713, 8719, 8729, 8733, 8742, 8748, 8754, 8771,
                                     8794, 8795, 8798, 8803, 8805, 8806, 8812, 8814, 8825,
                                     8827, 8838, 8849, 8851, 8854, 8855, 8870, 8878, 8882,
                                     8886, 8890, 8893, 8900, 8902, 8913, 8916, 8923, 8929,
                                     8935, 8942, 8943, 8949, 8953, 8955, 8966, 8968, 8971,
                                     8973, 8980, 8995, 8999, 9000, 9007, 9013, 9041, 9042,
                                     9047, 9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                     9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161, 9179,
                                     9181, 9183, 9185, 9190, 9196, 9203, 9207, 9226, 9227,
                                     9229, 9230, 9231, 9250, 9255, 9259, 9267, 9273, 9281,
                                     9282, 9289, 9292, 9303, 9310, 9312, 9315, 9327, 9333,
                                     9335, 9337, 9343, 9356, 9368, 9370, 9383, 9392, 9404,
                                     9410, 9421, 9428, 9432, 9437, 9468, 9479, 9483, 9485,
                                     9492, 9495, 9497, 9498, 9500, 9510, 9527, 9529, 9530,
                                     9538, 9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                     9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636, 9658,
                                     9666, 9672, 9684, 9692, 9700, 9704, 9706, 9711, 9719,
                                     9727, 9735, 9741, 9744, 9749, 9752, 9753, 9755, 9757,
                                     9764, 9783, 9784, 9788, 9790, 9808, 9820, 9839, 9841,
                                     9843, 9853, 9855, 9859, 9863, 9877, 9879, 9880, 9882,
                                     9883, 9885, 9901, 9904, 9908, 9910, 9912, 9920, 9929,
                                     9930, 9935, 9939, 9958, 9959, 9961, 9983, 10027, 10033,
                                     10038, 10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                     10073, 10075, 10078, 10079, 10081, 10092, 10106, 10110,
                                     10113, 10131) then 2
                    else 0 end as ads_campaign,
               count(action) filter (WHERE action = 'cancel_order') OVER (PARTITION BY order_id) as is_canceled
        FROM   user_actions) t1
WHERE  ads_campaign in (1, 2)
   and is_canceled = 0
GROUP BY ads_campaign
ORDER BY cac desc
```


### Задача 2

Рассчитайте ROI для каждого рекламного канала.

```sql
SELECT concat('Кампания № ', ads_campaign) as ads_campaign,
       round((sum(price) - 250000.0) / 250000.0 * 100, 2) as roi
FROM   (SELECT ads_campaign,
               user_id,
               order_id,
               product_id,
               price
        FROM   (SELECT ads_campaign,
                       user_id,
                       order_id
                FROM   (SELECT user_id,
                               order_id,
                               case when user_id in (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                                     8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788, 8791,
                                                     8804, 8810, 8815, 8828, 8830, 8845, 8853, 8859, 8867,
                                                     8869, 8876, 8879, 8883, 8896, 8909, 8911, 8933, 8940,
                                                     8972, 8976, 8988, 8990, 9002, 9004, 9009, 9019, 9020,
                                                     9035, 9036, 9061, 9069, 9071, 9075, 9081, 9085, 9089,
                                                     9108, 9113, 9144, 9145, 9146, 9162, 9165, 9167, 9175,
                                                     9180, 9182, 9197, 9198, 9210, 9223, 9251, 9257, 9278,
                                                     9287, 9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                                     9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472, 9476,
                                                     9478, 9491, 9494, 9505, 9512, 9518, 9524, 9526, 9528,
                                                     9531, 9535, 9550, 9559, 9561, 9562, 9599, 9603, 9605,
                                                     9611, 9612, 9615, 9625, 9633, 9652, 9654, 9655, 9660,
                                                     9662, 9667, 9677, 9679, 9689, 9695, 9720, 9726, 9739,
                                                     9740, 9762, 9778, 9786, 9794, 9804, 9810, 9813, 9818,
                                                     9828, 9831, 9836, 9838, 9845, 9871, 9887, 9891, 9896,
                                                     9897, 9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                                     9998, 9999, 10001, 10013, 10016, 10023, 10030, 10051,
                                                     10057, 10064, 10082, 10103, 10105, 10122, 10134, 10135) then 1
                                    when user_id in (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                                     8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700, 8704,
                                                     8712, 8713, 8719, 8729, 8733, 8742, 8748, 8754, 8771,
                                                     8794, 8795, 8798, 8803, 8805, 8806, 8812, 8814, 8825,
                                                     8827, 8838, 8849, 8851, 8854, 8855, 8870, 8878, 8882,
                                                     8886, 8890, 8893, 8900, 8902, 8913, 8916, 8923, 8929,
                                                     8935, 8942, 8943, 8949, 8953, 8955, 8966, 8968, 8971,
                                                     8973, 8980, 8995, 8999, 9000, 9007, 9013, 9041, 9042,
                                                     9047, 9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                                     9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161, 9179,
                                                     9181, 9183, 9185, 9190, 9196, 9203, 9207, 9226, 9227,
                                                     9229, 9230, 9231, 9250, 9255, 9259, 9267, 9273, 9281,
                                                     9282, 9289, 9292, 9303, 9310, 9312, 9315, 9327, 9333,
                                                     9335, 9337, 9343, 9356, 9368, 9370, 9383, 9392, 9404,
                                                     9410, 9421, 9428, 9432, 9437, 9468, 9479, 9483, 9485,
                                                     9492, 9495, 9497, 9498, 9500, 9510, 9527, 9529, 9530,
                                                     9538, 9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                                     9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636, 9658,
                                                     9666, 9672, 9684, 9692, 9700, 9704, 9706, 9711, 9719,
                                                     9727, 9735, 9741, 9744, 9749, 9752, 9753, 9755, 9757,
                                                     9764, 9783, 9784, 9788, 9790, 9808, 9820, 9839, 9841,
                                                     9843, 9853, 9855, 9859, 9863, 9877, 9879, 9880, 9882,
                                                     9883, 9885, 9901, 9904, 9908, 9910, 9912, 9920, 9929,
                                                     9930, 9935, 9939, 9958, 9959, 9961, 9983, 10027, 10033,
                                                     10038, 10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                                     10073, 10075, 10078, 10079, 10081, 10092, 10106, 10110,
                                                     10113, 10131) then 2
                                    else 0 end as ads_campaign,
                               count(action) filter (WHERE action = 'cancel_order') OVER (PARTITION BY order_id) as is_canceled
                        FROM   user_actions) t1
                WHERE  ads_campaign in (1, 2)
                   and is_canceled = 0) t2
            LEFT JOIN (SELECT order_id,
                              unnest(product_ids) as product_id
                       FROM   orders) t3 using(order_id)
            LEFT JOIN products using(product_id)) t4
GROUP BY ads_campaign
ORDER BY roi desc
```

### Задача 3

Для каждой рекламной кампании посчитайте среднюю стоимость заказа привлечённых пользователей за первую неделю использования приложения с 1 по 7 сентября 2022 года.

```sql
SELECT concat('Кампания № ', ads_campaign) as ads_campaign,
       round(avg(user_avg_check), 2) as avg_check
FROM   (SELECT ads_campaign,
               user_id,
               round(avg(order_price), 2) as user_avg_check
        FROM   (SELECT ads_campaign,
                       user_id,
                       order_id,
                       sum(price) as order_price
                FROM   (SELECT ads_campaign,
                               user_id,
                               order_id,
                               product_id,
                               price
                        FROM   (SELECT ads_campaign,
                                       user_id,
                                       order_id
                                FROM   (SELECT user_id,
                                               order_id,
                                               time,
                                               case when user_id in (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                                                     8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788, 8791,
                                                                     8804, 8810, 8815, 8828, 8830, 8845, 8853, 8859, 8867,
                                                                     8869, 8876, 8879, 8883, 8896, 8909, 8911, 8933, 8940,
                                                                     8972, 8976, 8988, 8990, 9002, 9004, 9009, 9019, 9020,
                                                                     9035, 9036, 9061, 9069, 9071, 9075, 9081, 9085, 9089,
                                                                     9108, 9113, 9144, 9145, 9146, 9162, 9165, 9167, 9175,
                                                                     9180, 9182, 9197, 9198, 9210, 9223, 9251, 9257, 9278,
                                                                     9287, 9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                                                     9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472, 9476,
                                                                     9478, 9491, 9494, 9505, 9512, 9518, 9524, 9526, 9528,
                                                                     9531, 9535, 9550, 9559, 9561, 9562, 9599, 9603, 9605,
                                                                     9611, 9612, 9615, 9625, 9633, 9652, 9654, 9655, 9660,
                                                                     9662, 9667, 9677, 9679, 9689, 9695, 9720, 9726, 9739,
                                                                     9740, 9762, 9778, 9786, 9794, 9804, 9810, 9813, 9818,
                                                                     9828, 9831, 9836, 9838, 9845, 9871, 9887, 9891, 9896,
                                                                     9897, 9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                                                     9998, 9999, 10001, 10013, 10016, 10023, 10030, 10051,
                                                                     10057, 10064, 10082, 10103, 10105, 10122, 10134, 10135) then 1
                                                    when user_id in (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                                                     8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700, 8704,
                                                                     8712, 8713, 8719, 8729, 8733, 8742, 8748, 8754, 8771,
                                                                     8794, 8795, 8798, 8803, 8805, 8806, 8812, 8814, 8825,
                                                                     8827, 8838, 8849, 8851, 8854, 8855, 8870, 8878, 8882,
                                                                     8886, 8890, 8893, 8900, 8902, 8913, 8916, 8923, 8929,
                                                                     8935, 8942, 8943, 8949, 8953, 8955, 8966, 8968, 8971,
                                                                     8973, 8980, 8995, 8999, 9000, 9007, 9013, 9041, 9042,
                                                                     9047, 9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                                                     9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161, 9179,
                                                                     9181, 9183, 9185, 9190, 9196, 9203, 9207, 9226, 9227,
                                                                     9229, 9230, 9231, 9250, 9255, 9259, 9267, 9273, 9281,
                                                                     9282, 9289, 9292, 9303, 9310, 9312, 9315, 9327, 9333,
                                                                     9335, 9337, 9343, 9356, 9368, 9370, 9383, 9392, 9404,
                                                                     9410, 9421, 9428, 9432, 9437, 9468, 9479, 9483, 9485,
                                                                     9492, 9495, 9497, 9498, 9500, 9510, 9527, 9529, 9530,
                                                                     9538, 9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                                                     9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636, 9658,
                                                                     9666, 9672, 9684, 9692, 9700, 9704, 9706, 9711, 9719,
                                                                     9727, 9735, 9741, 9744, 9749, 9752, 9753, 9755, 9757,
                                                                     9764, 9783, 9784, 9788, 9790, 9808, 9820, 9839, 9841,
                                                                     9843, 9853, 9855, 9859, 9863, 9877, 9879, 9880, 9882,
                                                                     9883, 9885, 9901, 9904, 9908, 9910, 9912, 9920, 9929,
                                                                     9930, 9935, 9939, 9958, 9959, 9961, 9983, 10027, 10033,
                                                                     10038, 10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                                                     10073, 10075, 10078, 10079, 10081, 10092, 10106, 10110,
                                                                     10113, 10131) then 2
                                                    else 0 end as ads_campaign,
                                               count(action) filter (WHERE action = 'cancel_order') OVER (PARTITION BY order_id) as is_canceled
                                        FROM   user_actions) t1
                                WHERE  ads_campaign in (1, 2)
                                   and is_canceled = 0
                                   and time::date >= '2022-09-01'
                                   and time::date < '2022-09-08') t2
                            LEFT JOIN (SELECT order_id,
                                              unnest(product_ids) as product_id
                                       FROM   orders) t3 using(order_id)
                            LEFT JOIN products using(product_id)) t4
                GROUP BY ads_campaign, user_id, order_id) t5
        GROUP BY ads_campaign, user_id) t6
GROUP BY ads_campaign
ORDER BY avg_check desc
```


### Задача 4

На основе данных в таблице user_actions рассчитайте показатель дневного Retention для всех пользователей, разбив их на когорты по дате первого взаимодействия с нашим приложением.  
В результат включите четыре колонки: месяц первого взаимодействия, дату первого взаимодействия, количество дней, прошедших с даты первого взаимодействия (порядковый номер дня начиная с 0), и само значение Retention.

```sql
SELECT date_trunc('month', start_date)::date as start_month,
       start_date,
       date - start_date as day_number,
       round(users::decimal / max(users) OVER (PARTITION BY start_date), 2) as retention
FROM   (SELECT start_date,
               time::date as date,
               count(distinct user_id) as users
        FROM   (SELECT user_id,
                       time::date,
                       min(time::date) OVER (PARTITION BY user_id) as start_date
                FROM   user_actions) t1
        GROUP BY start_date, time::date) t2
```

### Задача 5

Для каждой рекламной кампании посчитайте Retention 1-го и 7-го дня у привлечённых пользователей.  
В результат включите четыре колонки: колонку с наименованиями кампаний, дату первого взаимодействия с приложением, количество дней, прошедших с даты первого взаимодействия (порядковый номер), и само значение Retention.
```sql
SELECT concat('Кампания № ', ads_campaign) as ads_campaign,
       start_date,
       day_number,
       round(users::decimal / max(users) OVER (PARTITION BY ads_campaign,
                                                            start_date), 2) as retention
FROM   (SELECT ads_campaign,
               start_date,
               date - start_date as day_number,
               count(distinct user_id) as users
        FROM   (SELECT ads_campaign,
                       user_id,
                       date,
                       min(date) OVER (PARTITION BY ads_campaign,
                                                    user_id) as start_date
                FROM   (SELECT user_id,
                               time::date as date,
                               case when user_id in (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                                     8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788, 8791,
                                                     8804, 8810, 8815, 8828, 8830, 8845, 8853, 8859, 8867,
                                                     8869, 8876, 8879, 8883, 8896, 8909, 8911, 8933, 8940,
                                                     8972, 8976, 8988, 8990, 9002, 9004, 9009, 9019, 9020,
                                                     9035, 9036, 9061, 9069, 9071, 9075, 9081, 9085, 9089,
                                                     9108, 9113, 9144, 9145, 9146, 9162, 9165, 9167, 9175,
                                                     9180, 9182, 9197, 9198, 9210, 9223, 9251, 9257, 9278,
                                                     9287, 9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                                     9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472, 9476,
                                                     9478, 9491, 9494, 9505, 9512, 9518, 9524, 9526, 9528,
                                                     9531, 9535, 9550, 9559, 9561, 9562, 9599, 9603, 9605,
                                                     9611, 9612, 9615, 9625, 9633, 9652, 9654, 9655, 9660,
                                                     9662, 9667, 9677, 9679, 9689, 9695, 9720, 9726, 9739,
                                                     9740, 9762, 9778, 9786, 9794, 9804, 9810, 9813, 9818,
                                                     9828, 9831, 9836, 9838, 9845, 9871, 9887, 9891, 9896,
                                                     9897, 9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                                     9998, 9999, 10001, 10013, 10016, 10023, 10030, 10051,
                                                     10057, 10064, 10082, 10103, 10105, 10122, 10134, 10135) then 1
                                    when user_id in (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                                     8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700, 8704,
                                                     8712, 8713, 8719, 8729, 8733, 8742, 8748, 8754, 8771,
                                                     8794, 8795, 8798, 8803, 8805, 8806, 8812, 8814, 8825,
                                                     8827, 8838, 8849, 8851, 8854, 8855, 8870, 8878, 8882,
                                                     8886, 8890, 8893, 8900, 8902, 8913, 8916, 8923, 8929,
                                                     8935, 8942, 8943, 8949, 8953, 8955, 8966, 8968, 8971,
                                                     8973, 8980, 8995, 8999, 9000, 9007, 9013, 9041, 9042,
                                                     9047, 9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                                     9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161, 9179,
                                                     9181, 9183, 9185, 9190, 9196, 9203, 9207, 9226, 9227,
                                                     9229, 9230, 9231, 9250, 9255, 9259, 9267, 9273, 9281,
                                                     9282, 9289, 9292, 9303, 9310, 9312, 9315, 9327, 9333,
                                                     9335, 9337, 9343, 9356, 9368, 9370, 9383, 9392, 9404,
                                                     9410, 9421, 9428, 9432, 9437, 9468, 9479, 9483, 9485,
                                                     9492, 9495, 9497, 9498, 9500, 9510, 9527, 9529, 9530,
                                                     9538, 9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                                     9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636, 9658,
                                                     9666, 9672, 9684, 9692, 9700, 9704, 9706, 9711, 9719,
                                                     9727, 9735, 9741, 9744, 9749, 9752, 9753, 9755, 9757,
                                                     9764, 9783, 9784, 9788, 9790, 9808, 9820, 9839, 9841,
                                                     9843, 9853, 9855, 9859, 9863, 9877, 9879, 9880, 9882,
                                                     9883, 9885, 9901, 9904, 9908, 9910, 9912, 9920, 9929,
                                                     9930, 9935, 9939, 9958, 9959, 9961, 9983, 10027, 10033,
                                                     10038, 10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                                     10073, 10075, 10078, 10079, 10081, 10092, 10106, 10110,
                                                     10113, 10131) then 2
                                    else 0 end as ads_campaign
                        FROM   user_actions) t1
                WHERE  ads_campaign in (1, 2)) t2
        GROUP BY ads_campaign, start_date, date) t3
WHERE  day_number in (0, 1, 7)
```

### Задача 6

Для каждой рекламной кампании для каждого дня посчитайте две метрики:  
Накопительный ARPPU.  
Затраты на привлечение одного покупателя (CAC).

```sql
with main_table as (SELECT ads_campaign,
                           user_id,
                           order_id,
                           time,
                           product_id,
                           price
                    FROM   (SELECT ads_campaign,
                                   user_id,
                                   order_id,
                                   time
                            FROM   (SELECT user_id,
                                           order_id,
                                           time,
                                           case when user_id in (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                                                 8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788, 8791,
                                                                 8804, 8810, 8815, 8828, 8830, 8845, 8853, 8859, 8867,
                                                                 8869, 8876, 8879, 8883, 8896, 8909, 8911, 8933, 8940,
                                                                 8972, 8976, 8988, 8990, 9002, 9004, 9009, 9019, 9020,
                                                                 9035, 9036, 9061, 9069, 9071, 9075, 9081, 9085, 9089,
                                                                 9108, 9113, 9144, 9145, 9146, 9162, 9165, 9167, 9175,
                                                                 9180, 9182, 9197, 9198, 9210, 9223, 9251, 9257, 9278,
                                                                 9287, 9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                                                 9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472, 9476,
                                                                 9478, 9491, 9494, 9505, 9512, 9518, 9524, 9526, 9528,
                                                                 9531, 9535, 9550, 9559, 9561, 9562, 9599, 9603, 9605,
                                                                 9611, 9612, 9615, 9625, 9633, 9652, 9654, 9655, 9660,
                                                                 9662, 9667, 9677, 9679, 9689, 9695, 9720, 9726, 9739,
                                                                 9740, 9762, 9778, 9786, 9794, 9804, 9810, 9813, 9818,
                                                                 9828, 9831, 9836, 9838, 9845, 9871, 9887, 9891, 9896,
                                                                 9897, 9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                                                 9998, 9999, 10001, 10013, 10016, 10023, 10030, 10051,
                                                                 10057, 10064, 10082, 10103, 10105, 10122, 10134, 10135) then 1
                                                when user_id in (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                                                 8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700, 8704,
                                                                 8712, 8713, 8719, 8729, 8733, 8742, 8748, 8754, 8771,
                                                                 8794, 8795, 8798, 8803, 8805, 8806, 8812, 8814, 8825,
                                                                 8827, 8838, 8849, 8851, 8854, 8855, 8870, 8878, 8882,
                                                                 8886, 8890, 8893, 8900, 8902, 8913, 8916, 8923, 8929,
                                                                 8935, 8942, 8943, 8949, 8953, 8955, 8966, 8968, 8971,
                                                                 8973, 8980, 8995, 8999, 9000, 9007, 9013, 9041, 9042,
                                                                 9047, 9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                                                 9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161, 9179,
                                                                 9181, 9183, 9185, 9190, 9196, 9203, 9207, 9226, 9227,
                                                                 9229, 9230, 9231, 9250, 9255, 9259, 9267, 9273, 9281,
                                                                 9282, 9289, 9292, 9303, 9310, 9312, 9315, 9327, 9333,
                                                                 9335, 9337, 9343, 9356, 9368, 9370, 9383, 9392, 9404,
                                                                 9410, 9421, 9428, 9432, 9437, 9468, 9479, 9483, 9485,
                                                                 9492, 9495, 9497, 9498, 9500, 9510, 9527, 9529, 9530,
                                                                 9538, 9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                                                 9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636, 9658,
                                                                 9666, 9672, 9684, 9692, 9700, 9704, 9706, 9711, 9719,
                                                                 9727, 9735, 9741, 9744, 9749, 9752, 9753, 9755, 9757,
                                                                 9764, 9783, 9784, 9788, 9790, 9808, 9820, 9839, 9841,
                                                                 9843, 9853, 9855, 9859, 9863, 9877, 9879, 9880, 9882,
                                                                 9883, 9885, 9901, 9904, 9908, 9910, 9912, 9920, 9929,
                                                                 9930, 9935, 9939, 9958, 9959, 9961, 9983, 10027, 10033,
                                                                 10038, 10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                                                 10073, 10075, 10078, 10079, 10081, 10092, 10106, 10110,
                                                                 10113, 10131) then 2
                                                else 0 end as ads_campaign,
                                           count(action) filter (WHERE action = 'cancel_order') OVER (PARTITION BY order_id) as is_canceled
                                    FROM   user_actions) t1
                            WHERE  ads_campaign in (1, 2)
                               and is_canceled = 0) t2
                        LEFT JOIN (SELECT order_id,
                                          unnest(product_ids) as product_id
                                   FROM   orders) t3 using(order_id)
                        LEFT JOIN products using(product_id))
SELECT concat('Кампания № ', ads_campaign) as ads_campaign,
       concat('Day ', row_number() OVER (PARTITION BY ads_campaign
                                         ORDER BY date) - 1) as day,
       round(sum(revenue) OVER (PARTITION BY ads_campaign
                                ORDER BY date) / paying_users::decimal, 2) as cumulative_arppu,
       cac
FROM   (SELECT ads_campaign,
               time::date as date,
               sum(price) as revenue
        FROM   main_table
        GROUP BY ads_campaign, time::date) t1
    LEFT JOIN (SELECT ads_campaign,
                      count(distinct user_id) as paying_users,
                      round(250000.0 / count(distinct user_id), 2) as cac
               FROM   main_table
               GROUP BY ads_campaign) t2 using (ads_campaign)
```
