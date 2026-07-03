# grpc-快速启动一个测试用的grpc服务
{docsify-updated}

> https://github.com/bavix/gripmock  
> https://gripmock.org/guide/introduction/tls.html

安装：
```
brew tap gripmock/tap
brew trust --cask gripmock/tap/gripmock
brew install --cask gripmock
```

启动 tls:
```
GRPC_TLS_CERT_FILE=/Users/demox/workspace/cap/cap-infrastructure/src/main/resources/certs/demoxidemo.net.pem \
GRPC_TLS_CA_FILE=/Users/demox/workspace/cap/cap-infrastructure/src/main/resources/certs/ca.pem \
GRPC_TLS_KEY_FILE=/Users/demox/workspace/cap/cap-infrastructure/src/main/resources/certs/demoxidemo.net.key \
GRPC_TLS_MIN_VERSION=1.2 \
gripmock /Users/demox/workspace/apps/trade-gmt/trade-center-interfaces/src/main/proto/trade-service.proto
```