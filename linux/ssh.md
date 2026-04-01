#  SSH 简介
{docsify-updated}

- [SSH 简介](#ssh-简介)
	- [SSH免密登录](#ssh免密登录)


移步至阮一峰老师的[博客](https://wangdoc.com/ssh/client.html)

### SSH免密登录
1. `ssh-keygen` ： 生成公私钥对
2. `ssh-copy-id 10.4.153.130` : 会在目标登录机器上的相应用户的 .ssh 文件夹下的authorized_keys文件中写入相应的公钥信息

```
[root@LXUATGMAA2 ~]# cat ~/.ssh/authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC7f4fSmAsItzNsYlVbuyqkKOV2K2Qx/v0gZEmlirmuR0umUi24Y2bpXpeKODQTvP8uM+WUmgeJUEPH1XFV26wcwXAhDBcCmxBL/aSBAIDcHccSM26qPlqltIf8N2gPAAx/7uhrEziOxmJB2TElpnKzo3GMLPt/v8gBBiAARO8LVGPo8KODoAWv9z/nXYlS1wk+9UnlomGsseEciQUOro/SgmMRynFvfA0MdfV3CCfA+/ziQHp0HQvbeeFl6MY260dOJ5D/cwmXxlYWpab+3lMwk/D18GJ0IzvpRyhLEUzUoxbtjkFr7HAkCXWdJmLkv3IRWYSB5E0EbPy4JhhRyEUCpHH2zYm4kdJ2RPFDtUjpB0KF0jQq+j4bvwBLypQg6rGmD14jzLjuVm4T8uSs0JKYO3Dx2Xc9AZMo2/M+Nam9Tk5q5PBzlsiMOKRs2LbIOlLimZ7gxYjxbmkCWHgnuN/EDBf8G+SQiMkrP242NR9qeswe2zRxdU6ZCXmuxqgYgS0= root@LXUATGMAA3.demo.COM.UAT
```

### ssh 配置文件
ssh程序可以从以下途径获取配置参数：

+ 命令行选项
+ 用户配置文件 (~/.ssh/config)
+ 系统配置文件 (/etc/ssh/ssh_config)

> https://daemon369.github.io/ssh/2015/03/21/using-ssh-config-file


### .netrc 配置文件
用于配置网络登录帐号信息的 ~/.netrc 文件,保存用户名密码，减少输用户名密码。

文件 ~/.netrc 用于设置自动登录时所需要的帐号信息。
```
machine  your-git-server
login   your-username
password   your-password
```
如果有多个 server 就重复上面的三行， 分别输入对应的服务器、 用户名和密码即可.
```
machine kekxv.github.io login username password passwd
machine kekxv2.github.io login username2 password passwd2
default login username password passwd
```

netrc 文件可以用于下列程序：
+ curl
+ ftp
+ git