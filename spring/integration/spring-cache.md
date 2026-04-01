#  Spring Cache的集成
{docsify-updated}

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

需要注意的是`Spring Cache`并不针对多进程的应用环境进行专门的处理，也就是说，当应用程序处于分布式或者集群环境下时，需要针对具体的缓存进行相应的配置。

## Spring Cache 的注解
1. @Cacheable  
   @Cacheable是最主要的注解，它指定了被注解方法的返回值是可被缓存的。其工作原理是**Spring首先在缓存中查找数据，如果没有则执行方法并缓存结果，然后返回数据**。缓存名是必须提供的，可以使用引号、value或者 cacheNames 属性来定义名称。
   ```
    @Cacheable ("users")
    //Spring 3.x
    @Cacheable (value="users")
    //Spring 4.0新增了valve的别名cacheNames，比value 命名更达意，推荐使用
    @Cacheable(cacheNames)
   ```

	**键生成器** 
	缓存的本质就是键/值对集合。在默认情况下，缓存抽象使用方法签名及参数值作为一个键值，并将该键与方法调用的结果组成键/值对。如果在Cache注解上没有指定key,则Spring会使用	KeyGenerator来生成一个key。Sping默认提供了 SimplekeyGenerator 生成器。其生成规则如下:
	1. 如果方法没有入参，则使用 `Simplekey.EMPTY` 作为key。
	2. 如果只有一个入参，则使用该入参作为key。
	3. 如果有多个入参，则返回包含所有入参的一个Simplekey。

	@Cacheable注解提供了实现该功能的key属性，通过该属性，可以使用SpEL指定自定义键：
	`@Cacheable(cacheNames="users",key="#user.userId")`

	**带条件的缓存**  
	使用@Cacheable注解的condition属性可按条件进行缓存，condition属性使用了SpEL表达式动态评估方法入参是否满足缓存条件。

2. @CachePut  
	@CachePut注解与@Cacheable注解效果几乎一样，**它首先执行方法，然后将返回值放入缓存，区别在于它不会取缓存中的值，只会更新缓存**。  
	需要注意的是，在同一个方法内不能同时使用@CachePut和@Cacheable注解，因为它们拥有不同的特性。当@Cacheable注解跳过方法直接获取级存时，@CachePut注解会强制执行方法以更新缓存，这会导致意想不到的情况发生，如当注解都带入了条件屆性，就会使得它们彼此排斥。还需要注意的是，@CachePut注解的condition属性设置的缓存条件也不应该依赖于方法返回的结果(如condition="#result")，因为缓存条件是在方法执行前预先验证的。

3. @CacheEvict  
	@CacheEvict注解是@Cachable注解的反向操作，它负责从给定的缓存中移除一个值。大多数缓存框架都提供了缓存数据的有效期，使用该注解可以显式地从缓存中删除失效的缓存数据。该注解通常用于更新或者删除用户的操作。  
	@CacheEvict 注解还具有两个与@Cacheable 注解不同的属性:
	+ allEntries： 是布尔类型的，用来表示是否需要清除缓存中的所有元素。默认值为false,表示不需要。当指定allEntries为true时，SpringCache将忽路指定的key，清除缓存中的所有内容
	+ beforeInvocation：定义了在调用方法之前还是在调用方法之后完成移除操作。与@Cacheable 注解不同的是，在默认情况下，清除操作默认是在对应方法**执行成功后触发**的，即方法如果因为拋出异常而未能成功返回时，则不会触发清除操作。使用beforelnvocation属性可以改变触发清除操作的时间。当指定该属性值为true时，Spring会在调用该方法之前清除缓存中的指定元素。

## JSR-107 Cache

JavaCaching定义了4个核心接又，分别是CachingProvider、CacheManager、Cache和Entry。下面对这4个核心接又进行简单介绍。
+ CachingProvider： 定义了创建、配置、获取、管理和控制多个CacheManager。个应用可以在运行期访问多个CachingProvider。
+ CacheManager：定义了创建、配置、获取、管理和控制多个唯一命名的Cache,这些Cache存在于CacheManager的上下文中。一个CacheManager仅被一个CachingProvider所拥有。
+ Cache：是一个类似Map的数据结构并存储以key为索引的值。一个Cache仅被一个CacheManager所拥有。
+ Entry：是一个存储在Cache中的键值对。每个存储在Cache中的条目都有一个定义的有效期，即ExpiryDuration。一旦超过这个时间，条目即为过期状态。一旦过期，条目将不可访问。缓存有效期可以通过ExpiryPolicy设置。

<center><img src="/pics/jcahce.jpg"></center>

## Caffeine
```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    <dependency>
        <groupId>com.github.ben-manes.caffeine</groupId>
        <artifactId>caffeine</artifactId>
    </dependency>
</dependencies>

@Bean
public Caffeine caffeineConfig() {
    return Caffeine.newBuilder().expireAfterWrite(60, TimeUnit.MINUTES);
}

@Bean
public CacheManager cacheManager(Caffeine caffeine) {
    CaffeineCacheManager caffeineCacheManager = new CaffeineCacheManager();
    caffeineCacheManager.setCaffeine(caffeine);
    return caffeineCacheManager;
}
```



## Springboot 集成 redis 缓存
1. 添加依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
    <version>2.4.3</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>2.4.3</version>
</dependency>
```

2. 配置类上加上 @EnableCaching 
3. Spring Boot will auto-configure a RedisCacheManager with default cache configuration. However, we can modify this configuration prior to cache manager initialization in a couple of useful ways.
```
@Bean
public RedisCacheConfiguration cacheConfiguration() {
    return RedisCacheConfiguration.defaultCacheConfig()
      .entryTtl(Duration.ofMinutes(60))
      .disableCachingNullValues()
      .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
}
```

```
@Bean
public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
	RedisSerializationContext.SerializationPair jsoSerilizer = RedisSerializationContext
			.SerializationPair
			.fromSerializer(new GenericJackson2JsonRedisSerializer());

	return (builder) -> builder
			.withCacheConfiguration("sms",
					RedisCacheConfiguration.defaultCacheConfig()
							.entryTtl(Duration.ofMinutes(10))
							.serializeKeysWith(jsoSerilizer)
							.serializeValuesWith(jsoSerilizer))
			.withCacheConfiguration("customerCache",
					RedisCacheConfiguration.defaultCacheConfig()
							.entryTtl(Duration.ofMinutes(5))
							.serializeKeysWith(jsoSerilizer)
							.serializeValuesWith(jsoSerilizer));
}
```


## caffine+redis 双层缓存
使用 `CompositeCacheManager` 能组合多个缓存管理器：
```
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager caffeineCacheManager() {
        Caffeine<Object, Object> caffeine = Caffeine.newBuilder()
                .expireAfterWrite(10, TimeUnit.SECONDS)
                .maximumSize(10000);
        return new CaffeineCacheManager() {{
            setCaffeine(caffeine);
        }};
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration redisCacheConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10));

        RedisCacheManager redisCacheManager = RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(redisCacheConfig)
            .build();

        // 自定义 CompositeCacheManager，将本地和 Redis 管理器组合
        CompositeCacheManager cacheManager = new CompositeCacheManager(
            caffeineCacheManager(), redisCacheManager
        );

        // 如果所有缓存都 miss，则使用 fallback（防止 NPE）
        cacheManager.setFallbackToNoOpCache(false);
        return cacheManager;
    }
}
```