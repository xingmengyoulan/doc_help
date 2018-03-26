# heartbeat + pacemaker实现pg流复制自动切换

heartbeat + pacemaker + postgres_streaming_replication

说明：

该文档用于说明以hearbeat+pacemaker的方式实现PostgreSQL流复制自动切换。注意内容包括有关hearbeat/pacemaker知识总结以及整个环境的搭建过程和问题处理。

# 一、介绍

Heartbeat

自3版本开始，heartbeat将原来项目拆分为了多个子项目（即多个独立组件），现在的组件包括：heartbeat、cluster-glue、resource-agents。

各组件主要功能：

heartbeat：属于集群的信息层，负责维护集群中所有节点的信息以及各节点之间的通信。

cluster-glue：包括LRM（本地资源管理器）、STONITH，将heartbeat与crm（集群资源管理器）联系起来，属于一个中间层。

resource-agents：即各种资源脚本，由LRM调用从而实现各个资源的启动、停止、监控等。

Heartbeat内部组件关系图：

[![heartbeat + pacemaker实现pg流复制自动切换](http://image.codeweblog.com/upload/8/d9/8d9161582b98caaa_thumb.png)](http://image.codeweblog.com/upload/8/d9/8d9161582b98caaa.png)

Pacemaker

Pacemaker，即Cluster Resource Manager（CRM），管理整个HA，客户端通过pacemaker管理监控整个集群。

常用的集群管理工具：

（1）基于命令行

crm shell/pcs

（2）基于图形化

pygui/hawk/lcmc/pcs

Hawk:<http://clusterlabs.org/wiki/Hawk>

Lcmc:<http://www.drbd.org/mc/lcmc/>

Pacemaker内部组件、模块关系图：

[![heartbeat + pacemaker实现pg流复制自动切换](http://image.codeweblog.com/upload/9/03/90373fecab03f95e_thumb.png)](http://image.codeweblog.com/upload/9/03/90373fecab03f95e.png)

# 二、环境

## 2.1 OS

```
# cat /etc/issue
CentOS release 6.4 (Final)
Kernel \r on an \m

# uname -a
Linux node1 2.6.32-358.el6.x86_64 #1 SMP Fri Feb 22 00:31:26 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

```

2.2 IP

node1:

eth0 192.168.100.161/24 GW 192.168.100.1 ---真实地址

eth1 2.2.2.1/24 ---心跳地址

eth2 192.168.2.1/24 ---流复制地址

node2:

eth0 192.168.100.162/24 GW 192.168.100.1 ---真实地址

eth1 2.2.2.2/24 ---心跳地址

eth2 192.168.2.2/24 ---流复制地址

虚拟地址：

eth0:0 192.168.100.163/24 ---vip-master

eth0:0 192.168.100.164/24 ---vip-slave

eth2:0 192.168.2.3/24 ---vip-rep

## 2.3 软件版本

```
# rpm -qa | grep heartbeat
heartbeat-devel-3.0.3-2.3.el5
heartbeat-debuginfo-3.0.3-2.3.el5
heartbeat-3.0.3-2.3.el5
heartbeat-libs-3.0.3-2.3.el5
heartbeat-devel-3.0.3-2.3.el5
heartbeat-3.0.3-2.3.el5
heartbeat-debuginfo-3.0.3-2.3.el5
heartbeat-libs-3.0.3-2.3.el5

# rpm -qa | grep pacemaker
pacemaker-libs-1.0.12-1.el5.centos
pacemaker-1.0.12-1.el5.centos
pacemaker-debuginfo-1.0.12-1.el5.centos
pacemaker-debuginfo-1.0.12-1.el5.centos
pacemaker-1.0.12-1.el5.centos
pacemaker-libs-1.0.12-1.el5.centos
pacemaker-libs-devel-1.0.12-1.el5.centos
pacemaker-libs-devel-1.0.12-1.el5.centos

# rpm -qa | grep resource-agent
resource-agents-1.0.4-1.1.el5

# rpm -qa | grep cluster-glue
cluster-glue-libs-1.0.6-1.6.el5
cluster-glue-libs-1.0.6-1.6.el5
cluster-glue-1.0.6-1.6.el5
cluster-glue-libs-devel-1.0.6-1.6.el5

```

PostgreSQL Version：9.1.4

# 三、安装

## 3.1 设置YUM源

```
# wget -O /etc/yum.repos.d/pacemaker.repo http://clusterlabs.org/rpm/epel-5/clusterlabs.repo

```

## 3.2 安装heartbeat/pacemaker

安装libesmtp：

```
# wget ftp://ftp.univie.ac.at/systems/linux/fedora/epel/5/x86_64/libesmtp-1.0.4-5.el5.x86_64.rpm
# wget ftp://ftp.univie.ac.at/systems/linux/fedora/epel/5/i386/libesmtp-1.0.4-5.el5.i386.rpm
# rpm -ivh libesmtp-1.0.4-5.el5.x86_64.rpm
# rpm -ivh libesmtp-1.0.4-5.el5.i386.rpm

```

安装pacemaker corosync:

```
# yum install heartbeat* pacemaker*

```

通过命令查看资源脚本：

```
# crm ra list ocf
AoEtarget            AudibleAlarm         CTDB                 ClusterMon           Delay                Dummy                EvmsSCC
Evmsd                Filesystem           HealthCPU            HealthSMART          ICP                  IPaddr               IPaddr2
IPsrcaddr            IPv6addr             LVM                  LinuxSCSI            MailTo               ManageRAID           ManageVE
Pure-FTPd            Raid1                Route                SAPDatabase          SAPInstance          SendArp              ServeRAID
SphinxSearchDaemon   Squid                Stateful             SysInfo              SystemHealth         VIPArip              VirtualDomain
WAS                  WAS6                 WinPopup             Xen                  Xinetd               anything             apache
conntrackd           controld             db2                  drbd                 eDir88               exportfs             fio
iSCSILogicalUnit     iSCSITarget          ids                  iscsi                jboss                ldirectord           mysql
mysql-proxy          nfsserver            nginx                o2cb                 oracle               oralsnr              pgsql
ping                 pingd                portblock            postfix              proftpd              rsyncd               scsi2reservation
sfex                 syslog-ng            tomcat               vmware

```

禁止开机启动：

```
# chkconfig heartbeat off

```

3.3 安装PostgreSQL

安装目录为/opt/pgsql

{安装过程略}

为postgres用户配置环境变量：

```
[postgres@node1 ~]$ cat .bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
 . ~/.bashrc
fi

# User specific environment and startup programs

export PATH=/opt/pgsql/bin:$PATH:$HOME/bin
export PGDATA=/opt/pgsql/data
export PGUSER=postgres
export PGPORT=5432
export LD_LIBRARY_PATH=/opt/pgsql/lib:$LD_LIBRARY_PATH

```

# 四、配置

## 4.1 hosts设置

```
# vim /etc/hosts
192.168.100.161 node1
192.168.100.162 node2
```

## 4.2 配置heartbeat

创建配置文件：

```
# cp /usr/share/doc/heartbeat-3.0.3/ha.cf /etc/ha.d/
```

修改配置：

```
# vim /etc/ha.d/ha.cf
logfile /var/log/ha-log
logfacility local0
keepalive 2
deadtime 30
warntime 10
initdead 120
ucast eth1 2.2.2.2 #node2上改为2.2.2.1
auto_failback off
node node1
node node2
pacemaker respawn #自heartbeat-3.0.4改为crm respawn
```

4.3 生成密钥

```
# (echo -ne "auth 1\n1 sha1 ";dd if=/dev/urandom bs=512 count=1 | openssl sha1 ) > /etc/ha.d/authkeys
1+0 records in
1+0 records out
512 bytes (512 B) copied, 0.00032444 s, 1.6 MB/s

# chmod 0600 /etc/ha.d/authkeys
```

## 4.4 同步配置

```
[root@node1 ~]# cd /etc/ha.d/
[root@node1 ha.d]# scp authkeys ha.cf node2:/etc/ha.d/

```

## 4.5 下载替换脚本

pgsql脚本过旧，不支持配置pgsql.crm中设置的一些参数，需要从网上下载并替换pgsql

下载地址：

[https://github.com/ClusterLabs/resource-agents]()�

```
# unzip resource-agents-master.zip
# cd resource-agents-master/heartbeat/
# cp pgsql /usr/lib/ocf/resource.d/heartbeat/
# cp ocf-shellfuncs.in /usr/lib/ocf/lib/heartbeat/ocf-shellfuncs
# cp ocf-rarun /usr/lib/ocf/lib/heartbeat/ocf-rarun
# chmod 755 /usr/lib/ocf/resource.d/heartbeat/pgsql
```

修改ocf-shellfuncs：

if [ -z "$OCF_ROOT" ]; then

\# : ${OCF_ROOT=@OCF_ROOT_DIR@}

: ${OCF_ROOT=/usr/lib/ocf}

Fi

pgsql资源脚本特性：

●主节点失效切换

master宕掉时，RA检测到该问题并将master标记为stop，随后将slave提升为新的master。

●异步与同步切换

如果slave宕掉或者LAN中存在问题，那么当设置为同步复制时包含写操作的事务将会被终止，也就意味着服务将停止。因此，为防止服务停止RA将会动态地将同步转换为异步复制。

●初始启动时自动识别新旧数据

当两个或多个节点上的Pacemaker同时初始启动时，RA通过每个节点上最近的replay location进行比较，找出最新数据节点。这个拥有最新数据的节点将被认为是master。当然，若在一个节点上启动pacemaker或者该节点上的pacemaker是第一个被启动的，那么它也将成为master。RA依据停止前的数据状态进行裁定。

●读负载均衡

由于slave节点可以处理只读事务，因此对于读操作可以通过虚拟另一个虚拟IP来实现读操作的负载均衡。

## 4.6 启动heartbeat

启动：

```
[root@node1 ~]# service heartbeat start
[root@node2 ~]# service heartbeat start

```

检测状态：

```
[root@node1 ~]# crm status
============
Last updated: Fri Jan 24 08:02:54 2014
Stack: Heartbeat
Current DC: node2 (43a4f083-c5d3-4c66-a387-b05d79b5dd89) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
0 Resources configured.
============

Online: [ node1 node2 ]

```

{heartbeat启动成功}

测试：

禁用stonith，创建一个虚拟ip资源vip

```
[root@node1 ~]# crm configure property stonith-enabled="false"
[root@node1 ~]# crm configure
crm(live)configure# primitive vip ocf:heartbeat:IPaddr2 \
>     params \
>         ip="192.168.100.90" \
>         nic="eth0" \
>         cidr_netmask="24" \
>     op start   timeout="60s" interval="0s"  on-fail="stop" \
>     op monitor timeout="60s" interval="10s" on-fail="restart" \
>     op stop    timeout="60s" interval="0s"  on-fail="block"
crm(live)configure# commit
crm(live)configure# quit
bye

```

```
[root@node2 heartbeat]# crm_mon -1
============
Last updated: Fri Jan 24 08:23:09 2014
Stack: Heartbeat
Current DC: node2 (43a4f083-c5d3-4c66-a387-b05d79b5dd89) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
1 Resources configured.
============

Online: [ node1 node2 ]

 vip (ocf::heartbeat:IPaddr2): Started node1

```

{vip资源在node1上运行}

```
[root@node1 ~]# ping 192.168.100.90
PING 192.168.100.90 (192.168.100.90) 56(84) bytes of data.
64 bytes from 192.168.100.90: icmp_seq=1 ttl=64 time=0.060 ms
64 bytes from 192.168.100.90: icmp_seq=2 ttl=64 time=0.111 ms
64 bytes from 192.168.100.90: icmp_seq=3 ttl=64 time=0.123 ms

```

模拟node1故障

```
[root@node1 ~]# service heartbeat stop

[root@node2 heartbeat]# crm_mon -1
============
Last updated: Fri Jan 24 08:22:22 2014
Stack: Heartbeat
Current DC: node2 (43a4f083-c5d3-4c66-a387-b05d79b5dd89) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
1 Resources configured.
============

Online: [ node2 ]
OFFLINE: [ node1 ]

 vip (ocf::heartbeat:IPaddr2): Started node2

```

{node2顺利接管资源vip}

重新恢复node1

```
[root@node1 ~]# service heartbeat start

[root@node2 heartbeat]# crm_mon -1
============
Last updated: Fri Jan 24 08:23:09 2014
Stack: Heartbeat
Current DC: node2 (43a4f083-c5d3-4c66-a387-b05d79b5dd89) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
1 Resources configured.
============

Online: [ node1 node2 ]

 vip (ocf::heartbeat:IPaddr2): Started node1

```

{node1顺利收回vip资源的接管权}

删除资源：

```
[root@node1 ~]# crm_resource -D -r vip -t primitive
[root@node1 ~]# crm_resource -L
NO resources configured

```

4.7 配置流复制

在node1/node2上配置postgresql.conf/pg_hba.conf：

```
postgresql.conf :
listen_addresses = '*'
port = 5432
wal_level = hot_standby
archive_mode = on
archive_command = 'test ! -f /opt/archivelog/%f && cp %p /opt/archivelog/%f'
max_wal_senders = 4
wal_keep_segments = 50
hot_standby = on

pg_hba.conf :
host    replication     postgres        192.168.2.0/24           trust

```

## 4.8 配置pacemaker

{关于pacemaker的配置可通过多种方式，如crmsh、hb_gui、pcs等，该实验使用crmsh配置}

启动node1的heartbeat，关闭node2的heartbeat。

编写crm配置脚本：

```
[root@node1 ~]# cat pgsql.crm
property \
    no-quorum-policy="ignore" \
    stonith-enabled="false" \
    crmd-transition-delay="0s"

rsc_defaults \
    resource-stickiness="INFINITY" \
    migration-threshold="1"

ms msPostgresql pgsql \
    meta \
        master-max="1" \
        master-node-max="1" \
        clone-max="2" \
        clone-node-max="1" \
        notify="true"

clone clnPingCheck pingCheck
group master-group \
      vip-master \
      vip-rep \
      meta \
          ordered="false"

primitive vip-master ocf:heartbeat:IPaddr2 \
    params \
        ip="192.168.100.163" \
        nic="eth0" \
        cidr_netmask="24" \
    op start   timeout="60s" interval="0s"  on-fail="stop" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="block"

primitive vip-rep ocf:heartbeat:IPaddr2 \
    params \
        ip="192.168.2.3" \
        nic="eth2" \
        cidr_netmask="24" \
    meta \
            migration-threshold="0" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="block"

primitive vip-slave ocf:heartbeat:IPaddr2 \
    params \
        ip="192.168.100.164" \
        nic="eth0" \
        cidr_netmask="24" \
    meta \
        resource-stickiness="1" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="block"

primitive pgsql ocf:heartbeat:pgsql \
    params \
        pgctl="/opt/pgsql/bin/pg_ctl" \
        psql="/opt/pgsql/bin/psql" \
        pgdata="/opt/pgsql/data/" \
        start_opt="-p 5432" \
        rep_mode="sync" \
        node_list="node1 node2" \
        restore_command="cp /opt/archivelog/%f %p" \
        primary_conninfo_opt="keepalives_idle=60 keepalives_interval=5 keepalives_count=5" \
        master_ip="192.168.2.3" \
        stop_escalate="0" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="7s" on-fail="restart" \
    op monitor timeout="60s" interval="2s"  on-fail="restart" role="Master" \
    op promote timeout="60s" interval="0s"  on-fail="restart" \
    op demote  timeout="60s" interval="0s"  on-fail="stop" \
    op stop    timeout="60s" interval="0s"  on-fail="block" \
    op notify  timeout="60s" interval="0s"

primitive pingCheck ocf:pacemaker:ping \
    params \
        name="default_ping_set" \
        host_list="192.168.100.1" \
        multiplier="100" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="ignore"

location rsc_location-1 vip-slave \
    rule  200: pgsql-status eq "HS:sync" \
    rule  100: pgsql-status eq "PRI" \
    rule  -inf: not_defined pgsql-status \
    rule  -inf: pgsql-status ne "HS:sync" and pgsql-status ne "PRI"

location rsc_location-2 msPostgresql \
    rule -inf: not_defined default_ping_set or default_ping_set lt 100

colocation rsc_colocation-1 inf: msPostgresql        clnPingCheck
colocation rsc_colocation-2 inf: master-group        msPostgresql:Master

order rsc_order-1 0: clnPingCheck          msPostgresql
order rsc_order-2 0: msPostgresql:promote  master-group:start   symmetrical=false
order rsc_order-3 0: msPostgresql:demote   master-group:stop    symmetrical=false

```

导入配置脚本：

```
[root@node1 ~]# crm configure load update pgsql.crm
WARNING: pgsql: specified timeout 60s for stop is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for start is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for notify is smaller than the advised 90
WARNING: pgsql: specified timeout 60s for demote is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for promote is smaller than the advised 120
WARNING: pingCheck: specified timeout 60s for start is smaller than the advised 90
WARNING: pingCheck: specified timeout 60s for stop is smaller than the advised 100

```

一段时间后查看ha状态：

```
[root@node1 ~]# crm_mon -Afr1
============
Last updated: Mon Jan 27 05:23:59 2014
Stack: Heartbeat
Current DC: node1 (30b7dc95-25c5-40d7-b1e4-7eaf2d5cdf07) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
4 Resources configured.
============

Online: [ node1 ]
OFFLINE: [ node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node1
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql
     Masters: [ node1 ]
     Stopped: [ pgsql:0 ]
 Clone Set: clnPingCheck
     Started: [ node1 ]
     Stopped: [ pingCheck:1 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql:1                   : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000003000078
    + pgsql-status                     : PRI       

Migration summary:
* Node node1:

```

注：刚启动时为slave，一段时间后自动切换为master。

待资源在node1上都正常运行后在node2上执行基础备份同步：

```
[postgres@node2 data]$ pg_basebackup -h 192.168.2.3 -U postgres -D /opt/pgsql/data/ -P

```

启动node2的heartbeat：

```
[root@node2 ~]# service heartbeat start

```

过一段时间后查看集群状态：

```
[root@node1 ~]# crm_mon -Afr1
============
Last updated: Mon Jan 27 05:27:22 2014
Stack: Heartbeat
Current DC: node1 (30b7dc95-25c5-40d7-b1e4-7eaf2d5cdf07) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
4 Resources configured.
============

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql:1                   : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000003000078
    + pgsql-status                     : PRI
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql:0                   : 100
    + pgsql-data-status                : STREAMING|SYNC
    + pgsql-status                     : HS:sync   

Migration summary:
* Node node2:
* Node node1:

```

{vip-slave资源已经由node1切换到了node2上，流复制状态正常}

# 五、测试

## 5.1 备节点失效

在node2上杀死postgres数据库进程，模拟备节点上数据库崩溃：

```
[root@node2 ~]# killall -9 postgres

```

查看此时集群状态：

```
[root@node1 ~]# crm_mon -Afr1
============
Last updated: Mon Jan 27 08:36:49 2014
Stack: Heartbeat
Current DC: node1 (30b7dc95-25c5-40d7-b1e4-7eaf2d5cdf07) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
4 Resources configured.
============

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node1
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql
     Masters: [ node1 ]
     Stopped: [ pgsql:1 ]
 Clone Set: clnPingCheck
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql:0                   : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000010000000
    + pgsql-status                     : PRI
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql:1                   : -INFINITY
    + pgsql-data-status                : DISCONNECT
    + pgsql-status                     : STOP      

Migration summary:
* Node node1:
* Node node2:
   pgsql:1: migration-threshold=1 fail-count=1

Failed actions:
    pgsql:1_monitor_7000 (node=node2, call=11, rc=7, status=complete): not running

```

{vip-slave资源已成功切换到了node1上}

重启node2上的heartbeat，数据库将重新伴随启动：

```
[root@node2 ~]# service heartbeat restart

```

过段时间后查看状态：

```
[root@node1 ~]# crm_mon -Afr1
============
Last updated: Mon Jan 27 08:39:16 2014
Stack: Heartbeat
Current DC: node1 (30b7dc95-25c5-40d7-b1e4-7eaf2d5cdf07) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
4 Resources configured.
============

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql:0                   : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000010000000
    + pgsql-status                     : PRI
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql:1                   : 100
    + pgsql-data-status                : STREAMING|SYNC
    + pgsql-status                     : HS:sync   

Migration summary:
* Node node1:
* Node node2:

```

{vip-slave又重新回到了nod2上，且流复制重新建立}

## 5.2 主节点失效切换

在node1上杀死postgres数据库进程，模拟备节点上数据库崩溃：

```
[root@node1 ~]# killall -9 postgres

```

等会查看集群状态：

```
[root@node2 ~]# crm_mon -Afr -1
============
Last updated: Mon Jan 27 08:43:03 2014
Stack: Heartbeat
Current DC: node1 (30b7dc95-25c5-40d7-b1e4-7eaf2d5cdf07) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
4 Resources configured.
============

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node2
     vip-rep (ocf::heartbeat:IPaddr2): Started node2
 Master/Slave Set: msPostgresql
     Masters: [ node2 ]
     Stopped: [ pgsql:0 ]
 Clone Set: clnPingCheck
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql:0                   : -INFINITY
    + pgsql-data-status                : DISCONNECT
    + pgsql-status                     : STOP
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql:1                   : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 00000000120000B0
    + pgsql-status                     : PRI       

Migration summary:
* Node node1:
   pgsql:0: migration-threshold=1 fail-count=1
* Node node2: 

Failed actions:
    pgsql:0_monitor_2000 (node=node1, call=25, rc=7, status=complete): not running

```

{vip-master/vip-rep都已成功切换到node2上，且node2已变为master，node2上pg数据库状态已切换为PRI}

## 5.3 主节点恢复

修复原主节点后将其恢复为当前备节点

在node1上执行一次基础同步：

```
[postgres@node1 data]$ pwd
/opt/pgsql/data
[postgres@node1 data]$ rm -rf *
[postgres@node1 data]$ pg_basebackup -h 192.168.2.3 -U postgres -D /opt/pgsql/data/ -P
19172/19172 kB (100%), 1/1 tablespace
NOTICE:  pg_stop_backup complete, all required WAL segments have been archived
[postgres@node1 data]$ ls
backup_label      base    pg_clog      pg_ident.conf  pg_notify  pg_stat_tmp  pg_tblspc    PG_VERSION  postgresql.conf
backup_label.old  global  pg_hba.conf  pg_multixact   pg_serial  pg_subtrans  pg_twophase  pg_xlog     recovery.done

```

启动heartbeat之前必须删除资锁，不然资源将不会伴随heartbeat启动：

```
[root@node1 ~]# rm -rf /var/lib/pgsql/tmp/PGSQL.lock

```

{该锁文件在当节点为主节点时创建，但不会因为heartbeat的异常停止或数据库/系统的异常终止而自动删除，所以在恢复一个节点的时候只要该节点充当过主节点就需要手动清理该锁文件}

重启node1上的heartbeat：

```
[root@node1 ~]# service heartbeat restart

```

过段时间后查看集群状态：

```
[root@node2 ~]# crm_mon -Afr1
============
Last updated: Mon Jan 27 08:50:43 2014
Stack: Heartbeat
Current DC: node2 (f2dcd1df-7429-42f5-82e9-b73921f97cab) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
4 Resources configured.
============

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node1
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node2
     vip-rep (ocf::heartbeat:IPaddr2): Started node2
 Master/Slave Set: msPostgresql
     Masters: [ node2 ]
     Slaves: [ node1 ]
 Clone Set: clnPingCheck
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql:0                   : 100
    + pgsql-data-status                : STREAMING|SYNC
    + pgsql-status                     : HS:sync
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql:1                   : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 00000000120000B0
    + pgsql-status                     : PRI       

Migration summary:
* Node node1:
* Node node2:

```

{vip-slave已成功切到node1上，node1成功成为流复制备节点}

# 六、管理

## 6.1 启动关闭heartbeat

```
[root@node1 ~]# service heartbeat start
[root@node1 ~]# service heartbeat stop

```

## 6.2 查看HA状态

```
[root@node1 ~]# crm status

```

## 6.3 查看资源状态及节点属性

```
[root@node1 ~]# crm_mon -Afr -1

```

## 6.4 查看配置

```
[root@node1 ~]# crm configure show

```

## 6.5 实时监控HA

```
[root@node1 ~]# crm_mon -Afr

```

## 6.6 crm_resource命令

### 资源启动/关闭：

```
[root@node1 ~]# crm_resource -r vip-master -v started
[root@node1 ~]# crm_resource -r vip-master -v stoped

```

### 列举资源：

```
[root@node1 ~]# crm_resource -L
 vip-slave (ocf::heartbeat:IPaddr2): Started
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started
     vip-rep (ocf::heartbeat:IPaddr2): Started
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

```

### 查看资源位置：

```
[root@node1 ~]# crm_resource -W -r pgsql
resource pgsql is running on: node2

```

### 迁移资源：

```
[root@node1 ~]# crm_resource -M -r vip-slave -N node2

```

### 删除资源：

```
[root@node1 ~]# crm_resource -D -r vip-slave -t primitive

```

## 6.7 crm命令

### 列举指定的RA：

```
[root@node1 ~]# crm ra list ocf pacemaker
ClusterMon     Dummy          HealthCPU      HealthSMART    Stateful       SysInfo        SystemHealth   controld       ping           pingd
remote

```

### 删除节点：

```
[root@node1 ~]# crm node delete node2

```

### 停用节点：

```
[root@node1 ~]# crm node standby node2

```

### 启用节点：

```
[root@node1 ~]# crm node online node2

```

### 配置pacemaker：

```
[root@node1 ~]# crm configure
crm(live)configure#
……
……
crm(live)configure# commit
crm(live)configure# quit

```

## 6.8 重置failcount

```
[root@node1 ~]# crm resource
crm(live)resource# failcount pgsql set node1 0
crm(live)resource# failcount pgsql show node1
scope=status  name=fail-count-pgsql value=0

```

```
[root@node1 ~]# crm resource cleanup pgsql
Cleaning up pgsql:0 on node1
Waiting for 1 replies from the CRMd. OK

```

```
[root@node1 ~]# crm_failcount -G -U node1 -r pgsql
scope=status  name=fail-count-pgsql value=INFINITY
[root@node1 ~]# crm_failcount -D -U node1 -r pgsql

```

# 七、问题记录

## 7.1 Q1

问题现象：

heartbeat日志中报如下错误：

Jan 24 07:47:36 node1 heartbeat: [2515]: WARN: nodename node1 uuid changed to node2

Jan 24 07:47:38 node1 heartbeat: [2515]: WARN: nodename node2 uuid changed to node1

Jan 24 07:47:38 node1 heartbeat: [2515]: WARN: nodename node1 uuid changed to node2

Jan 24 07:47:40 node1 heartbeat: [2515]: WARN: nodename node2 uuid changed to node1

Jan 24 07:47:40 node1 heartbeat: [2515]: WARN: nodename node1 uuid changed to node2

Jan 24 07:47:42 node1 heartbeat: [2515]: WARN: nodename node2 uuid changed to node1

Jan 24 07:47:42 node1 heartbeat: [2515]: WARN: nodename node1 uuid changed to node2

解决方式：

因为是通过虚拟机克隆生成的node2,所以hb_uuid相同，需要删除后重新生成，如下：

```
[root@node2 ~]# rm -rf /var/lib/heartbeat/hb_uuid
[root@node2 ~]# service heartbeat restart

```

重启之后将会生成新的hb_uuid

## 7.2 Q2

问题现象：

加载配置时报错：

```
[root@node1 ~]# crm configure load update pgsql.crm
ERROR: pgsql: parameter rep_mode does not exist
ERROR: pgsql: parameter node_list does not exist
ERROR: pgsql: parameter master_ip does not exist
ERROR: pgsql: parameter restore_command does not exist
ERROR: pgsql: parameter primary_conninfo_opt does not exist
WARNING: pgsql: specified timeout 60s for stop is smaller than the advised 120
WARNING: pgsql: action monitor_Master not advertised in meta-data, it may not be supported by the RA
WARNING: pgsql: specified timeout 60s for start is smaller than the advised 120
WARNING: pgsql: action notify not advertised in meta-data, it may not be supported by the RA
WARNING: pgsql: action demote not advertised in meta-data, it may not be supported by the RA
WARNING: pgsql: action promote not advertised in meta-data, it may not be supported by the RA
WARNING: pingCheck: specified timeout 60s for start is smaller than the advised 90
WARNING: pingCheck: specified timeout 60s for stop is smaller than the advised 100
Do you still want to commit?

```

解决方式：

原因是pgsql脚本过旧，不支持配置pgsql.crm中设置的一些参数，需要从网上下载并替换pgsql

https://raw.github.com/ClusterLabs/resource-agents

## 7.3 Q3

问题现象：

加载配置时报错：

```
[root@node1 ~]# crm configure load update pgsql.crm
lrmadmin[15368]: 2014/01/24_09:18:44 ERROR: lrm_get_rsc_type_metadata(578): got a return code HA_FAIL from a reply message of rmetadata with function get_ret_from_msg.
ERROR: ocf:heartbeat:pgsql: could not parse meta-data:
ERROR: ocf:heartbeat:pgsql: could not parse meta-data:
ERROR: ocf:heartbeat:pgsql: no such resource agent
WARNING: pingCheck: specified timeout 60s for start is smaller than the advised 90
WARNING: pingCheck: specified timeout 60s for stop is smaller than the advised 100
Do you still want to commit?

```

解决方式：

原因是pgsql脚本权限不正确，使用下面命令修改即可：

\# chmod 755 /usr/lib/ocf/resource.d/heartbeat/pgsql

## 7.4 Q4

问题现象：

启动heartbeat时报错：

```
[root@node1 ~]# service heartbeat start
/usr/lib/ocf/lib//heartbeat/ocf-shellfuncs: line 56: @OCF_ROOT_DIR@/lib/heartbeat/ocf-binaries: No such file or directory

```

解决方式：

因为在CentOS5.5中@OCF_ROOT_DIR@变量无法替换为正确的路径导致，可通过修改脚本实现，如下：

编辑ocf-shellfuncs修改如下内容：

if [ -z "$OCF_ROOT" ]; then

\# : ${OCF_ROOT=@OCF_ROOT_DIR@}

: ${OCF_ROOT=/usr/lib/ocf}

fi

## 7.5 Q5

问题现象：

启动heartbeat时报错：

```
# service heartbeat start
/usr/lib/ocf/lib//heartbeat/ocf-shellfuncs: line 60: /usr/lib/ocf/lib/heartbeat/ocf-rarun: No such file or directory

```

解决方式：

因为缺少ocf-rarun脚本导致

下载放入相应路径即可：

下载地址https://raw.github.com/ClusterLabs/resource-agents

## 7.6 Q6

问题现象：

启动heartbeat时因找不到启动脚本而报错：

```
[root@db1 ~]# service heartbeat start
Starting High-Availability services:  Heartbeat failure [rc=6]. Failed.

heartbeat[2074]: 2014/01/23_09:06:59 info: Pacemaker support: yes
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Client child command [/usr/lib64/heartbeat/cib] is not executable
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Directive failfast  hacluster /usr/lib64/heartbeat/cib failed
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Client child command [/usr/lib64/heartbeat/stonithd] is not executable
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Directive respawn root /usr/lib64/heartbeat/stonithd failed
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Client child command [/usr/lib64/heartbeat/attrd] is not executable
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Directive respawn  hacluster /usr/lib64/heartbeat/attrd failed
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Client child command [/usr/lib64/heartbeat/crmd] is not executable
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Directive failfast  hacluster /usr/lib64/heartbeat/crmd failed
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Heartbeat not started: configuration error.
heartbeat[2074]: 2014/01/23_09:06:59 ERROR: Configuration error, heartbeat not started.

```

解决方式：

```
ln -s /usr/libexec/pacemaker/cib /usr/lib64/heartbeat/cib
ln -s /usr/libexec/pacemaker/stonithd /usr/lib64/heartbeat/stonithd
ln -s /usr/libexec/pacemaker/attrd /usr/lib64/heartbeat/attrd
ln -s /usr/libexec/pacemaker/crmd /usr/lib64/heartbeat/crmd

```

7.7 Q7

问题现象：

启动heartbeat时报错：

Jan 23 09:10:15 db1 heartbeat: [2129]: info: Heartbeat generation: 1390439416

Jan 23 09:10:15 db1 heartbeat: [2129]: info: No uuid found for current node - generating a new uuid.

Jan 23 09:10:15 db1 heartbeat: [2129]: info: Creating FIFO /var/lib/heartbeat/fifo.

Jan 23 09:10:15 db1 heartbeat: [2129]: info: glib: ucast: write socket priority set to IPTOS_LOWDELAY on eth1

Jan 23 09:10:15 db1 heartbeat: [2129]: info: glib: ucast: bound send socket to device: eth1

Jan 23 09:10:15 db1 heartbeat: [2129]: ERROR: glib: ucast: error setting option SO_REUSEPORT(w): Protocol not available

Jan 23 09:10:15 db1 heartbeat: [2129]: ERROR: make_io_childpair: cannot open ucast eth1

Jan 23 09:10:16 db1 heartbeat: [2132]: CRIT: Emergency Shutdown: Master Control process died.

Jan 23 09:10:16 db1 heartbeat: [2132]: CRIT: Killing pid 2129 with SIGTERM

Jan 23 09:10:16 db1 heartbeat: [2132]: CRIT: Emergency Shutdown(MCP dead): Killing ourselves.

解决方式：

1.升级内核版本，当前内核版本不支持ucast;

2.换用其它的检测方式，如mcast/bcast。

## 7.8 Q8

问题现象：

使用bcast心跳检测方式时报错：

Jan 24 01:30:20 db2 heartbeat: [29856]: ERROR: glib: Error binding socket (Address already in use). Retrying.

Jan 24 01:30:21 db2 heartbeat: [29856]: ERROR: glib: Error binding socket (Address already in use). Retrying.

Jan 24 01:30:22 db2 heartbeat: [29856]: ERROR: glib: Error binding socket (Address already in use). Retrying.

Jan 24 01:30:23 db2 heartbeat: [29856]: ERROR: glib: Error binding socket (Address already in use). Retrying.

Jan 24 01:30:24 db2 heartbeat: [29856]: ERROR: glib: Unable to bind socket (Address already in use). Giving up.

Jan 24 01:30:24 db2 heartbeat: [29856]: info: glib: UDP Broadcast heartbeat closed on port 694 interface eth1 - Status: 1

Jan 24 01:30:24 db2 heartbeat: [29856]: ERROR: make_io_childpair: cannot open bcast eth1

Jan 24 01:30:25 db2 heartbeat: [29859]: CRIT: Emergency Shutdown: Master Control process died.

Jan 24 01:30:25 db2 heartbeat: [29859]: CRIT: Killing pid 29856 with SIGTERM

Jan 24 01:30:25 db2 heartbeat: [29859]: CRIT: Emergency Shutdown(MCP dead): Killing ourselves.

解决方式：

说明694端口已经被占用，查看

```
[root@db1 ~]# netstat -nlp | grep 694
udp        0      0 0.0.0.0:694                 0.0.0.0:*                               1367/rpcbind
udp        0      0 :::694                      :::*                                    1367/rpcbind

```

换个UDP端口，如在ha.cf中指定udpport 692

# 八、参考资源

脚本：

[https://github.com/ClusterLabs/resource-agents/blob/master/heartbeat/pgsql]()�

脚本使用说明：

[https://github.com/t-matsuo/resource-agents/wiki/Resource-Agent-for-PostgreSQL-9.1-streaming-replication]()�

crm_resouce命令：

<http://www.novell.com/zh-cn/documentation/sle_ha/book_sleha/data/man_crmresource.html>�

crm_failcount命令：

<http://www.novell.com/zh-cn/documentation/sle_ha/book_sleha/data/man_crmfailcount.html>