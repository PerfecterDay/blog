# Spring Modulith
{docsify-updated}

> https://docs.spring.io/spring-modulith/reference/index.html
> https://www.baeldung.com/spring-modulith

Spring Modulith 是一个具有明确设计理念的工具包，用于基于 Spring Boot 构建领域驱动的模块化应用程序。正如 Spring Boot 对应用程序的技术架构有其设计理念，Spring Modulith 同样对应用程序的功能结构化方式提出明确主张，并允许其独立的逻辑组件相互协作。由此，Spring Modulith 使开发人员能够构建更易于更新的应用程序，从而适应随时间变化的业务需求。

## 使用
Spring Modulith 由一组可独立使用的库组成，具体取决于您需要使用其哪些功能。为简化各模块的声明过程， Spring boot 提供了（BOM）：
```
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.modulith</groupId>
      <artifactId>spring-modulith-bom</artifactId>
      <version>2.0.1</version>
      <scope>import</scope>
      <type>pom</type>
    </dependency>
  </dependencies>
</dependencyManagement>
```

