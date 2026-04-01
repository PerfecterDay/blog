# Springboot集成 flowable
{docsiy-updated}

> https://www.flowable.com/open-source/docs/bpmn/ch05a-Spring-Boot


```
<dependency>
    <groupId>org.flowable</groupId>
    <artifactId>flowable-spring-boot-starter</artifactId>
    <version>${flowable.version}</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>
```

配置数据库：
```
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/flowable?useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true&nullCatalogMeansCurrent=true&rewriteBatchedStatements=true # MySQL Connector/J 8.X 连接的示例
    username: root
    password: root
```

创建一个测试 BPMN 文件： `/resources/processes/one-task-process.bpmn20.xml`，内容如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<definitions
        xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
        xmlns:flowable="http://flowable.org/bpmn"
        targetNamespace="Examples">

    <process id="oneTaskProcess" name="The One Task Process">
        <startEvent id="theStart" />
        <sequenceFlow id="flow1" sourceRef="theStart" targetRef="theTask" />
        <userTask id="theTask" name="my task" flowable:assignee="kermit" />
        <sequenceFlow id="flow2" sourceRef="theTask" targetRef="theEnd" />
        <endEvent id="theEnd" />
    </process>

</definitions>
```

创建启动类：
```
@SpringBootApplication(proxyBeanMethods = false)
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

    @Bean
    public CommandLineRunner init(final RepositoryService repositoryService,
                                  final RuntimeService runtimeService,
                                  final TaskService taskService) {

        return new CommandLineRunner() {
            @Override
            public void run(String... strings) throws Exception {
                System.out.println("Number of process definitions : "
                    + repositoryService.createProcessDefinitionQuery().count());
                System.out.println("Number of tasks : " + taskService.createTaskQuery().count());
                runtimeService.startProcessInstanceByKey("oneTaskProcess");
                System.out.println("Number of tasks after process start: "
                    + taskService.createTaskQuery().count());
            }
        };
    }
}
```

启动运行，输出结果如下：
```
Number of process definitions : 1
Number of tasks : 0
Number of tasks after process start : 1
```


flowable 会自动创建一下表：
```
 ACT_APP_APPDEF                |
| ACT_APP_DEPLOYMENT            |
| ACT_APP_DEPLOYMENT_RESOURCE   |
| ACT_CMMN_CASEDEF              |
| ACT_CMMN_DEPLOYMENT           |
| ACT_CMMN_DEPLOYMENT_RESOURCE  |
| ACT_CMMN_HI_CASE_INST         |
| ACT_CMMN_HI_MIL_INST          |
| ACT_CMMN_HI_PLAN_ITEM_INST    |
| ACT_CMMN_RU_CASE_INST         |
| ACT_CMMN_RU_MIL_INST          |
| ACT_CMMN_RU_PLAN_ITEM_INST    |
| ACT_CMMN_RU_SENTRY_PART_INST  |
| ACT_DMN_DECISION              |
| ACT_DMN_DEPLOYMENT            |
| ACT_DMN_DEPLOYMENT_RESOURCE   |
| ACT_DMN_HI_DECISION_EXECUTION |
| ACT_EVT_LOG                   |
| ACT_GE_BYTEARRAY              |
| ACT_GE_PROPERTY               |
| ACT_HI_ACTINST                |
| ACT_HI_ATTACHMENT             |
| ACT_HI_COMMENT                |
| ACT_HI_DETAIL                 |
| ACT_HI_ENTITYLINK             |
| ACT_HI_IDENTITYLINK           |
| ACT_HI_PROCINST               |
| ACT_HI_TASKINST               |
| ACT_HI_TSK_LOG                |
| ACT_HI_VARINST                |
| ACT_ID_BYTEARRAY              |
| ACT_ID_GROUP                  |
| ACT_ID_INFO                   |
| ACT_ID_MEMBERSHIP             |
| ACT_ID_PRIV                   |
| ACT_ID_PRIV_MAPPING           |
| ACT_ID_PROPERTY               |
| ACT_ID_TOKEN                  |
| ACT_ID_USER                   |
| ACT_PROCDEF_INFO              |
| ACT_RE_DEPLOYMENT             |
| ACT_RE_MODEL                  |
| ACT_RE_PROCDEF                |
| ACT_RU_ACTINST                |
| ACT_RU_DEADLETTER_JOB         |
| ACT_RU_ENTITYLINK             |
| ACT_RU_EVENT_SUBSCR           |
| ACT_RU_EXECUTION              |
| ACT_RU_EXTERNAL_JOB           |
| ACT_RU_HISTORY_JOB            |
| ACT_RU_IDENTITYLINK           |
| ACT_RU_JOB                    |
| ACT_RU_SUSPENDED_JOB          |
| ACT_RU_TASK                   |
| ACT_RU_TIMER_JOB              |
| ACT_RU_VARIABLE               |
| FLW_CHANNEL_DEFINITION        |
| FLW_EVENT_DEFINITION          |
| FLW_EVENT_DEPLOYMENT          |
| FLW_EVENT_RESOURCE            |
| FLW_RU_BATCH                  |
| FLW_RU_BATCH_PART  
```