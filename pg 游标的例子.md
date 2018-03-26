# PostgreSQL 用游标优化的一个例子

*摘要：* 一位PG社区的朋友提到的一个应用场景，目前遇到性能问题。 数据结构大概是这样的，包含一个主键，一个数组，一个时间，其他字段。 请求分析： 有检索需求，比较频繁。查找数组中包含某些元素的记录，并按时间排序输出所有符合条件的记录，检索到的符合条件的记录可能上万条，也可能较少。

```sql
一位PG社区的朋友提到的一个应用场景，目前遇到性能问题。
数据结构大概是这样的，包含一个主键，一个数组，一个时间，其他字段。
请求分析：
有检索需求，比较频繁。查找数组中包含某些元素的记录，并按时间排序输出所有符合条件的记录，检索到的符合条件的记录可能上万条，也可能较少。
有插入需求，量不大。
有更新需求，一条记录最多一天会被更新一次，当然也可能不会被更新。
无删除需求。
数据量在千万级别。

这个应用场景的不安定因素来自于一些热点值。
例如，当输出的数据量较大时，排序对CPU的开销较大。而这些热点值可能也是查询的热点。
对于检索的条件是数组，这个可以用GIN索引来解决，只有排序是无法解决的。

测试，生成300万测试记录：
postgres=# create table test(id int primary key,info int[],crt_date date);
CREATE TABLE
postgres=# insert into test select generate_series(1,3000000), ('{'||round(random()*1000)||','||round(random()*1000)||','||round(random()*1000)||'}')::int[], current_date+round(random()*1000)::int;
INSERT 0 3000000
postgres=# create index idx_test_info on test using gin(info);
CREATE INDEX
当输出记录较少时，效率还是可以的，例如以下：
postgres=# explain (analyze,verbose,buffers,timing) select info,crt_date from test where info @> '{1,8}'::int[] order by crt_date desc;
                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=101.23..101.29 rows=22 width=37) (actual time=1.668..1.672 rows=21 loops=1)
   Output: info, crt_date
   Sort Key: test.crt_date DESC
   Sort Method: quicksort  Memory: 26kB
   Buffers: shared hit=26
   ->  Bitmap Heap Scan on public.test  (cost=16.17..100.74 rows=22 width=37) (actual time=1.609..1.647 rows=21 loops=1)
         Output: info, crt_date
         Recheck Cond: (test.info @> '{1,8}'::integer[])
         Heap Blocks: exact=21
         Buffers: shared hit=26
         ->  Bitmap Index Scan on idx_test_info  (cost=0.00..16.17 rows=22 width=0) (actual time=1.595..1.595 rows=21 loops=1)
               Index Cond: (test.info @> '{1,8}'::integer[])
               Buffers: shared hit=5
 Planning time: 0.224 ms
 Execution time: 1.722 ms
(15 rows)
返回21行，算上排序需要1.7毫秒。
但是如果返回记录数上万之后，来看看结果：
postgres=# explain (analyze,verbose,buffers,timing) select info,crt_date from test where info @> '{1}'::int[] order by crt_date desc;
                                                            QUERY PLAN                                                             
-----------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=7737.83..7754.58 rows=6700 width=37) (actual time=17.726..18.856 rows=8896 loops=1)
   Output: info, crt_date
   Sort Key: test.crt_date DESC
   Sort Method: quicksort  Memory: 1080kB
   Buffers: shared hit=5028
   ->  Bitmap Heap Scan on public.test  (cost=59.93..7312.04 rows=6700 width=37) (actual time=3.722..13.585 rows=8896 loops=1)
         Output: info, crt_date
         Recheck Cond: (test.info @> '{1}'::integer[])
         Heap Blocks: exact=5025
         Buffers: shared hit=5028
         ->  Bitmap Index Scan on idx_test_info  (cost=0.00..58.25 rows=6700 width=0) (actual time=2.620..2.620 rows=8896 loops=1)
               Index Cond: (test.info @> '{1}'::integer[])
               Buffers: shared hit=3
 Planning time: 0.151 ms
 Execution time: 19.637 ms
(15 rows)
返回8896行，算上排序需要19.6毫秒。（这是返回所有记录的时间，如果是分页的话，第一页会很快返回）

优化建议。
1. 如果遇到排序带来的CPU负载过高的问题，可以创建热值partial index
对于热值，创建partial index。例如以上热值：
postgres=# create index idx_test_info_1 on test (crt_date) where info @> '{1}'::int[];
CREATE INDEX
禁止排序
postgres=# set enable_sort=off;
SET
postgres=# explain (analyze,verbose,buffers,timing) select * from test where info @> '{1}'::int[] order by crt_date desc;
                                                                   QUERY PLAN                                                       
             
------------------------------------------------------------------------------------------------------------------------------------
-------------
 Index Scan Backward using idx_test_info_1 on public.test  (cost=0.29..18253.53 rows=6700 width=41) (actual time=0.013..9.147 rows=8
896 loops=1)
   Output: id, info, crt_date
   Buffers: shared hit=8909
 Planning time: 0.253 ms
 Execution time: 9.911 ms
(5 rows)
当然这么做有很大的弊端，因为如果热值比较多，我们要为各种热值相关的查询条件创建很多的索引。

2. 因为一条记录一天最多更新一次，所以完全可以使用应用层缓存，或者pgmemcache这样的缓存插件，降低数据库的负担。

3. 使用游标，我们注意到用户使用了分页显示，但是对于用户来说，可能只会看第一页或前几页的内容，所以每次都全部取到程序端是没有必要的，用游标会更好。（注意不要使用order by limit x offset x这种方式分页，会冗余扫描多次，请使用cursor，但是记得用完关闭。）详见驱动API，如pg-jdbc。

压力测试：
测量类似分页，我这里只取第一页的内容(使用热值partial index)。
注意这种用法不是游标的用法。只是方便这里测试的。
vi test.sql
select * from test where info @> '{1}'::int[] order by crt_date desc limit 10;
性能非常可观：
pg95@db-172-16-3-150-> pgbench -M prepared -n -r -f ./test.sql -P 1 -c 16 -j 16 -T 30
progress: 1.0 s, 72844.1 tps, lat 0.213 ms stddev 0.119
progress: 2.0 s, 73691.9 tps, lat 0.215 ms stddev 0.019
progress: 3.0 s, 73603.7 tps, lat 0.216 ms stddev 0.018
progress: 4.0 s, 73501.3 tps, lat 0.216 ms stddev 0.063
progress: 5.0 s, 73433.2 tps, lat 0.216 ms stddev 0.049
progress: 6.0 s, 73645.1 tps, lat 0.216 ms stddev 0.023
progress: 7.0 s, 73551.0 tps, lat 0.216 ms stddev 0.060
progress: 8.0 s, 73640.9 tps, lat 0.216 ms stddev 0.018
progress: 9.0 s, 73650.8 tps, lat 0.216 ms stddev 0.027
progress: 10.0 s, 73753.5 tps, lat 0.215 ms stddev 0.068
对比一次取完所有数据的性能：
pg95@db-172-16-3-150-> vi test.sql
select * from test where info @> '{1}'::int[] order by crt_date desc;

pg95@db-172-16-3-150-> pgbench -M prepared -n -r -f ./test.sql -P 1 -c 16 -j 16 -T 30
progress: 1.0 s, 219.9 tps, lat 68.165 ms stddev 7.355
progress: 2.0 s, 233.8 tps, lat 67.849 ms stddev 15.181
progress: 3.0 s, 238.4 tps, lat 68.023 ms stddev 10.556
progress: 4.0 s, 233.9 tps, lat 68.030 ms stddev 4.459
progress: 5.0 s, 233.6 tps, lat 68.019 ms stddev 4.131
progress: 6.0 s, 235.5 tps, lat 67.472 ms stddev 3.204
progress: 7.0 s, 237.7 tps, lat 67.627 ms stddev 3.257
progress: 8.0 s, 233.5 tps, lat 67.779 ms stddev 4.815
progress: 9.0 s, 238.7 tps, lat 67.723 ms stddev 7.603
progress: 10.0 s, 232.0 tps, lat 68.098 ms stddev 13.948

[参考]
1. http://www.postgresql.org/docs/9.4/static/functions-array.html
```