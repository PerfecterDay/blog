# Springboot 写入PID到指定文件
{docsify-updated}


```
@SpringBootApplication
@EnableAsync
@MapperScan({"com.demo.demo.repo.mapper", "com.demo.demo.repo.mapper.cms"})
public class App {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplicationBuilder(App.class).build();
        springApplication.addListeners(new ApplicationPidFileWriter("cap.pid"));
        springApplication.run(args);
    }
}
```

默认情况下写入到当前启动路径下的 `cap.pid` 文件中，可以通过配置文件配置写入文件的路径：
```
spring:
  pid:
    file: /Users/demo/cap.pid
```