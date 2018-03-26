## Postgres 常用命令

##### 创建数据库

```SQL
CREATE DATABASE test WITH OWNER = postgres ENCODING = 'UTF8';
```

##### 查看表的索引：

```SQL
select * from pg_indexes where tablename='log';
```

##### 导出备份数据库：

``` sql
pg_dump -h localhost -U postgres databasename > /tmp/databasename.bak.yyyymmdd.sql
```

##### 导入恢复数据库(sql文件是pg_dump导出的文件就行，可以是整个数据库，也可以只是单个表，也可以只是结构等)

``` sql lite
psql -h localhost -U postgres -d databasename < /tmp/databasename.bak.yyyymmdd.sql
```

##### 导出数据结构，主要是加上参数-s：

``` sql
pg_dump -U username -W dbname -f /tmp/filename.sql
```

##### 导出某个表：

``` sql
pg_dump -h localhost -U postgres -t tablename dbname > test.sql
```

##### 导出某个表的结构，同样是加参数"-s"：

``` sql
pg_dump -h localhost -U postgres -t tablename -s dbname > test_construct.sql
```

##### 导出某个表的数据，加参数"-a"：

``` sql
pg_dump -h localhost -U postgres -t tablename -a dbname > test_data.sql
```

``` sql
查看序列：select * from information_schema.sequences where sequence_schema = 'public';
查看数据库大小：select pg_size_pretty(pg_database_size('test'));
查看表的大小：select pg_size_pretty(pg_relation_size('test'));
```

##### 由于 copy 命令始终是到数据库服务端找文件，当以文件形式导入导出数据时需以超级用户执行，权限要求很高，适合数据库管理员操作;而 \copy 命令可在客户端执行导入客户端的数据文件，权限要求没那么高，适合开发人员，测试人员使用，因为生产库的权限掌握在 DBA 手中。

``` sql
copy (select * from tablename) to '/tmp/1.txt' (format csv,delimiter '|');
\copy (select * from tablename) to '/tmp/1.txt' with delimiter '|';

psql -c "\copy (select * from PBX_CASH_RCV_REG where TRAN_DATE='20170913') to 'PBX_CASH_RCV_REG.txt' with delimiter '|' NULL ' '"
```

##### 清除所有表

``` sql
CREATE FUNCTION aaa() RETURNS void AS $$
DECLARE
    tmp VARCHAR(512);
DECLARE names CURSOR FOR 
    select tablename from pg_tables where schemaname='public' ;
BEGIN
  FOR stmt IN names LOOP
    tmp := 'DROP TABLE '|| quote_ident(stmt.tablename) || ' CASCADE;';
RAISE NOTICE 'notice: %', tmp;
    EXECUTE 'DROP TABLE '|| quote_ident(stmt.tablename) || ' CASCADE;';
  END LOOP;
  RAISE NOTICE 'finished .....';
END;
$$ LANGUAGE plpgsql;

调用时用：
select aaa();
```

##### 获取系统表中字段信息

``` sql
SELECT a.attnum,a.attname AS field,t.typname AS type,a.attlen AS length,a.atttypmod AS lengthvar,a.attnotnull AS notnull FROM pg_class c,pg_attribute a,pg_type t
WHERE c.relname = 'dbmanager_hostlist' and a.attnum > 0 and a.attrelid = c.oid and a.atttypid = t.oid ORDER BY a.attnum;
```

``` sql
\dtS   --- 查看系统表
\dtS+  ---  查看详细信息
comment on table t IS 'this is a test'; --- 添加表注释
\set AUTOCOMMIT off --- 关闭自动更新
```

##### 指定去重的列排序

``` sql
 select distinct on (subject) id,name,subject,score from table order by subject,score desc;
 扫描10行
 select * from table limit 1 offset 10;
```

##### 创建序列

``` sql
create sequence seq
select nextval('seq');
select nextval('seq') from window_test;
alter function nextval(regclass) volatile; -- 只执行一遍 
alter function nextval(regclass) stable; 
select * from generate_series(1,30) as g(i) where i = nextval('seq'::regclass);
```

##### 锁表问题

```sql
--查询是否锁表了                                       
select oid from pg_class where relname='可能锁表了的表'
select pid from pg_locks where relation='上面查出的oid'
--如果查询到了结果，表示该表被锁 则需要释放锁定        
select pg_cancel_backend(上面查到的pid) 
```

#####  增删改查的操作

