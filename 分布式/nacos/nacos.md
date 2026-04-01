# Nacos 简介

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

## 安装运行
直接从[官方](https://github.com/alibaba/nacos/releases)下载最新安装包后，切换到安装目录下直接运行：
`startup.sh -m standalone`
关闭：
`shutdown.sh`

服务启动后，可以通过下边链接进入管理后台：http://127.0.0.1:8848/nacos

### 验证Nacos服务是否启动成功
进入${nacos.home}/logs/ 目录下， 使用tail -f start.out 查看日志，如果看到如下日志，说明服务启动成功。
```
Nacos started successfully in stand alone mode. use embedded storage
```
可以通过下列服务，快速检验Nacos的功能。

## Rest API
1. 服务注册 : `curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'`
2. 查看服务列表： `curl -X GET 'http://localhost:8848/nacos/v1/ns/catalog/services?pageNo=1&pageSize=10&namespaceId=http-application'`
3. 服务发现 : `curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'`
4. 发布配置 : `curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=HelloWorld"`
5. 获取配置 : `curl -X GET "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"`


## 开启鉴权
> https://nacos.io/docs/latest/manual/admin/auth/?spm=5238cd80.2ef5001f.0.0.3f613b7chivyRa

编辑 `nacos/conf/application.properties` 的配置文件：
```
###
nacos.core.auth.default.token.secret.key=$custom_base64_token_secret_key //自定义一个base64 编码格式的token

例如： nacos.core.auth.plugin.nacos.token.secret.key=ZGFzZGFzZmRzYWZkc2dkZmdkZmhmZHNmc2Zkc2Rhc2Rhc2Rhc2Rhcw==

### 配置自定义身份识别的key（不可为空）和value（不可为空）
nacos.core.auth.server.identity.key=$custom_server_identity_key
nacos.core.auth.server.identity.value=$custom_server_identity_value

比如：
nacos.core.auth.server.identity.key=nacos
nacos.core.auth.server.identity.value=nacos

###
nacos.core.auth.system.type=nacos
nacos.core.auth.enabled=true
```


修改完配置后，可以在admin 控制台界面修改密码（首次登陆会让你设置密码）；或者通过 API curl 设置：
```
curl -X POST 'http://127.0.01:8848/nacos/v1/auth/users/admin' -d 'password=123456'
{"username":"nacos","password":"123456"}% 
```

使用账号密码登录：
```
curl 'http://10.5.80.113:8848/nacos/v1/auth/users/login' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-raw 'username=nacos&password=123456'
```


curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080&healthy=false&namespaceId=44a78092-158a-4cf7-b772-e22c03c02109' \
-H 'Accept: application/json, text/javascript, */*; q=0.01' \
-H 'Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7' \
-H 'AccessToken: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3ODY3M30.bC46VFyT7OHBa5pVWKOY9dM0SwDEEeSoNRpXL0OKvE4' \
-H 'Authorization: {"accessToken":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3ODY3M30.bC46VFyT7OHBa5pVWKOY9dM0SwDEEeSoNRpXL0OKvE4","tokenTtl":18000,"globalAdmin":true,"username":"nacos"}'


curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName' \
-H 'Accept: application/json, text/javascript, */*; q=0.01' \
-H 'Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7' \
-H 'AccessToken: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3ODY3M30.bC46VFyT7OHBa5pVWKOY9dM0SwDEEeSoNRpXL0OKvE4' \
-H 'Authorization: {"accessToken":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3ODY3M30.bC46VFyT7OHBa5pVWKOY9dM0SwDEEeSoNRpXL0OKvE4","tokenTtl":18000,"globalAdmin":true,"username":"nacos"}'


curl -X GET 'http://127.0.0.1:8848/nacos/v2/console/namespace/list?serviceName=nacos.naming.serviceName' \
-H 'Accept: application/json, text/javascript, */*; q=0.01' \
-H 'Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7' \
-H 'AccessToken: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3ODY3M30.bC46VFyT7OHBa5pVWKOY9dM0SwDEEeSoNRpXL0OKvE4' \
-H 'Authorization: {"accessToken":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3ODY3M30.bC46VFyT7OHBa5pVWKOY9dM0SwDEEeSoNRpXL0OKvE4","tokenTtl":18000,"globalAdmin":true,"username":"nacos"}'

curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?accessToken=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTczOTg3ODY3M30.bC46VFyT7OHBa5pVWKOY9dM0SwDEEeSoNRpXL0OKvE4&dataId=TEST_CONFIG&group=TEST_GROUP&content=useLocalCache=true"