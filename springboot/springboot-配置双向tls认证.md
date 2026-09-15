# Springboot 与https/tls
{docsify-updated}

## 启用配置 https 以及配置 httpClient 使用 tls

1. 启用与配置 https
```
server:
  ssl:
    enabled: true

    key-store: classpath:server.p12
    key-store-password: xxx
    key-store-type: PKCS12

    enabled-protocols:
      - TLSv1.2

    ciphers:
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

2. 配置 RestClient/WebClient/RestTemplate 使用 tls
```
SSLContext sslContext = SSLContext.getInstance("TLSv1.2");

SSLParameters sslParameters = new SSLParameters();
sslParameters.setProtocols(new String[] {"TLSv1.2"});
sslParameters.setCipherSuites(new String[] {
    "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
});

HttpClient httpClient = HttpClient.newBuilder()
        .sslContext(sslContext)
        .sslParameters(sslParameters)
        .build();
```

总结：
```
Spring Boot
│
├── HTTPS Server
│     │
│     └── server.ssl
│           ├── enabled-protocols
│           └── ciphers
│
└── HTTPS Client
      │
      ├── RestClient
      ├── WebClient
      └── RestTemplate
             │
             └── 底层 HTTP Client
                    │
                    └── SSLContext / SSLParameters
                          ├── protocols
                          └── cipher suites
```




## 配置 tls 双重认证
`keytool` 是 Java 自带的工具，适合与 JKS 密钥库和信任库一起使用。

### 一、生成自签名CA证书

**生成CA密钥对和自签名证书**
```
keytool -genkeypair -alias my-ca -keyalg RSA -keysize 2048 -validity 3650 -keystore ca.jks -storepass changeit -keypass changeit -dname "CN=My CA, OU=My Organization, O=My Company, L=My City, ST=My State, C=US" -ext bc:c
```
+ `-alias my-ca`：CA 证书的别名。
+ `-keystore ca.jks`：生成的密钥库文件（包含CA密钥对和证书）。
+ `-storepass` 和 `-keypass`：密钥库和密钥的密码。
+ `-dname`：证书的 Distinguished Name（DN）。
+ `-ext bc:c`：将证书标记为 CA 证书。

**导出CA证书**

```
keytool -exportcert -alias my-ca -keystore ca.jks -storepass changeit -file ca.crt
```

`-file ca.crt`：导出的 CA 证书文件。

### 二、使用CA签发服务器证书

**生成服务器密钥对**
```
keytool -genkeypair -alias server -keyalg RSA -keysize 2048 -validity 365 -keystore server.jks -storepass changeit -keypass changeit -dname "CN=server.example.com, OU=My Organization, O=My Company, L=My City, ST=My State, C=US"
```
+ `-alias server`：服务器证书的别名。
+ `-keystore server.jks`：生成的服务器密钥库文件。

**生成证书签名请求（CSR）**

```
keytool -certreq -alias server -keystore server.jks -storepass changeit -file server.csr
```
+ `-file server.csr`：生成的 CSR 文件。

**使用CA签发服务器证书**
```
keytool -gencert -alias my-ca -infile server.csr -outfile server.crt -keystore ca.jks -storepass changeit -validity 365 -ext SAN=dns:server.example.com
```
+ `-infile server.csr`：输入的 CSR 文件。
+ `-outfile server.crt`：签发的服务器证书文件。
+ `-ext SAN=dns:server.example.com`：可选，添加 Subject Alternative Name（SAN）。

**将CA证书和服务器证书导入服务器密钥库**
```
keytool -importcert -alias my-ca -file ca.crt -keystore server.jks -storepass changeit -noprompt
keytool -importcert -alias server -file server.crt -keystore server.jks -storepass changeit
```

先导入 CA 证书，再导入签发的服务器证书。

### 三、使用CA签发客户端证书

**生成客户端密钥对**

> keytool -genkeypair -alias client -keyalg RSA -keysize 2048 -validity 365 -keystore client.jks -storepass changeit -keypass changeit -dname "CN=client.example.com, OU=My Organization, O=My Company, L=My City, ST=My State, C=US"

+   `-alias client`：客户端证书的别名。
+   `-keystore client.jks`：生成的客户端密钥库文件。

**生成证书签名请求（CSR）**

```
keytool -certreq -alias client -keystore client.jks -storepass changeit -file client.csr
```
+ `-file client.csr`：生成的 CSR 文件。

**使用CA签发客户端证书**

```
keytool -gencert -alias my-ca -infile client.csr -outfile client.crt -keystore ca.jks -storepass changeit -validity 365
```

+ `-infile client.csr`：输入的 CSR 文件。
+ `-outfile client.crt`：签发的客户端证书文件。

**将CA证书和客户端证书导入客户端密钥库**

```
keytool -importcert -alias my-ca -file ca.crt -keystore client.jks -storepass changeit -noprompt
keytool -importcert -alias client -file client.crt -keystore client.jks -storepass changeit
```

先导入 CA 证书，再导入签发的客户端证书。

### 四、配置信任库

**创建信任库并导入CA证书**

```
keytool -importcert -alias my-ca -file ca.crt -keystore truststore.jks -storepass changeit -noprompt
```
+ `-keystore truststore.jks`：生成的信任库文件。

### 五、配置服务器和客户端

#### 1\. 服务器配置

在 Spring Boot 中配置：
```
server:
  ssl:
    key-store: classpath:server.jks
    key-store-password: changeit
    key-alias: server
    trust-store: classpath:truststore.jks
    trust-store-password: changeit
    client-auth: need # 要求客户端提供证书
```

### 2\. 客户端配置

在 Java 中配置：
```
SSLContext sslContext = SSLContextBuilder.create()
        .loadKeyMaterial(Paths.get("client.jks"), "changeit".toCharArray(), "changeit".toCharArray())
        .loadTrustMaterial(Paths.get("truststore.jks"), "changeit".toCharArray())
        .build();
HttpClient client = HttpClients.custom()
        .setSSLContext(sslContext)
        .build();
```

## 六、总结

+   使用 `keytool` 可以完全替代 `openssl`，生成和管理自签名 CA 证书、服务器证书和客户端证书。
+   只需要将 CA 证书添加到信任库（`truststore.jks`），即可验证所有由该 CA 签发的证书。
+   这种方法适合 Java 生态系统，尤其是使用 JKS 密钥库和信任库的场景。

到此这篇关于Springboot实现TLS双向认证的文章就介绍到这了,更多相关Springboot TLS双向认证内容请搜索脚本之家以前的文章或继续浏览下面的相关文章希望大家以后多多支持脚本之家！