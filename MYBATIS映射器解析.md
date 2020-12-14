 



[toc]



### Mybatis映射器

>映射器是MyBatis最强大的⼯具，也是我们使用MyBatis时⽤得最多的工具，因此熟
>练掌握它⼗分必要。MyBatis是针对映射器构造的SQL构建的轻量级框架，并且通过配置
>生成对应的JavaBean返回给调用者，⽽这些配置主要便是映射器，在MyBatis中你可以根
>据情况定义动态SQL来满足不同场景的需要，它比其他框架灵活得多。MyBatis还支持⾃动绑定JavaBean,
>我们只要让SQL返回的字段名和JavaBean 的属性名保持一致（或者采⽤驼峰式命名），便可以省掉这些繁琐的映射配置

### 映射器的主要元素

映射器是由Java接口和XML文件（或注解）共同组成的，Java接口主要定义调用者接口，XML文件是配置映射器的核心文件，包括以下元素：

---
- select 查询语句，可以自定义参数，返回结果集；

- insert 插入语句，返回一个整数，表示插入的条数；

- update 更新语句，返回一个整数，表示更新的条数；

- delete 删除语句，返回一个整数，表示删除的条数；

- sql 允许定义一部分SQL，然后再各个地方引用；

- resultMap 用来描述从数据库结果集中来加载对象，还可以配置关联关系,提供映射规则；

- cache 给定命名空间的缓存配置
---

### Select元素

>select元素帮助我们
>从数据库中读出数据，组装数据给业务人员。执行select语句前，我们需要定义参数，它
>可以是⼀个简单的参数类型，例如int, float , String,也可以是⼀个复杂的参数类型
> 例如
>JavaBean、 Map等，这些都是MyBatis接受的参数类型。

执⾏SQL后，MyBatis也提供了
强⼤的映射规则，自动映射来帮助我们把返回的结果集绑定到JavaBean中。

select
元素的配置众多,下面简单说明下：

- id: id和Mapper的命名空间组成唯一值,提供给Mybatis调用,如果不唯一将会报错

- paramterType：传入的参数类型，可以是基本类型、map、自定义的java bean；

- resultType：返回的结果类型，可以是基本类型、自定义的java bean；

- resultMap：它是最复杂的元素，可以配置映射规则、级联、typeHandler等，与ResultType不能同时存在；

- flushCache：在调用SQL后，是否要求清空之前查询的本地缓存和二级缓存，主要用于更新缓存，默认为false；

- useCache：启动二级缓存的开关，默认只会启动一级缓存；

- timeout：设置超时参数，等超时的时候将抛出异常，单位为秒；

- fetchSize：获取记录的总条数设定；
---

#### select实例

需求:  查询名称等于`JAVA宝典`的用户

在UserDao中定义接口方法:

```java
User findIdByName(String name);
```
定义UserMapper.xml

```xml
<select id="findIdByName" parameterType="string" resultType="User">
        select
        u.*
        from t_user u
        where u.name=#{name}
</select>
 
```
对操作步骤进行归纳概括:
- Id标出了了这条SQL
- parameterType定义参数类型
- resuitType定义返回值类型

上面的例子只是传入单个参数,多个参数可以使用Map,JavaBean,使用注解方式,等等,下面我们会单独介绍.

---


### insert元素

insert属性和select大部分都相同， 说下3个不同的属性：


- keyProperty：指定哪个列是主键，如果是联合主键可以用逗号隔开；

- keyColumn：指定第几列是主键，不能和keyProperty共用；

- useGeneratedKeys：是否使用自动增长，默认为false；当useGeneratedKeys设为true时，在插入的时候，会回填Java Bean的id值，通过返回的对象可获取主键值。

如果想根据一些特殊关系设置主键的值，可以在insert标签内使用selectKey标签

```xml
<insert id="insertRole" useGeneratedKeys="true" keyProperty="id" >
    <selectKey keyProperty="id" resultType="int" order="before">
        select if(max(id) is null,1,max(id)+2) as newId from t_role    
  </selectKey> 
</insert>
```
update和delete 就不单独过多介绍了 

### sql元素
sql元素的意义，在于我们可以定义⼀串串SQL语句的组成部分，其他的语句可以通过引⽤来使⽤它。

