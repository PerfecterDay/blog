# Java 对Https 的支持-Java Secure Socket Extension (JSSE)
{docsify-updated}
> https://docs.oracle.com/en/java/javase/11/security/java-secure-socket-extension-jsse-reference-guide.html#GUID-93DEEE16-0B70-40E5-BBE7-55C3FD432345

Java Secure Socket Extension（JSSE）实现了安全的互联网通信。 它为 TLS 和 DTLS 协议的 Java 版本提供了一个框架和实现方法，包括数据加密、服务器验证、信息完整性和可选客户端验证功能。

与浏览器和操作系统类似，JRE的安装目录下也保存了一份默认可信的CA证书列表，一般在 jre/lib/security/cacerts文件中，使用JDK自带的 keytool 工具可以查看其中的内容，默认密码为： changeit.
+ `keytool -import -alias ca_wang -keystore cacerts -file ca_wang.crt`:从ca_wang.crt的文件中导入证书到cacerts的 TrustStore ，别名为 ca_wang
+ `keytool -list -cacerts -alias ca_wang`:显示指定别名为 ca_wang 的 TrustStore 证书信息
+ `keytool -delete -cacerts -alias ca_wang`:删除指定别名的的 keystore 条目
+ `keytool -import -alias ca_wang -file C:\Users\BaIcy\Documents\xca\ca_wang.crt -keypass "" -keystore C:\Users\BaIcy\Documents\xca\ca_wang.jks -storepass test123`:将 crt 证书文件转换为 jks 的 keystore 文件

Java 平台下，证书常常被存储为 keystore 文件中，上面的 cacerts 就是一个 keystore 文件， keystore 文件不仅可以存储数字证书，还可以存储秘钥，存储在 keystore 文件中的对象有三种： Certificate（证书）、PrivateKey（私钥）和 SecretKey（对称加密秘钥）。

keystore 只是一种文件格式而已，实际上在 Java 的世界里 `KeyStore` 文件分为两种： `keystore` 和 `truststore` ， `keystore` 保存公私钥，用来解密或者签名； `truststore` 保存信任的证书列表，访问 https 时，对被访问者进行认证，确定它是可信任的。

Java 使用以下主要类和接口来支持安全传输：
<center><img src="pics/jsse.jpg" width="40%"/></center>

`SSLSocket` 由 `SSLSocketFactory` 创建，或由接受传入连接的 `SSLServerSocket` 创建。而 `SSLServerSocket` 则由 `SSLServerSocketFactory` 创建。 `SSLSocketFactory` 和 `SSLServerSocketFactory` 对象均由 `SSLContext` 创建。 `SSLEngine` 由 `SSLContext` 直接创建，并依赖应用程序来处理所有 I/O 操作。

## 核心组件拆解
* `KeyStore` : 证书和私钥的存储容器，
* `SSLContext` ：核心引擎和大脑。作为一个工厂类，你需要在其中初始化协议版本（如 TLSv1.3）、身份凭证和信任规则。
* `TrustManager` ：入站安全把关者。它负责将对方的证书与你的 `TrustStore` （通常是 Java 默认的 cacerts 文件）进行比对，从而判断远程服务器是否安全可信。
* `KeyManager` ：出站安全管理者。它利用你的 `KeyStore` 提供你自己的私钥和证书。当远程服务器要求客户端认证（Client Authentication）时，它会出示这些凭证。
* `SSLSocket / SSLServerSocket` ：对标准 Java 网络套接字（Socket）的高级封装。它们会自动执行 TLS 握手，并对流经标准输入/输出流的数据进行自动加密。
* `SSLEngine` ：一个高级的、与传输协议解耦的类。用于复杂的非阻塞 I/O 操作（如 Java NIO 或 Netty）。它需要手动处理原始数据块，而不是依赖活跃的网络套接字连接。

