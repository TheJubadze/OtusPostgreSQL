#### Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
прописал в файле настроек `postgresql.conf`:
```
log_lock_waits = on
deadlock_timeout = 200ms
```

создал таблицу с тестовыми данными:
```
CREATE TABLE test_lock (
    id SERIAL PRIMARY KEY,
    data TEXT
);

INSERT INTO test_lock (data) VALUES ('Row 1'), ('Row 2'), ('Row 3');
```

открыл два сеанса подключения к БД
в первом сеансе:
```
postgres=# BEGIN;
BEGIN
postgres=*# SELECT * FROM test_lock WHERE id = 1 FOR UPDATE;
 id | data
----+-------
  1 | Row 1
(1 row)

postgres=*#
```

во втором сеансе:
```
postgres=# BEGIN;
BEGIN
postgres=*# SELECT * FROM test_lock WHERE id = 1 FOR UPDATE;
```

смотрим журнал:
```
2024-07-21 04:17:25.119 UTC [1325] postgres@postgres LOG:  process 1325 still waiting for ShareLock on transaction 738 after 200.228 ms
2024-07-21 04:17:25.119 UTC [1325] postgres@postgres DETAIL:  Process holding the lock: 1195. Wait queue: 1325.
2024-07-21 04:17:25.119 UTC [1325] postgres@postgres CONTEXT:  while locking tuple (0,1) in relation "test_lock"
2024-07-21 04:17:25.119 UTC [1325] postgres@postgres STATEMENT:  SELECT * FROM test_lock WHERE id = 1 FOR UPDATE;
```

#### Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
в трёх сеансах сделал
```
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_lock SET data = 'Upd' WHERE id = 1;
```
в первом сеансе вошел в режим редактирования транзации, а два других сеанса - заблокировались

смотрим таблицу блокировок:
```
postgres=*# SELECT pid, locktype, mode, granted, relation::regclass, transactionid, virtualxid, virtualtransaction FROM pg_locks;
 pid  |   locktype    |       mode       | granted |    relation    | transactionid | virtualxid | virtualtransaction
------+---------------+------------------+---------+----------------+---------------+------------+-------------------
 1430 | relation      | RowExclusiveLock | t       | test_lock_pkey |               |            | 5/13              
 1430 | relation      | RowExclusiveLock | t       | test_lock      |               |            | 5/13              
 1430 | virtualxid    | ExclusiveLock    | t       |                |               | 5/13       | 5/13              
 1325 | relation      | RowExclusiveLock | t       | test_lock_pkey |               |            | 4/50              
 1325 | relation      | RowExclusiveLock | t       | test_lock      |               |            | 4/50              
 1325 | virtualxid    | ExclusiveLock    | t       |                |               | 4/50       | 4/50              
 1195 | relation      | AccessShareLock  | t       | pg_locks       |               |            | 3/17              
 1195 | relation      | RowExclusiveLock | t       | test_lock_pkey |               |            | 3/17              
 1195 | relation      | RowExclusiveLock | t       | test_lock      |               |            | 3/17              
 1195 | virtualxid    | ExclusiveLock    | t       |                |               | 3/17       | 3/17              
 1430 | transactionid | ExclusiveLock    | t       |                |           742 |            | 5/13              
 1325 | tuple         | ExclusiveLock    | t       | test_lock      |               |            | 4/50              
 1325 | transactionid | ExclusiveLock    | t       |                |           741 |            | 4/50              
 1195 | transactionid | ExclusiveLock    | t       |                |           740 |            | 3/17              
 1430 | tuple         | ExclusiveLock    | f       | test_lock      |               |            | 5/13              
 1325 | transactionid | ShareLock        | f       |                |           740 |            | 4/50              
(16 rows)

postgres=*#
```
`RowExclusiveLock` означает блокировки, которые применяются к строкам при их изменении. `UPDATE` и `DELETE` команды применяют `RowExclusiveLock` к строкам, которые они изменяют.
`ExclusiveLock` - эти блокировки предотвращают доступ других транзакций к заблокированному объекту. Когда сеанс 1 выполняет `UPDATE`, он удерживает `ExclusiveLock` на строке, которую обновляет.
`ShareLock` - эти блокировки позволяют другим транзакциям считывать данные, но не модифицировать их. При выполнении команды `SELECT FOR UPDATE` или `SELECT FOR SHARE` транзакция удерживает `ShareLock`.

#### Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
Чтобы создать deadlock, в каждом сеансе начнем транзакцию с обновлением последующей строки по кругу, т.е. первая транзакция будет обновлять строку 1, вторая - 2, третья - 3. Затем в каждом из сеансов в текущей транзакции начнем обновлять строки со смещением айдишек на +1, т.е. в первом сеансе будем обновлять строку 2, которая уже заблокирована сеансом 2, во втором сеансе будем изменять строку 3, а в третьем - 1, соответственно. Таким образом получим состояние взаимоблокировки трёх транзакций.

