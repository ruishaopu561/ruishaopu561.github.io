---
title: MyBatis-2
date: 2022-09-25 14:14:40
top_img: /img/mybatis/cache.jpg
cover: /img/mybatis/cache.jpg
tags: [MyBatis, 笔记]
---

# MyBatis 源码阅读
第一篇记录了MyBatis的核心组成和工作逻辑，本篇再添加一些细节

## 缓存
代码位于`cache`包，只对关键部分摘抄，加深印象

### 缓存键
缓存键设计要求：
+ **无碰撞**: 必须保证两条不同的查询请求生成的键不一致，这是最重要也是必须满足的要求。否则会引发查询操作命中错误的缓存，并返回错误的结果
+ **高效比较**: 每次缓存查询操作都可能会引发键之间的多次比较，因此该操作必须是高效的
+ **高效生成**: 每次缓存查询和写入操作前都需要生成缓存的键，因此该操作也必须是高效的

`CacheKey`类的关键属性如下所示，在缓存键的构造过程中，会将重要信息按序调用`update()`函数保存，这样形成的`updateList`就确保了碰撞不会发生：
```java
// 乘数，用来计算hashcode时使用
private final int multiplier;
// 哈希值，整个CacheKey的哈希值。如果两个CacheKey该值不同，则两个CacheKey一定不同
private int hashcode;
// 求和校验值，整个CacheKey的求和校验值。如果两个CacheKey该值不同，则两个CacheKey一定不同
private long checksum;
// 更新次数，整个CacheKey的更新次数
private int count;
// 更新历史
private List<Object> updateList;

/**
* 更新CacheKey
* @param object 此次更新的参数
*/
public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);

    count++;
    checksum += baseHashCode;
    baseHashCode *= count;

    hashcode = multiplier * hashcode + baseHashCode;

    updateList.add(object);
}
```

比较函数`equals()`如下所示，通过`count`、`checksum`、`hashcode`这三个值实现了快速比较，最后再一一对比`updateList`中的值，保证了不会碰撞
```java
/**
* 比较当前对象和入参对象（通常也是CacheKey对象）是否相等
* @param object 入参对象
* @return 是否相等
*/
@Override
public boolean equals(Object object) {
    // 如果地址一样，是一个对象，肯定相等
    if (this == object) {
        return true;
    }
    // 如果入参不是CacheKey对象，肯定不相等
    if (!(object instanceof CacheKey)) {
        return false;
    }
    final CacheKey cacheKey = (CacheKey) object;
    // 依次通过hashcode、checksum、count判断。必须完全一致才相等
    if (hashcode != cacheKey.hashcode) {
        return false;
    }
    if (checksum != cacheKey.checksum) {
        return false;
    }
    if (count != cacheKey.count) {
        return false;
    }

    // 详细比较变更历史中的每次变更
    for (int i = 0; i < updateList.size(); i++) {
        Object thisObject = updateList.get(i);
        Object thatObject = cacheKey.updateList.get(i);
        if (!ArrayUtil.equals(thisObject, thatObject)) {
            return false;
        }
    }
    return true;
}
```

`BaseExecutor`类中生成缓存键的`createCacheKey()`方法如下所示，生成的`CacheKey`对象中包含了这次查询的所有信息，包括查询语句的id、查询的翻页限制、数据总量、完整的SQL语句，这些信息一致就保证了两次查询的一致
```java
/**
* 生成查询的缓存的键
* @param ms 映射语句对象
* @param parameterObject 参数对象
* @param rowBounds 翻页限制
* @param boundSql 解析结束后的SQL语句
* @return 生成的键值
*/
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // 创建CacheKey，并将所有查询参数依次更新写入
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
        if (parameterMapping.getMode() != ParameterMode.OUT) {
            Object value;
            String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
            value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
            value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
        } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
        // issue #176
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}
```

通过保存每次查询完整且有效的信息，就可以做到高效的比较和生成，并且完整的信息有利于避免碰撞

### 缓存装饰器
`Cache`类本身只是接口，实现仅有`PerpetualCache`类，该类的实现非常简单，但可以通过装饰器来为其增加更多的功能。

decorators子包中存在许多装饰器，根据装饰器的功能可以将它们分为以下几个大类：
+ **同步装饰器**: 为缓存增加同步功能，如`SynchronizedCache`类
+ **日志装饰器**: 为缓存增加日志功能，如`LoggingCache`类
+ **清理装饰器**: 为缓存中的数据增加清理功能，如`FifoCache`类、`LruCache`类、`WeakCacheSoftCache`类
+ **阻塞装饰器**: 为缓存增加阻塞功能，如`BlockingCache`类
+ **定时清理装饰器**: 为缓存增加定时刷新功能，如`ScheduledCache`类
+ **序列化装饰器**: 为缓存增加序列化功能，如`SerializedCache`类
+ **事务装饰器**: 用于支持事务操作的装饰器，如`TransactionalCache`类

