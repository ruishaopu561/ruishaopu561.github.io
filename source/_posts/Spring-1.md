---
title: Spring-1
date: 2021-05-25 21:26:26
top_img: /img/cover/spring.jpeg
cover: /img/cover/spring.jpeg
tags: [Spring, 笔记]
---

## Bean 装配
### 自动化装配
+ 自动化装配 = 组件扫描 + 自动装配
    + 组件扫描(component scanning): Spring会自动发现应用上下文中所创建的bean
    + 自动装配(autowiring): Spring 自动满足bean之间的依赖

+ `@Component`注解作用在类上，表明该类作为组件类，并告知 Spring 要为这个类创建 bean。
    + `@Component("bean-name")`，bean 的默认ID为类名，手动指明后为`bean-name`
+ `@ComponentScan`注解作用在配置类，强制 Spring 开启组件扫描，默认扫描位置为配置类所在的目录，找到所有带`@Component`注解的类。
    + `@ComponentScan(basePackages="soudsystem")`, `@ComponentScan(basePackages={"soudsystem", "video"})`，扫描这些包
    + `@ComponentScan(basePackages={CDPlayer.class, DVDPlayer.class})`，扫描这些类所在的包

+ `@Autowired`注解，作用于构造器、属性的setter方法、对象，表明当Spring调用这个构造器或方法或对象时，会传入一个合适的bean
    + 如果有且只有一个 bean 匹配依赖需求，那么这个 bean 会被装配进来；
    + 如果没有匹配的 bean，那么应用上下文创建的时候，Spring 会抛出异常；`@Autowired(required=false)`可以避免异常；
    + 如果有多个 bean 都能满足依赖关系，Spring 会抛出异常，表明没有指明选择哪个 bean；

### 通过Java代码装配bean
首先需要一个单独的配置类 `JavaConfig`，自动化装配会给配置类加上`@ComponentScan`，这里我们不选择这么做；

+ `@Bean`注解，作用于创建类实例的方法，告诉Spring该方法会返回一个对象，该对象要注册成为Spring应用上下文中的bean，方法体包含了创建对象的具体逻辑
    ```java
    @Bean
    public CompactDisc sgtPeppers() {
        return new SgtPeppers();
    }
    ```

### 通过XML装配bean
这个xml文件比较繁琐，且方式落后，详情见书本吧

### 配置 profile bean
环境的不同需要不同构建方式的同一个 bean。

在3.1版本中，Spring引入了bean profile的功能。要使用profile，你首先要将所有不同的bean定义整理到一个或多个profile之中，在将应用部署 到每个环境时，要确保对应的profile处于激活（active）的状态。

+ `@Profile`注解指定某个bean属于哪一个profile，可以用在类上，也可以用在方法上，只有profile被激活时，对应的bean才会被创建，没有指定profile的bean始终都被被创建
    + 示例：
    ```java
    @Configuration
    public class DataSourceConfig {
        @Bean(destroyMethod="shutdown")
        @Profile("dev")
        public DataSource embeddedDataSource() {
        }

        @Bean
        @Profile("prod")
        public DataSource jndiDataSource() {
        }
    }
    ```

+ profile的激活：`spring.profiles.active`和`spring.profiles.default`
    + 如果`spring.profiles.active`有设置，则激活`spring.profiles.active`对应的profile；如果没有设置，则看`spring.profiles.default`；
    + 如果`spring.profiles.default`有设置，则激活`spring.profiles.default`对应的profile；如果没有设置，则设置了profile的bean都不激活；
+ `@ActiveProfiles("dev")`注解，指定要激活的profile

### 条件化的 bean
假设你希望一个或多个bean只有在应用的类路径下包含特定的库时才创建。或者我们希望某个bean只有当另外某个特定的bean也声明了之后 才会创建。我们还可能要求只有某个特定的环境变量设置之后，才会创建某个bean。

+ `@Conditional`注解，可以用于带有`@Bean`注解的方法上；如果给的条件计算为true，就会创建这个bean，否则会被忽略
    + magicBean方法使用了`@Conditional`注解，传入了`MagicExistsCondition.class`，`MagicExistsCondition.class`实现了`Condition.class`的matches接口，判断有没有`magic`这个环境属性。如果有，则创建这个bean；如果没有，则不创建。
    ```java
    @Bean
    @Conditional(MagicExistsCondition.class)
    public MagicBean magicBean() {
        return MagicBean();
    }

    public interface Condition {
        boolean matches(ConditionContext ctxt, AnnotatedTypeMetadata metadata);
    }

    public class MagicExistsCondition implements Condition {
        public boolean matches(ConditionContext ctxt, AnnotatedTypeMetadata metadata) {
            Environment env = context.getEnvironment();
            return env.containsProperty("magic");
        }
    }
    ```

### 处理自动装配的歧义性
即有多个匹配的bean能够满足装配
#### 标示首选的 bean
+ `@Primary`注解，指定最喜欢的方案。`@Primary`能够与`@Component`组合用在组件扫描
的bean上，也可以与`@Bean`组合用在Java配置的bean声明中
    + 示例
    ```java
    // 自动化配置
    @Component
    @Primary
    public class IceCream implements Dessert { ... }

    // Java 配置
    @Bean
    @Primary
    public Dessert iceCream() {
        return new IceCream();
    };
    ```

