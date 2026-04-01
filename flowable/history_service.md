# 历史服务API-HistoryService
{docsify-updated}

历史服务接口 `HistoryService` 主要提供历史数据的查询、删除等服务，即提供与名称以 `ACT_HI_` 开头的相关的数据表服务。通过该服务接口可以查询历史流程实例、历史活动、历史任务、历史流程变量，以及删除历史流程实例、历史任务实例等。

`HistoryService` 接口代码如下：
```
~public interface HistoryService {
    // 创建历史流程实例查询
    HistoricProcessInstanceQuery createHistoricProcessInstanceQuery();

    // 创建历史活动实例查询
    HistoricActivityInstanceQuery createHistoricActivityInstanceQuery();

    // 创建历史任务实例查询
    HistoricTaskInstanceQuery createHistoricTaskInstanceQuery();

    // 创建历史详情查询
    HistoricDetailQuery createHistoricDetailQuery();

    // 创建历史变量查询
    HistoricVariableInstanceQuery createHistoricVariableInstanceQuery();

    // 查询流程实例历史日志
    ProcessInstanceHistoryLogQuery createProcessInstanceHistoryLogQuery(String processInstanceId);

    /***** 创建原生查询，可以通过传入SQL进行检索 *****/
    // 创建通过SQL直接查询历史流程实例的对象
    NativeHistoricProcessInstanceQuery createNativeHistoricProcessInstanceQuery();

    // 创建通过SQL直接查询历史活动实例的对象
    NativeHistoricActivityInstanceQuery createNativeHistoricActivityInstanceQuery();

    // 创建通过SQL直接查询历史任务实例的对象
    NativeHistoricTaskInstanceQuery createNativeHistoricTaskInstanceQuery();

    // 创建通过SQL直接查询历史详情的对象
    NativeHistoricDetailQuery createNativeHistoricDetailQuery();

    // 创建通过SQL直接查询历史变量的对象
    NativeHistoricVariableInstanceQuery createNativeHistoricVariableInstanceQuery();

    // 删除历史任务实例
    void deleteHistoricTaskInstance(String taskId);

    // 删除历史流程实例
    void deleteHistoricProcessInstance(String processInstanceId);

    // 查询与任务实例关联的用户
    List<HistoricIdentityLink> getHistoricIdentityLinksForTask(String taskId);

    // 查询与流程实例关联的用户
    List<HistoricIdentityLink> getHistoricIdentityLinksForProcessInstance(String processInstanceId);
}
```

`HistoryService` 接口主要提供以下4类方法：
+ 创建历史流程实例、历史活动实例、历史任务实例、历史详情、历史变量等对象的查询；
+ 创建历史流程实例、历史活动实例、历史任务实例、历史详情、历史变量等对象的原生查询，支持传入SQL；
+ 删除历史流程实例、历史任务实例；
+ 查询与历史流程实例、任务实例关联的用户。