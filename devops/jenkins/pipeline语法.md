# Pipeline 语法
{docsify-updated}

- [Pipeline 语法](#pipeline-语法)
    - [字符串插值](#字符串插值)
    - [环境变量](#环境变量)
    - [Agent](#agent)
    - [实例](#实例)


所有有效的声明式流水线必须包含在一个 pipeline 块中, 比如:
```
pipeline {
    /* insert Declarative Pipeline here */
}
```

### 字符串插值
尽管 Groovy 支持使用单引号或双引号声明一个字符串，但是只有双引号的字符串才支持基于美元符（$）的字符串插值，例如：
```
def username = 'Jenkins'
echo 'Hello Mr. ${username}'
echo "I said, Hello Mr. ${username}"
```
结果是：
```
Hello Mr. ${username}
I said, Hello Mr. Jenkins
```

### 环境变量
Jenkins 流水线通过全局变量 env 提供环境变量，它在 Jenkinsfile 文件的任何地方都可以使用。Jenkins 流水线中可访问的完整的环境变量列表记录在 ``${YOUR_JENKINS_URL}/pipeline-syntax/globals#env``.
引用环境变量：``${env.BUILD_ID}``

设置环境变量
```
environment { 
	CC = 'clang'
}

environment {
	// 使用 returnStdout
	CC = """${sh(
			returnStdout: true,
			script: 'echo "clang"'
		)}""" 
	// 使用 returnStatus
	EXIT_STATUS = """${sh(
			returnStatus: true,
			script: 'exit 1'
		)}"""
}
```


### Agent
agent 部分指定了整个流水线或特定的部分, 将会在Jenkins环境中执行的位置，这取决于 agent 区域的位置。该部分必须在 pipeline 块的顶层被定义, 但是 stage 级别的使用是可选的。参数如下：
+ any: 在任何可用的代理上执行流水线或阶段
+ none：当在 pipeline 块的顶部没有全局代理，该参数将会被分配到整个流水线的运行中并且每个 stage 部分都需要包含他自己的 agent 部分。
+ label：在提供了标签的 Jenkins 环境中可用的代理上执行流水线或阶段。
+ docker：使用给定的容器执行流水线或阶段。
	```
	agent {
		docker {
			image 'maven:3-alpine'
			label 'my-defined-label'
			args  '-v /tmp:/tmp'
		}
	}
	```
+ dockerfile：执行流水线或阶段, 使用从源代码库包含的 Dockerfile 构建的容器。



### 实例
```
pipeline {
    parameters {
        string defaultValue:'ssh://git@demo.net:223/wangzhongzhu026484/trade-center.git', name: 'repo'
        string defaultValue: 'develop', name: 'branch'
        string defaultValue: '1.0.0', name: 'version'
        choice choices: ['sit','uat', 'prod'], name: 'env'
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
                sh "mvn -pl '!trade-center-biz' -B -DskipTests clean package"
            }
        }

        stage('CopyFile'){
            steps {
                sh "export version=${params.version}"
                sh '''
                cd docker
                rm -rf *.jar
                cp ../trade-center-service/target/*.jar .
                ls -lh
                '''
            }
        }
        stage('Docker uat build'){
            when{
                expression{
                    params.env == 'uat'
                }
            }
            steps{
                sh '''
                    cd docker
                    docker build -t demo-registry-registry.cn-hongkong.cr.aliyuncs.com/demo/trade-gmt-ttl:$version .
                    docker push demo-registry-registry.cn-hongkong.cr.aliyuncs.com/demo/trade-gmt-ttl:$version
                 '''
            }
        }
        stage('Docker prod build'){
            when{
                expression{
                    params.env == 'prod'
                }
            }
            steps{
                sh '''
                    cd docker
                    docker build -t demoprd-registry-registry.cn-hongkong.cr.aliyuncs.com/demo-prof/trade-gmt-ttl:$version .
                    docker push demoprd-registry-registry.cn-hongkong.cr.aliyuncs.com/demo-prof/trade-gmt-ttl:$version
                 '''
            }
        }
    }
    post {
        success {
            archiveArtifacts artifacts: 'trade-center-service/target/*.jar', fingerprint: true
        }
    }
}
```