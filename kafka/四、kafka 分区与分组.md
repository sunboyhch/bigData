# 四、kafka 分区与分组 

**声明：本文根据网上资料整理的学习笔记，严禁商用**。

 1.kafka分区机制

分区机制是kafka实现高吞吐的秘密武器，但这个武器用得不好的话也容易出问题，今天主要就来介绍分区的机制以及相关的部分配置。

首先，从数据组织形式来说，kafka有三层形式，kafka有多个主题，每个主题有多个分区，每个分区又有多条消息。

而每个分区可以分布到不同的机器上，这样一来，从服务端来说，分区可以实现高伸缩性，以及负载均衡，动态调节的能力。

当然多分区就意味着每条消息都难以按照顺序存储，那么是不是意味着这样的业务场景kafka就无能为力呢？不是的，最简单的做法可以使用单个分区。单个分区，所有消息自然都顺序写入到一个分区中，就跟顺序队列一样了。而复杂些的还有其他办法，那就是使用按消息键，将需要顺序保存的消息存储的单独的分区，其他消息存储其他分区，这个在下面会介绍。

我们可以通过replication-factor指定创建topic时候所创建的分区数。

bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test

比如这里就是创建了1个分区的主题。值得注意的是，还有一种创建主题的方法，是使用zookeeper参数的，那种是比较旧的创建方法，这里是使用bootstrap参数的。

1.1 分区个数选择

既然分区效果这么好，是不是越多分区越好呢？显而易见并非如此。

分区越多，所需要消耗的资源就越多。甚至如果足够大的时候，还会触发到操作系统的一些参数限制。比如linux中的文件描述符限制，一般在创建线程，创建socket，打开文件的场景下，linux默认的文件描述符参数，只有1024，超过则会报错。

看到这里有读者就会不耐烦了，说这么多有啥用，能不能直接告诉我分区分多少个比较好？很遗憾，暂时没有。

因为每个业务场景都不同，只能结合具体业务来看。假如每秒钟需要从主题写入和读取1GB数据，而消费者1秒钟最多处理50MB的数据，那么这个时候就可以设置20-25个分区，当然还要结合具体的物理资源情况。

而如何无法估算出大概的处理速度和时间，那么就用基准测试来测试吧。创建不同分区的topic，逐步压测测出最终的结果。如果实在是懒得测，那比较无脑的确定分区数的方式就是broker机器数量的2~3倍。

1.2 分区写入策略

所谓分区写入策略，即是生产者将数据写入到kafka主题后，kafka如何将数据分配到不同分区中的策略。

常见的有三种策略，轮询策略，随机策略，和按键保存策略。其中轮询策略是默认的分区策略，而随机策略则是较老版本的分区策略，不过由于其分配的均衡性不如轮询策略，故而后来改成了轮询策略为默认策略。

轮询策略

所谓轮询策略，即按顺序轮流将每条数据分配到每个分区中。

举个例子，假设主题test有三个分区，分别是分区A，分区B和分区C。那么主题对接收到的第一条消息写入A分区，第二条消息写入B分区，第三条消息写入C分区，第四条消息则又写入A分区，依此类推。

轮询策略是默认的策略，故而也是使用最频繁的策略，它能最大限度保证所有消息都平均分配到每一个分区。除非有特殊的业务需求，否则使用这种方式即可。

随机策略

随机策略，也就是每次都随机地将消息分配到每个分区。其实大概就是先得出分区的数量，然后每次获取一个随机数，用该随机数确定消息发送到哪个分区。

在比较早的版本，默认的分区策略就是随机策略，但其实使用随机策略也是为了更好得将消息均衡写入每个分区。但后来发现对这一需求而言，轮询策略的表现更优，所以社区后来的默认策略就是轮询策略了。

按键保存策略

按键保存策略，就是当生产者发送数据的时候，可以指定一个key，计算这个key的hashCode值，按照hashCode的值对不同消息进行存储。

至于要如何实现，那也简单，只要让生产者发送的时候指定key就行。欸刚刚不是说默认的是轮询策略吗？其实啊，kafka默认是实现了两个策略，没指定key的时候就是轮询策略，有的话那激素按键保存策略了。

上面有说到一个场景，那就是要顺序发送消息到kafka。前面提到的方案是让所有数据存储到一个分区中，但其实更好的做法，就是使用这种按键保存策略。

