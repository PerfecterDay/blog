
# mybatis 原理分析(5) —— SqlSession 查询分析
{docsify-updated}




在前面的博文中，我们分析了 mybatis 使用接口编程时，用到了动态代理技术，实际上在代理对象中还是使用了 SqlSession 的相应方法去执行数据库操作，与直接使用 SqlSession 执行 sql 语句一样。所以，此文我们分析一下 SqlSession 执行 sql 的过程。

## SqlSession 的 selectList
    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
          MappedStatement ms = configuration.getMappedStatement(statement);
          return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
      }
首先，从 Configuration 实例中获取了一个 MappedStatement 对象，然后使用 Executor 执行 query 方法操作数据库查询。


## Executor 的 query
BaseExecutor 实现了 Executor 的一些基本方法。 SimpleExecutor 、 BatchExecutor 、 ReuseExecutor 都继承了 BaseExecutor 。

     public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
       BoundSql boundSql = ms.getBoundSql(parameter);
       CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
       return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }

     public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
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
         list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
         if (list != null) {
           handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
         } else {
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
     }

     private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
         List<E> list;
         localCache.putObject(key, EXECUTION_PLACEHOLDER);
         try {
           list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
         } finally {
           localCache.removeObject(key);
         }
         localCache.putObject(key, list);
         if (ms.getStatementType() == StatementType.CALLABLE) {
           localOutputParameterCache.putObject(key, parameter);
         }
         return list;
       }

        protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
              throws SQLException;
跟踪上述代码可以看出 BaseExecutor 中的 query 方法， 最终会调用**抽象方法** doQuery ,子类会重写 doQuery 方法。我们看看 SimpleExecutor 中如何重写 doQuery 方法的。

## SimpleExecutor 中的 doQuery
    public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
          Configuration configuration = ms.getConfiguration();
          StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
          stmt = prepareStatement(handler, ms.getStatementLog());
          return handler.<E>query(stmt, resultHandler);
        } finally {
          closeStatement(stmt);
        }
      }   
SimpleExecutor 的 doQuery 方法中，首先获取 Configuration 实例，然后调用 Configuration 的 newStatementHandler 方法获取了一个 StatementHandler 对象。接着调用了 prepareStatement 方法： 

      private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
          Statement stmt;
          Connection connection = getConnection(statementLog);
          stmt = handler.prepare(connection);
          handler.parameterize(stmt);
          return stmt;
        } 
在 prepareStatement 方法中首先获取 Connection 对象， 然后调用 StatementHandler 的 prepare 方法获得了 Statement 对象，接着使用 StatementHandler 的 parameterize 方法，从名字可以看出 parameterize 是为 sql 语句参数赋值的，其中还调用了 ParameterHandler 的 setParameters 方法，而且，ParameterHandler 又用到了 TypeHandler ， 此处，我们先不做分析。

接着上面的 doQuery 方法，获取到 Statement 后，接着调用了 StatementHandler 的 query 方法，看一下 PreparedStatementHandler 的 query 方法：
    
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        return resultSetHandler.<E> handleResultSets(ps);
      }
直接使用了 JDBC 中的 PreparedStatement 对象执行数据库操作了，所以 ORM框架底层还是使用的 JDBC 。执行完后，使用 ResultSetHandler 的 handleResultSets 方法处理结果集。


## 总结
SqlSession 下的四大对象：

+ Executor ：执行器，由它统一调度其他三个对象来执行对应的SQL；
+ StatementHandler ：使用数据库的Statement执行操作；
+ ParameterHandler ：用于SQL对参数的处理；
+ ResultHandler ：进行最后数据集的封装返回处理；

SqlSession 的查询方法实际上是靠 Executor 来实现的， Executor 在查询过程中使用了 StatementHandler 获取 JDBC 中的 Statement 对象，并为其置参数。 接着使用 Statement 对象执行查询。 在这期间，使用了 ParameterHandler 、 TypeHandler 来设置参数，使用 ResultSetHandler 来处理结果集， ResultSetHandler 内部使用了 ResultHandler 来处理结果集。

![sqlSession运行图](../pics/sqlSession运行图.png)