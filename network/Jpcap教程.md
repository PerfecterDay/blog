#  Jpcap教程
{docsify-updated}

### Windows 下的开发环境搭建

1. 克隆仓库：https://github.com/jpcap/jpcap (https://hub.fastgit.org/jovigb/jpcap-x64)
2. 将 lib 下的 jpcap.jar 添加到项目的 ClassPath 下
3. 将 Jpcap.dll 拷贝到 java.library.path 指定的路径下，一般是系统 Path ，可以使用`System.out.println(System.getProperty("java.library.path"));` 查看路径

 