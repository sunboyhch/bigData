## 六、ZooKeeper API

**声明：本文根据网上资料整理的学习笔记，严禁商用**。

- ZooKeeper具有Java和C的正式API绑定。ZooKeeper社区为大多数语言（.NET，python等）提供了非官方的API。使用ZooKeeper API，应用程序可以连接，交互，操纵数据，协调并最终与ZooKeeper集群断开连接。

  ZooKeeper API具有丰富的功能集，可通过简单安全的方式获得ZooKeeper集群的所有功能。ZooKeeper API提供同步和异步方法。

  ZooKeeper集群和ZooKeeper API在各个方面都完全互补，并且极大地有益于开发人员。让我们在本章中讨论Java绑定。

- #### ZooKeeper API的基础

  与ZooKeeper集群交互的应用程序称为**ZooKeeper客户端**或简称为**Client**。

  Znode是ZooKeeper集群的核心组件，而ZooKeeper API提供了一组方法来使用ZooKeeper集群来操纵znode的所有细节。

  客户端API应遵循以下给出的步骤，与ZooKeeper集群团进行清晰，干净的交互。

  - 连接到ZooKeeper集群。ZooKeeper集群为客户端分配会话ID。
  - 定期向服务器发送心跳。否则，ZooKeeper集群将使会话ID过期，并且客户端需要重新连接。
  - 只要会话ID处于活动状态，就可获取/设置znodes。
  - 一旦完成所有任务，请从ZooKeeper集群断开连接。如果客户端长时间不活动，则ZooKeeper集群将自动断开客户端的连接。

- #### Java绑定

  让我们了解本章中最重要的ZooKeeper API集。ZooKeeper API的中心部分是ZooKeeper类。它提供了在其构造函数中连接ZooKeeper集群的选项，并具有以下方法-

  - **connect** - 连接到ZooKeeper集群
  - **create** - 创建一个znode
  - **exists** - 检查znode是否存在及其信息
  - **getData** - 从特定Znode点获取数据
  - **setData** - 在一个特定的Znode点设定数据
  - **getChildren** - 获取特定Znode中所有可用的子节点
  - **delete** - 获取特定的znode及其所有子节点
  - **close** - 关闭连接

- #### 连接到ZooKeeper集群

  **ZooKeeper** 类通过其构造函数提供连接功能。构造函数的签名如下-

  ```
  ZooKeeper(String connectionString, int sessionTimeout, Watcher watcher)
  ```

  复制

  参数：，

  - **connectionString** - ZooKeeper集群主机。
  - **sessionTimeout** - 会话超时（以毫秒为单位）。
  - **watcher** - 一个实现“Watcher”接口的对象。ZooKeeper集群通过Watcher对象返回连接状态。

  让我们创建ZooKeeperConnection类并添加一个方法connect。该连接方法创建一个ZooKeeper对象，所连接到的ZooKeeper集群，然后返回该对象。在这里，CountDownLatch用于停止（等待）主进程，直到客户端与ZooKeeper集群连接为止。 ZooKeeper集群通过Watcher回调返回连接状态。客户端与ZooKeeper集群连接后，将调用Watcher回调，并且Watcher回调调用CountDownLatch的countDown方法释放锁，并在主进程中等待。

  这是连接ZooKeeper集群的完整代码。

  **例子：**

  ```
  //import java classes
  import java.io.IOException;
  import java.util.concurrent.CountDownLatch;
  
  //import zookeeper classes
  import org.apache.zookeeper.KeeperException;
  import org.apache.zookeeper.WatchedEvent;
  import org.apache.zookeeper.Watcher;
  import org.apache.zookeeper.Watcher.Event.KeeperState;
  import org.apache.zookeeper.ZooKeeper;
  import org.apache.zookeeper.AsyncCallback.StatCallback;
  import org.apache.zookeeper.KeeperException.Code;
  import org.apache.zookeeper.data.Stat;
  
  public class ZooKeeperConnection {
  
     //declare zookeeper instance to access ZooKeeper ensemble
     private ZooKeeper zoo;
     final CountDownLatch connectedSignal = new CountDownLatch(1);
  
     //Method to connect zookeeper ensemble.
     public ZooKeeper connect(String host) throws IOException,InterruptedException {
          
        zoo = new ZooKeeper(host,5000,new Watcher() {
                  
           public void process(WatchedEvent we) {
  
              if (we.getState() == KeeperState.SyncConnected) {
                 connectedSignal.countDown();
              }
           }
        });
                  
        connectedSignal.await();
        return zoo;
     }
  
     //Method to disconnect from zookeeper server
     public void close() throws InterruptedException {
        zoo.close();
     }
  }
  ```

  复制

  保存以上代码，该代码将在下一部分中用于连接ZooKeeper集群。

