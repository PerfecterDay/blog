# Jenkins 语法
{docsify-updated}
> https://www.jenkins.io/zh/doc/book/pipeline/syntax/

- [Jenkins 语法](#jenkins-语法)
    - [安装](#安装)
    - [Pipeline](#pipeline)
    - [问题](#问题)


Jenkins是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件。
Jenkins 支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序。

### 安装
`HOMEBREW_BOTTLE_DOMAIN= brew reinstall jenkins`
`vim /opt/homebrew/Cellar/jenkins/2.506/homebrew.mxcl.jenkins.plist` 修改绑定地址

### Pipeline
<center><img src="pics/realworld-pipeline-flow.png" width="60%"></center>

声明式和脚本化的流水线语法
Jenkinsfile 能使用两种语法进行编写 - 声明式和脚本化。

声明式和脚本化的流水线从根本上是不同的。 声明式流水线的是 Jenkins 流水线更近的特性:
+ 相比脚本化的流水线语法，它提供更丰富的语法特性,
+ 是为了使编写和读取流水线代码更容易而设计的。

然而，写到`Jenkinsfile`中的许多单独的语法组件(或者 "步骤"), 通常都是声明式和脚本化相结合的流水线。  
Jenkins一直允许以将自由式工作链接到一起的初级形式来执行顺序任务, [4] 流水线使这个概念成为了Jenkins的头等公民。

1. 流水线
	流水线是用户定义的一个CD流水线模型 。流水线的代码定义了整个的构建过程, 他通常包括构建, 测试和交付应用程序的阶段 。  
	另外 ， `pipeline` 块是 声明式流水线语法的关键部分.

2. 代理（agent）
	agent 部分指定了整个流水线或特定的部分, 将会在Jenkins环境中执行的位置，这取决于 agent 区域的位置。该部分必须在 pipeline 块的顶层被定义, 但是 stage 级别的使用是可选的。
	节点是一个机器 ，它是Jenkins环境的一部分 and is capable of执行流水线。  
	另外, `node`块是 脚本化流水线语法的关键部分.

3. 阶段
	stage 块定义了在整个流水线的执行任务的概念性地不同的的子集(比如 "Build", "Test" 和 "Deploy" 阶段), 它被许多插件用于可视化 或Jenkins流水线目前的 状态/进展.

4. 步骤
	本质上 ，一个单一的任务, a step 告诉Jenkins 在特定的时间点要做_what_ (或过程中的 "step")。 举个例子,要执行shell命令 ，请使用 sh 步骤: sh 'make'。当	一个插件扩展了流水线DSL,  通常意味着插件已经实现了一个新的 step。

```
pipeline {  //pipeline 块声明
    agent any  //在任何可用的代理上，执行流水线或它的任何阶段
    stages { //阶段块声明
        stage('Build') {  //定义 build 阶段
            steps { //声明 build 阶段的步骤
                // 
            }
        }
        stage('Test') {  //声明 test 阶段
            steps {
                // 
            }
        }
        stage('Deploy') { //声明 deploy 阶段
            steps {
                // 
            }
        }
    }
}
```

`pipeline` 是声明式流水线的一种特定语法，他定义了包含执行整个流水线的所有内容和指令的 "block" 。
`agent`是声明式流水线的一种特定语法，它指示 Jenkins 为整个流水线分配一个执行器 (在节点上)和工作区。
`stage` 是一个描述 stage of this Pipeline的语法块。在 Pipeline syntax 页面阅读更多有关声明式流水线语法的`stage`块的信息。如 above所述, 在脚本化流水线语法中，stage 块是可选的。
`steps` 是声明式流水线的一种特定语法，它描述了在这个 stage 中要运行的步骤。
`sh` 是一个执行给定的shell命令的流水线 step (由 Pipeline: Nodes and Processes plugin提供) 。
`junit` 是另一个聚合测试报告的流水线 step (由 JUnit plugin提供)。
`node` 是脚本化流水线的一种特定语法，它指示 Jenkins 在任何可用的代理/节点上执行流水线 (和包含在其中的任何阶段)这实际上等效于 声明式流水线特定语法的`agent`。


environment {
  test = "1"
  foo = "bar"
}


parameters {
  choice choices: ['git@test.com:demo/user-center-g/user-center-service.git', 'git@test.com:demo/trade/trade-center.git'], name: 'repo'
  string defaultValue: 'develop', name: 'branch'
  choice choices: ['uat', 'prod'], name: 'env'
}

booleanParam,

tools {
  git 'Default'
}


```
pipeline {
    parameters {
      choice choices: ['ssh://git@test.com:223/demo/user-center-g/user-center-service.git',
      'ssh://git@test.com:223/demo/trade/trade-center.git'], name: 'repo'
      string defaultValue: 'develop', name: 'branch'
      choice choices: ['uat', 'prod'], name: 'env'
    }
    tools {
      maven 'maven'
      dockerTool 'docker'
    }
    agent any
    stages {
        stage('CheckCode') {
            steps {
                git(
                    credentialsId: 'zhongzhuwang',
                    url: "${params.repo}",
                    branch: "${params.branch}",
                    changelog: true,
                    poll: true
                )
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
    }
    post {
        success {
            archiveArtifacts artifacts: 'user-center-service/target/*.jar', fingerprint: true
        }
    }
}
```

通过 archiveArtifacts 步骤和文件匹配表达式可以很容易的完成构建结果记录和存储

### 问题
mvn not found/docker not found:
需要添加
tools {
	maven 'maven'
	dockerTool 'docker'
}
**且在Jenkins全局工具配置中增加Maven和 docker 设置，docker 是包含 `bin/docker` 的父目录，如果本地安装目录是 `/usr/local/bin/docker`,那么因该填 `/usr/local/`**