>例如，你有一条SQL需要select⼏⼗个字段映射到JavaBean中去，我的第二
>条SQL也是这⼏⼗个字段映射到JavaBean中去，显然这些字段写两遍不太合适。那么我们
>就⽤sql元素来完成

定义：
```xml
<sql id="columns">
    id,
    name,
    remark,
    tid
</sql>
```

使用：
```xml
	<select id="getOne" resultType="com.liangtengyu.entity.Ttest">
		select  <include refid="columns"/>  from t_test where id  = #{id}
	</select>
```



### resultMap元素
resultMap是MyBatis里面最复杂的元素，它的作用是定义映射规则、级联的更新、定制类型转换器等。

由以下元素构成：
```xml
<resultMap>
    <constructor> <!-- 配置构造方法 -->
        <idArg/>
        <arg/>
    </constructor>
    <id/> <!--指明哪一列是主键-->
    <result/> <!--配置映射规则-->
    <association/> <!--一对一-->
    <collection/> <!--一对多-->
    <discriminator> <!--鉴别器级联-->
        <case/>
    </discriminator>
</resultMap>
```
有的实体不存在没有参数的构造方法，需要使用constructor配置有参数的构造方法：
```xml
<resultMap id="role" type="com.liangtengyu.entity.Role">
    <constructor>
        <idArg column="id" javaType="int"/>
        <arg column="role_name" javaType="string"/>
    </constructor>
</resultMap>
```
id指明主键列，result配置数据库字段和POJO属性的映射规则：

<resultMap id="role" type="com.liangtengyu.entity.Role">
    <id property="id" column="id" />
    <result property="roleName" column="role_name" />
    <result property="note" column="note" />
</resultMap>
association、collection用于配置级联关系的，分别为一对一和一对多，实际中，多对多关系的应用不多，因为比较复杂，会用一对多的关系把它分解为双向关系。

discriminator用于这样一种场景：比如我们去体检，男和女的体检项目不同，如果让男生去检查妇科项目，是不合理的， 通过discriminator可以根据性别，返回不同的对象。

级联关系的配置比较多，就不在此演示了，可查看文档进行了解。


### cache元素

在没有显示配置缓存时，只开启一级缓存，一级缓存是相对于同一个SqlSession而言的，在参数和SQL完全一样的情况下，使用同一个SqlSession对象调用同一个Mapper的方法，只会执行一次SQL。如果是不同的SqlSession对象，因为不同SqlSession是相互隔离的，即使用相同的Mapper、参数和方法，还是会再次发送SQL到数据库去执行。

二级缓存是SqlSessionFactory层面上的，需要进行显示配置 

 

这样很多设置是默认的，有如下属性可以配置：

- eviction：代表缓存回收策略，可选值有LRU最少使用、FIFO先进先出、SOFT软引用，WEAK弱引用；

- flushInterval：刷新间隔时间，单位为毫秒，如果不配置，当SQL被执行时才会刷新缓存；

- size：引用数目，代表缓存最多可以存储多少对象，不宜设置过大，设置过大会导致内存溢出；

- readOnly：只读，意味着缓存数据只能读取不能修改；

>在大型服务器上，可能会使用专用的缓存服务器，比如Redis缓存，可以通过实现org.apache.ibatis.cache.Cache接口很方便的实现：
```java
public interface Cache {    
    String getId(); //缓存编号
    void putObject(Object var1, Object var2); //保存对象
    Object getObject(Object var1); //获取对象
    Object removeObject(Object var1); //移除对象
    void clear(); //清空缓存
    int getSize(); //获取缓存对象大小
    ReadWriteLock getReadWriteLock(); //获取缓存的读写锁
    }
```

---
### 映射器的内部组成

一般而言，一个映射器是由`3个部分`组成：

> 打开Mybatis源码,在mapping包中可以找到他们

