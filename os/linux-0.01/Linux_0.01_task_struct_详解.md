# Linux 0.01 `task_struct` 详解

> `task_struct` 是理解 Linux 0.01 **进程、调度、地址空间、信号、文件系统和系统调用**的核心数据结构。
>
> 本文以 Linux 0.01 的设计为主，并把各字段放回到进程创建、`execve()`、调度和文件描述符等完整流程中理解。

---

## 1. `task_struct` 是什么？

一句话：

> **`task_struct` 是内核用来描述一个进程的核心数据结构。**

Linux 0.01 中的定义位于 `include/linux/sched.h`，核心结构大致如下：

```c
struct task_struct {
    long state;
    long counter;
    long priority;

    long signal;
    struct sigaction sigaction[32];
    long blocked;

    int exit_code;

    unsigned long start_code;
    unsigned long end_code;
    unsigned long end_data;
    unsigned long brk;
    unsigned long start_stack;

    long pid;
    long father;
    long pgrp;
    long session;
    long leader;

    unsigned short uid, euid, suid;
    unsigned short gid, egid, sgid;

    long alarm;
    long utime, stime, cutime, cstime;
    long start_time;

    unsigned short used_math;

    int tty;
    unsigned short umask;

    struct m_inode *pwd;
    struct m_inode *root;
    struct m_inode *executable;

    unsigned long close_on_exec;
    struct file *filp[NR_OPEN];
};
```

不同源码镜像或整理版本可能存在细微差异，但整体设计基本如此。

可以先把字段分成几组：

```text
task_struct
│
├── 调度
│   ├── state
│   ├── counter
│   └── priority
│
├── 信号
│   ├── signal
│   ├── sigaction[]
│   └── blocked
│
├── 进程退出
│   └── exit_code
│
├── 用户地址空间
│   ├── start_code
│   ├── end_code
│   ├── end_data
│   ├── brk
│   └── start_stack
│
├── 进程身份
│   ├── pid
│   ├── father
│   ├── pgrp
│   ├── session
│   └── leader
│
├── 用户/时间/终端
│   ├── uid / gid
│   ├── alarm
│   ├── utime / stime
│   ├── tty
│   └── umask
│
└── 文件系统
    ├── pwd
    ├── root
    ├── executable
    ├── close_on_exec
    └── filp[]
```

---

# 2. `task_struct` 和“进程”的关系

不要简单地把：

```text
task_struct = 进程
```

画等号。

更准确地说：

```text
进程
│
├── 用户态地址空间
│
├── CPU 执行状态
│
├── 内核栈
│
├── task_struct
│
├── 打开的文件
│
├── 当前目录
│
└── 其他内核资源
```

`task_struct` 是：

> **内核描述和管理这个进程的核心数据结构。**

例如内核需要知道：

```c
current->pid
```

就是当前进程的 PID；

```c
current->counter
```

就是当前进程剩余的时间片；

```c
current->pwd
```

就是当前工作目录；

```c
current->filp[fd]
```

就是当前进程的某个文件描述符对应的 `struct file`。

---

# 3. Linux 0.01 的核心设计：`task_struct` + 内核栈

早期 Linux 有一个非常经典的设计：

```text
一个进程的内核栈
        +
task_struct
```

通常放在一个固定大小的区域中，可以抽象为：

```text
          一个 4KB 区域
┌─────────────────────────────┐
│                             │
│       kernel stack          │
│                             │
│             ↓               │
│                             │
│                             │
├─────────────────────────────┤
│       task_struct           │
└─────────────────────────────┘
```

典型定义可以理解为：

```c
union task_union {
    struct task_struct task;
    char stack[PAGE_SIZE];
};
```

其中：

```c
#define PAGE_SIZE 4096
```

因此，一个任务的 `task_struct` 和它的内核栈处于同一个 4KB 对齐区域。

这会带来一个非常重要的性质：

> **只要知道当前内核栈的 `%esp`，就可以通过对齐操作快速定位当前进程的 `task_struct`。**

---

# 4. `current` 是怎么找到当前进程的？

