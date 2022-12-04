---
title: "Mybatis执行"
date: 2022-12-04T23:50:55+08:00
draft: false
categories: ["数据库","mybatis"]
---

# 一、概念

- Configuration           MyBatis所有的配置信息都保存在Configuration对象之中，配置文件中的大部分配置都会存储到该类中
- SqlSession                作为MyBatis工作的主要顶层API，表示和数据库交互时的会话，完成必要数据库增删改查功能
- Executor                    MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护
  - BaseExecutor:是一个抽象类，这种通过抽象的实现接口的方式是的体现，**是Executor 的默认实现**
    - SimpleExecutor:是 MyBatis 中默认使用的执行器，每执行一次 update 或 select，就开启一个`Statement` 对象，用完就直接关闭 `Statement` 对象
    - ReuseExecutor:可重用执行器，这里的重用指的是重复使用`Statement`，它会在内部使用一个 Map 把创建的`Statement` 都缓存起来，每次执行 SQL 命令的时候，都会去判断是否存在基于该 SQL 的 `Statement` 对象，如果存在`Statement` 对象并且对应的 `connection` 还没有关闭的情况下就继续使用之前的 `Statement` 对象，并将其缓存起来
    - BatchExecutor:批处理执行器，用于将多个 SQL 一次性输出到数据库

  - CachingExecutor:缓存执行器，先从缓存中查询结果，如果存在就返回之前的结果；如果不存在，再委托给`Executor delegate` 去数据库中取，delegate 可以是上面任何一个执行器。

- StatementHandler   封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数等
  - BaseStatementHandler:本身是一个抽象类，用于简化StatementHandler 接口实现的难度，属于适配器设计模式体现，它主要有三个实现类：
    - SimpleStatementHandler:java.sql.Statement对象创建处理器,管理 Statement 对象并向数据库中推送不需要预编译的SQL语句。
    - PreparedStatementHandler:java.sql.PrepareStatement对象的创建处理器，管理Statement对象并向数据中推送需要预编译的SQL语句。
    - CallableStatementHandler: java.sql.CallableStatement对象的创建处理器，管理 Statement 对象并调用数据库中的存储过程。

  - RoutingStatementHandler: 实际上整合了SimpleStatementHandler、PreparedStatementHandler、CallableStatementHandler。

- ParameterHandler   负责对用户传递的参数转换成JDBC Statement 所对应的数据类型
  - **setParameters**: 用于对 `PreparedStatement` 的参数赋值

- ResultSetHandler      负责将JDBC返回的ResultSet结果集对象转换成List类型的集合
- TypeHandler              负责java数据类型和jdbc数据类型(也可以说是数据表列类型)之间的映射和转换
- MappedStatement    MappedStatement维护一条<select|update|delete|insert>节点的封装
- SqlSource                  负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中
- BoundSql                    表示动态生成的SQL语句以及相应的参数信息


`Executor`对象在创建`SqlSession`对象的时候创建，并且缓存在`Configuration`对象里，负责管理一级缓存和二级缓存，并提供是事务管理的相关操作。`Executor`对象的主要功能是调用`StatementHandler`访问数据库，并将查询结果存入缓存中（如果配置了缓存的话)。`StatementHandler`首先通过`ParammeterHandler`完成SQL的实参绑定，然后通过`java.sql.Statement`对象执行sql语句并得到结果集`ResultSet`，最后通过`ResultSetHandler`完成结果集的映射，得到对象并返回。



`SimpleStatementHandler` 和 `PreparedStatementHandler` 的区别是 SQL 语句是否包含变量。是否通过外部进行参数传入

# 二、准备

```java
/**
 * @author peilizhi
 * @date 2021/10/9 12:52
 **/
@Data
public class Demo {
    private Long id;
    private String name;
    private Integer age;
}

/**
 * @author peilizhi
 * @date 2021/10/9 12:53
 **/
public interface DemoMapper {
    Demo selectDemoById(Integer id);
}

```

```xml
配置文件
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource="mysql.properties"/>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${mysql.driverClass}"/>
                <property name="url" value="${mysql.url}"/>
                <property name="username" value="${mysql.username}"/>
                <property name="password" value="${mysql.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <!--resource-->
        <mapper resource="mapper/DemoMapper.xml"/>

        <!--class-->
        <!-- <mapper class="com.wsdsg.spring.boot.analyze.mapper.UserMapper"/>-->

        <!--url-->
        <!--<mapper url="D:\coder_soft\idea_workspace\ecard_bus\spring-boot-analyze\target\classes\UserMapper.xml"/>-->

        <!--package-->
        <!--<package name="com.wsdsg.spring.boot.analyze.mapper" />-->
    </mappers>
</configuration>
```

```java
/**
 * @author peilizhi
 * @date 2021/10/9 12:59
 **/

public class DemoTest {


    public static void main(String[] args) throws IOException {
      // 不可引用容器中的内容
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        try (SqlSession session = sqlSessionFactory.openSession()) {
            /*UserMapper mapper = session.getMapper(UserMapper.class);
            User user = mapper.selectUserById(1);
            System.out.println(user);*/

            Object o = session.selectOne("com.huochai.repository.mapper.DemoMapper.selectDemoById", 1);
            System.out.println("我是第一次查询的" + o);
            System.out.println("-------------------------------我是分割线---------------------");
            Object z = session.selectOne("com.huochai.repository.mapper.DemoMapper.selectDemoById", 1);
            System.out.println("我是第二次查询的" + z);
           /*User user = new User();
           user.setAge(15);
           user.setName("achuan");
           int insert = session.insert("com.wsdsg.spring.boot.analyze.mapper.UserMapper.addOneUser", user);
           session.commit();
           System.out.println(insert);
           */
        }
    }


}
```

