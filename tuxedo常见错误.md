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

##### 4. 命令查询

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
export  TMTRACE=off   -UBB
changetrace -m  hostname on 关闭ubb

如果客户端出现12的错误,可能是由于tuxedo的debug,重启客户端

```

##### 4. 常见问题

```python
问题 1：
LIBGW_CAT:1099: ERROR: Unable to advertise initial services - TPELIMIT - a system limit has been reached
原因为resource的 maxservices配置太小
解决方案：增大resource段的 MAXSERVICES 大小

CMDGW_CAT:2077: ERROR: Specified Local domain FMTTEST1 not active or not configured
CMDGW_CAT:2077则没用链接连到远程域

问题 2：
ULOG： 105143.test1!tmadmin.12238.1.-2: LIBTUX_CAT:577: ERROR: Unable to register because the slot is already owned by another process
$ tmadmin
tmadmin - Copyright (c) 1996-1999 BEA Systems, Inc. Portions * Copyright 1986-1997 RSA Data Security, Inc. All Rights Reserved. Distributed under license by BEA Systems, Inc. Tuxedo is a registered trademark. TMADMIN_CAT:199: WARN: Cannot become administrator.Limited set of commands available.
原因：重复打开tmadmin管理，在重复打开的tmadmin中个别命令不能使用，通过help命令可以看到当前可以使用的命令。

问题 3：
174304.test1!WSH.20044.1.0: gtrid x0 x47fb1049 x16e: LIBTUX_CAT:1288: ERROR: File transfer creat failed, file=/var/tmp/TUXAAAa200441, errno=不允许 174304.test1!WSH.20044.1.0: gtrid x0 x47fb1049 x16e: WSNAT_CAT:1042: ERROR: tpcall() call failed, tperrno = 7
原因：
1288 ERROR: File transfer creat failed, file=filename, errno=errno_val
DESCRIPTION
The UNIX kernel call creat () failed on filename. This temporary file was being created to transfer a large message between two TUXEDO System processes on the same machine.
ACTION
Check temporary directory's permissions. Check disk space and inode counts for the temporary file system.

问题 4：
105516.test0!TMUSREVT.17177.1.0: gtrid x0 x48105214 xe: CMDTUX_CAT:3129: ERROR: tpenqueue() to qname PAYQUE failed for event EVT_PLC_EFFT tperrno=24
原因：PAYQUE队列没有建立，用qmadmin创建队列。
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
问题 5：
103331.test1!dydealtasksrv.21551.1.0: ERROR: msgsnd err: (LIBTUX_CAT:669: ERROR: Message operation failed because of the invalid message queue identifier) errno=22,qid=208507,buf=-9223372032559197904,bytes=293,flag=2048 103331.test1!dydealtasksrv.21551.1.0: LIBTUX_CAT:1286: ERROR: tpreturn could not send reply TPEOS - operating system error

原因：队列没有找到，可能是前台在后台返回前断开了服务连接，所以tpreturn时找不到
接收消息队列。或是其他原因导致队列被删除如 ipcrm -q qid
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
问题 6：

101503.lf2qjf2!TUXAGENT.17788: LIBTUX_CAT:536: ERROR: Unable to create request queue 101503.lf2qjf2!TUXAGENT.17788: LIBTUX_CAT:248: ERROR: System init function failed, Uunixerr = : msgget: No space left on device

原因：达到OS系统最大消息上限。使用ipcs -q|wc -l 查看当时建立得消息队列。
使用kmtune|grep msgmni 查看系统消息上限。
-----------------------------------------------------------------
问题 7：

111756.test1!BBL.23626.1.0: 12-11-2008: Tuxedo Version 8.1, 64-bit, Patch Level (none)
111756.test1!BBL.23626.1.0: LIBTUX_CAT:1000: ERROR: System clock has been reset to prior time. Reset again to time after Thu Dec 11 11:17:56 2008
.
111756.test1!BBL.23626.1.0: LIBTUX_CAT:248: ERROR: System init function failed, Uunixerr =
111756.test1!BBL.23626.1.0: CMDTUX_CAT:26: INFO: The BBL is exiting system
111756.test1!tmboot.23625.1.-2: 12-11-2008: Tuxedo Version 8.1, 64-bit
111756.test1!tmboot.23625.1.-2: CMDTUX_CAT:825: ERROR: Process BBL at ANNT_TEST failed with /T tperrno (TPESYSTEM - internal system error)
111756.test1!tmboot.23625.1.-2: WARN: No BBL available on site ANNT_TEST.
Will not attempt to boot server processes on that site.

原因：系统修改OS时间导致，重新创建TLOG 日志后此问题解决。
crdl 、crlog
```



