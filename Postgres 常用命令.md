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

### 查看表空间

```sql
postgres=# \db 
        Name         |      Owner      |                Location                
---------------------+-----------------+----------------------------------------
 idx_accountdb       | internetfinance | /data/dbdata/pgsql/idx_accountdb
 idx_internetfinance | internetfinance | /data/dbdata/pgsql/idx_internetfinance
 idx_productdb       | internetfinance | /data/dbdata/pgsql/idx_productdb

postgres=# select * from pg_tablespace ;
       spcname       | spcowner | spcacl | spcoptions 
---------------------+----------+--------+------------
 pg_default          |       10 |        | 
 pg_global           |       10 |        | 
 ts_internetfinance  |    16384 |        | 
 idx_internetfinance |    16384 |        | 
 ts_accountdb        |    16384 |        | 

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

#### 常用语句

```sql
select localdatetime,retcode,respinfo from  ol_transdetail where localdatetime> '20180109100000' and receprodcode='612020' and retcode='09209999';


select * from ol_transdetail where  localdatetime>'20180109163000';

select SUBSTR(localdatetime,9,4),logip,retcode,count(*),round(avg(protime),2) as 渠道耗时,round(avg(backtime),2) as 后台耗时 from ol_transdetail where localdate ='20180329'  and localdatetime > '20180329170300' GROUP BY SUBSTR(localdatetime,9,4),logip,retcode ORDER BY SUBSTR(localdatetime,9,4);
 
 select DISTINCT(logip) from ol_transdetail  ;
 
select logip,retcode,count(*),round(avg(protime),2) as 渠道耗时,round(avg(backtime),2) as 后台耗时  from ol_transdetail where localdatetime >= to_char(now() - interval '1 min','YYYYMMDDHHMMSS') group by  logip ,retcode;
```

#### 每分钟查询结果

```sql
select SUBSTR(localdatetime,9,4),logip,retcode,count(*),round(avg(protime),2) as 渠道耗时,round(avg(backtime),2) as 后台耗时 from ol_transdetail where  localdatetime > to_char(now() - interval '1 min','YYYYMMDDHHMISS') GROUP BY SUBSTR(localdatetime,9,4),logip,retcode ORDER BY SUBSTR(localdatetime,9,4),渠道耗时 desc;
```

### 创建基础备份

```sql
	vi pg_hba.conf
	host all all  192.168.67.35/24             md5
	host replication postgres 192.168.67.35/24 trust
	pg_basebackup -h node1 -U replication -D /data/postgresql/data/ -X stream -P
查看主备关系
	select * from pg_stat_replication ;
```

###用pg_rewind命令同步新备库

```
pg_rewind --target-pgdata $PGDATA --source-server='host=192.168.1.203 port=5432 user=postgres dbname=postgres' -P
mv recovery.done recovery.conf
vi recovery.conf

```

### 高可用（pcs+corosync+postgres）修复旧Master

```
检查集群状态   
由于设置了migration-threshold="3"，发生一次普通的错误，Pacemaker会在原地重新启动postgres进程，不发生主从切换。
（如果Master的物理机或网络发生故障，直接进行failover。）
```

第一种方法：可通过`pg_basebackup`修复旧Master（需要强行恢复的情况下）

```
# su - postgres
$ rm -rf /data/postgresql/data
$ pg_basebackup -h 192.168.41.136 -U postgres -D /data/postgresql/data -X stream -P
$ exit
# pcs resource cleanup msPostgresql
```

第二种方法：通过`pg_baseback`修复旧Master。`cls_rebuild_slave`是对`pg_basebackup`的包装，主要多了执行结果状态的检查。

```shell
[root@node1 pha4pgsql]# rm -rf /data/postgresql/data
[root@node1 pha4pgsql]# cls_rebuild_slave 
22636/22636 kB (100%), 1/1 tablespace
All resources/stonith devices successfully cleaned up
wait for recovery complete
.....
slave recovery of node1 successed
[root@node1 pha4pgsql]# cls_status
Last updated: Fri Apr 22 02:40:48 2016
Last change: Fri Apr 22 02:40:36 2016 by root via crm_resource on node1
Stack: corosync
Current DC: node2 (2) - partition with quorum
Version: 1.1.12-a14efad
2 Nodes configured
4 Resources configured
```

第三种方法: 9.5以上版本还可以通过`pg_rewind`修复旧Master

```shell
	[root@node1 pha4pgsql]# cls_repair_by_pg_rewind 
	connected to server
	servers diverged at WAL position 0/7000410 on timeline 2
	rewinding from last common checkpoint at 0/7000368 on timeline 2
	reading source file list
	reading target file list
	reading WAL in target
	need to copy 67 MB (total source directory size is 85 MB)
	69591/69591 kB (100%) copied
	creating backup label and updating control file
	syncing target data directory
	Done!
	All resources/stonith devices successfully cleaned up
	wait for recovery complete
	....
	slave recovery of node1 successed 
