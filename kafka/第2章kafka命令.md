## 二、 Kafka 命令行操作

 **声明：本文根据网上资料整理的学习笔记，严禁商用**。

**首先创建一个主题**

命令：

bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic  test--partitions 2 --replication- factor 1

--zookeeper：指定了Kafka所连接的Zookeeper服务地址

--topic：指定了所要创建主题的名称

--partitions：指定了分区个数

--replication-factor：指定了副本因子

--create：创建主题的动作指令



**展示所有主题**

命令：bin/kafka-topics.sh  --zookeeper  localhost:2181 --list

  

**查看主题详情**

命令：bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test

--describe  查看详情动作指令

 

**动消费端接收消息** 

命令：bin/kafka-console-consumer.sh  --bootstrap-server  localhost:9092  --topic test

--bootstrap-server   指定了连接Kafka集群的地址

--topic  指定了消费端订阅的主题

 

**生产端发送消息**

命令：bin/kafka-console-producer.sh  --broker-list  localhost:9092  --topic  test

--broker-list  指定了连接的Kafka集群的地址

--topic  指定了发送消息时的主题

