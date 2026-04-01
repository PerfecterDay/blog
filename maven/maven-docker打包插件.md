# maven docker 打包插件
{docsify-updated}


使用 `io.fabric8:docker-maven-plugin` 插件
```
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.46.0</version>
    <executions>
        <execution>
            <id>build-docker-image</id>
            <phase>package</phase> <!-- 在 package 阶段自动执行 -->
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
    </executions>
    <!--全局配置-->
    <configuration>
        <images>
            <!-- 单个镜像配置 -->
            <image>
                <!--镜像名(含版本号)-->
                <name>demo-registry-registry.cn-hongkong.cr.aliyuncs.com/demo/channel-gmt:${project.version}</name>
                <!--镜像build相关配置-->
                <build>
                    <!--使用dockerFile文件-->
                    <dockerFile>${project.basedir}/Dockerfile</dockerFile>
                    <contextDir>${project.basedir}</contextDir>
                </build>
            </image>
        </images>
    </configuration>
</plugin>
```

## 多架构问题
> https://dmp.fabric8.io/#build-buildx

增加 `buildx` 配置：
```
 <build>
    <!--使用dockerFile文件-->
    <dockerFile>${project.basedir}/Dockerfile</dockerFile>
    <contextDir>${project.basedir}</contextDir>
    <buildx>
        <platforms>
            <!-- set comma separated list of platforms in ${docker.platforms} to invoke buildkit -->
            <platform>linux/amd64</platform>
        </platforms>
    </buildx>
</build>
```


## profiles 配置
```
<profiles>
  <!-- 开发环境 -->
  <profile>
    <id>dev</id>
    <properties>
      <docker.registry>registry.dev.local</docker.registry>
      <docker.repo>dev-team</docker.repo>
    </properties>
  </profile>

  <!-- 测试环境 -->
  <profile>
    <id>test</id>
    <properties>
      <docker.registry>registry.test.example.com</docker.registry>
      <docker.repo>qa-team</docker.repo>
    </properties>
  </profile>

  <!-- 生产环境 -->
  <profile>
    <id>prod</id>
    <properties>
      <docker.registry>registry.prod.example.com</docker.registry>
      <docker.repo>mycompany</docker.repo>
    </properties>
  </profile>
</profiles>
```

执行mvn 命令时使用 `-P{profile}` 指定使用的 profile：
```
mvn clean package docker:build docker:push -Ptest
mvn clean package docker:build docker:push -Pdev
```