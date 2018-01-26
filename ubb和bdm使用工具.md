### 工具配置使用

#### ubb的工具使用

##### 1. 配置各台主机的大端和小端,

```sql
\d+ ubb_config_ip 
                              Table "public.ubb_config_ip"
   Column   |         Type          | Modifiers | Storage  | Stats target | Description 
------------+-----------------------+-----------+----------+--------------+-------------
 ip_address | character varying(16) | not null  | extended |              | 地址IP
 max_port   | character varying(8)  | not null  | extended |              | 大P
 min_port   | character varying(8)  | not null  | extended |              | 小p
 group_ip   | character varying(8)  | not null  | extended |              | 用户组

select * from ubb_config_ip;
 ip_address  | max_port | min_port | group_ip 
-------------+----------+----------+----------
 20.200.31.3 | 16001    | 21000    | chmap
 20.200.31.5 | 16001    | 21000    | chagent

```

##### 2. 配置server 配置表 

```sql
                               Table "public.ubb_model"
   Column    |         Type          | Modifiers | Storage  | Stats target | Description 
-------------+-----------------------+-----------+----------+--------------+-------------
 hostgroup   | character varying(16) | not null  | extended |              | 主机组名
 servername  | character varying(32) | not null  | extended |              | 服务名称
 srvgrp      | character varying(32) | not null  | extended |              | ubb组名
 srvid_space | integer               | not null  | plain    |              | 服务ID间隔
 srvnum      | integer               | not null  | plain    |              | 服务数
 mb          | character varying(16) |           | extended |              | 邮箱

hostgroup |  servername   |  srvgrp   | srvid_space | srvnum | mb  
-----------+---------------+-----------+-------------+--------+-----
 chmap     | khxx_server   | KHXXGRP   |           1 |      3 | 205
 chmap     | back_server   | BACKGRP   |           1 |      2 | 206
 chmng     | qdzw_server   | QDZWGRP   |           1 |      2 | 205
 chdb      | rggjjs_server | RGGJJSGRP |           1 |      2 | 210

```

##### 3. 配置ubb_services 

```sql
                               Table "public.ubb_services"
  Column   |         Type          | Modifiers | Storage  | Stats target |  Description  
-----------+-----------------------+-----------+----------+--------------+---------------
 hostgroup | character varying(16) | not null  | extended |              | 主机组名
 services  | character varying(32) | not null  | extended |              | service服务名
 remark    | character varying(80) |           | extended |              | serivce注释
 
select * from ubb_services;
 hostgroup |   services    | remark 
-----------+---------------+--------
 chmap     | 99190000000   | 
 chmap     | 99420000000   | 
 chmng     | QDGLPT_612000 | 
 chmng     | QDGLPT_612010 | 
 chmng     | QDGLPT_612020 |
 
```

#### 4. 生成ubb

```
登录20.200.31.14 用户ansible
cd  /app/ansible/install/update_conf/chmap
执行sh chmap.sh 3-3(主机范围)
```



#### bdm的工具使用

##### 1. 配置services

```
  Column   |         Type          | Modifiers | Storage  | Stats target | Description 
-----------+-----------------------+-----------+----------+--------------+-------------
 hostgroup | character varying(16) | not null  | extended |              | 用户组
 services  | character varying(32) | not null  | extended |              | 服务名
 remark    | character varying(90) |           | extended |              | 注释

```

#### 2. 配置services

```
                                Table "public.bdm_config"
   Column    |         Type          | Modifiers | Storage  | Stats target | Description 
-------------+-----------------------+-----------+----------+--------------+-------------
 tracecode   | character varying(25) | not null  | extended |              | 交易码
 local_ip    | character varying(32) | not null  | extended |              | 渠道IP
 localport   | character varying(8)  | not null  | extended |              | 渠道端口
 remote_ip   | character varying(32) | not null  | extended |              | 远程IP
 remote_port | character varying(8)  | not null  | extended |              | 远程端口
 ldom        | character varying(16) | not null  | extended |              | 本地域名
 rdom        | character varying(16) | not null  | extended |              | 远程域名
 gwgrp       | character varying(16) |           | extended |              | 本地组名

tracecode     |   local_ip   | localport |   remote_ip   | remote_port |      ldom       |       rdom       |   gwgrp   
-------------------+--------------+-----------+---------------+-------------+-----------------+------------------+-----------
 PBTS20120416      | 20.200.31.13 | 9100      | 20.13.0.38    | 8002        | GUIYUAN_9103_13 | pbtsejb_domain   | GUIYUAN1
 TexudoFront       | 20.200.31.13 | 9705      | 20.5.193.223  | 9904        | CHMAP_HFYY_9705 | EbillsFundPoint1 | GJJSGRP1
 COM_SVC           | 20.200.31.13 | 9301      | 20.200.9.67   | 7156        | DLBX_HFYY_9301  | TDOM_CPAI        | DLBXGRP1
 9970004AgtAuth    | 20.200.31.13 | 9105      | 20.200.52.247 | 5584        | DOM_HFYY_9105   | DOM_JR27_5584    | GUANLIAN1

```

#### 3 . 配置主机组

```
                               Table "public.ubb_bdm_link"
  Column   |         Type          | Modifiers | Storage  | Stats target | Description 
-----------+-----------------------+-----------+----------+--------------+-------------
 hostgroup | character varying(16) | not null  | extended |              | 
 srvgrp    | character varying(32) | not null  | extended |              | 
 srvnum    | integer               |           | plain    |              | 
Indexes:
    "ubb_bdm_link_srvgrp_key" UNIQUE CONSTRAINT, btree (srvgrp)

 hostgroup |  srvgrp  | srvnum 
-----------+----------+--------
 chcom     | GUIYUAN  |      1
 chcom     | DZYZGRP  |      2
 chcom     | GJYWGRP  |      2
 chcom     | GJJSGRP  |      1
 chcom     | GUIMIAN  |      4
 chcom     | JYQZGRP  |      1

```

