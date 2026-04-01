# Sa-token
{docsify-updated}

## 获取token值

<center><img src="pics/sa-token.png" alt=""></center>

## 生成的token/session key 名
`StpLogic`  类的这些方法：
<center><img src="pics/sa-token-2.png" alt=""></center>

## Session 会话
在 Sa-Token 中，Session 分为三种，分别是：

+ User-Session: 指的是框架为每个 账号id 分配的 Session
+ Token-Session: 指的是框架为每个 token 分配的 Session
+ Custom-Session: 指的是以一个 特定的值 作为SessionId，来分配的 Session

StpLogic -> distUsableToken 顶下线功能实现


Sa-token 登录成功后，会在 Redis 生成三个 key：
1. 以长 token 和 token 名字以及一些特定规则生成的 key
2. session 的 key
3. token session 的 key
这三个 key 的有效期与 sa-token 设置的有效期时间是一致的

+ 长token： "demo-international-app-token:login:token:d663bd6c-0717-4485-ad6f-4a63032de2fb"，保存的是登录的 loginId(uid)
+ session: demo-international-app-token:login:session:74627", 保存对应的 token（长 token）和 device：
  ```
  {
    "@class": "cn.dev33.satoken.dao.SaSessionForJacksonCustomized",
    "id": "demo-international-app-token:login:session:74627",
    "createTime": 1749192871405,
    "dataMap": { "@class": "java.util.concurrent.ConcurrentHashMap" },
    "tokenSignList": [
        "java.util.Vector",
        [
            {
                "@class": "cn.dev33.satoken.session.TokenSign",
                "value": "d663bd6c-0717-4485-ad6f-4a63032de2fb",
                "device": "APP"
            }
        ]
    ]
  }
  ```
+ token session: "demo-international-app-token:login:token-session:d663bd6c-0717-4485-ad6f-4a63032de2fb", 自定义保存数据
```
  {
    "@class": "cn.dev33.satoken.dao.SaSessionForJacksonCustomized",
    "id": "demo-international-app-token:login:token-session:d663bd6c-0717-4485-ad6f-4a63032de2fb",
    "createTime": 1749192871420,
    "dataMap": {
        "@class": "java.util.concurrent.ConcurrentHashMap",
        "login_type": ["com.demo.gjyw.business.login.event.LoginType", "TRADE"],
        "dev-id": "259b06f413243fb2081a801b28f3d70b6a1f7acf4fd1b10dd629bf64f3204f78",
        "short_token": "S-c988a151-11af-44a5-af4b-34086007f471"
        },
        "tokenSignList": ["java.util.Vector", []]
    }
  ```
+ 短 token: "demo-international-app-token:short-token:S-b0f3e72d-d8c0-41bf-8c24-298ff06927f1"，自定义保存的内容，保存 loginId

## 注解处理
+ `SaInterceptor` - 推荐
+ `SaAnnotationInterceptor` - Deprecated