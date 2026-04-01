# maven-依赖管理
{docsify-updated}

Spring Boot 本身通过“依赖管理”机制简化了依赖版本的控制，使用的是 Spring Boot 的 BOM（Bill of Materials）。

1. 使用 `spring-boot-starter-parent`（推荐方式）
    ```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version> <!-- 使用你需要的 Spring Boot 版本 -->
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    ```
2. 不使用 `starter-parent` 的话，可使用 `spring-boot-dependencies` BOM，适用于你已经有 **自定义 parent** 的情况：
   ```
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.5</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
   ```

同样的 spring-cloud 也有自己的 BOM,如果项目中使用了 spring cloud 的组件，这些组件版本与 springboot 不兼容的话就会出现各种问题，所以要使用 spring-cloud 的BOM：
```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.1</version> <!-- 对应 Spring Boot 3.2 -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

spring-cloud 版本与 springboot 的[兼容性查询](https://spring.io/projects/spring-cloud#overview).