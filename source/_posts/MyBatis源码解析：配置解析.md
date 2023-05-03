---
title: MyBatis源码解析：配置解析
date: 2023-02-18 09:38:55
tags: MyBatis源码解析
categories: MyBatis
description: 配置详解。
---

> 为了行文方便，后文 MyBatis 都使用全小写：mybatis。

最开始学习 mybatis 时，都会先写一个 mybatis-config.xml，在里面对 mybatis 进行配置，包括导入 mapper 文件等，而 mapper 文件又是单独的一些 xxxMapper.xml 文件，这还不算完，在 Java 中使用时，还需要使用一个 SqlSessionFactoryBuilder 读取配置，得到 SqlSessionFactory，然后才能获取 Mapper 等对象进行业务上的 CRUD。

OK，其实配置解析最终就是为了得到一个 SqlSessionFactory。

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory =
  new SqlSessionFactoryBuilder().build(inputStream);
```

配置文件的内容就不赘述，主要关注如何解析，我们直接开冲。

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
      	if (inputStream != null) {
      	  inputStream.close();
      	}
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }

  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

1. 首先初始化一个 XMLConfigBuilder，它是 mybatis 提供的用来解析 xml 配置文件的解析器。
2. 调用 `XMLConfigBuilder#parse()` 方法进行解析，得到一个 Configuration 对象，这也是本文着重讲的地方。
3. 调用 `#build` 方法初始化了一个 `DefaultSqlSessionFactory` 对象。

# 初始化 Configuration 对象

Configuration 对象并不是调用  `XMLConfigBuilder#parse()` 才创建，而是在创建 XMLConfigBuilder 对象时，内部就已经创建了一个 Configuration 对象。

```java
  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
```

这里的 `new Configuration()` 本身也做了很多事。

```java
  public Configuration() {
    typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
    typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);

    typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
    typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
    typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);

    typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
    typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
    typeAliasRegistry.registerAlias("LRU", LruCache.class);
    typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
    typeAliasRegistry.registerAlias("WEAK", WeakCache.class);

    typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);

    typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
    typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);

    typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
    typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
    typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
    typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
    typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
    typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
    typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);

    typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
    typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);

    languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
    languageRegistry.register(RawLanguageDriver.class);
  }
```

可以看见这里注册了很多的类的别名，比如 `JDBC` 对应 `JdbcTransactionFactory`。

除此之外，还注册了一个默认的 DriverClass。

# 配置文件解析

```java
  // XMLConfigBuilder
	public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
```

1. 首先判断 parsed 是否为 true，如果是 true，表示已经在解析了，则直接抛出异常。
2. 开始解析之前，将 pared 置为 true，放置重复解析。
3. 调用 `#parseConfiguration(XNode)` 方法对 `<configuration>` 标签进行解析，这里可以看出是使用 xpath 的方式进行解析的。

```java
  private void parseConfiguration(XNode root) {
    try {
      // 解析 properties 标签
      propertiesElement(root.evalNode("properties"));
      // 解析 settings 标签
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      // 加载配置日志打印实现
      loadCustomLogImpl(settings);
      // 加载类型别名配置
      typeAliasesElement(root.evalNode("typeAliases"));
      // 加载插件配置
      pluginElement(root.evalNode("plugins"));
      // 加载 objectFactory 配置
      objectFactoryElement(root.evalNode("objectFactory"));
      // 加载 objectWrapperFactory 配置
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      // 加载 reflectorFactory 配置
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      // 初始化 setting，在此之前只是得到了各项配置的值，但是并没有放到 configuration 中
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      // 解析 environments 标签，支持多环境配置
      environmentsElement(root.evalNode("environments"));
      // 解析 databaseIdProvider 
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      // 解析 typeHandlers，类型转换处理器
      typeHandlerElement(root.evalNode("typeHandlers"));
      // 解析 mapper 文件
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

## ObjectFactory

ObjectFactory 是用于创建创建 sql 语句的返回值对象的，mybatis 中有默认实现 `DefaultObjectFactory`，只是使用默认的构造函数创建对象实例。

```java
public interface ObjectFactory {

  /**
   * 设置 properties
   * @param properties configuration properties
   */
  default void setProperties(Properties properties) {
    // NOP
  }

  /**
   * 通过默认构造函数创建一个对象实例
   *
   * @param <T>
   *          the generic type
   * @param type
   *          Object type
   * @return the t
   */
  <T> T create(Class<T> type);

  /**
   * 通过指定参数的构造函数创建一个对象实例
   *
   * @param <T>
   *          the generic type
   * @param type
   *          Object type
   * @param constructorArgTypes
   *          Constructor argument types
   * @param constructorArgs
   *          Constructor argument values
   * @return the t
   */
  <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs);

  /**
   * 判断指定类型是否为 Collection 类型
   *
   * @param <T>
   *          the generic type
   * @param type
   *          Object type
   * @return whether it is a collection or not
   * @since 3.1.0
   */
  <T> boolean isCollection(Class<T> type);

}
```

如果要自定义这段逻辑的话，就可以实现这个接口，然后使用 `objectFactory` 标签进行配置。

```xml
    <objectFactory type="org.apache.ibatis.submitted.global_variables_defaults.SupportClasses$CustomObjectFactory">
        <property name="name" value="${objectFactory.name:default}"/>
    </objectFactory>
