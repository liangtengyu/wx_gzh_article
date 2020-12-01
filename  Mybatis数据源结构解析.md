
对于 ORM 框架而言，数据源的组织是一个非常重要的一部分，这直接影响到框架的性能问题。本文将通过对 MyBatis 框架的数据源结构进行详尽的分析，找出什么时候创建 Connection ,并且深入解析 MyBatis 的连接池。

--------
### 本章的组织结构：
- 零、什么是连接池和线程池
- 一、MyBatis 数据源 DataSource 分类
- 二、数据源 DataSource 的创建过程
- 三、 DataSource 什么时候创建 Connection 对象
- 四、不使用连接池的 UnpooledDataSource
- 五、使用了连接池的 PooledDataSource
###  连接池和线程池
#### 连接池：（降低物理连接损耗）
- 1、连接池是面向数据库连接的
- 2、连接池是为了优化数据库连接资源
- 3、连接池有点类似在客户端做优化

数据库连接是一项有限的`昂贵资源`，一个数据库连接对象均对应一个物理数据库连接，每次操作都打开一个物理连接，使用完都关闭连接，这样造成系统的性能低下。 数据库连接池的解决方案是在应用程序启动时建立足够的数据库连接，并将这些连接组成一个连接池，由应用程序动态地对池中的连接进行申请、使用和释放。对于多于连接池中连接数的并发请求，应该在请求队列中排队等待。并且应用程序可以根据池中连接的使用率，动态增加或减少池中的连接数。

-----

#### 线程池：（降低线程创建销毁损耗）
- 1、线程池是面向后台程序的
- 2、线程池是是为了提高内存和CPU效率
- 3、线程池有点类似于在服务端做优化

线程池是一次性创建一定数量的线程（应该可以配置初始线程数量的），当用请求过来不用去创建新的线程，直接使用已创建的线程，使用后又放回到线程池中。
避免了频繁创建线程，及销毁线程的系统开销，提高是内存和CPU效率。

#### 相同点：
都是事先准备好资源，避免频繁创建和销毁的代价。


###  数据源的分类
在Mybatis体系中,分为*3*种`DataSource`
>打开Mybatis源码找到datasource包,可以看到3个子package

