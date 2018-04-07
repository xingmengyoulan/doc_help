# shell 常用命令

### 1.时间加减法

```shell
date -d "+1 day" +%Y%m%d
date -d "-1 day" +%Y%m%d
date -d "20170101 -1 day" +%Y%m%d
date -d "20170101 +1 day" +%Y%m%d
date -d "+1 week" +%Y%m%d
date -d "+1 month" +%Y%m%d
date -d "+1 year" +%Y%m%d
```

###2.rz sz 命令安装

```shell
1.软件安装
1）编译安装
root 账号登陆后，依次执行以下命令：
cd /tmp
wget http://www.ohse.de/uwe/releases/lrzsz-0.12.20.tar.gz
tar zxvf lrzsz-0.12.20.tar.gz && cd lrzsz-0.12.20
./configure && make && make install
上面安装过程默认把lsz和lrz安装到了/usr/local/bin/目录下，现在我们并不能直接使用，下面创建软链接，并命名为rz/sz：
cd /usr/bin
ln -s /usr/local/bin/lrz rz
ln -s /usr/local/bin/lsz sz
```

###3.修改 UTC的时间

```sehll
修改时区时间：
host# date -s 07:13
Tue Jan 17 07:13:00 GMT 2017
```

### 4. 查看系统信息

```
cat /etc/issue
```

### 5. 查看线程状态

```
ps -elT |grep 7763
0 S   503  7763  7763 20273  0  80   0 - 198301 futex_ pts/4   00:00:00 TuxCom
1 S   503  7763  7764 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7765 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7766 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7767 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7768 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7769 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7770 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7771 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7772 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom
1 S   503  7763  7773 20273  0  80   0 - 198301 msgrcv pts/4   00:00:00 TuxCom

```

