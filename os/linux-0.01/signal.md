# Linux 0.01 中 `fn_ptr sig_fn[32]` 的调用过程

## 1. `sig_fn[32]` 是在哪里定义的？

`include/linux/sched.h` 中：

```c
typedef int (*fn_ptr)();

struct task_struct {
    long state;
    long counter;
    long priority;
    long signal;

    fn_ptr sig_restorer;
    fn_ptr sig_fn[32];

    ...
};
```

因此每个进程都有 32 个 signal handler 函数指针：

```text
current
  │
  ▼
task_struct
  │
  ├── signal       32 bit pending signal bitmap
  ├── sig_restorer
  │
  └── sig_fn[32]
       ├── sig_fn[0]   → signal 1 handler
       ├── sig_fn[1]   → signal 2 handler
       ├── ...
       └── sig_fn[31]  → signal 32 handler
```

`sig_fn[]` 保存的是各个 signal 对应的用户态 signal handler 地址。

---

## 2. 谁给 `sig_fn[i]` 赋值？

在 `kernel/sched.c` 的 `sys_signal()` 中：

```c
int sys_signal(long signal,long addr,long restorer)
{
    int i;

    if (signal < 1 || signal > 32)
        return -EINVAL;
    if (addr < 0 || addr > 0xBFFFFFFF)
        return -EINVAL;
    i = (long) current->sig_fn[signal-1];
    current->sig_fn[signal-1] = (fn_ptr) addr;
    current->sig_restorer = (fn_ptr) restorer;
    return i;
}
```

所以用户程序调用：

```c
signal(SIGINT, handler);
```

最终会形成：

```text
用户态
  │
  │ signal(SIGINT, handler)
  ▼
int 0x80
  │
  ▼
_system_call
  │
  ▼
sys_signal()
  │
  ▼
current->sig_fn[SIGINT-1] = handler
```

---

## 3. `sig_fn` 什么时候真正生效？

关键位置是：

```asm
ret_from_sys_call:
```

在 `kernel/system_call.s` 中，核心代码类似：

```asm
ret_from_sys_call:
    movl _current,%eax
    cmpl _task,%eax
    je 3f

    movl CS(%esp),%ebx
    testl $3,%ebx
    je 3f

    cmpw $0x17,OLDSS(%esp)
    jne 3f

2:
    movl signal(%eax),%ebx
    bsfl %ebx,%ecx
    je 3f

    btrl %ecx,%ebx
    movl %ebx,signal(%eax)

    movl sig_fn(%eax,%ecx,4),%ebx
```

最后这一句：

```asm
movl sig_fn(%eax,%ecx,4),%ebx
```

本质上就是：

```c
current->sig_fn[ecx]
```

其中：

```text
eax = current
ecx = signal number - 1
```

所以执行之后：

```text
EBX = 对应 signal handler 的地址
```

---

## 4. 但是这里并没有直接 `call` handler

这是理解 Linux 0.01 signal 机制最关键的一点。

你可能会期待：

```asm
call *%ebx
```

但实际上并没有这样调用。

后面的核心代码是：

```asm
cmpl $1,%ebx
jb default_signal
je 2b

movl $0,sig_fn(%eax,%ecx,4)
incl %ecx

xchgl %ebx,EIP(%esp)
```

其中最关键的是：

```asm
xchgl %ebx,EIP(%esp)
```

它把：

```text
EBX = signal handler 地址
```

和内核栈中保存的：

```text
EIP(%esp) = 原本用户程序的返回地址
```

交换。

于是：

```text
原来：

EIP(%esp)
    │
    └── 用户程序下一条指令


修改后：

EIP(%esp)
    │
    └── signal handler 地址
```

---

## 5. 为什么修改 EIP 就等于调用 handler？

因为内核接下来执行：

```asm
iret
```

`iret` 会从内核栈恢复用户态的：

```text
EIP
CS
EFLAGS
```

原来的：

```text
EIP = 用户程序下一条指令
```

已经被替换成：

```text
EIP = signal handler 地址
```

因此：

```text
iret
 │
 ▼
用户态
 │
 ▼
signal handler
```

