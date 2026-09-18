# musl-c 系统调用
{docsify-updated}

`arch/x86_64/syscall_arch.h` 下定义的系统调用函数：

```c
#define __SYSCALL_LL_E(x) (x)
#define __SYSCALL_LL_O(x) (x)

static __inline long __syscall0(long n)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall1(long n, long a1)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall2(long n, long a1, long a2)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2)
						  : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall3(long n, long a1, long a2, long a3)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall4(long n, long a1, long a2, long a3, long a4)
{
	unsigned long ret;
	register long r10 __asm__("r10") = a4;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3), "r"(r10): "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall5(long n, long a1, long a2, long a3, long a4, long a5)
{
	unsigned long ret;
	register long r10 __asm__("r10") = a4;
	register long r8 __asm__("r8") = a5;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3), "r"(r10), "r"(r8) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall6(long n, long a1, long a2, long a3, long a4, long a5, long a6)
{
	unsigned long ret;
	register long r10 __asm__("r10") = a4;
	register long r8 __asm__("r8") = a5;
	register long r9 __asm__("r9") = a6;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3), "r"(r10), "r"(r8), "r"(r9) : "rcx", "r11", "memory");
	return ret;
}
```
分别对应有1/2/3/4/5/6个参数的系统调用。

在 x86-64 下：
```
a  → RAX
D  → RDI
S  → RSI
d  → RDX
c  → RCX
r8 → R8
r9 → R9
r10 → R10
r11 → R11
r12 → R12
r13 → R13
r14 → R14
r15 → R15
```


Linux x86-64 syscall ABI：
```
系统调用号 → RAX
第1参数    → RDI
第2参数    → RSI
第3参数    → RDX
第4参数    → R10
第5参数    → R8
第6参数    → R9

返回值     ← RAX
```

CPU 执行 `syscall` 时， 会：
```
RCX ← 返回用户态后继续执行的位置
R11 ← 用户态 RFLAGS
```
系统调用完成后，会：
```
RAX ← 系统调用返回值
RSP ← 用户栈
RIP ← RCX
RFLAGS ← R11
```
然后跳到 `RCX` 恢复用户态执行。
