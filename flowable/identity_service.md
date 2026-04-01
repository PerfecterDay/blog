# 身份服务API-IdentityService
{docsify-updated}

Flowable使用 `ACT_ID_GROUP` 、 `ACT_ID_USER` 、 `ACT_ID_INFO` 和 `ACT_ID_MEMBERSHIP` 这4张表维护人员组织架构。身份服务接口`IdentityService` 负责提供用户、组和人员组织关系管理等服务。

`IdentityService` 接口核心代码如下：
```
~public interface IdentityService {
    // 新建用户
    User newUser(String userId);

    // 保存用户
    void saveUser(User user);

    // 更新密码
    void updateUserPassword(User user);

    // 创建用户查询
    UserQuery createUserQuery();

    // 创建本地SQL查询
    NativeUserQuery createNativeUserQuery();

    // 删除用户
    void deleteUser(String userId);

    // 检查指定密码与用户密码是否一致
    boolean checkPassword(String userId, String password);

    // 设置当前环境下的认证用户ID
    void setAuthenticatedUserId(String authenticatedUserId);

    // 设置用户图片
    void setUserPicture(String userId, Picture picture);

    // 获取用户图片
    Picture getUserPicture(String userId);

    // 新建组
    Group newGroup(String groupId);

    // 创建组查询对象
    GroupQuery createGroupQuery();

    // 创建本地组SQL查询
    NativeGroupQuery createNativeGroupQuery();

    // 查询指定流程定义ID上配置的发起小组
    List<Group> getPotentialStarterGroups(String processDefinitionId);

    // 查询指定流程定义ID上配置的发起用户
    List<User> getPotentialStarterUsers(String processDefinitionId);

    // 保存组
    void saveGroup(Group group);

    // 删除组
    void deleteGroup(String groupId);

    // 创建用户与组的关系
    void createMembership(String userId, String groupId);

    // 删除用户与组的关系
    void deleteMembership(String userId, String groupId);

    // 设置用户扩展信息
    void setUserInfo(String userId, String key, String value);

    // 查询用户指定key的扩展信息
    String getUserInfo(String userId, String key);

    // 查询用户所有扩展信息的key
    List<String> getUserInfoKeys(String userId);

    // 删除用户某个key的扩展信息
    void deleteUserInfo(String userId, String key);
}
```
`IdentityService` 接口主要提供以下4类方法。
+ 与用户管理相关的方法：包括新建用户、更新用户、查询用户、删除用户等。
+ 与组管理相关的方法：包括新建组、保存组、查询组、删除组等。
+ 维护用户与组的关系的方法：包括新建关系、删除关系等。
+ 管理用户扩展信息的方法：包括新增扩展信息、删除扩展信息及查询扩展信息。

## IdmEngine 配置
Flowable默认提供了一套用于身份和权限管理的身份管理引擎IdmEngine，提供用户和用户组及相应权限的创建、修改、删除等功能。在Flowable 6中身份管理引擎IdmEngine被从工作流引擎中拆分出来作为独立的组件，包含 `flowable-idm-api` 、 `flowable-idm-engine` 、 `flowable-idm-spring` 和`flowable-idm-engine-configurator` 等模块。因此要使用身份管理引擎IdmEngine，需要在项目的pom.xml文件中加入以下JAR依赖：
```
org.flowable:flowable-idm-engine:${version}
org.flowable:flowable-idm-api:${version}
org.flowable:flowable-idm-engine-configurator:${version}
org.flowable:flowable-idm-spring:${version}
```

1. 定义身份管理引擎配置 `idmEngineConfiguration` ，它对应的身份管理引擎配置类为 `org.flowable.idm. engine.IdmEngineConfiguration` 。它的 `dataSource` 属性用于设置数据源， `databaseSchemaUpdate` 属性用于设置数据库更新策略。 `IdmEngine` 不允许用户密码为明文，需要加密，必须配置用户密码编码器 `passwordEncoder` 和密码加密盐 `passwordSalt` 。用户密码编码器需要实现 `org.flowable.idm.api.PasswordEncoder` 接口，密码加密盐需要实现 `org.flowable.idm.api.PasswordSalt` 接口。如果在Spring环境下，可以使用 `IdmEngineConfiguration` 的子类 `org.flowable.idm.spring.SpringIdmEngineConfiguration` 来配置，从而实现与Spring的集成，需要通过Spring管理事务时建议采用这种方式。
2. 定义身份管理引擎配置器 `idmEngineConfigurator` ，通过它的 `idmEngineConfiguration` 属性指定了前面定义的身份管理引擎配置。
3. 定义工作流引擎配置 `processEngineConfiguration` ，通过配置 `disableIdmEngine` 为 `false` 启用身份管理引擎，通过 `idmEngineConfigurator` 属性指定前面定义的身份管理引擎配置器，从而实现了身份管理引擎与工作流引擎的集成。

## 用户管理