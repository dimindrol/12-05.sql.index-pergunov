# Домашнее задание к занятию "Индексы" - Пергунов Д.В

### Задание 1
Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.
```sql

```

### Задание 2
Выполните explain analyze следующего запроса:
Перечислите узкие места;
оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
```
Выполним explain запрос:  
-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=9.5..9.56 rows=602 loops=1)  
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=9.5..9.5 rows=602 loops=1)  
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=8.29..9.35 rows=642 loops=1)  
            -> Sort: c.customer_id, f.title  (actual time=8.28..8.31 rows=642 loops=1)  
                -> Stream results  (cost=23803 rows=15990) (actual time=0.0664..8.06 rows=642 loops=1)  
                    -> Nested loop inner join  (cost=23803 rows=15990) (actual time=0.0626..7.82 rows=642 loops=1)  
                        -> Nested loop inner join  (cost=18207 rows=15990) (actual time=0.0587..6.98 rows=642 loops=1)  
                            -> Nested loop inner join  (cost=12610 rows=15990) (actual time=0.0551..6.16 rows=642 loops=1)  
                                -> Nested loop inner join  (cost=7014 rows=15990) (actual time=0.0503..5.59 rows=642 loops=1)  
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1564 rows=15400) (actual time=0.0405..4.44 rows=634 loops=1)  
                                        -> Table scan on p  (cost=1564 rows=15400) (actual time=0.0325..3.51 rows=16044 loops=1)  
                                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.25 rows=1.04) (actual time=0.00131..0.00168 rows=1.01 loops=634)  
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=752e-6..773e-6 rows=1 loops=642)  
                            -> Single-row index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.00113..0.00115 rows=1 loops=642)  
                        -> Single-row index lookup on f using PRIMARY (film_id=i.film_id)  (cost=0.25 rows=1) (actual time=0.00117..0.00119 rows=1 loops=642)  
```
Узкие места:  
Table Scan - полный скан таблиц достаточно дорогостоящее удовольствие, в данном случае желательно использовать Join для оптимизации запроса  
Temporary table with deduplication- Создание временной таблицы для удаления дупликатов очень ресурсоемкая задача  
Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title ) Суммирование sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title ),требует достаточно много ресурсов  
Исправленный запрос:  
Использование JOIN улучшает читаемость и ускорить запрос.  

```sql
EXPLAIN ANALYZE
SELECT CONCAT(c.last_name, ' ', c.first_name) AS customer_name, 
SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title) AS total_amount
FROM payment p
JOIN rental r ON p.payment_date = r.rental_date
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON i.inventory_id = r.inventory_id
JOIN  film f ON i.film_id = f.film_id
WHERE DATE(p.payment_date) = '2005-07-30';
```
```
-> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=8.27..9.27 rows=642 loops=1)  
    -> Sort: c.customer_id, f.title  (actual time=8.26..8.29 rows=642 loops=1)  
        -> Stream results  (cost=23803 rows=15990) (actual time=0.1..8.05 rows=642 loops=1)  
            -> Nested loop inner join  (cost=23803 rows=15990) (actual time=0.0961..7.81 rows=642 loops=1)  
                -> Nested loop inner join  (cost=18207 rows=15990) (actual time=0.0925..6.98 rows=642 loops=1)  
                    -> Nested loop inner join  (cost=12610 rows=15990) (actual time=0.0892..6.16 rows=642 loops=1)  
                        -> Nested loop inner join  (cost=7014 rows=15990) (actual time=0.0842..5.57 rows=642 loops=1)  
                            -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1564 rows=15400) (actual time=0.0722..4.47 rows=634 loops=1)  
                                -> Table scan on p  (cost=1564 rows=15400) (actual time=0.061..3.51 rows=16044 loops=1)  
                            -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.25 rows=1.04) (actual time=0.00124..0.00161 rows=1.01 loops=634)  
                        -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=773e-6..795e-6 rows=1 loops=642)  
                    -> Single-row index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.00113..0.00115 rows=1 loops=642)  
                -> Single-row index lookup on f using PRIMARY (film_id=i.film_id)  (cost=0.25 rows=1) (actual time=0.00116..0.00119 rows=1 loops=642)  
```    
Индексы таблиц которые используется в запросе, помогут ускорить запрос:
```sql
CREATE INDEX idx_payment_date ON payment (payment_date);
CREATE INDEX idx_rental_date ON rental (rental_date);
CREATE INDEX idx_customer_id ON customer (customer_id);
CREATE INDEX idx_inventory_id ON inventory (inventory_id);
CREATE INDEX idx_film_id ON film (film_id);
```