```

###  高可用（pcs+corosync+postgres）Slave上进程故障

再强制杀死Master上的postgres进程2次后检查集群状态。

fail-count增加到3后，Pacemaker不再启动PostgreSQL，保持其为停止状态。

同时，Master(node1)(VIP所在的主机)上的复制模式被自动切换到异步复制，防止写操作hang住。

```shell
[root@node1 pha4pgsql]# tail /var/lib/pgsql/tmp/rep_mode.conf
synchronous_standby_names = ''
```

#### 修复Salve   

在node2上执行`cls_cleanup`，清除fail-count后，Pacemaker会再次启动PostgreSQL进程。

```shell
[root@node2 ~]# cls_cleanup 
All resources/stonith devices successfully cleaned up
[root@node2 ~]# cls_status 
Last updated: Sun May  8 01:43:13 2016
Last change: Sun May  8 01:43:08 2016 by root via crm_resource on node1
Stack: corosync
Current DC: node2 (2) - partition with quorum
Version: 1.1.12-a14efad
2 Nodes configured
4 Resources configured
```

同时，Master(node1)上的复制模式又自动切换回到同步复制。

```
[root@node1 pha4pgsql]# tail /var/lib/pgsql/tmp/rep_mode.conf
synchronous_standby_names = 'node2'(只会有同步流复制)
```

## 错误排查

出现故障时，可通过以下方法排除故障

1. 确认集群服务是否OK

   pcs status

2. 查看错误日志

   PostgreSQL的错误日志
   	/var/log/messages
   	/var/log/cluster/corosync.log

   Pacemaker输出的日志非常多，可以进行过滤。

   只看Pacemaker的资源调度（在Current DC节点上执行)：

   ```
   grep Initiating /var/log/messages 
   ```

   只查看expgsql RA的输出：

   ```
   grep expgsql /var/log/messages
   ```

## 其它故障的处理

### 无Master时的修复

如果切换失败或其它原因导致集群中没有Master，可以参考下面的步骤修复

#### 方法1：使用cleanup修复

```shell
cls_cleanup
```

大部分情况，cleanup就可以找到Master。而且应该首先使用cleanup。如果不成功，再采用下面的方法

#### 方法2：人工修复复制关系

1. 将资源脱离集群管理

   cls_unmanage

2. 人工修复PostgreSQL，建立复制关系
   至于master的选取，可以选择`pgsql_REPL_INFO`中的master节点，或根据xlog位置确定。

3. 在所有节点上停止PostgreSQL

4. 清除状态并恢复集群管理

   cls_manage
   	cls_reset_master

#### 方法3：快速恢复Master节点再恢复Slave

可以明确指定将哪个节点作为Master，省略则通过xlog位置比较确定master

```
cls_reset_master [master]
```

### 疑难的Pacemaker问题的处理

有时候可能会遇到一些顽固的问题，Pacemaker不按期望的动作，或某个资源处于错误状态却无法清除。
这时最简单的办法就是清除CIB重新设置。可执行下面的命令完成。

```
./setup.sh [master]
```

如果不指定master，并且PostgreSQL进程是活动的，通过当前PostgreSQL进程的主备关系决定谁是master。
如果当前没有处于主的PostgreSQL进程，通过比较xlog位置确定谁作为master。

在PostgreSQL服务启动期间，正常情况下，执行setup.sh不会使服务停止。

setup.sh还可以完全取代前面的`cls_reset_master`。

### fail-count的清除

如果某个节点上有资源的fail-count不为0，最好将其清除，即使当前资源是健康的。

```
cls_cleanup
ps resource cleanup msPostgresql
```

###注意事项

1. ./setup.sh会清除CIB，对Pacemaker资源定义的修改应该写到config.pcs里，防止下次执行setup.sh丢失。
2. 有些包装后的脚本容易超时，比如`cls_rebuild_slave`。此时可能执行还没有完成的，需要通过`cls_status`或日志进行确认。

### pg_log 和pg_xlog 和pg_clog的区别

```shell
#pg_xlog
这个日志是记录的Postgresql的WAL信息，也就是一些事务日志信息(transaction log)，默认单个大小是16M，源码安装的时候可以更改其大小。这些信息通常名字是类似'000000010000000000000013'这样的文件，这些日志会在定时回滚恢复(PITR)，流复制(Replication Stream)以及归档时能被用到，这些日志是非常重要的，记录着数据库发生的各种事务信息，不得随意删除或者移动这类日志文件，不然你的数据库会有无法恢复的风险

当你的归档或者流复制发生异常的时候，事务日志会不断地生成，有可能会造成你的磁盘空间被塞满，最终导致DB挂掉或者起不来。遇到这种情况不用慌，可以先关闭归档或者流复制功能，备份pg_xlog日志到其他地方，但请不要删除。然后删除较早时间的的pg_xlog，有一定空间后再试着启动Postgres。

#pg_clog
pg_clog这个文件也是事务日志文件，但与pg_xlog不同的是它记录的是事务的元数据(metadata)，这个日志告诉我们哪些事务完成了，哪些没有完成。这个日志文件一般非常小，但是重要性也是相当高，不得随意删除或者对其更改信息。

总结：
#pg_log记录各种Error信息，以及服务器与DB的状态信息，可由用户随意更新删除
#pg_xlog与pg_clog记录数据库的事务信息，不得随意删除更新，做物理备份时要记得备份着两个日志。
```

### WARNING:  out of shared memory

```
sysctl.conf
kernel.shmmax = 12884901888
kernel.shmall = 4718592
vm.swappiness = 0
vm.overcommit_memory = 2
vm.overcommit_ratio = 80
vm.dirty_ratio = 10
vm.dirty_background_ratio = 1
vm.vfs_cache_pressure=150
vm.dirty_writeback_centisecs = 50
net.ipv4.tcp_tw_recycle=1
net.ipv4.tcp_tw_reuse=1
vm.nr_hugepages=2300

postgresql.conf
shared_buffers = 4GB # min 128kB
huge_pages = on # on, off, or try
work_mem = 64MB # min 64kB
maintenance_work_mem = 512MB # min 1MB
max_stack_depth = 8MB # min 100kB
dynamic_shared_memory_type = posix # the default is the first option
```

