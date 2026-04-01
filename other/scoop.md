# Scoop 
{docsify-updated}

### Scoop 安装
powershell中运行以下命令：
1. 安装到默认目录：
    ```
    Invoke-Expression (New-Object System.Net.WebClient).DownloadString('https://get.scoop.sh')
    # or shorter
    iwr -useb get.scoop.sh | iex
    ```
    默认安装目录是 C:\Users\<user>\scoop. 全局安装的软件 (使用 --global 安装时) 安装到： C:\ProgramData\scoop
2. 安装到自定义目录：
   ```
    $env:SCOOP='D:\applications\scoop'
    $env:SCOOP_GLOBAL='D:\applications\globalScoopApps'
    [Environment]::SetEnvironmentVariable('SCOOP', $env:SCOOP, 'User')
    [Environment]::SetEnvironmentVariable('SCOOP_GLOBAL', $env:SCOOP_GLOBAL, 'Machine')
    # run the installer
   ```

### 安装配置 aria2
```
scoop install aria2

scoop config aria2-enabled false
scoop config aria2-warning-enabled false
```
PowerShell requires an execution policy in [Unrestricted, RemoteSigned, ByPass] to run Scoop. For example, to set the execution policy to 'RemoteSigned' please run :  
`Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`.

下载文件：
`aria2c -x 8 https://download.pytorch.org/whl/cu117/torch-1.13.0%2Bcu117-cp310-cp310-linux_x86_64.whl#sha256=7f013d8097cb3741ac8d6745a63ef3c945df9be40514fbc026409261c234c2e2`


### 更换国内源
1. 更换scoop自身源：`scoop config SCOOP_REPO http://mirror.repo`
2. 针对不同的bucket，进入到相应的 bucket 目录（比如$SCOOP/buckets/main），执行：`git remote set-url origin http://mirror.repo`

推荐源：https://codechina.csdn.net/mirrors 或者  https://hub.fastgit.org/。  
示例： https://github.com/Ash258/Scoop-Sysinternals.git/ ->  https://hub.fastgit.org/Ash258/Scoop-Sysinternals.git

### 使用代理
`scoop config proxy proxy.example.org:8080` ： 配置代理
`scoop config proxy localhost:10809` ：配置代理
`scoop config rm proxy` : 删除代理

### 常用命令
1. `scoop search program-name` : 查找软件
2. `scoop install program-name` : 安装软件
3. `scoop config aria2-options --check-certificate=false` ： 解决“由于吊销服务器已脱机，吊销功能无法检查”错误