![](https://imgkr2.cn-bj.ufileos.com/3c3a17cb-14a7-4808-af44-35dd492eaef2.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=nMV3KmcXYL5WqeidXA5qo29aLgU%253D&Expires=1607670088)


- `MappedStatement`，它保存映射器的一个节点（select|insert|delete|update）并且包括许多我们配置的sql，sql的id、缓存信息，resultMap，parameterType、resultType、languageDriver等重要的配置内容。



- `SqlSource`，它是提供BoundSql对象的地方，它是MappedStatement的一个属性。它的主要作用是根据参数和其他规则组装sql。
- `BoundSql`，它是建立SQL和参数的地方，他有3个常用的属性：`SQL`、 `parameterObject`、 `parameterMappings` 这3个等会介绍.

idea生成的依赖图


![](https://imgkr2.cn-bj.ufileos.com/99582c53-431e-4e97-a826-295eb41294ba.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=siCdnh%252FyNpSWvGwp1jbpB85d%252FGw%253D&Expires=1607668127)


#### MappedStatement创建过程

首先我们介绍一下MappedStatement

在SqlSessionFactoryBuilder.build()方法,它会帮我们创建一个SqlSessionFactory,它的build方法:
```java
 public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());//此处使用解析XML返回的数据构建 SqlSessionFactory 跟进去..
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
```

进入到parser.parse()方法看看:

```java
 public Configuration parse() {
    if (parsed) { //如果一个xml文件解析两次 就会报错 否则才去解析
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));//解析节点configuration,跟进去...
    return configuration;
  }
```

跟进代码parseConfiguration(parser.evalNode("/configuration"));
```java
 private void parseConfiguration(XNode root) {
    try {
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
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));//解析了很多,但是今天我们的主角是这个..跟进去.
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");//获取三种方式指定的路径 进行判断 告诉MyBatis 到哪里去找映射文件
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();//调用 mapperParser.parse();解析xml  跟进去...
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();//调用 mapperParser.parse();解析xml
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
可以看到在解析mapper标签
```java
  public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));//解析xml的mapper标签,并返回调用configurationElement()方法
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }

    parsePendingResultMaps();//缺少资源的再尝试一下.具体逻辑我们不深入走这里了,有兴趣的可以看看源码.
    parsePendingCacheRefs();
    parsePendingStatements();
  }
```

  configurationElement()方法  


```java
  private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.isEmpty()) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```
buildStatementFromContext()方法:
```java

  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        statementParser.parseStatementNode();//会调用此处,继续跟进
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
```
statementParser.parseStatementNode();
```java
  public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);

    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    //SqlSource是在这里创建的 记住这里等会我们再来分析她.先继续走完流程.
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    if (resultSetTypeEnum == null) {
      resultSetTypeEnum = configuration.getDefaultResultSetType();
    }
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");
    //对各种属性进行了解析并且调用addMappedStatement方法.
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType, 
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```
addMappedStatement()方法:
```java
public MappedStatement addMappedStatement(
      String id,
      SqlSource sqlSource,
      StatementType statementType,
      SqlCommandType sqlCommandType,
      Integer fetchSize,
      Integer timeout,
      String parameterMap,
      Class<?> parameterType,
      String resultMap,
      Class<?> resultType,
      ResultSetType resultSetType,
      boolean flushCache,
      boolean useCache,
      boolean resultOrdered,
      KeyGenerator keyGenerator,
      String keyProperty,
      String keyColumn,
      String databaseId,
      LanguageDriver lang,
      String resultSets) {

    if (unresolvedCacheRef) {
      throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        .cache(currentCache);

    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
      statementBuilder.parameterMap(statementParameterMap);
    }

    MappedStatement statement = statementBuilder.build();
    configuration.addMappedStatement(statement);//configuration内部放入了一个Map<String, MappedStatement>
    return statement;//最终返回MappedStatement
  }
```
至此,MappedStatement创建完毕

---

#### SqlSource的创建过程
回到parseStatementNode()方法找到:
```java
SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
```
createSqlSource方法是LanguageDriver接口的一个方法 她有4个实现

![](https://imgkr2.cn-bj.ufileos.com/191304dd-cc96-47e3-b98d-89eba33165d4.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=b46RjrOxH6%252FjFZ8IPtBPggPvlc4%253D&Expires=1607673167)
我们看一下xml实现
```java
public class XMLLanguageDriver implements LanguageDriver {

  @Override
  public ParameterHandler createParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    return new DefaultParameterHandler(mappedStatement, parameterObject, boundSql);
  }

  @Override
  public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
    XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
    return builder.parseScriptNode();//进行解析,我们跟进..
  }
  ....
}
```
```java

  public SqlSource parseScriptNode() {
    MixedSqlNode rootSqlNode = parseDynamicTags(context);//对动态标签的处理
    SqlSource sqlSource;//初始化Null
    if (isDynamic) {//判断是否动态,来调用不同的SqlSource实例.
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {//跟进RawSqlSource()
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
  }
```
```java
  public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> clazz = parameterType == null ? Object.class : parameterType;
    sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<>());
  }
