# 安全相关
{docsify-updated}

### 安全管理器与访问权限

一旦某个类被加载到虚拟机中，并由检验器检查过之后， Java 平台的第二种安全机制就会启动，这个机制就是**安全管理器**。安全管理器是一个负责控制具体操作是否允许执行的类。安全管理器负责检查的操作包括以下内容：
+ 创建一个新的类加载器
+ 退出虚拟机
+ 使用反射访问另一个类的成员
+ 访问本地文件
+ 打开 socket 连接
+ 启动打印作业
+ 访问系统剪贴板
+ 访问 AWT 事件队列
+ 打开一个顶层窗口

整个 Java 类库中还有许多其他类似的检查。运行Java 程序时，默认是不安装安全管理器的，这样所有的操作都是被允许的。

Java的安全策略建立了**代码来源**和**访问权限集**之间的映射关系。
<center><img src="pics/code-permission.png" width="30%"></center>

代码来源(code source)是由一个代码位置和一个证书集指定的。代码位置指定了代码的来源。例如，远程applet代码的代码位置是下载 applet 的 HTTP URL，位于JAR文件中的代码的代码位置是该文件的URL。证书的目的是要由某一方来保障代码没有被篡改过。

权限（ permission ）是指由安全管理器负责检查的任何属性。Java 平台支持许多访问权限类，每个类都封装了特定权限的详细信息。如： `FilePermission permission = new FilePermission("/tmp/*","read,write");` 还有许多类似 `FilePermission` 的类。
更为重要的是 `Policy` 类默认实现了可以从**访问权限文件**中读取权限。在权限文件中，同样的权限表示为：`permission java.io.FilePermission "/tmp/*,"read,write"`; 这样的话，我们就可以直接在权限文件中配置一些权限控制。

每个类(Class对象)都有一个保护域，它是一个用于封装类的代码来源和权限集合的对象。当 `SecurityManager` 类需要检查某个权限时，它要查看当前位于调用堆栈上的所有方法的类，然后它要获得所有类的保护域，并且询问每个保护域，其权限集合是否允许执行当前正在被检查的操作。如果所有的域都同意，那么检查得以通过。否则，就会抛出一个 `SecurityException` 异常。

#### 安全策略文件

策略管理器要读取相应的策略文件，这些文件包含了将代码来源映射为权限的指令。可以将策略文件的配置安装在标准位置上。 默认情况下，有两个位置可以配置策略文件：

+ Java 平台主目录的 `java.policy` 文件。
+ 用户主目录的 `.java.policy` 文件（注意文件名前面的圆点）。

也可以在 `java.security` 配置文件中修改这些文件的位置，默认位置设定为：
    ```
    policy.url.1=file:${java.home}/conf/security/java.policy
    policy.url.2=file:${user.home}/.java.policy 
    ```
系统管理员可以修改 `java.security` 文件，并可以指定驻留在另外一台服务器上并且用户无法修改的策略 URL。但是通常情况下，我们不愿意经常去修改这些默认标准文件，而更愿意为自己的程序单独建立一个策略配置文件 myapp.policy，要应用这个策略文件有两种方法：

1. 在 main 方法启动时，设置系统属性：`System.setProperty("java.security.policy","myapp.policy");`
2. 启动程序时，使用 -D 设置：`java -Djava.security.policy=myapp.policy myapp`，这样自定义的策略文件和标准策略文件会一起生效，如果使用`java -Djava.security.policy==myapp.policy myapp`则会忽略标准策略文件

另外，正如前文所说，默认时不安装安全管理器的，因此在安装安全管理器之前，是看不到策略文件的作用的。可以在 main 方法中调用 `System.setSecurityManager(new SecurityManager());` 设置安全管理器。也可以在启动时设置：`java -Djava.security.policy==myapp.policy -Djava.security.manager myapp`

策略文件包含一系列 grant 项。每一项都具有以下的形式：

