# 管理服务API-ManagementService
{docsify-updated}

管理服务接口 `ManagementService` 主要提供数据表和表元数据的管理和查询服务。另外，它还提供了查询和管理异步任务的功能。Flowable异步任务应用很广，如定时器、异步操作、延迟暂停、激活等。此外，它还可以通过 `ManagementService` 接口直接执行某个CMD命令。

## 数据库管理
`ManagementService` 接口提供了一系列数据库管理方法，包括数据表与表元数据的查询方法、数据库更新方法等：
```
public interface ManagementService {
    // 获取Flowable数据表名称和数据量
    Map<String, Long> getTableCount();

    // 获取实体对应的表名
    String getTableName(Class<?> entityClass);

    // 查询指定表的元数据
    TableMetaData getTableMetaData(String tableName);

    // 获取表的分页查询对象
    TablePageQuery createTablePageQuery();

    // 更新数据库schema
    String databaseSchemaUpgrade(Connection connection, String catalog, String schema);
}
```

## 异步任务管理
Flowable中的定时器中间事件、定时器边界事件等都会产生异步任务。Flowable提供了6个用于存储异步任务的表。运行时异步任务存储在 `ACT_RU_JOB` 表中，定时器任务存储在 `ACT_RU_TIMER_JOB` 表中，定时中断任务存储在 `ACT_RU_SUSPENDED_JOB` 表中，不可执行任务存储在 `ACT_RU_DEADLETTER_JOB` 表中，异步历史任务存储在 `ACT_RU_HISTORY_JOB` 表中，外部任务存储在 `ACT_RU_EXTERNAL_JOB` 表中。

以定时器中间事件为例，异步任务产生后会被写入 `ACT_RU_TIMER_JOB` 表。任务达到触发条件后，会写入 `ACT_RU_JOB` 表中执行。如果该任务执行异常，可以重新执行。当重试次数大于设定的最大重试次数时，任务会被写入 `ACT_RU_DEADLETTER_JOB` 表。如果在任务等待过程中调用了中断流程实例的方法，异步任务将会被写入 `ACT_RU_SUSPENDED_JOB` 表。

`ManagementService` 接口中关于异步任务管理的代码片段如下：
```
public interface ManagementService {
    // 获取异步任务查询对象
    JobQuery createJobQuery();

    // 获取外部工作任务查询对象
    ExternalWorkerJobQuery createExternalWorkerJobQuery();

    // 获取定时器任务查询对象
    TimerJobQuery createTimerJobQuery();

    // 获取挂起任务查询对象
    SuspendedJobQuery createSuspendedJobQuery();

    // 获取不可执行任务查询对象
    DeadLetterJobQuery createDeadLetterJobQuery();

    // 获取异步历史任务查询对象
    HistoryJobQuery createHistoryJobQuery();

    // 获取关联的ID查询job
    Job findJobByCorrelationId(String jobCorrelationId);

    // 强制执行任务
    void executeJob(String jobId);

    // 强制执行历史任务
    void executeHistoryJob(String historyJobId);

    // 查询历史任务JSON格式数据
    String getHistoryJobHistoryJson(String historyJobId);

    // 将定时器任务移入可执行任务
    Job moveTimerToExecutableJob(String jobId);

    // 将任务移入不可执行任务
    Job moveJobToDeadLetterJob(String jobId);

    // 将不可执行任务重新加入可执行任务
    Job moveDeadLetterJobToExecutableJob(String jobId, int retries);

    // 将不可执行任务加入历史任务
    HistoryJob moveDeadLetterJobToHistoryJob(String jobId, int retries);

    // 批量将不可执行任务加入可执行任务列表或历史任务列表
    void bulkMoveDeadLetterJobs(Collection<String> jobIds, int retries);

    // 批量将不可执行任务加入可执行历史任务列表
    void bulkMoveDeadLetterJobsToHistoryJobs(Collection<String> jobIds, int retries);

    // 将挂起任务重新加入可执行任务
    Job moveSuspendedJobToExecutableJob(String jobId);

    // 删除异步任务
    void deleteJob(String jobId);

    // 删除定时器任务
    void deleteTimerJob(String jobId);

    // 删除挂起的任务
    void deleteSuspendedJob(String jobId);

    // 删除不可执行的任务
    void deleteDeadLetterJob(String jobId);

    // 删除外部工作任务
    void deleteExternalWorkerJob(String jobId);

    // 删除历史任务
    void deleteHistoryJob(String jobId);

    // 设置异步任务剩余重试次数
    void setJobRetries(String jobId, int retries);

    // 设置定时器任务剩余重试次数
    void setTimerJobRetries(String jobId, int retries);

    // 重新设置定时器任务的执行日期
    Job rescheduleTimeDateJob(String jobId, String timeDate);

    // 通过时间段重新设置定时器任务的执行日期
    Job rescheduleTimeDurationJob(String jobId, String timeDuration);

    // 重新设置定时器的循环执行信息
    Job rescheduleTimeCycleJob(String jobId, String timeCycle);

    // 上述三个方式的合并，不过一次只能使用其中一种方式
    Job rescheduleTimerJob(String jobId, String timeDate, String timeDuration, String timeCycle, String endDate,
                    String calendarName);

    // 获取异步任务异常栈
    String getJobExceptionStacktrace(String jobId);

    // 获取定时器任务异常栈
    String getTimerJobExceptionStacktrace(String jobId);

    // 获取挂起任务异常栈
    String getSuspendedJobExceptionStacktrace(String jobId);

    // 获取不可执行的任务异常栈
    String getDeadLetterJobExceptionStacktrace(String jobId);

    // 获取外部工作任务的详细异常信息
    String getExternalWorkerJobErrorDetails(String jobId);
}
```