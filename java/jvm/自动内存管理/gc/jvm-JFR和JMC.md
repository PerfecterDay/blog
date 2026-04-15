# JFR和JMC
{docsify-updated}

## JFR
```
jcmd process_id JFR.start duration=100s filename=flight.jfr
```


## JMC
现在大多数 JDK 版本里已经“不再默认自带” Java Mission Control（JMC）了，需要单独安装。
安装：
```
brew install --cask jdk-mission-control
```

> https://www.oracle.com/java/technologies/javase/jmc9-install.html

如果安装了多个 JDK 版本，建议使用最新版本的 JDK 运行 JMC。 编辑 JMC 启动配置 (jmc.ini) 文件并添加要使用的 JDK 版本的位置（要求 JDK 17 或更高版本）。 在 Windows 和 Linux 中，jmc.ini 文件位于 JDK Mission Control 目录下；在 macOS 中，位于 `JDK\ Mission\ Control.app/Contents/Eclipse` 目录下。 添加 `-vm` 标志和 `<JDK 安装路径>/bin（windows 中为 <JDK 安装路径>/bin）`，如下例所示。 确保添加在 `-vmargs` 标志之前。

启动时使用指定的 JDK: 
```
-vm
/opt/homebrew/opt/sdkman-cli/libexec/candidates/java/17.0.13.fx-zulu/bin/
-vmargs
```

