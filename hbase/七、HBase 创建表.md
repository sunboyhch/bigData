# 七、HBase 创建表  

**声明：本文根据网上资料整理的学习笔记，严禁商用**。

您可以使用create命令创建表，在这里您必须指定表名和列族名。在HBase Shell中创建表的语法如下所示。 

create '<table name>','<column family>'

例 - 下面给出的是一个名为emp的表的示例架构。它有两个列族：“personal data”和“professional data”。

Row key        personal data        professional data

​                   

​                   

您可以在HBase Shell中创建此表，如下所示。

hbase(main):002:0> create 'emp', 'personal data', 'professional data'

它将为您提供以下输出。

 

0 row(s) in 1.1300 seconds

=> Hbase::Table - emp

验证

您可以验证是否使用list命令创建了该表，如下所示。在这里，您可以观察创建的emp表。 

hbase(main):002:0> list

TABLE 

emp

2 row(s) in 0.0340 seconds

 

**使用Java API创建表**

您可以使用HBaseAdmin类的createTable()方法在HBase中创建表。此类属于org.apache.hadoop.hbase.client软件包。下面给出了使用Java API在HBase中创建表的步骤。

 

步骤1：实例化HBaseAdmin

此类需要使用Configuration对象作为参数，因此首先实例化Configuration类并将此实例传递给HBaseAdmin。 

Configuration conf = HBaseConfiguration.create();

HBaseAdmin admin = new HBaseAdmin(conf);

 

步骤2：建立TableDescriptor

HTableDescriptor是一个属于org.apache.hadoop.hbase类的类。此类类似于表名和列族的容器。 

//creating table descriptor

HTableDescriptor table = new HTableDescriptor(toBytes("Table name")); 

//creating column family descriptor

HColumnDescriptor family = new HColumnDescriptor(toBytes("column family")); 

//adding coloumn family to HTable

table.addFamily(family);

 

步骤3：通过管理员执行

使用HBaseAdmin类的createTable（）方法，可以在Admin模式下执行创建的表。 

admin.createTable(table);

 

 

下面给出的是通过admin创建表的完整程序。 

import java.io.IOException;

import org.apache.hadoop.hbase.HBaseConfiguration;

import org.apache.hadoop.hbase.HColumnDescriptor;

import org.apache.hadoop.hbase.HTableDescriptor;

import org.apache.hadoop.hbase.client.Connection;

import org.apache.hadoop.hbase.client.ConnectionFactory;

import org.apache.hadoop.hbase.client.Admin;

import org.apache.hadoop.hbase.TableName;

 

import org.apache.hadoop.conf.Configuration;

@SuppressWarnings("deprecation")

public class CreateTable {

 

   public static void main(String[] args) throws IOException {

​      try {

​         // Instantiating configuration class

​         Configuration config = HBaseConfiguration.create();

​         Connection connection = ConnectionFactory.createConnection(config);

​         // Instantiating HbaseAdmin class

​         Admin admin  = connection.getAdmin();

 

​         // Instantiating table descriptor class

​         HTableDescriptor tableDescriptor = new HTableDescriptor(TableName.valueOf("empfromjava"));

 

​         // Adding column families to table descriptor

​         tableDescriptor.addFamily(new HColumnDescriptor("personal"));

​         tableDescriptor.addFamily(new HColumnDescriptor("professional"));

 

​         // Execute the table through admin

​         admin.createTable(tableDescriptor);

​         System.out.println(" Table created ");

​      } catch (Exception e) {

​         System.out.println(e.getMessage());

​      }

   }

}

编译并执行上述程序，如下所示。

 

$javac CreateTable.java

$java CreateTable

以下应该是输出：

 

Table created