假设当前进程正在内核态执行：

```text
          ESP
           │
           ↓
┌──────────────────────┐
│    kernel stack      │
│                      │
│                      │
├──────────────────────┤
│    task_struct       │
└──────────────────────┘
```

因为整个区域按照 4KB 对齐，可以把 `%esp` 的低 12 位清零：

```asm
andl $0xfffff000, %esp
```

例如：

```text
ESP = 0x01AB7F20

0x01AB7F20
&
0xFFFFF000
-----------
0x01AB7000
```

得到的：

```text
0x01AB7000
```

就是当前任务所在 4KB 区域的起始地址。

所以可以形成这样的理解：

```text
CPU 当前正在执行
        │
        ↓
      ESP
        │
        │ 低 12 位清零
        ↓
task_union 起始地址
        │
        ↓
task_struct
        │
        ↓
current
```

这就是早期 Linux 中 `current` 机制背后的基本思想。

---

# 5. 调度相关字段

## 5.1 `state`

```c
long state;
```

表示进程当前状态。

Linux 0.01 中可以看到类似：

```c
#define TASK_RUNNING        0
#define TASK_INTERRUPTIBLE  1
#define TASK_UNINTERRUPTIBLE 2
#define TASK_ZOMBIE         3
#define TASK_STOPPED        4
```

可以理解为：

```text
TASK_RUNNING
    │
    └── 进程可以运行

TASK_INTERRUPTIBLE
    │
    └── 可被信号唤醒的睡眠/等待状态

TASK_UNINTERRUPTIBLE
    │
    └── 不可被普通信号打断的等待

TASK_ZOMBIE
    │
    └── 进程已经退出，但父进程尚未完成 wait

TASK_STOPPED
    │
    └── 进程被停止
```

例如：

```c
current->state = TASK_INTERRUPTIBLE;
```

可以理解为当前进程主动进入等待状态。

---

## 5.2 `counter`

```c
long counter;
```

这是早期 Linux 调度器中的一个核心字段：

> **当前进程剩余的时间片。**

例如：

```text
进程 A   counter = 10
进程 B   counter = 5
进程 C   counter = 20
```

时钟中断不断到来：

```text
timer interrupt
       │
       ↓
current->counter--
       │
       ↓
counter 是否耗尽？
       │
       └── 是 → schedule()
```

因此：

```text
counter
   ↓
当前进程还剩多少调度时间片
```

---

## 5.3 `priority`

```c
long priority;
```

表示进程优先级。

Linux 0.01 的调度算法非常简单。当可运行进程的时间片都耗尽后，会重新计算 `counter`，思路类似：

```c
p->counter = (p->counter >> 1) + p->priority;
```

也就是：

```text
新的 counter
    =
旧 counter / 2
    +
priority
```

因此在这个简单调度模型中：

```text
priority 越高
      ↓
重新获得的时间片通常越多
```

---

# 6. 信号相关字段

## 6.1 `signal`

```c
long signal;
```

在早期 32 位 Linux 中，它可以看作一个 32 位信号位图：

```text
signal

31 30 29 ... 2 1 0
│
│
└── 每一位对应一个 signal
```

作用是记录：

> **当前进程有哪些信号处于 pending 状态。**

可以抽象理解成：

```text
signal
   ↓
“有哪些信号来了？”
```

---

## 6.2 `sigaction[32]`

```c
struct sigaction sigaction[32];
```

它保存不同 signal 的处理方式。

可以理解为：

```text
signal 1  → sigaction[0]
signal 2  → sigaction[1]
signal 3  → sigaction[2]
...
signal 32 → sigaction[31]
```

因此：

```text
signal
   ↓
来了什么信号？

sigaction[]
   ↓
这个信号应该怎么处理？
```

---

## 6.3 `blocked`

```c
long blocked;
```

同样是一个位图，表示：

> **当前进程暂时阻塞了哪些信号。**

三个字段合起来：

```text
signal
   ↓
哪些信号已经到达？

blocked
   ↓
哪些信号暂时不处理？

sigaction[]
   ↓
处理某个信号时采取什么动作？
```

