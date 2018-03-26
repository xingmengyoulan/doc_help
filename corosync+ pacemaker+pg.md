# corosync+ pacemaker实现pg流复制自动切换

corosync + pacemaker + postgres_streaming_replication

说明：

该文档用于说明以corosync+pacemaker的方式实现PostgreSQL流复制自动切换。注意内容包括有关corosync/pacemaker知识总结以及整个环境的搭建过程和问题处理。

```shell
删除 所有配置 --- /var/lib/pacemaker/cib
单播配置：cd /etc/corosync/corosync.conf
        interface {
                member {
                     memberaddr:192.168.67.134
                }

                 member {
                     memberaddr:192.168.67.135 
                }
                ringnumber: 0
                bindnetaddr: 192.168.67.0  （网段）
                mcastport: 5405
                ttl: 1
        }
 在推荐的网络接口配置中有几件事需要注意：
所有心跳网络的 ringnumber 配置不能重复，最小值为 0 。
bindnetaddr 是心跳网卡 IP 地址对应的网络地址。示例中使用了两个子网掩码为 /24 的 IPv4 网段。
Corosync 通信使用 UDP 协议，端口为 mcastport （接收数据）和 mcastport - 1 （发送数据）。配置防火墙时需要打开这两个端口。
pgsql.crm--------内容
primitive pingCheck ocf:pacemaker:ping \
   params \
        name="default_ping_set" \
        host_list="192.168.67.1" \ (使用网关地址，必须是ping通)
        multiplier="100" \
        
[postgres@node2 ~]$ crm_mon -Afr -1 ---（错误：权限不足，必须是root用户）
Could not establish cib_ro connection: No such file or directory (2)

Connection to cluster failed: Transport endpoint is not connected


VIP 192.168.67.130 必须在一个网段

```



# 一、介绍

Corosync

Corosync是由OpenAIS项目分离独立出来的项目，分离后能实现HA信息传输功能的就成为了Corosync，因此Corosync 60%的代码来源于OpenAIS。

Corosync与分离出来的Heartbeat类似，都属于集群信息层，负责传送集群信息以及节点间的心跳信息，单纯HA软件都不存在管理资源的功能，而是需要依赖上层的CRM来管理资源。目前最著名的资源管理器为Pacemaker，Corosync+Pacemaker也成为高可用集群方案中的最佳组合。

Pacemaker

Pacemaker，即Cluster Resource Manager（CRM），管理整个HA，客户端通过pacemaker管理监控整个集群。

常用的集群管理工具：

（1）基于命令行

crm shell/pcs

（2）基于图形化

pygui/hawk/lcmc/pcs

Pacemaker内部组件、模块关系图：

