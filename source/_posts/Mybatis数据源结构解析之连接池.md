---
title: MyBatis数据源结构解析之连接池
date: 2022-03-19 21:30:44
tags: MyBatis
---

### 概述

连接池的作用就是为了提高性能，将已经创建好的连接保存在池中，当有请求来时，直接使用已经创建好的连接对Server端进行访问。这样省略了创建连接和销毁连接的过程，从而在性能上得到了提高。

连接池设计的基本原理是这样的：
（1）建立连接池对象（服务启动）。
（2）按照事先指定的参数创建初始数量的连接（即：空闲连接数）。
（3）对于一个访问请求，直接从连接池中得到一个连接。如果连接池对象中没有空闲的连接，且连接数没有达到最大（即：最大活跃连接数），创建一个新的连接；如果达到最大，则设定一定的超时时间，来获取连接。
（4）运用连接访问服务。
（5）访问服务完成，释放连接（此时的释放连接，并非真正关闭，而是将其放入空闲队列中。如实际空闲连接数大于初始空闲连接数则释放连接）。

（6）释放连接池对象（服务停止、维护期间，释放连接池对象，并释放所有连接）。

### 数据源的分类

打开Mybatis源码找到datasource包，可以看到3个子package

- UNPOOLED 不使用连接池的数据源
- POOLED 使用连接池的数据源
- JNDI 使用JNDI实现的数据源

MyBatis内部分别定义了实现了java.sql.DataSource接口的UnpooledDataSource，PooledDataSource类来表示UNPOOLED、POOLED类型的数据源。 如下图所示：

- PooledDataSource和UnpooledDataSrouce都实现了java.sql.DataSource接口。
- PooledDataSource持有一个UnPooledDataSource的引用，当PooledDataSource要创建Connection实例时，实际还是通过UnPooledDataSource来创建的。PooledDataSource只是提供一种缓存连接池机制。

JNDI类型的数据源DataSource，则是通过JNDI上下文中取值。



### 数据源DataSource的创建过程

在mybatis的XML配置文件中，使用<dataSource>元素来配置数据源：

```xml
<!-- 配置数据源（连接池） -->
<dataSource type="POOLED"> //这里 type 属性的取值就是为POOLED、UNPOOLED、JNDI
  <property name="driver" value="${jdbc.driver}"/>
  <property name="url" value="${jdbc.url}"/>
  <property name="username" value="${jdbc.username}"/>
  <property name="password" value="${jdbc.password}"/>
</dataSource>
```

MyBatis在初始化时，解析此文件，根据<dataSource>的type属性来创建相应类型的的数据源DataSource，即：

- type=”POOLED” ：创建PooledDataSource实例。
- type=”UNPOOLED” ：创建UnpooledDataSource实例。
- type=”JNDI” ：从JNDI服务上查找DataSource实例。



Mybatis是通过工厂模式来创建数据源对象的 我们来看看源码:

```java
public interface DataSourceFactory {

void setProperties(Properties props);

DataSource getDataSource();//生产DataSource

}
```

上述3种类型的数据源，对应有自己的工厂模式,都实现了这个DataSourceFactory。

MyBatis创建了DataSource实例后，会将其放到Configuration对象内的Environment对象中， 供以后使用。

注意dataSource 此时只会保存好配置信息，连接池此时并没有创建好连接。只有当程序在调用操作数据库的方法时，才会初始化连接。

### DataSource什么时候创建Connection对象
我们需要创建SqlSession对象并需要执行SQL语句时，这时候MyBatis才会去调用dataSource对象来创建java.sql.Connection对象。也就是说，java.sql.Connection对象的创建一直延迟到执行SQL语句的时候。

```
  @Test
  public void testMyBatisBuild() throws IOException {
      Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
      SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader);
      SqlSession sqlSession = factory.openSession();
      TestMapper mapper = sqlSession.getMapper(TestMapper.class);
      Ttest one = mapper.getOne(1L);// 直到这一行，才会去创建一个数据库连接
      System.out.println(one);
      sqlSession.close();
  }
```

### 验证Connection创建时机

首先我们先查出现在数据库的所有连接数，在数据库中执行

```sql
mysql> SELECT * FROM performance_schema.hosts;
+-----------+---------------------+-------------------+
| HOST      | CURRENT_CONNECTIONS | TOTAL_CONNECTIONS |
+-----------+---------------------+-------------------+
| NULL      |                  36 |                50 |
| localhost |                   2 |                 5 |
+-----------+---------------------+-------------------+
2 rows in set (0.00 sec)
```
显示所有的任务列表

```sql

mysql>  show full processlist;
+----+-----------------+-----------+---------+---------+--------+------------------------+-----------------------+
| Id | User            | Host      | db      | Command | Time   | State                  | Info                  |
+----+-----------------+-----------+---------+---------+--------+------------------------+-----------------------+
|  5 | event_scheduler | localhost | NULL    | Daemon  | 883049 | Waiting on empty queue | NULL                  |
| 15 | root            | localhost | kangpan | Query   |      0 | init                   | show full processlist |
+----+-----------------+-----------+---------+---------+--------+------------------------+-----------------------+
2 rows in set (0.01 sec)
```

dataSource.getConnection()执行完，至此一个connection才创建完成。我们验证一下 在dataSource.getConnection()时打一下断点。按F8 执行一步在控制台可以看到connection = com.mysql.jdbc.JDBC4Connection@1500b2f3 实例创建完毕，我们再去数据库中看看连接数加一。