---

# 7. 进程退出：`exit_code`

```c
int exit_code;
```

进程退出时保存退出状态。

例如：

```c
exit(5);
```

可以理解为：

```text
current->exit_code = 5
```

之后父进程：

```c
wait(...)
```

就可以获得子进程的退出状态。

整个过程：

```text
子进程
   │
   │ exit(5)
   ↓
exit_code = 5
   │
   │ 等待父进程 wait
   ↓
父进程获得退出状态
```

这也是理解 **僵尸进程** 的关键。

---

# 8. 用户地址空间相关字段

这一组非常重要：

```c
unsigned long start_code;
unsigned long end_code;
unsigned long end_data;
unsigned long brk;
unsigned long start_stack;
```

它们记录进程用户地址空间中的重要边界。

可以抽象成：

```text
高地址
0xFFFFFFFF
        │
        │
        │        用户栈
        │          ↓
        │
        │
        │
        │          ↑
        │        heap
        │
        │       brk
        ├──────────────
        │
        │      data
        │
        ├──────────────
        │
        │      code
        │
        ├──────────────
        │
        │
0x00000000
```

---

## 8.1 `start_code`

```c
start_code
```

代码段起始地址。

---

## 8.2 `end_code`

```c
end_code
```

代码段结束地址。

因此：

```text
start_code
     │
     ↓
┌───────────────┐
│     text      │
│     code      │
└───────────────┘
     ↑
     │
  end_code
```

---

## 8.3 `end_data`

表示数据区域结束位置。

可以粗略理解为：

```text
┌───────────────┐
│     code      │
├───────────────┤
│     data      │
│     bss       │
└───────────────┘
        ↑
     end_data
```

---

## 8.4 `brk`

这个字段和后来的 `brk()` / `sbrk()` / `malloc()` 有直接关系。

```c
brk
```

表示：

> **用户堆当前的边界。**

可以理解成：

```text
heap
┌────────────────────┐
│                    │
│        heap        │
│                    │
└────────────────────┘
          ↑
         brk
```

当程序扩大 heap 时：

```text
brk ↑
```

---

## 8.5 `start_stack`

```c
start_stack
```

表示用户栈起始位置。

因此：

```text
start_code
end_code
end_data
brk
start_stack
```

整体就是：

> **进程用户地址空间布局的重要元数据。**

---

# 9. 进程身份相关字段

```c
long pid;
long father;
long pgrp;
long session;
long leader;
```

这几个字段最好放在一起理解。

---

## 9.1 `pid`

```c
long pid;
```

当前进程 ID。

例如：

```text
pid = 123
```

---

## 9.2 `father`

```c
long father;
```

表示父进程的 PID。

例如：

```text
父进程
pid = 10
   │
   │ fork()
   ↓
子进程
pid = 20
father = 10
```

---

## 9.3 `pgrp`

```c
long pgrp;
```

进程组 ID。

可以抽象成：

```text
session
   │
   ├── process group 100
   │      ├── pid 101
   │      ├── pid 102
   │      └── pid 103
   │
   └── process group 200
          ├── pid 201
          └── pid 202
```

进程组和终端、作业控制、信号发送密切相关。

---

## 9.4 `session`

```c
long session;
```

会话 ID。

可以理解成更高一级的进程组织单位：

```text
session
   │
   ├── process group
   │      ├── process
   │      └── process
   │
   └── process group
          ├── process
          └── process
```

---

## 9.5 `leader`

```c
long leader;
```

表示该进程是否为 session leader。

因此：

```text
pid
father
pgrp
session
leader
```

这一组主要描述：

> **进程之间的父子关系、进程组和会话关系。**

---

# 10. 用户和组权限

```c
unsigned short uid, euid, suid;
unsigned short gid, egid, sgid;
```

### UID

真实用户 ID：

```text
uid
```

### EUID

有效用户 ID：

```text
euid
```

内核执行权限检查时非常重要。

### SUID

保存用户 ID：

```text
suid
```

GID 同理：

```text
gid
egid
sgid
```

可以概括成：