让需要顺序存储的数据都指定相同的键，而不需要顺序存储的数据指定不同的键，这样一来，即实现了顺序存储的需求，又能够享受到kafka多分区的优势，岂不美哉。

 

1.3 实现自定义分区

说了这么多，那么到底要如何自定义分区呢？

kafka提供了两种让我们自己选择分区的方法，第一种是在发送producer的时候，在ProducerRecord中直接指定，但需要知道具体发送的分区index，所以并不推荐。

第二种则是需要实现Partitioner.class类，并重写类中的partition(String topic, Object key, byte[] keyBytes,Object value, byte[] valueBytes, Cluster cluster) 方法。后面在生成kafka producer客户端的时候直接指定新的分区类就可以了。

 

 package kafkaconf;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ThreadLocalRandom;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.PartitionInfo;


public class MyParatitioner implements Partitioner {
    @Override
    public void configure(Map<String, ?> configs) {
    }

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        //key不能空，如果key为空的会通过轮询的方式 选择分区
        if(keyBytes == null || (!(key instanceof String))){
            throw new RuntimeException("key is null");
        }
        //获取分区列表
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    
        //以下是上述各种策略的实现，不能共存
        //随机策略
        return ThreadLocalRandom.current().nextInt(partitions.size());
    
        //按消息键保存策略
        return Math.abs(key.hashCode()) % partitions.size();
    
        //自定义分区策略, 比如key为123的消息，选择放入最后一个分区
        if(key.toString().equals("123")){
            return partitions.size()-1;
        }else{
            //否则随机
            ThreadLocalRandom.current().nextInt(partitions.size());
        }
    }
    
    @Override
    public void close() {
    }
 

然后需要在生成kafka producer客户端的时候指定该类就行：

 val properties = new Properties()
    ......
    props.put("partitioner.class", "kafkaconf.MyParatitioner");  //主要这个配置指定分区类
    ......其他配置
    val producer = new KafkaProducer<String, String>(props)



 

2.kafka副本机制

说完了分区，再来说说副本。先说说副本的基本内容，在kafka中，每个主题可以有多个分区，每个分区又可以有多个副本。这多个副本中，只有一个是leader，而其他的都是follower副本。仅有leader副本可以对外提供服务。

多个follower副本通常存放在和leader副本不同的broker中。通过这样的机制实现了高可用，当某台机器挂掉后，其他follower副本也能迅速”转正“，开始对外提供服务。

这里通过问题来整理这部分内容。

Kafka的副本机制

副本，简单来说就是另一个存储文件，比如mysql的只读库就可以看成一个副本，一般使用副本主要有三个方面的优势

能够提供数据的冗余，在宕机或者磁盘损坏的时候，及时恢复数据 能够提供横向的扩展，通过增加机器（类似于mysql的只读库）来提高读操作的吞吐量 *改善数据的局部性，比如cdn的节点数据，将数据落到用户最近的节点上

kafka的副本都有哪些作用？

在kafka中，实现副本的目的就是冗余备份，且仅仅是冗余备份，所有的读写请求都是由leader副本进行处理的。follower副本仅有一个功能，那就是从leader副本拉取消息，尽量让自己跟leader副本的内容一致。

说说follower副本为什么不对外提供服务？

这个问题本质上是对性能和一致性的取舍。试想一下，如果follower副本也对外提供服务那会怎么样呢？首先，性能是肯定会有所提升的。但同时，会出现一系列问题。类似数据库事务中的幻读，脏读。

比如你现在写入一条数据到kafka主题a，消费者b从主题a消费数据，却发现消费不到，因为消费者b去读取的那个分区副本中，最新消息还没写入。而这个时候，另一个消费者c却可以消费到最新那条数据，因为它消费了leader副本。

看吧，为了提高那么些性能而导致出现数据不一致问题，那显然是不值得的。

leader副本挂掉后，如何选举新副本？

如果你对zookeeper选举机制有所了解，就知道zookeeper每次leader节点挂掉时，都会通过内置id，来选举处理了最新事务的那个follower节点。

从结果上来说，kafka分区副本的选举也是类似的，都是选择最新的那个follower副本，但它是通过一个In-sync（ISR）副本集合实现。

kafka会将与leader副本保持同步的副本放到ISR副本集合中。当然，leader副本是一直存在于ISR副本集合中的，在某些特殊情况下，ISR副本中甚至只有leader一个副本。

