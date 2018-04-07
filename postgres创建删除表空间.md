# [postgresql 表空间创建、删除]

表空间：字面上理解就是表存储的物理空间，其实包括数据库的表、索引、序列等。

可以将表空间创建在服务器的不同分区，这样做的好处有：

一、如果初始化集群所在分区已经用光，可以方便的其他分区上创建表空间已达到扩容的目的。

二、对于频繁访问的数据可以存储在性能较高、较快的磁盘分区上，而不常用的数据存储在便宜的较慢的磁盘分区上。

 

语法：

postgres=# \h create tablespace 
Command:     CREATE TABLESPACE
Description: define a new tablespace
Syntax:
CREATE TABLESPACE tablespace_name
    [ OWNER user_name ]
    LOCATION 'directory'
    [ WITH ( tablespace_option = value [, ... ] ) ]

用户必须有表空间所在目录访问权限，所以在创建表空间之前需要在对应分区下创建相应的目录，并为其分配权限。

[root@localhost ~]# mkdir /usr/local/pgdata
[root@localhost ~]# chown postgres:postgres /usr/local/pgdata/

 创建表空间示例：

```shell
postgres=# create tablespace tbs_test owner postgres location '/usr/local/pgdata';
CREATE TABLESPACE 
```

创建表空间成功后，可在数据库集群目录下看到一个新增的目录pg_tblspc下有一个连接文件51276,指向到/usr/local/pgdata下

```shell
[root@localhost ~]# ll /mnt/syncdata/pgsql941/data/pg_tblspc/
total 0
lrwxrwxrwx. 1 postgres postgres 17 Aug 30 02:06 51276 -> /usr/local/pgdata
```

```shell
[root@localhost ~]# ll /usr/local/pgdata/
total 4
drwx------. 2 postgres postgres 4096 Aug 30 02:06 PG_9.4_201409291
```

在此表空间内创建表：

```shell
postgres=# create table test(a int) tablespace tbs_test;
CREATE TABLE
```

现在在表空间目录下就会新增一个test表对应的文件：

[root@localhost ~]# ll /usr/local/pgdata/PG_9.4_201409291/13003/51277 
-rw-------. 1 postgres postgres 0 Aug 30 02:15 /usr/local/pgdata/PG_9.4_201409291/13003/51277

 

其中51277对应的是test表的relfilenode，13003是数据库postgres的oid。

```sql
postgres=# select oid,datname from pg_database where datname = 'postgres';
  oid  | datname  
-------+----------
 13003 | postgres
(1 row)

postgres=# select relname,relfilenode from pg_class where relname='test';
 relname | relfilenode 
---------+-------------
 test    |       51277
(1 row)
```

删除表空间：

postgres=# \h drop tablespace
Command:     DROP TABLESPACE
Description: remove a tablespace
Syntax:
DROP TABLESPACE [ IF EXISTS ] name

删除表空间前必须要删除该表空间下的所有数据库对象，否则无法删除。

如：

```sql
postgres=# drop tablespace if exists tbs_test;
ERROR:  tablespace "tbs_test" is not empty
```

删除刚才在此表空间创建的表test，然后再删除表空间。

```sql
postgres=# drop table if exists test;
DROP TABLE
postgres=# drop tablespace if exists tbs_test;
DROP TABLESPACE
```