- #### 创建一个Znode

  ZooKeeper类提供了create方法，以在ZooKeeper集成中创建新的znode。create方法的用法如下-

  ```
  create(String path, byte[] data, List<ACL> acl, CreateMode createMode)
  ```

  复制

  参数：，

  - **path** - Znode路径。例如，/myapp1，/myapp2，/myapp1/mydata1，myapp2/mydata1/myanothersubdata
  - **data** - 数据存储在指定的Znode点路径
  - **acl** - 要创建的节点的访问控制列表。ZooKeeper API提供了一个静态接口ZooDefs.Ids，以获取一些基本的ACL列表。例如，ZooDefs.Ids.OPEN_ACL_UNSAFE返回打开的znode的acl列表。
  - **createMode** - 节点的类型，临时的，序列的或两者兼而有之。这是一个枚举。

  让我们创建一个新的Java应用程序来检查ZooKeeper API 的创建功能。创建一个文件ZKCreate.java。在main方法中，创建类型为ZooKeeperConnection的对象，并调用connect方法以连接到ZooKeeper集群。 connect方法将返回ZooKeeper对象zk。现在，使用自定义路径和数据调用zk对象的create方法。创建znode的完整程序代码如下

  **编码：ZKCreate.java**

  ```
  import java.io.IOException;
  
  import org.apache.zookeeper.WatchedEvent;
  import org.apache.zookeeper.Watcher;
  import org.apache.zookeeper.Watcher.Event.KeeperState;
  import org.apache.zookeeper.ZooKeeper;
  import org.apache.zookeeper.KeeperException;
  import org.apache.zookeeper.CreateMode;
  import org.apache.zookeeper.ZooDefs;
  
  public class ZKCreate {
     // create static instance for zookeeper class.
     private static ZooKeeper zk;
  
     // create static instance for ZooKeeperConnection class.
     private static ZooKeeperConnection conn;
  
     // Method to create znode in zookeeper ensemble
     public static void create(String path, byte[] data) throws 
        KeeperException,InterruptedException {
        zk.create(path, data, ZooDefs.Ids.OPEN_ACL_UNSAFE,
        CreateMode.PERSISTENT);
     }
  
     public static void main(String[] args) {
  
        // znode path
        String path = "/MyFirstZnode"; // Assign path to znode
  
        // data in byte array
        byte[] data = "My first zookeeper app”.getBytes(); // Declare data
                  
        try {
           conn = new ZooKeeperConnection();
           zk = conn.connect("localhost");
           create(path, data); // Create the data to the specified path
           conn.close();
        } catch (Exception e) {
           System.out.println(e.getMessage()); //Catch error message
        }
     }
  }
  ```

  复制

  编译并执行应用程序后，将在ZooKeeper集群中创建具有指定数据的znode。您可以使用ZooKeeper CLI zkCli.sh进行检查。

  ```
  cd /path/to/zookeeper
  bin/zkCli.sh
  >>> get /MyFirstZnode
  ```

  复制