当leader挂掉时，kakfa通过zookeeper感知到这一情况，在ISR副本中选取新的副本成为leader，对外提供服务。

但这样还有一个问题，前面提到过，有可能ISR副本集合中，只有leader，当leader副本挂掉后，ISR集合就为空，这时候怎么办呢？这时候如果设置unclean.leader.election.enable参数为true，那么kafka会在非同步，也就是不在ISR副本集合中的副本中，选取出副本成为leader，但这样意味这消息会丢失，这又是可用性和一致性的一个取舍了。

ISR副本集合保存的副本的条件是什么？

上面一直说ISR副本集合中的副本就是和leader副本是同步的，那这个同步的标准又是什么呢？

答案其实跟一个参数有关：replica.lag.time.max.ms。

前面说到follower副本的任务，就是从leader副本拉取消息，如果持续拉取速度慢于leader副本写入速度，慢于时间超过replica.lag.time.max.ms后，它就变成“非同步”副本，就会被踢出ISR副本集合中。但后面如何follower副本的速度慢慢提上来，那就又可能会重新加入ISR副本集合中了。

producer的acks参数

前面说了那么多理论的知识，那么就可以来看看如何在实际应用中使用这些知识。

跟副本关系最大的，那自然就是acks机制，acks决定了生产者如何在性能与数据可靠之间做取舍。

配置acks的代码其实很简单，只需要在新建producer的时候多加一个配置：

 

val properties = new Properties()

​    ......

​    props.put("acks", "0/1/-1");  //配置acks，有三个可选值

​    ......其他配置

   val producer = new KafkaProducer<String, String>(props)

 

acks这个配置可以指定三个值，分别是0，1和-1。我们分别来说三者代表什么：

- acks为0：这意味着producer发送数据后，不会等待broker确认，直接发送下一条数据，性能最快
- acks为1：为1意味着producer发送数据后，需要等待leader副本确认接收后，才会发送下一条数据，性能中等
- acks为-1：这个代表的是all，意味着发送的消息写入所有的ISR集合中的副本（注意不是全部副本）后，才会发送下一条数据，性能最慢，但可靠性最强

 

还有一点值得一提，kafka有一个配置参数，min.insync.replicas，默认是1（也就是只有leader，实际生产应该调高），该属性规定了最小的ISR数。这意味着当acks为-1（即all）的时候，这个参数规定了必须写入的ISR集中的副本数，如果没达到，那么producer会产生异常。

但是kafka的副本的作用只是数据冗余的功能，其他的两个功能没有体现出来

kafka 副本可以是一个分区下按照不同的topic进行副本的创建，但是副本只是进行异步的拉取同步数据，不对外提供数据，因此将副本分成领导副本和追随者副本，只有在领导副本挂掉滞后，会在追随者副本中选取领导副本，对外提供服务； 关于追随者副本不对外提供数据的情况，能够提供稳定的对外数据，所写即所得，不会因为读取了不同副本的数据，副本之间数据不统一导致结果不一致； 关于追随者副本是否和leader 副本保持一致的界限，并不是说必须是消息一致才能是一致，而是允许在一个同步间隙内的误差，用 replica.lag.time.max.ms来控制，比如设置的是10秒，kafka允许在追随者副本的消息落后leader副本的时间在10秒之内 In-sync Replicas （ISR） 是记录与leader副本一致的副本集合，是一个动态的集合，因为追随者副本是通过异步拉取数据的，因此在某些情况下可能会不同步，而且落后时间较长，此时isr

记录中会将这个副本移除，在此之后如果副本同步到了一致性的要求的时候，会重新放到isr中 ISR 不只是追随者副本集合，它必然包括 Leader 副本。甚至在某些情况下，ISR 只有 Leader 这一个副本 在leader副本挂掉的时候，isr集合中可能为空，此时如何选取leader呢，需要看参数设置情况，一般说来，如果配置了unclean.leader.election.enable 的话，kafka会在那些不同步的副本中选取副本作为leader副本，以便提供高可用的服务，但是由于这些副本落后很多数据，可能会造成其他业务上的问题，因此一般该参数设置为false，来保证数据的一致性；（在高可用和数据一致性的选择上，倾向于选择一致性，因为高可用的话可以通过其他措施来弥补）

Kafka 是如何处理请求的

