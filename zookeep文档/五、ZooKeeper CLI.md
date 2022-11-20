## 二、ZooKeeper CLI 

**声明：本文根据网上资料整理的学习笔记，严禁商用**。

- ZooKeeper命令行界面（CLI）用于与ZooKeeper集成进行交互以进行开发。这对于调试和使用其他选项很有用。要执行ZooKeeper CLI操作，请首先打开ZooKeeper服务器（“ bin/zkServer.sh start”），然后打开 ZooKeeper客户端（“ bin/zkCli.sh”）。客户端启动后，您可以执行以下操作-

  - 创建znodes
  - 获取数据
  - 监视znode进行更改
  - 设置数据
  - 创建znode的子级
  - 列出znode的子级
  - 检查状态
  - 删除/删除znode

  现在，让我们逐个示例来看上面的命令。

- #### 创建Znodes

  创建具有给定路径的znode。标志参数指定是否创建的Z序节点将是临时的，持久的，或连续的。默认情况下，所有znode都是持久性的。

  - 当会话过期或客户端断开连接时，临时znode（标志：e）将被自动删除。
  - 顺序的znodes保证znode路径是唯一的。
  - ZooKeeper集群将在znode路径中添加序列号以及10位填充。例如，znode路径/myapp将转换为/myapp0000000001，下一个序列号将是/myapp0000000002。如果未指定标志，则将znode视为持久的。

  **语法：**

  ```
  create /path /data
  ```

  复制

  **例子：**

  ```
  create /FirstZnode "Myfirstzookeeper-app"
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 0] create /FirstZnode “Myfirstzookeeper-app”
  Created /FirstZnode
  ```

  复制

  要创建一个**顺序znode**，添加-s标志，如下所示。

  **语法：**

  ```
  create -s /path /data
  ```

  复制

  **例子：**

  ```
  create -s /FirstZnode second-data
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 2] create -s /FirstZnode “second-data”
  Created /FirstZnode0000000023
  ```

  复制

  要创建一个**临时Znode**，添加-e标志，如下所示。

  **语法：**

  ```
  create -e /path /data
  ```

  复制

  **例子：**

  ```
  create -e /SecondZnode "Ephemeral-data"
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 2] create -e /SecondZnode “Ephemeral-data”
  Created /SecondZnode
  ```

  复制

  请记住，当客户端连接断开时，临时znode将被删除。您可以通过退出ZooKeeper CLI并重新打开CLI来尝试。

- #### 获取数据

  它返回znode的关联数据和指定znode的元数据。您将获得诸如上次修改数据的时间，修改的位置以及有关数据的信息。此CLI还用于分配监视以显示有关数据的通知。

  **语法：**

  ```
  get /path
  ```

  复制

  **例子：**

  ```
  get /FirstZnode
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 1] get /FirstZnode
  “Myfirstzookeeper-app”
  cZxid = 0x7f
  ctime = Tue Sep 29 16:15:47 IST 2015
  mZxid = 0x7f
  mtime = Tue Sep 29 16:15:47 IST 2015
  pZxid = 0x7f
  cversion = 0
  dataVersion = 0
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 22
  numChildren = 0
  ```

  复制

  要访问顺序znode，必须输入znode的完整路径。

  **例子：**

  ```
  get /FirstZnode0000000023
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 1] get /FirstZnode0000000023
  “Second-data”
  cZxid = 0x80
  ctime = Tue Sep 29 16:25:47 IST 2015
  mZxid = 0x80
  mtime = Tue Sep 29 16:25:47 IST 2015
  pZxid = 0x80
  cversion = 0
  dataVersion = 0
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 13
  numChildren = 0
  ```

  复制

  **监视**

  当指定的znode或znode的子级数据更改时，监视将显示通知。您只能在get命令中设置监视。

  **语法：**

  ```
  get /path [watch] 1
  ```

  复制

  **例子：**

  ```
  get /FirstZnode 1
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 1] get /FirstZnode 1
  “Myfirstzookeeper-app”
  cZxid = 0x7f
  ctime = Tue Sep 29 16:15:47 IST 2015
  mZxid = 0x7f
  mtime = Tue Sep 29 16:15:47 IST 2015
  pZxid = 0x7f
  cversion = 0
  dataVersion = 0
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 22
  numChildren = 0
  ```

  复制

  输出类似于正常的get命令，但是它将等待znode在后台更改。