- #### exists – 检查Znode的存在

  ZooKeeper类提供了exists方法来检查znode的存在。如果指定的znode存在，它将返回znode的元数据。exists方法的用法如下-

  **语法：**

  ```
  exists(String path, boolean watcher)
  ```

  复制

  参数：，

  - **path** - Znode路径
  - **watcher** - 布尔值，指定是否要观看特定Z序节点或不看

  让我们创建一个新的Java应用程序来检查ZooKeeper API的“exists”功能。创建一个文件“ ZKExists.java”。在主要方法中，使用“ZooKeeperConnection”对象创建ZooKeeper对象“zk”。然后，使用自定义“path”调用“zk”对象的“ exists”方法。完整的清单如下-

  **编码：ZKExists.java**

  ```
  import java.io.IOException;
  
  import org.apache.zookeeper.ZooKeeper;
  import org.apache.zookeeper.KeeperException;
  import org.apache.zookeeper.WatchedEvent;
  import org.apache.zookeeper.Watcher;
  import org.apache.zookeeper.Watcher.Event.KeeperState;
  import org.apache.zookeeper.data.Stat;
  
  public class ZKExists {
     private static ZooKeeper zk;
     private static ZooKeeperConnection conn;
  
     // Method to check existence of znode and its status, if znode is available.
     public static Stat znode_exists(String path) throws
        KeeperException,InterruptedException {
        return zk.exists(path, true);
     }
  
     public static void main(String[] args) throws InterruptedException,KeeperException {
        String path = "/MyFirstZnode"; // Assign znode to the specified path
                          
        try {
           conn = new ZooKeeperConnection();
           zk = conn.connect("localhost");
           Stat stat = znode_exists(path); // Stat checks the path of the znode
                                  
           if(stat != null) {
              System.out.println("Node exists and the node version is " +
              stat.getVersion());
           } else {
              System.out.println("Node does not exists");
           }
                                  
        } catch(Exception e) {
           System.out.println(e.getMessage()); // Catches error messages
        }
     }
  }
  ```

  复制

  **编译并执行应用程序后，您将获得以下输出。**

  ```
  Node exists and the node version is 1.
  ```

  复制

- #### getData方法

  ZooKeeper类提供了getData方法来获取附加到指定znode中的数据及其状态。getData方法的用法如下-

  **语法：**

  ```
  getData(String path, Watcher watcher, Stat stat)
  ```

  复制

  参数：，

  - **path** - Znode路径。
  - **watcher** - 类型为Watcher的回调函数。当指定的znode的数据发生更改时，ZooKeeper集群将通过Watcher回调进行通知。这是一次性通知。
  - **stat** - 返回Znode点的元数据。

  让我们创建一个新的Java应用程序以了解ZooKeeper API 的getData功能。创建一个文件ZKGetData.java。在main方法中，使用ZooKeeperConnection对象创建一个ZooKeeper对象zk。然后，使用自定义路径调用zk对象的getData方法。

  这是从指定节点获取数据的完整程序代码-

  **编码：ZKGetData.java**

  ```
  import java.io.IOException;
  import java.util.concurrent.CountDownLatch;
  
  import org.apache.zookeeper.ZooKeeper;
  import org.apache.zookeeper.KeeperException;
  import org.apache.zookeeper.WatchedEvent;
  import org.apache.zookeeper.Watcher;
  import org.apache.zookeeper.Watcher.Event.KeeperState;
  import org.apache.zookeeper.data.Stat;
  
  public class ZKGetData {
  
     private static ZooKeeper zk;
     private static ZooKeeperConnection conn;
     public static Stat znode_exists(String path) throws 
        KeeperException,InterruptedException {
        return zk.exists(path,true);
     }
  
     public static void main(String[] args) throws InterruptedException, KeeperException {
        String path = "/MyFirstZnode";
        final CountDownLatch connectedSignal = new CountDownLatch(1);
                  
        try {
           conn = new ZooKeeperConnection();
           zk = conn.connect("localhost");
           Stat stat = znode_exists(path);
                          
           if(stat != null) {
              byte[] b = zk.getData(path, new Watcher() {
                                  
                 public void process(WatchedEvent we) {
                                          
                    if (we.getType() == Event.EventType.None) {
                       switch(we.getState()) {
                          case Expired:
                          connectedSignal.countDown();
                          break;
                       }
                                                          
                    } else {
                       String path = "/MyFirstZnode";
                                                          
                       try {
                          byte[] bn = zk.getData(path,
                          false, null);
                          String data = new String(bn,
                          "UTF-8");
                          System.out.println(data);
                          connectedSignal.countDown();
                                                          
                       } catch(Exception ex) {
                          System.out.println(ex.getMessage());
                       }
                    }
                 }
              }, null);
                                  
              String data = new String(b, "UTF-8");
              System.out.println(data);
              connectedSignal.await();
                                  
           } else {
              System.out.println("Node does not exists");
           }
        } catch(Exception e) {
          System.out.println(e.getMessage());
        }
     }
  }
  ```

  复制

  一旦应用程序被编译并执行，您将获得以下输出

  ```
  My first zookeeper app
  ```

  复制

  应用程序将等待ZooKeeper集群的进一步通知。使用ZooKeeper CLI zkCli.sh更改指定znode的数据。

  ```
  cd /path/to/zookeeper
  bin/zkCli.sh
  >>> set /MyFirstZnode Hello
  ```

  复制

  现在，该应用程序将打印以下输出并退出。

  ```
  Hello
  ```

  复制