- configuration（配置）
  - [properties（属性）](https://mybatis.org/mybatis-3/zh/configuration.html#properties)
  - [settings（设置）](https://mybatis.org/mybatis-3/zh/configuration.html#settings)
  - [typeAliases（类型别名）](https://mybatis.org/mybatis-3/zh/configuration.html#typeAliases)
  - [typeHandlers（类型处理器）](https://mybatis.org/mybatis-3/zh/configuration.html#typeHandlers)
  - [objectFactory（对象工厂）](https://mybatis.org/mybatis-3/zh/configuration.html#objectFactory)
  - [plugins（插件）](https://mybatis.org/mybatis-3/zh/configuration.html#plugins)
  - environments（环境配置）
    - environment（环境变量）
      - transactionManager（事务管理器）
      - dataSource（数据源）
  - [databaseIdProvider（数据库厂商标识）](https://mybatis.org/mybatis-3/zh/configuration.html#databaseIdProvider)
  - [mappers（映射器）](https://mybatis.org/mybatis-3/zh/configuration.html#mappers)

**初始化：解析xml文件里面的标签，构建configuration对象。并放在SqlSessionFactory 中**

# 三、流程

Mybatis 解析xml文件的时候广泛使用构建者模式，因为对象比较复杂

## 3.1、读取配置文件

## 3.2 构建会话工厂对象

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
        // Intentionally ignore. Prefer previous error.
      }
    }
  }

// 将解析好的Configuration 加入到SqlSessionFactory中
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

```java
 public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
  }

  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    // 创建一个Configuration 对象
    super(new Configuration());
    // 用于记录解析xml时出错的
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
```

**在做解析文件的时候，可以创建一个对象用于记录解析过程中出现的问题**

### 3.2.1 创建XMLConfigBuilder

```
XMLConfigBuilder  这个对象的作用就是解析主配置文件用的。
先说明一下，我们可以看出主配置文件的最外层节点是<configuration>标签，mybatis的初始化就是把这个标签以及他的所有子标签进行解析，把解析好的数据封装在Configuration这个类中。
```

```java
public class XMLConfigBuilder extends BaseBuilder {

  // 是否解析过
  private boolean parsed;
  // 用来解析xml文件的 包括检测xml文件格式
  private final XPathParser parser;
  // 环境ID，不同的环境对应不同的数据源
  private String environment;
  // 反射工场
  private final ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();
}

public abstract class BaseBuilder {
  // 
  protected final Configuration configuration;
  protected final TypeAliasRegistry typeAliasRegistry;
  protected final TypeHandlerRegistry typeHandlerRegistry;
}
```

**Mybatis初始化就是读取配置文件，构建Configuration类**

**Configuration 是mybatis 的核心类**

```java
// 与mybatis-config.xml配置文件相对应
public class Configuration {
  
  // 创建时就进行初始化
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  
  public Configuration() {
    // 事务
    typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
    typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);
    // 数据源
    typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
    typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
    typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);
    // 缓存
    typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
    typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
    typeAliasRegistry.registerAlias("LRU", LruCache.class);
    typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
    typeAliasRegistry.registerAlias("WEAK", WeakCache.class);
    // 供应商的DataBaseId 提供者
    typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);
    // mybatis 默认的XML 驱动为XMLLanguageDriver,用了解析动态sql
    typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
    typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);
    // 日志类型
    typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
    typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
    typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
    typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
    typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
    typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
    typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);
    // 代理工厂
    typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
    typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);
    // 设置默认的XML 驱动
    languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
    languageRegistry.register(RawLanguageDriver.class);
  }
  
}
```

**TypeAliasRegistry 是一种类型注册器。Registry 后缀的对象可以理解成一个Map 里面存储一些别名**

****

```java
public class TypeAliasRegistry {

  private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<String, Class<?>>();

  public TypeAliasRegistry() {
    registerAlias("string", String.class);

    registerAlias("byte", Byte.class);
    registerAlias("long", Long.class);
    registerAlias("short", Short.class);
    registerAlias("int", Integer.class);
    registerAlias("integer", Integer.class);
    registerAlias("double", Double.class);
    registerAlias("float", Float.class);
    registerAlias("boolean", Boolean.class);

    registerAlias("byte[]", Byte[].class);
    registerAlias("long[]", Long[].class);
    registerAlias("short[]", Short[].class);
    registerAlias("int[]", Integer[].class);
    registerAlias("integer[]", Integer[].class);
    registerAlias("double[]", Double[].class);
    registerAlias("float[]", Float[].class);
    registerAlias("boolean[]", Boolean[].class);

    registerAlias("_byte", byte.class);
    registerAlias("_long", long.class);
    registerAlias("_short", short.class);
    registerAlias("_int", int.class);
    registerAlias("_integer", int.class);
    registerAlias("_double", double.class);
    registerAlias("_float", float.class);
    registerAlias("_boolean", boolean.class);

    registerAlias("_byte[]", byte[].class);
    registerAlias("_long[]", long[].class);
    registerAlias("_short[]", short[].class);
    registerAlias("_int[]", int[].class);
    registerAlias("_integer[]", int[].class);
    registerAlias("_double[]", double[].class);
    registerAlias("_float[]", float[].class);
    registerAlias("_boolean[]", boolean[].class);

    registerAlias("date", Date.class);
    registerAlias("decimal", BigDecimal.class);
    registerAlias("bigdecimal", BigDecimal.class);
    registerAlias("biginteger", BigInteger.class);
    registerAlias("object", Object.class);

    registerAlias("date[]", Date[].class);
    registerAlias("decimal[]", BigDecimal[].class);
    registerAlias("bigdecimal[]", BigDecimal[].class);
    registerAlias("biginteger[]", BigInteger[].class);
    registerAlias("object[]", Object[].class);

    registerAlias("map", Map.class);
    registerAlias("hashmap", HashMap.class);
    registerAlias("list", List.class);
    registerAlias("arraylist", ArrayList.class);
    registerAlias("collection", Collection.class);
    registerAlias("iterator", Iterator.class);

    registerAlias("ResultSet", ResultSet.class);
  }
```

### 3.2.2  解析xml 文件 

parser.parse():

```java
public Configuration parse() {
        if (this.parsed) {
            throw new BuilderException("Each XMLConfigBuilder can only be used once.");
        } else {
            this.parsed = true;
          // 利用已经创建好的XPathParser解析xml 文件 
            this.parseConfiguration(this.parser.evalNode("/configuration"));
            return this.configuration;
        }
    }
```

```
首先判断是否解析过配置文件，如果解析过就抛异常，默认是未解析的

```

#### 3.2.2.1 判断是否解析过配置文件

解析过就抛出异常

#### 3.2.2.2 设置已经解析

#### 3,2.2.3 解析configuration标签

```java
private void parseConfiguration(XNode root) {
        try {
          // 解析properties标签
            this.propertiesElement(root.evalNode("properties"));
          // 校验settings 标签中的设置是否都能被mybatis 识别
            Properties settings = this.settingsAsProperties(root.evalNode("settings"));
            this.loadCustomVfs(settings);
          // 解析typeAliases 标签
            this.typeAliasesElement(root.evalNode("typeAliases"));
            this.pluginElement(root.evalNode("plugins"));
            this.objectFactoryElement(root.evalNode("objectFactory"));
            this.objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
            this.reflectorFactoryElement(root.evalNode("reflectorFactory"));
            this.settingsElement(settings);
            this.environmentsElement(root.evalNode("environments"));
            this.databaseIdProviderElement(root.evalNode("databaseIdProvider"));
            this.typeHandlerElement(root.evalNode("typeHandlers"));
            this.mapperElement(root.evalNode("mappers"));
        } catch (Exception var3) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + var3, var3);
        }
    }
```

就是将xml 文件里面的标签转换成XMLConfigBuilder 中 Configuration 的属性

##### properties

```xml
properties: 属性,属性可以在外部进行配置，并可以进行动态替换。你既可以在典型的 Java 属性文件中配置这些属性，也可以在 properties 元素的子元素中设置。例如：
<properties resource="org/mybatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>

可以在文件中动态替换参数
<dataSource type="POOLED">
  <property name="driver" value="${driver}"/>
  <property name="url" value="${url}"/>
  <property name="username" value="${username}"/>
  <property name="password" value="${password}"/>
</dataSource>

也可以设置属性
public SqlSessionFactory build(InputStream inputStream, Properties properties) {
    return build(inputStream, null, properties);
  }

首先读取在 properties 元素体内指定的属性。
然后根据 properties 元素中的 resource 属性读取类路径下属性文件，或根据 url 属性指定的路径读取属性文件，并覆盖之前读取过的同名属性。
最后读取作为方法参数传递的属性，并覆盖之前读取过的同名属性。
```

**properties 元素中的 resource 属性读取类路径下属性文件，或根据 url 属性指定的路径读取属性文件，并覆盖之前读取过的同名属性。**

```java
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      // <properties> 中配置的属性
      Properties defaults = context.getChildrenAsProperties();
      
      // resource 或者url 中的属性
      String resource = context.getStringAttribute("resource");
      String url = context.getStringAttribute("url");
      if (resource != null && url != null) {
        throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
      }
      if (resource != null) {
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      // 构造方法传入的参数
      Properties vars = configuration.getVariables();
      if (vars != null) {
        defaults.putAll(vars);
      }
      parser.setVariables(defaults);
      // 设置Configuration 对象的Properties 属性
      configuration.setVariables(defaults);
    }
  }
```

**通过方法参数传递的属性具有最高优先级，resource/url 属性中指定的配置文件次之，最低优先级的则是 properties 元素中指定的属性。**

 ##### typeAliases

```xml
typeAliases:类型别名 ,类型别名可为 Java 类型设置一个缩写名字。 它仅用于 XML 配置，意在降低冗余的全限定类名书写
<typeAliases>
  <typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
  <typeAlias alias="Comment" type="domain.blog.Comment"/>
  <typeAlias alias="Post" type="domain.blog.Post"/>
  <typeAlias alias="Section" type="domain.blog.Section"/>
  <typeAlias alias="Tag" type="domain.blog.Tag"/>
  
  每一个在包 中的 Java Bean，在没有注解的情况下，会使用 Bean 的首字母小写的非限定类名来作为它的别名
  <package name="domain.blog"/>
 
</typeAliases>
```

```java
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 为包中对象设置别名
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              // 自动获取类的简要名称
              typeAliasRegistry.registerAlias(clazz);
            } else {
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
```

##### plugins

```xml
plugins: MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
ParameterHandler (getParameterObject, setParameters)
ResultSetHandler (handleResultSets, handleOutputParameters)
StatementHandler (prepare, parameterize, batch, update, query)


/**
 * @author peilizhi
 * @date 2021/11/30 00:36
 **/
@Intercepts({
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),}
)
public class MyInterceptor implements Interceptor {
    /**
     * 拦截时要执行的方法
     */
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("被拦截方法执行之前，做的辅助服务······");
        Object proceed = invocation.proceed();
        System.out.println("被拦截方法执行之后，做的辅助服务······");
        return proceed;
    }

    /**
     * 用于封装目标对象，可以返回本身，也可以返回其代理
     *
     * @param target 目标对象
     *               表示被拦截的对象，此处为 Executor 的实例对象
     *               作用：如果被拦截对象所在的类有实现接口，就为当前拦截对象生成一个代理对象
     *               如果被拦截对象所在的类没有指定接口，这个对象之后的行为就不会被代理操作
     */
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    /**
     * 用于在 Mybatis 配置文件中指定一些属性的。
     */
    @Override
    public void setProperties(Properties properties) {
    }
}


<!-- mybatis-config.xml -->
<plugins>
  <plugin interceptor="org.mybatis.example.ExamplePlugin">
    <property name="someProperty" value="100"/>
  </plugin>
</plugins>
```

**MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。**

```jade
plugins: MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
ParameterHandler (getParameterObject, setParameters)
ResultSetHandler (handleResultSets, handleOutputParameters)
StatementHandler (prepare, parameterize, batch, update, query)


/**
 * @author peilizhi
 * @date 2021/11/30 00:36
 **/
@Intercepts({
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),}
)
public class MyInterceptor implements Interceptor {


    /**
     * 拦截时要执行的方法
     */
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("被拦截方法执行之前，做的辅助服务······");
        Object proceed = invocation.proceed();
        System.out.println("被拦截方法执行之后，做的辅助服务······");
        return proceed;
    }

    /**
     * 用于封装目标对象，可以返回本身，也可以返回其代理
     *
     * @param target 目标对象
     *               表示被拦截的对象，此处为 Executor 的实例对象
     *               作用：如果被拦截对象所在的类有实现接口，就为当前拦截对象生成一个代理对象
     *               如果被拦截对象所在的类没有指定接口，这个对象之后的行为就不会被代理操作
     */
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    /**
     * 用于在 Mybatis 配置文件中指定一些属性的。
     */
    @Override
    public void setProperties(Properties properties) {
    }
}


<!-- mybatis-config.xml -->
<plugins>
  <plugin interceptor="org.mybatis.example.ExamplePlugin">
    <property name="someProperty" value="100"/>
  </plugin>
</plugins>
```

**这可能会极大影响 MyBatis 的行为，务请慎之又慎。**

```java
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 获取拦截器类名
        String interceptor = child.getStringAttribute("interceptor");
        // 获取拦截器下的属性
        Properties properties = child.getChildrenAsProperties();
        // 从类别名注册器中获取，如果没有获取到就创建Class 对象，之后生成实例
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
        interceptorInstance.setProperties(properties);
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
```

##### objectFactory

```java
每次 MyBatis 创建结果对象的新实例时，它都会使用一个对象工厂（ObjectFactory）实例来完成实例化工作。 默认的对象工厂需要做的仅仅是实例化目标类，要么通过默认无参构造方法，要么通过存在的参数映射来调用带有参数的构造方法。 如果想覆盖对象工厂的默认行为，可以通过创建自己的对象工厂来实现。
public class ExampleObjectFactory extends DefaultObjectFactory {
    /**
     * 覆盖默认对象工厂实例化对象的操作
     * @param type
     * @param <T>
     * @return
     */
    @Override
    public <T> T create(Class<T> type) {
        System.out.println("在创建对象时执行了我");
        return create(type, null, null);
    }
    @Override
    public void setProperties(Properties properties) {
        // no props for default
    }
}
<objectFactory type="org.mybatis.example.ExampleObjectFactory">
  <property name="someProperty" value="100"/>
</objectFactory>
```

```java
private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
      // 读取接口中type指定的对象工厂
      String type = context.getStringAttribute("type");
      // 获取对象工厂设置的参数
      Properties properties = context.getChildrenAsProperties();
      // 实力化工厂对象
      ObjectFactory factory = (ObjectFactory) resolveClass(type).newInstance();
      factory.setProperties(properties);
      // 保存
      configuration.setObjectFactory(factory);
    }
  }
```

##### objectWrapperFactory

```java
// 读取配置中的定义并且注册到Configuration中
// objectWrapperFactory 用于感知创建对象
private void objectWrapperFactoryElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).newInstance();
      configuration.setObjectWrapperFactory(factory);
    }
  }
```

```java
// 读取配置中的定义并且注册到Configuration中
private void reflectorFactoryElement(XNode context) throws Exception {
    if (context != null) {
       String type = context.getStringAttribute("type");
       ReflectorFactory factory = (ReflectorFactory) resolveClass(type).newInstance();
       configuration.setReflectorFactory(factory);
    }
  }
```

##### reflectorFactory

```java
private void reflectorFactoryElement(XNode context) throws Exception {
    if (context != null) {
       String type = context.getStringAttribute("type");
       ReflectorFactory factory = (ReflectorFactory) resolveClass(type).newInstance();
       configuration.setReflectorFactory(factory);
    }
  }
```

##### settings

settings:它们会改变 MyBatis 的运行时行为

| 设置名       | 描述                                                     | 有效值        | 默认值 |
| :----------- | :------------------------------------------------------- | :------------ | :----- |
| cacheEnabled | 全局性地开启或关闭所有映射器配置文件中已配置的任何缓存。 | true \| false | true   |

```java
// 校验<Settings>标签中的属性是否都能被识别
private Properties settingsAsProperties(XNode context) {
    if (context == null) {
      return new Properties();
    }
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
      if (!metaConfig.hasSetter(String.valueOf(key))) {
        throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
      }
    }
    return props;
  }
```

```java
private void settingsElement(Properties props) throws Exception {
  // 设置配置
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
    configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
    configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
    configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
    configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
    configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
    configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
    configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
    configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
    configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
    configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
    configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
    configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
    configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
    configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
    configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
    configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
    @SuppressWarnings("unchecked")
    Class<? extends TypeHandler> typeHandler = (Class<? extends TypeHandler>)resolveClass(props.getProperty("defaultEnumTypeHandler"));
    configuration.setDefaultEnumTypeHandler(typeHandler);
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
    configuration.setLogPrefix(props.getProperty("logPrefix"));
    @SuppressWarnings("unchecked")
    Class<? extends Log> logImpl = (Class<? extends Log>)resolveClass(props.getProperty("logImpl"));
    configuration.setLogImpl(logImpl);
    configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
  }
```

##### environments

```java
environments:环境配置
尽管可以配置多个环境，但每个 SqlSessionFactory 实例只能选择一种环境。
每个数据库对应一个 SqlSessionFactory 实例

<environments default="development">
  <environment id="development">
    <transactionManager type="JDBC">
      <property name="..." value="..."/>
    </transactionManager>
    <dataSource type="POOLED">
      <property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment>
</environments>

MyBatis 中有两种类型的事务管理器（也就是 type="[JDBC|MANAGED]"）：

JDBC – 这个配置直接使用了 JDBC 的提交和回滚设施，它依赖从数据源获得的连接来管理事务作用域。
MANAGED – 这个配置几乎没做什么。它从不提交或回滚一个连接，而是让容器来管理事务的整个生命周期（比如 JEE 应用服务器的上下文）。 默认情况下它会关闭连接。然而一些容器并不希望连接被关闭，因此需要将 closeConnection 属性设置为 false 来阻止默认的关闭行为
如果你正在使用 Spring + MyBatis，则没有必要配置事务管理器，因为 Spring 模块会使用自带的管理器来覆盖前面的配置。

dataSource 元素使用标准的 JDBC 数据源接口来配置 JDBC 连接对象的资源。
有三种内建的数据源类型（也就是 type="[UNPOOLED|POOLED|JNDI]"）：

UNPOOLED– 这个数据源的实现会每次请求时打开和关闭连接
  driver – 这是 JDBC 驱动的 Java 类全限定名（并不是 JDBC 驱动中可能包含的数据源类）。
  url – 这是数据库的 JDBC URL 地址。
  username – 登录数据库的用户名。
  password – 登录数据库的密码。
  defaultTransactionIsolationLevel – 默认的连接事务隔离级别。
  defaultNetworkTimeout – 等待数据库操作完成的默认网络超时时间（单位：毫秒）

POOLED– 这种数据源的实现利用“池”的概念将 JDBC 连接对象组织起来，避免了创建新的连接实例时所必需的初始化和认证时间。 这种处理方式很流行，能使并发 Web 应用快速响应请求。还有更多属性用来配置 POOLED 的数据源：
  poolMaximumActiveConnections – 在任意时间可存在的活动（正在使用）连接数量，默认值：10
  poolMaximumIdleConnections – 任意时间可能存在的空闲连接数。
  poolMaximumCheckoutTime – 在被强制返回之前，池中连接被检出（checked out）时间，默认值：20000 毫秒（即 20 秒）
  poolTimeToWait – 这是一个底层设置，如果获取连接花费了相当长的时间，连接池会打印状态日志并重新尝试获取一个连接（避免在误配置的情况下一直失败且不打印日志），默认值：20000 毫秒（即 20 秒）。
  poolMaximumLocalBadConnectionTolerance – 这是一个关于坏连接容忍度的底层设置， 作用于每一个尝试从缓存池获取连接的线程。 如果这个线程获取到的是一个坏的连接，那么这个数据源允许这个线程尝试重新获取一个新的连接，但是这个重新尝试的次数不应该超过 poolMaximumIdleConnections 与 poolMaximumLocalBadConnectionTolerance 之和。 默认值：3（新增于 3.4.5）
  poolPingQuery – 发送到数据库的侦测查询，用来检验连接是否正常工作并准备接受请求。默认是“NO PING QUERY SET”，这会导致多数数据库驱动出错时返回恰当的错误消息。
  poolPingEnabled – 是否启用侦测查询。若开启，需要设置 poolPingQuery 属性为一个可执行的 SQL 语句（最好是一个速度非常快的 SQL 语句），默认值：false。
  poolPingConnectionsNotUsedFor – 配置 poolPingQuery 的频率。可以被设置为和数据库连接超时时间一样，来避免不必要的侦测，默认值：0（即所有连接每一时刻都被侦测 — 当然仅当 poolPingEnabled 为 true 时适用）。

JNDI – 这个数据源实现是为了能在如 EJB 或应用服务器这类容器中使用，容器可以集中或在外部配置数据源，然后放置一个 JNDI 上下文的数据源引用
```

```java
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      // 如果在构建XMLConfigBuilder时没有指定选择的环境，就从xml中获取默认的环境
      if (environment == null) {
        environment = context.getStringAttribute("default");
      }
      // 解析多个环境
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        // 当前环境是否为xml中指定的环境，不是就不解析
        if (isSpecifiedEnvironment(id)) {
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          DataSource dataSource = dsFactory.getDataSource();
          // 组装事物、数据源
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
    }
  }

// 将获取事务工厂
 private TransactionFactory transactionManagerElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      Properties props = context.getChildrenAsProperties();
      TransactionFactory factory = (TransactionFactory) resolveClass(type).newInstance();
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a TransactionFactory.");
  }


// 获取数据源
 private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      Properties props = context.getChildrenAsProperties();
      DataSourceFactory factory = (DataSourceFactory) resolveClass(type).newInstance();
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
  }


```

##### databaseIdProvider

```java
databaseIdProvider:数据库厂商标识
MyBatis 可以根据不同的数据库厂商执行不同的语句，这种多厂商的支持是基于映射语句中的 databaseId 属性。 MyBatis 会加载带有匹配当前数据库 databaseId 属性和所有不带 databaseId 属性的语句。 如果同时找到带有 databaseId 和不带 databaseId 的相同语句，则后者会被舍弃。
支持多厂商数据库
<databaseIdProvider type="DB_VENDOR" />
```

```java
private void databaseIdProviderElement(XNode context) throws Exception {
  DatabaseIdProvider databaseIdProvider = null;
  if (context != null) {
    String type = context.getStringAttribute("type");
    // awful patch to keep backward compatibility
    if ("VENDOR".equals(type)) {
        type = "DB_VENDOR";
    }
    Properties properties = context.getChildrenAsProperties();
    databaseIdProvider = (DatabaseIdProvider) resolveClass(type).newInstance();
    databaseIdProvider.setProperties(properties);
  }
  Environment environment = configuration.getEnvironment();
  if (environment != null && databaseIdProvider != null) {
    String databaseId = databaseIdProvider.getDatabaseId(environment.getDataSource());
    configuration.setDatabaseId(databaseId);
  }
}
```

##### typeHandlers

```xml
typeHandlers:类型处理器,MyBatis 在设置预处理语句（PreparedStatement）中的参数或从结果集中取出一个值时， 都会用类型处理器将获取到的值以合适的方式转换成 Java 类型。
```

| 类型处理器              | Java 类型                      | JDBC 类型                            |
| :---------------------- | :----------------------------- | :----------------------------------- |
| `BooleanTypeHandler`    | `java.lang.Boolean`, `boolean` | 数据库兼容的 `BOOLEAN`               |
| `ByteTypeHandler`       | `java.lang.Byte`, `byte`       | 数据库兼容的 `NUMERIC` 或 `BYTE`     |
| `ShortTypeHandler`      | `java.lang.Short`, `short`     | 数据库兼容的 `NUMERIC` 或 `SMALLINT` |
| `IntegerTypeHandler`    | `java.lang.Integer`, `int`     | 数据库兼容的 `NUMERIC` 或 `INTEGER`  |
| `LongTypeHandler`       | `java.lang.Long`, `long`       | 数据库兼容的 `NUMERIC` 或 `BIGINT`   |
| `FloatTypeHandler`      | `java.lang.Float`, `float`     | 数据库兼容的 `NUMERIC` 或 `FLOAT`    |
| `DoubleTypeHandler`     | `java.lang.Double`, `double`   | 数据库兼容的 `NUMERIC` 或 `DOUBLE`   |
| `BigDecimalTypeHandler` | `java.math.BigDecimal`         | 数据库兼容的 `NUMERIC` 或 `DECIMAL`  |

```java
重写已有的类型处理器或创建你自己的类型处理器来处理不支持的或非标准的类型.实现 org.apache.ibatis.type.TypeHandler 接口， 或继承一个很便利的类 org.apache.ibatis.type.BaseTypeHandler， 并且可以（可选地）将它映射到一个 JDBC 类型
  
  /***
* 将 JdbcType 转化成String 类型
**/
  @MappedJdbcTypes(JdbcType.VARCHAR)
public class ExampleTypeHandler extends BaseTypeHandler<String> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
    ps.setString(i, parameter);
  }

  @Override
  public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
    return rs.getString(columnName);
  }

  @Override
  public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    return rs.getString(columnIndex);
  }

  @Override
  public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    return cs.getString(columnIndex);
  }
}

通过类型处理器的泛型，MyBatis 可以得知该类型处理器处理的 Java 类型，不过这种行为可以通过两种方法改变：
在类型处理器的配置元素（typeHandler 元素）上增加一个 javaType 属性（比如：javaType="String"）；
在类型处理器的类上增加一个 @MappedTypes 注解指定与其关联的 Java 类型列表。 如果在 javaType 属性中也同时指定，则注解上的配置将被忽略。
 
可以通过两种方式来指定关联的 JDBC 类型：
在类型处理器的配置元素上增加一个 jdbcType 属性（比如：jdbcType="VARCHAR"）；
在类型处理器的类上增加一个 @MappedJdbcTypes 注解指定与其关联的 JDBC 类型列表。 如果在 jdbcType 属性中也同时指定，则注解上的配置将被忽略。
```

```java
// 获取javaType、jdbcType、handler对应的对象
// 无论是哪一种组合都有重载的方法使类型处理器 注册到typeHandlerRegistry 中
private void typeHandlerElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 如果指定类型处理器所在的包
        if ("package".equals(child.getName())) {
          String typeHandlerPackage = child.getStringAttribute("name");
          typeHandlerRegistry.register(typeHandlerPackage);
        } else {
          String javaTypeName = child.getStringAttribute("javaType");
          String jdbcTypeName = child.getStringAttribute("jdbcType");
          String handlerTypeName = child.getStringAttribute("handler");
          Class<?> javaTypeClass = resolveClass(javaTypeName);
          JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
          Class<?> typeHandlerClass = resolveClass(handlerTypeName);
          if (javaTypeClass != null) {
            if (jdbcType == null) {
              typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
            } else {
              typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
            }
          } else {
            typeHandlerRegistry.register(typeHandlerClass);
          }
        }
      }
    }
  }

// 注册包中的类型处理器
public void register(String packageName) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(TypeHandler.class), packageName);
    Set<Class<? extends Class<?>>> handlerSet = resolverUtil.getClasses();
    for (Class<?> type : handlerSet) {
      //Ignore inner classes and interfaces (including package-info.java) and abstract classes
      if (!type.isAnonymousClass() && !type.isInterface() && !Modifier.isAbstract(type.getModifiers())) {
        register(type);
      }
    }
  }
```

##### mappers

```xml
mappers:映射器

<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>

<!-- 使用完全限定资源定位符（URL） -->
<mappers>
  <mapper url="file:///var/mappers/AuthorMapper.xml"/>
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
  <mapper url="file:///var/mappers/PostMapper.xml"/>
</mappers>

<!-- 使用映射器接口实现类的完全限定类名 -->
<mappers>
  <mapper class="org.mybatis.builder.AuthorMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <mapper class="org.mybatis.builder.PostMapper"/>
</mappers>

<!-- 将包内的映射器接口实现全部注册为映射器 -->
<mappers>
  <package name="org.mybatis.builder"/>
</mappers>
```

###### Cache 标签

```
默认情况下，只启用了本地的会话缓存，它仅仅对一个会话中的数据进行缓存。 要启用全局的二级缓存，只需要在你的 SQL 映射文件中添加一行：
映射语句文件中的所有 select 语句的结果将会被缓存。
映射语句文件中的所有 insert、update 和 delete 语句会刷新缓存。
缓存会使用最近最少使用算法（LRU, Least Recently Used）算法来清除不需要的缓存。
缓存不会定时进行刷新（也就是说，没有刷新间隔）。
缓存会保存列表或对象（无论查询方法返回哪种）的 1024 个引用。
缓存会被视为读/写缓存，这意味着获取到的对象并不是共享的，可以安全地被调用者修改，而不干扰其他调用者或线程所做的潜在修改。

只要影响本名称空间内的sql 语句

<cache
  eviction="FIFO"
  
  设置的值应该是一个以毫秒为单位的合理时间量
  flushInterval="60000"
  size="512"
  
  readOnly（只读）属性可以被设置为 true 或 false。只读的缓存会给所有调用者返回缓存对象的相同实例。 因此这些对象不能被修改。这就提供了可观的性能提升。而可读写的缓存会（通过序列化）返回缓存对象的拷贝。 速度上会慢一些，但是更安全，因此默认值是 false。
  readOnly="true"/>
  这个更高级的配置创建了一个 FIFO 缓存，每隔 60 秒刷新，最多可以存储结果对象或列表的 512 个引用，而且返回的对象被认为是只读的，
  
  可用的清除策略有：

LRU – 最近最少使用：移除最长时间不被使用的对象。
FIFO – 先进先出：按对象进入缓存的顺序来移除它们。
SOFT – 软引用：基于垃圾回收器状态和软引用规则移除对象。
WEAK – 弱引用：更积极地基于垃圾收集器状态和弱引用规则移除对象。
默认的清除策略是 LRU。
```



```xml
<select ... flushCache="false" useCache="true"/>
<insert ... flushCache="true"/>
<update ... flushCache="true"/>
<delete ... flushCache="true"/>
```

###### cache-ref

```xml
对某一命名空间的语句，只会使用该命名空间的缓存进行缓存或刷新。 但你可能会想要在多个命名空间中共享相同的缓存配置和实例。要实现这种需求，你可以使用 cache-ref 元素来引用另一个缓存。

<cache-ref namespace="com.someone.application.data.SomeMapper"/>
```

###### this.mapperElement(root.evalNode("mappers"));

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      // 遍历xml中添加的mapper 标签
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
          
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            try(InputStream inputStream = Resources.getResourceAsStream(resource)) {
              XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
              mapperParser.parse();
            }
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            try(InputStream inputStream = Resources.getUrlAsStream(url)){
              XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
              mapperParser.parse();
            }
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```

```
通过循环来遍历xml 中的多个mapper 标签
如果含有package 标签，表示将包内所有映射器接口注册为映射器
根据不同的引入mapper方式来解析。如果是resource 、url 标签，流程大致相同
```

```java
// 指定配置文件
if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
  // 获取配置文件对应的字节流
            try(InputStream inputStream = Resources.getResourceAsStream(resource)) {
              //通过XMLMapperBuilder解析XXXMapper.xml，可以看到这里构建的XMLMapperBuilde还传入了configuration,所以之后肯定是会将mapper封装到configuration对象中去的。
              XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
              mapperParser.parse();
            }
          }
```

```
实例化一个错误上下文对象,这个对象的作用就是把错误信息封装起来，如果出现错误就会调用这个对象的toString方法,里面详细描述了错误的位置
这个resource参数就是String类型的xml的名字

创建一个XMLMapperBuilder ，用于解析xml 形式的mapper
```



###### mapperParser.parse();

```java
class XMLMapperBuilder
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }

  // 重新解析之前解析不了的节点 ，因为文件配置的顺序的问题导致有的标签解析不到，前面的标签依赖后面的标签
    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }
```

```
一开始就一个判断这个xml是否被解析过了，configuration对象会维护一个Set<String>  loadedResources，这个集合中存放了所有已经被解析过的xml的名字
```

```java
private void configurationElement(XNode context) {
    try {
      // 获取命名空间
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.isEmpty()) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
// <cache-ref>：cache只对特定的Namespace使用，即每个namespace使用一个cache实例，如果要多个namespace使用同一个cache实例，则可以使用cache-ref来引用
      cacheRefElement(context.evalNode("cache-ref"));
      //  <cache>：当前Mapper的缓存配置，二级缓存
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  
```

###### cacheRefElement

```java
private void cacheRefElement(XNode context) {
    if (context != null) {
      // 在Configuration.cacheRefMap 这个map对象中存储 当前命名空间与缓存命名空间之间的关系
      // key: 当前命名空间，value：指向的命名空间
      configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
      // 新建对象
      CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
      try {
        cacheRefResolver.resolveCacheRef();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteCacheRef(cacheRefResolver);
      }
    }
  }


public Cache useCacheRef(String namespace) {
    if (namespace == null) {
      throw new BuilderException("cache-ref element requires a namespace attribute.");
    }
    try {
      unresolvedCacheRef = true;
      Cache cache = configuration.getCache(namespace);
      if (cache == null) {
        throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.");
      }
      // 设置当前的缓存空间
      currentCache = cache;
      unresolvedCacheRef = false;
      return cache;
    } catch (IllegalArgumentException e) {
      throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.", e);
    }
  }
```

###### cacheElement

```java
// 解析cache 标签
private void cacheElement(XNode context) throws Exception {
    if (context != null) {
      // 获取自定义缓存
      String type = context.getStringAttribute("type", "PERPETUAL");
      // 获取缓存的类对象
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      // 获取清除策略
      String eviction = context.getStringAttribute("eviction", "LRU");
      // 获取清除策略的类对象
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      // 获取刷新间隔
      Long flushInterval = context.getLongAttribute("flushInterval");
      // 获取引用数目
      Integer size = context.getIntAttribute("size");
      // 获取是否只读
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      boolean blocking = context.getBooleanAttribute("blocking", false);
      Properties props = context.getChildrenAsProperties();
      //
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }
```

````java
public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
  // 创建对象
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
  // 添加到配置中
    configuration.addCache(cache);
  // 声明当前的缓存
    currentCache = cache;
    return cache;
  }
````



###### resultMapElements

```java
// 解析/mapper/resultMap 标签
private void resultMapElements(List<XNode> list) throws Exception {
  // 遍历每一条x ml
    for (XNode resultMapNode : list) {
      try {
        resultMapElement(resultMapNode);
      } catch (IncompleteElementException e) {
        // ignore, it will be retried
      }
    }
  }

private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping> emptyList());
  }

  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    // 是否继承其他mapper ,其他mapper 为父类
    String extend = resultMapNode.getStringAttribute("extends");
    // 是否开启自动映射
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    // 获取结果对应的类型
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<ResultMapping>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    // 遍历子标签
    for (XNode resultChild : resultChildren) {
      // 处理构造器标签
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        List<ResultFlag> flags = new ArrayList<ResultFlag>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
      // 将解析失败的标签记录下，后续补偿解析
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }
```

###### sqlElement

```java
private void sqlElement(List<XNode> list) throws Exception {
  // 如果指定了数据库id 
    if (configuration.getDatabaseId() != null) {
      sqlElement(list, configuration.getDatabaseId());
    }
  // 没有指定
    sqlElement(list, null);
  }

// 就是记录 sql的id 
private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
    for (XNode context : list) {
      String databaseId = context.getStringAttribute("databaseId");
      String id = context.getStringAttribute("id");
      id = builderAssistant.applyCurrentNamespace(id, false);
      if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
       // key: sql_id value: sql 语句
        sqlFragments.put(id, context);
      }
    }
  }