Изучим журнал:
```
2024-07-21 04:41:22.145 UTC [1195] postgres@postgres LOG:  process 1195 still waiting for ShareLock on transaction 745 after 200.306 ms
2024-07-21 04:41:22.145 UTC [1195] postgres@postgres DETAIL:  Process holding the lock: 1325. Wait queue: 1195.
2024-07-21 04:41:22.145 UTC [1195] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "test_lock"
2024-07-21 04:41:22.145 UTC [1195] postgres@postgres STATEMENT:  UPDATE test_lock SET data = 'Upd1 again' WHERE id = 2;
2024-07-21 04:41:30.164 UTC [1325] postgres@postgres LOG:  process 1325 still waiting for ShareLock on transaction 746 after 200.223 ms
2024-07-21 04:41:30.164 UTC [1325] postgres@postgres DETAIL:  Process holding the lock: 1430. Wait queue: 1325.
2024-07-21 04:41:30.164 UTC [1325] postgres@postgres CONTEXT:  while updating tuple (0,6) in relation "test_lock"
2024-07-21 04:41:30.164 UTC [1325] postgres@postgres STATEMENT:  UPDATE test_lock SET data = 'Upd2 again' WHERE id = 3;
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres LOG:  process 1430 detected deadlock while waiting for ShareLock on transaction 744 after 200.136 ms
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres DETAIL:  Process holding the lock: 1195. Wait queue: .
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test_lock"
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres STATEMENT:  UPDATE test_lock SET data = 'Upd3 again' WHERE id = 1;
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres ERROR:  deadlock detected
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres DETAIL:  Process 1430 waits for ShareLock on transaction 744; blocked by process 1195.
        Process 1195 waits for ShareLock on transaction 745; blocked by process 1325.
        Process 1325 waits for ShareLock on transaction 746; blocked by process 1430.
        Process 1430: UPDATE test_lock SET data = 'Upd3 again' WHERE id = 1;
        Process 1195: UPDATE test_lock SET data = 'Upd1 again' WHERE id = 2;
        Process 1325: UPDATE test_lock SET data = 'Upd2 again' WHERE id = 3;
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres HINT:  See server log for query details.
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test_lock"
2024-07-21 04:41:36.853 UTC [1430] postgres@postgres STATEMENT:  UPDATE test_lock SET data = 'Upd3 again' WHERE id = 1;
2024-07-21 04:41:36.854 UTC [1325] postgres@postgres LOG:  process 1325 acquired ShareLock on transaction 746 after 6889.447 ms
2024-07-21 04:41:36.854 UTC [1325] postgres@postgres CONTEXT:  while updating tuple (0,6) in relation "test_lock"
2024-07-21 04:41:36.854 UTC [1325] postgres@postgres STATEMENT:  UPDATE test_lock SET data = 'Upd2 again' WHERE id = 3;
```
`Process 1430 waits for ShareLock on transaction 744; blocked by process 1195` говорит нам о том, что процесс 1430 ожидает `ShareLock` блокировку на транзакцию 744, заблокированную процессом 1195
и т.д.

Анализируя эти сообщения, можно понять, какие процессы участвовали во взаимоблокировке, какие строки и таблицы были задействованы, и какие запросы вызвали взаимоблокировку. Это дает возможность выяснить причину проблемы постфактум

#### Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
Теоретически, могут, если они выполнятся одновременно или `UPDATE` будет выполняться достаточно долго (например, при большом объёме таблицы). Попробуем воспроизвести, создав большую таблицу

#### Задание со звездочкой* Попробуйте воспроизвести такую ситуацию.
Создал таблицу на 30 миллионов записей и в двух сеансах одновременно начал транзакцию с `UPDATE`:
```
postgres=# begin;
BEGIN
postgres=*# UPDATE big_table SET data = 'Upd1';
```
```
postgres=# begin;
BEGIN
postgres=*# UPDATE big_table SET data = 'Upd2';
```

Однако, к моему удивлению, взамоблокировки не произошло - через довольно продолжительное время первая транзакция успешно завершилась:
```
...
UPDATE 30000000
postgres=*# select * from big_table limit 10;
 id | data
----+------
  1 | Upd1
  2 | Upd1
  3 | Upd1
  4 | Upd1
  5 | Upd1
  6 | Upd1
  7 | Upd1
  8 | Upd1
  9 | Upd1
 10 | Upd1
(10 rows)

postgres=*#
```

Но вторая - продолжает висеть. Метрики инстанса ВМ показывают нулевую активность, т.е. вроде как постгря ничего не делает, а транзакция просто висит. При этом в журнале нет никаких записей о блокировках...

После коммита первой транзакции пошла активность на виртуалке, т.е. постгря вроде как начала маслать `UPDATE` во втором сеансе, и опять-таки через минут 15 закончилась:
```
UPDATE 30000000
postgres=*#
postgres=*# select * from big_table limit 10;
   id   | data
--------+------
 109521 | Upd2
 109522 | Upd2
 109523 | Upd2
 109524 | Upd2
 109525 | Upd2
 109526 | Upd2
 109527 | Upd2
 109528 | Upd2
 109529 | Upd2
 109530 | Upd2
(10 rows)

postgres=*#
```

Итого, резюмируя, могу сказать, что воспроизвести взаимоблокировку 2-х транзакций выполняющих единственную команду `UPDATE` одной и той же таблицы (без `where`) мне не удалось
