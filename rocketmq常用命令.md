### rocketmq 常用命令

#### 1. 关闭nameserver和所有的broker:

```
sh mqshutdown namesrv
sh mqshutdown broker
```

#### 2. 查看所有消费组group:

```sh mqadmin consumerProgress -n 192.168.23.159:9876```

#### 3. 查看指定消费组下的所有topic数据堆积情况：

```sh mqadmin consumerProgress -n 192.168.23.159:9876 -g ywdGroupConsumer```

#### 4. 查看所有topic :

```sh mqadmin topicList -n 192.168.23.159:9876```

#### 5. 查看topic信息列表详情统计

```sh mqadmin topicstatus -n 192.168.23.159:9876 -t myTopicTest1```

#### 6. 新增topic

```sh mqadmin updateTopic –n 10.45.47.168 –c DefaultCluster –t ZTEExample```

#### 7. 删除topic

```sh mqadmin deleteTopic –n 10.45.47.168:9876 –c DefaultCluster –t ZTEExample```

#### 8. 查看printmsg消息

```sh mqadmin  printMsg -n 20.200.21.15:9876  -t PCTEST2```

