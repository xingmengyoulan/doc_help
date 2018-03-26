 [crm用法](http://blog.chinaunix.net/uid-30212356-id-5345399.html) *2015-11-06 18:01:59*

分类： LINUX

### crm命令：

####crm有两种工作方式：

1，批处理模式,就是在shell命令行中直接输入命令；
2，交互式模式（crm(live)#），进入到crmsh中交互执行；
crm的特性：
​    1、任何操作都需要commit提交后才会生效；
​    2、想要删除一个资源之前需要先将资源停止
​    3、可以用  help COMMAND 获取该命令的帮助 
​    4、与Linux命令行一样，都支持TAB补全
crm命令详解：
1.一级子命令

```
root@node1 corosync]# crm

crm(live)# help    	# 获取当前可用命令

Available commands:

cib              manage shadow CIBs #cib管理模块

resource         resources management #所有的资源都在这个子命令后定义

configure        CRM cluster configuration # 编辑集群配置信息

node             nodes management #集群节点管理子命令

options          user preferences #用户优先级

history          CRM cluster history	#CRM历史

site             Geo-cluster support	#地理集群支持

ra               resource agents information center #资源代理子命令（所有与资源代理相关的程都在此命令之下）

status           show cluster status #显示当前集群的状态信息

help,?           show help (help topics for list of topics)	#查看当前区域可能的命令

end,cd,up        go back one level #返回第一级crm(live)#

quit,bye,exit    exit the program  	#退出crm（live）交互模式

```

常用子命令介绍：
1.resource子命令 #定义所有资源的状态

```
crm(live)resource# help

vailable commands:

status           show status of resources #显示资源状态信息

start            start a resource #启动一个资源

stop             stop a resource #停止一个资源

restart          restart a resource #重启一个资源

promote          promote a master-slave resource #提升一个主从资源

demote           demote a master-slave resource #降级一个主从资源

manage           put a resource into managed mode

unmanage         put a resource into unmanaged mode

migrate          migrate a resource to another node #将资源迁移到另一个节点

unmigrate        unmigrate a resource to another node

param            manage a parameter of a resource #管理资源的参数

secret           manage sensitive parameters #管理敏感参数

meta             manage a meta attribute #管理源属性

utilization      manage a utilization attribute

failcount        manage failcounts #管理失效计数器

cleanup          cleanup resource status #清理资源状态

refresh          refresh CIB from the LRM status #从LRM（LRM本地资源管理）更新CIB（集群信息库）

reprobe          probe for resources not started by the CRM #探测在CRM中没有启动的资源

trace            start RA tracing #启用资源代理（RA）追踪

untrace          stop RA tracing #禁用资源代理（RA）追踪

help             show help (help topics for list of topics) #显示帮助

end              go back one level #返回一级（crm(live)#）

quit             exit the program #退出交互式程序

```


2.configure子命令 #资源粘性、资源类型、资源约束

```
crm(live)configure# help

Available commands:

node             define a cluster node #定义一个集群节点

primitive        define a resource #定义资源

monitor          add monitor operation to a primitive #对一个资源添加监控选项（如超时时间，启动失败后的操作）

group            define a group #定义一个组类型（将多个资源整合在一起）

clone            define a clone #定义一个克隆类型（可以设置总的克隆数，每一个节点上可以运行几个克隆）

ms               define a master-slave resource #定义一个主从类型（集群内的节点只能有一个运行主资源，其它从的做备用）

rsc_template     define a resource template #定义一个资源模板

location         a location preference # 定义位置约束优先级（默认运行于那一个节点（如果位置约束值相同，默认倾向性哪一个高，就在哪一个节点上运行））

colocation       colocate resources #排列约束资源（多个资源在一起的可能性）

order            order resources #资源的启动的先后顺序

rsc_ticket       resources ticket dependency#

property         set a cluster property #设置集群属性

rsc_defaults     set resource defaults #设置资源默认属性（粘性）

fencing_topology node fencing order #隔离节点顺序

role             define role access rights #定义角色的访问权限

user             define user access rights #定义用用户访问权限

op_defaults      set resource operations defaults #设置资源默认选项

schema           set or display current CIB RNG schema

show             display CIB objects #显示集群信息库对

edit             edit CIB objects #编辑集群信息库对象（vim模式下编辑）

filter           filter CIB objects #过滤CIB对象

delete           delete CIB objects #删除CIB对象

default-timeouts     set timeouts for operations to minimums from the meta-data

rename           rename a CIB object #重命名CIB对象

modgroup         modify group	#改变资源组

refresh          refresh from CIB #重新读取CIB信息

erase            erase the CIB #清除CIB信息

ptest            show cluster actions if changes were committed

rsctest          test resources as currently configured

cib              CIB shadow management

cibstatus        CIB status management and editing

template         edit and import a configuration from a template

commit           commit the changes to the CIB #将更改后的信息提交写入CIB

verify           verify the CIB with crm_verify #CIB语法验证

upgrade          upgrade the CIB to version 1.0

save             save the CIB to a file #将当前CIB导出到一个文件中（导出的文件存于切换crm 之前的目录）

load             import the CIB from a file #从文件内容载入CIB

graph            generate a directed graph

xml              raw xml

help             show help (help topics for list of topics) #显示帮助信息

end              go back one level #回到第一级(crm(live)#)

```


3.node子命令 #节点管理和状态

```
crm(live)# node

crm(live)node# help

Node management and status commands.

Available commands:

status           show nodes status as XML #以xml格式显示节点状态信息

show             show node #命令行格式显示节点状态信息

standby          put node into standby #模拟指定节点离线（standby在后面必须的FQDN）

online           set node online #节点重新上线

maintenance      put node into maintenance mode

ready            put node into ready mode

fence            fence node #隔离节点

clearstate       Clear node state #清理节点状态信息

delete           delete node #删除 一个节点

attribute        manage attributes

utilization      manage utilization attributes

status-attr      manage status attributes

help             show help (help topics for list of topics)

end              go back one level

quit             exit the program

```

4.ra子命令  #资源代理类别都在此处

```
crm(live)# ra

crm(live)ra# help

Available commands:

classes          list classes and providers	# 为资源代理分类

list             list RA for a class (and provider)	# 显示一个类别中的提供的资源

meta             show meta data for a RA # 显示一个资源代理序的可用参数（如meta ocf💓IPaddr2）

providers        show providers for a RA and a class

help             show help (help topics for list of topics)

end              go back one level

quit             exit the program

```

5.show xml 显示完整的xml格式信息

```
crm(live)configure# show

node node1.a.com

node node2.a.com    	#当前集群共有2个节点

property cib-bootstrap-options: \

dc-version=1.1.11-97629de \   #DC的版本

cluster-infrastructure="classic openais (with plugin)" \    #底层基础架构(经典的openais，使用plugin方式来运行)

expected-quorum-votes=2 \   #当前节点一共有两票

stonith-enabled=false    	#stonith 设备已被禁用

```


6.禁用stonith设备(如果没有stonith设备，最好禁用)：

```
configure

crm(live)configure# property stonith-enabled=false

crm(live)configure# commit

crm_verify -L -V   #此时再检查就不会检查stoith设备了；

```

