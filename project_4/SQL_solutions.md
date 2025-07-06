
## Симулятор SQL Karpov.Courses - Мои ответы для Модуля 2

[Построение дашбордов](#построение-дашбордов)

[Анализ продуктовых метрик](#анализ-продуктовых-метрик)

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
