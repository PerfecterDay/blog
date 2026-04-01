#  Redission
{docsify-updated}

Redisson是一个具有内存数据网格功能的Redis Java客户端。它提供了更方便和最简单的方式来处理Redis。Redisson对象提供了一个分离的关注点，这使你能够保持对数据建模和应用逻辑的关注。


### 集成 SpringBoot
```
<dependency>
	<groupId>org.redisson</groupId>
	<artifactId>redisson-spring-boot-starter</artifactId>
	<version>3.22.1</version>
</dependency>
```
如有必要，将redisson-spring-data模块降级，以支持所需的Spring Boot版本，比如集成 SpringBoot 2.7:
```
<dependency>
	<groupId>org.redisson</groupId>
	<artifactId>redisson-spring-boot-starter</artifactId>
	<version>3.22.1</version>
	<exclusions>
		<exclusion>
			<groupId>org.redisson</groupId>
			<artifactId>redisson-spring-data-31</artifactId>
		</exclusion>
	</exclusions>
</dependency>

<dependency>
	<groupId>org.redisson</groupId>
	<artifactId>redisson-spring-data-27</artifactId>
	<version>3.22.1</version>
</dependency>
```



```
@Component
public class RedissonWrapper {
    @Resource
    Redisson redisson;

    public RLock getLock(String name) {
        return new CustomRedissonLock(redisson.getCommandExecutor(), name);
    }

    class CustomRedissonLock extends RedissonLock {

        public CustomRedissonLock(CommandAsyncExecutor commandExecutor, String name) {
            super(commandExecutor, name);
        }

        @Override
        protected RFuture<Boolean> unlockInnerAsync(long threadId) {
            return this.evalWriteAsync(this.getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN, "if (redis" +
                            ".call('hexists', KEYS[1], ARGV[3]) == 0) then return nil;end; local counter = redis.call" +
                            "('hincrby', KEYS[1], ARGV[3], -1); if (counter > 0) then redis.call('pexpire', KEYS[1], " +
                            "ARGV[2]); return 0; else redis.call('del', KEYS[1]); redis.call('"+ this.getSubscribeService().getPublishCommand() +"', KEYS[2], " +
                            "ARGV[1]); return 1; end; return nil;",
                    Arrays.asList(this.getRawName(), this.getChannelName()), new Object[]{LockPubSub.UNLOCK_MESSAGE,
                            this.internalLockLeaseTime, this.getLockName(threadId)});
        }

        protected String getChannelName() {
            return prefixName("custom_redisson_lock__channel", this.getRawName());
        }
    }
}
```

## 延时队列
https://www.cnblogs.com/zzyang/p/17119268.html
