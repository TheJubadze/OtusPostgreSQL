#### Настройте выполнение контрольной точки раз в 30 секунд.
```
void@ubmvcc:~$ vim /etc/postgresql/16/main/postgresql.conf
```
```
checkpoint_timeout = 30s
```

#### 10 минут c помощью утилиты pgbench подавайте нагрузку.
```
void@ubmvcc:~$ sudo du -sh /var/lib/postgresql/16/main/pg_wal
20K     /var/lib/postgresql/16/main/pg_wal
```
```
void@ubmvcc:~$ sudo -u postgres pgbench -c 10 -j 2 -T 600 testdb
Password:
pgbench (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
starting vacuum...end.
```
...
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 407816
number of failed transactions: 0 (0.000%)
latency average = 14.713 ms
initial connection time = 76.107 ms
tps = 679.658807 (without initial connection time)
void@ubmvcc:~$
```

#### Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
```
void@ubmvcc:~$ sudo du -sh /var/lib/postgresql/16/main/pg_wal
81M     /var/lib/postgresql/16/main/pg_wal
void@ubmvcc:~$
```
Посмотрим, сколько всего контрольных точек было сгенерировано:
```
SELECT 
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time
FROM 
    pg_stat_bgwriter;

checkpoints_timed|checkpoints_req|checkpoint_write_time|checkpoint_sync_time|
-----------------+---------------+---------------------+--------------------+
              106|             11|            3174201.0|              1605.0|
```
Итого:      106 + 11 = 117 контрольных точек
Получаем:   81М / 117 ~ 709K на точку в среднем

#### Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
Поскольку checkpoints_req = 11, это означает, что 11 точек были выполнены незапланированно, т.е. эти точки не были выполнены по расписанию, а были выполнены по требованию системы. Это произошло из-за того, что допустимый объём журнала WAL был превышен в эти моменты.

#### Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
```
synchronous_commit = on

void@ubmvcc:~$ sudo -u postgres pgbench -c 10 -j 2 -T 600 testdb
Password:
pgbench (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 414522
number of failed transactions: 0 (0.000%)
latency average = 14.474 ms
initial connection time = 76.041 ms
tps = 690.904949 (without initial connection time)
void@ubmvcc:~$
```

```
synchronous_commit = off

void@ubmvcc:~$ sudo -u postgres pgbench -c 10 -j 2 -T 600 testdb
Password:
pgbench (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 703696
number of failed transactions: 0 (0.000%)
latency average = 8.526 ms
initial connection time = 80.228 ms
tps = 1172.859208 (without initial connection time)
void@ubmvcc:~$
```

Значение tps выше в асинхронном режиме потому, что в этом режиме PostgreSQL подтверждает транзакцию, не ожидая завершения записи данных WAL на диск. Исходя из этого, можно сделать вывод, что синхронный режим более надёжный, но медленный, а асихронный - наоборот: быстрый, но менее надёжный.


#### Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
Создадим новый кластер в новом каталоге:
```
void@ubmvcc:~$ sudo systemctl stop postgresql
void@ubmvcc:~$ sudo mkdir -p /var/lib/postgresql/new_cluster
void@ubmvcc:~$ sudo chown -R postgres:postgres /var/lib/postgresql/new_cluster
void@ubmvcc:~$ sudo -u postgres /usr/lib/postgresql/16/bin/initdb --data-checksums -D /var/lib/postgresql/new_cluster
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/new_cluster ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/postgresql/new_cluster -l logfile start
```

Стартанём его:
```
void@ubmvcc:~$ sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/postgresql/new_cluster start
waiting for server to start....2024-07-19 11:27:34.780 UTC [3625] LOG:  starting PostgreSQL 16.3 (Ubuntu 16.3-1.pgdg20.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit
2024-07-19 11:27:34.780 UTC [3625] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2024-07-19 11:27:34.782 UTC [3625] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2024-07-19 11:27:34.789 UTC [3628] LOG:  database system was shut down at 2024-07-19 11:26:46 UTC
2024-07-19 11:27:34.797 UTC [3625] LOG:  database system is ready to accept connections
 done
server started
```

Создадим таблицу:
```
void@ubmvcc:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# create table t1 (i int);
CREATE TABLE
postgres=# insert into t1 values (1), (2), (3);
INSERT 0 3
postgres=# select * from t1;
 i
---
 1
 2
 3
(3 rows)
```

Изменил пару байт в файле таблицы при помощи hex-редактора

```
void@ubmvcc:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from t1;
2024-07-19 12:02:08.901 UTC [5143] WARNING:  page verification failed, calculated checksum 25251 but expected 22489
2024-07-19 12:02:08.901 UTC [5143] ERROR:  invalid page in block 0 of relation base/5/16389
2024-07-19 12:02:08.901 UTC [5143] STATEMENT:  select * from t1;
WARNING:  page verification failed, calculated checksum 25251 but expected 22489
ERROR:  invalid page in block 0 of relation base/5/16389
```

При попытке выборки данных получаем ошибку контрольной суммы.
Чтобы игнорировать эту ошибку, необходимо прописать параметр `ignore_checksum_failure = on` в файле настроек `postresql.conf` и перезапустить кластер

```
void@ubmvcc:~$ sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/postgresql/new_cluster restart
waiting for server to shut down...2024-07-19 12:06:06.529 UTC [5133] LOG:  received fast shutdown request
.2024-07-19 12:06:06.531 UTC [5133] LOG:  aborting any active transactions
2024-07-19 12:06:06.536 UTC [5133] LOG:  background worker "logical replication launcher" (PID 5139) exited with exit code 1
2024-07-19 12:06:06.536 UTC [5134] LOG:  shutting down
2024-07-19 12:06:06.539 UTC [5134] LOG:  checkpoint starting: shutdown immediate
2024-07-19 12:06:06.549 UTC [5134] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.004 s, sync=0.002 s, total=0.013 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB; lsn=0/1BB4440, redo lsn=0/1BB4440
2024-07-19 12:06:06.554 UTC [5133] LOG:  database system is shut down
 done
server stopped
waiting for server to start....2024-07-19 12:06:06.661 UTC [5188] LOG:  starting PostgreSQL 16.3 (Ubuntu 16.3-1.pgdg20.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit
2024-07-19 12:06:06.661 UTC [5188] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2024-07-19 12:06:06.664 UTC [5188] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2024-07-19 12:06:06.673 UTC [5191] LOG:  database system was shut down at 2024-07-19 12:06:06 UTC
2024-07-19 12:06:06.681 UTC [5188] LOG:  database system is ready to accept connections
 done
server started
void@ubmvcc:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from t1;
2024-07-19 12:06:10.394 UTC [5197] WARNING:  page verification failed, calculated checksum 25251 but expected 22489
WARNING:  page verification failed, calculated checksum 25251 but expected 22489
 i
---
 1
 2
 3
(3 rows)

postgres=#
```

Теперь данные выбираются, но включение этого параметра может привести к непредсказуемому поведению базы данных, поэтому использовать его стоит только в крайнем случае и временно, чтобы не потерять важные данные
