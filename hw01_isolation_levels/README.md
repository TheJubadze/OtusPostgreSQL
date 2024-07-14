#### выключить auto commit
`postgres=# \set AUTOCOMMIT off`

#### в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```
postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```

#### посмотреть текущий уровень изоляции: show transaction isolation level
```
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```

#### начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
```
postgres=# begin;
BEGIN
postgres=*#
```

#### в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
postgres=*#
```

#### сделать select from persons во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

postgres=*#
```

#### видите ли вы новую запись и если да то почему?
Новую запись не видим, т.к. открытая транзакция в первой сессии еще не закоммичена, а при данном уровне изоляции [`read committed`] мы можем видеть только закоммиченные изменения

#### завершить первую транзакцию - commit;
```
postgres=*# commit;
COMMIT
postgres=#
```

#### сделать select from persons во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=*#
```

#### видите ли вы новую запись и если да то почему?
Новую запись видим, т.к. транзакция в первой сессии закоммичена, а при данном уровне изоляции [`read committed`] мы можем видеть закоммиченные изменения

#### завершите транзакцию во второй сессии
```
postgres=*# commit;
COMMIT
postgres=#
```

#### начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
```
postgres=# set transaction isolation level repeatable read;
SET
postgres=*#
```

#### в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
postgres=*#
```

#### сделать select * from persons во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=*#
```

#### видите ли вы новую запись и если да то почему?
Новую запись не видим, т.к. при данном уровне изоляции [`repeatable read`] мы можем видеть только "снапшот" базы на начало транзакции. Любые изменения (закоммиченные или не закоммиченные), сделанные другими транзакциями, в процессе данной транзакции, нам не видны

#### завершить первую транзакцию - commit;
```
postgres=*# commit;
COMMIT
postgres=#
```

#### сделать select from persons во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=*#
```

#### видите ли вы новую запись и если да то почему?
Новую запись не видим, т.к. при данном уровне изоляции [`repeatable read`] мы можем видеть только "снапшот" базы на начало транзакции. Любые изменения (закоммиченные или не закоммиченные), сделанные другими транзакциями, в процессе данной транзакции, нам не видны

#### завершить вторую транзакцию
```
postgres=*# commit;
COMMIT
postgres=#
```

#### сделать select * from persons во второй сессии
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

postgres=*#
```

#### видите ли вы новую запись и если да то почему?
Новую запись видим, т.к. при данном уровне изоляции [`repeatable read`] мы можем видеть все изменения после коммита текущей транзакции
