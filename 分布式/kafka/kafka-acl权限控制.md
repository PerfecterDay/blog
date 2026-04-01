# kafka-acl
{docsify-updated}


在 Apache Kafka 中，对 topic 的权限配置通常通过 Kafka 的 ACL（Access Control List）机制 来实现。以下是配置 Kafka topic 权限的基本步骤：
⸻
🛠️ 前提条件
	1.	Kafka 开启了安全认证机制（如 SASL、SSL 等）。
	2.	启用了 Authorizer（权限控制器），通常是默认的 kafka.security.auth.SimpleAclAuthorizer。
⸻
⚙️ 配置文件中的关键参数

确保在 server.properties 中启用了 ACL 支持：
```
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false
```

🔒 Kafka ACL 权限模型概览

Kafka 的权限控制是基于 资源 + 操作 + 用户/客户端 的模型，资源可以是：
+ Topic
+ Group
+ Cluster
+ TransactionalId

常用操作有：
+ Read
+ Write
+ Describe
+ Create
+ Delete

⸻

✅ 配置权限的常用命令
使用 Kafka 自带的命令行工具 kafka-acls.sh 来添加、查看和删除 ACL。
1. 给用户添加写权限到某个 topic：
   ```
   bin/kafka-acls.sh \
    --authorizer-properties zookeeper.connect=localhost:2181 \
    --add \
    --allow-principal User:alice \
    --operation Write \
    --topic my-topic
  ```
2. 添加读权限：
    ```
    bin/kafka-acls.sh \
     --authorizer-properties zookeeper.connect=localhost:2181 \
     --add \
     --allow-principal User:alice \
     --operation Read \
     --topic my-topic
    ```
3. 查看已有的权限：
    ```
    bin/kafka-acls.sh \
     --authorizer-properties zookeeper.connect=localhost:2181 \
     --list \
     --topic my-topic
    ```
4. 删除 ACL：
    ```
    bin/kafka-acls.sh \
     --authorizer-properties zookeeper.connect=localhost:2181 \
     --remove \
     --allow-principal User:alice \
     --operation Read \
     --topic my-topic
    ```