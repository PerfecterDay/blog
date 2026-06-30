# Springboot 配置双向tls认证
{docsify-updated}

`keytool` 是 Java 自带的工具，适合与 JKS 密钥库和信任库一起使用。

## 一、生成自签名CA证书

**生成CA密钥对和自签名证书**

> keytool -genkeypair -alias my-ca -keyalg RSA -keysize 2048 -validity 3650 -keystore ca.jks -storepass changeit -keypass changeit -dname "CN=My CA, OU=My Organization, O=My Company, L=My City, ST=My State, C=US" -ext bc:c

+   `-alias my-ca`：CA 证书的别名。
+   `-keystore ca.jks`：生成的密钥库文件（包含CA密钥对和证书）。
+   `-storepass` 和 `-keypass`：密钥库和密钥的密码。
+   `-dname`：证书的 Distinguished Name（DN）。
+   `-ext bc:c`：将证书标记为 CA 证书。

**导出CA证书**

<table><tbody><tr><td class="gutter"><p>1</p></td><td class="code"><div class="container"><p><code class="plain plain">keytool -exportcert -alias my-ca -keystore ca.jks -storepass changeit -file ca.crt</code></p></div></td></tr></tbody></table>

`-file ca.crt`：导出的 CA 证书文件。

## 二、使用CA签发服务器证书

**生成服务器密钥对**

> keytool -genkeypair -alias server -keyalg RSA -keysize 2048 -validity 365 -keystore server.jks -storepass changeit -keypass changeit -dname "CN=server.example.com, OU=My Organization, O=My Company, L=My City, ST=My State, C=US"

+   `-alias server`：服务器证书的别名。
+   `-keystore server.jks`：生成的服务器密钥库文件。

**生成证书签名请求（CSR）**

> keytool -certreq -alias server -keystore server.jks -storepass changeit -file server.csr

+   `-file server.csr`：生成的 CSR 文件。

**使用CA签发服务器证书**

> keytool -gencert -alias my-ca -infile server.csr -outfile server.crt -keystore ca.jks -storepass changeit -validity 365 -ext SAN=dns:server.example.com

+   `-infile server.csr`：输入的 CSR 文件。
+   `-outfile server.crt`：签发的服务器证书文件。
+   `-ext SAN=dns:server.example.com`：可选，添加 Subject Alternative Name（SAN）。

**将CA证书和服务器证书导入服务器密钥库**

<table><tbody><tr><td class="gutter"><p>1</p><p>2</p></td><td class="code"><div class="container"><p><code class="plain plain">keytool -importcert -alias my-ca -file ca.crt -keystore server.jks -storepass changeit -noprompt</code></p><p><code class="plain plain">keytool -importcert -alias server -file server.crt -keystore server.jks -storepass changeit</code></p></div></td></tr></tbody></table>

先导入 CA 证书，再导入签发的服务器证书。

## 三、使用CA签发客户端证书

**生成客户端密钥对**

> keytool -genkeypair -alias client -keyalg RSA -keysize 2048 -validity 365 -keystore client.jks -storepass changeit -keypass changeit -dname "CN=client.example.com, OU=My Organization, O=My Company, L=My City, ST=My State, C=US"

+   `-alias client`：客户端证书的别名。
+   `-keystore client.jks`：生成的客户端密钥库文件。

**生成证书签名请求（CSR）**

<table><tbody><tr><td class="gutter"><p>1</p></td><td class="code"><div class="container"><p><code class="plain plain">keytool -certreq -alias client -keystore client.jks -storepass changeit -file client.csr</code></p></div></td></tr></tbody></table>

+   `-file client.csr`：生成的 CSR 文件。

**使用CA签发客户端证书**

<table><tbody><tr><td class="gutter"><p>1</p></td><td class="code"><div class="container"><p><code class="plain plain">keytool -gencert -alias my-ca -infile client.csr -outfile client.crt -keystore ca.jks -storepass changeit -validity 365</code></p></div></td></tr></tbody></table>

+   `-infile client.csr`：输入的 CSR 文件。
+   `-outfile client.crt`：签发的客户端证书文件。

**将CA证书和客户端证书导入客户端密钥库**

<table><tbody><tr><td class="gutter"><p>1</p><p>2</p></td><td class="code"><div class="container"><p><code class="plain plain">keytool -importcert -alias my-ca -file ca.crt -keystore client.jks -storepass changeit -noprompt</code></p><p><code class="plain plain">keytool -importcert -alias client -file client.crt -keystore client.jks -storepass changeit</code></p></div></td></tr></tbody></table>

先导入 CA 证书，再导入签发的客户端证书。

## 四、配置信任库

**创建信任库并导入CA证书**

<table><tbody><tr><td class="gutter"><p>1</p></td><td class="code"><div class="container"><p><code class="plain plain">keytool -importcert -alias my-ca -file ca.crt -keystore truststore.jks -storepass changeit -noprompt</code></p></div></td></tr></tbody></table>

+   `-keystore truststore.jks`：生成的信任库文件。

## 五、配置服务器和客户端

### 1\. 服务器配置

在 Spring Boot 中配置：

<table><tbody><tr><td class="gutter"><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p></td><td class="code"><div class="container"><p><code class="plain plain">server:</code></p><p><code class="plain spaces">&nbsp;&nbsp;</code><code class="plain plain">ssl:</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">key-store: classpath:server.jks</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">key-store-password: changeit</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">key-alias: server</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">trust-store: classpath:truststore.jks</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">trust-store-password: changeit</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">client-auth: need # 要求客户端提供证书</code></p></div></td></tr></tbody></table>

### 2\. 客户端配置

在 Java 中配置：

<table><tbody><tr><td class="gutter"><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p></td><td class="code"><div class="container"><p><code class="plain plain">SSLContext sslContext = SSLContextBuilder.create()</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">.loadKeyMaterial(Paths.get("client.jks"), "changeit".toCharArray(), "changeit".toCharArray())</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">.loadTrustMaterial(Paths.get("truststore.jks"), "changeit".toCharArray())</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">.build();</code></p><p><code class="plain plain">HttpClient client = HttpClients.custom()</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">.setSSLContext(sslContext)</code></p><p><code class="plain spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="plain plain">.build();</code></p></div></td></tr></tbody></table>

## 六、总结

+   使用 `keytool` 可以完全替代 `openssl`，生成和管理自签名 CA 证书、服务器证书和客户端证书。
+   只需要将 CA 证书添加到信任库（`truststore.jks`），即可验证所有由该 CA 签发的证书。
+   这种方法适合 Java 生态系统，尤其是使用 JKS 密钥库和信任库的场景。

到此这篇关于Springboot实现TLS双向认证的文章就介绍到这了,更多相关Springboot TLS双向认证内容请搜索脚本之家以前的文章或继续浏览下面的相关文章希望大家以后多多支持脚本之家！