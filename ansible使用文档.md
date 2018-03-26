## 1. ansible主机清单的配置

**vim /etc/ansible/hosts**

**定义方式：**

- 直接指明主机地址或主机名

  ```
    blue.example.com
    192.168.100.1
  ```

- 定义一个主机组`[组名]`把地址或主机名加进去

  ```
    [webservers]
    alpha.example.org
    beta.example.org
    192.168.1.100

    #组成员可以使用通配符来匹配，如下
    www[001:006].example.com  #表示从www001-www006主机
  ```
  嵌套

  ```
  [webservers]
  192.168.3.49 ansible_ssh_user=chdb ansible_ssh_pass=HF*cb303
  [dbservers]
  192.168.3.62
  [wolf:children]
  webservers
  dbservers
  ```

- 如果你没有使用公钥，想要使用密码，你也可以这样写（适用于第一次登陆控制）

  ```
    格式：【主机名】 【主机地址】 【主机密码】  默认是root用户来进行的
    [keepalived]
    keepalived1  ansible_ssh_host=192.168.146.136  ansible_ssh_user=chdb ansible_ssh_pass="test"
    keepalived2  ansible_ssh_host=192.168.146.137 ansible_ssh_pass="test"
  ```



# 2. ansible的使用`ansible-doc`

```shell
一般用法:
ansible-doc -l 获取模块信息
ansible-doc -s MOD_NAME 获取指定模块的使用帮助
```

## 3. ansible使用 之一 “命令管理方式”

### 常用模块：

=================================
**ping：探测目标主机是否存活；**

ping模块主要是无意义的测试模块，主要用来检查ansible是否可以用的模块以及python是否配置好的，在playbook中基本不会使用，在能成功连接之后，总是返回结果

```
[root@ansibleserver ~]# ansible all -m ping
SSH password:
ansiblemoniter | success >> {
    "changed": false,
    "ping": "pong"
}
```

=================================

**command：在远程主机执行命令；不支持|管道命令**

=================================

```
ansible storm_cluster(用户组) -m command -a "ls –al /tmp/resolv.conf"
```

**ansible 批量创建用户**

```
  ansible chmap   -m shell -a "useradd -m chdb -d /app/chdb"
  修改密码:
  ansible chmap   -m shell -a "echo 'chmap'  |passwd  chmap   --stdin"
```

```
相关选项如下：
creates：一个文件名，当该文件存在，则该命令不执行
free_form：要执行的linux指令
chdir：在执行指令之前，先切换到该目录
removes：一个文件名，当该文件不存在，则该选项不执行
executable：切换shell来执行指令，该执行路径必须是一个绝对路径
```

**shell：在远程主机上调用shell解释器运行命令，支持shell的各种功能，例如管道等 ；**

注意：command和shell模块的核心参数直接为命令本身；而其它模块的参数通常为“key=value”格式；

**ansible 批量创建用户**

```
  ansible chmap   -m shell -a "useradd -m chdb -d /app/chdb"
  修改密码:
  ansible chmap   -m shell -a "echo 'chmap'  |passwd  chmap   --stdin"
```

**shell：在远程主机上调用shell解释器运行命令，支持shell的各种功能，例如管道等 ；**

================================

**copy：复制文件到远程主机，可以改权限等**

================================

```shell
用法：
(1) 复制文件
    -a "src=  dest=  "
(2) 给定内容生成文件
    -a "content=  dest=  "
 例如:
 ansible all -m copy -a "src=/etc/fstab dest=/tmp/asnbiblecopy mode=640 backup=yes" 
  ansible all -m copy -a "content='hello\nworld' dest=/tmp/asnbiblecopy mode=640 backup=yes"
```

```shell
 相关选项如下：
backup：在覆盖之前，将源文件备份，备份文件包含时间信息。有两个选项：yes|no
content：用于替代“src”，可以直接设定指定文件的值
dest：必选项。要将源文件复制到的远程主机的绝对路径，如果源文件是一个目录，那么该路径也必须是个目录
directory_mode：递归设定目录的权限，默认为系统默认权限
force：如果目标主机包含该文件，但内容不同，如果设置为yes，则强制覆盖，如果为no，则只有当目标主机的目标位置不存在该文件时，才复制。默认为yes
others：所有的file模块里的选项都可以在这里使用
src：被复制到远程主机的本地文件，可以是绝对路径，也可以是相对路径。如果路径是一个目录，它将递归复制。在这种情况下，如果路径使用“/”来结尾，则只复制目录里的内容，如果没有使用“/”来结尾，则包含目录在内的整个内容全部复制，类似于rsync。
```

===============================
**file ：设置文件属性。**

**设置文件属性**

```
ansible pythonserver -m file -a "path=/root/123 owner=kel group=kel mode=0644"
```

文件路径为path，表示文件路径，设定所属用户和所属用户组，权限为0644

```
ansible pythonserver -m file -a "path=/tmp/kel/ owner=kel group=kel mode=0644 recurse=yes"
```

文件路径为path，使用文件夹进行递归修改权限，使用的参数为recurse表示为递归

**创建目录**

```
ansible pythonserver -m file -a "path=/tmp/kel state=directory mode=0755"
```

创建目录，使用的参数主要是state为directory

**修改权限**

```
 ansible pythonserver -m file -a "path=/tmp/kel mode=0444"
```

直接使用mode来进行修改权限