``` sql
--- 修改字段名
alter table ubb_model rename column hostname to hostgroup;
--- 删除唯一约束
alter table ubb_services drop constraint ubb_services_services_key;
--- 增加一列
ALTER TABLE table_name ADD column_name datatype;                   
--- 删除一列
ALTER TABLE table_name DROP  column_name;                           
--- 更改列的数据类型
ALTER TABLE table_name ALTER  column_name TYPE datatype;              
--- 表的重命名
ALTER TABLE table_name RENAME TO new_name;                           
--- 更改列的名字
ALTER TABLE table_name RENAME column_name to new_column_name;          
--- 字段的not null设置
ALTER TABLE table_name ALTER column_name {SET|DROP} NOT NULL;          
--- 给列添加default
ALTER TABLE table_name ALTER column_name SET DEFAULT expression;
--- 添加主键
alter table goods add primary key(sid);
--- 添加外键
alter table orders add foreign key(goods_id) references goods(sid)  on update cascade on delete cascade;
on update cascade: --- 被引用行更新时，引用行自动更新； 
on update restrict: --- 被引用的行禁止更新；
on delete cascade: --- 被引用行删除时，引用行也一起删除；
on delete restrict: --- 被引用的行禁止删除；
--- 删除外键
alter table orders drop constraint orders_goods_id_fkey;
--- 添加唯一约束
alter table goods add constraint unique_goods_sid unique(sid);
--- 删除默认值
alter table goods  alter column sid drop default;
---  修改字段的数据类型
alter table goods alter column sid type character varying;
--- 重命名字段
alter table goods rename column sid to ssid;
```

##### 时间的使用方法:

``` sql
select  to_char(CURRENT_DATE,'YYYYMMDD');
select  to_char(CURRENT_TIMESTAMP,'YYYYMMDD');
select  to_char(CURRENT_TIMESTAMP,'YYYYMMDDMSUS');	
select  to_char(CURRENT_TIMESTAMP,'HH24MISS');
--- 时间的计算
select CURRENT_DATE  + integer '7';
select CURRENT_TIME  + interval '1 hour';
select CURRENT_TIME  + time '03:00';
```

##### 如何找到postgreSQL数据库中占空间最大的表？

``` sql
 SELECT relname,relpages FROM pg_class ORDER BY relpages DESC;
如果你只想要最大的那个表，可以用limit参数来限制结果的数量，就像这样：
SELECT relname, relpages FROM pg_class ORDER BY relpages DESC limit 1;

1.relname - 关系名/表名
2.relpages - 关系页数（默认情况下一个页大小是8kb）
3.pg_class - 系统表, 维护着所有relations的详细信息
4.limit 1 - 限制返回结果只显示一行
```

##### 如何计算postgreSQL数据库所占用的硬盘大小？

```  sql
pg_database_size 这个方法是专门用来查询数据库大小的，它返回的结果单位是字节（bytes）。:
SELECT pg_database_size('geekdb');
如果你想要让结果更直观一点，那就使用pg_size_pretty方法，它可以把字节数转换成更友好易读的格式。
SELECT pg_size_pretty(pg_database_size('geekdb'));
```

##### 如何计算postgreSQL表所占用的硬盘大小？

``` sql
下面这个命令查出来的表大小是包含索引和toasted data的，如果你对除去索引外仅仅是表占的大小感兴趣，可以 使用后面提供的那个命令。
SELECT pg_size_pretty(pg_total_relation_size('big_table'));
```

##### 类别一、 如果两张张表（导出表和目标表）的字段一致，并且希望插入全部数据，可以用这种方法：

```
INSERT INTO  目标表  SELECT  * FROM  来源表 ;
```

##### 要将 articles 表插入到 newArticles 表中，则可以通过如下SQL语句实现：

```
INSERT INTO  newArticles  SELECT  * FROM  articles ;
```

#####  如果只希望导入指定字段，可以用这种方法：

```
INSERT INTO  目标表 (字段1, 字段2, ...)  SELECT   字段1, 字段2, ...   FROM  来源表 ;
```

``` sql
INSERT INTO TPersonnelChange(  
    UserId,  
    DepId,  
    SubDepId,  
    PostionType,  
    AuthorityId,  
    ChangeDateS,  
    InsertDate,  
    UpdateDate,  
    SakuseiSyaId  
)SELECT  
    UserId,  
    DepId,  
    SubDepId,  
    PostionType,  
    AuthorityId,  
    DATE_FORMAT(EmployDate, '%Y%m%d'),  
    NOW(),  
    NOW(),  
    1  
FROM  
    TUserMst  
WHERE  
    `Status` = 0  
AND QuitFlg = 0  
AND UserId > 2  
```

##### [generate_series函数应用](http://www.cnblogs.com/mchina/archive/2013/04/03/2997722.html)

##### 批量添加数据

``` sql
select generate_series(1, 10); --- int
select generate_series(1, 10, 3); --- 如果step 是正数，而start 大于stop，那么返回零行。相反，如果step 是负数，start 小于stop，则返回零行。如果是NULL 输入，也产生零行。step 为零则是一个错误。
select generate_series(5,1,-1);
select generate_series(now(), now() + '7 days', '1 day');
select generate_series(to_date('20130403','yyyymmdd'), to_date('20130404','yyyymmdd'), '3 hours');  
```

#### 锁的问题

```
1.如何锁一个表的某一行
select * from table rowlock where id =1

```

