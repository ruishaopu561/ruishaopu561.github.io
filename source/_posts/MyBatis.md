---
title: MyBatis
date: 2022-09-21 15:42:50
top_img: /img/cover/mybatis.jpeg
cover: /img/cover/mybatis.jpeg
tags: [MyBatis, 笔记]
---

# MyBatis 源码阅读
与 ORM 框架不同，MyBatis通过将Java方法与数据库SQL语句关联起来，从另一个角度实现了Java服务对数据库的操作

## 核心功能
+ **将包含`if`等标签的复杂数据库操作语句解析为纯粹的 SQL 语句**
+ **将数据库操作节点和映射接口中的抽象方法进行绑定，在抽象方法被调用时执行数据库操作**
+ **将输入参数对象转化为数据库操作语句中的参数**
+ **将数据库操作语句的返回结果转化为对象**

## 文件组成
### 全局配置文件
MyBatis 的配置文件为一个 XML 文件，通常被命名为 `mybatis-config.xml` ，该 XML文件的根节点为 `configuration` ，根节点内可以包含的一级节点及其含义如下所示

+ **properties**: 属性信息，相当于 MyBatis 的全局变量
+ **settings**: 设置信息，通过它对 MyBatis 的功能进行调整
+ **typeAliases**: 类型别名，在这里可以为类型设置一些简短的名字
+ **typeHandlers**: 类型处理器，在这里可以为不同的类型设置相应的处理器
+ **objectFactory**: 对象工厂，在这里可以指定 MyBatis 创建新对象时使用的工厂
+ **objectWrapperFactory**: 对象包装器工厂，在这里可以指定 MyBatis 使用的对象包装器工厂
+ **reflectorFactory**: 反射器工厂，在这里可以设置 MyBatis 的反射器工厂
+ **plugins**: 插件，在这里可以为 MyBatis 配置插件，从而修改或扩展 MyBatis 的行为 
+ **environments**: 环境，这里可以配置 MyBatis 运行的环境信息，如数据源信息等
+ **databaseIdProvider**: 数据库编号，在这里可以为不同的数据库配置不同的编号，这样可以对不同类型的数据库设置不同的数据库操作语句
+ **mappers**: 映射文件，在这里可以配置映射文件或映射接口文件的地址

> **注意**：配置文件中的一级节点是有顺序要求的，这些节点必须按照上面列举 的顺序出现。在使用中可以根据实际需要选择相应的节点依次写入配置文件。

示例：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <typeAliases>
        <package name="com.github.yeecode.mybatisdemo"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
                <dataSource type="POOLED">
                    <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                    <property name="url" value="jdbc:mysql://127.0.0.1:3306/yeecode?serverTimezone=UTC"/>
                    <property name="username" value="root"/>
                    <property name="password" value="yeecode"/>
                </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="com/github/yeecode/mybatisdemo/UserMapper.xml"/>
    </mappers>
</configuration>
```

### 映射文件
映射文件也是一个 XML 文件，用来完成Java方法与SQL语句的映射、Java对象与SQL参数的映射、SQL查询结果与Java对象的映射等

映射文件的根节点为 `mapper` ，在 `mapper` 节点下可以包含的节点及其含义如下所示

+ **cache**: 缓存，通过它可以对当前命名空间进行缓存配置
+ **cache-ref**: 缓存引用，通过它可以引用其他命名空间的缓存作为当前命名空间的缓存
+ **resultMap**: 结果映射，通过它来配置如何将 SQL 查询结果映射为对象
+ **parameterMap**: 参数映射，通过它来配置如何将参数对象映射为 SQL参数。该节点已废弃，建议直接使用内联参数
+ **sql**: SQL语句片段，通过它来设置可以被复用的语句片段
+ **insert**: 插入语句
+ **update**: 更新语句
+ **delete**: 删除语句
+ **select**: 查询语句

示例：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper   PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.github.yeecode.mybatisdemo.UserMapper">
    <!-- 数据库操作节点 -->
    <select id="queryUserBySchoolName" resultType="com.github.yeecode.mybatisdemo.User">
        <!-- 数据库操作语句 -->
        SELECT * FROM `user`
        <if test="schoolName != null">
            WHERE schoolName = #{schoolName}
        </if>
    </select>
</mapper>
```