####  限定自动装配的 bean
+ `@Qualifier`注解，使用限定符的主要方式。可以与`@Autowired`协同使用，参数为想注入的 bean 的 ID
    + 示例
    ```java
    @Autowired
    @Qualifier("iceCream")
    public void setDessert(Dessert dessert) {
        this.dessert = dessert;
    }
    ```

这时候的限定符指定的bean的ID还是紧耦合的，一旦bean改名之后代码就需要再次修改，因此设计了自定义的限定符。
为 bean 设置限定符，也就不依赖于 bean ID 作为限定符，为 bean 声明上添加 `@Qualifier`注解即可
```java
@Component
@Qualifier("cold")
public class IceCream implements Dessert { ... }
```
以上，就实现了与 bean 的类名解耦

### bean 的作用域
+ 单例(Singleton)：在整个应用中，只创建一个bean实例
    + 默认模式，但是不适用于易变类型
+ 原型(Prototype)：每次注入或通过Spring应用上下文获取的时候，都会创建一个bean实例
    + 使用`@Scope`注解选择其他作用域
    ```java
    @Component
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) // 原型作用域
    public class Notepad { ... }
    ```
+ 会话(Session)：在Web应用中，为每个会话创建一个bean实例
    + `SCOPE_SESSION`，会为每个会话创建一个`ShoppingCart`
    ```java
    @Component
    @Scope(ConfigurableBeanFactory.SCOPE_SESSION，proxyMode=ScopedProxyMode.INTERFACES) // 原型作用域
    public ShoppingCart cart() { ... }
    ```
    + 注意：对于每个会话来说，还是单例的
    + 关于`ProxyMode`，详见书本59页
+ 请求(Request)：在Web应用中，为每个请求创建一个bean实例

## 面向切面的Spring
在使用面向切面编程时，我们仍然在一个地方定义通用功能， 但是可以通过声明的方式定义这个功能要以何种方式在何处应用，而无需修改受影响的类。横切关注点可以被模块化为特殊的类，这些类被称 为切面（aspect）。这样做有两个好处：首先，现在每个关注点都集中于一个地方，而不是分散到多处代码中；其次，服务模块更简洁，因为 它们只包含主要关注点（或核心功能）的代码，而次要关注点的代码被转移到切面中了。

+ 通知 advice：定义了切面是什么以及何时使用
    + 前置通知 Before：在目标方法被调用之前调用通知功能
    + 后置通知 After：在目标方法调用之后调用通知，不关心方法的输出是什么
    + 返回通知 After-returning：在目标方法成功执行之后调用通知
    + 异常通知 After-throwing：在目标方法抛出异常后调用通知
    + 环绕通知 Around：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为
+ 连接点 join point：在应用执行过程中能够插入切面的一个点。这个点可以是调用方法时、抛出异常时、甚至修改一个字段时。切面代码可以利用这些点插入到应用的正常流程之中，并添加新的行为。
+ 切点 pointcut：切点定义了何处，有助于缩小切面所通知的连接点的范围。切点的定义会匹配通知所要织入的一个或多个连接点。我们通常使用明确的类和方法名称，或是利用正则表达式定义所匹配的类和方法名称来指定这些切点。
+ 切面 Aspect：切面是通知和切点的结合。通知和切点共同定义了切面的全部内容——它是什么，在何时和何处完成其功能。
+ 引入 Introduction：引入允许我们向现有的类添加新方法或属性。
    + 例如，我们可以创建一个Auditable通知类，该类记录了对象最后一次修改时的状态。这很简单，只需一个方法，setLastModified(Date)，和一个实例变量来保存这个状态。然后，这个新方法和实例变量就可以被引入到现有的类中，从而可以在无需修改这些现有的类的情况下，让它们具有新的行为和状态。
+ 织入 Weaving：织入是把切面应用到目标对象并创建新的代理对象的过程。切面在指定的连接点被织入到目标对象中。
    + 在目标对象的生命周期里有多个点可以进行织入：
        + 编译期：切面在目标类编译时被织入。这种方式需要特殊的编译器。AspectJ的织入编译器就是以这种方式织入切面的。
        + 类加载期：切面在目标类加载到JVM时被织入。这种方式需要特殊的类加载器（ClassLoader），它可以在目标类被引入应用之前增 强该目标类的字节码。AspectJ 5的加载时织入（load-time weaving，LTW）就支持以这种方式织入切面。
        + 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象动态地创建一个代理对象。Spring AOP就是以这种方式织入切面的。

+ 切面类实例：
    ```java
    package concert;

    public interface Performance {
        public void perform();
    }
    ```
    ```java
    package concert;

    @Aspect     // 表明是个切面
    public class Audience {
        // 定义命名的切点
        @Pointcut("execution(** concert.Performance.perform(..))")
        public void performance() {};

        @Before("performance()")
        public void slienceCellPhones() {
            System.out.println("Sliencing cell phones");
        }

        @Before("performance()")
        public void takeSeats() {
            System.out.println("Taking seats");
        }

        @AfterReturning("performance()")
        public void applause() {
            System.out.println("CLAP CLAP CLAP!!");
        }

        @AfterThrowing("performance()")
        public void demandRefund() {
            System.out.println("Demanding a refund");
        }
    }
    ```

## 第5章 构建Spring Web应用程序



## 参考
《[Spring实战](https://github.com/niuxinghua/iOS-collection/blob/master/Spring%E5%AE%9E%E6%88%98%EF%BC%88%E7%AC%AC4%E7%89%88%EF%BC%89%20(1)%20(1).pdf)》