### KeyStore
+ `getInstance(String type)`: Creates a new KeyStore instance of the specified type (e.g., JKS, PKCS12, JCEKS)
+ `load(InputStream stream, char[] password)`: Loads the keystore from the specified input stream. If null is passed, it initializes an empty keystore
+ `store(OutputStream stream, char[] password)`: Saves the keystore to the specified output stream, protected by the given password
+ `setEntry(String alias, KeyStore.Entry entry, KeyStore.ProtectionParameter protParam)`: Adds or updates an entry (e.g., key or certificate) in the keystore under the specified alias
+ `getEntry(String alias, KeyStore.ProtectionParameter protParam)`: Retrieves an entry from the keystore using the specified alias and protection parameters
+ `getCertificate(String alias)`: Retrieves a certificate from the keystore using the specified alias
+ `setCertificateEntry(String alias, Certificate cert)`: Adds or updates a certificate entry in the keystore under the specified alias
+ `containsAlias(String alias)`: Checks if the keystore contains an entry with the specified alias
+ `aliases()`: Returns an enumeration of all aliases in the keystore

```
KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
KeyStore ks2 = KeyStore.getInstance("pkcs12");

char[] pwdArray = "password".toCharArray();
ks.load(new FileInputStream("newKeyStoreFileName.jks"), pwdArray); //必须使用 load 初始化

try (FileOutputStream fos = new FileOutputStream("newKeyStoreFileName.jks")) {
    ks.store(fos, pwdArray);
}
```

### 获取 SSLSocketFactory 实例
可以通过以下方式获取 `SSLSocketFactory`：
+ 通过调用 `SSLSocketFactory.getDefault()` 静态方法获取默认工厂。
+ 将工厂作为 API 参数接收。也就是说，需要创建套接字但无需关心套接字配置细节的代码，可以包含一个带有 `SSLSocketFactory` 参数的方法，客户端可以通过调用该方法来指定创建套接字时应使用的 `SSLSocketFactory` （例如， `javax.net.ssl.HttpsURLConnection`）。
+ 构建一个具有特定配置行为的新工厂。可以通过实现自己的套接字工厂子类，或者使用另一个充当套接字工厂工厂的类，来创建新的套接字工厂实例。此类的一个示例是 `SSLContext` ，它作为基于提供者的配置类随 JSSE 实现一同提供。

## 获取 SSLSocket 实例
`javax.net.ssl.SSLSocket` 类是标准 Java `java.net.Socket` 类的子类。它支持所有标准套接字方法，并增加了安全套接字特有的方法。该类的实例封装了其创建时所基于的 `SSLContext` 。虽然提供了用于控制套接字实例安全套接字会话创建的 API，但信任和密钥管理并未直接对外暴露。

可以通过以下任一方式获取 `SSLSocket` 实例：
+ 可以通过 `SSLSocketFactory` 的实例，利用该类提供的多个 `createSocket` 方法之一来创建 `SSLSocket` 。
+ 可以通过 `SSLServerSocket` 类的 `accept` 方法来创建 `SSLSocket` 。

## 远端验证
在使用原始的 `SSLSocket` 和 `SSLEngine` 类时，在发送任何数据之前，应始终检查对端凭据。 `SSLSocket` 和 `SSLEngine` 类不会自动验证 URL 中的主机名是否与对端凭据中的主机名一致。如果未验证主机名，应用程序可能会因 URL 欺骗而遭到攻击。自 JDK 7 起，端点识别/验证流程可在 SSL/TLS 握手过程中处理。请参阅 `SSLParameters.getEndpointIdentificationAlgorithm` 方法。

HTTPS（基于 TLS 的 HTTP）等协议确实需要进行主机名验证。自 JDK 7 起， `HttpsURLConnection` 在握手过程中默认强制执行 HTTPS 端点识别。此外，应用程序还可以使用 `HostnameVerifier` 接口来覆盖默认的 HTTPS 主机名规则。

## SSLEngine