```

###### buildStatementFromContext

```java
// 首先判断有没有数据库ID
private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
  }

// 逐个遍历xml
  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }


// 先获取属性之后添加到
public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    Class<?> resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    String nodeName = context.getNode().getNodeName();
  //根据节点名，得到SQL操作的类型
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
   //判断是否是查询
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
  //是否刷新缓存 默认:增删改刷新 查询不刷新
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
   //是否使用二级缓存 默认值:查询使用 增删改不使用
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
  
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);
    
    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
  // 解析Sql（重要）  根据sql文本来判断是否需要动态解析 如果没有动态sql语句且 只有#{}的时候 直接静态解析使用?占位 当有 ${} 不解析
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    KeyGenerator keyGenerator;
  //设置主键自增规则
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```

###### 解析sql语句

XMLLanguageDriver 类的解析方法

```java
 @Override
  public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
    XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
    return builder.parseScriptNode();
  }

public SqlSource parseScriptNode() {
    MixedSqlNode rootSqlNode = parseDynamicTags(context);
    SqlSource sqlSource = null;
    if (isDynamic) {
      //如果是${}会直接不解析，等待执行的时候直接赋值
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
      // 用占位符方式来解析  #{} --> ?
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
  }

public DynamicSqlSource(Configuration configuration, SqlNode rootSqlNode) {
    this.configuration = configuration;
    this.rootSqlNode = rootSqlNode;
  }


public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> clazz = parameterType == null ? Object.class : parameterType;
  // 解析
    sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<String, Object>());
  }