```

## ObjectWrapperFactory

```java
public interface ObjectWrapperFactory {
  /**
   * 判断是否须要需要做wrapper封装
   */
  boolean hasWrapperFor(Object object);
  /**
   * 获取封装类
   */
  ObjectWrapper getWrapperFor(MetaObject metaObject, Object object);
}
```

这个接口提供了 2 个方法：

* `#hasWrapperFor(Object)` 方法用于判断是否需要做对象封装。
* `#getWrapperFor(MetaObject, Object)` 方法用于做具体的封装工作。

该接口的默认实现是 `DefaultObjectWrapperFactory`：

```java
public class DefaultObjectWrapperFactory implements ObjectWrapperFactory {

  @Override
  public boolean hasWrapperFor(Object object) {
    return false;
  }

  @Override
  public ObjectWrapper getWrapperFor(MetaObject metaObject, Object object) {
    throw new ReflectionException("The DefaultObjectWrapperFactory should never be called to provide an ObjectWrapper.");
  }
}
```

可以看见，在默认逻辑上，这个类是不能被调用的，换句话说，`ObjectWrapperFactory` 接口的功能在 mybatis 中是没有使用的。

MetaObject 类，从名称上就知道，是为了提供统一的元数据进行操作的工具类。

ObjectWrapper 接口则是通过 MetaObject 实现对对象的统一操作方法。

## settings 解析

settings 的解析，分了 2 步：

```java
  private void parseConfiguration(XNode root) {
    try {
			// ...
      // 解析 settings 标签
      Properties settings = settingsAsProperties(root.evalNode("settings"));
			// ...
      // 初始化 setting，在此之前只是得到了各项配置的值，但是并没有放到 configuration 中
      settingsElement(settings);
			// ...
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }

  private Properties settingsAsProperties(XNode context) {
    if (context == null) {
      return new Properties();
    }
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
      // 判断 Configuration 有没有 key 相应的 setter 方法
      if (!metaConfig.hasSetter(String.valueOf(key))) {
        throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
      }
    }
    return props;
  }

  private void settingsElement(Properties props) {
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
    configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
    configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
    // ...
  }
```

`#settingsAsProperties(XNode)` 方法，解析的是 `<settings>` 标签，首先解释将 `<settings>` 标签的子标签 `<setting>` 都解析成了 Properties，然后又逐一判断了 Configuration 类是否有相应的配置属性的 setter 方法，只要其中一个配置在 Configuration 中没有，则直接抛出异常。

`#settingElement(Properties)` 方法则就比较简单，因为前一个方法只是解析得到了每项配置的值，并未设置到 Configuration 类中，而这里就是单纯的做配置值的写入，对于 propeties 参数中没有的配置项，也都有相应的默认配置在此配置。

## environments 解析

```java
  private void environmentsElement(XNode context) throws Exception {
    // 如果 context 为 null 说明没有配置 environments
    if (context != null) {
      // 设置当前默认激活的环境配置 id
      if (environment == null) {
        // 获取 environments 标签的 default 属性值，作为当前默认激活的环境
        environment = context.getStringAttribute("default");
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        // 判断当前 environment 标签的 id 是否是指定的默认环境的 id，如果是，则解析相应的配置，否则直接跳过 
        if (isSpecifiedEnvironment(id)) {
          // 解析 transactionManager 标签
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          // 解析 dataSource 标签
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          DataSource dataSource = dsFactory.getDataSource();
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          // 设置 environment
          configuration.setEnvironment(environmentBuilder.build());
          break;
        }
      }
    }
  }
```

这里贴一段 environments 标签的配置片段，方便理解。

```xml
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC">
        <property name="" value=""/>
      </transactionManager>
      <dataSource type="UNPOOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
```

Environments 标签的解析主要分为 2 步：

1. 设置默认的环境 id。
2. 解析默认的环境配置。

默认的环境配置是必须设置，不然解析工作没法往下进行，并且在 `#isSpecifiedEnvironment(String)` 方法中，如果默认的环境未指定，即 `environment` 变量还是 null 的话，会直接报错。

解析默认的环境配置，又分为了 3 步：

1. 解析 `<transactionManager>` 标签，这里解析得到的是一个 `TransactionFactory` 对象。
2. 解析 `<dataSource>` 标签，这里解析得到的是一个 `DataSourceFactory` 对象，然后从该对象中才获取到了 DataSource。
3. 通过前 2 步解析的结果组装 `Environment` 对象，并设置进 configuration 中。

### transactionManager 标签解析

