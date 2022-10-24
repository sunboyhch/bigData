# 四、HBase Rowkey 设计和实现方式 

**声明：本文根据网上资料整理的学习笔记，严禁商用**。

**一、为什么 Rowkey 这么重要** 

HBase 由于它存储和读写的高性能，在 OLAP 即时分析中发挥着重要的作用。而 RowKey 作为 HBase 的核心知识点，其设计势必会影响到数据在 HBase 中的分布，还会影响我们查询效率，可以说 RowKey 的设计质量关乎了 HBase 的质量。

![](assets/4.1-region-rowkey.jpg)

对于关系型数据库，数据定位可以理解为“二维坐标”；但在 HBase 中，定位一条数据（即一个 Cell）我们需要 4 个维度的限定：行键（RowKey）、列族（Column Family）、列限定符（Column Qualifier）、时间戳（Timestamp）。其中，RowKey 是最容易出现问题的。除了根据业务和查询需求来设计之外，还有很多地方需要我们注意。

我们常说看一张 HBase 表设计的好不好，就看它的 RowKey 设计的好不好。可见 RowKey 在 HBase 中的地位。那么 RowKey 到底是什么?RowKey 的特点 如下:

类似于 MySQL、Oracle 中的主键，用于标示唯一的行;

完全是由用户指定的一串不重复的字符串;

HBase 中的数据永远是根据 Rowkey 的字典排序来排序的。

1.2 RowKey 的作用 

读写数据时通过 RowKey 找到对应的 Region;

MemStore 中的数据按 RowKey 字典顺序排序;

HFile 中的数据按 RowKey 字典顺序排序。

1.3 Rowkey 对查询的影响

如果我们的 RowKey 设计为 uid+phone+name，那么这种设计可以很好的支持

以下的场景:

uid = 111 AND phone = 123 AND name = iteblog uid = 111 AND phone = 123

uid = 111 AND phone = 12?

uid = 111

难以支持的场景:

phone = 123 AND name = iteblog phone = 123

name = iteblog

**二、RowKey 概念**

***HBase 中 RowKey 可以唯一标识一行记录，在 HBase 查询的时候有以下三种方式：***

1、通过 get 方式，指定 RowKey 获取唯一一条记录

2、通过 scan 方式，设置 startRow 和 stopRow 参数进行范围匹配

3、全表扫描，即直接扫描整张表中所有行记录 

***HBase的查询实现只提供两种方式：***

1、按指定RowKey获取唯一一条记录，get方法（org.apache.hadoop.hbase.client.Get）

2、按指定的条件获取一批记录，scan方法（org.apache.hadoop.hbase.client.Scan）

实现条件查询功能使用的就是scan方式，scan在使用时有以下几点值得注意：

1、scan可以通过setCaching与setBatch方法提高速度（以空间换时间）；

2、scan可以通过setStartRow与setEndRow来限定范围。范围越小，性能越高。

通过巧妙的RowKey设计使我们批量获取记录集合中的元素挨在一起（应该在同一个Region下），可以在遍历结果时获得很好的性能。

3、scan可以通过setFilter方法添加过滤器，这也是分页、多条件查询的基础。



从字面意思来看，RowKey 就是行键的意思，在曾删改查的过程中充当了主键的作用。它可以是任意字符串，在 HBase 内部 RowKey 保存为字节数组。

HBase 中的数据是按照 Rowkey 的 ASCII 字典顺序进行全局排序的，有伙伴可能对 ASCII 字典序印象不够深刻，下面举例说明：

假如有 5 个 Rowkey："012", "0", "123", "234", "3"，按 ASCII 字典排序后的结果为："0", "012","123", "234", "3"。

因此我们设计 RowKey 时，需要充分利用排序存储这个特性，将经常一起读取的行存储放到一起，要避免做全表扫描，因为效率特别低。

![](assets/4.2-scan-rowkey.jpg)

**三、什么是数据热点？**

