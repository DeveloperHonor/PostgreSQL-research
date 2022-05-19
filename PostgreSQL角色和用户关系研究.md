# **PostgreSQL 角色和用户关系**

#### 1.创建只读和读写角色

```
postgres=# CREATE ROLE role_ro;
CREATE ROLE
postgres=# CREATE ROLE role_rw;
CREATE ROLE
```

#### 2.创建两个用户

要求：两个用户一个只具有只读权限，一个具有读写权限

```
postgres=# CREATE USER user_ro WITH PASSWORD 'user_ro';
CREATE ROLE
postgres=# CREATE USER user_rw WITH PASSWORD 'user_rw';
CREATE ROLE
```

#### 3.创建一个数据库(或者使用默认库也可以)

```
postgres=# CREATE DATABASE db1;
CREATE DATABASE
```

#### 4.登录数据库 db1 创建表

这里使用 postgres 超级用户创建表

```
db1=# CREATE TABLE tab_test(id integer,name varchar);
CREATE TABLE
db1=# INSERT INTO tab_test VALUES(1,'PostgreSQL');
INSERT 0 1
```

#### 5.首先验证 user_ro 和 user_rw 对于 tab_test读写

```
--验证 user_ro 读取表
db1=# \c db1 user_ro
You are now connected to database "db1" as user "user_ro".
db1=> SELECT * FROM tab_test;
ERROR:  permission denied for table tab_test
--验证 user_rW 读取表
db1=> \c db1 user_rw
You are now connected to database "db1" as user "user_rw".
db1=> SELECT * FROM tab_test;
ERROR:  permission denied for table tab_test

两张表都无法读取数据表 tab_test
```



#### 6.回收 role_ro 的权限，为只读

```
db1=# REVOKE DELETE ON ALL TABLES IN SCHEMA public FROM role_ro;
REVOKE
db1=# REVOKE UPDATE ON ALL TABLES IN SCHEMA public FROM role_ro;
REVOKE
db1=# REVOKE INSERT ON ALL TABLES IN SCHEMA public FROM role_ro;
REVOKE
db1=# REVOKE TRUNCATE ON ALL TABLES IN SCHEMA public FROM role_ro;
REVOKE
db1=# REVOKE REFERENCES ON ALL TABLES IN SCHEMA public FROM role_ro;
REVOKE
db1=# REVOKE TRIGGER ON ALL TABLES IN SCHEMA public FROM role_ro;
REVOKE`

--授权只读
db1=# GRANT SELECT ON ALL TABLES IN SCHEMA public TO role_ro;
GRANT
```


#### 7.将 role_ro 角色分配给 user_ro

由于角色只能在创建用户的时候被指定，因此，删除之前的用户重新创建

```
db1=# DROP USER user_ro;
DROP ROLE
db1=# CREATE USER user_ro WITH PASSWORD 'user_ro' IN ROLE role_ro;
CREATE ROLE
```



#### 8.验证 user_ro 能否读取 tab_test 表

```
db1=# \c db1 user_ro ;
You are now connected to database "db1" as user "user_ro".
db1=> SELECT * FROM tab_test;
id |    name
----+------------
1 | PostgreSQL
(1 row)

--可以读取
```


#### 9.验证 user_ro　能否执行DML操作

```
db1=> INSERT INTO tab_test VALUES(2,'INSERT ITEM BY user_ro');
ERROR:  permission denied for table tab_test
db1=> UPDATE tab_test SET name = 'INSERT ITEM BY user_ro' WHERE id = 1;
ERROR:  permission denied for table tab_test
db1=> DELETE FROM tab_test WHERE id = 1;
ERROR:  permission denied for table tab_test
```

#### 10.授权 role_rw 角色具有读写权限

```
db1=# GRANT SELECT,INSERT,UPDATE,DELETE,REFERENCES,TRIGGER,TRUNCATE ON ALL TABLES IN SCHEMA public to role_rw;
GRANT
```

#### 11.分配 role_rw 角色给用户user_rw

```
db1=# DROP USER IF EXISTS user_rw ;
DROP ROLE
db1=# CREATE USER user_rw WITH IN ROLE role_rw PASSWORD 'user_rw';
CREATE ROLE
```

#### 12.验证 user_rw 是否能够读取和更改数据

```
db1=> \c db1 user_rw
You are now connected to database "db1" as user "user_rw".
db1=> SELECT * FROM tab_test;
id |    name
----+------------
1 | PostgreSQL
(1 row)

db1=> INSERT INTO tab_test VALUES(2,'INSERT ITEM BY user_rw');
INSERT 0 1
db1=> UPDATE tab_test SET name = 'UPDATE ITEM BY user_rw' WHERE id = 1;
UPDATE 1
db1=> DELETE FROM tab_test WHERE id = 1;
DELETE 1
db1=> SELECT * FROM tab_test;
id |          name
----+------------------------
2 | INSERT ITEM BY user_rw
(1 row)
```

