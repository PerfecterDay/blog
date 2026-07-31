# gdb
{docsify-updated}


添加断点处：
```
break _start_c          # crt1.c 里的 C 入口
break __libc_start_main # 启动主逻辑
break __init_libc       # 初始化 TLS/SSP/auxv 等
run
```
+ `starti` 启动程序并停在第一条可执行指令处
+ `run` 启动程序，开始调试
+ `bt` 查看调用栈
+ `frame 1` 切换栈帧
+ `info registers/info r` 查看所有寄存器
+ `info locals` 查看局部变量值
+ `info args` 查看当前函数参数值
+ `info variables` 显示程序中所有符号
+ `info auxv` 查看 auxv 信息
+ `p global_count` 打印变量global_count的值
+ `x/10gx p` 查看内存p的值
+ `c` continue继续执行
+ `n` next单步执行
+ `si` step into进入函数
+ `fin` finish执行到当前函数结束
+ `kill` 杀死当前调试的程序
+ `exit` 退出