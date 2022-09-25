---
title: MyBatis-3
date: 2022-09-25 15:57:09
top_img: /img/mybatis/drivers.png
cover: /img/mybatis/drivers.png
tags: [MyBatis, 笔记]
---

# MyBatis 源码阅读
第一篇记录了MyBatis的核心组成和工作逻辑，本篇再添加一些细节

## 数据源
代码位于`datasource`包，只对关键部分摘抄，加深印象

### java.sql包和javax.sql包
#### java.sql包
`java.sql`通常被称为JDBC核心API包，它为Java提供了访问数据源中数据的基础功能

`java.sql`提供了一个`Driver`接口作为数据库驱动的接口，以及`DriverManager`类管理一组驱动。

不同种类的数据库厂商只需根据自身数据库特点开发相应的`Driver`实现类，并通过`DriverManager`进行注册即可

<center>
    <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/mybatis/drivers.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
        Driver接口相关类
    </div>
</center>

基于`java.sql`包，Java程序能够完成各种数据库操作。通常完成一次数据库操作的流程如下所示:
+ 建立`DriverManager`对象
+ 从`DriverManager`对象中获取`Connection`对象
+ 从`Connection`对象中获取`Statement`对象
+ 将SQL语句交给`Statement`对象执行，并获取返回的结果，结果通常放在`ResultSet`中

#### javax.sql包
`javax.sql`通常被称为JDBC扩展API包，它扩展了JDBC核心API包的功能，提供了对服务器端的支持，是 Java企业版的重要部分

`javax.sql`提供了`DataSource`接口，通过它可以获取面向数据源的`Connection`对象，与`java.sql`中直接使用`DriverManager`建立连接的方式相比更为灵活（实际上，`DataSource`接口的实现中也是通过`DriverManager`对象获取`Connection`对象的）

使用`javax.sql`的`DataSource`类执行一条SQL语句的过程如下:
+ **建立`DataSource`对象**
+ **从`DataSource`对象中获取`Connection`对象**
+ **从`Connection`对象中获取`Statement`对象**
+ **将SQL语句交给`Statement`对象执行，并获取返回的结果，结果通常放在`ResultSet`中**

### DriverManager
JDBC驱动程序管理器，可以管理一组JDBC驱动程序

主要的静态方法：
+ void registerDriver: 向`DriverManager`中注册给定的驱动程序
+ void deregisterDriver: 从`DriverManager`中删除给定的驱动程序
+ Driver getDriver: 查找能匹配给定URL路径的驱动程序
+ Enumeration getDrivers: 获取当前调用者可以访问的所有已加载的JDBC驱动程序
+ **Connection getConnection: 建立到给定数据库的连接**

### DataSource
`DataSource`是`javax.sql`的一个**接口**，代表了一个实际的数据源，其功能是作为工厂提供数据库连接

只有以下两个接口方法，都用来获取一个`Connection`对象:
+ **getConnection()**: 从当前的数据源中建立一个连接
+ **getConnection(String,String)**: 从当前的数据源中建立一个连接，输入的参数为数据源的用户名和密码

常见实现:
+ 基本实现(`UnpooledDataSource`): 生成基本的到数据库的连接对象`Connection`
+ 连接池实现(`PoolDataSource`): 生成的`Connection`对象能够自动加到连接池中
+ 分布式事务实现(): 生成的`Connection`对象能够参与分布式事务

### Connection
`Connection`**接口**，代表对某个数据库的连接，执行SQL语句和获取结果

常用方法:
+ **Statement createStatement**: 创建一个`Statement`对象，通过它能将SQL语句发送到数据库
+ **CallableStatement prepareCall**: 创建一个`CallableStatement`对象，通过它能调用存储过程
+ **PreparedStatement prepareStatement**: 创建一个`PreparedStatement`对象，通过它能将参数化的SQL语句发送到数据库
+ void commit: 提交当前事务
+ void rollback: 回滚当前事务
+ void close: 关闭当前的 Connection对象

### Statement
`Statement`**接口**，定义了一些抽象方法能用来执行静态SQL语句并返回结果。通常`Statement`对象会返回一个结果集对象`ResultSet`

`Statement`、`PreparedStatement`和`CallableStatement`接口的区别:
+ `Statement`: 提供了执行语句和获取结果的基本方法，支持**普通的不带参的查询SQL**
+ `PreparedStatement`: 添加了处理输入参数的方法，支持**可变参数的SQL**
+ `CallableStatement`: 添加了调用存储过程核函数以及处理输出参数的方法，用于**执行对数据库已存储过程的调用**

`Statement`每次执行sql语句，数据库都要执行sql语句的编译，而`PreparedStatement`只会在第一次进行编译，后续仅需要解析然后填入参数

`PreparedStatement`还有效避免了SQL注入：
+ `${}`: 以`Statement`执行，将输入参数直接拼接到SQL中，如下情况就会发生SQL注入，导致必然执行
    ```sql
    "select * from table where id=${}" + "3 or 2==2"
    ==> "select * from table where id=3 or 2==2"
    ```
+ `#{}`: 以`PreparedStatement`执行，先预编译，再将输入参数以字符串直接赋值，如下，不会发生SQL注入
    ```sql
    "select * from table where id=#{}" + "3 or 2==2"
    ==> "select * from table where id='3 or 2==2'"
    ```

## 参考
《MyBatis源码详解——通用源码阅读指导书》