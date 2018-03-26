## window常用的cmd命令

##### 查看占用端口的进程得到pid

```
netstat -a -n -o
```

##### 杀掉指定进程

```
taskkill /pid 8990 -f
```

##### find使用

```shell
find "abc" D:\abc\abc.txt              //查找含有abc的行,并输出该行
find /n "abc" D:\abc\abc.txt       //查找含有abc的行,并输出该行和行号
find /i "abc" D:\abc\abc.txt          //查找含有abc(忽略大小写)的行,并输出该行
find /i /v "abc" D:\abc\abc.txt      //查找不含有abc(忽略大小写)的行,并输出所有行
find /c /i /v "any sting which not exist" D:\abc\abc.txt             // 统计所有行数
netstat -na | find /C  /I /V "any sting which not exist"          // 统计当前网络连接数
@echo off
title 显示系统信息
color 2f
systeminfo | find "主机名"
systeminfo | find "OS"
systeminfo | find "注册"
systeminfo | find "ID"
systeminfo | find "初始安装日期"
systeminfo | find "系统"
echo 系统相关信息已获得，按任意键退出。
pause > NULL
```

#####使用 powershell 的 grep 过滤文本

有个log文件，大小在4M左右，要求找出里面耗时超过100s 的记录。首先想到了强大的 grep ，那么就搞起。
先在网上找一下资料，这篇文章，有几种方式：

```
第一种：

Get-content somefile.txt|findstr "someregexp"

Get-content可以换成cat，Powershell已经给他们做了个别名，可真是体谅sheller。

这种方法算是commandline和Powershell混合，因为findstr是命令行工具，并不是Powershell的cmdlet。

第二种：

cat somefile.txt | where { $ -match "some_regexp"}

纯种Powershell实现了，利用了where过滤

第三种：

Select-String "some_regexp" somefile.txt

直接用Select-string的实现。

经过测试，最后写出的 powershell 命令如下:

cat .\log.log|where {$_ -match "\d{3,}.\d{2,}s"} >>result.log

用了 where 这个， 这个能使用正则， findstr 命令不行。里面的正则匹配字符串 "\d{3,}.\d{2,}s" 也很简单了，"3个数字.2个数字以上s "的意思。

```

