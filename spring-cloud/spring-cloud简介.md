# spring cloud 简介
{docsify-updated}

> https://spring.io/projects/spring-cloud

Spring Cloud 为开发人员提供了工具，可快速构建分布式系统中的常见模式（例如配置管理、服务发现、断路器、智能路由、微代理、控制总线和短生命周期微服务）。分布式系统的协调往往会产生一些重复的代码模式，而借助 Spring Cloud，开发者可以快速搭建实现这些模式的服务和应用程序。这些服务和应用程序在任何分布式环境中都能良好运行，包括开发者自己的笔记本电脑、裸机数据中心以及 Cloud Foundry 等托管平台。


## Features
Spring Cloud 致力于为典型用例提供良好的开箱即用体验，并提供可扩展机制以满足其他用例的需求。

+ Distributed/versioned configuration
+ Service registration and discovery
+ Routing
+ Service-to-service calls
+ Load balancing
+ Circuit Breakers
+ Distributed messaging
+ Short lived microservices (tasks)

## Quick start

### 创建一个新的 Spring Cloud 项目
入门最简单的方法是访问 `start.spring.io[http://start.spring.io/]` ，选择您所需的 Spring Boot 版本以及想要使用的 Spring Cloud 项目。这样在生成项目时，系统会将相应的 Spring Cloud BOM 版本添加到您的 Maven/Gradle 文件中。

### 在一个已有的 Springboot 项目中添加 Spring Cloud
如果你想在现有的 Spring Boot 应用中添加 Spring Cloud，第一步是确定应使用哪个版本的 Spring Cloud。应使用的版本取决于你当前使用的 Spring Boot 版本。  
具体的Spring boot 和相关项目的版本兼容性信息可以参考： https://spring.io/projects/generations