所以 Linux 0.01 的 signal 处理实际上是：

> 内核不直接 `call` 用户函数，而是在返回用户态之前篡改保存的 EIP，让 `iret` 之后 CPU 自然跳到 signal handler。

---

## 6. 整个过程串起来

假设用户程序：

```c
void handler(int sig)
{
    ...
}

signal(SIGINT, handler);
```

假设：

```text
handler = 0x8048123
```

注册 handler 后：

```c
current->sig_fn[SIGINT - 1] = 0x8048123;
```

随后某个地方给当前进程发送 SIGINT，设置 pending signal：

```c
current->signal |= 1 << (SIGINT - 1);
```

此时：

```text
current
│
├── signal = pending SIGINT
│
└── sig_fn[]
      │
      └── [SIGINT-1] = 0x8048123
```

接下来发生一次系统调用或者时钟中断，最终进入：

```asm
ret_from_sys_call
```

### 第一步：读取 pending signal

```asm
movl signal(%eax),%ebx
```

### 第二步：找到具体 signal

```asm
bsfl %ebx,%ecx
```

假设：

```text
SIGINT = 2
```

那么：

```text
ecx = 1
```

### 第三步：取出 handler

```asm
movl sig_fn(%eax,%ecx,4),%ebx
```

得到：

```text
ebx = 0x8048123
```

### 第四步：修改保存的 EIP

```asm
xchgl %ebx,EIP(%esp)
```

于是：

```text
EIP(%esp) = 0x8048123
```

### 第五步：返回用户态

```asm
iret
```

CPU 恢复：

```text
CS  = 用户代码段
EIP = 0x8048123
```

于是直接开始执行：

```c
handler(...)
```

---

## 7. 整体流程图

```text
                  内核
                    │
                    │ ret_from_sys_call
                    ▼
        ┌──────────────────────┐
        │ 读取 current->signal │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ 读取 sig_fn[signal]  │
        └──────────┬───────────┘
                   │
                   ▼
              handler 地址
                   │
                   ▼
        修改保存的 EIP
                   │
                   ▼
                 iret
                   │
                   ▼
                 用户态
                   │
                   ▼
             signal handler
```

---

## 8. 为什么是在 `ret_from_sys_call`？

Linux 0.01 把 signal 检查和从内核返回用户态绑定在一起。

### 系统调用路径

```text
用户程序
   │
   │ int 0x80
   ▼
_system_call
   │
   ▼
sys_xxx()
   │
   ▼
ret_from_sys_call
   │
   ├── 检查 signal
   │
   ▼
iret
   │
   ▼
用户程序 / signal handler
```

### 时钟中断路径

```text
用户程序
   │
   │ timer interrupt
   ▼
_timer_interrupt
   │
   ▼
do_timer()
   │
   ▼
ret_from_sys_call
   │
   ├── 检查 signal
   │
   ▼
iret
```

因此 Linux 0.01 的 signal 检查发生在系统调用返回和时钟中断返回用户态之前。

---

## 9. 最关键的理解

看到：

```c
fn_ptr sig_fn[32];
```

不要把它理解成：

```text
内核保存了一组内核函数
        ↓
内核直接 call 这些函数
```

更准确的是：

```text
sig_fn[32]
    ↓
保存用户态 signal handler 地址
    ↓
ret_from_sys_call 读取
    ↓
修改保存的用户态 EIP
    ↓
iret
    ↓
CPU 在用户态执行 handler
```

也就是说，`sig_fn` 实际上参与构造了一次特殊的用户态执行现场。

此外，`system_call.s` 后面的代码还会修改 `OLDESP`，向用户栈压入原来的返回地址和 signal 编号，使 handler 执行结束后能够通过 `sig_restorer` 回到原来的程序执行点。

---

## 10. 一句话总结

**Linux 0.01 中 `sig_fn[32]` 并不是通过 `call` 指令直接调用的，而是在 `ret_from_sys_call` 中取出 handler 地址，改写内核栈中保存的用户态 EIP，最后通过 `iret` 返回用户态，从而让 CPU 从 signal handler 开始执行。**
