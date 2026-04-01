# Springboot 集成 TOTP 和 DUO
{docsify-updated}

## TOTP
> https://dev.to/acetheninja/learn-by-building-what-is-totp-building-a-simple-totp-authentication-server-with-nodejs-52j9

### TOTP 原理
TOTP包含两个关键组件用于生成一次性密码（OTP）。
1. 种子密钥 Seed: 这是客户端与服务器之间共享的静态密钥。
2. 动态因子。该组件在每次请求新OTP或固定时间间隔内发生变化。TOTP的动态因子采用Unix时间戳。

TOTP算法采用对称密钥加密形式，因为客户端和服务器使用相同的密钥独立生成OTP。

TOTP认证流程如下：
1. 初始化：首次设置TOTP时，服务器会为每位用户生成一个 Seed 。该密钥通过安全方式传输至用户的TOTP生成器，通常将其编码为二维码，用户使用TOTP应用扫描后即可获取。
<center><img src="pics/totp.webp" alt=""></center>

2. 生成OTP：TOTP生成器（用户应用程序）将当前时间除以时间步长，计算自基准时间（通常为Unix纪元时间，即1970年1月1日）以来的时间步数。随后采用算法（通常为`HMAC-SHA-1`），将Seed值与时间步长计数作为输入进行组合，生成哈希值。最后提取该哈希值的部分内容，转换为6至8位数字代码，即为TOTP验证码。
3. 验证：用户在有效期内将生成器应用显示的TOTP输入至在线服务。同时，服务器凭借共享密钥和当前时间，通过相同流程生成预期TOTP。随后服务器将用户输入的TOTP与自身生成的预期TOTP进行比对。

### springboot 集成
> https://github.com/wstrange/GoogleAuth

```
<dependency>
    <groupId>com.warrenstrange</groupId>
    <artifactId>googleauth</artifactId>
    <version>1.5.0</version>
</dependency>
```

1. 生成TOTP的密钥  
```
GoogleAuthenticator gAuth = new GoogleAuthenticator();
final GoogleAuthenticatorKey secretKey = gAuth.createCredentials();
secretKey.getKey()
```

2. 生成 TOTP
```
GoogleAuthenticator gAuth = new GoogleAuthenticator();
int code = gAuth.getTotpPassword(secretKey);
```

3. 验证 TOTP  
```
GoogleAuthenticator gAuth = new GoogleAuthenticator();
boolean isCodeValid = gAuth.authorize(secretKey, password);
```


## DUO集成
DUO验证成功后跳转的URL：http://localhost:50000/duo-callback?state=f98e6b1d9c20bd6f844786dce9155206112b&duo_code=ep4q7ZSuH3DuwdUGoZLGv5qb4VYKhYKp

首先，注册一个 duo admin panel 的账号并登陆，然后创建一个 web sdk 类型的 application :
<center><img src="pics/duo.png" alt=""></center>

第二步，集成DUO 的SDK：
```
<!-- https://mvnrepository.com/artifact/com.duosecurity/duo-universal-sdk -->
<dependency>
    <groupId>com.duosecurity</groupId>
    <artifactId>duo-universal-sdk</artifactId>
    <version>1.3.1</version>
</dependency>
```

配置上第一步分配好的信息：
```
duo.clientId=DIQBHSIDUCBNAI7Y05A
duo.clientSecret=La6EzN8IUJHJNSIJNJNs8878sjhdsa9kaF0SDq0BRZDNTvP
duo.api.host=api-h8yshal.duosecurity.com
duo.redirect.uri=http://localhost:50000
```

### DUO 流程分析
客户访问登录网页（nginx的前端页面）---> 输入账号密码提交验证请求到后端服务器 ---> 服务器端使用DUO SDK生成一些信息，然后返回一个重定向URL给到前端（如果直接返回重定向302给到前端的话，会有CORS问题）---> 前端页面使用 window.location=url 的方式重定向到 DUO 2FA页面进行验证 ---> 验证完成后重新跳转到某个前端页面 ---> 前端页面调用后端接口校验2FA的结果，成功则返回登陆成功，失败则提示失败

用户账号密码校验成功后，java 服务响应302重定向到DUO网站做 2FA 验证：
```
https://api-b43484c8.duosecurity.com/oauth/v1/authorize?scope=openid&response_type=code&redirect_uri=http://localhost:50000/duo-callback&client_id=DIQ9C44MPO7Y05AA23J9&request=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJkdW9fdW5hbWUiOiJhYWEiLCJzY29wZSI6Im9wZW5pZCIsInJlc3BvbnNlX3R5cGUiOiJjb2RlIiwicmVkaXJlY3RfdXJpIjoiaHR0cDovL2xvY2FsaG9zdDo1MDAwMC9kdW8tY2FsbGJhY2siLCJzdGF0ZSI6IjlhNjI1YTRiZjQ5MTk0MWRmY2IyN2Q4MTU5ZTA3ZTMyZjY4NSIsImV4cCI6MTc1ODA5NzUzOSwidXNlX2R1b19jb2RlX2F0dHJpYnV0ZSI6dHJ1ZSwiY2xpZW50X2lkIjoiRElROUM0NE1QTzdZMDVBQTIzSjkifQ.Zun-Rdiv9YIocG4VIfah0ezjfJbGr8g-3q5yqq-4FW_tND1Vbaz1HTphILg-yuLFhpy2qI2RqedwaTxz_nCrQg
```

### DUO 集成的跨域问题
```
We've received a few more reports of this issue in the last couple weeks. I believe what is happening is:

Folks are trying to add Duo to web applications that send login credentials to the server via XHR rather than a simple html form submit
The server is replying to the XHR with a 302 redirect to the Duo URL
Since this 302 is a reply to an XHR, it triggers a CORS preflight check (this may depend on the JS framework in use?)
Duo is not expecting a CORS preflight check and so does not respond appropriately
The CORS preflight errors out and so the whole redirect is canceled.
if anyone who is still being affected by this issue could let me know it I seem to be on the right track, that would be very helpful.
If I am right, there may be a few options...
A) The example implementation of the server code is not assuming XHR is in use. Affected web applications could be updated to respond to the credential submit not with a 302 redirect but with something that indicates to the client-side JS about the URL to follow
B) Duo might be able to support the CORS preflight check better
```