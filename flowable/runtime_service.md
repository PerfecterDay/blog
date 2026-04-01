# 运行时服务API-RuntimeService
{docsify-updated}

`RuntimeService` 接口主要用于对Flowable中与流程运行时相关的数据进行操作，对应名称以 `ACT_RU_` 开头的相关运行时表。该接口提供的方法主要包括以下6类：
+ 创建和发起流程实例；
+ 唤醒等待状态的流程实例；
+ 流程权限的管理，主要指流程实例和人员之间的关系管理，如流程参与人管理等；
+ 流程变量的管理，包括流程变量的新增、删除、查询等；
+ 管理运行时流程实例、执行实例、数据对象等运行时对象；
+ 信号、消息等事件的发布与接收，以及事件监听器的管理。


## 发起流程实例
`RuntimeService` 接口提供了以下4类发起流程实例的方法：
+ 通过创建 `ProcessInstanceBuilder` 对象后发起流程实例；
+ 通过流程定义 `key` 发起流程实例；
+ 通过流程定义 `ID` 发起流程实例；
+ 通过消息名称发起流程实例。

```
@Slf4j
public class RunDemo5 extends FlowableEngineUtil {
    @Test
    public void runDemo5() {
        // 初始化工作流引擎
        loadFlowableConfigAndInitEngine("flowable.cfg.xml");
        // 部署流程
        ProcessDefinition procDef = deployByClasspathResource("processes/SimpleProcess.bpmn20.xml");
        // 根 据 流 程 定 义 ID发 起 流 程
        ProcessInstance procInst1 = runtimeService.startProcessInstanceById(procDef.getId());
        queryProcessInstance(procInst1.getId());
        // 根 据 流 程 定 义 key发 起 流 程
        ProcessInstance procInst2 = runtimeService.createProcessInstanceBuilder().processDefinitionKey(procDef.getKey())
                .name("SimpleProcessInstance").start();
        queryProcessInstance(procInst2.getId());
    }

    private void queryProcessInstance(String procInstId) {
        ProcessInstance procInst = runtimeService.createProcessInstanceQuery().processInstanceId(procInstId)
                .singleResult();
        log.info("流程实例ID为：{}，流程定义ID为：{}，流程实例名称为：{}", procInst.getId(), procInst.getProcessDefinitionId(),
                procInst.getName());
    }
}
```

## 唤醒一个等待状态的执行实例
`RuntimeService` 接口提供了 `trigger()` 方法，用于唤醒一个等待状态的执行实例。该接口支持根据执行实例ID进行唤醒操作，还支持唤醒时传入流程变量、瞬时变量等参数。相关代码如下：
```
public interface RuntimeService {
    // 唤醒等待状态的执行
    void trigger(String executionId);

    // 唤醒等待状态的执行，但是通过异步任务来执行
    void triggerAsync(String executionId);

    // 唤醒等待状态的执行，同时传入新的流程变量
    void trigger(String executionId, Map<String, Object> processVariables);

    // 唤醒等待状态的执行，同时传入新的流程变量，但是通过异步任务来执行
    void triggerAsync(String executionId, Map<String, Object> processVariables);

    // 唤醒等待状态的执行，同时传入新的流程变量和瞬时变量
    void trigger(String executionId, Map<String, Object> processVariables, Map<String, Object> transientVariables);
}
```

下图展示了一个包含接收任务的流程模型，当发起流程实例后，用户完成“申请”节点的操作，流程流转到“等待触发”节点时，流程的执行实例就会进入等待状态，需要调用RuntimeService的trigger()方法才能触发流程的继续流转。

<center><img src="/pics/flowable_trigger.png" alt=""></center>

下面是一个调用 `RuntimeService` 的 `trigger()` 方法唤醒等待执行实例的示例代码：
```
@Slf4j
public class RunDemo6 extends FlowableEngineUtil {
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");

    @Test
    public void runDemo6() {
        // 初始化工作流引擎
        loadFlowableConfigAndInitEngine("flowable.cfg.xml");
        // 部署流程
        ProcessDefinition procDef = deployByClasspathResource("processes/SimpleProcess2.bpmn20.xml");
        // 根据流程定义ID发起流程
        ProcessInstance procInst = runtimeService.startProcessInstanceById(procDef.getId());
        log.info("流程实例的ID为：{}", procInst.getId());
        // 查询第一个任务
        Task firstTask = taskService.createTaskQuery().processInstanceId(procInst.getId()).singleResult();
        log.info("第一个任务ID为：{}，任务名称为：{}", firstTask.getId(), firstTask.getName());
        taskService.setAssignee(firstTask.getId(), "huhaiqin");
        // 完成第一个任务
        taskService.complete(firstTask.getId());
        log.info("第一个任务办理完成！");
        Execution execution = runtimeService.createExecutionQuery().processInstanceId(procInst.getId())
                .onlyChildExecutions().singleResult();
        log.info("当前执行实例ID为：{}", execution.getId());
        runtimeService.trigger(execution.getId());
        log.info("触发机器节点，继续流转！");
        // 查询流程执行历史
        HistoricProcessInstance hisProcInst = historyService.createHistoricProcessInstanceQuery()
                .processInstanceId(procInst.getId()).singleResult();
        log.info("流程实例开始时间为：{}，结束时间为：{}", dateFormat.format(hisProcInst.getStartTime()),
                dateFormat.format(hisProcInst.getEndTime()));
    }
}
```