```text
uid / gid
    ↓
真实身份

euid / egid
    ↓
当前有效权限

suid / sgid
    ↓
保存的身份信息
```

---

# 11. `alarm`

```c
long alarm;
```

记录 `alarm()` 相关的定时信息。

例如：

```c
alarm(10);
```

之后：

```text
timer interrupt
       │
       ↓
检查 alarm
       │
       ↓
时间到
       │
       ↓
发送 SIGALRM
```

因此 `alarm` 是进程级的定时器状态。

---

# 12. CPU 时间统计

```c
long utime;
long stime;
long cutime;
long cstime;
```

可以这样记：

| 字段 | 含义 |
|---|---|
| `utime` | 当前进程用户态 CPU 时间 |
| `stime` | 当前进程内核态 CPU 时间 |
| `cutime` | 子进程用户态 CPU 时间 |
| `cstime` | 子进程内核态 CPU 时间 |

即：

```text
当前进程
├── utime
└── stime

子进程
├── cutime
└── cstime
```

---

# 13. `start_time`

```c
long start_time;
```

表示进程创建/开始运行相关的时间信息。

它与：

```text
utime
stime
cutime
cstime
```

一起构成进程的时间统计信息。

---

# 14. `used_math`

```c
unsigned short used_math;
```

表示该进程是否使用过浮点运算相关的数学协处理器状态。

早期 x86 的 FPU 状态管理和现代 CPU 不同，因此调度时需要考虑浮点寄存器状态的保存和恢复。

简单理解：

```text
used_math = 0
    ↓
还没有使用数学协处理器/FPU

used_math != 0
    ↓
使用过 FPU
```

---

# 15. `tty`

```c
int tty;
```

表示进程关联的终端设备。

例如一个 shell：

```text
shell
  │
  └── tty
        ↓
      /dev/tty1
```

终端输入输出的大致关系：

```text
键盘
 ↓
tty
 ↓
进程
```

它和前面的：

```text
pgrp
session
leader
```

存在紧密关系。

---

# 16. `umask`

```c
unsigned short umask;
```

表示创建文件时使用的权限掩码。

例如：

```text
umask = 022
```

创建文件默认权限：

```text
0666
```

最终：

```text
0666 & ~0022
= 0644
```

因此 `umask` 是进程自己的文件创建权限属性。

---

# 17. 文件系统相关字段

这一组：

```c
struct m_inode *pwd;
struct m_inode *root;
struct m_inode *executable;
unsigned long close_on_exec;
struct file *filp[NR_OPEN];
```

是理解 Linux 0.01 文件系统非常关键的一组字段。

---

# 18. `pwd`

```c
struct m_inode *pwd;
```

`pwd` = present working directory，即当前工作目录。

它指向一个：

```c
struct m_inode
```

例如：

```text
current
   │
   └── pwd
         ↓
       inode
         ↓
      /home/user
```

执行：

```c
chdir("/tmp");
```

本质上就是让：

```text
current->pwd
```

指向新的目录 inode。

---

# 19. `root`

```c
struct m_inode *root;
```

表示该进程看到的根目录。

通常：

```text
current->root
      ↓
      /
```

这也是理解 `chroot()` 的基础。

---

# 20. `executable`

```c
struct m_inode *executable;
```

表示当前执行程序对应的 inode。

例如执行：

```bash
/bin/bash
```

可以抽象成：

```text
current
   │
   └── executable
          ↓
        inode
          ↓
       /bin/bash
```

这对 `execve()` 尤其重要。

---

# 21. `close_on_exec`

```c
unsigned long close_on_exec;
```

这是一个文件描述符位图。

它表示：

> **哪些文件描述符在 `execve()` 时应该自动关闭。**

例如：

```text
fd:
0 1 2 3 4 5 ...

close_on_exec:
0 0 0 1 0 1 ...
```

表示：

```text
fd 3 → exec 时关闭
fd 5 → exec 时关闭
```

---

# 22. `filp[NR_OPEN]`

```c
struct file *filp[NR_OPEN];
```

这是进程的文件描述符表。

例如：

