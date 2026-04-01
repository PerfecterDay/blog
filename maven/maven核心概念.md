#  maven 核心概念
{docsify-updated}

Maven 是一套项目管理该工具，用于管理项目的开发、测试、打包、部署等。  
Maven 约定项目的主代码位于 src/main/java 目录，测试代码位于 src/test/java 目录；测试代码只会在运行测试时才会用到，打包时不会打包。

## Maven 坐标与依赖配置
Maven定义了一组规则：任何一个构件都可以使用Maven坐标唯一标识，Maven坐标的元素包括：（groupId、artifactId、version，必须定义）、packaging（可选）、classifier（不能直接定义，由插件帮助生成）。

### 依赖范围
Maven 在编译主代码时会使用一套 classpath1 ，在编译和运行测试代码时，会用到另外一套 classpath2 ，最后，打包（运行）Maven 项目时又是另外一套classpath3 。 实际上 Maven 依赖的 scope 就是用来控制 Maven 依赖与这三种 classpath 的关系的，Scope可以取如下值：
+ compile : 默认值，会把依赖加入到上述三种 classpath 中
+ test : 只在编译、执行测试代码时有效，即上述第二种 classpath2
+ provided : 对于编译和测试时有效，但是运行时无效（不会打包），即会加到 classpath1/classpath2 中
+ runtime : 对于测试和运行 classpath 有效，编译主代码时无效
+ system : 系统依赖范围，与 provided 依赖范围完全一致
+ import : 只有当依赖的pom是 pom 类型，才能在 `dependencyManagement` 部分引入该依赖，代表

<center>
<img src="pics/maven-scope.png" alt="" width=60%>
</center>

#### 依赖范围与传递性依赖的依赖范围
A依赖B，B依赖C，则A依赖C；我们称 A与B是第一直接依赖，B与C是第二直接依赖， A与C是传递性依赖.

当直接依赖与间接依赖的scope 不同时，最终的scope是啥呢？举个例子，你的项目依赖A，scope 是compile，A依赖B的scope 是 runtime，那么B在你的项目中的scope 是什么呢？
如下图所示，第一列是直接依赖的 scope，第一行代表间接依赖的 scope，表中各个单元格代表最终的scope。

|     |  compile   |  provided   | runtime    |   test  |
| --- | --- | --- | --- | --- |
|  compile   | compile    |  -   | runtime    |  -   |
|  provided   |  provided   | -    |  provided   |  -   |
|  runtime   |  runtime   | -    |  runtime   |  -   |
|  test   |  test   |  -   | test    |  -   |


#### 依赖调解
当依赖冲突时，例如：
A->B->C->X(1.0)/A->D->X(2.0)
依赖调解第一原则：路径最近者优先。上例中 X(1.0) 路径长为3，而X(2.0)长为2，所以X(2.0)会被解析引用。
A-B->Y(1.0)/A->C->Y(2.0)
依赖路径长途相同时，要靠依赖调解第二原则：第一声明优先。B先声明，则用Y(1.0)；C先声明的话就用Y(2.0)。

#### 可选依赖
A->B/B->X(optional)/B->Y(optional): A不会依赖 X和Y。

#### 排除依赖
当我们想去掉某些传递性依赖时，可以使用 exclusions :
```
<dependency>
    <groupId>com.ebayinc.platform.mayfly</groupId>
    <artifactId>mayfly-client</artifactId>
    <exclusions>
        <exclusion>
            <artifactId>uKernelCore</artifactId>
            <groupId>com.ebay.kernel</groupId>
        </exclusion>
        <exclusion>
            <artifactId>netty-common</artifactId>
            <groupId>io.netty</groupId>
        </exclusion>
    </exclusions>
</dependency>
```
exclusion 中只需要指出 groupId 和 artifactId 。

### 仓库与镜像
<center>
<img src="pics/maven-repo.png" alt="" width=40%>
</center>

Maven 会去仓库中查找依赖，优先从本地库查找，本地库找不到就会去远程仓库查找，Maven 在 super pom 中为我们配置了 `central` 库，我们也可以配置自定义的远程仓库。仓库通过 id 来标识，我们配置镜像、服务器认证时都通过 id 关联到对应的仓库。

在 pom 文件中，使用 `repositories > repository` 元素配置远程仓库，id为 `central` 是中央仓库；如果仓库需要认证，使用 `server` 元素配置认证信息， `repository` 与 `server` 之间通过id关联，即 `server` 配置的是 id 相同的 `repository` 的认证信息。另外认证信息不能配置在 pom 文件中，必须在 settings 文件中配置。

每个maven项目都会继承一个maven自带的 super pom，在super pom 中会配置了默认的中央仓库：
```
<repositories>
	<repository>
		<id>central</id>
		<name>Maven Repository Switchboard</name>
		<url>http://repol.maven.org/maven2</url>
		<layout>default</layout>
		<snapshots>
			<enabled>false</enabled>
		</snapshots>
	</repository>
</repositories>
```
如果项目需要其他的私有仓库配置，只需要在pom中添加类似的 `repositories > repository` 即可。

