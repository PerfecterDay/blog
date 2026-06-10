#  HTTPS及Java支持
{docsify-updated}
> https://help.aliyun.com/document_detail/160093.html  
> https://stackoverflow.com/questions/27388583/relationship-between-key-store-trust-store-and-certificate  
> https://howhttps.works/certificate-authorities/

## HTTPS协议原理

思考一下通信过程应该要处理的问题：

1. 网络环境下，假设A与B通信，要保证通信内容安全，必须使用加密算法加密，出于效率考虑（非对称加密比较复杂，计算量大，每次传输都使用的话，效率低）一般使用对称加密算法，双方用同一秘钥加解密。
2. 在web模型中，服务器与N个浏览器客户端通信，如果使用相同的秘钥，则相当于未加密，必须为不同的客户端使用不同的秘钥。所以每次通信的第一步是协商出一个安全的秘钥。
3. 客户端如果直接将秘钥发送给服务器，告诉服务器使用这个秘钥进行加密。如果发送过程被截取，则秘钥会泄露。
4. 所以，发送秘钥时，需要使用非对称加密算法，即用服务器端的公钥对秘钥加密，只有服务器端的私钥才可解密。即使被截获也无法解密对称秘钥。
5. 那么客户端如何获取服务端的公钥呢？
6. 客户端发起第一次请求的时候，服务器端可以下发公钥，因为下发过程是未加密的，所以如果只是简单的下发公钥，有可能被中间人调包，即使用中间人的公钥替换服务器端公钥，客户端无法确认公钥是否可信。保证可信的方法一般是签名，即如果服务器使用私钥对报文进行签名，则客户端可以使用服务器的公钥进行验签。但是，此时下发的内容正是公钥，所以显然客户端还没有公钥。所以只能考虑使用第三方签名。
7. 服务器向第三方申请一个数字证书：将公钥和相关信息提交到第三方机构，第三方机构使用私钥对服务器提供的公钥及其它信息进行证书制作：首先，针对服务器的网址、公钥、摘要算法及其它信息计算摘要，然后利用自身的私钥对摘要签名。
8. 客户端保存了第三方机构的公钥(浏览器和操作系统会保留信任机构的公钥)，并信任这些公钥，这些公钥本身一般也是以证书的形式存在。
9.  服务器可以下发数字证书，客户端使用第三方公钥对其验签。获取摘要后，使用相同的摘要算法计算摘要内容，然后比较计算出来的摘要和验签出来的摘要是否一致。如果一致说明这个证书是有效的数字证书，并且证书中的信息（服务器下发的公钥、服务器网址信息、摘要算法等）未经篡改。
10. 客户端可以比较证书中网址信息是否是当前网站，如果不是，说明证书虽然有效，但是该网站并不能其就是其所声明的真实网站。说到底，证书的一个作用是证明你访问的网址是真实的网址而不是钓鱼网站。证书里边的网址是经过认证的，你访问的网址是否与证书的网址一致说明其是否是假冒网站。
11. 假设某个不怀好意的中间人在同一家机构申请了一个证书，当然，此证书不会与服务器的证书一样，然后用这个证书替换了服务器下发的证书。那么客户端收到证书后，虽然能够解密验签通过，但是显然，证书中的网址显然不是服务器网址，所以客户端应该放弃这次通信。
12. 综合起来看，数字证书的作用就是证明某人是真实的某人的作用：你想访问某个网站，但是你不能肯定这个网站就是真实的网站而不是钓鱼网站，那么如果有https，那么获取到证书后，证书中的内容就能证明该网站是不是真实的网站。并且，证书中包含了该网站的公钥。

反过来说，我们访问一个网站的大致过程是这样的：
<center><img src="pics/https-1.png" width="60%"></center>

1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务器端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端读取公钥证书（server.crt），使用第三方机构的公钥（一般以CA证书的形式存在）对证书进行验证，通过证书验证该网站是不是真实的网站。取出了服务端公钥；
4. 客户端生成一个随机数（密钥R），用刚才得到的服务器公钥去加密这个随机数形成密文，发送给服务端；
5. 服务端用自己的私钥（server.key）去解密这个密文，得到了密钥R
6. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。

## 双向认证
<center><img src="pics/https-2.png" width="60%"></center>

1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端读取公钥证书（server.crt），取出了服务端公钥；
4. 客户端将客户端公钥证书（client.crt）发送给服务器端；
5. 服务器端使用根证书（root.crt）解密客户端公钥证书，拿到客户端公钥；
6. 客户端发送自己支持的加密方案给服务器端；
7. 服务器端根据自己和客户端的能力，选择一个双方都能接受的加密方案，使用客户端的公钥加密后发送给客户端；
8. 客户端使用自己的私钥解密加密方案，生成一个随机数R，使用服务器公钥加密后传给服务器端；
9.  服务端用自己的私钥去解密这个密文，得到了密钥R
10. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。


## nginx使用自签名证书配https
我们使用xca工具来制作证书，先下载安装xca工具，地址是http://xca.hohnstaedt.de/。
首先要制作一张CA证书，然后用CA证书签发一张或多张证书，每张证书模拟每个网站下发的证书，里边存放的就是网站服务器端的公钥及网址等信息。制作每张证书的时候，都需要一对公私钥密码。下图是制作好的证书，其中一张是CA证书，用来签发其它证书：
<center><img src="pics/cert_generation.png" width=60%></center>

然后，我们将CA证书和制作的证书分别导出来：
<center><img src="pics/cert_export.png"></center>

上图中 ca_wang.crt 是CA证书，只包含公钥信息，后边需要将其安装到浏览器的信任根证书列表中，这样浏览器收到服务器下发的证书时，就能用CA证书验证其是否可被信任了（因为下发的证书是经过CA证书对应的私钥签名的）。
localhost_cert.crt 是CA签发的服务端证书，https通信时，服务端下发的就是这张证书。
locahost_keypair.pem 是 localhost_cert.crt 对应的私钥，配置 nginx 时会用到：
```
server {
       listen       443 ssl;
       server_name  localhost;

       ssl_certificate      C:\Users\BaIcy\Documents\xca\localhost_cert.crt;
       ssl_certificate_key  C:\Users\BaIcy\Documents\xca\localhost_keypair.pem;

       ssl_session_cache    shared:SSL:1m;
       ssl_session_timeout  5m;

       ssl_ciphers  HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers  on;

       location / {
           root   html;
           index  index.html index.htm;
       }
    }
```
nginx配置好后，将 ca_wang.crt 导入到 chrome 浏览器的信任根证书列表中：
<center><img src="pics/chrome_ca.png"></center>

然后重启chrome以https方式访问 localhost，就会发现是信任的网站了：
<center><img src="pics/chrome_https.png"></center>

## java 对 Https 的支持
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