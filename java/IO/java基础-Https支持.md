# Java 对Https 的支持-Java Secure Socket Extension (JSSE)
{docsify-updated}
> https://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/JSSERefGuide.html

Java Secure Socket Extension（JSSE）实现了安全的互联网通信。 它为 TLS 和 DTLS 协议的 Java 版本提供了一个框架和实现方法，包括数据加密、服务器验证、信息完整性和可选客户端验证功能。


与浏览器和操作系统类似，JRE的安装目录下也保存了一份默认可信的CA证书列表，一般在 jre/lib/security/cacerts文件中，使用JDK自带的 keytool 工具可以查看其中的内容，默认密码为： changeit.
+ `keytool -import -alias ca_wang -keystore cacerts -file ca_wang.crt`:从ca_wang.crt的文件中导入证书到cacerts的 TrustStore ，别名为 ca_wang
+ `keytool -list -cacerts -alias ca_wang`:显示指定别名为 ca_wang 的 TrustStore 证书信息
+ `keytool -delete -cacerts -alias ca_wang`:删除指定别名的的 keystore 条目
+ `keytool -import -alias ca_wang -file C:\Users\BaIcy\Documents\xca\ca_wang.crt -keypass "" -keystore C:\Users\BaIcy\Documents\xca\ca_wang.jks -storepass test123`:将 crt 证书文件转换为 jks 的 keystore 文件

Java 平台下，证书尝尝被存储为 keystore 文件中，上面的 cacerts 就是一个 keystore 文件， keystore 文件不仅可以存储数字证书，还可以存储秘钥，存储在 keystore 文件中的对象有三种： Certificate（证书）、PrivateKey（私钥）和 SecretKey（对称加密秘钥）。

keystore 只是一种文件格式而已，实际上在 Java 的世界里 KeyStore 文件分为两种： keystore 和 truststore， keystore 保存公私钥，用来解密或者签名； truststore 保存信任的证书列表，访问 https 时，对被访问者进行认证，确定它是可信任的。

Java 使用以下主要类和接口来支持安全传输：
<center><img src="pics/jsse.jpg" width="50%"/></center>

### KeyManager
`KeyManager` 的主要职责是选择最终将被发送到远程主机的认证凭证。为了向远程对等体认证自己（本地安全套接字对等体），你必须用一个或多个KeyManager对象初始化一个SSLContext对象。你必须为每个将被支持的不同认证机制传递一个KeyManager。如果在SSLContext的初始化中传递了null，那么将创建一个空的KeyManager。如果使用内部默认上下文（例如，由SSLSocketFactory.getDefault()或SSLServerSocketFactory.getDefault()创建的SSLContext），那么将会创建一个默认的KeyManager

### TrustManager
TrustManager 的主要责任是确定所提交的认证凭证是否应该被信任。如果凭证不被信任，那么连接将被终止。为了验证安全套接字对等体的远程身份，你必须用一个或多个TrustManager对象初始化一个SSLContext对象。你必须为每个支持的认证机制传递一个TrustManager。如果在SSLContext的初始化中传递了null，那么将为你创建一个信任管理器。通常，一个信任管理器支持基于X.509公钥证书的认证（例如，X509TrustManager）。一些安全套接字的实现也可能支持基于共享秘钥、Kerberos或其他机制的认证。

TrustManager对象可以由TrustManagerFactory创建，或者通过提供接口的具体实现来创建。

TrustManager 和 KeyManager 之间的关系
+ TrustManager决定远程认证凭证（以及连接）是否应该被信任(通常是认证通信的对方)。
+ KeyManager决定向远程主机发送哪些认证凭证（通常是提供让对方认证的凭证）。

### 启用 Java SSL 调试日志
+ 命令行参数：`java -Djavax.net.debug=ssl:handshake -jar MyApp.jar`
+ 使用系统参数：`System.setProperty("javax.net.debug", "ssl:handshake");`
+ 打开网络调试: `-Djavax.net.debug=all` 

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