- #### setData方法

  ZooKeeper类提供setData方法来修改附加到指定znode中的数据。setData方法的用法如下-

  ```
  setData(String path, byte[] data, int version)
  ```

  复制

  参数：，

  - **path** - Znode路径
  - **data** - 数据存储在指定的Znode点的路径。
  - **version** - znode的当前版本。每当更改数据时，ZooKeeper都会更新znode的版本号。

  现在让我们创建一个新的Java应用程序，以了解ZooKeeper API 的setData功能。创建一个文件ZKSetData.java。在main方法中，使用ZooKeeperConnection对象创建一个ZooKeeper对象zk。然后，使用指定的路径，新数据和节点版本调用zk对象的setData方法。这是修改指定znode中附加数据的完整程序代码。

  **编码：ZKSetData.java**

  ```
  import org.apache.zookeeper.ZooKeeper;
  import org.apache.zookeeper.KeeperException;
  import org.apache.zookeeper.WatchedEvent;
  import org.apache.zookeeper.Watcher;
  import org.apache.zookeeper.Watcher.Event.KeeperState;
  
  import java.io.IOException;
  
  public class ZKSetData {
     private static ZooKeeper zk;
     private static ZooKeeperConnection conn;
  
     // Method to update the data in a znode. Similar to getData but without watcher.
     public static void update(String path, byte[] data) throws
        KeeperException,InterruptedException {
        zk.setData(path, data, zk.exists(path,true).getVersion());
     }
  
     public static void main(String[] args) throws InterruptedException,KeeperException {
        String path= "/MyFirstZnode";
        byte[] data = "Success".getBytes(); //Assign data which is to be updated.
                  
        try {
           conn = new ZooKeeperConnection();
           zk = conn.connect("localhost");
           update(path, data); // Update znode data to the specified path
        } catch(Exception e) {
           System.out.println(e.getMessage());
        }
     }
  }
  ```

  复制

  一旦应用程序被编译和执行，指定znode的数据将被更改，可以使用ZooKeeper CLI zkCli.sh对其进行检查。

  ```
  cd /path/to/zookeeper
  bin/zkCli.sh
  >>> get /MyFirstZnode
  ```

  复制

