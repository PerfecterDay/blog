# mitproxy

1. 安装： `brew install mitproxy`
2. 启动： `mitweb -p 10000`
3. 安装CA证书： `sudo security add-trusted-cert -d -p ssl -p basic -k /Library/Keychains/System.keychain ~/.mitmproxy/mitmproxy-ca-cert.pem`
