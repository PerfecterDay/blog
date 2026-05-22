#  maven 常用命令参数和配置
{docsify-updated}

## maven 常用命令
0. `mvn -B archetype:generate -DgroupId=com.my -DartifactId=simple-weather -DpackageName=com.baicy -Dversion=1.0 `: 以batch模式创建新项目，会选择 Archetype ，可能需要删除 settings配置文件中 archetypeRepository 和 archetypeCatalog 配置。
1. `mvn archetype:generate -DgroupId=com.coder.wang -DartifactId=maven-count-plugin -DarchetypeArtifactId=maven-archetype-plugin -DarchetypeVersion=3.0 `:使用指定的 Archetype 生成项目。
2. ` mvn dependency:list` ：输出依赖所有依赖的包
3. ` mvn dependency:tree -Dverbose` ：输出依赖树，解决依赖冲突时有用
4. ` mvn dependency:analyze` ：分析依赖树
5. `-U` :强制更新本地库
6. `mvn clean cobertura:cobertura -Dcobertura.report.format=html`:生成html格式的测试覆盖率报告
7. `mvn help:effective-pom`：生成完整的 POM 文件，包含 super pom
8. `mvn help:effective-settings`: 查看当前 maven 生效的设置
9. 安装本地JAR包到本地仓库：`mvn install:install-file -Dfile="D:\workspace\trade\trade-center\trade-center-service\lib\ZHConverter.jar" -DgroupId="com.spreada.utils.chinese" -DartifactId="ZHConverter" -Dversion="1.0" -Dpackaging="jar"`
10. `mvn package -DskipTests`: 跳过测试运行
11. `mvn package -Dmaven.test.skip=true`：跳过测试代码编译同时跳过测试运行
12. `mvn test -Dtest=testClass1,testClass2|Random*Test`：运行指定的测试类（|代表或）
13. `mvn -pl '!submodule1,!submodule2' install 或者 mvn -pl -submodule1,-submodule2 install` ：打包时跳过某些模块
14. `mvn dependency:sources`: 下载依赖包的源码文件
15. `mvn dependency:resolve -Dclassifier=javadoc`: 下载依赖包的 javadoc
16. `mvn dependency:sources dependency:resolve -Dclassifier=javadoc` ：同时下载以来包的javadoc和源码文件

## maven 常用配置
最佳实践是将 maven 安装目录下的 `conf/settings.xml` 文件复制到用户目录下的 `.m2` 文件夹中，并修改其中的配置。
1. 修改本地仓库路径: `<localRepository>D:/repository</localRepository>`
2. 修改远程仓库镜像：
  ```
    <mirror>
    <id>aliyun</id>
    <name>aliyun Maven</name>
    <mirrorOf>*</mirrorOf>
    <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    <!-- <url>http://maven.oschina.net/content/groups/public</url> -->
    </mirror>
  ```
3. 配置Java的版本
```
<properties>
    <maven.compiler.target>1.8</maven.compiler.target>
    <maven.compiler.source>1.8</maven.compiler.source>
</properties>
或者
<plugins>
    <plugin>    
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
            <source>1.8</source>
            <target>1.8</target>
        </configuration>
    </plugin>
</plugins>
```

4. clean 时清除指定额外文件夹
```
<build>
    <plugins>
      <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-clean-plugin</artifactId>
          <version>3.3.2</version>
          <configuration>
              <filesets>
                  <fileset>
                      <directory>${project.basedir}/logs</directory>
                  </fileset>
                  <fileset>
                      <directory>${project.basedir}</directory>
                      <includes>
                          <include>cap.pid</include>
                      </includes>
                  </fileset>
              </filesets>
          </configuration>
      </plugin>
    </plugins>
  </build>
  ```

## maven 问题
1. 有时候使用 https 时会遇到如下错误：`sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target`

解决方法：加上 `-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true`

2. 如何将一个 lib 引入到项目中并打进jar 包内 ？[参考](https://stackoverflow.com/questions/364114/can-i-add-jars-to-maven-2-build-classpath-without-installing-them)
在pom 项目中引入一个开发项目所在路径的一个本地 repo。假如，特殊jar 包在项目的 libs 文件夹下，那么:
   1. 第一步安装jar 文件到项目目录中： `mvn install:install-file -DlocalRepositoryPath=lib -DcreateChecksum=true -Dpackaging=jar -Dfile=lib/ZHConverter.jar -DgroupId=com.demo.demo -DartifactId=ZHConverter -Dversion=1.0 -DgeneratePom=true`
   2. 在pom 中添加本地仓库配置:
      ```
      <repository>
        <id>ProjectRepo</id>
        <name>ProjectRepo</name>
        <url>file://${project.basedir}/lib</url>
      </repository>
      ```
   3. 添加依赖：
      ```
      <dependency>
       <groupId>com.demo.demo</groupId>
       <artifactId>ZHConverter</artifactId>
       <version>1.0</version>
      </dependency>
      ```
    4. 注意本地repo 的id 不能被 mirro 挡掉，否则会下载不到依赖
     ```
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*,!ProjectRepo</mirrorOf>
        <name>阿里云公共仓库</name>
        <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
    ```

3. spring-boot-maven-plugin 打的 jar 包 不能被依赖  
spring-boot-maven-plugin 在打包时，会把依赖放到 BOOT-INF/lib/ 里，把你的 classes 放到 BOOT-INF/classes/，并加上一个特殊的启动器类（org.springframework.boot.loader.JarLauncher 等）。

这种结构是为了用 java -jar 启动，而不是为了类路径加载。
Maven/Java 的普通类加载器找不到 BOOT-INF/classes 里的类，所以作为依赖引用时会报 ClassNotFoundException。