public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    String sql = parser.parse(originalSql);
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
```

这里会生成一个GenericTokenParser，这个对象可以传入一个openToken和closeToken，如果是`#{}`，那么openToken就是`#{`，closeToken就是 `}`，然后通过parse方法中的`handler.handleToken()`方法进行替换。

在这之前由于已经进行过SQL是否含有`#{}`的判断了，所以在这里如果是只有`${}`，那么handler就是BindingTokenParser的实例化对象，如果存在`#{}`，那么handler就是ParameterMappingTokenHandler的实例化对象。



**每一条sql 对应一个MappedStatement**

**对外部 resultMap 的命名引用。结果映射是 MyBatis 最强大的特性，resultType 和 resultMap 之间只能同时使用一个。**

```xml
constructor - 用于在实例化类时，注入结果到构造方法中
idArg - ID 参数；标记出作为 ID 的结果可以帮助提高整体性能
arg - 将被注入到构造方法的一个普通结果

<!--基础，处理简单列-->
id – 一个 ID 结果；标记出作为 ID 的结果可以帮助提高整体性能
result – 注入到字段或 JavaBean 属性的普通结果


association – 一个复杂类型的关联；许多结果将包装成这种类型
嵌套结果映射 – 关联可以是 resultMap 元素，或是对其它结果映射的引用
collection – 一个复杂类型的集合
嵌套结果映射 – 集合可以是 resultMap 元素，或是对其它结果映射的引用
discriminator – 使用结果值来决定使用哪个 resultMap
case – 基于某些值的结果映射
嵌套结果映射 – case 也是一个结果映射，因此具有相同的结构和元素；或者引用其它的结果映射

<resultMap> 属性：
  id	当前命名空间中的一个唯一标识，用于标识一个结果映射。
	type	类的完全限定名, 或者一个类型别名（关于内置的类型别名，可以参考上面的表格）。
	autoMapping	如果设置这个属性，MyBatis 将会为本结果映射开启或者关闭自动映射。 这个属性会覆盖全局的属性 autoMappingBehavior。		默认值：未设置（unset）。
 <id>&<result> 属性：
property	 映射到列结果的字段或属性。如果 JavaBean 有这个名字的属性（property），会先使用该属性。否则 MyBatis 将会寻找给定	              字段（field）。 无论是哪一种情形，你都可以使用常见的点式分隔形式进行复杂属性导航。 比如，你可以这样映射一些简单的东                西：“username”，或者映射到一些复杂的东西上：“address.street.number”。
column	  数据库中的列名，或者是列的别名。一般情况下，这和传递给 resultSet.getString(columnName) 方法的参数一样。
javaType	一个 Java 类的全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）。 如果你映射到一个 JavaBean，MyBatis           通常可以推断类型。然而，如果你映射到的是 HashMap，那么你应该明确地指定 javaType 来保证行为与期望的相一致。
jdbcType	JDBC 类型，所支持的 JDBC 类型参见这个表格之后的“支持的 JDBC 类型”。 只需要在可能执行插入、更新和删除的且允许空值的列上           指定 JDBC 类型。这是 JDBC 的要求而非 MyBatis 的要求。如果你直接面向 JDBC 编程，你需要对可以为空值的列指定这个类型。
typeHandler	我们在前面讨论过默认的类型处理器。使用这个属性，你可以覆盖默认的类型处理器。 这个属性值是一个类型处理器实现类的全限定名，或者是类型别名。
```



 ```java
// 处理构造器标签
private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    List<XNode> argChildren = resultChild.getChildren();
    for (XNode argChild : argChildren) {
      List<ResultFlag> flags = new ArrayList<ResultFlag>();
      flags.add(ResultFlag.CONSTRUCTOR);
      if ("idArg".equals(argChild.getName())) {
        flags.add(ResultFlag.ID);
      }
      resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
  }


private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    String property;
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
      property = context.getStringAttribute("name");
    } else {
      property = context.getStringAttribute("property");
    }
  // 获取标签属性
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    String nestedResultMap = context.getStringAttribute("resultMap",
        processNestedResultMappings(context, Collections.<ResultMapping> emptyList()));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    Class<?> javaTypeClass = resolveClass(javaType);
    @SuppressWarnings("unchecked")
    Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
  // 构造对象
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
  }
 ```

