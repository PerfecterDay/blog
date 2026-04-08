# 理解 Spring cache 抽象
{docsify-updated}
> https://docs.spring.io/spring-framework/reference/integration/cache/strategies.html

## 缓存基础知识
1. 缓存命中率
	即从缓存中 读取数据的次数与总读取 次数的 比率。一般来说，命中率越高越好。 
	1. 命中率 = 从缓存中读取的次数/总读取次数(从缓存中读取的次数+从慢速设备上读取的次数)
	2. Miss率 = 没有从缓存中读取的次数/总读取次数(从缓存中读取的次数+从慢速设备上读取的次数)

2. 缓存过期策略
	即如果缓存满了，从缓存中移除数据的策略。常见的有 LFU 、 LRU 、 FIFO 。
	+ `FIFO(FirstInFirstOut)`:先进先出策略，即先放入缓存的数据先被移除。
	+ `LRU(LcastRecentlyUsed)`:最久未使用策略，即上次使用时间距离现在最久的那个数据被移除。
	+ `LFU(LeastFrequentlyUsed)`:最近最少使用策略，即一定时间段内使用次数(频率)最少的那个数据被移除。
	+ `TMTL(TimeToLive)`:存活期，即从缓存中创建时间点开始直至到期的一个时间段(不管在这个时间段内有没有访问都将过期)。
	+ `TTI(TimeToldle)`:空闲期，即一个数据多久没被访问就从缓存中移除的时间。

## 自己实现缓存的简单实现
```
public class CacheManager<T> {
    private Map<String, T> cache = new ConcurrentHashMap<String, T>();

    public T getValue(Object key) {
        return cache.get(key);
    }

    public void addOrUpdateCache(String key, T value) {
        cache.put(key, value);
    }

    public void evictcache(String key) {
        if (cache.containsKey(key)) {
            cache.remove(key);
        }
    }

    public void evictCache() {
        cache.clear();
    }
}

public class UserService {
    private CacheManager<User> cacheManager;

    public UserService() {
        //构造一个缓存管理器
        cacheManager = new CacheManager<>();
    }
    public User getUserById(String userId) {
        //首先查询缓存
        User result = cacheManager.getValue(userId);
        if (result != null) {
            System.out.println("get from cache... . " + userId);
            return result;
        }
        result = getFromDB(userId);
        if (result != null) {
            //将结果更新到缓存中去
            cacheManager.addOrUpdateCache(userId, result);
        }
        return result;
    }

    public void reload() {
        cacheManager.evictCache();
    }

    private User getFromDB(String userId) {
        System.out.println("real querying db..." + userId);
        return new User(userId);
    }
}
```

使用SpringCache带来的好处如下:
+ 支持开箱即用(Out-Of-The-Box)，并提供基本的Cache抽象，方便切换各种底层Cache。
+ 类似于Spring提供的数据库事务管理，通过Cache注解即可实现缓存逻辑透明化，让开发者关注业务逻辑。
+ 当事务回滚时，缓存也会自动回滚。
+ 支持比较复杂的缓存逻辑。
+ 提供缓存编程的一致性抽象，方便代码维护

## spring cache 的功能
从本质上讲， `spring-cache` 将缓存机制应用于 Java 方法，从而根据缓存中已有的信息减少方法的执行次数。也就是说，每次调用目标方法时，该抽象都会触发缓存机制，检查该方法是否已针对给定的参数被调用过。如果已被调用，则直接返回缓存结果，而无需再次调用实际方法。如果该方法尚未被调用，则会调用该方法，并将结果缓存并返回给用户，以便下次调用该方法时直接返回缓存结果。通过这种方式，对于给定的一组参数，耗时较长的方法（无论是CPU密集型还是I/O密集型）只需调用一次，其结果即可被复用，而无需再次实际调用该方法。缓存逻辑以透明的方式应用，不会对调用者造成任何干扰。

此方法仅适用于那些无论被调用多少次，对于给定的输入（或参数）都能保证返回相同输出（结果）的方法。

`spring-cache` 还提供了其他与缓存相关的操作，例如更新缓存内容或删除一个或所有条目的功能。如果缓存处理的是在应用程序运行过程中可能发生变化的数据，这些操作将非常有用。

与 Spring 框架中的其他服务一样， `spring-cache` 是一种抽象（而非缓存实现），需要使用实际的存储实现来存储缓存数据——也就是说，这种抽象使您无需编写缓存逻辑，但并不提供实际的数据存储。这种抽象通过 `org.springframework.cache.Cache` 和 `org.springframework.cache.CacheManager` 接口来实现。

Spring 提供了该抽象概念的几种实现：基于 JDK `java.util.concurrent.ConcurrentMap` 的缓存、 `Gemfire` 缓存、 `Caffeine` 以及符合 `JSR-107` 标准的缓存（例如 Ehcache 3.x）。

需要注意的是`Spring Cache`并不针对多进程的应用环境进行专门的处理，也就是说，当应用程序处于分布式或者集群环境下时，需要针对具体的缓存进行相应的配置。

如果处于多进程环境（即应用程序部署在多个节点上），则需要相应地配置缓存提供程序。根据具体用例，在多个节点上复制相同的数据可能就已足够。但是，如果在应用程序运行过程中修改了数据，则可能需要启用其他传播机制以是缓存同步到其他节点。

缓存一个指定项与编程中常见的 `get-if-not-found-then-proceed-and-put-eventually` 的逻辑代码块完全等同。此过程不使用任何锁，多个线程可能会并发尝试加载同一项。删除缓存操作也是如此。如果多个线程正在并发尝试更新或删除数据，则可能会使用过期数据。某些缓存提供商在此方面提供了高级功能。有关更多详细信息，请参阅缓存提供商的文档。

要使用 `spring-cache` ，需要注意以下两个方面：
+ 缓存声明：确定需要缓存的方法及想要的缓存策略。
+ 缓存配置：用于存储数据并从中读取数据的底层缓存。