3.1 热点现象产生

HBase 中的行是按照 Rowkey 的字典顺序排序的，这种设计优化了 scan 操作，可以将相关的行以及会被一起读取的行存取在临近位置，便于 scan。

然而糟糕的 Rowkey 设计是热点的源头。 热点发生在大量的 client 直接访问集群的一个或极少数个节点（访问可能是读，写或者其他操作）。

大量访问会使热点 region 所在的单个机器超出自身承受能力，引起性能下降甚至 region 不可用，这也会影响同一个 RegionServer 上的其他 region，由于主机无法服务其他 region 的请求，这样就造成数据热点现象。 （这一点其实和数据倾斜类似）

所以我们在向 HBase 中插入数据的时候，应优化 RowKey 的设计，使数据被写入集群的多个 region，而不是一个。尽量均衡地把记录分散到不同的 Region 中去，平衡每个 Region 的压力。

3.2 避免数据热点的方法

在日常使用中，主要有 3 个方法来避免热点现象，分别是反转，加盐和哈希，下面咱们逐个举例分析：

3.2.1 反转（Reversing）

第一种咱们要分析的方法是反转，顾名思义它就是把固定长度或者数字格式的 RowKey 进行反转，反转分为一般数据反转和时间戳反转，其中以时间戳反转较常见。

反转固定格式的数值以手机号为例，手机号的前缀变化比较少（如152、185等），但后半部分变化很多。如果将它反转过来，可以有效地避免热点。不过其缺点就是失去了有序性。

反转时间这个操作严格来讲不算“打散”，但可以调整数据的时间排序。如果将时间按照字典序排列，最近产生的数据会排在旧数据后面。如果用一个大值减去时间（比如用99999999减去yyyyMMdd，或者Long.MAX_VALUE减去时间戳），最新的数据就可以排在前面了。

3.2.2 加盐（Salting）

这里的“加盐”与密码学中的“加盐”不是一回事。它是指在 RowKey 的前面增加一些前缀，加盐的前缀种类越多，RowKey 就被打得越散。

需要注意的是分配的随机前缀的种类数量应该和我们想把数据分散到的那些 region 的数量一致。只有这样，加盐之后的 rowkey 才会根据随机生成的前缀分散到各个 region 中，避免了热点现象。

3.2.3 哈希（Hashing）

其实哈希和加盐的使用场景类似，但我们前缀不可以是随机的，因为必须要让客户端能够完整地重构 RowKey。所以一般会拿原 RowKey 或其一部分计算 Hash 值，然后再对 Hash 值做运算作为前缀。

四、RowKey 的设计原则

通过前面的分析我们应该知道了 HBase 中 RowKey 设计的重要性了，为了帮助我们设计出完美的 RowKey，HBase 提出了 RowKey 的设计原则主要有以下四点：长度原则、唯一原则、排序原则、散列原则。

**4.1 RowKey 长度原则**

![](assets/4.3-length-rowkey.png)

RowKey 是一个二进制码流，可以是任意字符串，最大长度 64kb ，实际应用中一般为 10-100bytes，以 byte[] 形式保存，一般设计成定长。建议越短越好，不要超过 16 个字节，原因如下：

在 HBase 的底层存储 HFile 中，RowKey 是 KeyValue 结构中的一个域。假设 RowKey 长度 100B，那么 1000 万条数据中，光 RowKey 就占用掉100*1000w=10亿个字节将近 1G 空间，这样会极大影响 HFile 的存储效率。

![](assets/4.4-ef-rowkey.png)

HBase 中设计有 MemStore 和 BlockCache，分别对应列族/Store 级别的写入缓存，和 RegionServer 级别的读取缓存。如果 RowKey 字段过长，内存的有效利用率就会降低，系统不能缓存更多的数据，这样会降低检索效率。

另外，我们目前使用的服务器操作系统都是 64 位系统，内存是按照 8B 对齐的，因此设计 RowKey 时一般做成 8B 的整数倍，如 16B 或者 24B，可以提高寻址效率。

