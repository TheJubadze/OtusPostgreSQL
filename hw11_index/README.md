#### Создать индекс к какой-либо из таблиц вашей БД
на ВМ в gcloud создал БД с парой таблиц:
```
CREATE DATABASE mydb;
\c mydb;

CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    title VARCHAR(200) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

далее создал индекс:
```
mydb=# CREATE INDEX idx_username ON users(username);
CREATE INDEX
```
Этот индекс ускоряет поиск пользователей по имени. полезен для систем с большим количеством пользователей, где поиск по имени может быть частым

#### Прислать текстом результат команды explain, в которой используется данный индекс
```
mydb=# EXPLAIN SELECT * FROM users WHERE username = 'testuser';
                                 QUERY PLAN
----------------------------------------------------------------------------
 Index Scan using idx_username on users  (cost=0.14..8.16 rows=1 width=348)
   Index Cond: ((username)::text = 'testuser'::text)
(2 rows)

mydb=#
```

#### Реализовать индекс для полнотекстового поиска
```
mydb=# CREATE INDEX idx_content_fulltext ON posts USING gin(to_tsvector('english', content));
CREATE INDEX
```
```
mydb=# EXPLAIN SELECT * FROM posts WHERE to_tsvector('english', content) @@ to_tsquery('test');
                                          QUERY PLAN
----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on posts  (cost=8.81..13.32 rows=1 width=466)
   Recheck Cond: (to_tsvector('english'::regconfig, content) @@ to_tsquery('test'::text))
   ->  Bitmap Index Scan on idx_content_fulltext  (cost=0.00..8.81 rows=1 width=0)
         Index Cond: (to_tsvector('english'::regconfig, content) @@ to_tsquery('test'::text))
(4 rows)

mydb=#
```
Полнотекстовый индекс позволяет эффективно искать ключевые слова в поле `content` таблицы `posts`. Это полезно для блогов или форумов, где часто требуется поиск по содержимому постов

#### Реализовать индекс на часть таблицы или индекс на поле с функцией
Создание индекса по дате создания:
```
mydb=# CREATE INDEX idx_created_at ON posts(created_at);
CREATE INDEX
```
```
mydb=# EXPLAIN SELECT * FROM posts WHERE created_at > NOW() - INTERVAL '7 days';
                       QUERY PLAN
---------------------------------------------------------
 Seq Scan on posts  (cost=0.00..12.80 rows=53 width=466)
   Filter: (created_at > (now() - '7 days'::interval))
(2 rows)

mydb=#
```
помогает ускорить запросы, фильтрующие посты по дате создания, например, за последние 7 дней

Создание индекса с использованием функции (например, нижний регистр email):
```
mydb=# CREATE INDEX idx_email_lower ON users((lower(email)));
CREATE INDEX
```
```
mydb=# EXPLAIN SELECT * FROM users WHERE lower(email) = lower('test@example.com');
                                  QUERY PLAN
-------------------------------------------------------------------------------
 Index Scan using idx_email_lower on users  (cost=0.14..8.16 rows=1 width=348)
   Index Cond: (lower((email)::text) = 'test@example.com'::text)
(2 rows)

mydb=#
```
используется для поиска по email без учета регистра

#### Создать индекс на несколько полей
```
mydb=# CREATE INDEX idx_user_created ON posts(user_id, created_at);
CREATE INDEX
```
```
mydb=# EXPLAIN SELECT * FROM posts WHERE user_id = 1 AND created_at > NOW() - INTERVAL '30 days';
                                   QUERY PLAN
--------------------------------------------------------------------------------
 Index Scan using idx_user_created on posts  (cost=0.15..8.17 rows=1 width=466)
   Index Cond: ((user_id = 1) AND (created_at > (now() - '30 days'::interval)))
(2 rows)

mydb=#
```
ускоряет запросы, которые фильтруют посты по `user_id` и `created_at`. Это полезно для выборки постов определенного пользователя за определенный период времени