## SSLSession
`javax.net.ssl.SSLSession` 接口表示 `SSLSocket` 或 `SSLEngine` 连接中两个对等方之间协商的安全上下文。会话建立后，同一对等方之间后续连接的 `SSLSocket` 或 `SSLEngine` 对象可以共享该会话。

在某些情况下，握手过程中协商的参数在后续握手阶段需要被引用，以便做出信任决策。例如，有效的签名算法列表可能会限制可用于身份验证的证书类型。在握手过程中，可以通过在 `SSLSocket` 或 `SSLEngine` 上调用 `getHandshakeSession()` 方法来获取 `SSLSession` 。 `TrustManager` 或 `KeyManager` 的实现可以利用 `getHandshakeSession()` 方法获取有关会话参数的信息，以辅助决策。

## SSLContext
本节中的类和接口旨在支持 `SSLContext` 对象的创建和初始化， `SSLContext` 对象用于创建 `SSLSocketFactory` 、 `SSLServerSocketFactory` 和 `SSLEngine` 对象。这些辅助类和接口属于 `javax.net.ssl` 包。

`javax.net.ssl.SSLContext` 类是安全套接字协议实现的引擎类。该类的实例充当 `SSLSocket` 、 `SSLServerSocket` 和 `SSLEngine` 的工厂。一个 `SSLContext` 对象保存了在该上下文中创建的所有对象共用的所有状态信息。例如，当由该上下文提供的套接字工厂创建的套接字通过握手协议协商会话状态时，该会话状态便与 `SSLContext` 相关联。这些缓存的会话可被同一上下文中创建的其他套接字重用和共享。

每个 `SSLContext` 实例都通过其 `init` 方法，使用其执行身份验证所需的密钥、证书链和受信任的根 CA 证书进行配置。该配置以密钥管理器和信任管理器的形式提供。这些管理器为上下文所支持的密码套件中的身份验证和密钥协商方面提供支持。

获取并初始化 `SSLContext` 有多种方法：
1. 最简单的方法是在 `SSLSocketFactory` 或 `SSLServerSocketFactory` 类上调用静态方法 `SSLContext.getDefault`。该方法会创建一个默认的 `SSLContext` ，其中包含默认的 `KeyManager` 、 `TrustManager` 和 `SecureRandom` （一个安全的随机数生成器）。分别使用默认的 `KeyManagerFactory` 和 `TrustManagerFactory` 来创建 `KeyManager` 和 `TrustManager` 。所使用的密钥材料位于默认密钥库和信任库中，具体由“自定义默认密钥库和信任库、存储类型及存储密码”一节中描述的系统属性决定。调用 `SSLSocketFactory.getDefault()` 方法时，系统会自动创建、初始化一个 `SSLContext` 对象，并将其静态赋值给 `SSLSocketFactory` 类。因此，无需直接创建和初始化 `SSLContext` 对象（除非想覆盖默认行为）。
2. 让调用者对所创建上下文的行为拥有最大控制权的方法是：调用 `SSLContext` 类的静态方法 `SSLContext.getDefault` ，然后通过调用该实例的相应 `init()` 方法来初始化上下文。 `init()` 方法的一种变体接受三个参数：一个 `KeyManager` 对象数组、一个 `TrustManager` 对象数组以及一个 `SecureRandom` 对象。 `KeyManager` 和 `TrustManager` 对象可通过实现相应的接口，或使用 `KeyManagerFactory` 和 `TrustManagerFactory` 类生成实现来创建。随后，可以利用作为参数传递给 `TrustManagerFactory` 或 `KeyManagerFactory` 类 `init()` 方法的 `KeyStore` 中包含的密钥素材，分别初始化 `KeyManagerFactory` 和 `TrustManagerFactory` 。最后，可以调用 `getTrustManagers()` 方法（位于 `TrustManagerFactory` 中）和 `getKeyManagers()` 方法（位于 `KeyManagerFactory` 中），以获取信任管理器或密钥管理器的数组，每种类型的信任或密钥材料对应一个管理器。
3. 若要通过调用 `SSLContext.getInstance()` 工厂方法来创建 `SSLContext` 对象，必须指定协议名称。您还可以指定希望由哪个提供程序来提供所请求协议的实现:
   ```
   	public static SSLContext getInstance(String protocol);
	public static SSLContext getInstance(String protocol, String provider);
	public static SSLContext getInstance(String protocol, Provider provider);
   ```
