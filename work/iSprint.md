
## Token/session
1. User Session
2. Dynamic Agent Session -client-authenticated SSL，不会闲置超时和过期
Accessing REST API over HTTPS URL using mutual/client-authenticated SSL using agent/client certificate,
or setting simulated agent certificate property in amsystem.properties (this is a quick workaround only for convenience of testing in non-production environments; it should NOT be used in production environments)

"dynamicAgentSession": "true",

3. Access Token - You can generate an access token for the non-interactive user in the Access Token tab of that user's configurations page in the Admin Console.
"Authorization: Bearer " request header is set

Dynamic Agent Session 是如何设置的？我看测试环境用的是这种方式 ？ 到时候生产同样嘛？


## 绑定
1. `update/user/create`  
   1. 只需要 jmeter 中的参数嘛？ assignRealm 什么意思？

2. `/OATH/create-assign-and-encrypt`
   1. clientPublicKey/clientPublicKeyId 等参数如何生成 ？app SDK ？
   2. batchId 有什么用 ？ userStoreId 是什么，API 文档里边没有 ？

3. `/token/activateByOTP`
   1. serialNumber 是 SDK 生成的嘛 ？
   2. otp 是发送的 activationCode ?

三步中间某一步中断的话，如何继续绑定流程 ？

## pin 码验证
1. SDK 验证即可

## 设备查询
4. 查询客户绑定的设备 `device/listDevicesOfUser` ？

## 重置 pin
5. 生物认证解锁需要调用 UAS API 嘛？ 
6. 重置 pin 是用的哪（几）个 API ？ `password/reset` ?



那客户端是不是 调用 activationTokenEx 之后，就能使用 pin 认证了，即使 est/token/activateByOTP  没有成功 ？因为之前说 pin 认证是纯SDK 操作的

是可以，但是服务器令牌的状态不是Activated的话，服务器验证不了OTP(生成的OTP验证码)


/token/activateByOTP 这步是可选的，在create-assign-and-encrypt时，这个参数改成status: Activation，那么服务器令牌就是激活状态


## 问题
如何使客户端的iSprint 失效 ？ 如果 pin 认证不在后端验证
@yik(i-Sprint)   @Tony  如果pin 验证直接在 前段时间SDK完成，不用跟后端交互。那是不是没有办法在后端控制一台设备禁止 pin 登录了啊 ？

如果pin认证是纯前端的，登录时如何让后端信任pin 已经认证成功并颁发交易 token ?



## 接口逻辑
### 绑定
SDK preTokenActivation ----> /cap/bind/apply 参数就是 key/token
                        ----> 用户中心校验 token ？
                        ---> UAS create
                        ---> UAS assign-and-encrypt

                        ---> 发送 activationCode(SMS), 以及 result 给 SDK


SDK enablePin/activateToken 成功（要求输入ping 和 activationCode）
                        ----> /cap/bind/success token or 交易账号？
                        ---> 用户中心校验 token ？
                        ---> 调用OMS同步 CAP 方式到 clientAuth
                        ---> 通知用户中心将之前的信任设备全部删除。
                        ---> 保存用户校验方式入库

### 登录
登录 ----> 输入账号密码 ---> 返回iSprint 方式 ----> app 要求用户输入 ping 完成 ----> 去NRTS拿 tradeToken？ （后端如何信任？如何保证安全性问题？）

### 重置 pin 是手機本地處理, 不用連到 UAS
生物认证解锁 可調用 SDK 的 loginFingerprint (Android) / loginTouchIDOrFaceID (iOS), 需要先啟用生物認證
重置 pin 流程: 如果已啟動生物認證, 可以在生物認證登錄後再調用 enablePIN
參考: 用戶按 app 的 [重置 pin] > 要求輸入新 PIN > 要求生物認證登錄 > disablePIN > enablePIN
(如果沒有啟動生物認證, 不能重置 pin。只能註銷現有令牌, 重新激活新令牌)


### 设备列表
新版本 app ： 如果使用了 iSprint，是否展示老的信任设备嘛 ？？没使用iSprint 呢？
老版本 app ： 如果使用了 iSprint ---> 强制升级。
             没使用维持原样 ？还是能看到iSprint 的设备 ？

老版本登录后，如果启用过 iSprint 也只能看到 iSprint 的设备。

## oms API
https://pingcode.demo.com.cn/wiki/spaces/ADC/pages/ElK-ApNY