```text
              task_struct
                   │
                   ↓
             ┌───────────┐
             │   filp[]  │
             ├───────────┤
fd 0 ───────→│ file *    │──→ stdin
fd 1 ───────→│ file *    │──→ stdout
fd 2 ───────→│ file *    │──→ stderr
fd 3 ───────→│ file *    │──→ ...
fd 4 ───────→│ file *    │──→ ...
             └───────────┘
```

因此可以记住 Linux 文件描述符的一条核心链：

```text
进程
  ↓
task_struct
  ↓
filp[fd]
  ↓
struct file
  ↓
m_inode
```

这条链对于继续学习 Linux 0.01 的 `open()`、`read()`、`write()`、`close()` 非常重要。

---

# 23. `task_struct` 和 `fork()`

理解 `task_struct` 最好的方式之一，就是观察 `fork()`。

用户程序：

```c
fork();
```

大致调用链：

```text
fork()
  ↓
系统调用
  ↓
sys_fork
  ↓
copy_process
```

`copy_process()` 创建新的任务：

```text
父进程

task_struct A
      │
      │ copy_process()
      ↓
task_struct B
```

例如：

```text
A:
pid = 10

B:
pid = 11
father = 10
```

新进程会继承/复制父进程的大量属性，例如：

```text
uid/gid
pwd
root
filp[]
信号处理设置
地址空间相关信息
```

但进程身份等字段会按照子进程规则设置。

因此：

> **`fork()` 的核心之一，就是创建一个新的 `task_struct`，从父进程复制出新的进程控制信息。**

---

# 24. `execve()` 和 `task_struct`

这与你前面研究的：

```asm
sys_execve:
    lea EIP(%esp),%eax
    pushl %eax
    call do_execve
    addl $4,%esp
    ret
```

正好可以串起来。

用户程序：

```c
execve("/bin/xxx", argv, envp);
```

进入内核：

```text
用户态
  │
  ↓
system_call
  │
  ↓
sys_execve
  │
  ↓
do_execve
```

`execve()` 和 `fork()` 有一个非常重要的区别：

```text
fork()
  ↓
创建一个新的进程

execve()
  ↓
不创建新的 PID
  ↓
替换当前进程正在运行的程序
```

因此：

```text
PID 基本保持不变
task_struct 仍然是当前这个任务
```

但进程的用户地址空间会发生重大变化：

```text
旧程序
  │
  ├── 旧 code
  ├── 旧 data
  ├── 旧 stack
  └── 旧 brk
       │
       │ execve()
       ↓
新程序
  │
  ├── 新 code
  ├── 新 data
  ├── 新 stack
  └── 新 brk
```

同时：

```text
current->executable
```

也会对应新的可执行文件。

---

# 25. 为什么 `sys_execve` 要把 EIP 所在位置传给 `do_execve()`？

前面代码：

```asm
sys_execve:
    lea EIP(%esp),%eax
    pushl %eax
    call do_execve
    addl $4,%esp
    ret
```

关键是：

```asm
lea EIP(%esp), %eax
```

这里的 `EIP` 是一个**栈布局中的偏移量**，不是 `%eip` 寄存器本身。

`lea` 的含义是计算地址：

```text
EAX = ESP + EIP
```

所以：

```asm
lea EIP(%esp), %eax
```

得到的是：

> **保存的用户态寄存器现场中 EIP 所在位置的地址。**

然后：

```asm
pushl %eax
```

把这个地址作为参数传给：

```c
do_execve(...)
```

可以粗略理解为：

```c
struct pt_regs *regs = ...;

do_execve(..., regs);
```

也就是说，`do_execve()` 不只是需要：

```text
filename
argv
envp
```

还需要访问：

> **系统调用进入内核时保存下来的用户态寄存器现场。**

这就是 `sys_execve` 中 `lea EIP(%esp), %eax` 的意义。

---

# 26. `task_struct`、调度和 CPU 切换

最终，`task_struct` 会和调度器连接起来。

大致关系：

