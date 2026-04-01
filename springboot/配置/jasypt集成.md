# Jasypt 集成
{docsify-updated}

> https://github.com/ulisesbocchio/jasypt-spring-boot

1. 添加依赖
```
<dependency>
        <groupId>com.github.ulisesbocchio</groupId>
        <artifactId>jasypt-spring-boot-starter</artifactId>
        <version>3.0.5</version>
</dependency>
```
2. 生成加密密码，方式见下文
3. 在配置文件中使用密文密码
4. 在配置文件中配置加密使用的密码：
```
jasypt:
  encryptor:
    password: the password
```
以上方会在代码中泄漏加密密码，更好的方式是使用命令行：
```
1. java -jar target/jasypt-spring-boot-demo-0.0.1-SNAPSHOT.jar --jasypt.encryptor.password=password
2. java -Djasypt.encryptor.password=password -jar target/jasypt-spring-boot-demo-0.0.1-SNAPSHOT.jar
3. 配置文件中使用
jasypt:
    encryptor:
        password: ${JASYPT_ENCRYPTOR_PASSWORD:}
启动时：
JASYPT_ENCRYPTOR_PASSWORD=password java -jar target/jasypt-spring-boot-demo-1.5-SNAPSHOT.jar
```

## 使用 Maven plugin 生成密文
1. 添加maven plugin
```
<build>
  <plugins>
    <plugin>
      <groupId>com.github.ulisesbocchio</groupId>
      <artifactId>jasypt-maven-plugin</artifactId>
      <version>3.0.5</version>
    </plugin>
  </plugins>
</build>
```

2. 命令行为单个明文生成密文： `mvn jasypt:encrypt-value -Djasypt.encryptor.password="the password" -Djasypt.plugin.value="theValueYouWantToEncrypt"`

3. 加密文件中的所有要加密的密码
    1. 将要加密的明文用下述方式表示：
    ```
    sensitive.password=DEC(secret value)
    regular.property=example
    ```
    2. 默认加密的文件路径：`src/main/resources/application.properties`
    执行 `mvn jasypt:encrypt -Djasypt.encryptor.password="the password"`
    3. 加密指定路径下的文件：`mvn jasypt:encrypt -Djasypt.plugin.path="file:src/main/resources/configuration/dev/application.yml" -Djasypt.encryptor.password="the password"`

## How everything Works?
This will trigger some configuration to be loaded that basically does 2 things:

1. It registers a Spring post processor that decorates all PropertySource objects contained in the Spring Environment so they are "encryption aware" and detect when properties are encrypted following jasypt's property convention.
2. It defines a default StringEncryptor that can be configured through regular properties, system properties, or command line arguments.

## 使用自己的自定义加密器
```
@Bean("jasyptStringEncryptor")
public StringEncryptor stringEncryptor() {
    PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
    SimpleStringPBEConfig config = new SimpleStringPBEConfig();
    config.setPassword("password");
    config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
    config.setKeyObtentionIterations("1000");
    config.setPoolSize("1");
    config.setProviderName("SunJCE");
    config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
    config.setIvGeneratorClassName("org.jasypt.iv.RandomIvGenerator");
    config.setStringOutputType("base64");
    encryptor.setConfig(config);
    return encryptor;
}
```
