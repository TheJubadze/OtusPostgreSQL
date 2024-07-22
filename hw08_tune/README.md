#### развернуть виртуальную машину любым удобным способом, поставить на неё PostgreSQL 15 любым способом
Развернул ВМ в гугл клауде 2 CPU (1 core, 2 Threads per CPU), 4GB RAM

#### настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
настройки кластера по умолчанию после его установки:
```
void@ub08tune:~$ sudo -u postgres psql
Password for user postgres:
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# SELECT name, setting, unit
postgres-# FROM pg_settings
postgres-# WHERE name IN (
postgres(#     'shared_buffers',
postgres(#     'work_mem',
postgres(#     'maintenance_work_mem',
postgres(#     'effective_cache_size',
postgres(#     'wal_level',
postgres(#     'synchronous_commit',
postgres(#     'wal_writer_delay',
postgres(#     'commit_delay',
postgres(#     'checkpoint_timeout',
postgres(#     'max_wal_size'
postgres(# );
             name             | setting | unit
------------------------------+---------+------
 checkpoint_timeout           | 300     | s
 commit_delay                 | 0       |
 effective_cache_size         | 524288  | 8kB
 maintenance_work_mem         | 65536   | kB
 max_wal_size                 | 1024    | MB
 shared_buffers               | 16384   | 8kB
 synchronous_commit           | on      |
 wal_level                    | replica |
 wal_writer_delay             | 200     | ms
 work_mem                     | 4096    | kB
(11 rows)
```

применил следующие настройки конфигурации:
```
shared_buffers = 1GB               # Поставил оптимальные 25% от RAM, как советовали в лекции
work_mem = 32MB                    # Сделал 1/32 от shared_buffers - по советам в интернете
maintenance_work_mem = 512MB       # Увеличил для ускорения операций VACUUM и CREATE INDEX
effective_cache_size = 3GB         # Поставил приблизительно 75% от общей оперативной памяти - по советам из интернета

# Настройки журнала
max_wal_senders = 0                # Эту хреновину пришлось проставить, т.к. иначе не стартовал кластер из-за настройки wal_level = minimal - требовал чтобы был replica либо logical
wal_level = minimal                # Минимальный уровень журнала для максимальной производительности
synchronous_commit = off           # Отключение синхронной фиксации
wal_writer_delay = 10ms            # Уменьшение задержки записи в журнал
commit_delay = 10000               # Установка задержки фиксации для накопления транзакций
checkpoint_timeout = 30min         # Увеличение времени между контрольными точками
max_wal_size = 4GB                 # Увеличение максимального размера WAL
```

#### нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
Результаты теста при параметрах по умолчанию:
```
void@ub08tune:~$ sudo pgbench -c 10 -j 2 -T 60 -P 5 -U postgres pgb_test
Password:
pgbench (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 5.0 s, 929.0 tps, lat 10.678 ms stddev 5.757, 0 failed
progress: 10.0 s, 966.8 tps, lat 10.340 ms stddev 5.423, 0 failed
progress: 15.0 s, 955.4 tps, lat 10.470 ms stddev 5.580, 0 failed
progress: 20.0 s, 948.2 tps, lat 10.542 ms stddev 5.706, 0 failed
progress: 25.0 s, 911.0 tps, lat 10.973 ms stddev 10.005, 0 failed
progress: 30.0 s, 993.0 tps, lat 10.072 ms stddev 5.465, 0 failed
progress: 35.0 s, 995.4 tps, lat 10.049 ms stddev 5.457, 0 failed
progress: 40.0 s, 981.0 tps, lat 10.187 ms stddev 5.450, 0 failed
progress: 45.0 s, 957.0 tps, lat 10.451 ms stddev 5.707, 0 failed
progress: 50.0 s, 1006.2 tps, lat 9.937 ms stddev 5.380, 0 failed
progress: 55.0 s, 961.4 tps, lat 10.401 ms stddev 6.141, 0 failed
progress: 60.0 s, 966.4 tps, lat 10.345 ms stddev 9.576, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 57864
number of failed transactions: 0 (0.000%)
latency average = 10.365 ms
latency stddev = 6.479 ms
initial connection time = 32.827 ms
tps = 964.464394 (without initial connection time)
```