maven 检索、下载依赖包的顺序：
<center><img src="pics/maven-repos.png" width="50%"></center>

#### 部署jar包到私服仓库 
```
项目POM:
<distributionManagement>
	<repository>
		<id>nexus</id>
		<name>Releases</name>
		<url>http://localhost:8081/repository/local-maven-repo/</url>
	</repository>
	<snapshotRepository>
		<id>nexus</id>
		<name>Snapshot</name>
		<url>http://localhost:8081/repository/local-maven-repo/</url>
	</snapshotRepository>
</distributionManagement>

settings.xml:
<servers>
	<server>
		<id>nexus</id>
		<username>admin</username>
      	<password>admin</password>
	  </server>
</servers>
```
往远程仓库部署构件的时候，往往需要认证。就需要在 settings. xml 中创建一个 `server` 元素，其 id 与仓库的 id 匹配，并配置正确的认证信息。不论从远程仓库下载构件，还是部署构件至远程仓库，当需要认证的时候，配置的方式是一样的。

配置正确后，运行 `mvn clean deploy` 即可部署到仓库。

#### 镜像
如果仓库 X 可以提供仓库 Y 存储的所有内容，那么就可以认为 X 是 Y 的一个镜像。换句话说，任何一个可以从仓库 Y 获得的构件，都能够从它的镜像中获取。举个例子， http://maven.oschina.net/content/groups/public/ 是中央仓库 http://repo1.maven.org/maven2/ 在中国的镜像，由于地理位置的因素，该镜像往往能够提供比中央仓库更快的服务。因此，可以配置 Maven 使用该镜像来替代中央仓库。编辑 settings.xml:
```
<mirror>
	<id>nexus-osc</id>
	<mirrorOf>central</mirrorOf>
	<name>Nexus osc</name>
	<url>http://maven.oschina.net/content/groups/public/</url>
</mirror>
```

为了满足一些复杂的需求， Maven 还支持更高级的镜像配置：
+ `<mirrorOf>*</mirrorOf>`：匹配所有远程仓库。
+ `<mirrorOf>external.*</mirrorOf>`：匹配所有远程仓库，使用 localhost 的除外，使用 file://协议的除外。也就是说，匹配所有不在本机上的远程仓库。
+ `<mirrorOf>repo1,rep02</mirrorOf>`：匹配仓库 repo1 和 repo2 ，使用逗号分隔多个远程仓库。
+ `<mirrorOf>*,!repo1</mirrorOf>`：匹配所有远程仓库，但 repo1 除外，使用感叹号将仓库从匹配中排除。

需要注意的是，由于镜像仓库完全屏蔽了被镜像仓库，当镜像仓库不稳定或者停止服务的时候， Maven 仍将无法访问被镜像仓库，因而将无法下载构件。

#### 从仓库解析依赖的机制
1. 当依赖范围是 system 的时候， Maven 直接从本地文件系统解析构件
2. 根据依赖坐标计算仓库路径后，尝试直接从本地仓库寻找构件，如果发现相应构件，则解析成功
3. 在在本地仓库不存在相应构件的情况下，如果依赖的版本是显示的发布版本（非SNAPSHOT/RELEASE/LATEST）构件，则遍历所有的远程仓库，发现后，下载并解析使用。
4. 如果依赖的版本是 RELEASE 或者 LATEST，则基于更新策略读取所有远程仓库的元数据 `groupid/artifactid/maven-metadata.xml`，将其与本地仓库的对应元数据合并后，计算出 RELEASE 或者 LATEST 真实的值，然后基于这个真实的值检查本地和远程仓库。
5. 如果依赖版本是 SNAPSHOT ，则基于更新策略(总是更新、每日更新、每周更新)读取所有远程仓库的元数据 `groupid/artifactid/maven-metadata.xml`，将其与本地仓库的对应元数据合并后，得到**最新快照**版本的值，然后基于该值检查本地仓库，或者从远程仓库下载。
6. 如果解析得到的构件版本是时间戳格式的快照，如1.4.1-20201103.132350-123，则复制其时间戳格式的文件至非时间戳格式，如 SNAPSHOT，并使用该非时间戳格式的构件。

### 聚合与继承

#### 聚合(多模块)
当我们的两个Maven项目是相关的，共同作用以构成一个更大的项目，我们可以把他们称为更大项目的模块。如 regs-api/regs-serve/regs-ft。一个简单的需求就是：我们想要一次构建两个模块而不是分别进入到不同模块目录下执行 mvn 命令。Maven 聚合就是为这一需求服务的。    
首先，需要创建另一个模块，并且 packaging 类型必须是 POM ，然后，在 pom 文件中配置 modules，**每个 module 的值都是一个相对于当前 pom 的目录，该目录下包含一个maven 项目（pom.xml）**。聚合模块通常只包含一个 pom.xml 文件，没有源码资源等目录。

