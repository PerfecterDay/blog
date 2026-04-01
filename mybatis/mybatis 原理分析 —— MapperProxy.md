
# mybatis 原理分析(4) —— MapperProxy
{docsify-updated}




## MapperProxy 
前文分析到了使用 SqlSession 获取到一个接口的**动态代理**的过程。由此，我们终于知道了：   
mybatis 只要声明一个接口，不用实现，就可以进行数据库操作的大致原理了，实际上执行数据库操作的肯定是这个动态代理对象。

接下来看看到底是如何做到的：

    public class MapperProxy<T> implements InvocationHandler, Serializable {
      private static final long serialVersionUID = -6424540398559729838L;
      private final SqlSession sqlSession;
      private final Class<T> mapperInterface;
      private final Map<Method, MapperMethod> methodCache;

      public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
      }

      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (Object.class.equals(method.getDeclaringClass())) {
          try {
            return method.invoke(this, args);
          } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
          }
        }
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
      }

      private MapperMethod cachedMapperMethod(Method method) {
        MapperMethod mapperMethod = methodCache.get(method);
        if (mapperMethod == null) {
          mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
          methodCache.put(method, mapperMethod);
        }
        return mapperMethod;
      }
    }
分析 invoke 方法的代码可知：

1. 如果 method 是个对象，则直接调用该方法
2. 否则生成一个 MapperMethod 对象，并缓存起来
3. 调用 MapperMethod 的 execute 方法。

## MapperMethod 的 execute
    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        if (SqlCommandType.INSERT == command.getType()) {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = rowCountResult(sqlSession.insert(command.getName(), param));
        } else if (SqlCommandType.UPDATE == command.getType()) {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = rowCountResult(sqlSession.update(command.getName(), param));
        } else if (SqlCommandType.DELETE == command.getType()) {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = rowCountResult(sqlSession.delete(command.getName(), param));
        } else if (SqlCommandType.SELECT == command.getType()) {
          if (method.returnsVoid() && method.hasResultHandler()) {
            executeWithResultHandler(sqlSession, args);
            result = null;
          } else if (method.returnsMany()) {
            result = executeForMany(sqlSession, args);
          } else if (method.returnsMap()) {
            result = executeForMap(sqlSession, args);
          } else {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = sqlSession.selectOne(command.getName(), param);
          }
        } else if (SqlCommandType.FLUSH == command.getType()) {
            result = sqlSession.flushStatements();
        } else {
          throw new BindingException("Unknown execution method for: " + command.getName());
        }
        if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
          throw new BindingException("Mapper method '" + command.getName() 
              + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
      }
execute 方法根据命令类型调用 sqlSession 相应的方法，回忆入门篇中，我们可以使用 sqlSession 的 相应方法直接执行查询，也可以使用接口编程的方法，即获取 mapper 的方法。通过上面的分析，面向接口的方法实际上还是调用了 sqlSession 的相应方法，殊途同归。所以接下来，我们直接分析 sqlSession 中如何执行数据库操作的即可，详见后文。

## 总结
至此，我们从了解了 SqlSessionFactoyBuilder 构建 SqlSessionFactory , 从 SqlSessionFactory 中获取 SqlSession 实例，使用 SqlSession 获取 Mapper 接口的动态代理对象的过程，并且也知道了，动态代理对象实际上也是调用了 SqlSession 的数据库查询方法执行的。
