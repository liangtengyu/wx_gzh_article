### 准备

在阅读源码前,需要先clone源码 地址:https://github.com/mybatis/mybatis-3

>Mybatis框架使用大量常见的设计模式,学习Mybatis源码我们主要学习以下几点:

- 学习大佬们的编码思想及规范
- 学习一些传承下来的设计模式
- 实践java基础理论

>带着问题阅读源码 

- 问题1:源码中用了哪些设计模式？为什么要用这些设计模式？
- 问题2.Mybatis打开调试模式之后，能打印sql语句等信息，这是怎么实现的？实现过程中使用了什么设计模式？假如有多个日志实现,加载顺序是什么?


带着这些个问题 我们去源码中找寻答案
首先我们先来看看源码包功能模块图 
![image.png](http://liangtengyu.com/opt/deploy/upload/9eef3423-1606354697191.png)
Mybatis框架提供了一个顶层接口`SqlSession` ,核心处理层,基础支撑层 这里使用了一个常见的设计模式:`外观模式/门面模式`
今天我们主要讲解`基础支撑层`中`日志模块`内部是如何处理的.

### 源码-logging

打开源码项目,可以看到有20个目录,这里我们先不深入去看,把目光聚焦到logging目录
![image.png](http://liangtengyu.com/opt/deploy/upload/349d6bed-1606355259874.png)
打开目录,我们可以看到映入眼帘的就是 *Log*   *LogException*    *LogFactory*  (红色部分)和具体的日志实现(黄色部分),
![image.png](http://liangtengyu.com/opt/deploy/upload/b1303293-1606355485078.png)
打开Log接口,我们可以看到Log接口提供了七个接口方法
![image.png](http://liangtengyu.com/opt/deploy/upload/7a064b4f-1606355877768.png)
实现这个接口的实现方法我们看看都是谁呢,可以看到就是我们刚刚黄色框框部分,那么他是怎么来处理那么多Log实现的呢,别急,继续往下看
![image.png](http://liangtengyu.com/opt/deploy/upload/d4635646-1606355934556.png)
我们随便打开一个Log的实现类,可一看到实际上这并不是真正的实现了所需要的Log只是在对每个用到的Log实现做了参数适配而已
(其他的类也是类似,这里使用了适配器模式)
![image.png](http://liangtengyu.com/opt/deploy/upload/b12347ff-1606356255602.png)
那么有那么多日志实现,他的加载顺序是什么呢.到这里我们会想到刚刚看到的LogFactory类,大叫都知道工厂方法就是用来获取实例的既然要获取实例那么他的处理也肯定在工厂内部处理完毕了(这里使用了简单工厂模式) 那么我们打开LogFactory类来看看他的加载顺序到底是怎么实现的:

```java 
import java.lang.reflect.Constructor;


public final class LogFactory {


  public static final String MARKER = "MYBATIS";

  private static Constructor<? extends Log> logConstructor; //定义logConstructor 他继承于Constructor

  static {	//加载顺序在这里.初始化时必定按照这个顺序执行,可以发现下面这一排排就是我们刚刚看到的各个适配器
    tryImplementation(LogFactory::useSlf4jLogging);
    tryImplementation(LogFactory::useCommonsLogging);
    tryImplementation(LogFactory::useLog4J2Logging);
    tryImplementation(LogFactory::useLog4JLogging);
    tryImplementation(LogFactory::useJdkLogging);
    tryImplementation(LogFactory::useNoLogging);
  }

  private LogFactory() {
    // 在这里私有化构造器 就不能通过new来创建实例了
  }

  public static Log getLog(Class<?> clazz) {//第一步.如果我们需要一个Log 那么我们就调用这个方法,传入需要的类即可
    return getLog(clazz.getName());
  }

  public static Log getLog(String logger) {//第二步.根据类的名字获取Log实现.如果获取不到会报错,这个logConstructor就是我们上面的,他内部在初始化时已经通过static的静态代码块进行了处理,所以会处理为对应的日志实现,
    try {
      return logConstructor.newInstance(logger);
    } catch (Throwable t) {
      throw new LogException("Error creating logger for logger " + logger + ".  Cause: " + t, t);
    }
  }

  public static synchronized void useCustomLogging(Class<? extends Log> clazz) {
    setImplementation(clazz);
  }

  public static synchronized void useSlf4jLogging() {//每个静态同步方法都是去调用setImplementation
    setImplementation(org.apache.ibatis.logging.slf4j.Slf4jImpl.class);
  }

  public static synchronized void useCommonsLogging() {
    setImplementation(org.apache.ibatis.logging.commons.JakartaCommonsLoggingImpl.class);
  }

  public static synchronized void useLog4JLogging() {
    setImplementation(org.apache.ibatis.logging.log4j.Log4jImpl.class);
  }

  public static synchronized void useLog4J2Logging() {
    setImplementation(org.apache.ibatis.logging.log4j2.Log4j2Impl.class);
  }

  public static synchronized void useJdkLogging() {
    setImplementation(org.apache.ibatis.logging.jdk14.Jdk14LoggingImpl.class);
  }

  public static synchronized void useStdOutLogging() {
    setImplementation(org.apache.ibatis.logging.stdout.StdOutImpl.class);
  }

  public static synchronized void useNoLogging() {
    setImplementation(org.apache.ibatis.logging.nologging.NoLoggingImpl.class);
  }

  private static void tryImplementation(Runnable runnable) {//将传入的代码块执行,注意这里是.run 不是.start 
    if (logConstructor == null) {//如果还没有实现才执行
      try {
        runnable.run();
      } catch (Throwable t) {//如果加载这个实现出现了异常,如:ClassNotFoundException 这个时候直接忽略,等待下一个实现调用
        // ignore
      }
    }
  }
		//在获取到一个实现成功后 会调用这个方法来初始化Log
  private static void setImplementation(Class<? extends Log> implClass) {
    try {
      Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
      Log log = candidate.newInstance(LogFactory.class.getName());
      if (log.isDebugEnabled()) {
        log.debug("Logging initialized using '" + implClass + "' adapter.");
      }
      logConstructor = candidate;
    } catch (Throwable t) {
      throw new LogException("Error setting Log implementation.  Cause: " + t, t);
    }
  }

}

```

回顾一下问题:源码中用了哪些设计模式？
到目前我们看到Mybatis使用了 1.外观模式/门面模式 2.适配器模式 3简单工厂模式
到这里我们已经拿到了Log的实例不管他是通过log4j还是slf4j的实现,那他是怎么实现对日志进行增强的呢 怎么打印出来的SQL语句和参数以及结果计数的?
如果想要输出这些信息,那么首先肯定要获取到这些信息.
下面我们就去看这块,看看他是怎么获取和存贮这些信息的.
![image.png](http://liangtengyu.com/opt/deploy/upload/41204163-1606358700692.png)

```java
public abstract class BaseJdbcLogger {

  protected static final Set<String> SET_METHODS;//set方法容器
  protected static final Set<String> EXECUTE_METHODS = new HashSet<>();//执行方法容器

  private final Map<Object, Object> columnMap = new HashMap<>();//列的键值对

  private final List<Object> columnNames = new ArrayList<>();//所有key的列表
  private final List<Object> columnValues = new ArrayList<>();//所有value的列表

  protected final Log statementLog;  // Log实现
  protected final int queryStack;

  /*
   * Default constructor
   */
  public BaseJdbcLogger(Log log, int queryStack) {//默认的构造
    this.statementLog = log;
    if (queryStack == 0) {
      this.queryStack = 1;
    } else {
      this.queryStack = queryStack;
    }
  }

  static {//初始化 就加载两个容器 这两个容器用来装所有的预编译set参数的方法和所有执行方法
    SET_METHODS = Arrays.stream(PreparedStatement.class.getDeclaredMethods())//这里使用JAVA8新特性将PreparedStatement类的所有set开头的方法映射成一个Set
            .filter(method -> method.getName().startsWith("set"))
            .filter(method -> method.getParameterCount() > 1)
            .map(Method::getName)
            .collect(Collectors.toSet());

    EXECUTE_METHODS.add("execute");//将4个执行方法添加到EXECUTE_METHODS这个Set
    EXECUTE_METHODS.add("executeUpdate");
    EXECUTE_METHODS.add("executeQuery");
    EXECUTE_METHODS.add("addBatch");
  }

  protected void setColumn(Object key, Object value) {
    columnMap.put(key, value);
    columnNames.add(key);
    columnValues.add(value);
  }

  protected Object getColumn(Object key) {
    return columnMap.get(key);
  }

  protected String getParameterValueString() {
    List<Object> typeList = new ArrayList<>(columnValues.size());
    for (Object value : columnValues) {
      if (value == null) {
        typeList.add("null");
      } else {
        typeList.add(objectValueString(value) + "(" + value.getClass().getSimpleName() + ")");
      }
    }
    final String parameters = typeList.toString();
    return parameters.substring(1, parameters.length() - 1);
  }

  protected String objectValueString(Object value) {
    if (value instanceof Array) {
      try {
        return ArrayUtil.toString(((Array) value).getArray());
      } catch (SQLException e) {
        return value.toString();
      }
    }
    return value.toString();
  }

  protected String getColumnString() {
    return columnNames.toString();
  }

  protected void clearColumnInfo() {
    columnMap.clear();
    columnNames.clear();
    columnValues.clear();
  }

  protected String removeExtraWhitespace(String original) {
    return SqlSourceBuilder.removeExtraWhitespaces(original);
  }

  protected boolean isDebugEnabled() {
    return statementLog.isDebugEnabled();
  }

  protected boolean isTraceEnabled() {
    return statementLog.isTraceEnabled();
  }

  protected void debug(String text, boolean input) {
    if (statementLog.isDebugEnabled()) {
      statementLog.debug(prefix(input) + text);
    }
  }

  protected void trace(String text, boolean input) {
    if (statementLog.isTraceEnabled()) {
      statementLog.trace(prefix(input) + text);
    }
  }

  private String prefix(boolean isInput) {
    char[] buffer = new char[queryStack * 2 + 2];
    Arrays.fill(buffer, '=');
    buffer[queryStack * 2 + 1] = ' ';
    if (isInput) {
      buffer[queryStack * 2] = '>';
    } else {
      buffer[0] = '<';
    }
    return new String(buffer);
  }

}
```
到这我们知道了他是怎么存贮信息的获取的方式就是在调用时通过存贮的方法和set方法列表来查询,那么他是什么时候被调用呢?仔细看看这个图我们发现了端倪 在BaseJdbcLogger下面还有
- ConnectionLogger
- PreparedStatementLogger
- ResultSetLogger
- StatementLogger
这些不就是我们很眼熟的流程吗 获取数据库连接-->  预编译SQL-->  执行SQL-->  获取结果 
![image.png](http://liangtengyu.com/opt/deploy/upload/41204163-1606358700692.png)
我们随便打开一个 看看他们是怎么在自己的一亩三分地干活的
```java
public final class ConnectionLogger extends BaseJdbcLogger implements InvocationHandler {//继承于BaseJdbcLogger 实现InvocationHandler 看到InvocationHandler 就会想到->代理模式

  private final Connection connection;//他实要代理的就是这connection

  private ConnectionLogger(Connection conn, Log statementLog, int queryStack) {
    super(statementLog, queryStack);
    this.connection = conn;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] params)
      throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, params);
      }
      if ("prepareStatement".equals(method.getName()) || "prepareCall".equals(method.getName())) {//判断方法名是不是这两中的一个 短语或 有一个就为true
        if (isDebugEnabled()) {//原来开启debug 就会打出日志是在这里呀
          debug(" Preparing: " + removeExtraWhitespace((String) params[0]), true);
        }
        PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);// 通过connection创建了PreparedStatement
        stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack); //把stmt 传过去给PreparedStatementLogger的newInstance 创建了一个stmt的代理返回
        return stmt;//注意这里返回的不是真实stmt 而是代理过后的stmt
      } else if ("createStatement".equals(method.getName())) {//如果是创建Statement
        Statement stmt = (Statement) method.invoke(connection, params); //拿到连接去创建一个Statement
        stmt = StatementLogger.newInstance(stmt, statementLog, queryStack);//把Statement传进StatementLogger 获取Statement的代理对象来返回
        return stmt;//注意这里返回的不是真实stmt 而是代理过后的stmt
      } else {
        return method.invoke(connection, params);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }

  /**
   * Creates a logging version of a connection.
   *
   * @param conn
   *          the original connection
   * @param statementLog
   *          the statement log
   * @param queryStack
   *          the query stack
   * @return the connection with logging
   */
  public static Connection newInstance(Connection conn, Log statementLog, int queryStack) {
    InvocationHandler handler = new ConnectionLogger(conn, statementLog, queryStack);
    ClassLoader cl = Connection.class.getClassLoader();
    return (Connection) Proxy.newProxyInstance(cl, new Class[]{Connection.class}, handler);
  }

  /**
   * return the wrapped connection.
   *
   * @return the connection
   */
  public Connection getConnection() {
    return connection;
  }
```


```java
public final class PreparedStatementLogger extends BaseJdbcLogger implements InvocationHandler {

  private final PreparedStatement statement; //还是一样 这里是被代理的实际对象

  private PreparedStatementLogger(PreparedStatement stmt, Log statementLog, int queryStack) {
    super(statementLog, queryStack);
    this.statement = stmt;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] params) throws Throwable {//增强的逻辑在这里
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, params);
      }
      if (EXECUTE_METHODS.contains(method.getName())) {//还记得那几个容器吗 用来存储各种方法的  现在他来了
        //开启DEBUG 我就把参数打印出来
        if (isDebugEnabled()) {
          debug("Parameters: " + getParameterValueString(), true);
        }
        clearColumnInfo();
        if ("executeQuery".equals(method.getName())) {
          ResultSet rs = (ResultSet) method.invoke(statement, params);//获取到一个resultset
          return rs == null ? null : ResultSetLogger.newInstance(rs, statementLog, queryStack);//把resultset 也代理了来返回
        } else {
          return method.invoke(statement, params);
        }
      } else if (SET_METHODS.contains(method.getName())) {
        if ("setNull".equals(method.getName())) {
          setColumn(params[0], null);
        } else {
          setColumn(params[0], params[1]);
        }
        return method.invoke(statement, params);
      } else if ("getResultSet".equals(method.getName())) {
        ResultSet rs = (ResultSet) method.invoke(statement, params);
        return rs == null ? null : ResultSetLogger.newInstance(rs, statementLog, queryStack);
      } else if ("getUpdateCount".equals(method.getName())) {
        int updateCount = (Integer) method.invoke(statement, params);
        if (updateCount != -1) {
          debug("   Updates: " + updateCount, false);
        }
        return updateCount;
      } else {
        return method.invoke(statement, params);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }

  /**
   * Creates a logging version of a PreparedStatement.
   *
   * @param stmt - the statement
   * @param statementLog - the statement log
   * @param queryStack - the query stack
   * @return - the proxy
   */
  public static PreparedStatement newInstance(PreparedStatement stmt, Log statementLog, int queryStack) {
    InvocationHandler handler = new PreparedStatementLogger(stmt, statementLog, queryStack);
    ClassLoader cl = PreparedStatement.class.getClassLoader();
    return (PreparedStatement) Proxy.newProxyInstance(cl, new Class[]{PreparedStatement.class, CallableStatement.class}, handler);
  }

  /**
   * Return the wrapped prepared statement.
   *
   * @return the PreparedStatement
   */
  public PreparedStatement getPreparedStatement() {
    return statement;
  }

}

```
最后是ResultSetLogger
```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] params) throws Throwable {//ResultSetLogger的增强方法
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, params);
      }
      Object o = method.invoke(rs, params);
      if ("next".equals(method.getName())) { 
        if ((Boolean) o) {
          rows++;//只要还有下一条next,就记数加1  rows++;
          if (isTraceEnabled()) {
            ResultSetMetaData rsmd = rs.getMetaData();
            final int columnCount = rsmd.getColumnCount();
            if (first) {
              first = false;
              printColumnHeaders(rsmd, columnCount);
            }
            printColumnValues(columnCount);
          }
        } else {
          debug("     Total: " + rows, false);//开启debug情况下 打印出总数计数
        }
      }
      clearColumnInfo();
      return o;
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
```

到这里我们基本已经摸完了这块逻辑 现在复盘一下问题
- 问题1.源码中用了哪些设计模式？为什么要用这些设计模式？
	门面/外观模式 适配器模式 代理模式 简单工厂模式. 为什么要使用这些模式 太长了我就不写了,大家自行总结
- 问题2.Mybatis打开调试模式之后，能打印sql语句等信息，这是怎么实现的？实现过程中使用了什么设计模式？假如有多个日志实现,加载顺序是什么?
	在调试模式能打印日志,是mybatis框架在实际执行时对对象进行了代理,进行了增强.他使用了代理设计模式.他实现日志的顺序是:
```java
  static {
    tryImplementation(LogFactory::useSlf4jLogging);
    tryImplementation(LogFactory::useCommonsLogging);
    tryImplementation(LogFactory::useLog4J2Logging);
    tryImplementation(LogFactory::useLog4JLogging);
    tryImplementation(LogFactory::useJdkLogging);
    tryImplementation(LogFactory::useNoLogging);
  }
```