#### 继承
当多个模块之间有许多重复的配置时，可以将他们抽取出来作为一个父模块供这些模块继承共享。父模块的打包类型也必须是 pom，而且不需要源代码之类的目录。子模块继承时，使用 parent 元素指定父模块， **parent下的 `groupId、artifactId、version` 指定了父模块的坐标， `relativePath` 指定父模块 pom 文件相对与本 pom 的路径，默认值是 `../pom.xml` ，即上层目录中的 `pom.xml` ，maven 会首先根据 `relativePath` 查找父 POM， 如果找不到，再从本地/远程仓库查找 。如果使用 `<relativePath/>` 会直接在本地仓库中查找，找不到就去远程仓库查找**

父 pom 中一些配置元素是可以被继承的，下边是一个完整的列表：
+ `groupId` ：项目组 ID ，项目坐标的核心元素；  
+ `version` ：项目版本，项目坐标的核心元素；  
+ `description` ：项目的描述信息；  
+ `organization` ：项目的组织信息；  
+ `inceptionYear` ：项目的创始年份；  
+ `url` ：项目的 url 地址  
+ `develoers` ：项目的开发者信息；  
+ `contributors` ：项目的贡献者信息；  
+ `distributionManagerment` ：项目的部署信息；  
+ `issueManagement` ：缺陷跟踪系统信息；  
+ `ciManagement` ：项目的持续继承信息；  
+ `scm` ：项目的版本控制信息；  
+ `mailingListserv` ：项目的邮件列表信息；  
+ `properties` ：自定义的 Maven 属性；  
+ `dependencies` ：项目的依赖配置；  
+ `dependencyManagement` ：项目的依赖管理配置；  
+ `repositories` ：项目的仓库配置；  
+ `build` ：包括项目的源码目录配置、输出目录配置、插件配置、插件管理配置等；  
+ `reporting` ：包括项目的报告输出目录配置、报告插件配置等。 

##### 继承下的依赖
当模块A和B都依赖了一些共同的 jar，可以将这些共同的依赖放到父模块的依赖配置下，这样子模块就不用添加这些配置直接继承即可。但是，这样的话，以后新加的子模块C都会依赖这些A和B的依赖，有可能对C是没用的。    
可以使用 `dependencyManagement` 统一管理这些共同依赖， dependencyManagement 下的依赖不会引入实际的依赖（打包不打），但是可以为一些依赖项配置好版本，这样子元素要使用某一项依赖时，只需要直接添加 groupId、artifactId 的 dependency，版本就是父模块中的版本，当然也可以写上 version ，这样会使用子模块中的版本（类似覆盖）。与 dependencyManagement 还有 pluginManagement 。

#### 聚合与继承的关系
聚合与继承的目的是完全不同的，聚合是为了方便快速的构建多个项目，后者主要是为了消除重复的配置。两者其实没有什么关系，如果非要说两者的共同点，那就是两者的 packaging 类型都必须是 pom ,同时，聚合模块与父模块中除了 pom.xml 文件没有其他内容。
但是实际使用中，大多数 POM 可以既是父 POM 又是聚合 POM 。

### Maven 测试
Maven 测试主要是通过 maven-surefire-plugin 插件来完成，它能很好的支持 Junit 和 TestNG 测试框架。Maven test 阶段被定义为“使用测试框架完成测试”，默认情况下，正是与 maven-surefire-plugin 的 test 目标相绑定的。默认情况下，maven-surefire-plugin test 目标会自动执行测试源码路径下（src/test/java/）下所有符合下述命名模式的类：
1. **/Test\*.java :任何子目录下以Test开头的Java类
2. **/*Test.java :任何子目录下以Test结尾的Java类
3. **/\*TestCase.java :任何子目录下以TestCase结尾的Java类
只要以上述模式命名测试类，Maven 就能自动运行它们。  
Maven 也支持通过命令行动态指定要运行的某个或某几个测试用例：通过 test 参数指定。
+ mvn test -Dtest=RandomGeneratorTest
+ mvn test -Dtest=Random*Test
+ mvn test -Dtest=Random*Test,OnceTest

符合上述命名模式的Java测试用例会被自动执行，另外，还可以通过配置来包含或者排除某些测试用例：
```
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.6</version>
    <configuration>
        <useFile>false</useFile>
        <includes>
            <!-- This is not a mistake. Has to be .java even though its a groovy file. Without it, tests won't run.-->
            <include>**/*Spec.java</include>
            <include>**/*Tests.java</include>
        </includes>
        <excludes>
            <exclude>RandomGeneratorTest</exclude>
        </excludes>
        <skipTests>true/false</skipTests>
        <testFailureIgnore>true</testFailureIgnore>
    </configuration>
</plugin>
```
如果是 TestNG 测试，则可以在目录下创建一个称为 suite 的 xml 文件来配置测试用例的运行，具体配置要看 TestNG 的文档。另外，还需要在 maven-surefire-plugin 插件中配置使用该 suite 文件来运行：
```
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.6</version>
    <configuration>
      <suiteXmlFile>testng.xml</suiteXmlFile>
      <groups>group1,group2<groups>
    </configuration>
</plugin>
```
也可以用 mvn test -DsuiteXmlFile=testng.xml 参数在命令行中指定 suite 配置文件。