```
开始解析 xxMaper.xml 文件
先判断是否有名称空间，否则抛异常
处理 标签
  cache-ref:引用其它命名空间的缓存配置。
  cache:  该命名空间的缓存配置。
  parameterMap： 被废弃
  resultMap：结果映射
  sql：这个元素可以用来定义可重用的 SQL 代码片段，以便在其它语句中使用
  select|insert|update|delete sql 动作
 sql语句是否指定了特定数据库厂商
 生成XMLStatementBuilder 每一个XMLStatementBuilder 对应一条sql
 解析sql 标签内每一个属性，封装成MappedStatement 对象，每个MappedStatement 都对应一条sql 标签
 将MappedStatement 加入到Configuration key:sql标签的id.value MappedStatement实例
 
```

###### configuration.addLoadedResource(resource);

```java
public void addLoadedResource(String resource) {
  // 保存已经解析的xml
    loadedResources.add(resource);
```

###### bindMapperForNamespace();

```java
private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
        // 获取名称空间对象
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        // ignore, bound type is not required
      }
      if (boundType != null && !configuration.hasMapper(boundType)) {
        // Spring may not know the real resource name so we set a flag
        // to prevent loading again this resource from the mapper interface
        // look at MapperAnnotationBuilder#loadXmlResource
        configuration.addLoadedResource("namespace:" + namespace);
        configuration.addMapper(boundType);
      }
    }
```