![](https://imgkr2.cn-bj.ufileos.com/d5668e9a-5062-4afe-8164-00e4aa8cf40e.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=Ntrx%252Bc63Fe1e4AAPqocOllg8PDo%253D&Expires=1606806446)
- UNPOOLED    `不使用连接池`的数据源

- POOLED      使用`连接池`的数据源

- JNDI        使用`JNDI`实现的数据源

![](https://imgkr2.cn-bj.ufileos.com/7dacb524-dd04-4836-b196-9d27b88efa56.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=mcRWrY21qyBKCowtw1fXlFvHu%252FQ%253D&Expires=1606806572)

>MyBatis内部分别定义了实现了java.sql.DataSource接口的UnpooledDataSource，PooledDataSource类来表示UNPOOLED、POOLED类型的数据源。 如下图所示：

![](https://imgkr2.cn-bj.ufileos.com/8334f7b8-8c11-42ec-bbd1-39e78caacfa7.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=%252F%252B2fjuE2kH%252BsZFYehlPPaBMRNIE%253D&Expires=1606807011)
- PooledDataSource和UnpooledDataSrouce都实现了java.sql.DataSource接口.
- PooledDataSource持有一个UnPooledDataSource的引用,当PooledDataSource要创建Connection实例时,实际还是通过UnPooledDataSource来创建的.(PooledDataSource)只是提供一种`缓存连接池机制`.

> JNDI类型的数据源DataSource，则是通过JNDI上下文中取值。

###  数据源 DataSource 的创建过程
>在mybatis的XML配置文件中，使用<dataSource>元素来配置数据源：
>```xml
><!-- 配置数据源（连接池） -->
><dataSource type="POOLED"> //这里 type 属性的取值就是为POOLED、UNPOOLED、JNDI
><property name="driver" value="${jdbc.driver}"/>
><property name="url" value="${jdbc.url}"/>
><property name="username" value="${jdbc.username}"/>
><property name="password" value="${jdbc.password}"/>
></dataSource>
>```

   MyBatis在初始化时，解析此文件，根据`<dataSource>`的type属性来创建`相应类型的的数据源`DataSource，即：


- type=”POOLED”     ：`创建PooledDataSource实例`
- type=”UNPOOLED”   ：`创建UnpooledDataSource实例`
- type=”JNDI”       ：`从JNDI服务上查找DataSource实例`
  
  ----
  
  Mybatis是通过工厂模式来创建数据源对象的 我们来看看源码:
  
  ```java
  public interface DataSourceFactory {

  void setProperties(Properties props);

  DataSource getDataSource();//生产DataSource
  
  }
  ```
  
  上述3种类型的数据源,对应有自己的工厂模式,都实现了这个`DataSourceFactory` 
  

![](https://imgkr2.cn-bj.ufileos.com/83d23c09-c20d-4044-bee2-826868a9ca80.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=Ex1M6JcaZeho46dm7Ae510zbhUg%253D&Expires=1606809179)
 > MyBatis创建了DataSource实例后，会将其放到Configuration对象内的Environment对象中， 供以后使用。

  >注意dataSource 此时只会保存好配置信息.连接池此时并没有创建好连接.只有当程序在调用操作数据库的方法时,才会初始化连接.

  ### DataSource什么时候创建Connection对象
  我们需要创建SqlSession对象并需要执行SQL语句时，这时候MyBatis才会去调用dataSource对象来创建java.sql.Connection对象。也就是说，java.sql.Connection对象的创建一直延迟到执行SQL语句的时候。


  `例子:`
  ```java
    @Test
    public void testMyBatisBuild() throws IOException {
        Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader);
        SqlSession sqlSession = factory.openSession();
        TestMapper mapper = sqlSession.getMapper(TestMapper.class);
        Ttest one = mapper.getOne(1L);//直到这一行,才会去创建一个数据库连接!
        System.out.println(one);
        sqlSession.close();
    }
  ```
  口说无凭,跟进源码看看他们是在什么时候创建的...

  #### 跟进源码,验证Datasource 和Connection对象创建时机
  ##### 验证Datasource创建时机
  -  上面我们已经知道,pooled数据源实际上也是使用的unpooled的实例,那么我们在UnpooledDataSourceFactory的
getDataSource方法的源码中做一些修改 并运行测试用例:
  ```java
  @Override
  public DataSource getDataSource() {//此方法是UnpooledDataSourceFactory实现DataSourceFactory复写
    System.out.println("创建了数据源");
    System.out.println(dataSource.toString());
    return dataSource;
  }
  ```


![](https://imgkr2.cn-bj.ufileos.com/9741a651-23fc-44c7-8ab4-c7e3d83db91a.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=Yc%252BgbBmiWdxRUVCVJF%252Btkb2fTVI%253D&Expires=1606810522)

  >结论:在创建完SqlsessionFactory时,DataSource实例就创建了.
  >-----

  ##### 验证Connection创建时机

  首先我们先查出现在数据库的所有连接数,在数据库中执行

  ```sql
  SELECT * FROM performance_schema.hosts;
  ```

  >返回数据:  显示当前连接数为1,总连接数70

 

![](https://imgkr2.cn-bj.ufileos.com/aa171928-d65b-471a-b1f1-6b4615bfbaa8.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=BQZVr9qRtmVzx61cL5t5oS1vnjA%253D&Expires=1606873256)

   ```sql
  show full processlist; //显示所有的任务列表
   ```
  >返回:当前只有一个查询的连接在运行

![](https://imgkr2.cn-bj.ufileos.com/cf3dbea7-1579-40bc-b7c0-845381527e83.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=8q2xZ8Dt33Hhuc8p2qXpOmwkEYU%253D&Expires=1606873373)

  

  

  重新启动项目,在运行到需要执行实际的sql操作时,可以看到他已经被代理增强了

![](https://imgkr2.cn-bj.ufileos.com/f5edd70e-1462-48de-b4ba-e2cb125bf1a6.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=bS%252Fbmbk4OKtR%252FiRy%252BVD2lAbz7eo%253D&Expires=1606810977)

  >直到此时,连接数还是没有变,说明连接还没有创建,我们接着往下看.


  >我们按F7进入方法,可以看到,他被代理,，这时候会执行到之前的代理方法中调用invoke方法.这里有一个判断,但是并不成立,于是进入cachedInvoker(method).invoke()方法代理执行一下操作

![](https://imgkr2.cn-bj.ufileos.com/a2fb54c5-1789-41e9-a218-d7bd053babd1.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=84urhrF%252FofIA7gbLRTTcped%252BbmQ%253D&Expires=1606811512)

  cachedInvoker(method).invoke()方法 

```java
  
    @Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
      return mapperMethod.execute(sqlSession, args);
    }
```
 >继续F7进入方法,由于我们是单条查询select 所以会case进入select块中的selectOne

![](https://imgkr2.cn-bj.ufileos.com/f8610c54-d157-4a2e-bdc5-89aee3dab88f.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=v1kmFIFxKMQ5%252FI98xgrvQiL0IMA%253D&Expires=1606811880)

  继续F7

![](https://imgkr2.cn-bj.ufileos.com/c7ca4677-7b82-4070-9420-74e5f9e96ee8.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=cJWY1cKBd794zoUep22ai%252B2OBxE%253D&Expires=1606811989)

继续F7
  >通过configuration.getMappedStatement获取MappedStatement

![](https://imgkr2.cn-bj.ufileos.com/d0734182-df7f-4ec0-843f-9369efb47e06.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=A%252FvDWc%252BbqIdiWTIgONthjCeKoRU%253D&Expires=1606812043)

>单步步过,F8后,进入executor.query方法

  ```java
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
  ```

>继续走到底,F7进入query方法


![](https://imgkr2.cn-bj.ufileos.com/3aa5ea6f-ef0b-499b-b8d3-17a02e9535b6.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=6fM2aFRk4VMZof1fOkZRlR3owCY%253D&Expires=1606812313)


>此时,会去缓存中查询,这里的缓存是二级缓存对象 ,生命周期是mapper级别的(一级缓存是一个session级别的),因为我们此时是第一次运行程序,所以肯定为Null,这时候会直接去查询,调用`delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql)`方法,F7进入这个方法

![](https://imgkr2.cn-bj.ufileos.com/ed5346a9-4b30-42dd-9322-fc56b1344e6b.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=AzlxWhrDruXMOS0aB5IZqv1gMwk%253D&Expires=1606812850)

>二级缓存没有获取到,又去查询了一级缓存,发现一级缓存也没有,这个时候,才去查数据库

  `queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql)`;//没有缓存则去db查

  >F7进入queryFromDatabase方法.看到是一些对一级缓存的操作,我们主要看`doQuery`方法F7进入它.

![](https://imgkr2.cn-bj.ufileos.com/6725e872-c1ad-46cd-b38f-95e6c5e1cf51.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=bWByzYIvjScd8ZqKLiEBkFW3VHc%253D&Expires=1606813095)

  >可以看到它准备了一个空的Statement

![](https://imgkr2.cn-bj.ufileos.com/b607d94a-5712-4115-baa0-cdf35b4a11c0.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=GNtJAS4%252FK1mHiLLkJAgEaoHwj1c%253D&Expires=1606813201)

>我们F7跟进看一下prepareStatement方法 ,发现他调用了getConnection,哎!有点眼熟了,继续F7进入getConnection()方法,

![](https://imgkr2.cn-bj.ufileos.com/9ca34002-ed78-4ea6-9108-8ad39d6fbf87.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=WWL%252FwVoLYWLHOTcZIIYp5jWl1ok%253D&Expires=1606813317)

>又是一个getConnection()....继续F7进入`transaction.getConnection()`方法

![](https://imgkr2.cn-bj.ufileos.com/b28b6c7b-c824-4e64-9110-4cca4adcdf76.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=SzgN%252BX0ibQn7GMwgZtjNt0ECzqE%253D&Expires=1606813515)

>又是一个getConnection()方法.判断connection是否为空.为空openConnection()否则直接返回connection;我们F7继续跟进openConnection()方法

![](https://imgkr2.cn-bj.ufileos.com/c4a180ad-88f2-49d2-a70d-275fef7a1534.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=AXbFCpRbjYYZPnRBNb6DExVlDBc%253D&Expires=1606813650)

  ```java
    protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
      log.debug("Opening JDBC Connection");
    }
    connection = dataSource.getConnection();//最终获取连接的地方在这句.
    if (level != null) {
      connection.setTransactionIsolation(level.getLevel());//设置隔离等级
    }
    setDesiredAutoCommit(autoCommit);//是否自动提交,默认false,update不会提交到数据库,需要手动commit
  }
  ```

  dataSource.getConnection()执行完,至此一个connection才创建完成.
  我们验证一下 在dataSource.getConnection()时打一下断点.

![](https://imgkr2.cn-bj.ufileos.com/3b8ad797-16b4-4d15-b048-fa4d53927194.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=piQ5UHeyCrYCEHUA%252Bj2%252FG8%252BErxM%253D&Expires=1606873768)

  此时数据库中的连接数依然没变 还是1


![](https://imgkr2.cn-bj.ufileos.com/ba3f0cf5-1705-4e5a-a1d7-9d9cb33e901d.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=%252FNLCwjmopDq7n4h4zWqoTIHwzRY%253D&Expires=1606873801)

  我们按F8 执行一步

  在控制台可以看到connection = com.mysql.jdbc.JDBC4Connection@1500b2f3 实例创建完毕 我们再去数据库中看看连接数

![](https://imgkr2.cn-bj.ufileos.com/c353d1d2-b678-47e3-aae3-e77e2866c018.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=8yEFE16%252BSNIqFUxjahccgbkpf6U%253D&Expires=1606873878)

两个连接分别是:

![](https://imgkr2.cn-bj.ufileos.com/37b9987e-ed9e-49ff-8ce7-5adbb08eafea.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=6p4n%252B%252FaKDCHQ7L%252F6TDVRWEiEfKg%253D&Expires=1606873891)


###  不使用连接池的 UnpooledDataSource

  >当 <dataSource>的type属性被配置成了”UNPOOLED”，MyBatis首先会实例化一个UnpooledDataSourceFactory工厂实例，然后通过.getDataSource()方法返回一个UnpooledDataSource实例对象引用，我们假定为dataSource。
  >使用UnpooledDataSource的getConnection(),每调用一次就会产生一个新的Connection实例对象。

UnPooledDataSource的getConnection()方法实现如下：
  ```java
  /*
UnpooledDataSource的getConnection()实现
*/
public Connection getConnection() throws SQLException
{
    return doGetConnection(username, password);
}

private Connection doGetConnection(String username, String password) throws SQLException
{
    //封装username和password成properties
    Properties props = new Properties();
    if (driverProperties != null)
    {
        props.putAll(driverProperties);
    }
    if (username != null)
    {
        props.setProperty("user", username);
    }
    if (password != null)
    {
        props.setProperty("password", password);
    }
    return doGetConnection(props);
}

/*
 *  获取数据连接
 */
private Connection doGetConnection(Properties properties) throws SQLException
{
    //1.初始化驱动
    initializeDriver();
    //2.从DriverManager中获取连接，获取新的Connection对象
    Connection connection = DriverManager.getConnection(url, properties);
    //3.配置connection属性
    configureConnection(connection);
    return connection;
}
  ```
>UnpooledDataSource会做以下几件事情：

- 1.  初始化驱动：    判断driver驱动是否已经加载到内存中，如果还没有加载，则会动态地加载driver类，并实例化一个Driver对象，使用DriverManager.registerDriver()方法将其注册到内存中，以供后续使用。

- 2.  创建Connection对象：    使用DriverManager.getConnection()方法创建连接。

- 3.  配置Connection对象：    设置是否自动提交autoCommit和隔离级别isolationLevel。

- 4.  返回Connection对象。
  
  

![](https://imgkr2.cn-bj.ufileos.com/4154e35d-7f0b-4779-bd19-46adc93c1826.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=PdqltEZtW7YtxS2QQr6IsUw4lqc%253D&Expires=1606874412)
>我们每调用一次getConnection()方法，都会通过DriverManager.getConnection()返回新的java.sql.Connection实例,这样当然对于资源是一种浪费,为了防止重复的去创建和销毁连接,于是引入了连接池的概念.

  ### 使用了连接池的 PooledDataSource
  要理解连接池,首先要了解它对于connection的容器,它使用PoolState容器来管理所有的conncetion

![](https://imgkr2.cn-bj.ufileos.com/0d82075d-b74e-4584-af50-ad0a1685f0bd.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=40%252FXSCTGqHIJLyeLy1COEao4j74%253D&Expires=1606874724)
  在PoolState中,它将connection分为两种状态,`空闲状态（idle）`和`活动状态(active)`,他们分别被存储到PoolState容器内的`idleConnections`和`activeConnections`两个ArrayList中

![](https://imgkr2.cn-bj.ufileos.com/92c91046-1b63-4bd4-80af-d0d24eda7175.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=CRtWJGEbA8EMPgvGUFLFrJWBTTw%253D&Expires=1606874860)

 -  `idleConnections`:空闲(idle)状态PooledConnection对象被放置到此集合中，表示当前闲置的没有被使用的PooledConnection集合，调用PooledDataSource的getConnection()方法时，会优先从此集合中取PooledConnection对象。当用完一个java.sql.Connection对象时，MyBatis会将其包裹成PooledConnection对象放到此集合中。

- `activeConnections`:活动(active)状态的PooledConnection对象被放置到名为activeConnections的ArrayList中，表示当前正在被使用的PooledConnection集合，调用PooledDataSource的getConnection()方法时，会优先从idleConnections集合中取PooledConnection对象,如果没有，则看此集合是否已满，如果未满，PooledDataSource会创建出一个PooledConnection，添加到此集合中，并返回。

#### 从连接池中获取一个连接对象的过程
下面让我们看一下PooledDataSource 的popConnection方法获取Connection对象的实现：
```java
private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    while (conn == null) {
      synchronized (state) {//给state对象加锁
        if (!state.idleConnections.isEmpty()) {//如果空闲列表不空,就从空闲列表中拿connection
          // Pool has available connection
          conn = state.idleConnections.remove(0);//拿出空闲列表中的第一个,去验证连接是否还有效
          if (log.isDebugEnabled()) {
            log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
          }
        } else {
          // 空闲连接池中没有可用的连接,就来看看活跃连接列表中是否有..先判断活动连接总数 是否小于 最大可用的活动连接数
          if (state.activeConnections.size() < poolMaximumActiveConnections) {
            // 如果连接数小于list.size 直接创建新的连接.
            conn = new PooledConnection(dataSource.getConnection(), this);
            if (log.isDebugEnabled()) {
              log.debug("Created connection " + conn.getRealHashCode() + ".");
            }
          } else {
            // 此时连接数也满了,不能创建新的连接. 找到最老的那个,检查它是否过期
            //计算它的校验时间，如果校验时间大于连接池规定的最大校验时间，则认为它已经过期了
            // 利用这个PoolConnection内部的realConnection重新生成一个PooledConnection
            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
            if (longestCheckoutTime > poolMaximumCheckoutTime) {
              // 可以要求过期这个连接.
              state.claimedOverdueConnectionCount++;
              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
              state.accumulatedCheckoutTime += longestCheckoutTime;
              state.activeConnections.remove(oldestActiveConnection);
              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                try {
                  oldestActiveConnection.getRealConnection().rollback();
                } catch (SQLException e) {
                  /*
                     Just log a message for debug and continue to execute the following
                     statement like nothing happened.
                     Wrap the bad connection with a new PooledConnection, this will help
                     to not interrupt current executing thread and give current thread a
                     chance to join the next competition for another valid/good database
                     connection. At the end of this loop, bad {@link @conn} will be set as null.
                   */
                  log.debug("Bad connection. Could not roll back");
                }
              }
              conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
              conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
              conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
              oldestActiveConnection.invalidate();
              if (log.isDebugEnabled()) {
                log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
              }
            } else {
              //如果不能释放，则必须等待
              // Must wait
              try {
                if (!countedWait) {
                  state.hadToWaitCount++;
                  countedWait = true;
                }
                if (log.isDebugEnabled()) {
                  log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
                }
                long wt = System.currentTimeMillis();
                state.wait(poolTimeToWait);
                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
              } catch (InterruptedException e) {
                break;
              }
            }
          }
        }
        if (conn != null) {
          // ping to server and check the connection is valid or not
          if (conn.isValid()) {//去验证连接是否还有效.
            if (!conn.getRealConnection().getAutoCommit()) {
              conn.getRealConnection().rollback();
            }
            conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
            conn.setCheckoutTimestamp(System.currentTimeMillis());
            conn.setLastUsedTimestamp(System.currentTimeMillis());
            state.activeConnections.add(conn);
            state.requestCount++;
            state.accumulatedRequestTime += System.currentTimeMillis() - t;
          } else {
            if (log.isDebugEnabled()) {
              log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
            }
            state.badConnectionCount++;
            localBadConnectionCount++;
            conn = null;
            if (localBadConnectionCount > (poolMaximumIdleConnections + poolMaximumLocalBadConnectionTolerance)) {
              if (log.isDebugEnabled()) {
                log.debug("PooledDataSource: Could not get a good connection to the database.");
              }
              throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
            }
          }
        }
      }

    }

    if (conn == null) {
      if (log.isDebugEnabled()) {
        log.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
      }
      throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
    }

    return conn;
  }
```


![](https://imgkr2.cn-bj.ufileos.com/02841017-c06d-4917-9e01-c1c218a82e1d.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=tgs8ZN%252BzqvICbPQcz8%252B1wSRRGLc%253D&Expires=1606876223)


  >如上所示,对于PooledDataSource的getConnection()方法内，先是调用类PooledDataSource的popConnection()方法返回了一个PooledConnection对象，然后调用了PooledConnection的getProxyConnection()来返回Connection对象。

#### 复用连接的过程
  >如果我们使用了连接池，我们在用完了Connection对象时，需要将它放在连接池中，该怎样做呢？ 如果让我们来想的话,应该是通过代理Connection对象,在调用close时,并不真正关闭,而是丢到管理连接的容器中去. 要验证这个想法 那么 来看看Mybatis帮我们怎么实现复用连接的.
```java
class PooledConnection implements InvocationHandler {

  //......
  //所创建它的datasource引用
  private PooledDataSource dataSource;
  //真正的Connection对象
  private Connection realConnection;
  //代理自己的代理Connection
  private Connection proxyConnection;

  //......
}
```
  >PooledConenction实现了InvocationHandler接口，并且，proxyConnection对象也是根据这个它来生成的代理对象：
```java
public PooledConnection(Connection connection, PooledDataSource dataSource) {
    this.hashCode = connection.hashCode();
    this.realConnection = connection;//真实连接
    this.dataSource = dataSource;
    this.createdTimestamp = System.currentTimeMillis();
    this.lastUsedTimestamp = System.currentTimeMillis();
    this.valid = true;
    this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
  }
```
  >实际上，我们调用PooledDataSource的getConnection()方法返回的就是这个proxyConnection对象。
  >当我们调用此proxyConnection对象上的任何方法时，都会调用PooledConnection对象内invoke()方法。
  >让我们看一下PooledConnection类中的invoke()方法定义：
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    //当调用关闭的时候，回收此Connection到PooledDataSource中
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
      dataSource.pushConnection(this);
      return null;
    } else {
      try {
        if (!Object.class.equals(method.getDeclaringClass())) {
          checkConnection();
        }
        return method.invoke(realConnection, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }
```
>结论:当我们使用了pooledDataSource.getConnection()返回的Connection对象的close()方法时，不会调用真正Connection的close()方法，而是将此Connection对象放到连接池中。调用`dataSource.pushConnection(this)`实现
```java
protected void pushConnection(PooledConnection conn) throws SQLException {

    synchronized (state) {
      state.activeConnections.remove(conn);
      if (conn.isValid()) {
        if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
          state.idleConnections.add(newConn);
          newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
          newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
          conn.invalidate();
          if (log.isDebugEnabled()) {
            log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
          }
          state.notifyAll();
        } else {
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          conn.getRealConnection().close();
          if (log.isDebugEnabled()) {
            log.debug("Closed connection " + conn.getRealHashCode() + ".");
          }
          conn.invalidate();
        }
      } else {
        if (log.isDebugEnabled()) {
          log.debug("A bad connection (" + conn.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
        }
        state.badConnectionCount++;
      }
    }
  }
```
