#  OpenSSL
{docsify-updated}


+ [Mozilla的证书](https://hg.mozilla.org/releases/mozilla-beta/file/tip/security/nss/lib/ckfw/builtins/certdata.txt)
+ [Curl的证书](https://curl.se/docs/caextract.html)

### 命令
+ `openssl version -a` : 查看版本信息，`OPENSSLDIR` 中存放了openssl配置的路径
+ `openssl help` : 帮助信息


### 生成自己的CA-Certification Authority

#### 创建配置文件 root-ca.conf
(windows 注意文件编码问题： configuration file routines:def_load_bio:missing equal sign)

```
   [default]
   name                    = root-ca
   domain_suffix           = example.com
   aia_url                 = http://$name.$domain_suffix/$name.crt
   crl_url                 = http://$name.$domain_suffix/$name.crl
   ocsp_url                = http://ocsp.$name.$domain_suffix:9080
   default_ca              = ca_default
   name_opt                = utf8,esc_ctrl,multiline,lname,align
   
   [ca_dn]
   countryName             = "GB"
   organizationName        = "Example"
   commonName              = "Root CA"
   
   [ca_default]
   home                    = .
   database                = $home/db/index
   serial                  = $home/db/serial
   crlnumber               = $home/db/crlnumber
   certificate             = $home/$name.crt
   private_key             = $home/private/$name.key
   RANDFILE                = $home/private/random
   new_certs_dir           = $home/certs
   unique_subject          = no
   copy_extensions         = none
   default_days            = 3650
   default_crl_days        = 365
   default_md              = sha256
   policy                  = policy_c_o_match
   
   [policy_c_o_match]
   countryName             = match
   stateOrProvinceName     = optional
   organizationName        = match
   organizationalUnitName  = optional
   commonName              = supplied
   emailAddress            = optional
   
   [req]
   default_bits            = 4096
   encrypt_key             = yes
   default_md              = sha256
   utf8                    = yes
   string_mask             = utf8only
   prompt                  = no
   distinguished_name      = ca_dn
   req_extensions          = ca_ext
   
   [ca_ext]
   basicConstraints        = critical,CA:true
   keyUsage                = critical,keyCertSign,cRLSign
   subjectKeyIdentifier    = hash
   
   [sub_ca_ext]
   authorityInfoAccess     = @issuer_info
   authorityKeyIdentifier  = keyid:always
   basicConstraints        = critical,CA:true,pathlen:0
   crlDistributionPoints   = @crl_info
   extendedKeyUsage        = clientAuth,serverAuth
   keyUsage                = critical,keyCertSign,cRLSign
   nameConstraints         = @name_constraints
   subjectKeyIdentifier    = hash
   
   [crl_info]
   URI.0                   = $crl_url
   
   [issuer_info]
   caIssuers;URI.0         = $aia_url
   OCSP;URI.0              = $ocsp_url
   
   [name_constraints]
   permitted;DNS.0=example.com
   permitted;DNS.1=example.org
   excluded;IP.0=0.0.0.0/0.0.0.0
   excluded;IP.1=0:0:0:0:0:0:0:0/0:0:0:0:0:0:0:0
   
   [ocsp_ext]
   authorityKeyIdentifier  = keyid:always
   basicConstraints        = critical,CA:false
   extendedKeyUsage        = OCSPSigning
   noCheck                 = yes
   keyUsage                = critical,digitalSignature
   subjectKeyIdentifier    = hash
   ```
#### 创建文件夹
```
   $ mkdir root-ca
   $ cd root-ca
   $ mkdir certs db private
   $ chmod 700 private
   $ touch db/index
   $ openssl rand -hex 16  > db/serial (windos: openssl rand -hex 16 | Out-File -encoding ascii -NoNewline "db\serial"，否则会报 error while loading serial number openssl)
   $ echo 1001 > db/crlnumber
   ```
 + certs/: 证书存储；新的证书将在发布时放置在此处。
 + db/:该目录用于证书数据库（索引）和保存下一个证书和CRL序列号。OpenSSL将创建一些附加文件需要。
 + private/: 此目录将存储私钥，一个用于CA，另一个用于OCSP响应者。储存这些信息的机器应该只有最少数量的用户帐户。

#### 创建CA
1. 创建密钥和CSR：
	```
	openssl req -new \
	-config root-ca.conf \
	-out root-ca.csr \
	-keyout private/root-ca.key
	```
2. 创建自签名root证书
	```
	openssl ca -selfsign \
	-config root-ca.conf \
	-in root-ca.csr \
	-out root-ca.crt \
	-extensions ca_ext
	```

#### 使用CA签发证书
   

### 测试TLS
```
openssl s_client -crlf \
-connect www.feistyduck.com:443 \
-servername www.feistyduck.com
```
+ `-connect` 选项用于建立TCP连接
+ `-servername` 用于指定在TLS级别发送的主机名。

1.1.1 之后的版本会自动发送 servername ，所以我们可以省略这个参数。但是以下情况仍让需要手动指定这个参数：
1. 使用的是老版本（1.1.1之前）
2. connect 的是一个 IP 地址
3. TLS 主机与 servername 不同

可以使用 `-noservername` 选项来禁止在 TLS 握手的时候发送主机域名信息。

#### 证书验证
如果证书验证失败，通常会显示：
`Verify return code: 20 (unable to get local issuer certificate)`

#### 域名验证
openssl 默认不会验证域名，需要手动加上`-verify_hostname www.feistyduck.com` 参数：
```
openssl s_client -connect www.feistyduck.com:443 -verify_hostname www.feistyduck.com
```
如果域名验证失败会返回:
`Verify return code: 62 (Hostname mismatch)`

如果测试的是IP，那么需要使用`-verify_ip` 选项指定验证的IP


CRL： Certiﬁcate Revocation List (CRL) 
OSCP: Online Certiﬁcate Status Protocol 
AIA: Authority Information Access

#### 测试性能
```
＃ 只通过新会话获取远端test.html页面
openssl s_time -connect remote.host:443 -www /test.html -new

# 使用SSL v3和高级别加密算法
openssl s_time \
	-connect remote.hot:443 -www /test.html -new \
	-ssl3 -cipher HIGH
	
# 测试不同加密算法的性能，每种算法测试10秒
IFS=":"
for c in $(openssl ciphers -ssl3 RSA); do
	echo $C
	openssl s_time -connect remote.host:443 \
		-www / -new -time 10 -cipher $c 2>&1 | \
		grep bytes
	echo
done


＃ 在一台主机上创建服务端，默认使用4433端口
openssl s_server -cert mycert.pem -www

# 在另一台主机上(可以跟服务端同一台)，运行s_time
openssl s_time -connect myhost:4433 -www / -new -ssl3
```
### 操作 pfx 文件
```
openssl pkcs12 -in ClientCert.pfx -nocerts -nodes -out cms.key
openssl pkcs12 -in ClientCert.pfx -clcerts -nokeys -out cms.crt

当不使用证书时：
1e8a2045-7eb3-458c-95f1-ebe46da1574f
```

### 实战
其它openssl 命令：
```
导出一个网站的公钥：
openssl s_client -connect 10.18.172.216:44300 -showcerts < /dev/null | openssl x509 -outform pem > cms_cert.pem

查看公钥的信息：
openssl x509 -in cms_cert.pem -noout -text

查看证书指纹，用于与keytool输出指纹比对：
openssl x509 -in cert.pem -noout -fingerprint -sha256

用CA证书验证某个证书的合法性：
sudo openssl verify -CAfile ~/Desktop/ISRGRootX1.pem ~/Desktop/R3.cer

测试 https 的性能：
openssl s_time -time 60 -connect appapi.demo.com:443

openssl s_client -crlf -connect ttl-api-uat.demo.net:443 -debug | ts
openssl s_client -crlf -connect ttl-api-uat.demo.net:443 -msg

openssl s_client -crlf -connect 172.80.231.216:44500 -debug
openssl s_client -crlf -connect 172.80.231.216:44500 -msg
```