```
获取当前xml 文件中的名称空间  mapper接口的全限定名
反射出实例对象
判断configuration 中是否含有这个对象
将名称空间保存
将对象保存起来，key:mapper的class对象，value:通过动态代理生产的class对象的代理对象。
```

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        // 将MappedStatement 加入到configuration中
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```

**每个sql标签解析成mapperstatement对象装进集合，然后把mapper接口的class对象以及代理对象装进集合**

###### this.mapperElement(root.evalNode("mappers"));

```java
else if (resource == null && url == null && mapperClass != null) {
     Class<?> mapperInterface = Resources.classForName(mapperClass);
     configuration.addMapper(mapperInterface);
} 
```

```
反射出实例对象
将对象保存起来，key:mapper的class对象，value:通过动态代理生产的class对象的代理对象。
构建注解解析类
判断是否解析过mapper类
根据类名查找配置文件
解析配置文件
```



```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```

```java
class MapperAnnotationBuilder
public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      parseCache();
      parseCacheRef();
      for (Method method : type.getMethods()) {
        if (!canHaveStatement(method)) {
          continue;
        }
        if (getAnnotationWrapper(method, false, Select.class, SelectProvider.class).isPresent()
            && method.getAnnotation(ResultMap.class) == null) {
          parseResultMap(method);
        }
        try {
          parseStatement(method);
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
  }
```

```java
private void loadXmlResource() {
    // Spring may not know the real resource name so we check a flag
    // to prevent loading again a resource twice
    // this flag is set at XMLMapperBuilder#bindMapperForNamespace
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
      String xmlResource = type.getName().replace('.', '/') + ".xml";
      // #1347
      InputStream inputStream = type.getResourceAsStream("/" + xmlResource);
      if (inputStream == null) {
        // Search XML mapper that is not in the module but in the classpath.
        try {
          inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
        } catch (IOException e2) {
          // ignore, resource is not required
        }
      }
      if (inputStream != null) {
        XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
        xmlParser.parse();
      }
    }
  }
```

###### this.mapperElement(root.evalNode("mappers"));3

```java
if ("package".equals(child.getName())) {
  // 获取包的名称
          String mapperPackage = child.getStringAttribute("name");
  
          configuration.addMappers(mapperPackage);
} 

```

```java
public void addMappers(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
  // 找到包下面所有的类
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      addMapper(mapperClass);
    }
  }

