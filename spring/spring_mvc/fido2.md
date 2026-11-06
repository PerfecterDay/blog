https://webauthn.io/


FIDO2 是什么？
FIDO2（Fast IDentity Online 2）是一套用于无密码身份认证的开放标准。

它允许用户使用指纹、人脸识别、设备 PIN、安全密钥等方式登录系统，而不需要输入传统密码。

例如：
- 手机：使用 Face ID / Touch ID 登录。
- Windows：使用 Windows Hello 登录。
- Mac：使用 Touch ID 登录。
- 硬件安全密钥：使用 YubiKey 等设备进行认证。
FIDO2 的核心价值是：服务器不再保存用户密码，也不需要在登录时验证用户输入的密码，而是验证用户设备使用私钥生成的数字签名。
1. FIDO2 包含哪些标准？
FIDO2 主要由两个标准组成：
WebAuthn（Web Authentication）
W3C 标准
定义浏览器与 Web 应用之间的认证 API，以及客户端与服务器之间的交互。

CTAP2（Client to Authenticator Protocol 2）
FIDO Alliance 标准
定义客户端（浏览器、操作系统）如何与认证器（手机、安全密钥等）通信。

可以简单理解：
- WebAuthn：网站如何请求注册和认证。
- CTAP2：电脑或手机如何与安全认证设备交互。


FIDO2 的核心原理：非对称加密

FIDO2 使用非对称密钥体系。
+ 用户注册时，认证器会生成一对密钥。
以用户第一次在网站注册 Passkey 为例：
1. 浏览器向服务器请求创建 FIDO2 凭证。
2. 服务器生成随机 Challenge，返回给浏览器。
3. 浏览器调用 WebAuthn API：navigator.credentials.create()。
4. 操作系统调用认证器，要求用户进行指纹、人脸或 PIN 验证。
5. 认证器生成密钥对，并使用私钥签署注册数据。
6. 浏览器将注册结果（Attestation）返回服务器。
7. 服务器验证注册结果，保存公钥、Credential ID 等信息。
```
public class FidoCredential {
    private String userId;
    // 凭证唯一标识
    private byte[] credentialId;
    // 用户公钥
    private byte[] publicKey;
    // 签名计数器
    private long signCount;
}
```

用户登录时：
1. 浏览器向服务器请求认证。
2. 服务器生成随机 Challenge。
3. 浏览器调用 navigator.credentials.get()。
4. 认证器要求用户验证身份。
5. 认证器使用私钥对认证数据进行签名。
6. 浏览器将 Assertion 返回服务器。
7. 服务器根据 Credential ID 找到对应公钥，验证签名和 Challenge。

FIDO2 为什么能够防钓鱼？
这是 FIDO2 非常重要的特性。

假设真实网站是：`https://login.example.com`
攻击者创建了一个钓鱼网站：`https://login-example.attacker.com`

传统密码认证中，用户可能把密码输入钓鱼网站。
但 FIDO2 有一个重要机制：RP ID（Relying Party ID，依赖方标识）。
认证器生成凭证时，会将凭证与 RP ID 绑定。由于 RP ID 不匹配，浏览器不会允许钓鱼网站使用真实网站的凭证。
因此，即使用户访问了钓鱼网站，也不能直接使用真实网站的 FIDO2 凭证完成认证。

Passkey 可以保存在：
- 手机或电脑本地。
- 硬件安全密钥。
- Apple、Google 等生态的密码管理器中，并通过云端同步。
这里有一个重要区别：
- Device-bound Passkey：与特定设备绑定，私钥通常不能迁移。
- Synced Passkey：可以在同一生态的多个设备之间同步。
所以，FIDO2 并不意味着私钥一定永远只能存在于一台物理设备上。


## FIDO2 与 Passkey
FIDO2 是技术标准，Passkey（通行密钥）是基于这套标准实现的一种用户认证凭证。

Discoverable Credential（可发现凭证）
它允许认证器根据 RP ID 等信息找到对应凭证，而不必先由服务器提供 Credential ID。
这也是现代 Passkey 能够实现类似「选择账号 → 指纹验证 → 登录」体验的重要基础。