```java
grant codebase <url>{
    permission <permissionClassName> <targetName>, <actionList>;
    permission <permissionClassName> <targetName>, <actionList>;
    permission <permissionClassName> <targetName>, <actionList>;
    ...
}
```
如果 url 以 “/” 结束，则被视为一个目录，否则视为一个jar 的名字。


### JAAS(Java Authentication and Authorization)

Java认证和授权服务包含两部分：

+ 认证部分主要负责确定程序使用者的身份
+ 授权将各个用户映射到相应的权限

Java的安全基础架构主要有以下特点：

+ 实现是互相独立的，java中的安全服务是由 provider 提供的，这些 provider 通过标准接口插入到 java 平台，应用程序可以直接请求这些 provider 提供的安全服务。
+ provider 可跨应用程序进行互操作。具体而言，应用程序未绑定到特定的 provider ，并且 provider 也未绑定到特定的应用程序。
+ 可扩展的， java 中提供了一些基础的安全服务实现，应用程序可以对这些服务进行扩展。且 java 平台也支持安装自定义的 provider

### java SPI(Servicec provider Interface)机制

是JDK内置的一种服务提供发现机制。SPI是一种动态替换发现的机制， 比如有个接口，想运行时动态的给它添加实现，你只需要添加一个实现。我们经常遇到的就是java.sql.Driver接口，其他不同厂商可以针对同一接口做出不同的实现，mysql和postgresql都有不同的实现提供给用户，而Java的SPI机制可以为某个接口寻找服务实现。

当服务的提供者提供了一种接口的实现之后，需要在classpath下的META-INF/services/目录里创建一个以服务接口命名的文件，这个文件里的内容就是这个接口的具体的实现类。当其他的程序需要这个服务的时候，就可以通过查找这个jar包（一般都是以jar包做依赖）的META-INF/services/中的配置文件，配置文件中有接口的具体实现类名，可以根据这个类名进行加载实例化，就可以使用该服务了。JDK中查找服务实现的工具类是：java.util.ServiceLoader。

```java
ServiceLoader<SPITest>  spiTests = ServiceLoader.load(com.company.SPITest.class);
for (SPITest spiTest : spiTests) {
    System.out.println(spiTest.test());
}
```

### 安全加密算法

常见的算法如下：

#### 消息摘要-防篡改

消息摘要是数据块的指纹。可以基于任意长的数据块计算出一个定长的不可逆的数字序列，通常称为指纹。人们希望，任何两个不同的输入数据，计算得到的指纹也不一样。当然这是不可能的，因为输入的数据有无数种可能，而定长的指纹序列是可数的，与长度有关。但是，通过设计合适的算法，人们可以使得伪造一个具有相同指纹得数据块在计算复杂度上是不可能。

Java实现了
+ MD5
+ SHA-1
+ SHA-256
+ SHA-384
+ SHA-512

等摘要算法。 `MessageDigest` 类是创建封装了指纹算法得对象的工厂，它的静态方法 `getInstance(String algorithm)` 返回继承自 `MessageDigest` 的一个算法对象。 `MessageDigest`既是一个工厂，也是消息摘要算法的超类。

总结来说，消息摘要就是能提取出一个数据块的指纹，任何对数据块的篡改都能导致指纹的不匹配，所以消息摘要具有**防篡改**功能。

#### 消息签名-防伪造

假如要传送的消息被截获，并且攻击者篡改了内容后又重新生成了指纹（重放攻击），接收者是无法识别出这条消息被篡改了的。基于这点，发送者可以用自己的私钥对消息和指纹进行签名，然后将公钥告知接收方，接收方收到消息后，用公钥进行验签。如果验签成功，就能说明这条消息确实是由发送者发送的，因为没有人可以伪造发送方的私钥。但是，在这里有一个很重要的环节是你必须确保你得到的公钥确实是发送者自己的公钥。因为任何人都可以自己生成一对公私钥对。一个可用的方案是我们找一个权威机构来对公钥进行认证，对真实可信的公钥就用权威机构的私钥进行签名，并且权威机构将自己的公钥公布在网上。这样接收方在确认收到的公钥是否可信时，就能直接用权威机构的公钥去验签，验签成功的公钥就是可信的公钥。
常用的数字签名算法包括:
+ DSA
+ RSA
+ HMAC-SHA