```java
  protected void openConnection() throws SQLException {
  if (log.isDebugEnabled()) {
    log.debug("Opening JDBC Connection");
  }
  connection = dataSource.getConnection();// 最终获取连接的地方在这句.
  if (level != null) {
    connection.setTransactionIsolation(level.getLevel());// 设置隔离等级
  }
  setDesiredAutoCommit(autoCommit);// 是否自动提交，默认false,update不会提交到数据库，需要手动commit
}
```

### 不使用连接池的 UnpooledDataSource

当 <dataSource>的type属性被配置成了”UNPOOLED”，MyBatis首先会实例化一个UnpooledDataSourceFactory工厂实例，然后通过.getDataSource()方法返回一个UnpooledDataSource实例对象引用，我们假定为dataSource。
使用UnpooledDataSource的getConnection()，每调用一次就会产生一个新的Connection实例对象。

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
  // 封装username和password成properties
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
  // 1.初始化驱动
  initializeDriver();
  // 2.从DriverManager中获取连接，获取新的Connection对象
  Connection connection = DriverManager.getConnection(url, properties);
  // 3.配置connection属性
  configureConnection(connection);
  return connection;
}
```

UnpooledDataSource会做以下几件事情：

- 初始化驱动： 判断driver驱动是否已经加载到内存中，如果还没有加载，则会动态地加载driver类，并实例化一个Driver对象，使用DriverManager.registerDriver()方法将其注册到内存中，以供后续使用。

- 创建Connection对象： 使用DriverManager.getConnection()方法创建连接。
- 配置Connection对象： 设置是否自动提交autoCommit和隔离级别isolationLevel。
- 返回Connection对象。

我们每调用一次getConnection()方法，都会通过DriverManager.getConnection()返回新的java.sql.Connection实例,这样当然对于资源是一种浪费，为了防止重复的去创建和销毁连接,于是引入了连接池的概念。

### 使用了连接池的 PooledDataSource

要理解连接池，首先要了解它对于connection的容器，它使用PoolState容器来管理所有的conncetion。

```
public class PooledDataSource implements DataSource {

  private static final Log log = LogFactory.getLog(PooledDataSource.class);

  private final PoolState state = new PoolState(this);

  private final UnpooledDataSource dataSource;
}
```

在PoolState中，它将connection分为两种状态，空闲状态（idle）和活动状态(active)，他们分别被存储到PoolState容器内的idleConnections和activeConnections两个ArrayList中。

```
public class PoolState {

  protected PooledDataSource dataSource;

  protected final List<PooledConnection> idleConnections = new ArrayList<PooledConnection>();
  
  protected final List<PooledConnection> activeConnections = new ArrayList<PooledConnection>();
}
```

- idleConnections：空闲(idle)状态PooledConnection对象被放置到此集合中，表示当前闲置的没有被使用的PooledConnection集合，调用PooledDataSource的getConnection()方法时，会优先从此集合中取PooledConnection对象。当用完一个java.sql.Connection对象时，MyBatis会将其包裹成PooledConnection对象放到此集合中。

- activeConnections：活动(active)状态的PooledConnection对象被放置到名为activeConnections的ArrayList中，表示当前正在被使用的PooledConnection集合，调用PooledDataSource的getConnection()方法时，会优先从idleConnections集合中取PooledConnection对象，如果没有，则看活跃状态的线程集合是否已满，如果未满，PooledDataSource会创建出一个PooledConnection，添加到此集合中，并返回。

#### 从连接池中获取一个连接对象的过程

```java
private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    while (conn == null) {
      synchronized (state) {// 给state对象加锁
        if (!state.idleConnections.isEmpty()) {// 如果空闲列表不空，就从空闲列表中拿connection
          // Pool has available connection
          conn = state.idleConnections.remove(0);// 拿出空闲列表中的第一个，去验证连接是否还有效
          if (log.isDebugEnabled()) {
            log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
          }
        } else {
          // 空闲连接池中没有可用的连接，就来看看活跃连接列表中是否有，先判断活动连接总数 是否小于 最大可用的活动连接数
          if (state.activeConnections.size() < poolMaximumActiveConnections) {
            // 如果连接数小于list.size 直接创建新的连接。
            conn = new PooledConnection(dataSource.getConnection(), this);
            if (log.isDebugEnabled()) {
              log.debug("Created connection " + conn.getRealHashCode() + ".");
            }
          } else {
            // 此时连接数也满了，不能创建新的连接。找到最老的那个，检查它是否过期
            // 计算它的校验时间，如果校验时间大于连接池规定的最大校验时间，则认为它已经过期了
            // 利用这个PoolConnection内部的realConnection重新生成一个PooledConnection
            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
            if (longestCheckoutTime > poolMaximumCheckoutTime) {
              // 可以要求过期这个连接。
              state.claimedOverdueConnectionCount++;
              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
              state.accumulatedCheckoutTime += longestCheckoutTime;
              state.activeConnections.remove(oldestActiveConnection);
              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                try {
                  oldestActiveConnection.getRealConnection().rollback();
                } catch (SQLException e) {
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
              // 如果不能释放，则必须等待
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
          if (conn.isValid()) {// 去验证连接是否还有效。
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

#### 复用连接的过程

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    // 当调用关闭的时候，回收此Connection到PooledDataSource中
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
