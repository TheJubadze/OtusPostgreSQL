#### создайте новый кластер PostgresSQL 14
Сделал в GC

#### зайдите в созданный кластер под пользователем postgres
```
void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```

#### создайте новую базу данных testdb
```
postgres=# create database testdb;
CREATE DATABASE
postgres=#
```

#### зайдите в созданную базу данных под пользователем postgres
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```

#### создайте новую схему testnm
```
testdb=# create schema testnm;
CREATE SCHEMA
testdb=#
```

#### создайте новую таблицу t1 с одной колонкой c1 типа integer
```
testdb=# create table t1 (c1 integer);
CREATE TABLE
testdb=#
```

#### вставьте строку со значением c1=1
```
testdb=# insert into t1 values (1);
INSERT 0 1
testdb=#
```

#### создайте новую роль readonly
```
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=#
```

#### дайте новой роли право на подключение к базе данных testdb
```
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=#
```

#### дайте новой роли право на использование схемы testnm
```
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=#
```

#### дайте новой роли право на select для всех таблиц схемы testnm
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=#

#### создайте пользователя testread с паролем test123
```
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
testdb=#
```

#### дайте роль readonly пользователю testread
```
testdb=# GRANT readonly TO testread;
GRANT ROLE
testdb=#
```

#### зайдите под пользователем testread в базу данных testdb
```
void@ubnt:~$ sudo psql -U testread -d testdb
Password for user testread:
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

testdb=>
```

#### сделайте select * from t1;
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=>
```

#### получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
нет (((

#### напишите что именно произошло в тексте домашнего задания
нет доступа на селект из таблицы t1

#### у вас есть идеи почему? ведь права то дали?
```
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

Видим, что таблица в схеме public. Решил попробовать дать прав на схему public, и заработало!

```
void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT
testdb=# \q
void@ubnt:~$ sudo psql -U testread -d testdb
Password for user testread:
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

testdb=> select * from t1;
 c1
----
  1
(1 row)

testdb=>
```

#### вернитесь в базу данных testdb под пользователем postgres
```
void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```

#### удалите таблицу t1
```
testdb=# drop table t1;
DROP TABLE
testdb=#
```

#### создайте ее заново но уже с явным указанием имени схемы testnm
```
testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
testdb=#
```
#### вставьте строку со значением c1=1
```
testdb=# insert into testnm.t1 values (1);
INSERT 0 1
testdb=#
```

#### зайдите под пользователем testread в базу данных testdb
```
testdb=# \q
void@ubnt:~$ sudo psql -U testread -d testdb
Password for user testread:
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

testdb=>
```

#### сделайте select * from testnm.t1;
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
testdb=>
```

#### получилось?
доступ запрещен!

#### есть идеи почему? если нет - смотрите шпаргалку
подсмотрел в шпаргалку - дело было в том, что мы пересоздали таблицу, а права на схему давали до этого, поэтому надо раздать права заново:
```
testdb=> \q
void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#  GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=# \q
void@ubnt:~$ sudo psql -U testread -d testdb
Password for user testread:
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```
получилось! =)

#### теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
testdb=> create table testnm.t2(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 integer);
                     ^
```
у пользователя нет прав на создание таблиц

дадим права на схему testnm:
```
testdb=> \q
void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# GRANT CREATE ON SCHEMA testnm TO testread;
GRANT
```

проверяем:
```
testdb=# \q
void@ubnt:~$ sudo psql -U testread -d testdb
Password for user testread:
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

testdb=> create table testnm.t2(c1 integer);
CREATE TABLE
testdb=>
```
всё работает! =)