- Kafka作为一个高性能的读写消息引擎，所有的连接都是tcp的socket连接，为了提高吞吐量使用了Reactor      模型来处理整个请求和响应，在kafka中将请求分为两大类，数据请求和控制类请求，其中控制类请求的优先级高于数据类请求，控制类请求可以使数据类请求立即失败。
- kafka中数据类请求的请求模型

 

![img](assets/4.1group.jpg)

- 请求进来之后，首先会进入到网络线程池中，这一部分就是使用reactor模型处理的，但是并不是直接响应，而是将数据放到共享队列中，使用一个io线程池去异步的处理共享队列中的请求（将数据放到可以提高整个数据的吞吐量，如果不放到共享队列中的话，大部分读写请求是需要直接处理的，如果直接）
- reactor和线程池之间的区别是什么呢？

Kafka 的高水位和Epoch如何让各个副本异步保持同步状态

高水位 High WaterMark 是标记消息提交和未提交的分界线，在hw之前的数据都可以认为是已提交到kafka上了，否则还认为没有成功提交到kafka上；此处的提交是说认为某条消息全部同步到所有副本上的时候才认为是已提交；

但是hw是有延时的，为了等各个副本的进度，这个值是所有副本中落后的消息数记录，消息还是要不断的提交的leader中的，因此还需要一个缓冲空间来记录消息该写第几个位置，此时出现了LEO (log end Offset),消息可以先写着，kafka之间慢慢同步提交的位置

![img](assets/4.2group.jpg)

那kafka是如何移动自己的hw和leo呢？

leader 是如何移动自己的leo，leader收到producer产生的消息的时候，leo会先位移标志位

其他副本是如何移动自己的leo，副本异步拉去leader中的数据，然后同步自己的leo，正常情况下，这时候leader和副本之间的leo都是一样的.

leader的hw是如何移动的，副本在拉取leader的leo的时候，会告诉leader自己现在的leo在哪，因此上一步的时候，告诉leader自己的leo是0，所以leader的hw不移动，如果一直没有producer产生消息，那么在下一次同步的时候，副本会告诉leader自己的leo是1，然后leader会将所有副本的信息进行一次保存，并获取最小的leo，作为自己的hw；也就是说副本已经将消息同步过去了，最少同步到某个地方了，因此hw是所有副本中的最小leo

副本的hw是怎么移动的呢，副本需要去找leader去同步的获取leader的hw，也就是它需要直到当前所有副本中的最小同步位置在哪，然后在定自己的同步位置应该在什么地方，如果获取到leader的hw的值大于自己的leo的话，那自己就是那个最落后的，因此自己的hw就是自己的leo，否则的话，副本的hw和leader的hw保持一致

副本如果宕机重新启动的时候，在0.1.1.0版本之前呢，启动之后，找不到宕机之前的leo，直接将hw作为leo的最后值，这个认为也是一个比较靠谱的策略，但是为什么不计leo呢，不知道，需要之后进行源码级别的讨论，那么这就会导致，如果副本宕机重启，恢复完之后，开始去拉leader的数据的时候，自己被选为leader了，但是自己的leo却不是最新的，由于leader天生具有权威性，因此会认为自己都是对的，之后原来的leader恢复的时候，要去同步leader的数据，然后将自己的数据更新，发现丢了一些消息，就是副本在恢复的时候副本当时的高水位和原始副本的leo之间的差值

为了改进这个措施，之后版本对这个操作进行了更新，在恢复的时候，首先去leader上获取先leo值，并和自己的进行对比，如果自己落后的话，可能需要更新否则的话不动，因此需要进行记录自己的leo，如果能对这个管理leo的数据进行管理的话，需要有一个确切的版本号来控制，小的版本号会被废弃，这就是epoch的设计原则，能够实时的直到自己到底是领先还是落后的标示，切换leader；具体是有一个单调增加的版本号。每当副本领导权发生变更时，都会增加该版本号。小版本号的 Leader 被认为是过期 Leader，不能再行使 Leader 权力。只要每次有变化，就+1，然后其他重启的数据都和这个保持了一致，所以数据会同步；

副本是否更新leo的值不再依赖hw，而是实实在在的记录值，类似于乐观锁，永远是最新的leo值，所以不会丢数据,当一个副本挂了，如何进行恢复呢？从日志中查询数据，然后逐渐恢复，如果没有标识可能会落后于原始的进度，导致消息的丢失！

