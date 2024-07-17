#### Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
сделал в GC

#### Установить на него PostgreSQL 15 с дефолтными настройками
```
void@ubmvcc:~$ sudo apt install postgresql-16

...
```

#### Создать БД для тестов: выполнить pgbench -i postgres
```
void@ubmvcc:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \q
void@ubmvcc:~$ pgbench -i -U postgres -d testdb
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.10 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.57 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.41 s, vacuum 0.08 s, primary keys 0.08 s).
void@ubmvcc:~$
```

#### Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
```
void@ubmvcc:~$ pgbench -c8 -P 6 -T 60 -U postgres testdb
Password:
pgbench (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 757.3 tps, lat 10.418 ms stddev 5.969, 0 failed
progress: 12.0 s, 735.7 tps, lat 10.866 ms stddev 6.186, 0 failed
progress: 18.0 s, 740.3 tps, lat 10.810 ms stddev 6.292, 0 failed
progress: 24.0 s, 758.7 tps, lat 10.538 ms stddev 5.858, 0 failed
progress: 30.0 s, 687.5 tps, lat 11.637 ms stddev 9.731, 0 failed
progress: 36.0 s, 727.5 tps, lat 10.996 ms stddev 6.270, 0 failed
progress: 42.0 s, 748.3 tps, lat 10.687 ms stddev 5.770, 0 failed
progress: 48.0 s, 726.5 tps, lat 11.014 ms stddev 6.229, 0 failed
progress: 54.0 s, 785.5 tps, lat 10.182 ms stddev 5.802, 0 failed
progress: 60.0 s, 705.3 tps, lat 11.336 ms stddev 6.663, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 44244
number of failed transactions: 0 (0.000%)
latency average = 10.835 ms
latency stddev = 6.551 ms
initial connection time = 76.886 ms
tps = 738.093402 (without initial connection time)
void@ubmvcc:~$
```

#### Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла. Протестировать заново
```
void@ubmvcc:~$ pgbench -c8 -P 6 -T 60 -U postgres testdb
Password:
pgbench (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 641.8 tps, lat 12.273 ms stddev 7.015, 0 failed
progress: 12.0 s, 647.5 tps, lat 12.339 ms stddev 7.332, 0 failed
progress: 18.0 s, 649.7 tps, lat 12.325 ms stddev 7.065, 0 failed
progress: 24.0 s, 645.2 tps, lat 12.393 ms stddev 7.108, 0 failed
progress: 30.0 s, 660.2 tps, lat 12.122 ms stddev 6.753, 0 failed
progress: 36.0 s, 580.0 tps, lat 13.788 ms stddev 7.814, 0 failed
progress: 42.0 s, 628.8 tps, lat 12.722 ms stddev 7.183, 0 failed
progress: 48.0 s, 717.0 tps, lat 11.152 ms stddev 6.310, 0 failed
progress: 54.0 s, 660.0 tps, lat 12.126 ms stddev 6.833, 0 failed
progress: 60.0 s, 619.5 tps, lat 12.903 ms stddev 7.204, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 38706
number of failed transactions: 0 (0.000%)
latency average = 12.384 ms
latency stddev = 7.084 ms
initial connection time = 83.913 ms
tps = 645.743860 (without initial connection time)
void@ubmvcc:~$
```

#### Что изменилось и почему?
Выросли тайминги latency и упал tps потому, что мы уменьшили максимальное кол-во соединений

#### Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
```
-- Создание таблицы
CREATE TABLE random_text (
    id SERIAL PRIMARY KEY,
    text_field TEXT
);

-- Функция для генерации случайного текста
CREATE OR REPLACE FUNCTION generate_random_text(length INT) RETURNS TEXT AS $$
DECLARE
    chars TEXT := 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    result TEXT := '';
    i INT := 0;
BEGIN
    FOR i IN 1..length LOOP
        result := result || substring(chars FROM (floor(random() * length(chars) + 1)::int) FOR 1);
    END LOOP;
    RETURN result;
END;

-- Вставка данных
INSERT INTO random_text (text_field)
SELECT generate_random_text(10)
FROM generate_series(1, 1000000);
```

#### Посмотреть размер файла с таблицей
```
select pg_total_relation_size('random_text');

pg_total_relation_size|
----------------------+
            66 822 144|
```

#### 5 раз обновить все строчки и добавить к каждой строчке любой символ
```
-- Функция для обновления строк, добавляя случайный символ
CREATE OR REPLACE FUNCTION update_random_text() RETURNS VOID AS $$
DECLARE
    chars TEXT := 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    random_char TEXT;
BEGIN
    -- Обновить все строки, добавляя случайный символ
    UPDATE random_text
    SET text_field = text_field || substring(chars FROM (floor(random() * length(chars) + 1)::int) FOR 1);
END;
$$ LANGUAGE plpgsql;

-- Запуск обновления 5 раз
DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..5 LOOP
        PERFORM update_random_text();
    END LOOP;
END;
```

#### Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```
-- Количество мертвых строк в таблице
SELECT relname AS table_name, n_dead_tup AS dead_tuples
FROM pg_stat_all_tables
WHERE relname = 'random_text';

table_name |dead_tuples|
-----------+-----------+
random_text|          0|
```

Автовакуум успел стриггериться до того как я выполнил запрос, поэтому к-во мертвых строк == 0

```
-- Дата последнего автовакуума
SELECT relname AS table_name, last_autovacuum
FROM pg_stat_all_tables
WHERE relname = 'random_text';

table_name |last_autovacuum              |
-----------+-----------------------------+
random_text|2024-07-17 11:46:17.054 +0300|
```

#### 5 раз обновить все строчки и добавить к каждой строчке любой символ. Посмотреть размер файла с таблицей
```
pg_total_relation_size|
----------------------+
           411 197 440|
```

#### Отключить Автовакуум на конкретной таблице
```
ALTER TABLE random_text SET (autovacuum_enabled = false);
```

#### 10 раз обновить все строчки и добавить к каждой строчке любой символ. Посмотреть размер файла с таблицей
```
select pg_total_relation_size('random_text');

pg_total_relation_size|
----------------------+
           821 673 984|
```
```
-- Количество мертвых строк в таблице
SELECT relname AS table_name, n_dead_tup AS dead_tuples
FROM pg_stat_all_tables
WHERE relname = 'random_text';

table_name |dead_tuples|
-----------+-----------+
random_text| 10 000 000|
```

#### Объясните полученный результат
Размер таблицы значительно вырос потому, что появилось большое кол-во мёртвых строк после `UPDATE` каждой строки, что в сущности есть создание новой и удаление старой строки. Этот процесс и плодит мёртвые (удаленные) строки

#### Задание со *: Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. Не забыть вывести номер шага цикла.
```
DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..10 LOOP
        -- Обновление всех строк таблицы, добавляя случайный символ к text_field
        UPDATE random_text
        SET text_field = text_field || chr((65 + random() * 25)::integer); 

        -- Вывод номера шага цикла
        RAISE NOTICE 'Step %', i;
    END LOOP;
END;
$$;
```
