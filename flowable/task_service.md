# 任务服务API-TaskService
{docsify-updated}

`TaskService` 接口主要用于操作正在运行的流程任务，主要提供以下6类方法：
+ 任务实例的创建、保存、查询、删除等，主要操作运行时任务表 `ACT_RU_TASK` 
+ 任务权限相关操作，主要指任务和人员之间的关系管理，可以设置办理人、候选人、候选组，以及其他类型的关系，主要操作运行时身份关系表`ACT_RU_IDENTITYLINK`
+ 任务办理相关操作，包括认领、委托、办理等；
+ 变量管理相关操作，包括变量的新增、删除、查询等，主要操作运行时变量表 `ACT_RU_ VARIABLE`
+ 任务评论管理相关操作，包括任务评论的新增、删除、查询等，主要操作评论表 `ACT_HI_COMMENT`
+ 任务附件管理相关操作，包括任务附件的新增、删除、查询等，主要操作附件表 `ACT_HI_ ATTACHMENT`

## 待办任务查询
获取用户的待办任务是Flowable常用应用场景之一。待办任务存储在 `ACT_RU_TASK` 表中。 `TaskService` 接口提供查询 `ACT_RU_TASK` 表的方法，因此可以通过 `TaskService` 接口获取用户的待办任务。 `TaskService` 接口中提供了2种查询任务实例的方法：
+ 一种是通过创建 `TaskQuery` 对象传入参数查询
+ 另一种是通过创建 `NativeTaskQuery` 对象传入SQL查询。
 
TaskService接口相关代码如下：
```
public interface TaskService {
    // 通过创建TaskQuery对象传入参数查询
    TaskQuery createTaskQuery();

    // 通过创建NativeTaskQuery对象传入SQL查询
    NativeTaskQuery createNativeTaskQuery();
}
```

## 任务办理及权限控制
用户任务需要经过人工办理后，才能触发流程的继续流转。一般在实际应用中，都需要对任务办理人权限进行控制。某些任务可以指定由某人专门进行办理；某些任务可以指定仅有某几人可以办理；某些任务，任务办理人还可以委托他人进行办理。

`TaskService` 接口提供了多种任务办理方法，以及人员与任务关系管理方法，上层应用可以根据实际情况选用一种方法。 `TaskService` 接口相关代码如下：
```
public interface TaskService {
    // 任务办理相关设置
    void claim(String taskId, String userId);
    void unclaim(String taskId);
    void delegateTask(String taskId, String userId);
    void resolveTask(String taskId);
    void resolveTask(String taskId, Map<String, Object> variables);
    void resolveTask(String taskId, Map<String, Object> variables, Map<String, Object> transientVariables);
    void complete(String taskId);
    void complete(String taskId, Map<String, Object> variables);
    void complete(String taskId, Map<String, Object> variables, Map<String, Object> transientVariables);
    void complete(String taskId, Map<String, Object> variables, boolean localScope);
    
    // 人员与任务关系管理
    void setAssignee(String taskId, String userId);
    void setOwner(String taskId, String userId);
    void addCandidateUser(String taskId, String userId);
    void addCandidateGroup(String taskId, String groupId);
    void addUserIdentityLink(String taskId, String userId, String identityLinkType);
    void addGroupIdentityLink(String taskId, String groupId, String identityLinkType);
    void deleteCandidateUser(String taskId, String userId);
    void deleteCandidateGroup(String taskId, String groupId);
    void deleteUserIdentityLink(String taskId, String userId, String identityLinkType);
    void deleteGroupIdentityLink(String taskId, String groupId, String identityLinkType);
    List<IdentityLink> getIdentityLinksForTask(String taskId);
}
```

## 评论和附件管理
Flowable可以针对流程实例或任务实例添加评论和附件。评论数据存储在 `ACT_HI_COMMENT` 表中，附件数据存储在 `ACT_HI_ATTACHMENT` 和`ACT_GE_BYTEARRAY` 表中。评论和附件相关的接口都由 `TaskService` 提供。核心接口如下所示：
```
public interface TaskService {
    // 为流程任务实例或流程实例添加评论
    Comment addComment(String taskId, String processInstanceId, String message);

    // 为流程任务实例或流程实例添加特定类型的评论
    Comment addComment(String taskId, String processInstanceId, String type, String message);

    // 根据评论ID更新评论，如果ID不存在，则抛出异常
    void saveComment(Comment comment);

    // 根据评论ID查询评论，如果ID不存在，则返回null
    Comment getComment(String commentId);

    // 根据任务ID或流程实例ID删除评论
    void deleteComments(String taskId, String processInstanceId);

    // 根据评论ID删除评论
    void deleteComment(String commentId);

    // 根据任务ID查询所有评论
    List<Comment> getTaskComments(String taskId);

    // 根据任务ID和类型查询所有评论
    List<Comment> getTaskComments(String taskId, String type);

    // 查询某一类型的所有评论
    List<Comment> getCommentsByType(String type);

    // 根据流程实例ID查询评论
    List<Comment> getProcessInstanceComments(String processInstanceId);

    // 根据流程实例ID和类型查询评论
    List<Comment> getProcessInstanceComments(String processInstanceId, String type);

    // 创建附件
    Attachment createAttachment(String attachmentType, String taskId, String processInstanceId,
                    String attachmentName, String attachmentDescription, InputStream content);

    // 根据URL创建附件
    Attachment createAttachment(String attachmentType, String taskId, String processInstanceId,
                    String attachmentName, String attachmentDescription, String url);

    // 根据附件ID更新附件
    void saveAttachment(Attachment attachment);

    // 根据附件ID查询附件（不包含附件的具体内容）
    Attachment getAttachment(String attachmentId);

    // 查询附件内容，并将其转换为输入流
    InputStream getAttachmentContent(String attachmentId);

    // 查询任务ID对应的附件列表
    List<Attachment> getTaskAttachments(String taskId);
}
```