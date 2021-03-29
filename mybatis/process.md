# Mybatis 流程汇总说明 记录 mybatis 的大致零碎流程说明 ![](images/stratification.png) ### configuration 配置加载流程

mybatis 加载 xml 配置流程使用，

## 基础组成模块

### 解析模块

解析模块用于解析 XML 配置配置文件数据。

#### 关键类

- XMLMapperEntityResolver

用于返回需要解析的 Mybatis 的 DTD 文件，继承自 EntityResolver 接口。

- XPathParser

![](images/XPathParser_fields.png)

该类负责将 XML 文件解析成 document 对象（每个 XML 文件对应一个 document 和 XPathParse），使用 JDK 自带 StAX 的解析功能进行解析。依赖于 EntityResolver 对象（XMLMapperEntityResolver），并且对 document 对象进行数据解析返回节点数据。  
对于 String 的数据解析 Mybatis 会进行占位符的处理。

- PropertyParser

String 数据结果解析器，该类中定义了是否开启默认值功能以及默认值分隔符。

- GenericTokenParser

通用占位符解析类，根据占位符的开启和结尾。找到占位符的字面值，并将找到的字面值传递给 TokenHandler 进行解析，然后将解析后的结果重新拼接成字符串并返回。

- VariableTokenHandler

实现 TokenHandler 接口。用于配置文件字符串配置数据占位符字面值解析，先判断本地是否有设置对应的 Properties 属性值，如果有则进行字面值的解析查找，包含默认值的处理逻辑。不存在则返回字面值（加上占位符修饰符号）。

- XMLConfigBuilder

该类主要通过 XML 配置文件来创建全局的 Configuration 核心对象。使用来建造者模式，将整个大的 XML 数据解析分割成对每个 xml 的节点来实现，每个 XML 节点都使用一个单独的方法来处理，这样实现了每个标签节点的解藕。

### 反射模块

mybatis 的反射内部封装类一个工具类。会记录对应类的字段和属性，set 方法与 get 方法。对于一个方法会通过前缀来判断是否是 get/set/is 等类型，会递归获取本身类和父类以及所有接口的方法。set 方法只会对只有一个方法的才计算在内。

#### 关键类

- Reflector

每个类都有一个对应的 Reflector 对象与之关联，里面记录需要反射类的信息。包含字段信息，get 方法，set 方法等集合。

```java
  // 原始类信息
  private final Class<?> type;
  // 包含set方法的字段名称
  private final String[] readablePropertyNames;
  // 包含get方法的字段名称
  private final String[] writablePropertyNames;
  // set方法名称集合，key - 字段名称， value - 方法信息
  private final Map<String, Invoker> setMethods = new HashMap<>();
  // get方法名称集合，key - 字段名称， value - 方法信息
  private final Map<String, Invoker> getMethods = new HashMap<>();
  // set方法对应的参数类型
  private final Map<String, Class<?>> setTypes = new HashMap<>();
  // get方法对应的返回值类型
  private final Map<String, Class<?>> getTypes = new HashMap<>();
  // 默认的构造函数
  private Constructor<?> defaultConstructor;
  // 所有属性名称集合
  private Map<String, String> caseInsensitivePropertyMap = new HashMap<>();
```

### 类型转换

mybatis 类型转换负责数据库的 JDBC 类型与 JAVA 的类型映射。TypeHandler 接口定义了类型转换的统一入口。

TypeHandlerRegistry：每个类型转换的注册服务工具类，该类中注册了需要进行转换的类型实现类。

### Mybatis 中用到的是设计模式

- 单例模式：对于某些类对象在整个系统过程中只需要一个就可以满足的情况下。为了资源的开销方面和性能考虑在这个生命周期中我们会只创建一个对象来进行服务提供。单例模式的实现分为两种：延迟加载和启动加载，延迟加载推荐使用内部类静态属性实现，如果是通过双层校验的方式来创建对于定义的单例对象需要使用 volatile 的方式来进行修饰,防止乱序执行导致的并发问题。

- 适配器模式：对于统一的接口公开，由于各种原因具体的实现类不能直接通过统一的接口进行调用。在次需要通过一层转换才能使用，这种方式就是适配器模式。  
  使用模块：日志模块

- 装饰者模式：装饰者模式，主要体现的方式为 使用组合替代继承来实现某些功能。装饰器和组件的实现都实现某个统一的业务接口，组件的实现负责具体的单一模块功能。而装饰器中包含了这个组件对象，并且对该组件添加一些装饰功能来进行业务的扩展实现。

- 建造者模式：建造者模式也叫生成者模式，是将一个复杂对象的构建过程和它的表示分离，从而可以使用同样的构建过程创建不同的表示。建造者模式是将一个负责对象的创建过程分割成一个个简单的步骤，用户只需要了解复杂对象的类型和内容，而无需关心复杂对象的具体构建过程，帮组用户屏蔽了复杂对象的内部具体构建构建细节。

