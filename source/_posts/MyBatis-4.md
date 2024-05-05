---
title: MyBatis-4
date: 2022-09-26 21:22:51
top_img: /img/mybatis/mapper.png
cover: /img/mybatis/mapper.png
tags: [MyBatis, 笔记]
---

# MyBatis 源码阅读
第一篇记录了MyBatis的核心组成和工作逻辑，本篇再添加一些细节

## Java方法与SQL语句绑定
代码位于`binding`包，只对关键部分摘抄，加深印象

`binding`包具有以下两个功能：
+ **维护映射接口中抽象方法与数据库操作节点之间的关联关系**
+ **为映射接口中的抽象方法接入对应的数据库操作**
 
### 数据库操作接入总结
#### 初始化阶段
MyBatis在初始化阶段会解析各个映射文件，然后将各个数据库操作节点的信息记录到`Configuration`对象的`mappedStatements`属性中

```java
protected final Map<String, MappedStatement> mappedStatements = 
                new StrictMap<MappedStatement>("Mapped Statements collection");
```

如上所示，`mappedStatements`属性结构是一个`StrictMap`（一个不允许覆盖键值的HashMap），该`StrictMap`的键为SQL语句的"**namespace值.语句id值**"，值为数据库操作节点的详细信息，即`MappedStatement`对象

以如下映射文件为例，该`select`节点的键为`com.github.yeecode.mybatisdemo.dao.UserMapper.queryUserBySchoolName`，值为`select`节点
```xml
<mapper namespace="com.github.yeecode.mybatisdemo.dao.UserMapper">
    <select id="queryUserBySchoolName" resultType="com.github.yeecode.mybatisdemo.model.User">
        SELECT * FROM `user` WHERE schoolName=#{schoolName}
    </select>
</mapper>
```

MyBatis还会在初始化阶段**扫描所有的映射接口，并根据映射接口创建与之关联的`MapperProxyFactory`，两者的关联关系由`MapperRegistry`维护**。当调用`MapperRegistry`的`getMapper()`方法（SqlSession的getMapper方法最终也会调用到这里）时，`MapperProxyFactory`会生产出一个`MapperProxy`对象作为映射接口的代理

#### 数据读写阶段 
当映射接口中有方法被调用时，会被代理对象`MapperProxy`劫持，转而触发了`MapperProxy`对象中的`invoke`方法。`MapperProxy`对象中的`invoke`方法会创建或取出该映射接口方法对应的`MapperMethod`对象，如下。
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
    // 找对对应的 MapperMethod 对象
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 调用MapperMethod中的execute方法
    return mapperMethod.execute(sqlSession, args);
}
// 创建或取出
private MapperMethod cachedMapperMethod(Method method) {
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
}
```

在创建`MapperMethod`对象的过程中，`MapperMethod`中`SqlCommand`子类的构造方法会去`Configuration`对象的`mappedStatements`属性中根据当前映射接口名、方法名索引前期已经存好的 SQL语句信息，如下
```java
/**
 * 找出指定接口指定方法对应的MappedStatement对象
 * @param mapperInterface 映射接口
 * @param methodName 映射接口中具体操作方法名
 * @param declaringClass 操作方法所在的类。一般是映射接口本身，也可能是映射接口的子类
 * @param configuration 配置信息
 * @return MappedStatement对象
 */
private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
        Class<?> declaringClass, Configuration configuration) {
    // 数据库操作语句的编号是：接口名.方法名
    String statementId = mapperInterface.getName() + "." + methodName;
    // configuration保存了解析后的所有操作语句，去查找该语句
    if (configuration.hasStatement(statementId)) {
        // 从configuration中找到了对应的语句，返回
        return configuration.getMappedStatement(statementId);
    } else if (mapperInterface.equals(declaringClass)) {
        // 说明递归调用已经到终点，但是仍然没有找到匹配的结果
        return null;
    }
    // 从方法的定义类开始，沿着父类向上寻找。找到接口类时停止
    for (Class<?> superInterface : mapperInterface.getInterfaces()) {
        if (declaringClass.isAssignableFrom(superInterface)) {
            MappedStatement ms = resolveMappedStatement(superInterface, methodName,
                declaringClass, configuration);
            if (ms != null) {
                return ms;
            }
        }
    }
    return null;
}
```

然后，`MapperMethod`对象的`execute`方法被触发，在 `execute`方法内会根据不同的SQL语句类型进行不同的数据库操作，如下
```java
/**
* 执行映射接口中的方法
* @param sqlSession sqlSession接口的实例，通过它可以进行数据库的操作
* @param args 执行接口方法时传入的参数
* @return 数据库操作结果
*/
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) { // 根据SQL语句类型，执行不同操作
        case INSERT: { // 如果是插入语句
            // 将参数顺序与实参对应好
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行操作并返回结果
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: { // 如果是更新语句
            // 将参数顺序与实参对应好
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行操作并返回结果
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: { // 如果是删除语句MappedStatement
            // 将参数顺序与实参对应好
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行操作并返回结果
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT: // 如果是查询语句
            // 方法返回值为void，且有结果处理器
            if (method.returnsVoid() && method.hasResultHandler()) {
                // 使用结果处理器执行查询
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) { // 多条结果查询
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) { // Map结果查询
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) { // 游标类型结果查询
                result = executeForCursor(sqlSession, args);
            } else { // 单条结果查询
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
                if (method.returnsOptional()
                    && (result == null || !method.getReturnType().equals(result.getClass()))) {
                    result = Optional.ofNullable(result);
                }
            }
            break;
        case FLUSH: // 清空缓存语句
            result = sqlSession.flushStatements();
            break;
        default: // 未知语句类型，抛出异常
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
        // 查询结果为null,但返回类型为基本类型。因此返回变量无法接收查询结果，抛出异常。
        throw new BindingException("Mapper method '" + command.getName()
            + " attempted to return null from a method with a primitive return type ("
            + method.getReturnType() + ").");
    }
    return result;
}
```

这样，一个针对映射接口中的方法调用，终于被转化为了对应的数据库操作。

###
## 参考
《MyBatis源码详解——通用源码阅读指导书》