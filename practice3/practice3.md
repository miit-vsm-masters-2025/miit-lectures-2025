# PostgreSQL

Документация: https://www.postgresql.org/docs/17/index.html

Документация на русском: https://postgrespro.ru/docs/postgrespro/17

## Подготовка

```
# docker-compose.yml
# Use `docker-compose up -d` to start
services:
  postgres:
    # See https://hub.docker.com/_/postgres
    image: postgres:17-alpine
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=5FCCBA11-B16E-4C47-BFDF-06051C17226D # random password
      - POSTGRES_USER=postgres
      - POSTGRES_DB=postgres
volumes:
  postgres_data:
```

- `docker compose up -d` - поднимаем контейнер с postgres
- `docker logs -f practice3-postgres-1` - смотрим логи, убеждаемся что все стартануло корректно
- `docker exec -ti -u postgres practice3-postgres-1 psql` - получаем доступ к командной строке постгреса

### Создаем таблицу и заполняем ее данными

```sql
CREATE TABLE accounts(id serial PRIMARY KEY, username varchar(255), balance int);
INSERT INTO accounts(username, balance) VALUES ('a', 100), ('b', 110);
```

### Проверяем

```sql
postgres=# SELECT * FROM accounts;
 id | username | balance
----+----------+---------
  1 | a        |     100
  2 | b        |     110
(2 rows)
```

## Индексы, сложность поиска и EXPLAIN

Скорость поиска по таблице отличается в зависимости от того, есть ли у нас индекс по нужной колонке, или придется
сканировать таблицу целиком в поисках нужных данных.

Поиск по PRIMARY KEY - всегда быстрый, индекс по нему создается автоматически.

По остальным колонкам мы можем создавать индексы вручную. Но не стоит этим злоупотреблять: дополнительные индексы
замедляют запись и требуют дополнительного места на диске.

Понять, как Postgres будет выполнять наш запрос можно при помощи запроса EXPLAIN (показывает только план) или EXPLAIN
ANALYZE (дополнительно выполняет запрос и замеряет фактическое время выполнения):

```postgresql
postgres=# EXPLAIN ANALYSE SELECT * FROM accounts WHERE id = 2;
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Index Scan using accounts_pkey on accounts  (cost=0.14..8.16 rows=1 width=524) (actual time=0.021..0.022 rows=1 loops=1)
   Index Cond: (id = 2)
 Planning Time: 0.219 ms
 Execution Time: 0.049 ms
(4 rows)
```

В примере выше виден факт использования индекса: `Index Scan`.

Но если мы начнем искать по колонке без индекса - увидим `Seq Scan` (последовательное прохождение по строкам в таблице).
По мере роста таблицы выполнение таких запросов будет занимать все больше времени.

```postgresql
postgres=# EXPLAIN ANALYSE SELECT * FROM accounts WHERE balance = 100;
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 Seq Scan on accounts  (cost=0.00..11.75 rows=1 width=524) (actual time=0.119..0.122 rows=1 loops=1)
   Filter: (balance = 100)
   Rows Removed by Filter: 1
 Planning Time: 0.508 ms
 Execution Time: 0.266 ms
(5 rows)
```

## Типы индексов

https://postgrespro.ru/docs/postgrespro/17/indexes-types

### B-tree

Используется по умолчанию, и скорее всего именно он вам и нужен.

- Под капотом - дерево. Сложность поиска: `O(log n)`.
- Поддерживает очень широкий перечень операций: `<   <=   =   >=   >`, LIKE по префиксу строки.
- Позволяет эффективно получать отсортированные результаты по ключу, по которому построен индекс

```postgresql
postgres=# CREATE INDEX accounts_balance ON accounts USING btree (balance);
CREATE INDEX
postgres=# EXPLAIN ANALYSE SELECT * FROM accounts WHERE balance = 100;
                                             QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on accounts  (cost=0.00..1.02 rows=1 width=524) (actual time=0.031..0.033 rows=1 loops=1)
   Filter: (balance = 100)
   Rows Removed by Filter: 1
 Planning Time: 1.540 ms
 Execution Time: 0.156 ms
(5 rows)

postgres=# EXPLAIN ANALYSE SELECT * FROM accounts WHERE balance > 0;
                                             QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on accounts  (cost=0.00..1.02 rows=1 width=524) (actual time=0.095..0.097 rows=2 loops=1)
   Filter: (balance > 0)
 Planning Time: 0.395 ms
 Execution Time: 0.305 ms
(4 rows)
```

### Hash

Пример альтернативного типа индекса. Использует под капотом механизм, аналогичный hash-таблицам.

Работает за константное время `O(1)`, но позволяет находить лишь точные совпадения (`=`).

```postgresql
postgres=# CREATE INDEX accounts_username ON accounts USING hash (username);
CREATE INDEX
postgres=# EXPLAIN ANALYSE SELECT * FROM accounts WHERE username = 'a';
                                             QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on accounts  (cost=0.00..1.02 rows=1 width=524) (actual time=0.151..0.154 rows=1 loops=1)
   Filter: ((username)::text = 'a'::text)
   Rows Removed by Filter: 1
 Planning Time: 0.811 ms
 Execution Time: 0.200 ms
(5 rows)
```

Внезапно мы видим, что постгрес предпочел использовать seq scan вместо hash-индекса. Дело в том, что таблица слишком
мала, и так дешевле. Но мы можем заставить использовать индексы при помощи `SET enable_seqscan = off;`.

```postgresql
postgres=#  SET enable_seqscan = off;
SET
postgres=# EXPLAIN ANALYSE SELECT * FROM accounts WHERE username = 'a';
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Index Scan using accounts_username on accounts  (cost=0.00..12.02 rows=1 width=524) (actual time=0.136..0.137 rows=1 loops=1)
   Index Cond: ((username)::text = 'a'::text)
 Planning Time: 0.420 ms
 Execution Time: 0.206 ms
(4 rows)
```

Важно заметить, что `enable_seqscan = off` не запрещает seq scan полностью: он все еще может использоваться в ситуациях,
когда иного выхода нет (т.е. подходящего индекса нет).

Hash-индексы не поддерживают LIKE-операции даже по префиксам, давайте в этом убедимся:

```postgresql
postgres=# SET enable_seqscan = off;
postgres=# EXPLAIN ANALYSE SELECT * FROM accounts WHERE username LIKE 'a%';
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on accounts  (cost=10000000000.00..10000000001.02 rows=1 width=524) (actual time=70.459..70.467 rows=1 loops=1)
   Filter: ((username)::text ~~ 'a%'::text)
   Rows Removed by Filter: 1
 Planning Time: 0.632 ms
 JIT:
   Functions: 2
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 0.234 ms (Deform 0.045 ms), Inlining 35.967 ms, Optimization 18.358 ms, Emission 16.066 ms, Total 70.625 ms
 Execution Time: 120.570 ms
(9 rows)
```

## Транзакции и изоляция транзакций
