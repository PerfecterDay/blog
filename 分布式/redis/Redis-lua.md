#  Redis Lua 脚本
{docsify-updated}
> https://redis.com.cn/commands/eval.html

- [Redis Lua 脚本](#redis-lua-脚本)
		- [执行Lua 脚本](#执行lua-脚本)
		- [Debug Lua 脚本](#debug-lua-脚本)


### 执行Lua 脚本
EVAL 和 EVALSHA 使用 Lua 解释器执行脚本
EVAL 命令基本语法如下：

```
redis 127.0.0.1:6379> EVAL script numkeys key [key ...] arg [arg ...] 
```
参数说明：

+ `script` ： 参数是一段 Lua 5.1 脚本程序，在 redis 服务器上下文执行。脚本不必(也不应该)定义为一个 Lua 函数。
+ `numkeys` ： 用于指定键名参数的个数。
+ `key [key ...]` ： 从 EVAL 的第三个参数开始算起，表示在脚本中所用到的那些 Redis 键(key)，这些键名参数可以在 Lua 中通过全局变量 KEYS 数组，用 1 为基址的形式访问( KEYS[1] ， KEYS[2] ，以此类推)。
+ `arg [arg ...]` ： 附加参数，在 Lua 中通过全局变量 ARGV 数组访问，访问的形式和 KEYS 变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。
通过一个实例来解释一下：
```
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

### Debug Lua 脚本
```
./redis-cli --ldb --eval /tmp/script.lua mykey somekey , arg1 arg2
```