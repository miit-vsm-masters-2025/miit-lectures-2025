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

https://postgrespro.ru/docs/postgrespro/17/transactions

В postgres есть транзакции. Они гарантируют, что какой-то набор изменений либо применится целиком, либо не применится
совсем.

Транзакция начинается командой BEGIN (или START TRANSACTION) и завершается либо командой COMMIT, либо ROLLBACK.

Если в процессе транзакции были ошибки - по умолчанию COMMIT в конце приведет к ROLLBACK. Разные GUI-клиенты могут
неявно отлавливать такие ошибки за счет использования SAVEPOINT и ROLLBACK TO SAVEPOINT (например так делают продукты
JetBrains).

```postgresql
SELECT * FROM accounts;

BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
UPDATE accounts SET balance = balance + 50 WHERE id = 2;

-- В рамках транзакции видим собственные изменения
SELECT * FROM accounts;

-- Ошибка: пытаемся создать новую строку с уже существующим ID
INSERT INTO accounts(id, balance) VALUES (1, 100);
COMMIT;

-- Изменения не применились
SELECT * FROM accounts;
```

По умолчанию транзакция видит собственные незакомиченные изменения и закомиченные изменения из соседних транзакций. Это
дефолтный уровень изоляции транзакций - READ COMMITTED.

## Уровни изоляции транзакций

https://en.wikipedia.org/wiki/Isolation_(database_systems)

Теоретически существует четыре уровня изоляции транзакций (постгрес поддерживает только три):

- READ UNCOMMITTED
    - Транзакции видят "грязные" (незакомиченные) изменения друг друга
    - Теоретически очень дешевый и быстрый
    - **Не поддерживается в PostgreSQL**
- READ COMMITTED
    - Транзакции видят только закомиченные изменения из других транзакций
    - Это уровень изоляции транзакций по умолчанию в PostgreSQL
    - Если в середине нашей транзакции кто-то закоммитит изменения - мы их увидим. Это называется Non-repeatable reads.
    - Потенциально возможны потерянные изменения
        - Транзакция A прочитала (SELECT) значение, сделала какие-то вычисления в памяти и сделала UPDATE.
        - Между SELECT и UPDATE (из транзакции A), транзакция B успела сделать COMMIT, изменив изначальные данные
        - Получается, что вычисления и UPDATE в транзакции A не учтут изменения из транзакции B (т.к. мы не сделали
          повторный SELECT в A)
- REPEATABLE READ
    - Гарантирует, что SELECT в течение всей транзакции будет возвращать одинаковые значения (даже если кто-то успеет их
      изменить коммитом параллельной транзакции)
    - Но при попытке сделать UPDATE по строке, которую за время нашей транзакции кто-то успел изменить - получим ошибку
- SERIALIZABLE
    - Обеспечивает иллюзию последовательного исполнения транзакций.
    - Не пересекающиеся транзакции продолжают выполняться параллельно
    - Вы все так же можете получить ошибку вида `could not serialize access due to read/write dependencies` или
      `could not serialize access due to concurrent update`, тогда транзакцию придется повторить.

### Пример с REPEATABLE READ

#### Transaction A

```postgresql
postgres=# BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN
postgres=*# SELECT * FROM accounts WHERE id = 1;
id | username | balance
----+----------+---------
1 | a        |     100
(1 row)

postgres=*# -- Тут происходит некоторая пауза, в которой наш прикладной код считает новое значение баланса
postgres=*# -- Параллельно транзакция B уже успела сделать UPDATE и COMMIT
postgres=*# -- Важно, что REPEATABLE READ продолжает при SELECT'ах показывать старое значение
postgres=*# SELECT * FROM accounts WHERE id = 1;
id | username | balance
----+----------+---------
1 | a        |     100
(1 row)

postgres=*# -- Но при попытке обновить устаревшее значение - мы получим ошибку.
postgres=*# UPDATE accounts SET balance = 101 WHERE id = 1;
ERROR:  could not serialize access due to concurrent update
postgres=!# COMMIT;
ROLLBACK
postgres=# -- После отката транзакции мы снова видим изменения, закомиченные транзакцией B
postgres=# SELECT * FROM accounts WHERE id = 1;
 id | username | balance
----+----------+---------
  1 | a        |     102
(1 row)
```

#### Transaction B

```postgresql
postgres=# BEGIN;
BEGIN
postgres=*# SELECT * FROM accounts WHERE id = 1;
 id | username | balance
----+----------+---------
  1 | a        |     100
(1 row)

postgres=*# UPDATE accounts SET balance = 102 WHERE id = 1;
UPDATE 1
postgres=*# COMMIT;
COMMIT
postgres=# SELECT * FROM accounts WHERE id = 1;
 id | username | balance
----+----------+---------
  1 | a        |     102
(1 row)
```

### Write skew: различие поведения REPEATABLE READ и SERIALIZABLE

#### Сценарий

У нас есть таблица, в которой отмечаются дежурные врачи. По правилам - в любой момент времени хотя бы один врач должен
быть на дежурстве.

```postgresql
CREATE TABLE doctors (
    id SERIAL PRIMARY KEY,
    on_call BOOLEAN NOT NULL
);

INSERT INTO doctors (on_call) VALUES (TRUE), (TRUE);
```

В интерфейсе у врача есть кнопка "уйти с дежурства". Допустим, что сейчас дежурят два врача и они нажимают ее одновременно.

#### Что будет в REPEATABLE READ
```postgresql
postgres=# -- Transaction A
postgres=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN
postgres=*# -- Проверяем, есть ли кто-то на смене
postgres=*# SELECT count(*) FROM doctors WHERE on_call = TRUE;
 count
-------
     2
(1 row)

postgres=*#
postgres=*# -- Снимаем себя с дежурства
postgres=*# UPDATE doctors SET on_call = FALSE WHERE id = 1;
UPDATE 1
postgres=*#
postgres=*# -- Выполняем транзакцию B, затем коммитим A
postgres=*# COMMIT;
COMMIT
postgres=# -- Видим, что дежурных врачей больше нет
postgres=# SELECT count(*) FROM doctors WHERE on_call = TRUE;
 count
-------
     0
(1 row)
```

```postgresql
-- Transaction B
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Проверяем, есть ли кто-то на смене. Видим цифру 2, т.к. snapshot isolation
SELECT count(*) FROM doctors WHERE on_call = TRUE;

-- Снимаем себя с дежурства
UPDATE doctors SET on_call = FALSE WHERE id = 2;

COMMIT;
```