[![corosync+ pacemaker实现pg流复制自动切换](http://image.codeweblog.com/upload/7/5c/75c7d2ac33b21262_thumb.png)](http://image.codeweblog.com/upload/7/5c/75c7d2ac33b21262.png)

# 二、环境

## 2.1 OS

```
# cat /etc/issue
CentOS release 6.4 (Final)
Kernel \r on an \m

# uname -a
Linux node1 2.6.32-358.el6.x86_64 #1 SMP Fri Feb 22 00:31:26 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

```

## 2.2 IP

node1:

eth0 192.168.100.201/24 GW 192.168.100.1 ---真实地址

eth1 10.10.10.1/24 ---心跳地址

eth2 192.168.1.1/24 ---流复制地址

node2:

eth0 192.168.100.202/24 GW 192.168.100.1 ---真实地址

eth1 10.10.10.2/24 ---心跳地址

eth2 192.168.1.2/24 ---流复制地址

虚拟地址：

eth0:0 192.168.100.213/24 ---vip-master

eth0:0 192.168.100.214/24 ---vip-slave

eth2:0 192.168.1.3/24 ---vip-rep

## 2.3 软件版本

```
# rpm -qa | grep corosync
corosync-1.4.5-2.3.x86_64
corosynclib-1.4.5-2.3.x86_64

# rpm -qa | grep pacemaker
pacemaker-libs-1.1.10-14.el6_5.2.x86_64
pacemaker-cli-1.1.10-14.el6_5.2.x86_64
pacemaker-1.1.10-14.el6_5.2.x86_64
pacemaker-cluster-libs-1.1.10-14.el6_5.2.x86_64

# rpm -qa | grep crmsh
crmsh-1.2.6-6.1.x86_64

```

PostgreSQL Version：9.1.4

# 三、安装

## 3.1 设置YUM源

```
# cat /etc/yum.repos.d/ha-clustering.repo
[haclustering]
name=HA Clustering
baseurl=http://download.opensuse.org/repositories/network:/ha-clustering:/Stable/CentOS_CentOS-6/
enabled=1
gpgcheck=0

```

## 3.2 安装pacemaker/corosync/crmsh

```
# yum install pacemaker corosync crmsh

```

安装后会在/usr/lib/ocf/resource.d下生成相应的ocf资源脚本，如下：

```
# cd /usr/lib/ocf/resource.d/
[root@node1 resource.d]# ls
heartbeat  pacemaker  redhat

```

通过命令查看资源脚本：

```
[root@node1 resource.d]# crm ra list ocf
ASEHAagent.sh        AoEtarget            AudibleAlarm         CTDB                 ClusterMon           Delay                Dummy
EvmsSCC              Evmsd                Filesystem           HealthCPU            HealthSMART          ICP                  IPaddr
IPaddr2              IPsrcaddr            IPv6addr             LVM                  LinuxSCSI            MailTo               ManageRAID
ManageVE             Pure-FTPd            Raid1                Route                SAPDatabase          SAPInstance          SendArp
ServeRAID            SphinxSearchDaemon   Squid                Stateful             SysInfo              SystemHealth         VIPArip
VirtualDomain        WAS                  WAS6                 WinPopup             Xen                  Xinetd               anything
apache               apache.sh            asterisk             clusterfs.sh         conntrackd           controld             db2
dhcpd                drbd                 drbd.sh              eDir88               ethmonitor           exportfs             fio
fs.sh                iSCSILogicalUnit     iSCSITarget          ids                  ip.sh                iscsi                jboss
ldirectord           lvm.sh               lvm_by_lv.sh         lvm_by_vg.sh         lxc                  mysql                mysql-proxy
mysql.sh             named                named.sh             netfs.sh             nfsclient.sh         nfsexport.sh         nfsserver
nfsserver.sh         nginx                ocf-shellfuncs       openldap.sh          oracle               oracledb.sh          orainstance.sh
oralistener.sh       oralsnr              pgsql                ping                 pingd                portblock            postfix
postgres-8.sh        pound                proftpd              remote               rsyncd               rsyslog              samba.sh
script.sh            scsi2reservation     service.sh           sfex                 slapd                smb.sh               svclib_nfslock
symlink              syslog-ng            tomcat               tomcat-5.sh          tomcat-6.sh          varnish              vm.sh
vmware               zabbixserver

```

启动corosync：

```
[root@node1 ~]# service corosync start
Starting Corosync Cluster Engine (corosync):               [  OK  ]
[root@node2 ~]# service corosync start
Starting Corosync Cluster Engine (corosync):               [  OK  ]
[root@node2 ~]# crm status
Last updated: Sat Jan 18 07:00:34 2014
Last change: Sat Jan 18 06:58:11 2014 via crmd on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
0 Resources configured

Online: [ node1 node2 ]

```

若出现以下错误可先禁止掉stonith，该错误是因为stonith未配置导致，错误如下：

crm_verify[4921]: 2014/01/10_07:34:34 ERROR: unpack_resources: Resource start-up disabled since no STONITH resources have been defined

crm_verify[4921]: 2014/01/10_07:34:34 ERROR: unpack_resources: Either configure some or disable STONITH with the stonith-enabled option

crm_verify[4921]: 2014/01/10_07:34:34 ERROR: unpack_resources: NOTE: Clusters with shared data need STONITH to ensure data integrity

禁止stonith（只在一个节点上执行即可）：

```
[root@node1 ~]# crm configure property stonith-enabled=false

```

## 3.3 安装PostgreSQL

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
192.168.100.201 node1
192.168.100.202 node2

```

## 4.2 配置corosync

```json
[root@node1 ~]# cd /etc/corosync/
[root@node1 corosync]# ls
corosync.conf.example  corosync.conf.example.udpu  service.d  uidgid.d
[root@node1 corosync]# cp corosync.conf.example corosync.conf
[root@node1 corosync]# vim corosync.conf
compatibility: whitetank //兼容旧版本

totem { //节点间通信协议定义
 version: 2
 secauth: on //是否开启安全认证
 threads: 0
 interface { //心跳配置
 ringnumber: 0
 bindnetaddr: 10.10.10.0 //绑定网络
 mcastaddr: 226.94.1.1 //向外发送多播的地址
 mcastport: 5405 //多播端口
 ttl: 1
 }
}

logging { //日志设置
 fileline: off
 to_stderr: no //是否发送错误信息到标准输出
 to_logfile: yes //是否记录到日志文件
 to_syslog: yes //是否记录到系统日志
 logfile: /var/log/cluster/corosync.log //日志文件，注意/var/log/cluster目录必须存在
 debug: off
 timestamp: on //日志中是否标记时间
 logger_subsys {
 subsys: AMF
 debug: off
 }
}

amf {
 mode: disabled
}

service {
        ver: 0
        name: pacemaker //启用pacemaker
}  

aisexec {
        user:   root
        group:  root
}
```

## 4.3 生成密钥

{默认利用random生成，但如果中断的系统随机数不够用就需要较长的时间，此时可以通过urandom来替代random}

```shell
[root@node1 corosync]# mv /dev/random /dev/random.bak
[root@node1 corosync]# ln -s /dev/urandom /dev/random
[root@node1 corosync]# corosync-keygen
Corosync Cluster Engine Authentication key generator.
Gathering 1024 bits for key from /dev/random.
Press keys on your keyboard to generate entropy.
Writing corosync key to /etc/corosync/authkey.
```

## 4.4 SSH互信配置

node1 -> node2 :

```shell
[root@node1 ~]# cd .ssh/
[root@node1 .ssh]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
2c:ed:1e:a6:a7:cd:e3:b2:7c:de:aa:ff:63:28:9a:19 root@node1
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|       o         |
|      . S        |
|       o         |
|    E   +.       |
|     =o*=oo      |
|    +.*%O=o.     |
+-----------------+
[root@node1 .ssh]# ssh-copy-id -i id_rsa.pub node2
The authenticity of host 'node2 (192.168.100.202)' can't be established.
RSA key fingerprint is be:76:cd:29:af:59:76:11:6a:c7:7d:72:27:df:d1:02.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node2,192.168.100.202' (RSA) to the list of known hosts.
root@node2's password:
Now try logging into the machine, with "ssh 'node2'", and check in:

  .ssh/authorized_keys

to make sure we haven't added extra keys that you weren't expecting.
[root@node1 .ssh]# ssh node2 date
Sat Jan 18 06:36:21 CST 2014
```

node2 -> node1 :

```shell
[root@node2 ~]# cd .ssh/
[root@node2 .ssh]# ssh-keygen -t rsa
[root@node2 .ssh]# ssh-copy-id -i id_rsa.pub node1
[root@node2 .ssh]# ssh node1 date
Sat Jan 18 06:37:31 CST 2014		
```

## 4.5 同步配置

```shell
[root@node1 corosync]# scp authkey corosync.conf node2:/etc/corosync/
authkey         100%  128     0.1KB/s   00:00
corosync.conf   100% 2808     2.7KB/s   00:00
```

## 4.6 下载替换脚本

虽然安装了上述软件后会生成pgsql资源脚本，但是其版本过旧，且自带的pgsql不能实现自动切换功能，所以在安装了pacemaker/corosync之后需要从网上下载进行替换，如下：

[https://github.com/ClusterLabs/resource-agents/tree/master/heartbeat]()�

下载pgsql与ocf-shellfuncs.in

替换：

```shell
# cp pgsql /usr/lib/ocf/resource.d/heartbeat/
# cp ocf-shellfuncs.in /usr/lib/ocf/lib/heartbeat/ocf-shellfuncs
```

{注意要将ocf-shellfuncs.in名称改为ocf-shellfuncs，否则pgsql可能会找不到要用的函数。新下载的函数定义文件中添加了一些新功能函数，如ocf_local_nodename等}

pgsql资源脚本特性：

●主节点失效切换

master宕掉时，RA检测到该问题并将master标记为stop，随后将slave提升为新的master。

●异步与同步切换

如果slave宕掉或者LAN中存在问题，那么当设置为同步复制时包含写操作的事务将会被终止，也就意味着服务将停止。因此，为防止服务停止RA将会动态地将同步转换为异步复制。

●初始启动时自动识别新旧数据

当两个或多个节点上的Pacemaker同时初始启动时，RA通过每个节点上最近的replay location进行比较，找出最新数据节点。这个拥有最新数据的节点将被认为是master。当然，若在一个节点上启动pacemaker或者该节点上的pacemaker是第一个被启动的，那么它也将成为master。RA依据停止前的数据状态进行裁定。

●读负载均衡

由于slave节点可以处理只读事务，因此对于读操作可以通过虚拟另一个虚拟IP来实现读操作的负载均衡。

## 4.7 启动corosync

启动：

```shell
[root@node1 ~]# service corosync start
[root@node2 ~]# service corosync start
```

检测状态：

```shell
[root@node1 ~]# crm status
Last updated: Tue Jan 21 23:55:13 2014
Last change: Tue Jan 21 23:37:36 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
0 Resources configured

Online: [ node1 node2 ]
```

{corosync启动成功}

## 4.8 配置流复制

在node1/node2上配置postgresql.conf/pg_hba.conf：

postgresql.conf :

listen_addresses = '*'

port = 5432

wal_level = hot_standby

archive_mode = on

archive_command = 'test ! -f /opt/archivelog/%f && cp %p /opt/archivelog/%f'

max_wal_senders = 4

wal_keep_segments = 50

hot_standby = on

​	pg_hba.conf :

host replication postgres 192.168.1.0/24 trust

在node2上执行基础同步：

```shell
[postgres@node2 data]$ pg_basebackup -h 192.168.1.1 -U postgres -D /opt/pgsql/data -P
```

若需测试流复制是否能够成功，可在此处手工配置（corosync启动数据库时自动生成，若已经存在将会被覆盖）recovery.conf进行测试：

standby_mode = 'on'

primary_conninfo = 'host=192.168.1.1 port=5432 user=postgres application_name=node2 keepalives_idle=60 keepalives_interval=5 keepalives_count=5'

restore_command = 'cp /opt/archivelog/%f %p'

recovery_target_timeline = 'latest'

```shell
[postgres@node2 data]$ pg_ctl start
[postgres@node1 pgsql]$ psql
postgres=# select client_addr,sync_state from pg_stat_replication;
 client_addr | sync_state
-------------+------------
 192.168.1.2 | sync
(1 row)
```

停止数据库：

```shell
[postgres@node2 ~]$ pg_ctl stop -m f
[postgres@node1 ~]$ pg_ctl stop -m f
```

## 4.9 配置pacemaker

{关于pacemaker的配置可通过多种方式，如crmsh、hb_gui、pcs等，该实验使用crmsh配置}

编写crm配置脚本：

```sql
[root@node1 ~]# cat pgsql.crm
property \ //设置全局属性
    no-quorum-policy="ignore" \ //关闭法定投票人数策略，多节点时启用
    stonith-enabled="false" \ //禁用stonith设备检测
    crmd-transition-delay="0s"

rsc_defaults \ //资源默认属性配置
    resource-stickiness="INFINITY" \ //资源留在所处位置的自愿程度，INFINITY为无限自愿
    migration-threshold="1" //设置资源发生多少次故障时节点将失去管理该资源的资格

ms msPostgresql pgsql \ //
    meta \
        master-max="1" \
        master-node-max="1" \
        clone-max="2" \
        clone-node-max="1" \
        notify="true"

clone clnPingCheck pingCheck //克隆资源
group master-group \ //定义资源组
      vip-master \
      vip-rep

primitive vip-master ocf:heartbeat:IPaddr2 \ //定义vip-master资源
    params \
        ip="192.168.100.213" \
        nic="eth0" \
        cidr_netmask="24" \
    op start   timeout="60s" interval="0s"  on-fail="stop" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="block"

  primitive vip-rep ocf:heartbeat:IPaddr2 \ //定义vip-rep资源
    params \
        ip="192.168.1.3" \
        nic="eth2" \
        cidr_netmask="24" \
    meta \
            migration-threshold="0" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="block"

primitive vip-slave ocf:heartbeat:IPaddr2 \ //定义vip-slave资源
    params \
        ip="192.168.100.214" \
        nic="eth0" \
        cidr_netmask="24" \
    meta \
        resource-stickiness="1" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="block"

primitive pgsql ocf:heartbeat:pgsql \ //定义pgsql资源
    params \ //设置相关参数
        pgctl="/opt/pgsql/bin/pg_ctl" \
        psql="/opt/pgsql/bin/psql" \
        pgdata="/opt/pgsql/data/" \
        start_opt="-p 5432" \
        rep_mode="sync" \
        node_list="node1 node2" \
        restore_command="cp /opt/archivelog/%f %p" \
        primary_conninfo_opt="keepalives_idle=60 keepalives_interval=5 keepalives_count=5" \
        master_ip="192.168.1.3" \
        stop_escalate="0" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="7s" on-fail="restart" \
    op monitor timeout="60s" interval="2s"  on-fail="restart" role="Master" \
    op promote timeout="60s" interval="0s"  on-fail="restart" \
    op demote  timeout="60s" interval="0s"  on-fail="stop" \
    op stop    timeout="60s" interval="0s"  on-fail="block" \
    op notify  timeout="60s" interval="0s"

primitive pingCheck ocf:pacemaker:ping \ //定义pingCheck资源
    params \
        name="default_ping_set" \
        host_list="192.168.100.1" \
        multiplier="100" \
    op start   timeout="60s" interval="0s"  on-fail="restart" \
    op monitor timeout="60s" interval="10s" on-fail="restart" \
    op stop    timeout="60s" interval="0s"  on-fail="ignore"

location rsc_location-1 vip-slave \ //定义资源vip-slave选择位置
    rule  200: pgsql-status eq "HS:sync" \
    rule  100: pgsql-status eq "PRI" \
    rule  -inf: not_defined pgsql-status \
    rule  -inf: pgsql-status ne "HS:sync" and pgsql-status ne "PRI"

location rsc_location-2 msPostgresql \ //定义资源msPostgresql选择位置
    rule -inf: not_defined default_ping_set or default_ping_set lt 100

colocation rsc_colocation-1 inf: msPostgresql        clnPingCheck //定义在相同节点上运行的资源
colocation rsc_colocation-2 inf: master-group        msPostgresql:Master

order rsc_order-1 0: clnPingCheck          msPostgresql //定义对资源的操作顺序
order rsc_order-2 0: msPostgresql:promote  master-group:start   symmetrical=false
order rsc_order-3 0: msPostgresql:demote   master-group:stop    symmetrical=false

```

注：该脚本针对网上的配置方式做了一点修改，因为网上是针对pacemaker-1.0.*进行配置的，而本实验使用的是pacemaker-1.1.10。

导入配置脚本：

```shell
[root@node1 ~]# crm configure load update pgsql.crm
WARNING: pgsql: specified timeout 60s for stop is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for start is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for notify is smaller than the advised 90
WARNING: pgsql: specified timeout 60s for demote is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for promote is smaller than the advised 120

```

一段时间后查看ha状态：

```
[root@node1 ~]# crm status
Last updated: Tue Jan 21 23:55:13 2014
Last change: Tue Jan 21 23:37:36 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

[root@node1 ~]# crm_mon -Afr -1
Last updated: Tue Jan 21 23:37:20 2014
Last change: Tue Jan 21 23:37:36 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql                     : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000006000078
    + pgsql-status                     : PRI
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql                     : 100
    + pgsql-data-status                : STREAMING|SYNC
    + pgsql-status                     : HS:sync   

Migration summary:
* Node node2:
* Node node1:

```

注：刚启动时两节点均为slave，一段时间后node1自动切换为master。

# 五、测试

## 5.1 备节点失效

在node2上杀死postgres数据库进程，模拟备节点上数据库崩溃：

```shell
[root@node2 ~]# killall -9 postgres
```

查看此时集群状态：

```shell
[root@node1 ~]# crm_mon -Afr -1
Last updated: Wed Jan 22 02:15:06 2014
Last change: Wed Jan 22 02:15:33 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node1
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Stopped: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql                     : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000006000078
    + pgsql-status                     : PRI
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql                     : -INFINITY
    + pgsql-data-status                : DISCONNECT
    + pgsql-status                     : STOP      

Migration summary:
* Node node2:
   pgsql: migration-threshold=1 fail-count=1 last-failure='Wed Jan 22 02:15:35 2014'
* Node node1: 

Failed actions:
    pgsql_monitor_7000 on node2 'not running' (7): call=42, status=complete, last-rc-change='Wed Jan 22 02:14:58 2014', queued=0ms, exec=0ms
```

{vip-slave资源已成功切换到了node1上}

重启node2上的corosync，数据库将重新伴随启动：

```shell
[root@node2 ~]# service corosync restart
[root@node1 ~]# crm_mon -Afr -1
Last updated: Wed Jan 22 02:16:24 2014
Last change: Wed Jan 22 02:16:55 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql                     : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000006000078
    + pgsql-status                     : PRI
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql                     : 100
    + pgsql-data-status                : STREAMING|SYNC
    + pgsql-status                     : HS:sync   

Migration summary:
* Node node2:
* Node node1:
```

{vip-slave又重新回到了nod2上}

## 5.2 主节点失效切换

在node1上杀死postgres数据库进程，模拟主节点上数据库崩溃：

```
[root@node1 ~]# killall -9 postgres

```

等会查看集群状态：

```shell
[root@node2 ~]# crm_mon -Afr -1
Last updated: Wed Jan 22 02:17:50 2014
Last change: Wed Jan 22 02:18:16 2014 via crm_attribute on node2
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node2
     vip-rep (ocf::heartbeat:IPaddr2): Started node2
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node2 ]
     Stopped: [ node1 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql                     : -INFINITY
    + pgsql-data-status                : DISCONNECT
    + pgsql-status                     : STOP
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql                     : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000008014A70
    + pgsql-status                     : PRI       

Migration summary:
* Node node2:
* Node node1:
   pgsql: migration-threshold=1 fail-count=1 last-failure='Wed Jan 22 02:18:11 2014'

Failed actions:
    pgsql_monitor_2000 on node1 'not running' (7): call=2435, status=complete, last-rc-change='Wed Jan 22 02:18:11 2014', queued=0ms, exec=0ms
```

{vip-master/vip-rep都已成功切换到node2上，且node2已变为master，node2上pg数据库状态已切换为PRI}

停止node1上的corosync：

```shell
[root@node1 ~]# service corosync stop
```

执行一次基础同步：

```shell
[postgres@node1 data]$ pwd
/opt/pgsql/data
[postgres@node1 data]$ rm -rf *
[postgres@node1 data]$ pg_basebackup -h 192.168.1.3 -U postgres -D /opt/pgsql/data/ -P
19172/19172 kB (100%), 1/1 tablespace
NOTICE:  pg_stop_backup complete, all required WAL segments have been archived
[postgres@node1 data]$ ls
backup_label      base    pg_clog      pg_ident.conf  pg_notify  pg_stat_tmp  pg_tblspc    PG_VERSION  postgresql.conf
backup_label.old  global  pg_hba.conf  pg_multixact   pg_serial  pg_subtrans  pg_twophase  pg_xlog     recovery.done

```

启动node1上的corosync：

```powershell
[root@node1 ~]# service corosync start
```

## 5.3 主节点恢复

修复原主节点后将其恢复为当前备节点

在node1上执行一次基础同步：

```shell
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

```shell
[root@node1 ~]# rm -rf /var/lib/pgsql/tmp/PGSQL.lock	
```

{该锁文件在当节点为主节点时创建，但不会因为heartbeat的异常停止或数据库/系统的异常终止而自动删除，所以在恢复一个节点的时候只要该节点充当过主节点就需要手动清理该锁文件}

重启node1上的heartbeat：

```shell
[root@node1 ~]# service heartbeat restart
```

过段时间后查看集群状态：

```shell
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

## 6.1 启动关闭corosync

```
[root@node1 ~]# service corosync start
[root@node1 ~]# service corosync stop

```

## 6.2 查看HA状态

```shell
[root@node1 ~]# crm status
Last updated: Tue Jan 21 23:55:13 2014
Last change: Tue Jan 21 23:37:36 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

```

## 6.3 查看资源状态及节点属性

```sql
[root@node1 ~]# crm_mon -Afr -1
Last updated: Tue Jan 21 23:37:20 2014
Last change: Tue Jan 21 23:37:36 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                 : 100
    + master-pgsql                     : 1000
    + pgsql-data-status                : LATEST
    + pgsql-master-baseline            : 0000000006000078
    + pgsql-status                     : PRI
* Node node2:
    + default_ping_set                 : 100
    + master-pgsql                     : 100
    + pgsql-data-status                : STREAMING|SYNC
    + pgsql-status                     : HS:sync   

Migration summary:
* Node node2:
* Node node1:

```

## 6.4 查看配置

```shell
[root@node1 ~]# crm configure show
node node1 \
        attributes pgsql-data-status="LATEST"
node node2 \
        attributes pgsql-data-status="STREAMING|SYNC"
primitive pgsql ocf:heartbeat:pgsql \
        params pgctl="/opt/pgsql/bin/pg_ctl" psql="/opt/pgsql/bin/psql" pgdata="/opt/pgsql/data/" start_opt="-p 5432" rep_mode="sync" node_list="node1 node2" restore_command="cp /opt/archivelog/%f %p" primary_conninfo_opt="keepalives_idle=60 keepalives_interval=5 keepalives_count=5" master_ip="192.168.1.3" stop_escalate="0" \
        op start timeout="60s" interval="0s" on-fail="restart" \
        op monitor timeout="60s" interval="7s" on-fail="restart" \
        op monitor timeout="60s" interval="2s" on-fail="restart" role="Master" \
        op promote timeout="60s" interval="0s" on-fail="restart" \
        op demote timeout="60s" interval="0s" on-fail="stop" \
……
……

```

## 6.5 实时监控HA

```shell
[root@node1 ~]# crm_mon -Afr
Last updated: Wed Jan 22 00:40:12 2014
Last change: Tue Jan 21 23:37:36 2014 via crm_attribute on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

Full list of resources:

vip-slave (ocf::heartbeat:IPaddr2): Started node2
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep    (ocf::heartbeat:IPaddr2): Started node1
 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Started: [ node1 node2 ]

Node Attributes:
* Node node1:
    + default_ping_set                  : 100
    + master-pgsql                      : 1000
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 0000000006000078
    + pgsql-status                      : PRI
* Node node2:
    + default_ping_set                  : 100
    + master-pgsql                      : 100
    + pgsql-data-status                 : STREAMING|SYNC
    + pgsql-status                      : HS:sync   

Migration summary:* Node node2: * Node node1:

```

## 6.6 crm_resource命令

### 资源启动/关闭：

```shell
[root@node1 ~]# crm_resource -r vip-master -v started
[root@node1 ~]# crm_resource -r vip-master -v stoped

```

### 列举资源：

```shell
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

```shell
[root@node1 ~]# crm_resource -W -r pgsql
resource pgsql is running on: node2

```

### 迁移资源：

```shell
[root@node1 ~]# crm_resource -M -r vip-slave -N node2

```

### 删除资源：

```shell
[root@node1 ~]# crm_resource -D -r vip-slave -t primitive

```

## 6.7 crm命令

### 列举指定的RA：

```shell
[root@node1 ~]# crm ra list ocf pacemaker
ClusterMon     Dummy          HealthCPU      HealthSMART    Stateful       SysInfo        SystemHealth   controld       ping           pingd
remote

```

### 删除节点：

```shell
[root@node1 ~]# crm node delete node2

```

### 停用节点：

```shell
[root@node1 ~]# crm node standby node2

```

### 启用节点：

```shell
[root@node1 ~]# crm node online node2

```

### 配置pacemaker：

```shell
[root@node1 ~]# crm configure
crm(live)configure#
……
……
crm(live)configure# commit
crm(live)configure# quit

```

## 6.8 重置failcount

```shell
[root@node1 ~]# crm resource
crm(live)resource# failcount pgsql set node1 0
crm(live)resource# failcount pgsql show node1
scope=status  name=fail-count-pgsql value=0

[root@node1 ~]# crm resource cleanup pgsql
Cleaning up pgsql:0 on node1
Waiting for 1 replies from the CRMd. OK

[root@node1 ~]# crm_failcount -G -U node1 -r pgsql
scope=status  name=fail-count-pgsql value=INFINITY
[root@node1 ~]# crm_failcount -D -U node1 -r pgsql

```

# 七、问题记录

## 7.1 Q1

问题现象：

corosync.log日志中报错：

Jan 15 10:23:57 node1 lrmd: [6327]: info: RA output: (pgsql:0:monitor:stderr) /usr/lib/ocf/resource.d//heartbeat/pgsql: line 1749: ocf_local_nodename: command not found

Jan 15 10:23:57 node1 crm_attribute: [11094]: info: Invoked: /usr/sbin/crm_attribute -l reboot -N -n -v 0000000006000090 pgsql-xlog-loc lrm_get_rsc_type_metadata(578)

Jan 15 10:23:57 node1 lrmd: [6327]: info: RA output: (pgsql:0:monitor:stderr) Could not map uname=-n to a UUID: The object/attribute does not exist

解决方式：

查看pgsql脚本，发现其中使用了ocf_local_nodename，该函数本该在ocf-shellfuncs.in中有定义，但却没有这个函数，上网查看相关论坛

<http://www.gossamer-threads.com/lists/linuxha/users/89379?do=post_view_threaded>�

指出此时需要相关补丁，解决ocf_local_nodename函数的补丁：

[https://github.com/ClusterLabs/resource-agents/commit/abc1c3f6464f6e5e7a1e41cd7c9b8179896c1903]()�

最新的版本没有ocf_local_nodename函数，所以使用以下版本：

{注：确保pacemaker版本>1.1.8，不然crm_node -n命令无法使用}

[https://github.com/ClusterLabs/resource-agents/blob/a6f4ddf76cb4bbc1b3df4c9b6632a6351b63c19e/heartbeat/pgsql]()�

[https://github.com/ClusterLabs/resource-agents/tree/abc1c3f6464f6e5e7a1e41cd7c9b8179896c1903/heartbeat]()�

不含有ocf_local_nodename函数的pgsql脚本：

[https://raw.github.com/ClusterLabs/resource-agents/a6f4ddf76cb4bbc1b3df4c9b6632a6351b63c19e/heartbeat/pgsql]()�

## 7.2 Q2

问题现象：

```shell
[root@node1 ~]# crm configure load update pgsql.crm
WARNING: pingCheck: specified timeout 60s for start is smaller than the advised 90
WARNING: pingCheck: specified timeout 60s for stop is smaller than the advised 100
WARNING: pgsql: specified timeout 60s for stop is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for start is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for notify is smaller than the advised 90
WARNING: pgsql: specified timeout 60s for demote is smaller than the advised 120
WARNING: pgsql: specified timeout 60s for promote is smaller than the advised 120
ERROR: master-group: attribute ordered does not exist
Do you still want to commit? no

```

解决方式：

错误提示：在定义的master-group中ordered属性不存在

（1）该问题是pacemaker版本所致，在pacemaker-1.1版本中不支持ordered,colocated属性，通过以下方法以1.0版本的cibconfig.py替换当前新版本试图解决此问题，结果失败：

```shell
[root@node1 ~]# vim /usr/lib64/python2.6/site-packages/crmsh/cibconfig.py
[root@node1 ~]# cd /usr/lib64/python2.6/site-packages/crmsh/
[root@node1 crmsh]# mv cibconfig.py cibconfig.py.bak
[root@node1 crmsh]# wget https://github.com/ClusterLabs/pacemaker-1.0/blob/fa1a99ab36e0ed015f1bcbbb28f7db962a9d1abc/shell/modules/cibconfig.py

```

（2）从配置脚本中去除关于ordered的定义（成功）：

group master-group \

vip-master \

vip-rep \

meta \

ordered="false"

改为：

group master-group \

vip-master \

vip-rep

## 7.3 Q3

问题现象：

安装pacemaker时报错：

```shell
# yum install pacemaker*
……
--> Processing Dependency: libesmtp.so.5()(64bit) for package: pacemaker
--> Finished Dependency Resolution
pacemaker-1.0.12-1.el5.centos.i386 from clusterlabs has depsolving problems
  --> Missing Dependency: libesmtp.so.5 is needed by package pacemaker-1.0.12-1.el5.centos.i386 (clusterlabs)
pacemaker-1.0.12-1.el5.centos.x86_64 from clusterlabs has depsolving problems
  --> Missing Dependency: libesmtp.so.5()(64bit) is needed by package pacemaker-1.0.12-1.el5.centos.x86_64 (clusterlabs)
Error: Missing Dependency: libesmtp.so.5 is needed by package pacemaker-1.0.12-1.el5.centos.i386 (clusterlabs)
Error: Missing Dependency: libesmtp.so.5()(64bit) is needed by package pacemaker-1.0.12-1.el5.centos.x86_64 (clusterlabs)
 You could try using --skip-broken to work around the problem
 You could try running: package-cleanup --problems
                        package-cleanup --dupes
                        rpm -Va --nofiles --nodigest
The program package-cleanup is found in the yum-utils package.

```

解决方式：

提示缺少libesmtp，安装即可

```shell
# wget ftp://ftp.univie.ac.at/systems/linux/fedora/epel/5/x86_64/libesmtp-1.0.4-5.el5.x86_64.rpm
# wget ftp://ftp.univie.ac.at/systems/linux/fedora/epel/5/i386/libesmtp-1.0.4-5.el5.i386.rpm
# rpm -ivh libesmtp-1.0.4-5.el5.x86_64.rpm
# rpm -ivh libesmtp-1.0.4-5.el5.i386.rpm

```

## 7.4 Q4

问题现象：

加载crm配置时报错：

```shell
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
Do you still want to commit? no

```

解决方式：

参数不存在是因为pgsql脚本太旧，需要替换

```shell
scp pgsql root@192.168.100.201:/usr/lib/ocf/resource.d/heartbeat/
scp ocf-shellfuncs.in root@192.168.100.201:/usr/lib/ocf/lib/heartbeat/

scp pgsql root@192.168.100.202:/usr/lib/ocf/resource.d/heartbeat/
scp ocf-shellfuncs.in root@192.168.100.202:/usr/lib/ocf/lib/heartbeat/

```

## 7.5 Q5

问题现象：

```sql
[root@node1 ~]# crm_mon -Afr -1
Last updated: Tue Jan 21 05:10:56 2014
Last change: Tue Jan 21 05:10:08 2014 via cibadmin on node1
Stack: classic openais (with plugin)
Current DC: node1 - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
2 Nodes configured, 2 expected votes
7 Resources configured

Online: [ node1 node2 ]

Full list of resources:

 vip-slave (ocf::heartbeat:IPaddr2): Stopped
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Stopped
     vip-rep (ocf::heartbeat:IPaddr2): Stopped
 Master/Slave Set: msPostgresql [pgsql]
     Stopped: [ node1 node2 ]
 Clone Set: clnPingCheck [pingCheck]
     Stopped: [ node1 node2 ]

Node Attributes:
* Node node1:
* Node node2:

Migration summary:
* Node node1:
* Node node2: 

Failed actions:
    pingCheck_monitor_0 on node1 'invalid parameter' (2): call=23, status=complete, last-rc-change='Tue Jan 21 05:10:10 2014', queued=200ms, exec=0ms
    pingCheck_monitor_0 on node2 'invalid parameter' (2): call=23, status=complete, last-rc-change='Tue Jan 21 05:09:36 2014', queued=281ms, exec=0ms

```

解决方式：

该错误是因为脚本定义中的pingCheck调用的pingd脚本中存在未知参数，经查ocf/pacemaker/pingd中不存在multiplier参数：

primitive pingCheck ocf:pacemaker:pingd \

params \

name="default_ping_set" \

host_list="192.168.100.1" \

multiplier="100" \

op start timeout="60s" interval="0s" on-fail="restart" \

op monitor timeout="60s" interval="10s" on-fail="restart" \

op stop timeout="60s" interval="0s" on-fail="ignore"

因此将调用改为ocf:heartbeat:pingd

## 7.6 Q6

问题现象：

corosync日志中报错：

Jan 21 04:36:02 corosync [TOTEM ] Received message has invalid digest... ignoring.

Jan 21 04:36:02 corosync [TOTEM ] Invalid packet data

解决方式：

说明网络中存在相同的多播，更改多播地址即可。

## 7.6 Q6

问题现象：

crm configure load update pgsql.crm 报错：

```sql
lrmadmin[10677]: 2018/01/04_21:34:11 ERROR: lrm_get_rsc_type_metadata(578): got a return code HA_FAIL from a reply message of rmetadata with functio
n get_ret_from_msg.ERROR: ocf:heartbeat:pgsql: could not parse meta-data: 
lrmadmin[10681]: 2018/01/04_21:34:12 ERROR: lrm_get_rsc_type_metadata(578): got a return code HA_FAIL from a reply message of rmetadata with functio
n get_ret_from_msg.ERROR: ocf:pacemaker:ping: could not parse meta-data: 
lrmadmin[10689]: 2018/01/04_21:34:12 ERROR: lrm_get_rsc_type_metadata(578): got a return code HA_FAIL from a reply message of rmetadata with functio
n get_ret_from_msg.ERROR: ocf:heartbeat:IPaddr2: could not parse meta-data: 
ERROR: ocf:heartbeat:IPaddr2: could not parse meta-data: 
ERROR: ocf:heartbeat:IPaddr2: could not parse meta-data: 
ERROR: ocf:heartbeat:pgsql: could not parse meta-data: 
ERROR: ocf:heartbeat:pgsql: no such resource agent
ERROR: ocf:pacemaker:ping: could not parse meta-data: 
ERROR: ocf:pacemaker:ping: no such resource agent
ERROR: ocf:heartbeat:IPaddr2: could not parse meta-data: 
ERROR: ocf:heartbeat:IPaddr2: no such resource agent
ERROR: ocf:heartbeat:IPaddr2: could not parse meta-data: 
ERROR: ocf:heartbeat:IPaddr2: no such resource agent
ERROR: ocf:heartbeat:IPaddr2: could not parse meta-data: 
ERROR: ocf:heartbeat:IPaddr2: no such resource agent
Do you still want to commit? Ctrl-C, leaving

```

解决方式：



# 八、参考资源

脚本：

[https://github.com/ClusterLabs/resource-agents/blob/master/heartbeat/pgsql]()�

脚本使用说明：

[https://github.com/t-matsuo/resource-agents/wiki/Resource-Agent-for-PostgreSQL-9.1-streaming-replication]()�

crm_resouce命令：

<http://www.novell.com/zh-cn/documentation/sle_ha/book_sleha/data/man_crmresource.html>�

crm_failcount命令：

<http://www.novell.com/zh-cn/documentation/sle_ha/book_sleha/data/man_crmfailcount.html>