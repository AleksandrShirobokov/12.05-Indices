# «Индексы»
# Александр Широбоков
## Задание 1
### Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.
![Снимок экрана (233)](https://github.com/AleksandrShirobokov/12.05-Indices/assets/69298696/2473c8ae-d0fd-4375-a3e4-30c16335443d)
```
SELECT (SUM(INDEX_LENGTH) / SUM(DATA_LENGTH)) * 100 AS отношение_индекса_к_таблице
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'sakila';
```
## Задание 2
### Выполните explain analyze следующего запроса:
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
 - перечислите узкие места;
 - оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

### Анализ:
 - Таблица payment используется для фильтрации данных по дате, но нет индекса на поле payment_date, что может приводить к полному сканированию таблицы. Думаю будет лучше добавить индекс на поле payment_date в таблице payment для более эффективной фильтрации.
 - Присутствуют несколько таблиц в операторе FROM без явного указания связей через оператор JOIN. Вместо перечисления таблиц через запятую, буду использовать явные операторы JOIN для связи таблиц по соответствующим полям. Это поможет оптимизатору запросов определить более эффективный план выполнения запроса.
 - Использование функции concat в операторе SELECT с двумя полными именами может замедлить выполнение запроса. Вместо использую оператор CONCAT_WS для объединения имен с пробелом между ними, что может быть более эффективным.

 - **Оптимизированный запрос может выглядеть следующим образом:**

![Снимок экрана (235)](https://github.com/AleksandrShirobokov/12.05-Indices/assets/69298696/2b1dddbd-a59d-43ec-810c-7b1d7ae02b14)
```
-- Создание необходимых индексов
CREATE INDEX idx_rental_date ON rental (rental_date);
CREATE INDEX idx_payment_date ON payment (payment_date);

-- Оптимизированный запрос
SELECT DISTINCT CONCAT_WS(c.last_name,' ', c.first_name) AS full_name, SUM(p.amount) AS total_amount
FROM payment p
JOIN rental r ON p.payment_date = r.rental_date
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON r.inventory_id = i.inventory_id
WHERE p.payment_date >= '2005-07-30' AND p.payment_date < '2005-07-31'
GROUP BY c.customer_id;
```
 - **Также хочу отметить, для того, чтобы присутствовало разделение между last и first name, необходимо команду CONCAT_WS заменить на CONCAT, и получим следующий результат:**

![Снимок экрана (236)](https://github.com/AleksandrShirobokov/12.05-Indices/assets/69298696/81055d4e-8561-48bc-8602-b31c400f65da)

 - **explain analyze**: *время на обработку значительно сократилось*
```
-> -> Limit: 200 row(s)  (actual time=19..19.1 rows=200 loops=1)
    -> Sort with duplicate removal: full_name, total_amount  (actual time=19..19.1 rows=200 loops=1)
        -> Table scan on <temporary>  (actual time=18.1..18.3 rows=391 loops=1)
            -> Aggregate using temporary table  (actual time=18.1..18.1 rows=391 loops=1)
                -> Nested loop inner join  (cost=799 rows=645) (actual time=0.0695..15.7 rows=642 loops=1)
                    -> Nested loop inner join  (cost=576 rows=645) (actual time=0.0634..12.6 rows=642 loops=1)
                        -> Nested loop inner join  (cost=351 rows=634) (actual time=0.0425..5.37 rows=634 loops=1)
                            -> Filter: ((r.rental_date >= TIMESTAMP'2005-07-30 00:00:00') and (r.rental_date < TIMESTAMP'2005-07-31 00:00:00'))  (cost=129 rows=634) (actual time=0.029..1.91 rows=634 loops=1)
                                -> Covering index range scan on r using rental_date over ('2005-07-30 00:00:00' <= rental_date < '2005-07-31 00:00:00')  (cost=129 rows=634) (actual time=0.0263..1.34 rows=634 loops=1)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.00491..0.00499 rows=1 loops=634)
                        -> Index lookup on p using idx_payment_date (payment_date=r.rental_date)  (cost=0.254 rows=1.02) (actual time=0.00862..0.0109 rows=1.01 loops=634)
                    -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.246 rows=1) (actual time=0.0042..0.00431 rows=1 loops=642)

```
## Задание 3
### Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.
 - GIN (Generalized Inverted Index): Этот тип индекса используется для поиска по элементам не только простых данных, таких как числа или строки, но и для поиска по сложным структурам данных, таким как массивы, JSON-объекты или полнотекстовые документы. GIN-индекс позволяет эффективно выполнять операции поиска и запросы, связанные с поиском с использованием таких структур данных.

 - GiST (Generalized Search Tree): Этот тип индекса предоставляет возможность создавать пользовательские алгоритмы индексирования для различных типов данных. GiST-индексы поддерживают поиск на основе разных алгоритмов, таких как поиск по диапазонам, полнотекстовый поиск, географический поиск и другие.

 - BRIN (Block Range Index): Этот тип индекса используется для индексирования больших таблиц с отсортированными данными. BRIN-индекс делит данные на блоки и сохраняет минимальные и максимальные значения блоков. Это позволяет эффективно выполнить операции поиска и агрегации на основе диапазонов значений.

 - SP-GiST (Space-Partitioned Generalized Search Tree): Этот тип индекса используется для поиска и индексирования сложных данных, таких как геометрические формы, текстовые документы и другие. SP-GiST-индексы позволяют создавать пользовательские алгоритмы индексирования для различных типов данных.

 - B-tree с сортировкой по обратному порядку: В PostgreSQL B-tree-индексы поддерживают возможность создания индексов, отсортированных по обратному порядку. Это полезно, когда требуется выполнить операции сортировки и поиска в обратном порядке.
