# 3、kafka 分区与分组 

**声明：本文根据网上资料整理的学习笔记，严禁商用**。

 一、

1、原理图

![img](assets/3.1pa-gr.png)

2、原理描述

一个topic 可以配置几个partition，produce发送的消息分发到不同的partition中，consumer接受数据的时候是按照group来接受，kafka确保每个partition只能同一个group中的同一个consumer消费，如果想要重复消费，那么需要其他的组来消费。Zookeerper中保存这每个topic下的每个partition在每个group中消费的offset 

 

新版kafka把这个offsert保存到了一个__consumer_offsert的topic下 

这个__consumer_offsert 有50个分区，通过将group的id哈希值%50的值来确定要保存到那一个分区.  这样也是为了考虑到zookeeper不擅长大量读写的原因。

所以，如果要一个group用几个consumer来同时读取的话，需要多线程来读取，一个线程相当于一个consumer实例。当consumer的数量大于分区的数量的时候，有的consumer线程会读取不到数据。 

假设一个topic test 被groupA消费了，现在启动另外一个新的groupB来消费test，默认test-groupB的offset不是0，而是没有新建立，除非当test有数据的时候，groupB会收到该数据，该条数据也是第一条数据，groupB的offset也是刚初始化的ofsert, 除非用显式的用–from-beginnging 来获取从0开始数据 

 

3、查看topic-group的offsert 

位置：zookeeper 

路径：[zk: localhost:2181(CONNECTED) 3] ls /brokers/topics/__consumer_offsets/partitions 

在zookeeper的topic中有一个特殊的topic __consumer_offserts 

计算方法：（放入哪个partitions）

int hashCode = Math.abs("ttt".hashCode());

int partition = hashCode % 50;

先计算group的hashCode，再除以分区数(50),可以得到partition的值 

使用命令查看： kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 11 --broker-list localhost:9092,localhost:9093,localhost:9094 --formatter "kafka.coordinator.GroupMetadataManager\$OffsetsMessageFormatter"

![img](assets/3.2pa-calculate.png)

4.参数 

auto.offset.reset:默认值为largest，代表最新的消息，smallest代表从最早的消息开始读取，当consumer刚开始创建的时候没有offset这种情况，如果设置了largest，则为当收到最新的一条消息的时候开始记录offsert,若设置为smalert，那么会从头开始读partition

 

二、

1、Topic

​     Topic在逻辑上可以被认为是一个queue，每条消费都必须指定它的Topic，可以简单理解为必须指明把这条消息放进哪个queue里。为了使得Kafka的吞吐率可以线性提高，物理上把Topic分成一个或多个Partition，每个Partition在物理上对应一个文件夹，该文件夹下存储这个Partition的所有消息和索引文件。若创建topic1和topic2两个topic，且分别有13个和19个分区，则整个集群上会相应会生成共32个文件夹（本文所用集群共8个节点，此处topic1和topic2 replication-factor均为1），如下图所示。

![img](assets/3.3pa.png)

2、对于传统的message queue而言，一般会删除已经被消费的消息，而Kafka集群会保留所有的消息，无论其被消费与否。当然，因为磁盘限制，不可能永久保留所有数据（实际上也没必要），

​     因此Kafka提供两种策略删除旧数据。一是基于时间，二是基于Partition文件大小。

​     例如可以通过配置$KAFKA_HOME/config/server.properties，让Kafka删除一周前的数据，也可在Partition文件超过1GB时删除旧数据，配置如下所示。

