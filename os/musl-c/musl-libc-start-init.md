# `libc_start_init` 函数详细分析

## 概述

`libc_start_init` 位于 `src/env/__libc_start_main.c`,是 musl 在程序启动流程中负责**运行 C 运行时的初始化构造器 (constructors)** 的函数——即执行 `_init`(旧式 `.init` 段)以及 `.init_array` 中登记的所有初始化函数(包括 C++ 全局对象构造函数、`__attribute__((constructor))` 标记的函数等)。

它通过 `weak_alias` 被暴露为 `__libc_start_init`:

```c
static void libc_start_init(void)
{
	_init();
	uintptr_t a = (uintptr_t)&__init_array_start;
	for (; a<(uintptr_t)&__init_array_end; a+=sizeof(void(*)()))
		(*(void (**)(void))a)();
}
weak_alias(libc_start_init, __libc_start_init);
```

---

## 逐行解析

### 1. `_init();` —— 执行旧式 `.init` 段

```c
_init();
```

- `_init` 是传统 SysV ABI 的初始化入口,对应 ELF 的 `.init` 段。它由架构相关的 `crti.s`(段起始/函数序言)与 `crtn.s`(段结尾/函数收尾)拼接组成,中间由编译器/链接器插入 `.init` 段片段。例如 s390x 的 `crti.s`:

```asm
.section .init
.global _init
_init:
	stmg %r14, %r15, 112(%r15)
	...
```

- 在本文件中,`_init` 有一个 **weak dummy 定义**作为兜底(当程序没有链接 crti/crtn 时):

```c
static void dummy(void) {}
weak_alias(dummy, _init);
```

- `_init` 是**遗留机制**。现代工具链主要用 `.init_array`,`_init`/`.init` 逐渐被淘汰。在标记了 `NO_LEGACY_INITFINI` 的架构(如 aarch64)上,动态链接器版本甚至完全跳过它。

### 2. 遍历 `.init_array` 并逐个调用

```c
uintptr_t a = (uintptr_t)&__init_array_start;
for (; a<(uintptr_t)&__init_array_end; a+=sizeof(void(*)()))
	(*(void (**)(void))a)();
```

- `__init_array_start` 和 `__init_array_end` 是链接器脚本(以及 crt)提供的符号,分别标记 `.init_array` 段的**起止边界**。该段是一个**函数指针数组**,每项类型为 `void (*)(void)`。

```c
extern weak hidden void (*const __init_array_start)(void), (*const __init_array_end)(void);
```

- 它们被声明为 `weak hidden`:`weak` 保证程序即使没有任何初始化函数(两符号相等,循环体不执行)也能链接通过;`hidden` 表示内部可见性。
- 循环逻辑:从 `start` 到 `end`,按指针大小 `sizeof(void(*)())` 步进,把当前地址 `a` 解释为"指向函数指针的指针"`(void (**)(void))a`,解引用取出函数指针,再 `()` 调用它。
- **调用顺序是从前往后**(低地址到高地址),与析构方向相反。对应的清理端 `libc_exit_fini`(在 `src/exit/exit.c`)则**从后往前**遍历 `.fini_array`,最后调用 `_fini`,顺序完全对称。

---

## 在启动流程中的位置

它由 `__libc_start_init()` 被调用,而后者在 `libc_start_main_stage2` 中、**真正进入 `main` 之前**被调用:

```c
static int libc_start_main_stage2(...)
{
	char **envp = argv+argc+1;
	__libc_start_init();
	/* Pass control to the application */
	exit(main(argc, argv, envp));
	return 0;
}
```

完整调用链(静态链接场景):

```
_start (crt_arch.h 汇编)
  → __libc_start_main(main, argc, argv, ...)
      → __init_libc(envp, argv[0])   // 环境、auxv、TLS、SSP、安全检查
      → stage2 = libc_start_main_stage2   // 编译器屏障,防止 init 栈帧长期驻留
          → __libc_start_init()
              → libc_start_init()    // 本函数:_init() + .init_array
          → exit(main(...))          // 进入用户 main
```

⚠️ **关键时序**:`libc_start_init` 在 `__init_libc`(其中已完成 `__init_tls`、`__init_ssp`)**之后**运行。`__libc_start_main` 中特意用内联汇编屏障隔开两个阶段:

```c
/* Barrier against hoisting application code or anything using ssp
 * or thread pointer prior to its initialization above. */
lsm2_fn *stage2 = libc_start_main_stage2;
__asm__ ( "" : "+r"(stage2) : : "memory" );
```

这保证:用户构造函数(可能使用栈保护 canary、线程指针 TLS)一定在 TLS/SSP 初始化**之后**才运行,编译器不会把 stage2 的代码提前。

---

## 与动态链接的关系(弱符号覆盖)

`libc_start_init` 是 `__libc_start_init` 的**弱别名**。在动态链接程序里,动态链接器 `ldso/dynlink.c` 提供一个**强符号** `__libc_start_init`,从而覆盖本文件的静态版本:

```c
void __libc_start_init(void)
{
	do_init_fini(main_ctor_queue);
	if (!__malloc_replaced && main_ctor_queue != builtin_ctor_queue)
		free(main_ctor_queue);
	main_ctor_queue = 0;
}
```

两个版本的差异:

| | 静态版 `libc_start_init` | 动态版(dynlink.c) |
|---|---|---|
| 处理对象 | 仅主程序自身的 `_init` + `.init_array` | 整个依赖图(主程序 + 所有共享库) |
| 顺序 | 简单线性遍历 | `do_init_fini` 按依赖拓扑排序 (`main_ctor_queue`),保证被依赖的库先构造 |
| 遗留 `_init` | 直接调用 `_init()` | 通过 `DT_INIT` 动态标签处理,且 `NO_LEGACY_INITFINI` 架构跳过 |
| 并发 | 无锁 | 有 `init_fini_lock`/`ctor_cond`,处理 `dlopen` 并发构造 |

动态版的 `do_init_fini` 对每个 DSO 读取其动态段,依次执行 `DT_INIT`(旧式)和 `DT_INIT_ARRAY`:

```c
if (dyn[0] & (1<<DT_INIT_ARRAY)) {
	size_t n = dyn[DT_INIT_ARRAYSZ]/sizeof(size_t);
	size_t *fn = laddr(p, dyn[DT_INIT_ARRAY]);
	while (n--) ((void (*)(void))*fn++)();
}
```

---

## 关键设计要点小结

1. **两类初始化机制**:先跑遗留的 `_init`(`.init` 段),再跑现代的 `.init_array` 函数指针数组;两者并存以兼容不同工具链和历史代码。
2. **`weak_alias` 覆盖模式**:静态链接用本文件的简单实现;动态链接用 ldso 提供的强符号版本(支持多 DSO 依赖排序与并发)。这是 musl 广泛使用的"弱默认 + 强覆盖"模式(`_init`、`__init_tls` 亦然)。
3. **边界符号 + weak**:`__init_array_start/end` 用 weak 声明,保证无初始化函数时也能正确链接且循环为空。
4. **严格时序**:必须在 TLS 与 SSP 初始化之后、`main` 之前运行,`__libc_start_main` 用编译器内存屏障强制这一顺序,防止用户构造函数在运行时环境就绪前被调用。
5. **与 fini 对称**:`.init_array` 正序执行、`libc_exit_fini` 中 `.fini_array` 逆序执行,`_init`/`_fini` 亦对称。
