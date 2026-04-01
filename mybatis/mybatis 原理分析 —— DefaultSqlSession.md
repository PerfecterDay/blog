
# mybatis 原理分析(3) —— DefaultSqlSession
{docsify-updated}




## DefaultSqlSession 的域
    public class DefaultSqlSession implements SqlSession {
      private Configuration configuration;
      private Executor executor;
      private boolean autoCommit;
      private boolean dirty;
      .......
    }

## 运行跟踪
在上一篇博文中，我们简要分析了构建一个 SqlSession 的过程，在实际使用中，我们获取 SqlSession 实例后，一般使用它的 selectOne 或者 getMapper 方法继续操作。因为推荐使用 getMapper 方法，所以我们就沿着 getMapper 方法往后看。

    public <T> T getMapper(Class<T> type) {
        return configuration.<T>getMapper(type, this);
      }
getMapper 方法调用了 configuration.getMapper ，继续跟：
    
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
      }
Configuration 类中 getMapper 方法调用了 mapperRegistry.getMapper 方法。所以要分析一下 MapperRegistry 类。

### MapperRegistry 类
直接上源码：
    
    public class MapperRegistry {
        private final Configuration config;
        private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();
        ........
    }
可见 MapperRegistry 类保存了 Configuration 的引用，还有一个 Map 对象 knownMappers ，从命名上看是已知的 mappers 的意思，所以 Mapper 映射器应该就保存在这个 Map 类型的 knownMappers 中。

看一下这个 Map 的定义：是一个以 Class 对象为键， MapperProxyFactory 为值的 Map 。回忆一下，我们调用 SqlSession 的 getMapper 时，会传入一个 Class 的对象。

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
          throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
          return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
          throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
      }
首先使用传入的 Class 在 knownMappers 中检索出对应的 MapperProxyFactory 对象，然后调用 mapperProxyFactory.newInstance 方法返回一个对象。接着看 MapperProxyFactory 类吧。

### MapperProxyFactory 类
MapperProxyFactory 类代码并不多，我们看一下所有代码：

    public class MapperProxyFactory<T> {

      private final Class<T> mapperInterface;
      private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

      public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
      }

      public Class<T> getMapperInterface() {
        return mapperInterface;
      }

      public Map<Method, MapperMethod> getMethodCache() {
        return methodCache;
      }

      @SuppressWarnings("unchecked")
      protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
      }

      public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
      }
    }
可以看出该类是用了 JDK 的动态代理， 为 mapperInterface 域创建了一个代理对象，动态代理使用 MapperProxy 类对象作为其处理器。猜想一下 MapperProxy 肯定实现了 InvocationHandler 接口。


### 总结
到此为止，我们可以做出如下结论：   
sqlSession.getMapper 方法实际上返回的是一个**动态代理对象**。

整个运行流程：   
sqlSession.getMapper --> configuration.getMapper --> mapperRegistry.getMapper --> mapperProxyFactory.newInstance --> return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);

上述分析中，我们有两点还没有做详细分析：  

1. Configuration 中的 mapperRegistry 是如何初始化的，后续博文分析。
2. 可知动态代理对象会调用 MapperProxy 里边的方法，后续博文分析。





  



