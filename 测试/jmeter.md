#  Jmeter 入门
{docsify-updated}

> https://www.elibaron.com/develop/test/dev-y-pu-jmeter-install.html

## 安装遇到的问题
1. Jmeter响应中的中文乱码：  
	添加 BeanShell 后置处理程序，添加如下脚本：
	```
	prev.setDataEncoding("utf-8")
	```
2. 命令行脚本运行后，不停止运行bug -> https://github.com/apache/jmeter/issues/6008
	手动在测试计划的 jmx 文件中，在每个 `ThreadGroup.main_controller`（如果有多个线程组会有多个）元素下添加 `<boolProp name="LoopController.continue_forever">false</boolProp>`

## 命令行参数
`jmeter -n -t [jmx file] -l [results file] -e -o [Path to web report folder]`

jmeter -n -t ./trade2.jmx -l result.jtl3.10.1  -e -o ./trade

`JVM_ARGS="-Xms1024m -Xmx1024m"` 设置JVM参数，会覆盖配置文件中的配置

+ `-n` ：这指定了JMeter在cli模式下运行
+ `-t` :包含测试计划的JMX文件名称]。
+ `-l` : 记录样本结果的JTL文件的名称。
+ `-j JMeter运行日志文件的名称`: 指定日志文件。
+ `-r` : 在由JMeter属性 "remote_hosts "指定的服务器中运行测试
+ `-R 远程服务器的列表` 在指定的远程服务器中运行测试
+ `-g CSV文件的路径` 仅生成报告仪表板
+ `-e`: 在负载测试后生成报告仪表板
+ `-o`: 负载测试后生成报告仪表板的输出文件夹。文件夹必须不存在或为空

该脚本还允许你指定可选的防火墙/代理服务器信息：
+ `-H`: [代理服务器的主机名或ip地址]
+ `-P`: [代理服务器端口］

## 阶梯式压测线程组
1. 点击选项，下面的Plugins Manager，搜索Custom Threads Group，安装。
2. 在测试计划下面添加线程，选择jp@gc-Stepping Thread Group线程组。