# Powershell 与 cmd

powershell 中查看环境变量
1. `ls env:`
2. `echo $env:JAVA_HOME`
3. `tree /f` : 显示目录树

### powershell 为变量设置别名
如果你想为自己的 Windows PowerShell 设置永久的命令别名 (Alias)，可以遵循以下步骤：

1. 打开 PowerShell ，运行 `echo $profile` ，会输出一个文件路径。创建这个文件。
2. 打开刚创建的文件，按以下格式设置多条别名：
 ```function 别名 { 需要替代的命令，可以包含空格 }```
 如：
 ```
 	Set-Alias docker podman
    function gst {git status}
    function grv {git remote -vv}
    function gbv {git branch -vv}
    function cdw {cd D:\workspace\}
 ```
3. 以管理员身份打开 PowerShell，执行 Set-ExecutionPolicy RemoteSigned。
4. 重新启动 PowerShell ，应该已经完成了。

cmd 中查看环境变量
1. `echo %JAVA_HOME%`

windows 的证书管理工具：
1. certlm.msc ： 本地计算机的证书管理工具
2. certmgr.msc ： 当前用户的证书管理工具