## xml 解析过程

### XMLConfigBuilder 解析

Mybatis 的 xml 解析过程中，就是一个建造者模式的过程。里面有个 Settings 的节点，该节点配置能够改变 Mybatis 的一些内置配置默认值。

1.  根据配置路径创建 XMLConfigBuilder 对象，创建该对象包含了一些内部所需类数据的初始化。  
    1.1 XPathParser 的构建  
    根据传入的配置文件路径信息和 Mybatis 的 DTD 文件解析对象来进行初始化操作。初始化过程中根据传入的参数获取需要解析的 xml 文件而生成 document 文档对象
    1.2 创建空 Configuration 对象，并且将参数中的属性信息添加到新创建的 Configuration 属性集合中。

2.  XMLConfigBuilder 进行 xml 文件的解析,对于文件的解析只允许执行一次。否则抛出异常信息，对于文档的解析 Mybatis 针对每一个不同的节点标签分别进行单独解析。  
    先获取整体的 configuration 节点数据，然后对里面的每个节点进行解析。整体解析步骤模块代码

```java
  private void parseConfiguration(XNode root) {
    try {
      // 解析属性信息
      propertiesElement(root.evalNode("properties"));
      // 解析全局配置项信息
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      // 配置项应用
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      // 解析别名
      typeAliasesElement(root.evalNode("typeAliases"));
      // 解析自定义插件数据
      pluginElement(root.evalNode("plugins"));
      // 解析自定义对象工厂
      objectFactoryElement(root.evalNode("objectFactory"));
      // 解析自定义对象封装工厂
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      // 解析自定义反射工厂
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      // 配置项应用
      settingsElement(settings);
      // 加载数据库环境信息
      environmentsElement(root.evalNode("environments"));
      // 解析数据库信息
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      // 解析自定义类型转换数据
      typeHandlerElement(root.evalNode("typeHandlers"));
      // 解析mapper的配置XML
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

2.1 重点说明  
typeAliases 解析：别名解析，对于 value 的 class 加载 Mybatis 会使用多个 classLoad 对来进行加载，对于别名的生成有两种方式。xml 配置中自定义别名或者使用 mybatis 的字段别名生成规则来创建，生成规则根据别名类中是否包含别名注解，存在则使用注解中的值否则使用别名类的简单名称。别名不能只能对应一个唯一的 Class 类且不区分大小写，如果存在同名不同类型的配置则抛出异常信息。

plugins 解析：对于插件的加载，插件类必须包含一个无参的构造函数，否则会初始化失败。初始化完成之后加入到 Configuration 配置 interceptorChain 集合中（内部是一个 ArrayList，查询是构建一个不可变集合信息）

environments 解析：数据库配置解析，数据库配置信息可以根据不同的环境进行不同的配置设置，但是 Mybatis 只能使用一个且必须指定其中的一个(可为空字符串)。事务类型必填信息，分为 jdbc 和 managed 两种类型（managed 类型的事务提交与回滚有管理者进行处理）根据指定的参数创建对应的事务管理工厂。接着创建数据源对象和 environment 对象，赋值给 Configuration。

databaseIdProvider：Mybatis 不能动态的处理多数据库的语法转换，通过该配置来告诉 Mybatis 当前的数据库产品类型。在执行 SQL 语句是根据该设置动态的选择对应的数据库 sql 模块。（if 标签判断 - \_databaseId）

typeHandlers：通过解析对应的 xml 节点获取数据，如果当前未设置 jdbcType 字段则根据指定的 TypeHandler 实现类中的 MappedJdbcTypes 注解数据来进行推断，而 javaType 则通过 MappedTypes 注解来进行推断获取。

mappers：解析对应的 sql 执行 xml 文件，有三种方式进行加载但是只能同时使用一直方式（resource,url,mapperClass）否则抛出异常信息。
mapper XML 配置文件使用单独的 XMLMapperBuilder 来进行解析构建，属于 BaseBuild 的子类。

### XMLMapperBuilder 解析

XMLMapperBuilder 用于解析 sql Mapper 文件，将之前解析的 Configuration 配置信息与需要加载的资源传递给 XMLMapperBuilder。由于 Mybatis 中允许存在多个 mapper 配置文件的存在，为了防止在加载过程中由于一些原因多次加载的发生。对于加载过的配置文件会存储在 configuration 的 loadedResources 中，在解析过程中发现已经加载过的 xml 文件则不会再次进行 xml 的解析。

```java
public void parse() {
// 已经加载过的配置文件则不会再次解析
if (!configuration.isResourceLoaded(resource)) {
    // 解析xml配置文件
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    bindMapperForNamespace();
}

parsePendingResultMaps();
parsePendingCacheRefs();
parsePendingStatements();
}
```

configurationElement 配置解析流程

```java
// xml mapper 文件的 namespace 不允许为空否则抛出异常信息
String namespace = context.getStringAttribute("namespace");
if (namespace == null || namespace.equals("")) {
    throw new BuilderException("Mapper's namespace cannot be empty");
}
// 将当前的命名空间赋值给辅助对象
builderAssistant.setCurrentNamespace(namespace);
// 解析引用缓存配置
cacheRefElement(context.evalNode("cache-ref"));
// 解析当前命名空间的缓存配置
cacheElement(context.evalNode("cache"));
parameterMapElement(context.evalNodes("/mapper/parameterMap"));
resultMapElements(context.evalNodes("/mapper/resultMap"));
sqlElement(context.evalNodes("/mapper/sql"));
buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
```

- 主要流程说明
1. 在解析cache-ref配置是，会创建一个CacheRefResolver对象。并且会进行缓存对象的尝试获取，如果当前引用的缓存mapper还没有进行加载则将该对象加入configuration的incompleteCacheRefs集合中。缓存获取成功则将缓存对象赋值给MapperBuilderAssistant的currentCache，后续直接获取该缓存对象，这样就少了一次对象的引用查询。
2. 

#### 重点说明

- MapperBuilderAssistant

```java
// xml 配置的命名空间
private String currentNamespace;
// xml 配置文件路径
private String resource;
// 当前mapper的缓存配置对象，可能使用的是其它mapper的缓存对象（共享）
private Cache currentCache;
// 缓存引用是否已经未验证
private boolean unresolvedCacheRef; // issue #676
```

mapper 配置文件辅助对象，记录 mapper 的命名空间信息等信息。如果当前

- CacheRefResolver

```java
// mapper 配置辅助对象
private final MapperBuilderAssistant assistant;
// 引用的缓存mapper配置文件命名空间名称
private final String cacheRefNamespace;
```

- 备注：Mybatis 在对加入的类型解析是会优先查询别名注册信息，如果存在别名注册信息则使用别名注册中的类型。只有对别名集合中不存在的类型名称才会进行类型的加载。

### 缓存实现

Mybatis 的缓存分为一级缓存和二级缓存，二级缓存通过 xml 配置中的<cache>标签来开启

## 主要类属性源码说明

### Configuration

```java
// 数据库环境配置
protected Environment environment;
// 安全行界限已启用
protected boolean safeRowBoundsEnabled;
// 安全结果处理已开启
protected boolean safeResultHandlerEnabled = true;
//
protected boolean mapUnderscoreToCamelCase;
protected boolean aggressiveLazyLoading;
protected boolean multipleResultSetsEnabled = true;
protected boolean useGeneratedKeys;
protected boolean useColumnLabel = true;
protected boolean cacheEnabled = true;
protected boolean callSettersOnNulls;
protected boolean useActualParamName = true;
protected boolean returnInstanceForEmptyRow;
protected String logPrefix;
protected Class<? extends Log> logImpl;
protected Class<? extends VFS> vfsImpl;
protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
protected Set<String> lazyLoadTriggerMethods = new HashSet<String>(Arrays.asList(new String[]{"equals", "clone", "hashCode", "toString"}));
protected Integer defaultStatementTimeout;
protected Integer defaultFetchSize;
protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;
protected AutoMappingUnknownColumnBehavior autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;
protected Properties variables = new Properties();
protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
protected ObjectFactory objectFactory = new DefaultObjectFactory();
protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
protected boolean lazyLoadingEnabled = false;
protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // #224 Using internal Javassist instead of OGNL
protected String databaseId;
protected Class<?> configurationFactory;
protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
protected final InterceptorChain interceptorChain = new InterceptorChain();
protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
protected final Map<String, Cache> caches = new StrictMap<Cache>("Caches collection");
protected final Map<String, ResultMap> resultMaps = new StrictMap<ResultMap>("Result Maps collection");
protected final Map<String, ParameterMap> parameterMaps = new StrictMap<ParameterMap>("Parameter Maps collection");
protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<KeyGenerator>("Key Generators collection");
// xml mapper 已价值配置
protected final Set<String> loadedResources = new HashSet<String>();
// xml mapper sql
protected final Map<String, XNode> sqlFragments = new StrictMap<XNode>("XML fragments parsed from previous mappers");
protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<XMLStatementBuilder>();
// 不完整的缓存引用配置，如果配置的mapper的xml文件配置的缓存为引用类型，但是在加载该mapper配置是引用的mapper还未加载则会加入该集合
protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<CacheRefResolver>();
protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<ResultMapResolver>();
protected final Collection<MethodResolver> incompleteMethods = new LinkedList<MethodResolver>();
// 缓存ref 映射配置，key 当前解析的mapper xml的namespace名称，value 引用的mapper namespace名称
protected final Map<String, String> cacheRefMap = new HashMap<String, String>();
```