所有的协议/算法名字参考：https://docs.oracle.com/en/java/javase/11/docs/specs/security/standard-names.html#sslcontext-algorithms

获取到 `SSLContext` 对象后必须使用 `init(...)` 方法初始化：
```
	public void init(KeyManager[] km, TrustManager[] tm, SecureRandom random);
```
如果 `KeyManager[]` 参数为 `null` ，则将为该上下文定义一个空的 `KeyManager` 。如果 `TrustManager[]` 参数为 `null` ，则将在已安装的安全提供程序中搜索 `TrustManagerFactory` 类的最高优先级实现（参见 `TrustManagerFactory` 类），并从中获取相应的 `TrustManager` 。同样， `SecureRandom` 参数可以为 `null` ，在这种情况下将使用默认实现。

如果使用内部默认 `SSLContext`（例如，通过 `SSLSocketFactory.getDefault()` 或 `SSLServerSocketFactory.getDefault()` 创建 `SSLContext` ），则会创建默认的 `KeyManager` 和 `TrustManager` 。同时还会选择默认的 `SecureRandom` 实现。

## TrustManager
`TrustManager` 的主要责任是确定所提交的认证凭证是否应该被信任。如果凭证不被信任，那么连接将被终止。为了验证安全套接字对等体的远程身份，你必须用一个或多个 `TrustManager` 对象初始化一个 `SSLContext` 对象。你必须为每个支持的认证机制传递一个 `TrustManager` `。如果在SSLContext` 的初始化中传递了 `null` ，那么将为你创建一个信任管理器。通常，一个信任管理器支持基于X.509公钥证书的认证（例如， `X509TrustManager` ）。一些安全套接字的实现也可能支持基于共享秘钥、 `Kerberos` 或其他机制的认证。

`TrustManager` 对象可以由 `TrustManagerFactory` 创建，或者通过提供接口的具体实现来创建:
```
CertificateFactory certFactory = CertificateFactory.getInstance("X.509");
InputStream inputStream = ResourceUtils.getURL("classpath:certs/ca.pem").openStream();
Certificate certificate = certFactory.generateCertificate(inputStream);

KeyStore trustStore = KeyStore.getInstance("PKCS12");
trustStore.load(null, null); // 传入 null 代表在内存中初始化，不加载磁盘文件

// 3. 将 CA 证书放入 KeyStore 中
// 信任证书不需要私钥，直接使用 setCertificateEntry
trustStore.setCertificateEntry("my-custom-ca", certificate);
TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
trustManagerFactory.init(trustStore);
```

## KeyManager
`KeyManager` 的主要职责是选择最终将被发送到远程主机的认证凭证。为了向远程对等体认证自己（本地安全套接字对等体），你必须用一个或多个KeyManager对象初始化一个 `SSLContext` 对象。你必须为每个将被支持的不同认证机制传递一个 `KeyManager` 。如果在 `SSLContext` 的初始化中传递了 `null` ，那么将创建一个空的 `KeyManager` 。如果使用内部默认上下文（例如，由 `SSLSocketFactory.getDefault()` 或 `SSLServerSocketFactory.getDefault()` 创建的 `SSLContext` ），那么将会创建一个默认的 `KeyManager`


`TrustManager` 和 `KeyManager` 之间的关系
+ `TrustManager` 决定远程认证凭证（以及连接）是否应该被信任(通常是认证通信的对方)。
+ `KeyManager` 决定向远程主机发送哪些认证凭证（通常是提供让对方认证的凭证）。

