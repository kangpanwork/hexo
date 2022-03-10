---
title: MyBatis-Plugin-Development
date: 2022-03-10 14:59:54
tags: MyBatis
---



## MyBatis插件开发流程

- 类实现Interceptor接口；

- 类上添加注解

  ```
  @Intercepts({@Signature(type, method, args)})
  ```

  - **type**：需要拦截的对象，只可取四大对象之一Executor.class、StatementHandler.class、ParameterHandler.class、ResultSetHandler.class。
  - **method**：拦截的对象方法。
  - **args**：拦截的对象方法参数。

- 实现拦截的方法Object intercept(Invocation invocation)。

##  Interceptor接口

```
public interface Interceptor {

    /**
     * 此方法将直接覆盖被拦截对象的原有方法
     *
     * @param invocation 通过该对象可以反射调度拦截对象的方法
     * @return
     * @throws Throwable
     */
    Object intercept(Invocation invocation) throws Throwable;

    /**
     * 为被拦截对象生成一个代理对象，并返回它
     *
     * @param target 被拦截的对象
     * @return
     */
    Object plugin(Object target);

    /**
     * 设置插件配置的参数
     *
     * @param properties 插件配置的参数
     */
    void setProperties(Properties properties);

}
```

## 步骤

### 确定拦截的方法签名

需要在实现Interceptor接口的类上加入@Intercepts({@Signature(type, method, args)})注解才能够运行插件。

#### type－拦截的对象

- Executor 执行的SQL 全过程，包括组装参数、组装结果返回和执行SQL的过程等都可以拦截。
- StatementHandler 执行SQL的过程，拦截该对象可以重写执行SQL的过程。
- ParameterHandler 执行SQL 的参数组装，拦截该对象可以重写组装参数的规则。
- ResultSetHandler 执行结果的组装，拦截该对象可以重写组装结果的规则。

对于分页插件，我们只需要拦截StatementHandler对象，重写SELECT类型的SQL语句，实现分页功能。



#### method－拦截的方法

我们已经能够确定拦截的对象是StatementHandler了，现在我们要确定拦截的是哪个方法，因为StatementHandler是通过prepare方法对SQL进行预编译的，所以我们需要对prepare方法进行拦截，在这个方法执行之前，完成SQL的重新编写，加入limit。

**StatementHandler**

```
public interface StatementHandler {

  /**
   * 预编译SQL
   *
   * @param connection
   * @return
   * @throws SQLException
   */
  Statement prepare(Connection connection)
      throws SQLException;

  /**
   * 设置参数
   *
   * @param statement
   * @throws SQLException
   */
  void parameterize(Statement statement)
      throws SQLException;

  /**
   * 批处理
   *
   * @param statement
   * @throws SQLException
   */
  void batch(Statement statement)
      throws SQLException;

  /**
   * 执行更新操作
   *
   * @param statement
   * @return 返回影响行数
   * @throws SQLException
   */
  int update(Statement statement)
      throws SQLException;

  /**
   * 执行查询操作，将结果交给ResultHandler进行结果的组装
   *
   * @param statement
   * @param resultHandler
   * @param <E>
   * @return 返回查询的数据列表
   * @throws SQLException
   */
  <E> List<E> query(Statement statement, ResultHandler resultHandler)
      throws SQLException;

  /**
   * 得到绑定的sql
   * 
   * @return
   */
  BoundSql getBoundSql();

  /**
   * 得到参数处理器
   * 
   * @return
   */
  ParameterHandler getParameterHandler();

}
```

#### args－拦截的参数

args是一个Class类型的数组，表示的是被拦截方法的参数列表。由于我们已经确定了拦截的是StatementHandler的prepare方法，而该方法只有一个参数Connection，所以我们只需要拦截这一个参数即可。

### 实现拦截方法

定义一个封装分页参数的类Page