各缓存的组件过程在`CacheBuilder`类的`build()`方法实现，如下所示，该过程先选用默认配置，再根据用户配置进行调整，最终将需要的装饰器全部用上，最终得到组合了各种功能的缓存
```java
/**
* 组建缓存
* @return 缓存对象
*/
public Cache build() {
    // 设置缓存的默认实现、默认装饰器（仅设置，并未装配）
    setDefaultImplementations();
    // 创建默认的缓存
    Cache cache = newBaseCacheInstance(implementation, id);
    // 设置缓存的属性
    setCacheProperties(cache);
    if (PerpetualCache.class.equals(cache.getClass())) { // 缓存实现是PerpetualCache，即不是用户自定义的缓存实现
        // 为缓存逐级嵌套自定义的装饰器
        for (Class<? extends Cache> decorator : decorators) {
        // 生成装饰器实例，并装配。入参依次是装饰器类、被装饰的缓存
        cache = newCacheDecoratorInstance(decorator, cache);
        // 为装饰器设置属性
        setCacheProperties(cache);
        }
        // 为缓存增加标准的装饰器
        cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
        // 增加日志装饰器
        cache = new LoggingCache(cache);
    }
    // 返回被包装好的缓存
    return cache;
}
```

### 缓存机制
由于SQL语句的实际执行者是`Executor`相关类，缓存机制的实现也与它有关。

`Executor`相关类的继承关系如下图所示，`CachingExecutor`类是带有缓存实现的装饰执行器：
<center>
    <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/mybatis/executors.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
        Executor类继承关系
    </div>
</center>

#### 一级缓存
一级缓存又叫本地缓存，相关配置项有两个：
+ **配置文件的`settings`节点，可以改变一级缓存的作用范围**: 如下所示，配置值选项有`SESSION`和`STATEMENT`，分别对应一次会话和一条语句，默认是`SESSION`
    ```xml
    <setting name="localCacheScope" value="SESSION"/>
    ```
+ **映射文件的数据库操作节点，增加`flushCache`选项**：如下所示，属性可为`true`或`false`，为`true`时清空一、二级缓存，默认为`false`
    ```xml
    <select id="queryUserBySchoolName" resultType="User" flushCache="false">
        SELECT * FROM `user` WHERE schoolName=#{schoolName}
    </select>
    ```

一级缓存功能由`BaseExecutor`类实现。`BaseExecutor`内，有两个与一级缓存相关的属性，分别是`localCache`和`localOutputParameterCache`。
+ `localCache`缓存的是数据库查询操作的结果
+ 对于`CALLABLE`形式的语句，因为最终向上返回的是输出参数，便使用`localOutputParameterCache`直接缓存的输出参数

一级缓存作用范围很有限，**最大范围只是一次会话**，不支持各种装饰器的修饰，因此不能进行容量配置、清理策略设置及阻塞设置等
```java
// 查询操作的结果缓存
protected PerpetualCache localCache;
// Callable查询的输出参数缓存
protected PerpetualCache localOutputParameterCache;
```

#### 二级缓存
**二级缓存的作用范围是一个命名空间（即一个映射文件），而且可以实现多个命名空间共享一个缓存**

4个配置项：
+ 配置文件的`settings`节点，启用或关闭二级缓存: 默认为`true`，即默认启用二级缓存
    ```xml
    <setting name="cacheEnabled" value="true"/>
    ```
+ 映射文件，`cache`标签开启并配置本命名空间的缓存，或`cache-ref`标签声明本命名空间使用其他命名空间的缓存，如果两项都不配置则表示命名空间没有缓存
    > 注：**该项配置只有在第一项配置中选择启用二级缓存时才有效**。
    
    ```xml
    <cache type="PERPETUAL"
            eviction="FIFO"
            flushInterval="60000"
            size="512"
            readOnly="true"
            blocking="true">
    </cache>

    <cache-ref namespace="com.github.yeecode.mybatisdemo.dao.UserMapper"/>
    ```
+ 数据库操作节点的`useCache`属性，**配置该数据库操作节点是否使用二级缓存**，默认为`true`
    > 注：只有当第一、二项配置均启用了缓存时，该项 配置才有效

    ```xml
    <select id="queryUserBySchoolName" resultType="User" flushCache="false" useCache="true">
        SELECT * FROM `user` WHERE schoolName=#{schoolName}
    </select>
    ```
+ 数据库操作节点的`flushCache`属性，与一级缓存共用，表示是否清除一二级缓存

**二级缓存功能由`CachingExecutor`类实现**，它是一个装饰器类，内部有其他类型的执行器，通过装饰实际执行器为它们增加二级缓存功能。

#### 两级缓存机制
`CachingExecutor`作为装饰器会先运行，然后才会调用实际执行器，这时`BaseExecutor`中的方法才会执行。因此，在数据库查询操作中，MyBatis 会**先访问二级缓存再访问一级缓存**

<center>
    <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/mybatis/executor_cache.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
        两级缓存示意图
    </div>
</center>

## 参考
《MyBatis源码详解——通用源码阅读指导书》