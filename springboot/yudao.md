# yudao
{docsify-updated}

## security
AuthorizationFilter
RequestMatcherDelegatingAuthorizationManager
AuthorizeHttpRequestsConfigurer

## RPC
为什么 rpc-api 的请求不用 token ？
```
cn.com.demo.module.system.framework.security.config.SecurityConfiguration
cn.com.demo.module.infra.framework.security.config.SecurityConfiguration
```

## mail
MailSendConsumer
MailSendService
MailTemplateService system_mail_template
MailAccountDO system_mail_account


## RUNME
后端：
1. 安转mysql/redis
2. 执行 sql/mysql/init.sql
3. 启动 src/main/java/cn.com.demo/server/BlockChainManagementServerApplication.java
4. 业务逻辑写在 distribution-management-biz 中

使用 maven 或 mvnd 编译时：`mvn package -Dmaven.test.skip=true`

前端：
修改 src/config/axios/service.ts 第 117 行为 `const code = parseInt(data.code) || result_code`

npm i
npm run dev


## 一般开发流程
1. 增加菜单，配置前端跳转路由/组件
2. 前端开发组件：src/views/system/dict1/index.vue ，对接后端接口

## 单体应用打包
修改 yudao-module-system-server 和 yudao-module-infra-server 模块的 pom.xml 文件，将 spring-boot-maven-plugin 配置成 skip。
```asciidoc
<build>
    <!-- 设置构建的 jar 包名 -->
    <finalName>${project.artifactId}</finalName>
    <plugins>
        <!-- 打包 -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal> <!-- 将引入的 jar 打入其中 -->
                    </goals>
                </execution>
            </executions>
            <!-- 注意！！！重点是下面 3 行！！！ -->
            <configuration>
                <skip>true</skip>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 系统数据库表
system_login_log 登录日志表
system_mail_account 邮箱账号表
system_mail_log 邮件记录表
system_mail_template 邮件模版

system_users 系统用户表
system_menu 系统菜单
system_role 角色表
system_role_menu 角色菜单关联表
system_user_role 用户角色表

system_notice 系统公告
system_notify_message 系统通知消息
system_notify_template 系统通知模板

system_operate_log 系统操作日志记录

system_tenant 租户表

## 配置
禁用租户功能，在 application.yaml -- `TenantProperties`
```aiignore
distribution-management:
    tenant: # 多租户相关配置项
        enable: false
```

## DAL
BaseMapperX

## 全局异常处理
GlobalExceptionHandler

## 登录注册相关
curl 'http://localhost:48080/admin-api/system/auth/get-permission-info' \
-H 'Content-type: application/json' \
-H 'Accept: application/json, text/plain, */*' \
-H 'Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7' \
-H 'Authorization: Bearer 19dc2a753f0447169b8c20ffbd949a59' 

{
"id" : 5013,
"parentId" : 0,
"name" : "稳定币issue",
"path" : "/dict1",
"component" : "system/dict1/index",
"componentName" : "",
"icon" : "ep:aim",
"visible" : true,
"keepAlive" : true,
"alwaysShow" : true
}

TokenAuthenticationFilter
AuthController

oauth2_access_token:de4330a6f9224108a259741fcdbc9230   authorization header


system_oauth2_client 表中保存了token 的有效期
system_oauth2_refresh_token 表中保存了 refresh token
system_oauth2_access_token 表中存放 access token

/refresh-token 刷新token 令牌

AuthorizationFilter
UserController

AuthorizationManagerBeforeMethodInterceptor
SecurityFrameworkServiceImpl 检查权限
    LocalCache ---》 缓存权限

## 权限校验
```asciidoc
@PostMapping("/create")
@Operation(summary = "新增用户")
@PreAuthorize("@ss.hasPermission('system:user:create')")
public CommonResult<Long> createUser(@Valid @RequestBody UserSaveReqVO reqVO) {
    Long id = userService.createUser(reqVO);
    return success(id);
}
```

## 通知信息查询
system_notify_message

## 权限
平台管理  ACCESS_CONTROLLER
项目管理  ACCESS_CONTROLLER
MINTER
BURNER
PAUSER
RESUME

代理管理   TRANSFTER
用户管理   BLACKLISTER
WHITELISTER
FREEZER

授权签名管理 ALL


## 交易流程
客户购买 ：subscribe(客户申请认购) --> mint（铸币） --->  issue(通过 cross_to 私链向公链转稳定币)
客户赎回 ：redeem(通过cross_to 公链向私链转稳定币) ---> redemption(客户申请赎回，通过后直接创建 burn 记录)---> burn(burner 查看记录，点击burn 发起多签，多签完成，销毁私链的稳定币)

## 框架常用方法
return success(AuthConvert.INSTANCE.convert(user, Collections.emptyList(), Collections.emptyList()));
SecurityFrameworkUtils.setLoginUser(loginUser, request); //设置登录用户信息
ServletUtils.getClientIP() // 获取请求IP


## curl
1. 发起一笔 redeem
curl -XPOST 'http://localhost:48080/cross-chain/crossTo/redeem' \
-H 'Content-type: application/json' \
-H 'Accept: application/json, text/plain, */*' \
-H 'Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7' \
-H 'Authorization: Bearer e1a1b20b7fb449618322a234b9a38368' \
--data-raw '{"crossToId":"1","amount":1}'

2. 查询redeem 记录
curl -XPOST 'http://localhost:48080/cross-chain/crossTo/redeem/list' \
-H 'Content-type: application/json' \
-H 'Accept: application/json, text/plain, */*' \
-H 'Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7' \
-H 'Authorization: Bearer e1a1b20b7fb449618322a234b9a38368' \
--data-raw '{"pageNo":1,"pageSize":10}'

3. 发送消息
curl 'http://localhost:48080/admin-api/system/notify-template/send-notify' \
-H 'Content-type: application/json' \
-H 'Accept: application/json, text/plain, */*' \
-H 'Authorization: Bearer 87a9014cb00e4591ad809bf4276ee558' \
--data-raw '{"content":"您有一个redeem 需要您签名，点击{}进行签名","params":[""],"mobile":"","templateCode":"2","templateParams":{"":"www"},"userType":2,"userId":"1"}'

4. 新增赎回申请
curl 'http://localhost:48080/cross-chain/callable/add' \
-H 'Content-type: application/json' \
-H 'Accept: application/json, text/plain, */*' \
-H 'Authorization: Bearer e660104733364126bf81837cb6750421' \
--data-raw '{
   "projectId": 0,
   "userAddress": "userAddress_c721db36fb6a",
   "amount": 0.00,
   "proofDocs": [{}]
   }'

调用合约时，
环境要做修改：
1. chain-client-config/sdk_config_pk_admin1.yml 配置文件中，user_sign_key_file_path 改为 ：【自己项目路径】chain-client-config/config/crypto-config/node1/admin/admin1/admin1.key
2. distribution-management-server/start/src/main/resources/chain-config.properties 配置文件中，project.base.dir改为自己项目路径

合约要注意：
1. 调用人grantRole
2. 账户地址是加入白名单的

## 启动时的 VM 参数
```aiignore
--add-opens java.base/java.lang=ALL-UNNAMED 
--add-opens java.base/java.lang.reflect=ALL-UNNAMED 
--add-opens java.base/java.util=ALL-UNNAMED 
--add-opens java.base/java.security=ALL-UNNAMED
```

## 以太坊初始化配置
EthereumConfig


## 一键改包
`src/main/java/cn/com/demo/server/ProjectReactor.java`