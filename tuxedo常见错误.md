## tuxedo 常见错误

#### 1. tmshutdown无法关闭

现象: tmshutdown: internal error: CMDTUX_CAT:766: ERROR: must run on master node

```c
tmipcrm -y // 资源未完全删除 ,删除所有资源
```

##### 2. ubb 配置

```
  MAXACCESSERS          1800   最大访问数，连接BBS的个数
  MAXSERVERS            800    最大exe个数
  MAXSERVICES           1820   最大服务数 = sum（每个exe提供服务数*该类exe个数）
  MAXWSCLIENTS=600             最大多少个连接WSL和JSL的客户端的个数

```

##### 3. 域连接

```
dmloadcf -y dbm.cfg
pd -d GUIYUAN_9100
```

#### 3. 命令查询

```shell
echo pq|tmadmin|grep qdgl    -- 消息队列是否堵塞,  ---（一般交易变慢，服务不够）
echo pclt |tmadmin|grep BUSY -- 查询客户端连接
echo psr|tmadmin |grep -v IDLE|grep qdgl_server -- 查看tuxedo服务是否繁忙
echo bbs|tmadmin            --  查看服务数总数
tmadmin psr 查看服务(exe)
Prog Name      Queue Name  Grp Name      ID      RqDone    Load Done             CurrenService
服务名称         队列名       组名      服务ID   tux调用次数   tux调用次数*50             当前服务
tmadmin psc 查看服务(原子函数)进程

读邮箱的程序是越忙得到的时间片越多. Out
读消息队列的程序,平均得到时间片. In

WSL SRVGRP=WSLGRP  SRVID=802 RQADDR=svrq802  MIN=1 MAX=1  CLOPT="-A -r -- -n //22.224.54.131:20000 -m 100 -M 300 -x 100 "

--m wsh初始化启动个数 echo pclt |tmadmin|grep BUSY
--M wsh的最大连接数 echo pclt |tmadmin|grep BUSY
--x 线程数 ()
Administrator
@WSX3edc

REQID=9901  --进平台
RESPID=9902 -- 出平台
ipcs -q 查到 msqid，根据msqid
ipcs -pq 查看进程号

export TM_NOALOG=Y   ----  access.* 日志文件不打
export TM_DISABLE_IMPLICIT_TPINIT=yes

如果客户端出现12的错误,可能是由于tuxedo的debug,重启客户端

```