```
public class Page {

    /**
     * 当前页码
     */
    private Integer pageIndex;
    /**
     * 每页数据条数
     */
    private Integer pageSize;
    /**
     * 总数据数
     */
    private Integer total;
    /**
     * 总页数
     */
    private Integer totalPage;

    public Page() {
    }

    public Page(Integer pageIndex, Integer pageSize) {
        this.pageIndex = pageIndex;
        this.pageSize = pageSize;
    }
	// 省略get、set方法...
}
```

实现插件分页的功能

```
import org.apache.ibatis.executor.parameter.ParameterHandler;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.SystemMetaObject;
import org.apache.ibatis.scripting.defaults.DefaultParameterHandler;
import org.apache.ibatis.session.Configuration;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.Map;
import java.util.Properties;
import java.util.Set;

@Intercepts({@Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class}
)})
public class PagingPlugin implements Interceptor {

    /**
     * 默认页码
     */
    private Integer defaultPageIndex;
    /**
     * 默认每页数据条数
     */
    private Integer defaultPageSize;

    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = getUnProxyObject(invocation);
        MetaObject metaObject = SystemMetaObject.forObject(statementHandler);
        String sql = getSql(metaObject);
        if (!checkSelect(sql)) {
            // 不是select语句，进入责任链下一层
            return invocation.proceed();
        }

        BoundSql boundSql = (BoundSql) metaObject.getValue("delegate.boundSql");
        Object parameterObject = boundSql.getParameterObject();
        Page page = getPage(parameterObject);
        if (page == null) {
            // 没有传入page对象，不执行分页处理，进入责任链下一层
            return invocation.proceed();
        }

        // 设置分页默认值
        if (page.getPageIndex() == null) {
            page.setPageIndex(this.defaultPageIndex);
        }
        if (page.getPageSize() == null) {
            page.setPageSize(this.defaultPageSize);
        }
        // 设置分页总数，数据总数
        setTotalToPage(page, invocation, metaObject, boundSql);
        // 校验分页参数
        checkPage(page);
        return changeSql(invocation, metaObject, boundSql, page);
    }

    public Object plugin(Object target) {
        // 生成代理对象
        return Plugin.wrap(target, this);
    }

    public void setProperties(Properties properties) {
        // 初始化配置的默认页码，无配置则默认1
        this.defaultPageIndex = Integer.parseInt(properties.getProperty("default.pageIndex", "1"));
        // 初始化配置的默认数据条数，无配置则默认20
        this.defaultPageSize = Integer.parseInt(properties.getProperty("default.pageSize", "20"));
    }

    /**
     * 从代理对象中分离出真实对象
     *
     * @param invocation
     * @return
     */
    private StatementHandler getUnProxyObject(Invocation invocation) {
        // 取出被拦截的对象
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MetaObject metaStmtHandler = SystemMetaObject.forObject(statementHandler);
        Object object = null;
        // 分离代理对象
        while (metaStmtHandler.hasGetter("h")) {
            object = metaStmtHandler.getValue("h");
            metaStmtHandler = SystemMetaObject.forObject(object);
        }

        return object == null ? statementHandler : (StatementHandler) object;
    }

    /**
     * 判断是否是select语句
     *
     * @param sql
     * @return
     */
    private boolean checkSelect(String sql) {
        // 去除sql的前后空格，并将sql转换成小写
        sql = sql.trim().toLowerCase();
        return sql.indexOf("select") == 0;
    }

    /**
     * 获取分页参数
     *
     * @param parameterObject
     * @return
     */
    private Page getPage(Object parameterObject) {
        if (parameterObject == null) {
            return null;
        }

        if (parameterObject instanceof Map) {
            // 如果传入的参数是map类型的，则遍历map取出Page对象
            Map<String, Object> parameMap = (Map<String, Object>) parameterObject;
            Set<String> keySet = parameMap.keySet();
            for (String key : keySet) {
                Object value = parameMap.get(key);
                if (value instanceof Page) {
                    // 返回Page对象
                    return (Page) value;
                }
            }
        } else if (parameterObject instanceof Page) {
            // 如果传入的是Page类型，则直接返回该对象
            return (Page) parameterObject;
        }

        // 初步判断并没有传入Page类型的参数，返回null
        return null;
    }

    /**
     * 获取数据总数
     *
     * @param invocation
     * @param metaObject
     * @param boundSql
     * @return
     */
    private int getTotal(Invocation invocation, MetaObject metaObject, BoundSql boundSql) {
        // 获取当前的mappedStatement对象
        MappedStatement mappedStatement = (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        // 获取配置对象
        Configuration configuration = mappedStatement.getConfiguration();
        // 获取当前需要执行的sql
        String sql = getSql(metaObject);
        // 改写sql语句，实现返回数据总数 $_paging取名是为了防止数据库表重名
        String countSql = "select count(*) as total from (" + sql + ") $_paging";
        // 获取拦截方法参数，拦截的是connection对象
        Connection connection = (Connection) invocation.getArgs()[0];
        PreparedStatement pstmt = null;
        int total = 0;

        try {
            // 预编译查询数据总数的sql语句
            pstmt = connection.prepareStatement(countSql);
            // 构建boundSql对象
            BoundSql countBoundSql = new BoundSql(configuration, countSql, boundSql.getParameterMappings(),
                    boundSql.getParameterObject());
            // 构建parameterHandler用于设置sql参数
            ParameterHandler parameterHandler = new DefaultParameterHandler(mappedStatement, boundSql.getParameterObject(),
                    countBoundSql);
            // 设置sql参数
            parameterHandler.setParameters(pstmt);
            //执行查询
            ResultSet rs = pstmt.executeQuery();
            while (rs.next()) {
                total = rs.getInt("total");
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            if (pstmt != null) {
                try {
                    pstmt.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }

        // 返回总数据数
        return total;
    }

    /**
     * 设置总数据数、总页数
     *
     * @param page
     * @param invocation
     * @param metaObject
     * @param boundSql
     */
    private void setTotalToPage(Page page, Invocation invocation, MetaObject metaObject, BoundSql boundSql) {
        // 总数据数
        int total = getTotal(invocation, metaObject, boundSql);
        // 计算总页数
        int totalPage = total / page.getPageSize();
        if (total % page.getPageSize() != 0) {
            totalPage = totalPage + 1;
        }

        page.setTotal(total);
        page.setTotalPage(totalPage);
    }

    /**
     * 校验分页参数
     *
     * @param page
     */
    private void checkPage(Page page) {
        // 如果当前页码大于总页数，抛出异常
        if (page.getPageIndex() > page.getTotalPage()) {
            throw new RuntimeException("当前页码［" + page.getPageIndex() + "］大于总页数［" + page.getTotalPage() + "］");
        }
        // 如果当前页码小于总页数，抛出异常
        if (page.getPageIndex() < 1) {
            throw new RuntimeException("当前页码［" + page.getPageIndex() + "］小于［1］");
        }
    }

    /**
     * 修改当前查询的sql
     *
     * @param invocation
     * @param metaObject
     * @param boundSql
     * @param page
     * @return
     */
    private Object changeSql(Invocation invocation, MetaObject metaObject, BoundSql boundSql, Page page) throws Exception {
        // 获取当前查询的sql
        String sql = getSql(metaObject);
        // 修改sql，$_paging_table_limit取名是为了防止数据库表重名
        String newSql = "select * from (" + sql + ") $_paging_table_limit limit ?, ?";
        // 设置当前sql为修改后的sql
        setSql(metaObject, newSql);

        // 获取PreparedStatement对象
        PreparedStatement pstmt = (PreparedStatement) invocation.proceed();
        // 获取sql的总参数个数
        int parameCount = pstmt.getParameterMetaData().getParameterCount();
        // 设置分页参数
        pstmt.setInt(parameCount - 1, (page.getPageIndex() - 1) * page.getPageSize());
        pstmt.setInt(parameCount, page.getPageSize());

        return pstmt;
    }

    /**
     * 获取当前查询的sql
     *
     * @param metaObject
     * @return
     */
    private String getSql(MetaObject metaObject) {
        return (String) metaObject.getValue("delegate.boundSql.sql");
    }

    /**
     * 设置当前查询的sql
     *
     * @param metaObject
     */
    private void setSql(MetaObject metaObject, String sql) {
        metaObject.setValue("delegate.boundSql.sql", sql);
    }
}
```

