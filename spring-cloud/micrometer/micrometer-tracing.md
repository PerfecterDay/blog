# micrometer tracing
{docsify-updated}

> https://docs.micrometer.io/tracing/reference/index.html

## 概念

<center><img src="pics/trace-id.jpg" alt=""></center>


## Springboot 集成
```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-tracing-bom</artifactId>
            <version>${micrometer-tracing.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-tracing</artifactId>
    </dependency>
</dependencies>
```

迁移到 springboot 4.0：
> https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide

添加以下依赖，缺一不可：
```
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-micrometer-tracing-brave</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-micrometer-tracing</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

logback配置：
```
%d{HH:mm:ss.SSS} [%thread] %-5level %X{traceId:-} [%X{traceId:-},%X{spanId:-}] - %msg%n
```

### 跨线程传递 traceId/spanId :
```
@Bean
public TaskDecorator taskDecorator() {
    return runnable -> ContextSnapshotFactory
            .builder()
            // you can limit what thread locals to propagate
            // (e.g. .captureKeyPredicate(ObservationThreadLocalAccessor.KEY::equals))
            .build()
            .captureAll()
            .wrap(runnable);
}

@Bean
public ThreadPoolTaskExecutor executor(TaskDecorator taskDecorator) {
    return new ThreadPoolTaskExecutorBuilder()
            // add your own configs (e.g. maxPoolSize, threadNamePrefix)
            .taskDecorator(taskDecorator)
            .build();
}
```

###
```

```