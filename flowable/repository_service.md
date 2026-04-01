# 存储服务API-RepositoryService
{docsify-updated}

`RepositoryService` 接口主要用于执行Flowable中与流程存储相关的数据操作，主要有以下4类。
+ 流程部署相关操作：包括对部署记录的创建、删除、查询等，主要操作部署记录表 `ACT_RE_DEPLOYMENT` 。
+ 流程定义相关操作：包括对流程定义的查询、挂起、激活等，主要操作流程定义表 `ACT_RE_PROCDEF` 。
+ 流程模型相关操作：包括对模型的新建、保存、删除等，主要操作流程模型表 `ACT_RE_MODEL` 。
+ 流程与发起人关系管理操作：包括对候选人和候选组的新增、删除、查询等，主要操作流程人员关系表 `ACT_HI_IDENTITYLINK` 。

```
@Slf4j
public class RunDemo1 extends FlowableEngineUtil {
    @Test
    public void runDemo1() {
        // 加载Flowable配置文件并初始化工作流引擎及服务
        loadFlowableConfigAndInitEngine("flowable.cfg.xml");
        // 部署流程
        Deployment deployment = repositoryService.createDeployment()
                .addClasspathResource("processes/SimpleProcess.bpmn20.xml").deploy();
        // 查询流程定义
        ProcessDefinition processDefinition = repositoryService.createProcessDefinitionQuery()
                .deploymentId(deployment.getId()).singleResult();
        log.info("流程定义ID为：{}，流程名称为：{}，版本号：{}", processDefinition.getId(), processDefinition.getName(),
                processDefinition.getVersion());
        // 再次部署流程
        Deployment deployment2 = repositoryService.createDeployment()
                .addClasspathResource("processes/SimpleProcess.bpmn20.xml").deploy();
        // 再次查询流程定义
        ProcessDefinition processDefinition2 = repositoryService.createProcessDefinitionQuery()
                .deploymentId(deployment2.getId()).singleResult();
        log.info("流程定义ID为：{}，流程名称为：{}，版本号：{}", processDefinition2.getId(), processDefinition2.getName(),
                processDefinition2.getVersion());

        //删除流程定义
        repositoryService.deleteDeployment(processDefinition2.getDeploymentId());

        //挂起流程定义
        repositoryService.suspendProcessDefinitionById(processDefinition2.getId());

        //激活流程定义
        repositoryService.activateProcessDefinitionById(processDefinition2.getId());
    }
}
```

流程部署操作会向 `ACT_RE_DEPLOYMENT` 、 `ACT_RE_PROCDEF` 和 `ACT_GE_BYTEARRAY` 这3张表中插入相应的记录。其中， `ACT_RE_DEPLOYMENT` 表存储流程定义名称和部署时间，每部署一次流程就会增加一条记录； `ACT_RE_PROCDEF` 表存储流程定义的基本信息，每次部署新版本的流程定义都会在该表中增加一条记录，同时使版本号加1； `ACT_GE_BYTEARRAY` 表存储流程定义相关资源信息，每部署一次流程至少会在该表中增加两条记录，一条是``.bpmn` 流程描述文件记录，另一条是 `.png` 流程图片文件记录。流程描述文件和流程图片文件均以二进制形式存储在数据库中。流程定义表 `ACT_RE_PROCDEF` 和资源表 `ACT_GE_ BYTEARRAY` 均通过字段`DEPLOYMENT_ID_` 与流程部署表 `ACT_RE_DEPLOYMENT` 的 `ID_` 字段关联。