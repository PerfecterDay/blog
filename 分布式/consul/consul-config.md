# consul 作为配置中心
{docsify-updated}

> https://developer.hashicorp.com/consul/docs/automate/kv

## consul-cli 中添加配置并查询
UI创建配置值：
<center><img src="pics/config-key.png" alt=""></center>

读取配置值：
```
consul kv get /redis/conf/username
consul kv get /test
```

也可以使用命令行创建配置值：
```
consul kv put redis/conf/pwd test
```
但是注意，此时 key 名不能以 `/` 开头，否则会报错。

## Restful API
+ 查询 key 的值： `GET /kv/:key` - `curl http://127.0.0.1:8500/v1/kv/my-key`
+ 创建/更新 key值, contents 参数是任意的，consul 按照原样存储： `PUT /kv/:key` - `curl --request PUT --data @contens http://127.0.0.1:8500/v1/kv/my-key`
+ 删除 key： `DELETE /kv/:key` - `curl --request DELETE http://127.0.0.1:8500/v1/kv/my-key`

## Springboot 集成
> https://docs.spring.io/spring-cloud-consul/reference/config.html

```
<project>
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>{spring-boot-version}</version>
    <relativePath/> <!-- lookup parent from repository -->
  </parent>

  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-consul-config</artifactId>
    </dependency>
  </dependencies>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

Spring Cloud Consul Config 会根据**应用名**和 **profile** 自动构造出要读取的 KV 路径：
+ 当指定的是 `YAML/PROPERTIES` format 时， `data-key` 必须被指定并且创建：
```
config/test,default/data    --> {prefix}/{applicationName},{profile}/{dataKey}
config/test/data
config/application,default/data ---> {prefix}/{applicationName},{profile}/{dataKey}
config/application/data
```

+ 当指定的是 `files` format 时：
```
config/test,default.properties
config/test,default.yaml
config/test,default.yaml
config/test.properties
config/test.yaml
config/test.yml
```


配置：
```
spring:
  config:
    import: consul
  application:
    name: user-center-service
  cloud:
    consul:
      host: localhost
      port: 8500
      config:
        enabled: true
        format: yaml
        prefix: user-center
        default-context: default
```

| 属性                                           | 作用                                | 默认值       |
| ---------------------------------------------- | ----------------------------------- | ------------ |
| `spring.cloud.consul.config.prefixes`            | Consul KV 的配置的文件夹路径数组     | `config`     |
| `spring.cloud.consul.config.data-key`          | 每个路径下用于存放配置内容的 key 名 | `data`       |
| `spring.cloud.consul.config.format`            | 存储内容格式（yaml 或 properties）  | `properties` |
| `spring.cloud.consul.config.profile-separator` | 应用名和 profile 之间的分隔符       | `,`          |
| `spring.cloud.consul.config.defaultContext`    | 所用应用共享的文件夹路径      | `application`     |
| `spring.cloud.consul.config.watch.delay`       | Consul 监听配置变化的刷新周期       | `1000ms`     |

配置类 ：`ConsulConfigProperties`

## Consul 配置监听实现动态刷新
`ConfigWatch` : 定时拉取配置，有更新时发布 `RefreshEvent`事件。  `spring.cloud.consul.config.watch.delay` 可以配置拉取间隔时间。