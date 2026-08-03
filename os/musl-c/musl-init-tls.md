# `static_init_tls` 函数详细分析

```
_start (汇编) → _start_c (crt1.c) → __libc_start_main
                                        ├─ __init_libc(envp, argv[0])   ← 本函数
                                                |- __init_tls(aux);
                                                |- __init_ssp((void *)aux[AT_RANDOM]);
                                        └─libc_start_main_stage2((main, argc, argv))
												|- __libc_start_init()
												|- exit(main(argc, argv, envp));                                         
```

## 概述

`static_init_tls` 位于 `src/env/__init_tls.c`,是 musl libc 在**静态链接**(以及动态链接场景的初始 fallback)启动时初始化**线程局部存储 (TLS, Thread-Local Storage)** 的核心函数。它通过 `weak_alias` 被暴露为 `__init_tls`:

```c
static void static_init_tls(size_t *aux)
{ ... }
weak_alias(static_init_tls, __init_tls);
```

它在程序启动早期(`__init_libc` 之后)被调用,参数 `aux` 是内核传入的 **auxiliary vector**(以 `AT_*` 为索引的数组)。

它的最终目标:计算主程序需要的 TLS 内存布局大小 → 分配内存 → 复制 TLS 初始化镜像 → 设置线程指针 (TP)。

---

## 分步解析

### 1. 遍历程序头 (Program Headers),提取关键信息

```c
for (p=(void *)aux[AT_PHDR],n=aux[AT_PHNUM]; n; n--,p+=aux[AT_PHENT]) {
    phdr = (void *)p;
    if (phdr->p_type == PT_PHDR)
        base = aux[AT_PHDR] - phdr->p_vaddr;
    if (phdr->p_type == PT_DYNAMIC && _DYNAMIC)
        base = (size_t)_DYNAMIC - phdr->p_vaddr;
    ...
```

- 从 `auxv` 取得程序头表地址 `AT_PHDR`、数目 `AT_PHNUM`、每项大小 `AT_PHENT`,遍历所有 program header。
- **计算加载基址 `base`**(用于把 ELF 里的虚拟地址 `p_vaddr` 换算成运行时真实地址):
  - `PT_PHDR`:程序头本身的映射地址已知(`AT_PHDR`),用 `AT_PHDR - p_vaddr` 得到基址。这是位置无关可执行文件 (PIE) 的典型手段。
  - `PT_DYNAMIC` 且存在 `_DYNAMIC`:用动态段的实际地址(链接器解析的 `_DYNAMIC` 符号)减去其 `p_vaddr`。`_DYNAMIC` 是 weak 符号,静态非 PIE 程序没有,此时 `base` 保持为 0。
- **`PT_TLS`**:记录 TLS 段的 program header 到 `tls_phdr`,这是本函数关注的重点。
- **`PT_GNU_STACK`**:如果段指定了更大的栈需求,更新 `__default_stacksize`(受 `DEFAULT_STACK_MAX = 8MB` 上限约束)。这与 TLS 无直接关系,只是顺带在同一遍历里完成。

### 2. 填充主程序的 TLS 模块描述 `main_tls`

```c
if (tls_phdr) {
    main_tls.image = (void *)(base + tls_phdr->p_vaddr);
    main_tls.len = tls_phdr->p_filesz;
    main_tls.size = tls_phdr->p_memsz;
    main_tls.align = tls_phdr->p_align;
    libc.tls_cnt = 1;
    libc.tls_head = &main_tls;
}
```

`struct tls_module` 定义于 `libc.h`:`{ next, image, len, size, align, offset }`。

- `image`:TLS 初始化数据在内存中的地址(基址 + 段虚拟地址)。
- `len` = `p_filesz`:需要**复制**的已初始化数据字节数(`.tdata`)。
- `size` = `p_memsz`:TLS 段总大小(含 `.tbss` 未初始化部分,`memsz ≥ filesz`)。
- `align` = `p_align`:对齐要求。
- 把该模块登记进全局 `libc`:`tls_cnt = 1`(只有主程序一个模块),`tls_head` 指向它。

若没有 `PT_TLS`(程序不使用 TLS),`main_tls` 保持全 0,后续计算仍能正确工作(相当于 size=0)。

### 3. 对齐修正与计算模块偏移 `offset`

```c
main_tls.size += (-main_tls.size - (uintptr_t)main_tls.image)
    & (main_tls.align-1);
#ifdef TLS_ABOVE_TP
    main_tls.offset = GAP_ABOVE_TP;
    main_tls.offset += (-GAP_ABOVE_TP + (uintptr_t)main_tls.image)
        & (main_tls.align-1);
#else
    main_tls.offset = main_tls.size;
#endif
if (main_tls.align < MIN_TLS_ALIGN) main_tls.align = MIN_TLS_ALIGN;
```

这里存在两种架构 ABI 变体(通过 `TLS_ABOVE_TP` 区分,定义在各架构 `pthread_arch.h`):

- **`!TLS_ABOVE_TP`(变体 II,如 x86_64/i386)**:TLS 位于线程指针 (TP) **之下**。`offset = size`,即通过 `TP - offset` 访问变量。`size` 先做对齐修正,保证 `image` 相对对齐正确。
- **`TLS_ABOVE_TP`(变体 I,如 arm/aarch64/mips/riscv 等)**:TLS 位于 TP **之上**,前面还有一个架构相关的空隙 `GAP_ABOVE_TP`(如 aarch64 是 16,arm/sh 是 8,mips 系是 0)。`offset` 从该 gap 起算并做对齐补偿。

对齐表达式 `(-x) & (align-1)` 是常见的"向上对齐到 align 边界所需的补齐字节数"的位运算技巧(要求 `align` 是 2 的幂)。

