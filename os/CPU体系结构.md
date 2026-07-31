# CPU 体系结构
{docsify-updated}


## 大端模式与小端模式
IA-32 是 little endian，即内存中最小有效字节在前。 例如：
<center><img src="pics/little-endiannes.png" width="30%"></center>

## 寄存器
[引自](https://zhuanlan.zhihu.com/p/272135463)

<center><img src="pics/cpu-regs.jpg" width="50%"></center>


## 调用约定ABI
Linux x86-64 规定的参数传递：
+ arg1	RDI
+ arg2	RSI
+ arg3	RDX
+ arg4	RCX
+ arg5	R8
+ arg6	R9


标准 cdecl 调用约定下的栈帧示意图:
```
高地址 (主调函数的领地)
  |   ... 主调函数的局部变量 ...
  |   参数 2
  |   参数 1                 <-- [EBP + 8] (被调函数访问参数的起点)
  |   返回地址 (Return Addr) <-- [EBP + 4] (CALL 指令自动压入)
---------------------------------------- 两个栈帧的分界线
  |   旧的 EBP               <-- 【当前 EBP 指向这里】
  |   局部变量 1             <-- [EBP - 4]
  |   局部变量 2             
低地址 (被调函数的领地)       <-- 【ESP 指向当前栈顶】
```

kernel 加载 ELF 后，会构造初始用户栈：
```
高地址
+----------------+
| auxiliary vec  |
|   AT_*         |
+----------------+
| NULL           |
+----------------+
| envp[1]        |
| envp[0]        |
+----------------+
| NULL           |
+----------------+
| argv[2]        |
| argv[1]        |
| argv[0]        |
+----------------+
| argc           | <--- rsp
+----------------+
低地址
```

auxiliary vector（简称 auxv，辅助向量）是 Linux 内核在启动一个 ELF 程序时，额外传递给用户空间的一组键值对信息。它的主要作用是：
让 C 运行时（glibc/musl）、动态链接器（ld.so）等在程序启动时快速获取内核提供的运行环境信息，而不需要再次通过系统调用查询。
gdb 中使用 `info auxv` 能直接查看该信息。