- #### getChildren方法

  ZooKeeper类提供了getChildren方法来获取特定znode的所有子节点。getChildren方法的用法如下 -

  ```
  getChildren(String path, Watcher watcher)
  ```

  复制

  参数：，

  - **path** - Znode路径。
  - **watcher** - 类型为Watcher的回调函数。当指定的znode的数据发生更改时，ZooKeeper集群将通过Watcher回调进行通知。这是一次性通知。

  让我们创建一个新的Java应用程序以了解ZooKeeper API 的getData功能。创建一个文件ZKGetData.java。在main方法中，使用ZooKeeperConnection对象创建一个ZooKeeper对象zk。然后，使用自定义路径调用zk对象的getData方法。

  这是从指定节点获取数据的完整程序代码-

  **编码：ZKGetChildren.java**

  ```
  import java.io.IOException;
  import java.util.*;
  
  import org.apache.zookeeper.ZooKeeper;
  import org.apache.zookeeper.KeeperException;
  import org.apache.zookeeper.WatchedEvent;
  import org.apache.zookeeper.Watcher;
  import org.apache.zookeeper.Watcher.Event.KeeperState;
  import org.apache.zookeeper.data.Stat;
  
  public class ZKGetChildren {
     private static ZooKeeper zk;
     private static ZooKeeperConnection conn;
  
     // Method to check existence of znode and its status, if znode is available.
     public static Stat znode_exists(String path) throws 
        KeeperException,InterruptedException {
        return zk.exists(path,true);
     }
  
     public static void main(String[] args) throws InterruptedException,KeeperException {
        String path = "/MyFirstZnode"; // Assign path to the znode
                  
        try {
           conn = new ZooKeeperConnection();
           zk = conn.connect("localhost");
           Stat stat = znode_exists(path); // Stat checks the path
  
           if(stat!= null) {
  
              //“getChildren” method- get all the children of znode.It has two
              args, path and watch
              List <String> children = zk.getChildren(path, false);
              for(int i = 0; i < children.size(); i++)
              System.out.println(children.get(i)); //Print children's
           } else {
              System.out.println("Node does not exists");
           }
  
        } catch(Exception e) {
           System.out.println(e.getMessage());
        }
  
     }
  
  }
  ```

  复制

  在运行程序之前，让我们使用ZooKeeper CLI zkCli.sh为/MyFirstZnode创建两个子节点。

  ```
  cd /path/to/zookeeper
  bin/zkCli.sh
  >>> create /MyFirstZnode/myfirstsubnode Hi
  >>> create /MyFirstZnode/mysecondsubmode Hi
  ```

  复制

  现在，编译并运行该程序将输出上面创建的znodes。

  ```
  myfirstsubnode
  mysecondsubnode
  ```

  复制

- #### 删除Znode

  ZooKeeper类提供了delete方法来删除指定的znode。delete方法的用法如下-

  ```
  delete(String path, int version)
  ```

  复制

  参数：，

  - **path** - Znode路径。
  - **version** - znode的当前版本。

  让我们创建一个新的Java应用程序以了解ZooKeeper API 的删除功能。创建一个文件ZKDelete.java。在main方法中，使用ZooKeeperConnection对象创建一个ZooKeeper对象zk。然后，使用节点的指定路径和版本调用zk对象的delete方法。删除znode的完整程序代码如下-

  **编码：ZKDelete.java**

  ```
  import org.apache.zookeeper.ZooKeeper;
  import org.apache.zookeeper.KeeperException;
  
  public class ZKDelete {
     private static ZooKeeper zk;
     private static ZooKeeperConnection conn;
  
     // Method to check existence of znode and its status, if znode is available.
     public static void delete(String path) throws KeeperException,InterruptedException {
        zk.delete(path,zk.exists(path,true).getVersion());
     }
  
     public static void main(String[] args) throws InterruptedException,KeeperException {
        String path = "/MyFirstZnode"; //Assign path to the znode
                  
        try {
           conn = new ZooKeeperConnection();
           zk = conn.connect("localhost");
           delete(path); //delete the node with the specified path
        } catch(Exception e) {
           System.out.println(e.getMessage()); // catches error messages
        }
     }
  
  ```