```java
  private TransactionFactory transactionManagerElement(XNode context) throws Exception {
    // 检查 transactionManager 是否配置，如果未配置，则抛异常
    if (context != null) {
      // 获取 type 属性字段值
      String type = context.getStringAttribute("type");
      // 将所有 property 子标签解析成一个 Properties 对象
      Properties props = context.getChildrenAsProperties();
      // 通过 type 得到 TransactionFactory 具体的实现类，并得到一个具体的实例
      TransactionFactory factory = (TransactionFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      // 通过 setProperties 方法设置属性
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a TransactionFactory.");
  }
```

1. 获取 type 字段值。
2. 解析 property 子标签，放入一个 Properties 对象。
3. 通过 type 获取 TransactionFactory 具体的实现，并得到实例化对象，深入查看 `#resolveClass(String)` 方法的逻辑，其实 type 是一个类别名，类别名除了在解析 `<typeAliases>` 标签时会注册，在 Configuration 类的默认无参构造中，也会注册很多。
4. 通过 `TransactionFactory#setProperties(Properties)` 方法设置属性。

这里再看看 TransactionFactory 接口：

```java
public interface TransactionFactory {

  /**
   * 设置自定义参数，接口的 default 方法，默认空实现，实现类可不实现该方法
   */
  default void setProperties(Properties props) {
  }

  /**
   * 从现有连接中新创建一个 Transaction 
   */
  Transaction newTransaction(Connection conn);

  /**
   * 从现有的 DataSource 中创建一个新的 Transaction
   */
  Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit);

}
```

可以看见，setProperties() 方法作为一个自定义实现的参数设置的方法，实现类可以不实现该方法。

### dataSource 标签解析

```java
  private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    // 判断 dataSource 是否配置，如果没配置直接抛异常
    if (context != null) {
      // 获取 type 属性值
      String type = context.getStringAttribute("type");
      // 解析 property 子标签
      Properties props = context.getChildrenAsProperties();
      // 通过 type 获取 DataSourceFactory 实现类，并实例化
      DataSourceFactory factory = (DataSourceFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      // 通过 setProperties 方法设置参数
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
  }
```

dataSource 标签的解析过程和 transactionManager 标签的解析逻辑基本一致，不再赘述，这里再看看 DataSourceFactory 接口的定义：

```java
public interface DataSourceFactory {
  void setProperties(Properties props);
  DataSource getDataSource();
}
```

可以看见，和 TransactionFactory 不同的是，DataSourceFacory 的 `#setProperties(Properties)` 方法是必须实现的。 

## mapper 解析

```java
private void mapperElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 判断子标签是否为 package
      if ("package".equals(child.getName())) {
        String mapperPackage = child.getStringAttribute("name");
        // 通过包名导入 mapper
        configuration.addMappers(mapperPackage);
      } else {
        // 解析 mapper 子标签
        // 获取 resource 属性值
        String resource = child.getStringAttribute("resource");
        // 获取 url 属性值
        String url = child.getStringAttribute("url");
        // 获取 class 属性值
        String mapperClass = child.getStringAttribute("class");
        if (resource != null && url == null && mapperClass == null) {
          // 只配置了 resource，加载相应的资源并通过 XMLMapperBuilder 进行解析，换言之，
          // 这里是导入了 Mapper 的 xml 配置文件
          ErrorContext.instance().resource(resource);
          try(InputStream inputStream = Resources.getResourceAsStream(resource)) {
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            // 解析 mapper xml 配置文件
            mapperParser.parse();
          }
        } else if (resource == null && url != null && mapperClass == null) {
          // 只配置了 url，依然是当成 xml 配置文件进行解析
          ErrorContext.instance().resource(url);
          try(InputStream inputStream = Resources.getUrlAsStream(url)){
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            // 解析 mapper xml 配置文件
            mapperParser.parse();
          }
        } else if (resource == null && url == null && mapperClass != null) {
          // 只配置了 class 
          Class<?> mapperInterface = Resources.classForName(mapperClass);
          configuration.addMapper(mapperInterface);
        } else {
          // url、class、resource 3 个属性，只能配置其中一个，否则解析过程就会抛出异常
          throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
        }
      }
    }
```

1. 如果子标签是 `<package>` 则按包名导入。
2. 如果子标签是 `<mapper>`，则开始解析其 resource、url、class 3 个属性，这 3 个属性只能配置其中一个，resource、url 指定的是 xml 配置文件，调用`XMLMapperBuilder#parse()` 方法进行解析和导入，而 class 属性指定的是 Java 类，加载了对应的 Class 对象之后，直接调用 `Configuration#addMapper(Class)` 方法进行导入，内部是直接调用了 `MapperRegistry#addMapper(Class)` 方法进行导入，其实通过包名导入时，最终也是通过这个方法进行的导入。

那么，接下来需要关注 2 个方法：

1. `XMLMapperBuilder#parse()` 方法，负责解析 mapper 的 xml 配置文件并进行导入。
2. `MapperRegistry#addMapper(Class)` 方法，直接通过 Class 进行导入。

### mapper xml 配置文件解析



### 通过 Class 导入 mapper

