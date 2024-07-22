#### Создаем БД, схему и в ней таблицу.
```
void@ub09backup:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE DATABASE mydatabase;
CREATE DATABASE
postgres=# \c mydatabase
You are now connected to database "mydatabase" as user "postgres".
mydatabase=# CREATE SCHEMA myschema;
CREATE SCHEMA
mydatabase=#
mydatabase=# CREATE TABLE myschema.mytable (
mydatabase(#     id SERIAL PRIMARY KEY,
mydatabase(#     name VARCHAR(50),
mydatabase(#     value INTEGER
mydatabase(# );
CREATE TABLE
```

#### Заполним таблицы автосгенерированными 100 записями.
```
mydatabase=# INSERT INTO myschema.mytable (name, value)
mydatabase-# SELECT
mydatabase-#     md5(random()::text) AS name,
mydatabase-#     (random() * 100)::int AS value
mydatabase-# FROM generate_series(1, 100);
INSERT 0 100
```
```
mydatabase=# select * from myschema.mytable;

 id  |               name               | value
-----+----------------------------------+-------
   1 | 4e2be9712395fcdfd3235829c7374f2f |    26
   2 | 22abdc417a0ec7e3a32a7ed3c15e8349 |    85
   3 | bfb451c9d32b76ca3a43a73da04cfff9 |    43
   4 | b2e136fed561ceea1f319f377b02b0a8 |    44
   5 | 1ba1d6208d168eff42cd8294ff2ae87f |    14
   6 | e3222448fd29764b59cb1bc3fc512b14 |    63
   7 | 8b05580898b7d78ef1e5dd87025af817 |     4
   8 | 0ee04f3275bf173e20444979c79df19a |    42
   9 | 464e04d581efa56e4a13aba3dce0bd62 |    38
  10 | 7de8add51260c8f6fbce887fb72b6ab3 |    58
  ...
```

#### Под линукс пользователем Postgres создадим каталог для бэкапов
```
void@ub09backup:~$ sudo -u postgres mkdir /var/lib/postgresql/backups
res chmod 700 /var/lib/postgresql/backups
void@ub09backup:~$ sudo -u postgres chmod 700 /var/lib/postgresql/backups
```

#### Сделаем логический бэкап используя утилиту COPY
```
void@ub09backup:~$ sudo -u postgres psql -d mydatabase -c "COPY myschema.mytable TO '/var/lib/postgresql/backups/mytable_backup.csv' DELIMITER ',' CSV HEADER;"
COPY 100
```

#### Восстановим в 2 таблицу данные из бэкапа.
```
mydatabase=# CREATE TABLE myschema.mytable_copy (
mydatabase(#     id SERIAL PRIMARY KEY,
mydatabase(#     name VARCHAR(50),
mydatabase(#     value INTEGER
mydatabase(# );
CREATE TABLE
```

```
void@ub09backup:~$ sudo -u postgres psql -d mydatabase -c "COPY myschema.mytable_copy FROM '/var/lib/postgresql/backups/mytable_backup.csv' DELIMITER ',' CSV HEADER;"
COPY 100
```

```
mydatabase=# select * from myschema.mytable_copy;

 id  |               name               | value
-----+----------------------------------+-------
   1 | 4e2be9712395fcdfd3235829c7374f2f |    26
   2 | 22abdc417a0ec7e3a32a7ed3c15e8349 |    85
   3 | bfb451c9d32b76ca3a43a73da04cfff9 |    43
   4 | b2e136fed561ceea1f319f377b02b0a8 |    44
   5 | 1ba1d6208d168eff42cd8294ff2ae87f |    14
   6 | e3222448fd29764b59cb1bc3fc512b14 |    63
   7 | 8b05580898b7d78ef1e5dd87025af817 |     4
   8 | 0ee04f3275bf173e20444979c79df19a |    42
   9 | 464e04d581efa56e4a13aba3dce0bd62 |    38
  10 | 7de8add51260c8f6fbce887fb72b6ab3 |    58
  ...
```

#### Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
```
void@ub09backup:~$ sudo -u postgres pg_dump -d mydatabase -n myschema -t myschema.mytable -t myschema.mytable_copy -Fc -f /var/lib/postgresql/backups/mydatabase_backup.dump
```

#### Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
```
mydatabase=# CREATE DATABASE mydatabase_restore;
CREATE DATABASE
mydatabase=# \c mydatabase_restore
You are now connected to database "mydatabase_restore" as user "postgres".
mydatabase_restore=# create schema myschema;
CREATE SCHEMA
```

```
void@ub09backup:~$ sudo -u postgres pg_restore -d mydatabase_restore -n myschema /var/lib/postgresql/backups/mydatabase_backup.dump
void@ub09backup:~$
```

```
mydatabase_restore=# select * from myschema.mytable;

 id  |               name               | value
-----+----------------------------------+-------
   1 | 4e2be9712395fcdfd3235829c7374f2f |    26
   2 | 22abdc417a0ec7e3a32a7ed3c15e8349 |    85
   3 | bfb451c9d32b76ca3a43a73da04cfff9 |    43
   4 | b2e136fed561ceea1f319f377b02b0a8 |    44
   5 | 1ba1d6208d168eff42cd8294ff2ae87f |    14
   6 | e3222448fd29764b59cb1bc3fc512b14 |    63
   7 | 8b05580898b7d78ef1e5dd87025af817 |     4
   8 | 0ee04f3275bf173e20444979c79df19a |    42
   9 | 464e04d581efa56e4a13aba3dce0bd62 |    38
  10 | 7de8add51260c8f6fbce887fb72b6ab3 |    58
  ...

mydatabase_restore=# select * from myschema.mytable_copy;

 id  |               name               | value
-----+----------------------------------+-------
   1 | 4e2be9712395fcdfd3235829c7374f2f |    26
   2 | 22abdc417a0ec7e3a32a7ed3c15e8349 |    85
   3 | bfb451c9d32b76ca3a43a73da04cfff9 |    43
   4 | b2e136fed561ceea1f319f377b02b0a8 |    44
   5 | 1ba1d6208d168eff42cd8294ff2ae87f |    14
   6 | e3222448fd29764b59cb1bc3fc512b14 |    63
   7 | 8b05580898b7d78ef1e5dd87025af817 |     4
   8 | 0ee04f3275bf173e20444979c79df19a |    42
   9 | 464e04d581efa56e4a13aba3dce0bd62 |    38
  10 | 7de8add51260c8f6fbce887fb72b6ab3 |    58
  ...
```