- #### 设置数据

  设置指定znode的数据。完成此设置操作后，可以使用get CLI命令检查数据。

  **语法：**

  ```
  set /path /data
  ```

  复制

  **例子：**

  ```
  set /SecondZnode Data-updated
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 1] get /SecondZnode “Data-updated”
  cZxid = 0x82
  ctime = Tue Sep 29 16:29:50 IST 2015
  mZxid = 0x83
  mtime = Tue Sep 29 16:29:50 IST 2015
  pZxid = 0x82
  cversion = 0
  dataVersion = 1
  aclVersion = 0
  ephemeralOwner = 0x15018b47db00000
  dataLength = 14
  numChildren = 0
  ```

  复制

  如果您在get命令中分配了watch选项（如上一条命令中所示），则输出将如下所示(动态的监视到数据的更改)-

  ```
  [zk: localhost:2181(CONNECTED) 1] get /FirstZnode “Mysecondzookeeper-app”
  
  WATCHER: :
  
  WatchedEvent state:SyncConnected type:NodeDataChanged path:/FirstZnode
  cZxid = 0x7f
  ctime = Tue Sep 29 16:15:47 IST 2015
  mZxid = 0x84
  mtime = Tue Sep 29 17:14:47 IST 2015
  pZxid = 0x7f
  cversion = 0
  dataVersion = 1
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 23
  numChildren = 0
  ```

  复制

- #### 创建子代/子znode

  创建子代类似于创建新的znodes。唯一的区别是子znode的路径也将具有父路径。

  **语法：**

  ```
  create /parent/path/subnode/path /data
  ```

  复制

  **例子：**

  ```
  create /FirstZnode/Child1 firstchildren
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 16] create /FirstZnode/Child1 “firstchildren”
  created /FirstZnode/Child1
  [zk: localhost:2181(CONNECTED) 17] create /FirstZnode/Child2 “secondchildren”
  created /FirstZnode/Child2
  ```

  复制

  **列出子代：**

  此命令用于列出和显示znode 的子级。

  **语法：**

  ```
  ls /path
  ```

  复制

  **例子：**

  ```
  ls /MyFirstZnode
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 2] ls /MyFirstZnode
  [mysecondsubnode, myfirstsubnode]
  ```

  复制

- #### 检查状态

  状态描述了指定znode的元数据。它包含详细信息，如时间戳，版本号，ACL，数据长度和子级znode。

  **语法：**

  ```
  stat /path
  ```

  复制

  **例子：**

  ```
  stat /FirstZnode
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 1] stat /FirstZnode
  cZxid = 0x7f
  ctime = Tue Sep 29 16:15:47 IST 2015
  mZxid = 0x7f
  mtime = Tue Sep 29 17:14:24 IST 2015
  pZxid = 0x7f
  cversion = 0
  dataVersion = 1
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 23
  numChildren = 0
  ```

  复制

- #### 删除一个Znode

  删除指定的znode并递归删除其所有子级。仅当此类znode可用时，才会发生这种情况。

  **语法：**

  ```
  rmr /path
  ```

  复制

  **例子：**

  ```
  rmr /FirstZnode
  ```

  复制

  **输出：**

  ```
  [zk: localhost:2181(CONNECTED) 10] rmr /FirstZnode
  [zk: localhost:2181(CONNECTED) 11] get /FirstZnode
  Node does not exist: /FirstZnode
  ```

  复制

  Delete （delete/path）命令类似于remove命令，不同之处在于它仅适用于没有子级的znode。