public <T> void addMapper(Class<T> type) {
  // 包下面是不是接口，不是不处理
    if (type.isInterface()) {
      // 是否已经处理过
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      // 记录没有处理过
      boolean loadCompleted = false;
      try {
        // 放入到已处理Map中
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }

public MapperAnnotationBuilder(Configuration configuration, Class<?> type) {
  // 更改资源路径
    String resource = type.getName().replace('.', '/') + ".java (best guess)";
    this.assistant = new MapperBuilderAssistant(configuration, resource);
    this.configuration = configuration;
    this.type = type;

    sqlAnnotationTypes.add(Select.class);
    sqlAnnotationTypes.add(Insert.class);
    sqlAnnotationTypes.add(Update.class);
    sqlAnnotationTypes.add(Delete.class);

    sqlProviderAnnotationTypes.add(SelectProvider.class);
    sqlProviderAnnotationTypes.add(InsertProvider.class);
    sqlProviderAnnotationTypes.add(UpdateProvider.class);
    sqlProviderAnnotationTypes.add(DeleteProvider.class);
  }
```

@SelectProvider：这种是直接用于接口中的方法，来代替书写sql

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface InsertProvider {
    // 用于指定获取 sql 语句的类
    Class<?> type();
    // 指定类中要执行获取 sql 语句的方法
    String method();
}

public interface AutoConstructorMapper {
    @SelectProvider(type = SubjectSqlProvider.class, method = "getSubjectTestProvider")
    PrimitiveSubject getSubjectTestProvider(@Param("id") int id);
}


/**
* 方法入参必须为 Map
* 方法的权限修饰符 必须是 public
* 方法返回的必须是拼接好的 sql 字符串
* 用于外部引用sql
*/
public class SubjectSqlProvider {
    public String getSubjectTestProvider(Map<String, Object> params) {
        return new SQL()
                .SELECT("*")
                .FROM("subject")
                .WHERE("id = " + params.get("id"))
                .toString();
    }
}
```

```
解析通过包定义方式
与解析类的方式相同
```

## 3.2 获取session会话

```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
// 返回的是DefaultSqlSessionFactory

SqlSession session = sqlSessionFactory.openSession();
```

```
1、创建事务工厂
2、实例化一个事务对象
3、生成一个执行器Executor
4、根据类型生成执行器 SIMPLE, REUSE, BATCH
```



**通过SqlSession，您可以执行sql命令、获取映射器和管理事务。**

```java
class DefaultSqlSessionFactory
  /***
  * 默认使用 ExecutorType.SIMPLE;
  */
@Override
  public SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level) {
    return openSessionFromDataSource(execType, level, false);
  }

？ 什么时候设置的ExecutorType：默认值
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // 获取解析好的环境
      final Environment environment = configuration.getEnvironment();
      // 从环境中获取事务工厂
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 创建一个事务
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  // 设置默认值
        executorType = executorType == null ? this.defaultExecutorType : executorType;
        executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
        Object executor;
  
  // 配置指定的
        if (ExecutorType.BATCH == executorType) {
            executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
            executor = new ReuseExecutor(this, transaction);
        } else {
            executor = new SimpleExecutor(this, transaction);
        }

  // 是否开启缓存
        if (this.cacheEnabled) {
            executor = new CachingExecutor((Executor)executor);
        }

  // 如果拦截器链中执行执行器
        Executor executor = (Executor)this.interceptorChain.pluginAll(executor);
        return executor;
    }
```

```
SimpleExecutor: 简单执行器，是 MyBatis 中默认使用的执行器，每执行一次 update 或 select，就开启一个 Statement 对象，用完就直接关闭 Statement 对象(可以是 Statement 或者是 PreparedStatment 对象)

ReuseExecutor: 可重用执行器，这里的重用指的是重复使用 Statement，它会在内部使用一个 Map 把创建的 Statement 都缓存起来，每次执行 SQL 命令的时候，都会去判断是否存在基于该 SQL 的 Statement 对象，如果存在 Statement 对象并且对应的 connection 还没有关闭的情况下就继续使用之前的 Statement 对象，并将其缓存起来。

BatchExecutor: 批处理执行器，用于将多个SQL一次性输出到数据库
```

## 3.3 sql执行

### 3.3.1 获取sqlSession

SqlSession实际上是我们操作数据库的一个真实对象

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
 }

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // 获取之前解析好的环境参数
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 创建执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      // 构造sqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

// 都会创建一个事物工厂，就算没有手动配置，也会创建一个默认的ManagedTransactionFactory
private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
    if (environment == null || environment.getTransactionFactory() == null) {
      return new ManagedTransactionFactory();
    }
    return environment.getTransactionFactory();
  }

// 根据配置创建执行器
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }

```

### 3.3.2 获取mapper对象

```java
// 使用 DefaultSqlSession的getMapper 方法
// 这时每次都生成新的执行器
StudentMapper mapper = session.getMapper(StudentMapper.class);
```

```java
@Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }

public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  // 获取工厂
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      // 通过反射创建代理对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }

public T newInstance(SqlSession sqlSession) {
  // 代理对象
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

从knownMappers 中获取工厂，knownMappers是在解析xml文件的时候将mapper.xml里面MappedStatement 的类型缓存起来

### 3.3.3 执行

```java
List<StudentPO> studentPOS = mapper.selectAll();
```

根据反射规则，实际执行的时候会转换到代理对象的invoke（）上

```java
@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        // 判断方法所属的类
        //是不是调用的Object默认的方法
        //如果是  则不代理，不改变原先方法的行为
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
         //对于默认方法的处理
          //判断是否为default方法，即接口中定义的默认方法。
          //如果是接口中的默认方法则把方法绑定到代理对象中然后调用。
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // mybatis 实际执行位置
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

private MapperMethod cachedMapperMethod(Method method) {
 		 //动态代理会有缓存，computeIfAbsent 如果缓存中有则直接从缓存中拿
      //如果缓存中没有，则new一个然后放入缓存中
      //因为动态代理是很耗资源的
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
```

mapperMethod.execute(sqlSession, args):

```java
// 判断实际sql 是那种类型，根据不同的类型有不同的处理
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          // 执行没有返回值的
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          // 执行多个返回值
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          // 执行返回是map的
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          // 执行返回是cursor
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          // 执行查询单个的
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

executeForMany(sqlSession, args);

```java
private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
 
    if (method.hasRowBounds()) {
      // 分页的情况
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
      // 不分页的情况
      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      // 如果xml中定义的返回不是list,这里转换成list
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }


@Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      // 之前解析配置文件的时候将MappedStatement 放到configuration里面
      MappedStatement ms = configuration.getMappedStatement(statement);
      // executor 可以是BaseExecutor ，也可以是CachingExecutor ，CachingExecutor 是使用缓存的情况
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

CachingExecutor#query

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 获取sql 语句
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }


@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    // 二级缓存
    Cache cache = ms.getCache();
    if (cache != null) {
      // 二级缓存不为空
      // 有需要的话就刷新缓存
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          // 缓存中数据为空，查询一级缓存，delegate默认是BaseExecutor
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          // 放入二级缓存
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    // 直接查询一级缓存
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

BaseExecutor#query

一级缓存查询

```java
ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 一级缓存
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
         //对于存储过程有输出资源的处理
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
         //如果缓存为空，则从数据库拿
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;



private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
  // 先暂时在一级缓存中占个位置
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      // 由子类去实现
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      // 向一级缓存中放入实际的结果
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```



SimpleExecutor#doQuery

```java
@Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
     
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //#{} -> ? 的SQL在这里初始化
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

 public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
   // 实际上创建的是RoutingStatementHandler
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
   // 执行插件
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }


private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
  // 获取链接
    Connection connection = getConnection(statementLog);
  // 创建Statement
    stmt = handler.prepare(connection, transaction.getTimeout());
  // 参数填充
    handler.parameterize(stmt);
    return stmt;
  }

```

PreparedStatementHandler#parameterize

```java
@Override
  public void parameterize(Statement statement) throws SQLException {
    // 调用parameterHandler 来设置参数
    parameterHandler.setParameters((PreparedStatement) statement);
  }
```

```java
@Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    // 参数列表
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          //拿到xml中#{}   参数的名字  例如 #{id}  propertyName==id
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            //metaObject存储了参数名和参数值的对应关系
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            //在这里给preparedStatement赋值，使用TypeHandler ，不同的类型有不同的赋值实现
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }


```



PreparedStatementHandler#query

```java
@Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 执行sql
    ps.execute();
    // 处理结果
    return resultSetHandler.<E> handleResultSets(ps);
  
```

**无论是如何定义sql语句【注解、xml文件、Mapper方法】最终都可能是转化成SqlSession 的方法**

```java
<T> T selectOne(String statement);

// 检索从语句键和参数映射的单行。
<T> T selectOne(String statement, Object parameter);

// 从语句键和参数中检索映射对象列表。
<E> List<E> selectList(String statement);

// 在指定的行范围内，从语句键和参数中检索映射对象列表
<E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);

// selectMap 是一种特殊情况，它旨在根据结果对象中的一个属性将结果列表转换为 Map。 例如。 为 selectMap("selectAuthors","id") 返回 Map[Integer,Author] 
<K, V> Map<K, V> selectMap(String statement, String mapKey);

// Cursor 提供与 List 相同的结果，除了它使用 Iterator 延迟获取数据。
<T> Cursor<T> selectCursor(String statement);

// 使用ResultHandler检索从语句键和参数映射的ResultHandler 。
void select(String statement, Object parameter, ResultHandler handler);

// 执行插入语句。
int insert(String statement);

// 执行更新语句。 将返回受影响的行数。
int update(String statement);

// 执行删除语句。 将返回受影响的行数
int delete(String statement);

// 刷新批处理语句并提交数据库连接。 请注意，如果没有调用更新/删除/插入，则不会提交数据库连接。 强制提交调用commit(boolean)
void commit();

// 丢弃挂起的批处理语句并回滚数据库连接。 请注意，如果没有调用更新/删除/插入，则不会回滚数据库连接。 强制回滚调用rollback(boolean)
void rollback();

// 清除本地会话缓存
void clearCache();

// 检索内部数据库连接
Connection getConnection();
```

### 3.3.4 封装返回值

DefaultResultSetHandler#handleResultSets

```java
@Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    //resultMap可以通过多个标签指定多个值，所以存在多个结果集
    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
     //拿到当前第一个结果集
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    // 拿到所有的resultMap
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    //resultMap的数量
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    //循环处理每一个结果集
    while (rsw != null && resultMapCount > resultSetCount) {
      //开始封装结果集 list.get(index) 获取结果集
      ResultMap resultMap = resultMaps.get(resultSetCount);
      // 传入resultMap处理结果集 rsw 当前结果集
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }
    return collapseSingleResultList(multipleResults);
  }

private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          // 处理行数据
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }


public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      //存在内嵌的结果集
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      // 不存在内嵌的结果集
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }

private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    skipRows(rsw.getResultSet(), rowBounds);
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
       //遍历结果集
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
      //拿到行数据，将行数据包装成一个Object
      Object rowValue = getRowValue(rsw, discriminatedResultMap);
      storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
  }
```

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    //创建一个空对象装行数据
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, null);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      // 通过反射操作返回值
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      boolean foundValues = this.useConstructorMappings;
      // 是否使用自动装配，自动装配就是没显示配置resultMap
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
      }
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
  }

