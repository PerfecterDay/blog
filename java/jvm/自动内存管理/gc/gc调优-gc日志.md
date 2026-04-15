# gc 日志
{docsify-updated}

## GC日志配置
```
# 打印基本 GC 信息
-XX:+PrintGCDetails  ---> -Xlog:gc*
-XX:+PrintGCDateStamps  ---> -Xlog:gc*::time
-verbose:gc

# 打印对象分布
-XX:+PrintTenuringDistribution 

# 打印堆数据
-XX:+PrintHeapAtGC 

# 打印Reference处理信息
-XX:+PrintReferenceGC 

# 打印STW时间
-XX:+PrintGCApplicationStoppedTime

# 打印safepoint信息
-XX:+PrintSafepointStatistics 
-XX:PrintSafepointStatisticsCount=1

# GC日志输出的文件路径
-Xloggc:/path/to/gc-%t.log ---> -Xlog:gc*:gc.log
# 开启日志文件分割
-XX:+UseGCLogFileRotation 
# 最多分割几个文件，超过之后从头文件开始写
-XX:NumberOfGCLogFiles=14
# 每个文件上限大小，超过就触发分割
-XX:GCLogFileSize=100M
```

JDK 9 之后可以使用
```
-Xlog:gc*:file="gc.log":tags,time,uptime,level:filecount=10,filesize=10M
```