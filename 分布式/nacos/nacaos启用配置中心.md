# 启用Nacos 配置中心
{docsify-updated}

1. 添加依赖
   ```
   <dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-config-spring-boot-starter</artifactId>
    <version>${latest.version}</version>
   </dependency>
   ```
2. 在 application.properties 中配置 Nacos server 的地址：
   ```
   nacos:
    config:
      server-addr: 127.0.0.1:8848
      username: nacos
      password: 123456
   ```
3. 使用 `@NacosPropertySource` 加载 dataId 为 example 的配置源，并开启自动更新：
   ```
    @SpringBootApplication
    @NacosPropertySource(dataId = "TEST_CONFIG", groupId = "TEST_GROUP", autoRefreshed = true)
    public class NacosConfigApplication {
        public static void main(String[] args) {
            SpringApplication.run(NacosConfigApplication.class, args);
        }
    }
   ```
4. 通过 Nacos 的 `@NacosValue` 注解设置属性值。
   ```
   @Controller
   @RequestMapping("config")
   public class ConfigController {
       @NacosValue(value = "${useLocalCache:false}", autoRefreshed = true)
       private boolean useLocalCache;
   
       @RequestMapping(value = "/get", method = GET)
       @ResponseBody
       public boolean get() {
           return useLocalCache;
       }
   }
   ```

## 原理
> https://zhuanlan.zhihu.com/p/618729460