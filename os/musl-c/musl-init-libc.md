# musl `__init_libc` 函数详解

> 源码位置：`src/env/__libc_start_main.c`
> 相关：`src/env/__init_tls.c`、`src/env/__stack_chk_fail.c`、`src/internal/libc.h`

`__init_libc` 是 libc 在 `main` 之前完成「环境初始化」的核心函数，由 `__libc_start_main` 调用。

## 启动链中的位置

```
_start (汇编) → _start_c (crt1.c) → __libc_start_main
                                        ├─ __init_libc(envp, argv[0])   ← 本函数
                                        └─ stage2 → __libc_start_init → exit(main(...))
```

调用点被刻意做成 **外部链接 + noinline**：

```c
__attribute__((__noinline__))
void __init_libc(char **envp, char *pn)
```

> `noinline` 是有意为之：让初始化用的这一大块栈帧在函数返回后立即释放，
> 不会白白占用整个进程生命周期（源码 77~79 行注释）。

## 逐段功能拆解

### ① 保存环境变量指针

```c
__environ = envp;
```

把内核传来的 `envp` 存入全局变量 `__environ`（即 `environ`/`getenv` 背后的那个）。
`__environ` 定义在 `src/env/__environ.c`，另有 `environ`/`_environ` 等弱别名。

### ② 定位并解析 auxv（辅助向量）

```c
for (i=0; envp[i]; i++);               // 走到 envp 数组末尾(NULL)
libc.auxv = auxv = (void *)(envp+i+1); // auxv 紧跟在 envp 之后
for (i=0; auxv[i]; i+=2)
    if (auxv[i]<AUX_CNT) aux[auxv[i]] = auxv[i+1];
```

**auxv（auxiliary vector）** 是内核加载 ELF 时放在栈上、紧跟环境变量之后的一串
`(类型, 值)` 键值对。这里：

1. 把 auxv 起始地址存进 `libc.auxv`（后续 `dl_iterate_phdr`、mallocng 取随机数等会用）；
2. 把它「摊平」成本地数组 `aux[]`，用类型编号当下标，方便按 `AT_*` 直接取值。

### ③ 从 auxv 提取关键系统信息

```c
__hwcap = aux[AT_HWCAP];                          // CPU 硬件能力位
if (aux[AT_SYSINFO]) __sysinfo = aux[AT_SYSINFO]; // vDSO 系统调用入口(某些架构)
libc.page_size = aux[AT_PAGESZ];                  // 页大小
```

- `__hwcap`：CPU 特性标志，后续字符串/内存函数据此选更快实现（aarch64 常见）。
- `__sysinfo`：i386 等架构通过它走 vDSO 发系统调用（参见 `src/internal/i386/defsysinfo.s`）。
- `libc.page_size`：全局页大小，`PAGE_SIZE` 宏就是 `libc.page_size`。

### ④ 设置程序名

```c
if (!pn) pn = (void*)aux[AT_EXECFN];   // pn 为空则退回用 auxv 里的可执行文件名
if (!pn) pn = "";
__progname = __progname_full = pn;
for (i=0; pn[i]; i++) if (pn[i]=='/') __progname = pn+i+1;  // 取最后一个 '/' 之后
```

`pn` 就是 `argv[0]`，同时设置：

- `__progname_full`：完整路径；
- `__progname`：去掉目录的短名（供 `program_invocation_short_name`、`err()`/`perror` 使用）。

### ⑤ 初始化 TLS 与栈保护（SSP）

```c
__init_tls(aux);
__init_ssp((void *)aux[AT_RANDOM]);
```

- **`__init_tls(aux)`**：初始化线程局部存储和线程指针。解析程序头找 `PT_TLS` 段，
  分配 TLS 内存，`__copy_tls` 拷贝初始镜像，`__init_tp` 设置线程指针
  （`set_thread_area`、`tid`、`sysinfo`、locale 等）。**失败直接 `a_crash()`**，
  因为之后所有涉及线程指针的代码（含栈保护）都依赖它。定义于 `src/env/__init_tls.c`。
- **`__init_ssp(aux[AT_RANDOM])`**：初始化栈溢出保护 canary。`AT_RANDOM` 指向内核提供的
  16 字节随机数，拷进 `__stack_chk_guard` 并写入当前线程 `canary`。64 位下把第 2 字节清零
  （防字符串函数泄漏/覆盖 canary）。定义于 `src/env/__stack_chk_fail.c`：

```c
void __init_ssp(void *entropy)
{
    if (entropy) memcpy(&__stack_chk_guard, entropy, sizeof(uintptr_t));
    else __stack_chk_guard = (uintptr_t)&__stack_chk_guard * 1103515245;
    ...
    __pthread_self()->canary = __stack_chk_guard;
}
```

> 顺序关键：必须先 `__init_tls`（线程指针可用）才能 `__init_ssp`（写线程 canary）。
> `__libc_start_main` 在调完 `__init_libc` 后加了编译器屏障
> （`__asm__("":::"memory")`），防止用到线程指针/SSP 的应用代码被提前。

### ⑥ setuid/setgid 安全加固

```c
if (aux[AT_UID]==aux[AT_EUID] && aux[AT_GID]==aux[AT_EGID]
    && !aux[AT_SECURE]) return;   // 普通进程：到此结束

struct pollfd pfd[3] = { {.fd=0}, {.fd=1}, {.fd=2} };
int r = __syscall(SYS_poll, pfd, 3, 0);   // (或 SYS_ppoll)
if (r<0) a_crash();
for (i=0; i<3; i++) if (pfd[i].revents&POLLNVAL)
    if (__sys_open("/dev/null", O_RDWR)<0)
        a_crash();
libc.secure = 1;
```

只对「特权提升」进程生效（实际 UID≠有效 UID、GID≠有效 GID，或 `AT_SECURE` 置位，
即典型 setuid/setgid 程序）：

- 用 `poll` 检查 fd 0/1/2 是否都已打开；
- 若某个是 `POLLNVAL`（未打开），用 `/dev/null` 占上。

**目的**：防经典 fd 劫持攻击——攻击者关闭 fd 1 再运行 setuid 程序，程序内部 `open()`
敏感文件会拿到 fd 1，之后写 stdout 就覆盖该文件。`/dev/null` 兜底堵住此漏洞。
最后置 `libc.secure = 1`，后续代码据此收紧行为（如是否信任 `LD_*` 等）。

## `libc` 全局结构里被本函数填的字段

```c
struct __libc {
    char secure;            // ⑥ 设置
    size_t *auxv;           // ② 设置
    struct tls_module *tls_head;
    size_t tls_size, tls_align, tls_cnt;  // __init_tls 填
    size_t page_size;       // ③ 设置
    struct __locale_struct global_locale;
};
```

## 一句话总结

`__init_libc` 是 libc 的「环境自举」函数，在 `main` 之前搭好 C/POSIX 运行环境，做六件事：

1. 保存 `environ`；
2. 定位并解析内核传来的 auxv；
3. 提取 `__hwcap` / `__sysinfo` / `page_size` 等系统信息；
4. 设置程序名（`__progname` / `__progname_full`）；
5. 初始化 **TLS/线程指针** 和 **栈保护 canary**（顺序关键，失败即 crash）；
6. 对 setuid/setgid 进程做 **stdio fd 加固** 并标记 `libc.secure`。