**创建软连接** 	

```
 ansible pythonserver -m file -a "src=/tmp/1 dest=/tmp/2 owner=kel state=link"
```

==============================
**fetch：** 从远程某一个主机获取文件到本地

文件拉取模块主要是将远程主机中的文件拷贝到本机中，和copy模块的作用刚刚相反，并且在保存的时候使用hostname来进行保存，当文件不存在的时候，会出现错误，除非设置了选项fail_on_missing为yes

```shell
ansible pythonserver -m fetch -a "src=/root/123 dest=/root"
192.168.1.60 | success >> {
    "changed": true,
    "dest": "/root/192.168.1.60/root/123",
    "md5sum": "31be5a34915d52fe0a433d9278e99cac",
    "remote_md5sum": "31be5a34915d52fe0a433d9278e99cac"
}
src表示为远程主机上需要传送的文件路径，dest表示为本机上的路径，在传送过来的文件，是按照IP地址进行分类，然后路径是源文件的路径
#在拉取文件的时候，必须拉取的是文件，不能拉取文件夹
```

==============================

==============================

hostname: 修改主机名称

```

[root@ansibleserver ~]# ansible pythonserver -m hostname -a "name=python"
SSH password:
192.168.1.60 | success >> {
    "changed": true,
    "name": "python"
}
在查看的时候，主要查看文件/etc/sysconfig/network，重启之后才能生效
```

==============================

=============================

**cron: 管理cron计划任务**

```
- a "": 设置管理节点生成定时任务
action: cron
backup=    # 如果设置，创建一个crontab备份
cron_file=          #如果指定, 使用这个文件cron.d，而不是单个用户crontab
day=       # 日应该运行的工作( 1-31, *, */2, etc )
hour=      # 小时 ( 0-23, *, */2, etc )
job=       #指明运行的命令是什么
minute=    #分钟( 0-59, *, */2, etc )
month=     # 月( 1-12, *, */2, etc )
name=     #定时任务描述
reboot    # 任务在重启时运行，不建议使用，建议使用special_time
special_time       # 特殊的时间范围，参数：reboot（重启时）,annually（每年）,monthly（每月）,weekly（每周）,daily（每天）,hourly（每小时）

state        #指定状态，prsent表示添加定时任务，也是默认设置，absent表示删除定时任务

user         # 以哪个用户的身份执行
weekday      # 周 ( 0-6 for Sunday-Saturday, *, etc )

新增一个任务，每五分钟执行一次，任务名称为check
ansible pythonserver -m cron -a "name=check minute=5 job='crontab -l >>/root/123'"
-------------------------------------------------------------
删除定时任务
ansible pythonserver -m cron -a "name=check state=absent"
SSH password:
192.168.1.60 | success >> {
    "changed": true,
    "jobs": []
}
------------------------------------	-------------------------
新建一个cron文件
ansible pythonserver -m cron -a "name='for test' weekday='2' minute='0' hour=12 user='root' job='cat /etc/passwd >/root/111' cron_file='test ansible'"
新增一个任务，在目录/etc/cron.d/目录中，文件名称为test ansible，用户为root
-------------------------------------------------------------
删除一个cron文件
ansible pythonserver -m cron -a "name='for test' cron_file='test ansible' state=absent"
SSH password:
192.168.1.60 | success >> {
    "changed": true,
    "cron_file": "test ansible",
    "jobs": []
}
```

==============================

**收集fact并且进行保存**

```
ansible pythonserver -m setup --tree /tmp/facts
```

执行之后，会显示相关的fact，并且在/tmp/facts中会保存fact信息，如下：

```
[root@ansibleserver tmp]# ls -l /tmp/facts/
total 12-rw-r--r-- 1 root root 8990 Jan 18 13:16 192.168.1.60
```

使用--tree选项，在分类的时候，会根据主机的名称进行分类

```
收集内存信息并输出
ansible 20.200.21.22 -m setup  -a "filter=ansible*mb" -u chmap
SSH password:
192.168.1.60 | success >> {
    "ansible_facts": {
        "ansible_memfree_mb": 746,
        "ansible_memtotal_mb": 996,
        "ansible_swapfree_mb": 2015,
        "ansible_swaptotal_mb": 2015
    },
    "changed": false
}
-----------------------------------------------------
收集主机网卡信息
[root@ansibleserver tmp]# ansible pythonserver -m setup -a "filter=ansible_eth[02]"
SSH password:
192.168.1.60 | success >> {
    "ansible_facts": {
        "ansible_eth0": {
            "active": true,
            "device": "eth0",
            "ipv4": {
                "address": "192.168.1.60",
                "netmask": "255.255.255.0",
                "network": "192.168.1.0"
            },
            "ipv6": [
                {
                    "address": "fe80::a00:27ff:fee5:e8a8",
                    "prefix": "64",
                    "scope": "link"
                }
            ],
            "macaddress": "08:00:27:e5:e8:a8",
            "module": "e1000",
            "mtu": 1500,
            "promisc": false,
            "type": "ether"
        }
    },
    "changed": false
}
收集可用的facts,收集每个节点的相关信息：架构信息，IP,时间，域名，网卡，MAC，主机名，CPU等信息。
这些收集的信息，可以作为变量。
```

**在远程主机执行本地脚本：script**

```
ansible erp -m script -a '/tmp/a.sh'
```