```
sqlSourceParser.parse()方法要对sql进行处理,这里我们打个断点,可以看到原始参数sql为

`insert into t_test set context = #{context}`

![](https://imgkr2.cn-bj.ufileos.com/ae09689e-1294-4c2a-acb9-a9fa7ab03772.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=yzGxDwF%252BxwTBTBxQ4hXJYUcTUr8%253D&Expires=1607673950)

parse方法处理完毕返回:`insert into t_test set context = ?`并且调用了StaticSqlSource的构造方法
![](https://imgkr2.cn-bj.ufileos.com/28da4c83-fff4-4ef8-b5ea-8abf4b52daee.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=jHyUAYYvsw%252FygkgeoPyB2r%252FzhOM%253D&Expires=1607674036)


StaticSqlSource类继实现了SqlSource接口,
```java
public class StaticSqlSource implements SqlSource {

  private final String sql;
  private final List<ParameterMapping> parameterMappings;
  private final Configuration configuration;

  public StaticSqlSource(Configuration configuration, String sql) {
    this(configuration, sql, null);
  }

  public StaticSqlSource(Configuration configuration, String sql, List<ParameterMapping> parameterMappings) {
    this.sql = sql;
    this.parameterMappings = parameterMappings;
    this.configuration = configuration;
  }

  @Override
  public BoundSql getBoundSql(Object parameterObject) {//这个方法是不是眼熟,BoundSql的构建
    return new BoundSql(configuration, sql, parameterMappings, parameterObject);
  }

}

```
至此,SqlSource创建完毕.

---

#### BoundSql
```java
public class BoundSql {

  private final String sql;
  private final List<ParameterMapping> parameterMappings;
  private final Object parameterObject;
  private final Map<String, Object> additionalParameters;
  private final MetaObject metaParameters;

........
}
```

创建SqlSource时,就把参数放入BoundSql类中来构建一个new BoundSql
```java
public BoundSql getBoundSql(Object parameterObject) {
    return new BoundSql(configuration, sql, parameterMappings, parameterObject);
  }
```

configuration, sql, parameterMappings, parameterObject这些参数是非常重要的我们逐个分析下

##### configuration
这个不用多说了吧 如果不认识她,那直接打回新手村

##### sql
这个属性保存的是我们在映射文件xml中写的sql语句

例子就是刚刚解析出来的Sql语句:insert into t_test set context = ?

##### parameterMappings
`List<ParameterMapping> parameterMappings;`

它是一个list，每一个元素都是parameterMapping对象，这个对象会描述包括属性、名称、表达式、javaType、jdbcType、typeHandler等重要信息。

###### ParameterMapping
```java
public class ParameterMapping {

  private Configuration configuration;

  private String property;
  private ParameterMode mode;
  private Class<?> javaType = Object.class;
  private JdbcType jdbcType;
  private Integer numericScale;
  private TypeHandler<?> typeHandler;
  private String resultMapId;
  private String jdbcTypeName;
  private String expression;
  ......
  }

```


##### parameterObject
parameterObject是参数，可以传递简单对象，pojo、map或者@param注解的参数

`传入简单对象` （包括int、 float、 double等），⽐如当我们传递int类型时，MyBatis
会把参数包装成为Integer对象传递，类似的long、 float、 double也是如此.

`传入pojo、map`那么这个parameterObject参数就是你传入的POJO或者Map不变

`传入多个参数,不带＠Param注解`那么MyBatis就会把
parameterObject变为一个Map<String, Object＞对象，其键值的关系是按顺序来规划
的，类似于这样的形式｛"1":Obj1,"2":Obj2,"3":Obj3…}所以在编写的时候我们都可以使用＃{param 1｝或者＃{1}去引⽤第⼀个参数

`传入多个参数,带@Param注解`如果我们使⽤＠Param注解，那么MyBatis就会把parameterObject变为一个Map<String, Object＞对象类似于没有＠Param注解，只是把其数字的键值对应置换为了＠Param注解的键值。

⽐如我们注解
(＠Param{"name"} String var1,＠Param{"age"} int var2)数字的键值对应置换为了name和age