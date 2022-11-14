# 九、HBase 列出表  

**声明：本文根据网上资料整理的学习笔记，严禁商用**。

使用HBase Shell列出表

list是用于列出HBase中所有表的命令。

当您键入此命令并在HBase提示符下执行时，它将显示HBase中所有表的列表，如下所示。

hbase(main):001:0> list

TABLE

emp

在这里，您可以观察到一个名为emp的表。



**使用Java API列出表**

请按照下面给出的步骤使用Java API从HBase获取表列表。 

第1步

在HBaseAdmin类中，有一个名为listTables()的方法来获取HBase中所有表的列表。此方法返回HTableDescriptor对象的数组。

 

// Instantiating configuration class

Configuration config = HBaseConfiguration.create();

Connection connection = ConnectionFactory.createConnection(config);

// Instantiating HbaseAdmin class

Admin admin = connection.getAdmin();

 

//Getting all the list of tables using HBaseAdmin object

HTableDescriptor[] tableDescriptor = admin.listTables();

 

第2步

您可以使用HTableDescriptor类的length变量来获取HTableDescriptor []数组的长度。使用getNameAsString（）方法从此对象获取表的名称。使用它们运行“ for”循环，并获取HBase中的表列表。

下面给出了使用Java API列出HBase中所有表的程序。

 

import java.io.IOException;

import org.apache.hadoop.hbase.HBaseConfiguration;

import org.apache.hadoop.hbase.HTableDescriptor;

import org.apache.hadoop.hbase.client.Connection;

import org.apache.hadoop.hbase.client.ConnectionFactory;

import org.apache.hadoop.hbase.client.Admin;

import org.apache.hadoop.conf.Configuration;

@SuppressWarnings("deprecation")

public class CreateTable {

   public static void main(String[] args) throws IOException {

​      try {

​         // Instantiating configuration class

​         Configuration config = HBaseConfiguration.create();

​         Connection connection = ConnectionFactory.createConnection(config);

​         // Instantiating HbaseAdmin class

​         Admin admin= connection.getAdmin();

 

​         // Getting all the list of tables using HBaseAdmin object

​         HTableDescriptor[] tableDescriptor = admin.listTables();

 

​         // printing all the table names.

​         for (int i = 0; i < tableDescriptor.length; i++) {

​            System.out.println(tableDescriptor[i].getNameAsString());

​         }

​      } catch (Exception e) {

​         System.out.println(e.getMessage());

​      }

   }

}

编译并执行上述程序，如下所示。

 

$javac ListTables.java

$java ListTables

以下应该是输出：

 

emp

empfromjava