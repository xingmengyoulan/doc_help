# redis 高可用性能

主虚拟机1--IP ：22.224.54.136  

从虚拟机2--IP ：22.224.54.141

虚拟-VIP ：22.224.54.178

###测试场景 一：

```shell
主节点 keepalived 的 进程被杀，redis服务正常,从节点 keepalived 正常，redis服务正常
执行后现象：
虚拟-VIP ：22.224.54.178 漂移到从节点（22.224.54.141），使用在/etc/keepalived/使用test.sh 查看
/app/chredis/bin/redis-cli -p 4000 info |grep -A 5 Replication
从节点（22.224.54.136） redis服务变为主
主节点（22.224.54.141） redis服务变为从
# keepalived 重新启动，redis主从关系不会改变
```

### 测试场景 二：

```shell
主节点 keepalived 的 进程正常，redis服务异常,从节点 keepalived 正常，redis服务正常
执行后现象：
# 虚拟-VIP ：22.224.54.178 漂移到从节点（22.224.54.141），使用在/etc/keepalived/使用test.sh 查看
/app/chredis/bin/redis-cli -p 4000 info |grep -A 5 Replication
从节点（22.224.54.141） redis服务变为主
# redis重新启动，redis主从关系发生变化，22.224.54.136 redis为从。
```

### 测试场景 三：

```shell
主节点 keepalived 的 进程正常，redis服务正常,从节点 keepalived 异常，redis服务正常
执行后现象：
# 虚拟-VIP ：22.224.54.178 不会发生变化。
/app/chredis/bin/redis-cli -p 4000 info |grep -A 5 Replication
# redis重新启动，redis主从关系不发生变化。
```

### 测试场景 四：

```shell
主节点 keepalived 的 进程正常，redis服务正常,从节点 keepalived 正常，redis服务异常
执行后现象：
# 虚拟-VIP ：22.224.54.178 不会发生变化。
/app/chredis/bin/redis-cli -p 4000 info |grep -A 5 Replication
# redis需要重新启动，停掉keepalived的进程，keepalived 再重启，避免出现双主的现象。
```