```text
task_struct
     │
     ├── state
     ├── counter
     └── priority
             │
             ↓
         schedule()
             │
             ↓
        选择下一个任务
             │
             ↓
        switch_to()
             │
             ↓
       CPU 上下文切换
```

所以：

```text
task_struct
      ↓
描述“这个进程是谁、有什么资源、处于什么状态”

schedule()
      ↓
决定“下一个运行谁”

switch_to()
      ↓
真正把 CPU 执行环境切换过去
```

---

# 27. 从 `task_struct` 看 Linux 0.01 的整个进程模型

现在可以把 Linux 0.01 的进程体系串起来：

```text
                         进程
                          │
                          ↓
                    task_struct
                          │
        ┌─────────────────┼──────────────────┐
        │                 │                  │
        ↓                 ↓                  ↓
      调度              地址空间            文件
        │                 │                  │
 state/counter       code/data/brk        filp[]
 priority            start_stack             │
        │                 │                  ↓
        ↓                 ↓              struct file
    schedule()          execve()             │
        │                                    ↓
        ↓                                m_inode
   switch_to()
        │
        ↓
      CPU
```

同时：

```text
fork()
   ↓
copy_process()
   ↓
创建新的 task_struct
```

而：

```text
execve()
   ↓
do_execve()
   ↓
替换当前进程的用户程序/地址空间
```

---

# 28. Linux 0.01 `task_struct` 的本质

如果把所有字段压缩成几个问题，那么 `task_struct` 实际上是在回答：

```text
① 这个进程是谁？
   ├── pid
   ├── father
   ├── pgrp
   ├── session
   └── uid/gid

② 这个进程现在能不能运行？
   ├── state
   ├── counter
   └── priority

③ 这个进程收到什么信号？
   ├── signal
   ├── blocked
   └── sigaction[]

④ 这个进程的用户地址空间在哪里？
   ├── start_code
   ├── end_code
   ├── end_data
   ├── brk
   └── start_stack

⑤ 这个进程打开了什么？
   └── filp[]

⑥ 这个进程当前在哪里？
   ├── pwd
   ├── root
   └── tty

⑦ 这个进程什么时候运行、什么时候退出？
   ├── utime
   ├── stime
   ├── start_time
   ├── alarm
   └── exit_code
```

所以最值得记住的一句话是：

> **`task_struct` 就是 Linux 内核管理进程时保存的“进程控制块（PCB）”。**

---

# 29. 推荐的 Linux 0.01 源码阅读路线

如果你正在从源码学习 Linux 0.01，不建议只把 `task_struct` 当作一个结构体背字段。

更好的路线是沿着它的“生命周期”阅读：

```text
                    task_struct
                         │
            ┌────────────┼────────────┐
            │            │            │
            ↓            ↓            ↓
          创建           执行          调度
            │            │            │
            ↓            ↓            ↓
       copy_process()  do_execve()  schedule()
            │            │            │
            │            │            ↓
            │            │        switch_to()
            │            │            │
            │            │            ↓
            │            │           CPU
            │            │
            ↓            ↓
        新进程       替换程序
            │
            └────────────┐
                         ↓
                       exit()
                         │
                         ↓
                    exit_code
                         │
                         ↓
                       wait()
```

如果继续深入，最值得按这个顺序看：

1. `include/linux/sched.h` —— 先把 `task_struct` 和任务状态搞清楚
2. `kernel/fork.c` —— 看 `copy_process()` 如何创建任务
3. `kernel/sched.c` —— 看 `schedule()` 如何选择任务
4. `kernel/system_call.s` —— 看系统调用如何进入内核、如何保存寄存器
5. `fs/exec.c` —— 看 `do_execve()` 如何替换进程地址空间
6. `kernel/exit.c` —— 看 `exit()` / `wait()` 和僵尸进程
7. `include/linux/fs.h` + `fs/` —— 沿 `filp[] → struct file → m_inode` 学文件系统

这样会把：

```text
进程创建
    ↓
task_struct
    ↓
系统调用
    ↓
调度
    ↓
CPU 上下文切换
    ↓
exec
    ↓
exit/wait
```

完整串成一条线。