## 启用 Java SSL 调试日志
+ 命令行参数：`java -Djavax.net.debug=ssl:handshake -jar MyApp.jar`
+ 使用系统参数：`System.setProperty("javax.net.debug", "ssl:handshake");`
+ 打开网络调试: `-Djavax.net.debug=all` 

## 实战
假如已经有了下面的证书文件：
```
.
├── ca.pem
├── demo.net.key
├── demo.net.pem
```

+ `ca.pem` : 公司的 CA 证书
+ `demo.net.key` : 客户端请求的私钥
+ `demo.net.pem` : 客户端请求的证书

java 无法单独将一对公私钥对（demo.net.key + demo.net.pem）加载到 keystore 中，必须将他们转换为 p12 文件：
```
openssl pkcs12 -export -out keystore.p12 -inkey demo.net.key -in demo.net.pem
openssl pkcs12 -export -in demo.net.pem -inkey demo.net.key -certfile ca.pem -out keystore.p12

keytool -list -v -keystore 1.p12 -storetype PKCS12 -storepass "" > 1_struct.txt
```
1. 不带 `-certfile`：生成的 `keystore.p12` 里面只含有 2 个元素：你的私钥 `a.key` 和你的用户证书 `a.pem` 。
2. 带有 `-certfile ca.pem`：生成的 `keystore.p12` 里面含有 3 个元素：你的私钥 `a.key` 、你的用户证书 `a.pem` ，以及上级颁发机构的 CA 证书 `ca.pem` 。它们在内部会按照“用户证书 -> CA证书”的顺序自动组装成一条证书链（Certificate Chain）。

如果对方服务器要求你的 Java 客户端提供证书验证身份：使用命令一：Java 发送证书时只发送 a.pem。如果服务器的 TrustStore（信任库）里没有提前导入 ca.pem，服务器就无法验证你的 a.pem 是谁签发的，从而直接中断连接，抛出 `SSLHandshakeException` 。
使用命令二：Java 会把 a.pem 和 ca.pem 打包成一条完整的链条一起发送给服务器。即使服务器不认识你的 a.pem，但只要它信任 ca.pem（根证书），就能通过这条链条顺藤摸瓜信任你的客户端，握手成功。

如果黑客自己生成一套 bad.key、bad.pem，然后企图把 Google 的上级 CA 证书（比如 GTS Root R1）通过 `-certfile` 参数打包在一起，企图以此“沾光”骗过验证，这也是行不通的。证书链的验证是逐级校验签名的：客户端在拿到这条证书链后，并不是只要看到链上有知名 CA 就会信任。它会做出如下严苛的数学检查：提取 bad.pem。检查 bad.pem 的“颁发者（Issuer）”是谁。找到链上的 CA 证书，用 CA 证书的公钥去解密 bad.pem 尾部的数字签名。结果：因为 bad.pem 是黑客自己签发的，不是知名 CA 用它们的私钥签发的。知名 CA 的公钥根本无法解开这个签名。客户端会立刻判定“证书链断裂 / 签名无效”，直接抛出 `SSLHandshakeException` 或浏览器红色警告。