SHA-256 和 HMAC-SHA256 都是加密散列函数，但它们的用途不同。 SHA-256 是一个单向函数，它接收输入信息并产生一个固定大小的输出（称为散列），该输出对输入信息是唯一的。 另一方面，HMAC-SHA256 是一种密钥散列函数，它在应用 SHA-256 散列函数之前将密钥与输入信息结合在一起。 这就提供了一个额外的安全层，确保只有获得秘钥的人才能验证信息的完整性和真实性。 HMAC-SHA256 常用于各种协议和应用中的信息验证和完整性检查，包括数字签名、VPN 和安全信息传送。

#### 对称加密

上述摘要、签名算法只能确认消息来源的可靠性，具体的说，他们能保证消息是特定的发送者发出的而且没有被篡改。但是，消息的内容本身没有被加密（当然如果使用公钥加密传送内容也是可以的）。通常使用对称加密来加密传送的消息。

算法名称是一个字符串，比如“ AES ”或者“ DES/CBC/PKCS5Padding ” 。DES ，即数据加密标准，是一个密钥长度为 56 位的古老的分组密码 。 DES 加密算法在现在看来已经是过时了，因为可以用穷举法将它破译（[参见该网页中的例子](http://w2.eff.org/Privacy/Crypto/Crypto_misc/DESCracker) ） 。 更好的选择是采用它的后续版本，即高级加密标准AES。

AES目前支持三种模式：AES-128 (128 bits), AES-192 (192 bits), and AES-256 (256 bits)。

Java种的 `Cipher` 类是所有加密算法的超类，通过调用 `getInstance(String algorithm)` 或者 `getInstance(String algorithm,String provider)` 来获得一个加密算法对象。一个`Cipher`可以有多种使用方式，比如加密、解密等，Java中定义了4种模式：

1. Cipher.ENCRYPT_MODE
2. Cipher.DECRYPT_MODE
3. Cipher.WRAP_MODE
4. Cipher.UNWRAP_MODE

在使用 Cipher 之前必须调用它的

1. `init(...)` 方法初始化设置好密钥（ `Key` 类来表示）和模式或者其它参数，不同的加密算法要求初始化的参数不同
2. 调用`update()`方法将要加密的数据传递给 Cipher
3. 最后调用 `doFinal()`方法完成加密并返回加密后的字节数据。

生成密钥：

1. 为加密算法获取 `KeyGenerator`，通过`KeyGenerator.getInstance("AES")`来获得。
2. 用随机源或指定长度来初始化 `KeyGenerator`。
3. 调用 `KeyGenerator` 的 `generateKey()` 生成密钥

```java
KeyGenerator keyGenerator = KeyGenerator.getInstance(cipher);
keyGenerator.init(keySize);
Key key = keyGenerator.generateKey();

Cipher cipher = Cipher.getInstance("AES");
cipher.init(Cipher.ENCRYPT_MODE, key);
return new String(Base64.getEncoder().encode(cipher.doFinal(val.getBytes())));
```

#### 非对称加密

非对称加密在需要一对公私钥对，所以在生成密钥时与对称加密算法有一点差别。

```java
Cipher rsaCipher = Cipher.getInstance("RSA");
KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
SecureRandom secureRandom = new SecureRandom();
keyPairGenerator.initialize(2048,secureRandom);
KeyPair keyPair = keyPairGenerator.generateKeyPair();
Key publicKey = keyPair.getPublic();
Key privateKey = keyPair.getPrivate();
rsaCipher.init(Cipher.UNWRAP_MODE,publicKey);
byte[] wrapKey = rsaCipher.wrap(aeskey);
```

### SSL证书主流的格式

仅在X.509证书中就有几种编码格式和扩展名。你可能见过.pem、.crt、.p7b、.der、pfx等等的证书。如果我们问是否所有的扩展名都是一样的，答案是既是又不是。所有这些证书都是一样的，但有不同的编码标准。所有这些都取决于你的应用程序，或操作系统接受的编码类型。

<center><img src="pics/x509.png" width="70%"></center>

SSL证书格式主要有公钥证书格式标准X.509中定义的PEM和DER、公钥密码标准PKCS中定义的PKCS#7和PKCS#12、Java环境专用的JKS。

PEM是基于Base64编码的证书格式，扩展名包括pem、crt和cer。Linux系统使用crt，Windows系统使用cer。PEM证书通常将根证书、中间证书和用户证书分开存放，主要用于Apache和Nginx。

DER是基于二进制编码的证书格式，扩展名包括der、crt和cer。DER证书相当于PEM证书的二进制版本，主要用于Java平台。

PKCS#7是基于Base64编码的证书格式，扩展名包括p7b和p7c。PKCS#7证书通常将根证书和中间证书合并存放，将用户证书单独存放。PKCS#7证书不包含私钥，主要用于Tomcat和Windows Server。

PKCS#12是基于二进制编码的证书格式，扩展名包括pfx和p12。PKCS#12证书通常将根证书、中间证书、用户证书和私钥合并存放并设置密码，主要用于Windows Server。

JKS是是基于二进制编码的证书格式，扩展名是jks。JKS证书通常将根证书、中间证书、用户证书和私钥合并存放并设置密码，主要用于Java Web Server。

证书颁发机构CA签发的证书通常是PEM格式或PKCS#7格式，而PKCS#12格式和JKS格式的证书需要进行证书格式转换才能得到。我们可以通过OpenSSL、Keytool或在线证书转换工具等方式将PEM格式或PKCS#7格式的证书转换为我们需要的其他格式。

#### SSL证书格式转换方法

常见SSL证书主要的文件类型和协议有: PEM、DER、PFX、JKS、KDB、CER、KEY、CSR、CRT、CRL、OCSP、SCEP等。而最常见的SSL证书格式之间的转换有：

1. 将JKS转换成PFX  
	可以使用Keytool工具，将JKS格式转换为PFX格式。
	```
		keytool -importkeystore -srckeystore D:\server.jks -destkeystore D:\server.pfx -srcstoretype JKS -deststoretype PKCS12
	```
2. 将PFX转换为JKS  
	可以使用Keytool工具，将PFX格式转换为JKS格式。
	```
	keytool -importkeystore -srckeystore D:\server.pfx -destkeystore D:\server.jks -srcstoretype PKCS12 -deststoretype JKS
	```
3. 将PEM/KEY/CRT转换为PFX  
	使用OpenSSL工具，可以将密钥文件KEY和公钥文件CRT转化为PFX文件。
	```
	openssl pkcs12 -export -out server.pfx -inkey server.key -in server.crt
	```
4. 将PFX转换为PEM/KEY/CRT  
	```
	openssl pkcs12 -in server.pfx -nodes -out server.pem
	openssl rsa -in server.pem -out server.key
	```

### keytool 工具

1. 生成密钥对： `keytool -genkeypair -keystore alice.certs -alias alice`
2. 将生成的密钥导出为一个证书： `keytool -exportcert -keystore alice.certs -alias alice -file alice.cer`
3. 打印证书内容： `keytool -printcert -file alice.cer`
4. `keytool -list -keystore jre/lib/security/cacerts -storepass changeit`
5. 导入证书到密钥库： `keytool -importcert -keystore bob.certs -alias alice -file alice.cer` 绝对不妥将你并不完全信任的证书导入到密钥库中 。 一旦证书添加到密钥库中，使用密钥库的任何程序都会认为这些证书可以用来对签名进行校验。注意：当你的cer/pem 文件中有多个CA证书时，keytool 只会导入文件中的第一个证书。
6. `-storepass` : 需要传递密码时使用这个选项

### Https 双向认证

> https://help.aliyun.com/document_detail/160093.html