### 配置分页插件

在mybatis-config.xml配置文件中配置自定义的分页插件

```
<plugins>
	<plugin interceptor="PagingPlugin">
		<property name="default.pageIndex" value="1"/>
		<property name="default.pageSize" value="20"/>
	</plugin>
</plugins>
```

### 实现DAO

```
public class Role {

   private Long id;
   private String roleName;
   private String note;
   // 省略get、set...
}
```

定义Mapper接口，通过分页对象查询角色列表

```
public interface RoleMapper {
    List<Role> listRoleByPage(Page page);
}
```

定义Mapper.xml编写查询的SQL语句

```
<mapper namespace="RoleMapper">
    <select id="listRoleByPage" resultType="Role">
        SELECT id, role_name, note FROM role
    </select>
</mapper>
```

### 测试分页插件

测试代码

```
@Test
public void test() {
	InputStream inputStream = null;
	SqlSessionFactory sqlSessionFactory;
	SqlSession sqlSession = null;
	try {
		inputStream = Resources.getResourceAsStream("mybatis-config.xml");
		sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
		sqlSession = sqlSessionFactory.openSession();
		RoleMapper roleMapper = sqlSession.getMapper(RoleMapper.class);
		// 分页参数，从第一页开始，每页显示5条数据
		Page page = new Page(1, 5);
		List<Role> roleList = roleMapper.listRoleByPage(page);
		System.out.println("===分页信息===");
		System.out.println("当前页码：" + page.getPageIndex());
		System.out.println("每页显示数据数：" + page.getPageSize());
		System.out.println("总数据数：" + page.getTotal());
		System.out.println("总页数：" + page.getTotalPage());
		System.out.println("=============");
		System.out.println("===数据列表===");
		for (Role role : roleList) {
			System.out.println(role);
		}
	} catch (IOException e) {
		e.printStackTrace();
	} finally {
		if (sqlSession != null) {
			sqlSession.close();
		}
		if (inputStream != null) {
			try {
				inputStream.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}
}
```