同样地，列族、列名的命名在保证可读的情况下也应尽量短。value 永远和它的 key 一起传输的。当具体的值在系统间传输时，它的 RowKey，列名，时间戳也会一起传输（因此实际上列族命名几乎都用一个字母，比如‘c’或‘f’）。如果你的 RowKey 和列名和值相比较很大，那么你将会遇到一些有趣的问题。Hfile 中的索引最终占据了 HBase 分配的大量内存。

4.2 唯一原则

其实唯一原则咱们可以结合 HashMap 的源码设计或者主键的概念来理解，由于 RowKey 用来唯一标识一行记录，所以必须在设计上保证 RowKey 的唯一性。

需要注意：由于 HBase 中数据存储的格式是 Key-Value 对格式，所以如果向 HBase 中同一张表插入相同 RowKey 的数据，则原先存在的数据会被新的数据给覆盖掉（和 HashMap 效果相同）。

4.3 排序原则

RowKey 是按照字典顺序排序存储的，因此，设计 RowKey 的时候，要充分利用这个排序的特点，将经常读取的数据存储到一块，将最近可能会被访问的数据放到一块。

一个常见的数据处理问题是快速获取数据的最近版本，使用反转的时间戳作为 RowKey 的一部分对这个问题十分有用，可以用Long.Max_Value-timestamp追加到key的末尾。

例如key , [key]的最新值可以通过scan [key]获得[key]的第一条记录，因为 HBase 中 RowKey 是有序的，第一条记录是最后录入的数据。

4.4 散列原则

散列原则用大白话来讲就是咱们设计出的 RowKey 需要能够均匀的分布到各个 RegionServer 上。

比如设计 RowKey 的时候，当 Rowkey 是按时间戳的方式递增，就不要将时间放在二进制码的前面，可以将 Rowkey 的高位作为散列字段，由程序循环生成，可以在低位放时间字段，这样就可以提高数据均衡分布在每个 Regionserver 实现负载均衡的几率。

结合前面分析的热点现象的起因思考： 如果没有散列字段，首字段只有时间信息，那就会出现所有新数据都在一个 RegionServer 上堆积的热点现象，这样在做数据检索的时候负载将会集中在个别 RegionServer 上，降低查询效率。

**五、举例**

5.RowKey 设计案例剖析 

5.1 交易类表 Rowkey 设计

查询某个卖家某段时间内的交易记录

sellerId + timestamp + orderId

查询某个买家某段时间内的交易记录

buyerId + timestamp +orderId

 

根据订单号查询 orderNo

如果某个商家卖了很多商品，可以如下设计 Rowkey 实现快速搜索salt+sellerId + timestamp 其中，salt 是随机数。

可以支持的场景: 

全表 Scan

按照 sellerId 查询

按照 sellerId + timestamp 查询

5.2 金融风控 Rowkey 设计 

查询某个用户的用户画像数据 

prefix + uid

prefix + idcard

prefix + tele

其中 prefix = substr(md5(uid),0 ,x)， x 取 5-6。uid、idcard 以及 tele 分别表示 用户唯一标识符、身份证、手机号码。

5.3 车联网 Rowkey 设计 查询某辆车在某个时间范围的交易记录

carId + timestamp

某批次的车太多，造成热点

prefix + carId + timestamp 其中 prefix = substr(md5(uid),0 ,x)

 

5.4 查询最近的数据

查询用户最新的操作记录或者查询用户某段时间的操作记录，RowKey 设计如下: uid + Long.Max_Value - timestamp

支持的场景 

查询用户最新的操作记录

Scan [uid] startRow uid stopRow uid

查询用户某段时间的操作记录

Scan [uid] startRow uid stopRow uid

 

如果 RowKey 无法满足我们的需求，可以尝试二级索引。Phoenix、Solr 以及 ElasticSearch 都可以用于构建二级索引。