### 映射接口文件
映射接口文件是一个Java接口文件，并且该接口不需要实现类。通常情况下，每个映射接口文件都有一个同名的映射文件与之相对应。

在映射接口文件中可以定义一些抽象方法，这些抽象方法可以分为两类：

+ 第一类抽象方法与对应的**映射文件**中的数据库操作节点相对应
+ 第二类抽象方法通过**注解声明**自身的数据库操作语句

> **注意**：当整个接口文件中均为注解类抽象方法时，则该映射接口文件可以没有对应的映射文件

示例：
```java
public interface UserMapper {
    // 该抽象方法对应着映射文件中的数据库操作节点
    List<User> queryUserBySchoolName(User user);

    // 该抽象方法通过注解声明自身的数据库操作语句
    @Select("SELECT * FROM `user` WHERE `id`=#{id}")
    User queryUserById(Integer id);
}
```

## 工作过程
项目运行中， MyBatis 操作大致分为两个阶段：

+ MyBatis初始化阶段：MyBatis启动时运行一次，完成MyBatis运行环境的**配置工作**
+ 数据读写阶段：由数据读写操作触发，完成增删改查请求

### 初始化阶段
MyBatis的初始化在整个项目启动时开始执行，主要完成**配置文件的解析**、**数据库的连接**等操作

#### 配置信息读取
读取配置文件`mybatis-config.xml`得到`InputStream`，通过调用`SqlSessionFactoryBuilder`类的`build()`方法，生成`SqlSessionFactory`对象。

演示用户代码如下：
```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

其中`SqlSessionFactoryBuilder`类的`build()`核心方法如下，方法内创建`XMLConfigBuilder`类并调用`parse()`方法解析`mybatis-config.xml`得到`configuration`对象，也就是记录MyBatis全部配置信息的全局配置对象
```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
        XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
        return build(parser.parse());
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
        ErrorContext.instance().reset();
        try {
            inputStream.close();
        } catch (IOException e) {
        }
    }
}

// 上面方法中实际创建 SqlSessionFactory 实例的方法
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```

`XMLConfigBuilder`类的`parse()`方法如下，解析过程固定，因此前文强调必须按顺序写
```java
public Configuration parse() {
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}