```
@Bean
@Profile("staging")
public ISprintClient stagingSprintService(@Value("${isprint.url}") String baseUrl) {

	try {
		KeyStore keyStore = KeyStore.getInstance("JKS");
		keyStore.load(ResourceUtils.getURL("classpath:certs/keystore.p12").openStream(), null);

		KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance("SunX509");
		keyManagerFactory.init(keyStore, null);

		CertificateFactory certFactory = CertificateFactory.getInstance("X.509");
		InputStream inputStream = ResourceUtils.getURL("classpath:certs/ca.pem").openStream();
		Certificate certificate = certFactory.generateCertificate(inputStream);

		// 2. 创建一个空的、内存中的 KeyStore
		// 信任库推荐使用 "PKCS12" 类型
		KeyStore trustStore = KeyStore.getInstance("PKCS12");
		trustStore.load(null, null); // 传入 null 代表在内存中初始化，不加载磁盘文件

		// 3. 将 CA 证书放入 KeyStore 中
		// 信任证书不需要私钥，直接使用 setCertificateEntry
		trustStore.setCertificateEntry("my-custom-ca", certificate);

		TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
		trustManagerFactory.init(trustStore);

		SSLContext sslContext = SSLContext.getInstance("TLS");
		sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), null);

		HttpClient httpClient = HttpClient.newBuilder()
				.connectTimeout(Duration.ofSeconds(1)) //建连超时
				.sslContext(sslContext)
				.build();

		JdkClientHttpRequestFactory jdkClientHttpRequestFactory = new JdkClientHttpRequestFactory(httpClient);
		jdkClientHttpRequestFactory.setReadTimeout(Duration.ofSeconds(5));

		RestClient restClient = RestClient.builder()
				.requestFactory(jdkClientHttpRequestFactory)
				.baseUrl(baseUrl)
				.build();

		HttpServiceProxyFactory factory =
				HttpServiceProxyFactory
						.builderFor(RestClientAdapter.create(restClient))
						.build();

		return factory.createClient(ISprintClient.class);
	}
}
```

## 问题总结
当遇到 https 证书验证失败时，你需要选择下面三种方法中的一个来解决：
+ Configure SSLContext with a TrustManager that accepts any certificate (see below).  
	```
		public class SSLTest {
		
		public static void main(String [] args) throws Exception {
			// configure the SSLContext with a TrustManager
			SSLContext ctx = SSLContext.getInstance("TLS");
			ctx.init(new KeyManager[0], new TrustManager[] {new DefaultTrustManager()}, new SecureRandom());
			SSLContext.setDefault(ctx);

			URL url = new URL("https://mms.nw.ru");
			HttpsURLConnection conn = (HttpsURLConnection) url.openConnection();
			conn.setHostnameVerifier(new HostnameVerifier() {
				@Override
				public boolean verify(String arg0, SSLSession arg1) {
					return true;
				}
			});
			System.out.println(conn.getResponseCode());
			conn.disconnect();
		}
		
		private static class DefaultTrustManager implements X509TrustManager {

				@Override
				public void checkClientTrusted(X509Certificate[] arg0, String arg1) throws CertificateException {}

				@Override
				public void checkServerTrusted(X509Certificate[] arg0, String arg1) throws CertificateException {}

				@Override
				public X509Certificate[] getAcceptedIssuers() {
					return null;
				}
			}
		}
	```

+ Configure SSLContext with an appropriate trust store that includes your certificate.

	```
	SSLContext context = SSLContext.getInstance("TLS");
	context.init(null,null,null);
	URL url = new URL("https://localhost");
	HttpsURLConnection httpsURLConnection = (HttpsURLConnection) url.openConnection();
	/** 下边这句会抛出 PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target 异常**/
	try{
		httpsURLConnection.getInputStream();
	}catch(Exception e){
		e.printStackTrace();
	}


	// 解决方案
	String keyStoreFile = "C:\\Users\\BaIcy\\Documents\\xca\\ca_wang.jks";
	String password = "test123";
	KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
	ks.load(new FileInputStream(keyStoreFile),password.toCharArray());

	TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
	tmf.init(ks);
	SSLContext sslContext = SSLContext.getInstance("TLS");
	sslContext.init(null,tmf.getTrustManagers(),null);
	httpsURLConnection.setSSLSocketFactory(sslContext.getSocketFactory());

	InputStream ins = httpsURLConnection.getInputStream();
	while (ins.read() > 0){

	}
	```

+ Add the certificate for that site to the default Java trust store.
  ```
  	COPY ./demo.pem demo.pem
  	keytool -import -alias myserver -file ./demo.pem -keystore /usr/local/openjdk-21/lib/security/cacerts -storepass changeit -noprompt
  ```