## pg_pathman 安装

```sql lite
# 原因是root下的，而pg的path是配置在postgres用户下的
[root@bogon opt]#  export PATH=/home/postgres:$PATH
[root@bogon opt]# cd pg_pathman
[root@bogon opt]# make USE_PGXS=1
[root@bogon opt]# make USE_PGXS=1 install

#更改pg的配置文件
[root@bogon pg_pathman]# cd $PGDATA
[root@bogon data]# vim postgresql.conf
#将shared_preload_libraries注释取消，将下面变量赋值进去
shared_preload_libraries = 'pg_pathman,pg_stat_statements' 
# esc退出，wq!保存退出！

#启动数据库
[root@bogon data]# su - postgres
[postgres@bogon ~]$ pg_ctl start -D $PGDATA
#如果已经启动，更改配置后快速重启
# [postgres@bogon ~]$ pg_ctl restart -m fast

#进入数据库，配置扩展，比如我之前创建的一个 Test数据库
[postgres@bogon ~]$ psql Test
psql (9.6.0)
Type "help" for help.

Test=# create extension pg_pathman;
CREATE EXTENSION
# 查看已安装的扩展
Test=# \dx

# 创建分区表
create table part_test(id int, info text, crt_time timestamp not null);
insert into part_test select id,md5(random()::text),clock_timestamp() from generate_series(1,10000) t(id);
select create_range_partitions(
        'part_test'::regclass,           --主表oid
        'crt_time',                     --分区字段，一定要not null约束
        '2017-11-07 01:51:27'::timestamp, --开始时间
        interval '1 day',               --分区间隔，一个月
        10,                             --分区表数量
        false                           --  不立即将数据从主表迁移到分区,
);

select partition_table_concurrently('part_test'::regclass,10000,1.0);

select partition_table_concurrently
('oa_epp_user'::regclass,                --主表OID
10000,                                    --一个事务批量迁移多少记录
 1.0);   

```