// 核心解析方法
private void parseConfiguration(XNode root) {
    try {
        //issue #117 read properties first
        propertiesElement(root.evalNode("properties"));
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        loadCustomLogImpl(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

`DefaultSqlSessionFactory`类是`SqlSessionFactory`接口的默认实现类，其中保存了`configuration`对象
```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {

    private final Configuration configuration;

    public DefaultSqlSessionFactory(Configuration configuration) {
        this.configuration = configuration;
    }
    // 核心方法之一，创建SqlSession对象
    @Override
    public SqlSession openSession() {
        return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
    }

    ...
}
```

> `SqlSessionFactory`对象是生产`SqlSession`对象的工厂，`SqlSession`对象是数据读写阶段的操作接口，详见下文

### 数据读写阶段
后续分析各个包的作用过程中，请**牢牢记住本节的内容，把握主线，串联细节，融合贯通**

#### 获取SqlSession对象
演示用户代码如下：
```java
SqlSession session = sqlSessionFactory.openSession()
```

`DefaultSqlSessionFactory`类中`openSession()`的具体实现如下：
```java
// 核心方法之一，创建SqlSession对象
@Override
public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level,
                                            boolean autoCommit) {
    Transaction tx = null;
    try {
        // 找出要使用的指定环境
        final Environment environment = configuration.getEnvironment();
        // 从环境中获取事务工厂
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        // 从事务工厂中生产事务
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        // 创建执行器
        final Executor executor = configuration.newExecutor(tx, execType);
        // 创建DefaultSqlSession对象
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        closeTransaction(tx); // may have fetched a connection so lets call close()
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

`DefaultSqlSession`类是`SqlSession`接口的实现类，它提供了查询、增加、更新、删除、提交、回滚等大量的方法


> **注意**: 数据读写阶段是在进行数据读写时触发的，但并不是每次读写都会触发
> “SqlSession session=sqlSessionFactory.openSession()”操作
> 因为该操作得到的 DefaultSqlSession 对象可以供多次数据库读写操作复用

#### 映射接口文件与映射文件的绑定
演示代码如下，通过调用`DefaultSqlSession`类的`getMapper(Class<T>)`方法获取`UserMapper`对象
```java
// 找到接口对应的实现
UserMapper userMapper = session.getMapper(UserMapper.class);
```

`DefaultSqlSession`类的`getMapper(Class<T>)`方法经过`configuration`对象转交，最终到`MapperRegistry`类中的`getMapper(Class<T>, SqlSession)`方法

`MapperRegistry`类记录了配置文件中所有的映射文件路径，给定java类名后，`getMapper(Class<T>, SqlSession)`方法通过映射接口信息从所有已经解析的映射文件中找到对应的映射文件，然后根据该映射文件组建并返回接口的一个实现对象（代理对象）

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 找出指定映射接口的代理工厂
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        // 通过mapperProxyFactory给出对应代理器的实例
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```

#### 映射接口的代理
上文说到，`MapperRegistry`类的`getMapper`方法返回的是一个代理对象，其创建者实际上是`MapperProxyFactory`类，对应代码如下：
```java
// getMapper调用的
public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}

protected T newInstance(MapperProxy<T> mapperProxy) {
    // 三个参数分别是：
    // 创建代理对象的类加载器、要代理的接口、代理类的处理器（即具体的实现）。
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```

代理对象本身为`MapperProxy`类，其实现了`InvocationHandler`接口，重载了`invoke()`函数，因此当出现`*Mapper`类中的方法被调用时，会通过该函数完成调用
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) { // 继承自Object的方法
            // 直接执行原有方法
            return method.invoke(this, args);
        } else if (method.isDefault()) { // 默认方法
            // 执行默认方法
            return invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
    // 找对对应的MapperMethod对象
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //** 调用MapperMethod中的execute方法
    return mapperMethod.execute(sqlSession, args);
}
```

这里的核心是`mapperMethod.execute(sqlSession, args)`，MyBatis将映射接口与`MapperProxy`对应，将映射接口中的抽象方法与`MapperMethod`对应，`MapperMethod`类中的`execute()`方法则根据语句类型重新调用`SqlSession`对象中的增删改查方法
```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) { // 根据SQL语句类型，执行不同操作
        case INSERT: { // 如果是插入语句
            ...
            break;
        }
        case UPDATE: { // 如果是更新语句
            ...
            break;
        }
        case DELETE: { // 如果是删除语句MappedStatement
            ...
            break;
        }
        case SELECT: // 如果是查询语句
            ...
            break;
        case FLUSH: // 清空缓存语句
            ...
            break;
        default: // 未知语句类型，抛出异常
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
        // 查询结果为null,但返回类型为基本类型。因此返回变量无法接收查询结果，抛出异常。
        throw new BindingException("Mapper method '" + command.getName()
            + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
}
```

到这里，MyBatis已经完成了**为映射接口注入实现**的过程，对映射接口中抽象方法的调用转变为了`SqlSesion`类的方法调用

#### SQL 语句的查找
`MapperMethod`类中的`execute()`方法中根据实际操作类型不同将控制权交给`SqlSession`类的不同方法

以`selectList()`方法为例，具体过程都是先构建`MappedStatement`对象，然后交由`Executor`对象执行
```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // 获取查询语句
        MappedStatement ms = configuration.getMappedStatement(statement);
        // 交由执行器进行查询
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

#### 查询结果缓存
`Executor`是一个接口，有`BaseExecutor`类和`CachingExecutor`类两种实现，`CachingExecutor`类表示查询过程使用缓存，即可以将之前的查询结果缓存，避免一直访问数据库

因此示例实际执行的代码是`CachingExecutor`类中的`query()`方法
```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

`BoundSql`是经过层层转化后去除掉`if`、`where`等标签的 SQL 语句，而`CacheKey`是为该次查询操作计算出来的缓存键

接下来，MyBatis查看当前的查询操作是否命中缓存，命中则从缓存中获取数据结果；否则通过`delegate`调用`query()`方法
```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
    // 获取MappedStatement对应的缓存，可能的结果有：该命名空间的缓存、共享的其它命名空间的缓存、无缓存
    Cache cache = ms.getCache();
    // 如果映射文件未设置<cache>或<cache-ref>则，此处cache变量为null
    if (cache != null) { // 存在缓存
        // 根据要求判断语句执行前是否要清除二级缓存，如果需要，清除二级缓存
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) { // 该语句使用缓存且没有输出结果处理器
            // 二级缓存不支持含有输出参数的CALLABLE语句，故在这里进行判断
            ensureNoOutParams(ms, boundSql);
            // 从缓存中读取结果
            @SuppressWarnings("unchecked")
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) { // 缓存中没有结果
                // 交给被包装的执行器执行
                list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                // 缓存被包装执行器返回的结果
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            return list;
        }
    }
    // 交由被包装的实际执行器执行
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

#### 数据库查询
整个流程的关键步骤如下：
+ 在进行数据库查询前，**先查询缓存**；如果确实需要查询数据库，则数据库查询之后的结果也放入缓存中
+ SQL语句的执行经过了层层转化，依次经过了`MappedStatement`对象、`Statement`对象和`PreparedStatement`对象，最后才得以执行
+ 最终数据库查询得到的结果交给`ResultHandler`对象处理

`delegate`调用的`query()`方法实际流向`BaseExecutor`类中的`query()`方法，其中核心方法为`queryFromDatabase()`方法

`queryFromDatabase()`方法代码如下：
```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, 
    ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 向缓存中增加占位符，表示正在查询
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        // 删除占位符
        localCache.removeObject(key);
    }
    // 将查询结果写入缓存
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

MyBatis先在缓存中放置一个占位符，然后调用`doQuery()`方法实际执行查询操作，最后，又把缓存中的占位符替换成真正的查询结果

`doQuery()`方法是`BaseExecutor`类中的抽象方法，实际运行代码如下：
```java
public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds,
                        ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        flushStatements();
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameterObject, 
                                        rowBounds, resultHandler, boundSql);
        Connection connection = getConnection(ms.getStatementLog());
        stmt = handler.prepare(connection, transaction.getTimeout());
        handler.parameterize(stmt);
        return handler.query(stmt, resultHandler);
    } finally {
        closeStatement(stmt);
    }
}
```

`handler.query(stmt，resultHandler)`调用的是`StatementHandler`接口的抽象方法，实际调用了`PreparedStatementHandler`类的代码，如下所示：
```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 执行真正的查询，查询完成后，结果就在ps中了
    ps.execute();
    // 由resultSetHandler继续处理结果
    return resultSetHandler.handleResultSets(ps);
}
```

查询完成之后的结果放在`PreparedStatement`对象中，交由`ResultSetHandler`对象处理

#### 处理结果集
查询得到的结果并没有直接返回，而是交给`ResultSetHandler`对象处理

`ResultSetHandler`是结果集处理器接口，实现类是`DefaultResultSetHandler`，因此实际处理结果的是实现类中的`handleResultSets`方法

`handleResultSets`方法将查询出来的结果被遍历后放入了列表`multipleResults`中并返回，`multipleResults`中存储的就是这次查询期望的结果`List＜User＞`

`handleResultSets`方法整个调用链很长，具体如下：
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/mybatis/handleresultsets.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        handleResultSets方法调用链
    </div>
</center>

重点关注的是图中粗线边框标注的三个方法：
+ **createResultObject**`(ResultSetWrapper, ResultMap, List<Class<?>>, List<Object>, String)`方法: 该方法创建了输出结果对象。在示例中，为`User`对象
+ **applyAutomaticMappings**方法: 在自动属性映射功能开启的情况下，该方法将数据记录的值赋给输出结果对象
+ **applyPropertyMappings**方法: 该方法按照用户的映射设置，给输出结果对象的属性赋值


经过以上过程，MyBatis将数据库输出的记录转化为了对象列表。之后，以上方法逐级返回。

最后，装载着对象列表的 `multipleResults` 被返回给`List<User> userList`变量，我们便拿到了查询结果
```java
List<User> userList = userMapper.queryUserBySchoolName(userParam);
```

## 项目包结构
<center>
    <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/mybatis/packages.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
        MyBatis 项目包结构
    </div>
</center>

## 参考
《MyBatis源码详解——通用源码阅读指导书》