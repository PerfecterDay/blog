#  maven 插件
{docsify-updated}

- [maven 插件](#maven-插件)
  - [生命周期](#生命周期)
    - [插件及绑定](#插件及绑定)
  - [Maven 插件开发](#maven-插件开发)
    - [Maven 插件开发的一般步骤](#maven-插件开发的一般步骤)
    - [调试自己开发的 Maven 插件](#调试自己开发的-maven-插件)
  - [常用的插件](#常用的插件)
    - [IDEA 集成 mvnd](#idea-集成-mvnd)

## 生命周期
在 Maven 出现之前，项目构建的生命周期就已经存在了，软件开发人员会对项目进行清理、编译、测试及部署。但是不同的开发人员、不同的公司以及不同的项目之间，往往使用不同的方式做类似的工作。造成了大量的重复劳动。

**Maven 的生命周期就是为了所有的构建过程进行抽象和统一**。Maven 从大量的项目和构建工具中学习和反思，然后总结了一套高度完善、以扩展的生命周期。这个生命周期包含了项目的清理、初始化、编译、测试、打包、集成测试、验证、部署和站点生成等几乎所有的构建步骤。Maven 定义的生命周期和插件机制一方面保证了所有Maven项目有一致的构建标准，另一方面又通过默认的插件简化和稳定了实际项目的构建。另外，用户还可以通过配置或自行编写插件来自定义构建行为。

三套相互独立的生命周期： 
+ clean: 清理项目，包含 pre-calen 、 clean 、 post-clean 三个阶段
+ default: 构建项目，阶段很多
+ site: 建立项目站点，pre-site 、 site 、 post-site 、 site-deploy 四个阶段

**每个生命周期包含一些阶段，这些阶段是有顺序的，并且后边的阶段会依赖前边的阶段，因此执行后边的阶段，前边依赖的阶段也会执行**，用户和Maven最直接的交互方式就是调用这些生命周期的阶段。   
三套生命周期之间是**相互独立**的，也就是说调用不同生命周期中的阶段不会对其它生命周期有影响，更不会执行其它生命周期内的阶段。

### 插件及绑定
Maven的生命周期是抽象的，**实际的任务都由插件来完成。每个构建步骤（phase）都可以绑定一个或者多个插件目标（goal），而且 Maven 为大多数的构建步骤编写并绑定了默认插件。**
插件一般会提供多种功能，而不是每一个功能都用一个新的插件来完成，同一个插件的不同功能叫做目标(goal)。比如 `maven-dependency-plugin` 有十多个目标，常用的目标有 `dependency:analyze` 、`dependency:tree` 、 `dependency:list` 等。  

下面是default生命周期中各个阶段与默认提供的插件的目标之间的绑定关系：
<center>
<img src="pics/mvn-goal.png" alt="" width=60%>
</center>

当默认的插件不能满足项目的需求时，需要引用第三方插件或者自行编写插件来完成功能。
可以在 pom 的 `build>plugins>plugin>executions>execution` 下为为项目绑定某个插件目标到特定的生命周期阶段上。当多个目标绑定到同一个阶段时，插件的**声明顺序**会决定目标执行的先后顺序。

```
<plugins>
    <plugin>
        <groupId>com.coder.wang</groupId>
        <artifactId>maven-count-plugin</artifactId>
        <version>1.0-SNAPSHOT</version>
        <executions>
            <execution>
                <phase>compile</phase>
                <goals>
                    <goal>count</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
</plugins>
```

有时候，不用配置 `phase` 元素，插件目标也会绑定到特定的阶段上，这是因为插件在开发时已经定义了默认的绑定阶段。

```mvn [options] <plugin:goal> <phase>```
Maven不仅可以执行生命周期的阶段，进而执行绑定在这些阶段上的目标，而且可以直接执行某些插件目标。

## Maven 插件开发
### Maven 插件开发的一般步骤
1. 创建一个 maven-plugin项目：使用 `maven archetype:generate` ，然后选择 maven-archetype-plugin 快速创建一个插件项目
2. 为插件编写目标：每个插件必须包含一个或多个目标，Maven称之为 Mojo，继承自 AbstractMojo 类。
3. 为目标提供配置点：在编写Mojo时，提供可配置的参数
4. 编写代码实现目标行为：根据实际需求实现Mojo
5. 错误处理及日志
6. 测试插件：编写自动化的测试代码测试插件。

Maven插件的 pom 有两个特殊的地方：
  1. packaging 类型必须是 maven-plugin
  2. 必须依赖一个 maven-plugin-api 的 artifact。
创建好插件项目之后，我们要创建一个Mojo：继承 AbstractMojo 、实现 execute() 方法、提供 @goal 标注（不是注解）。

```
/**
* @goal package
*/
public class CountMojo extends AbstractMojo{
    public void execute() throws MojoExecutionException {

    }
}
```

### 调试自己开发的 Maven 插件
1. 将自己开发的插件 mvn install 到本地
2. 在一个项目X中使用开发的插件，然后，在该项目路径下执行`mvnDebug com.ebay.raptor.build:assembler-maven-plugin:3.0.25-RELEASE:package`，注意插件的版本要一致，不然代码无法匹配，代码断点会失效；执行后会打开一个端口等待 debugger 连接 
3. 在插件开发的IDEA项目中，新建一个 remote 的 debug/run configuration，port修改为上面的port，然后在项目代码中打上断点，然后debug执行即可


## 常用的插件
+ [maven-shade-plugin](https://headcrashing.wordpress.com/2020/02/23/reducing-jar-footprint-with-maven-shade-plugin-3-2-2/):缩小包的体积
  
### IDEA 集成 mvnd
1. 安装 maven helper 插件
2. 设置 maven helper 插件是用 mvnd 运行命令
3. 不要使用右侧 IDEA 集成的 maven 栏执行 maven 操作
4. 右击项目/或者模块目录，点击 `Run Maven` 执行操作