数据库role表中的全部数据信息

| id   | role_name   | note       |
| ---- | ----------- | ---------- |
| 1    | SUPER_ADMIN | 超级管理员 |
| 2    | admin       | 管理员     |
| 3    | user        | 用户       |
| 4    | user2       | 用户2      |
| 8    | user3       | 用户3      |
| 9    | test        | 测试       |
| 10   | test2       | 测试2      |
| 11   | test3       | 测试3      |
| 12   | test4       | 测试4      |
| 13   | test5       | 测试5      |

代码执行结果

```
===分页信息===
当前页码：1
每页显示数据数：5
总数据数：10
总页数：2
=============
===数据列表===
Role{id=1, roleName='SUPER_ADMIN', note=' 超级管理员'}
Role{id=2, roleName='admin', note='管理员'}
Role{id=3, roleName='user', note='用户'}
Role{id=4, roleName='user2', note='用户2'}
Role{id=8, roleName='user3', note='用户3'}
```

打印的SQL信息

```
==>  Preparing: select count(*) as total from (SELECT id, role_name, note FROM role) $_paging 
==> Parameters: 
<==    Columns: total
<==        Row: 10
<==      Total: 1
==>  Preparing: select * from (SELECT id, role_name, note FROM role) $_paging_table_limit limit ?, ? 
==> Parameters: 0(Integer), 5(Integer)
<==    Columns: id, role_name, note
<==        Row: 1, SUPER_ADMIN,  超级管理员
<==        Row: 2, admin, 管理员
<==        Row: 3, user, 用户
<==        Row: 4, user2, 用户2
<==        Row: 8, user3, 用户3
<==      Total: 5
```