private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
      for (UnMappedColumnAutoMapping mapping : autoMapping) {
        // 通过类型处理器获取返回结果
        final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          // 在这里赋值
          metaObject.setValue(mapping.property, value);
        }
      }
    }
    return foundValues;
  }
```

在赋值的时候，因为我们的返回类型可以是bean，也可以是map,处理时需要兼顾两种情况

```java
public void setValue(String name, Object value) {
    PropertyTokenizer prop = new PropertyTokenizer(name);
    if (prop.hasNext()) {
      MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
      if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
        if (value == null && prop.getChildren() != null) {
          // don't instantiate child path if value is null
          return;
        } else {
          // 最终都会调用objectWrapper.set方法
          metaValue = objectWrapper.instantiatePropertyValue(name, prop, objectFactory);
        }
      }
      metaValue.setValue(prop.getChildren(), value);
    } else {
      // 如果返回的是map就调用MapWrapper的set方法，如果是Bean就调用BeanWrapper的set方法
      objectWrapper.set(prop, value);
    }
  }
```



​		<img src="/Users/peilizhi/Desktop/流程图.jpg" alt="流程图" style="zoom:50%;" />

# 四、整合springboot

## 4.1 引入配置类

springboot会找寻 MATE-INF里面的spring.factories里面的autoConfig

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

MybatisAutoConfiguration

会装配SqlSessionFactory，SqlSessionTemplate

```java
@Configuration
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnBean({DataSource.class})
@EnableConfigurationProperties({MybatisProperties.class})
@AutoConfigureAfter({DataSourceAutoConfiguration.class})
public class MybatisAutoConfiguration {
    private static final Logger logger = LoggerFactory.getLogger(MybatisAutoConfiguration.class);
    private final MybatisProperties properties;
    private final Interceptor[] interceptors;
    private final ResourceLoader resourceLoader;
    private final DatabaseIdProvider databaseIdProvider;
    private final List<ConfigurationCustomizer> configurationCustomizers;

    public MybatisAutoConfiguration(MybatisProperties properties, ObjectProvider<Interceptor[]> interceptorsProvider, ResourceLoader resourceLoader, ObjectProvider<DatabaseIdProvider> databaseIdProvider, ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider) {
        this.properties = properties;
        this.interceptors = (Interceptor[])interceptorsProvider.getIfAvailable();
        this.resourceLoader = resourceLoader;
        this.databaseIdProvider = (DatabaseIdProvider)databaseIdProvider.getIfAvailable();
        this.configurationCustomizers = (List)configurationCustomizersProvider.getIfAvailable();
    }

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        factory.setVfs(SpringBootVFS.class);
        if (StringUtils.hasText(this.properties.getConfigLocation())) {
            factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
        }

        org.apache.ibatis.session.Configuration configuration = this.properties.getConfiguration();
        if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
            configuration = new org.apache.ibatis.session.Configuration();
        }

        if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
            Iterator var4 = this.configurationCustomizers.iterator();

            while(var4.hasNext()) {
                ConfigurationCustomizer customizer = (ConfigurationCustomizer)var4.next();
                customizer.customize(configuration);
            }
        }

        factory.setConfiguration(configuration);
        if (this.properties.getConfigurationProperties() != null) {
            factory.setConfigurationProperties(this.properties.getConfigurationProperties());
        }

        if (!ObjectUtils.isEmpty(this.interceptors)) {
            factory.setPlugins(this.interceptors);
        }

        if (this.databaseIdProvider != null) {
            factory.setDatabaseIdProvider(this.databaseIdProvider);
        }

        if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
            factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
        }

        if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
            factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
        }

        if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
            factory.setMapperLocations(this.properties.resolveMapperLocations());
        }

        return factory.getObject();
    }

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        ExecutorType executorType = this.properties.getExecutorType();
        return executorType != null ? new SqlSessionTemplate(sqlSessionFactory, executorType) : new SqlSessionTemplate(sqlSessionFactory);
    }
```

## 4.2 @MapperScan

@mapperScan将Mapper类加入到容器中

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {
}
```

引入MapperScannerRegistrar 类

```java
// 实现ImportBeanDefinitionRegistrar接口
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {

  private ResourceLoader resourceLoader;

  /**
   * {@inheritDoc}
   */
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    // 获取MapperScan 注解
    AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // this check is needed in Spring 3.1
    if (resourceLoader != null) {
      scanner.setResourceLoader(resourceLoader);
    }

    // 解析MapperScan注解里面的参数，最终获取所有要扫描的包
    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
      scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      scanner.setMapperFactoryBean(BeanUtils.instantiateClass(mapperFactoryBeanClass));
    }

    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    List<String> basePackages = new ArrayList<String>();
    for (String pkg : annoAttrs.getStringArray("value")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (String pkg : annoAttrs.getStringArray("basePackages")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (Class<?> clazz : annoAttrs.getClassArray("basePackageClasses")) {
      basePackages.add(ClassUtils.getPackageName(clazz));
    }
    // 过滤
    scanner.registerFilters();
    // 对MapperScan 定义的basePackages进行扫描
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }
```

```java
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
  // 调用将搜索并注册所有候选人的父搜索。
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      // 然后对注册的对象进行后处理以将它们设置为 MapperFactoryBeans
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
```



ClassPathBeanDefinitionScanner#doScan

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
  // 遍历包
		for (String basePackage : basePackages) {
      // 获取包下类定义
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
          // 根据bean类型来设置属性
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
        
				if (checkCandidate(beanName, candidate)) {
          // 创建BeanDefinitionHolder
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
          // 注册
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

将接口的注册成BeanDefinitionHolder，但是这个BeanDefinition肯定不能实例化啦，

之后就后置处理BeanDefinitionHolder

processBeanDefinitions

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();

      if (logger.isDebugEnabled()) {
        logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
          + "' and '" + definition.getBeanClassName() + "' mapperInterface");
      }

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
      // 设置类型为mapperFactoryBean
      definition.setBeanClass(this.mapperFactoryBean.getClass());

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        if (logger.isDebugEnabled()) {
          logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        }
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
```

## 4.3 MapperFactoryBean

包扫描之后，已经将MapperFactoryBean 注册到容器内。

MapperFactoryBean是DaoSupport 的子类，DaoSupport 实现InitializingBean，checkDaoConfig（）会在执行InitializingBean#afterPropertiesSet中调用

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        // 将mapper加入到configuration中
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
}
```