![img](assets/3.4.pa.png)

   这里要注意，因为Kafka读取特定消息的时间复杂度为O(1)，即与文件大小无关，所以这里删除过期文件与提高Kafka性能无关。选择怎样的删除策略只与磁盘以及具体的需求有关。另外，Kafka会为每一个Consumer Group保留一些metadata信息——当前消费的消息的position，也即offset。这个offset由Consumer控制。正常情况下Consumer会在消费完一条消息后递增该offset。当然，Consumer也可将offset设成一个较小的值，重新消费一些消息。因为offet由Consumer控制，所以Kafka broker是无状态的，它不需要标记哪些消息被哪些消费过，也不需要通过broker去保证同一个Consumer Group只有一个Consumer能消费某一条消息，因此也就不需要锁机制，这也为Kafka的高吞吐率提供了有力保障。

 

 3、producer

Producer发送消息到broker时，会根据Paritition机制选择将其存储到哪一个Partition。如果Partition机制设置合理，所有消息可以均匀分布到不同的Partition里，这样就实现了负载均衡。如果一个Topic对应一个文件，那这个文件所在的机器I/O将会成为这个Topic的性能瓶颈，而有了Partition后，不同的消息可以并行写入不同broker的不同Partition里，极大的提高了吞吐率。可以在$KAFKA_HOME/config/server.properties中通过配置项num.partitions来指定新建Topic的默认Partition数量，也可在创建Topic时通过参数指定，同时也可以在Topic创建之后通过Kafka提供的工具修改。

 

在发送一条消息时，可以指定这条消息的key，Producer根据这个key和Partition机制来判断应该将这条消息发送到哪个Parition。Paritition机制可以通过指定Producer的paritition. class这一参数来指定，该class必须实现kafka.producer.Partitioner接口。本例中如果key可以被解析为整数则将对应的整数与Partition总数取余，该消息会被发送到该数对应的Partition。（每个Parition都会有个序号,序号从0开始）

import kafka.producer.Partitioner;
import kafka.utils.VerifiableProperties;

public class JasonPartitioner<T> implements Partitioner {

    public JasonPartitioner(VerifiableProperties verifiableProperties) {}
     
    @Override
    public int partition(Object key, int numPartitions) {
        try {
            int partitionNum = Integer.parseInt((String) key);
            return Math.abs(Integer.parseInt((String) key) % numPartitions);
        } catch (Exception e) {
            return Math.abs(key.hashCode() % numPartitions);
        }
    }
如果将上例中的类作为partition.class，并通过如下代码发送20条消息（key分别为0，1，2，3）至topic3（包含4个Partition）。

public void sendMessage() throws InterruptedException{
for(int i = 1; i <= 5; i++){
      List messageList = new ArrayList<KeyedMessage<String, String>>();
      for(int j = 0; j < 4; j++）{
          messageList.add(new KeyedMessage<String, String>("topic2", j+"", "The " + i + " message for key " + j));
      }
      producer.send(messageList);
    }
producer.close();
}

则key相同的消息会被发送并存储到同一个partition里，而且key的序号正好和Partition序号相同。（Partition序号从0开始，本例中的key也从0开始）。下图所示是通过Java程序调用Consumer后打印出的消息列表。

![img](assets/3.5pa.png)

4、consumer group   （本节所有描述都是基于Consumer hight level API而非low level API）。

​     使用Consumer high level API时，同一Topic的一条消息只能被同一个Consumer Group内的一个Consumer消费，但多个Consumer Group可同时消费这一消息。

![img](assets/3.6pa.png)

这是Kafka用来实现一个Topic消息的广播（发给所有的Consumer）和单播（发给某一个Consumer）的手段。一个Topic可以对应多个Consumer Group。如果需要实现广播，只要每个Consumer有一个独立的Group就可以了。要实现单播只要所有的Consumer在同一个Group里。用Consumer Group还可以将Consumer进行自由的分组而不需要多次发送消息到不同的Topic。

实际上，Kafka的设计理念之一就是同时提供离线处理和实时处理。根据这一特性，可以使用Storm这种实时流处理系统对消息进行实时在线处理，同时使用Hadoop这种批处理系统进行离线处理，还可以同时将数据实时备份到另一个数据中心，只需要保证这三个操作所使用的Consumer属于不同的Consumer Group即可。

