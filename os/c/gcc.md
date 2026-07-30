# GCC
{docsify-updated}


## spec
gcc spec 文件（specification file）是 GCC 内部的规则配置文件，用于告诉 GCC：**当用户执行 gcc xxx 时，应该如何组织后续的编译、链接步骤，以及传递哪些默认参数。** 它控制：
+ 调用哪个编译器（cc1）
+ 调用哪个汇编器（as）
+ 调用哪个链接器（ld）
+ 默认链接哪些库
+ 默认动态链接器路径
+ 默认启动文件（crt1.o、crti.o 等）
+ 默认编译选项
+ 目标平台相关参数

使用 `gcc -dumpspecs` 可以查看当前 gcc 使用的 spec 文件内容。  
使用 `gcc -specs=my.spec` 可以指定使用自定义的 spec 文件。


musl-gcc 使用的 spec 如下：
```
%rename cpp_options old_cpp_options

*cpp_options:
-nostdinc -isystem /usr/local/musl/include -isystem include%s %(old_cpp_options)

*cc1:
%(cc1_cpu) -nostdinc -isystem /usr/local/musl/include -isystem include%s

*link_libgcc:
-L/usr/local/musl/lib -L .%s

*libgcc:
libgcc.a%s %:if-exists(libgcc_eh.a%s)

*startfile:
%{!shared: /usr/local/musl/lib/Scrt1.o} /usr/local/musl/lib/crti.o crtbeginS.o%s

*endfile:
crtendS.o%s /usr/local/musl/lib/crtn.o

*link:
-dynamic-linker /lib/ld-musl-aarch64.so.1 -nostdlib %{shared:-shared} %{static:-static} %{rdynamic:-export-dynamic}

*esp_link:

*esp_options:

*esp_cpp_options:
```

### spec 文件语法
典型的specs文件由预处理指令和规则块组成，关键语法包括：
+ `%rename`：重命名规则
+ `%include`：包含其他文件
+ `*link`: 定义链接器参数
+ `%{}`: 条件判断
+ `%{s}`：如果命令行中传了 -s 选项，则替换为对应的参数值。
+ `%{!s}`：如果命令行中没有传 -s 选项，则进行替换。
+ `%{s:val}`：如果传了 -s，则替换为 val。
+ `%{!s:val}`：如果没传 -s，则替换为 val。
+ `%(name)`：引用并展开另一个名为 name 的 Spec 变量。
+ `%rename old new`：将名为 old 的 Spec 规则重命名为 new。
+ `%include <file>`：包含并解析另外一个 Spec 文件。
+ `%etext`：打印错误信息 text 并中止编译。
+ `*cpp`:预处理阶段
+ `*cc1`:C 编译阶段
+ `*asm`:汇编阶段
+ `*link`:链接阶段
+ `*lib`:默认库
+ `*startfile`:启动文件
+ `*endfile`:结束文件