最后确保 `align` 不小于 `MIN_TLS_ALIGN`(= `struct builtin_tls` 中 `pt` 的偏移,亦即 `char c` 之后 `struct pthread` 的对齐),以保证 `struct pthread` 本身也能正确对齐。

### 4. 计算整块 TLS 区所需总大小 `libc.tls_size`

```c
libc.tls_align = main_tls.align;
libc.tls_size = 2*sizeof(void *) + sizeof(struct pthread)
#ifdef TLS_ABOVE_TP
        + main_tls.offset
#endif
        + main_tls.size + main_tls.align
        + MIN_TLS_ALIGN-1 & -MIN_TLS_ALIGN;
```

这一块内存要容纳:
- `2*sizeof(void*)`:DTV(动态线程向量)的头部空间(`dtv[0]` 存计数 + `dtv[1]` 主模块指针,这里预留了 2 项)。
- `sizeof(struct pthread)`:线程控制块 (TCB),TP 通常指向它。
- `TLS_ABOVE_TP` 下额外加 `main_tls.offset`(TP 之上的 gap)。
- `main_tls.size`:实际 TLS 数据。
- `main_tls.align`:额外对齐余量,保证运行时能把数据块对齐到任意起点。
- 最后 `+ MIN_TLS_ALIGN-1 & -MIN_TLS_ALIGN`:把总大小**向上取整**到 `MIN_TLS_ALIGN` 的倍数。

⚠️ **运算符优先级要点**:C 中 `+`/`-` 优先级高于 `&`,所以整个表达式等价于
`(2*sizeof(void*) + ... + main_tls.align + MIN_TLS_ALIGN-1) & (-MIN_TLS_ALIGN)`
即"把前面所有项之和向上取整到 `MIN_TLS_ALIGN`"。这是 musl 里刻意利用优先级、看起来"缺括号"实则正确的写法。

### 5. 分配 TLS 内存

```c
if (libc.tls_size > sizeof builtin_tls) {
    mem = (void *)__syscall(SYS_mmap2, 0, libc.tls_size,
        PROT_READ|PROT_WRITE, MAP_ANONYMOUS|MAP_PRIVATE, -1, 0);
} else {
    mem = builtin_tls;
}
```

- **小尺寸优化**:若所需大小不超过内置静态缓冲 `builtin_tls`(`struct { char c; struct pthread pt; void *space[16]; }`),直接用它,**避免启动早期就发起 `mmap` 系统调用**——这在很多程序(TLS 很小)中省去一次内核往返。
- 否则用 `mmap` 匿名私有映射分配。
- 注释说明:不检查错误码——失败时返回值 `-4095..-1` 作为指针后续解引用会直接崩溃,故不额外膨胀启动代码去显式检查。

### 6. 复制 TLS 镜像并设置线程指针

```c
if (__init_tp(__copy_tls(mem)) < 0)
    a_crash();
```

- **`__copy_tls(mem)`**:在这块内存里按 ABI 布局排布 TCB、DTV 和 TLS 数据:
  - 计算 DTV 位置与对齐后的 `td`(`struct pthread` 指针);
  - 遍历 `libc.tls_head` 链表(此处只有 `main_tls`),把每个模块的 `image` 复制 `len` 字节到对应偏移处(`.tbss` 部分因为是 mmap/静态零页,天然为 0),并填 `dtv[i]`(加上架构相关 `DTP_OFFSET`);
  - `dtv[0] = tls_cnt`,`td->dtv = dtv`,返回 `td`。
- **`__init_tp(td)`**:设置线程指针,完成 TCB 初始化:
  - `td->self = td`;
  - `__set_thread_area(TP_ADJ(td))` 通过系统调用把 TP 寄存器指向 TCB(`TP_ADJ` 在 `TLS_ABOVE_TP` 时会把指针 +`sizeof(struct pthread)`+`TP_OFFSET`);
  - 返回 0 表示内核支持,置 `libc.can_do_threads = 1`;
  - 设置 `detach_state=DT_JOINABLE`、通过 `SYS_set_tid_address` 取得 `tid`、绑定 `locale`、初始化 robust futex 链表头(自指)、`sysinfo`、把线程链表 `next/prev` 指向自身(主线程构成单元素环)。
- 任何一步失败(`__init_tp` 返回 <0)都 `a_crash()`——**TLS/TP 初始化失败视为致命**,因为之后 errno、canary、线程等都无法工作。

---

## 与动态链接的关系

`static_init_tls` 是 `__init_tls` 的**弱别名**。在动态链接程序中,动态链接器 (`ldso/dynlink.c`) 会提供一个**强符号** `__init_tls`,处理多个共享库各自的 `PT_TLS`、构建更长的 tls_module 链表,从而**覆盖**这个静态版本。静态链接时没有 ldso,就使用这里的 `static_init_tls`,只处理主程序单个 TLS 模块。

---

## 关键设计要点小结

1. **单模块假设**:静态场景只有主程序一个 TLS 模块(`tls_cnt=1`),逻辑因此得以简化。
2. **两种 ABI 布局**:`TLS_ABOVE_TP` 决定 TLS 在 TP 之上还是之下,以及 `GAP_ABOVE_TP`、`TP_OFFSET`、`DTP_OFFSET` 等架构常量的使用。
3. **启动性能**:内置 `builtin_tls` 缓冲避免小 TLS 情况下的 mmap 调用。
4. **紧凑的位运算对齐**:大量使用 `(-x) & (align-1)` 和依赖运算符优先级的对齐取整,牺牲可读性换取精简的启动代码。
5. **fail-fast**:不检查 mmap 错误、初始化失败直接 `a_crash()`,保持早期启动代码极简。