Результаты теста при измененных настройках:
```
void@ub08tune:~$ sudo pgbench -c 10 -j 2 -T 60 -P 5 -U postgres pgb_test
Password:
pgbench (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 5.0 s, 2351.8 tps, lat 4.210 ms stddev 1.604, 0 failed
progress: 10.0 s, 2462.4 tps, lat 4.061 ms stddev 1.464, 0 failed
progress: 15.0 s, 2496.2 tps, lat 4.006 ms stddev 1.468, 0 failed
progress: 20.0 s, 2435.8 tps, lat 4.104 ms stddev 1.957, 0 failed
progress: 25.0 s, 2530.0 tps, lat 3.952 ms stddev 1.453, 0 failed
progress: 30.0 s, 2496.0 tps, lat 4.005 ms stddev 1.494, 0 failed
progress: 35.0 s, 2464.4 tps, lat 4.057 ms stddev 1.469, 0 failed
progress: 40.0 s, 2512.8 tps, lat 3.979 ms stddev 1.461, 0 failed
progress: 45.0 s, 2514.8 tps, lat 3.976 ms stddev 1.620, 0 failed
progress: 50.0 s, 2489.0 tps, lat 4.016 ms stddev 1.507, 0 failed
progress: 55.0 s, 2485.2 tps, lat 4.023 ms stddev 1.417, 0 failed
progress: 60.0 s, 2454.2 tps, lat 4.074 ms stddev 1.628, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 148473
number of failed transactions: 0 (0.000%)
latency average = 4.038 ms
latency stddev = 1.555 ms
initial connection time = 45.615 ms
tps = 2475.422174 (without initial connection time)
```

Итого, `tps` вырос со значения 964 до 2475! =)

#### Задание со *: аналогично протестировать через утилиту https://github.com/Percona-Lab/sysbench-tpcc
Установил всё необходимое по инструкции в интернете (шаги установки сюда не копирую, т.к. там всё просто)

Инициализирую БД:
```
sysbench --db-driver=pgsql --pgsql-db=tpcc_test --pgsql-user=postgres --pgsql-password=123 --threads=2 --tables=10 --scale=50 ./tpcc.lua --tables=10 --scale=50 prepare
```
Эта команда по сути заняла у меня весь день. После часа-двух работы закончилось место на диске - пришлось останавливать виртуалку и добавлять гигов на диск. Презапустил. Потом, спустя пару часов работы этой команды, когда я понял, что на каждую таблицу он собирается производить по 50 длительных операций, и я прикинул, что ближайшие пару дней оно не закончится, я подкрутил `--tables=5 --scale=10`, но всё равно это было слишком долго, и, прервав процесс в очередной раз, я поправил эти параметры на `--tables=5 --scale=1`. После того, как команда подготовки к тесту в итоге отработала, запускаю собственно сам тест:
```
sysbench --db-driver=pgsql --pgsql-db=tpcc_test --pgsql-user=postgres --pgsql-password=123 --threads=2 --time=300 --report-interval=10 ./tpcc.lua run
```
... и получаю такое вот сообщение:
```
void@ub08tune:~/sysbench-tpcc$ sysbench --db-driver=pgsql --pgsql-db=tpcc_test --pgsql-user=postgres --pgsql-password=123 --threads=2 --time=300 --report-interval=10 ./tpcc.lua run
sysbench 1.1.0-2ca9e3f (using bundled LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 2
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

DB SCHEMA public
DB SCHEMA public
Threads started!

FATAL: `thread_run' function failed: ./tpcc_run.lua:363: attempt to perform arithmetic on a nil value
void@ub08tune:~/sysbench-tpcc$
```
После попыток гугления, пересборки и переустановки `sysbench`, пересоздания БД, перезапуска всего вышеперечисленного по новому, всё равно получаю эту ошибку. В общем, на эту утилиту день потрачен почём зря - результатов нет, обидно ((

Может быть у Вас есть идеи, что с ней не так и почему сам тест не запускается?
