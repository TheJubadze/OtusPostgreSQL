#### На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
На ВМ1:

```
void@ub10replica1:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE TABLE test (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     data TEXT
postgres(# );
CREATE TABLE
postgres=#
postgres=# CREATE TABLE test2 (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     data TEXT
postgres(# );
CREATE TABLE
```

#### Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
На ВМ1:

```
postgres=# CREATE PUBLICATION test_pub FOR TABLE test;
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to "logical" before creating subscriptions.
CREATE PUBLICATION
postgres=#
```

Поправил параметры кластера, установив:
```
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Рестартанул кластер

```
postgres=# CREATE SUBSCRIPTION test2_sub
CONNECTION 'host=34.31.133.62 dbname=postgres user=postgres password=222'
PUBLICATION test2_pub;
WARNING:  publication "test2_pub" does not exist on the publisher
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
postgres=#
```


#### На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
На ВМ2:

```
void@ub10replica2:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE TABLE test2 (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     data TEXT
postgres(# );
CREATE TABLE
postgres=#
postgres=# CREATE TABLE test (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     data TEXT
postgres(# );
CREATE TABLE
postgres=#
```

#### Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
На ВМ2:

Поправил параметры кластера, установив:
```
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Рестартанул кластер

```
void@ub10replica2:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE PUBLICATION test2_pub FOR TABLE test2;
CREATE PUBLICATION
postgres=# CREATE SUBSCRIPTION test_sub
CONNECTION 'host=34.121.206.162 dbname=postgres user=postgres password=111'
PUBLICATION test_pub;
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
postgres=#
```

#### 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
На ВМ3:

```
postgres=# CREATE SUBSCRIPTION test_sub1
CONNECTION 'host=34.121.206.162 dbname=postgres user=postgres password=111'
PUBLICATION test_pub;
NOTICE:  created replication slot "test_sub1" on publisher
CREATE SUBSCRIPTION
postgres=#

postgres=# CREATE SUBSCRIPTION test2_sub2
CONNECTION 'host=34.31.133.62 dbname=postgres user=postgres password=222'
PUBLICATION test2_pub;
NOTICE:  created replication slot "test2_sub2" on publisher
CREATE SUBSCRIPTION
postgres=# 
```

#### Проверим, что у нас получилось
На ВМ1 сгенерим данных в таблицу test:
```
DO $$
BEGIN
    FOR i IN 1..10 LOOP
        INSERT INTO test (data)
        VALUES (substring(md5(random()::text) from 1 for 10));
    END LOOP;
END $$;
```
```
void@ub10replica1:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 id |    data
----+------------
  1 | 99059e9230
  2 | 251b3b39e6
  3 | e42c4fe399
  4 | e8fb43124d
  5 | db040669f9
  6 | 81cf053571
  7 | 0693c0ed1c
  8 | 89111281e5
  9 | 223a825d27
 10 | 946f4f64c1
(10 rows)
```

На ВМ2 проверим, есть ли эти данные в таблице:
```
void@ub10replica2:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 id |    data
----+------------
  1 | 99059e9230
  2 | 251b3b39e6
  3 | e42c4fe399
  4 | e8fb43124d
  5 | db040669f9
  6 | 81cf053571
  7 | 0693c0ed1c
  8 | 89111281e5
  9 | 223a825d27
 10 | 946f4f64c1
(10 rows)
```

Далее, на ВМ2 сгенерим данные для таблицы test2:
```
DO $$
BEGIN
    FOR i IN 1..10 LOOP
        INSERT INTO test2 (data)
        VALUES (substring(md5(random()::text) from 1 for 10));
    END LOOP;
END $$;
```
```
void@ub10replica2:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test2;
 id |    data
----+------------
  1 | 2d47e2e298
  2 | 2da94cb58b
  3 | 2ae2e27a57
  4 | fb4b6fb4dd
  5 | 9526236e00
  6 | 58bdcf3688
  7 | 3b826c456e
  8 | 444d51fa14
  9 | da6df73dc0
 10 | ba26cc9f7f
(10 rows)
```

На ВМ1 проверим, есть ли эти данные в таблице:
```
void@ub10replica1:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test2;
 id |    data
----+------------
  1 | 2d47e2e298
  2 | 2da94cb58b
  3 | 2ae2e27a57
  4 | fb4b6fb4dd
  5 | 9526236e00
  6 | 58bdcf3688
  7 | 3b826c456e
  8 | 444d51fa14
  9 | da6df73dc0
 10 | ba26cc9f7f
(10 rows)
```

На ВМ3 проверим данные в обеих таблицах:
```
void@ub10replica3:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 id |    data
----+------------
  1 | 99059e9230
  2 | 251b3b39e6
  3 | e42c4fe399
  4 | e8fb43124d
  5 | db040669f9
  6 | 81cf053571
  7 | 0693c0ed1c
  8 | 89111281e5
  9 | 223a825d27
 10 | 946f4f64c1
(10 rows)

postgres=# select * from test2;
 id |    data
----+------------
  1 | 2d47e2e298
  2 | 2da94cb58b
  3 | 2ae2e27a57
  4 | fb4b6fb4dd
  5 | 9526236e00
  6 | 58bdcf3688
  7 | 3b826c456e
  8 | 444d51fa14
  9 | da6df73dc0
 10 | ba26cc9f7f
(10 rows)

postgres=#
```

#### * реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3
На ВМ3:
```
void@ub10replica3:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE ROLE replica_user WITH REPLICATION PASSWORD '123' LOGIN;
CREATE ROLE
```
Подкидываем в конфиг:
```
wal_level = replica
max_wal_senders = 3
hot_standby = on
```
Рестарт кластера

На ВМ4:
```
void@ub10replica4:~$ sudo pg_basebackup -h 35.223.252.159 -D /var/lib/postgresql/16/main_rep -U replica_user -P -v --wal
-method=stream
Password:
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/2000060 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_1684"
23243/23243 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/2000138
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
void@ub10replica4:~$
```

Подкидываем в конфиг:
```
primary_conninfo = 'host=35.223.252.159 port=5432 user=replica_user password=123'
```

Столкнулся с проблемой - не стартовал кластер после изменения каталога с данными на ВМ4 `data_directory = '/var/lib/postgresql/16/main_rep'
`. Оказалось, что пользователь postgres не был владельцем каталога. После того, как дал права `sudo chown -R postgres:postgres /var/lib/postgresql/16/main_rep`, всё заработало.

Итак, подключаемся к кластеру с репликой на ВМ4 и проверяем наличие данных:
```
void@ub10replica4:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                  Log file
16  main    5432 online postgres /var/lib/postgresql/16/main_rep /var/log/postgresql/postgresql-16-main.log
void@ub10replica4:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

postgres=# select * from test;
 id |    data
----+------------
  1 | 99059e9230
  2 | 251b3b39e6
  3 | e42c4fe399
  4 | e8fb43124d
  5 | db040669f9
  6 | 81cf053571
  7 | 0693c0ed1c
  8 | 89111281e5
  9 | 223a825d27
 10 | 946f4f64c1
(10 rows)

postgres=# select * from test2;
 id |    data
----+------------
  1 | 2d47e2e298
  2 | 2da94cb58b
  3 | 2ae2e27a57
  4 | fb4b6fb4dd
  5 | 9526236e00
  6 | 58bdcf3688
  7 | 3b826c456e
  8 | 444d51fa14
  9 | da6df73dc0
 10 | ba26cc9f7f
(10 rows)

postgres=#
```
