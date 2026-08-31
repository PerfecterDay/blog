# Linux 0.01 boot.s 详细分析

`boot/boot.s` 是软盘的第一个扇区（512 字节）引导代码，由 BIOS 加载到物理地址 `0x7C00` 后执行，负责把内核加载进内存并把 CPU 从实模式切入保护模式。

## 一、boot.s 的总体使命

BIOS 把它加载到物理地址 `0x7C00` 后执行。它的完整任务是：

1. 把自己从 `0x7C00` 搬到 `0x90000`；
2. 用 BIOS 中断把内核 `system` 加载到 `0x10000`；
3. 关中断，把 `system` 搬到物理地址 `0x0000`；
4. 加载 GDT/IDT、开 A20、重编程 8259；
5. 切入保护模式，跳转到 `system` 的入口（`head.s`）。

> 编译细节：Makefile 在编译前会先生成 `SYSSIZE = (...+15)/16` 一行拼到 boot.s 前面，所以 `SYSSIZE` 是内核占用的**节（16 字节）数**。它用 as86/ld86 编译成 512 字节的引导扇区。

## 二、段地址常量

```asm
BOOTSEG = 0x07c0   ; boot.s 被 BIOS 加载的段地址 (0x07c0<<4 = 0x7C00)
INITSEG = 0x9000   ; boot.s 要把自己搬去的段地址 (0x90000)
SYSSEG  = 0x1000   ; system 被加载到 0x10000
ENDSEG  = SYSSEG + SYSSIZE  ; system 加载后的结束段地址
```

实模式下**物理地址 = 段 × 16 + 偏移**。所以 `0x07c0:0000` 就是物理地址 `0x7C00`。

## 三、自我搬移（核心代码）

```asm
start:
	mov	ax,#BOOTSEG
	mov	ds,ax          ; ds = 0x07c0 (源)
	mov	ax,#INITSEG
	mov	es,ax          ; es = 0x9000 (目的)
	mov	cx,#256        ; 256 个字 = 512 字节
	sub	si,si          ; si = 0
	sub	di,di          ; di = 0
	rep
	movw               ; ds:si -> es:di, 复制 256 字
	jmpi	go,INITSEG     ; 段间跳转到 0x9000:go
```

**逐行含义：**

- `ds:si = 0x07c0:0` 指向自身（源），`es:di = 0x9000:0` 指向目标 `0x90000`；
- `cx=256`，`rep movw` 复制 256 个字（=512 字节），把整个引导扇区搬到 `0x90000`；
- `jmpi go,INITSEG` 是**段间跳转**，同时把 CS 设为 `0x9000`，让 CPU 到搬移后的副本继续执行 `go` 标号。

**为什么要搬走？** 因为内核马上要被加载到 `0x10000` 并最终搬到 `0x0000`，`0x7C00` 这块低地址区要腾出来给缓冲/内核使用。搬到 `0x90000`（接近 640KB 低端内存顶部）可避免冲突。

## 四、go：重设段与栈、打印信息

```asm
go:	mov	ax,cs          ; cs 现在是 0x9000
	mov	ds,ax
	mov	es,ax
	mov	ss,ax
	mov	sp,#0x400      ; 栈设在 0x9000:0x400，足够大且 >512
```

- 把 `ds/es/ss` 全部设为 `cs`（0x9000），统一到新位置；
- 设置栈指针 `sp=0x400`（1KB），栈位于搬移后代码上方；
- 随后用 BIOS `int 0x10`（功能 0x03 读光标、功能 0x1301 写字符串）打印 `"Loading system ..."`。

## 五、加载内核 system

```asm
	mov	ax,#SYSSEG
	mov	es,ax          ; es = 0x1000，目标段
	call	read_it        ; 把 system 读到 0x10000
	call	kill_motor     ; 关软驱马达
```

- `read_it` 用 BIOS `int 0x13` 从软盘读扇区，把内核加载到 `0x10000`；
- 读完保存光标位置到 `[510]`（即 `0x90510`），供后面 `con_init` 使用；
- `kill_motor` 关闭软驱马达，让内核在一个已知的干净状态下启动。

### read_it / read_track 的读盘逻辑

- 用 `sread`（当前磁道已读扇区）、`head`（磁头）、`track`（磁道）三个变量跟踪进度；
- 关键约束：每次不能跨越 **64KB 段边界**（`test ax,#0x0fff` / `die` 死循环检查 es 必须在 64KB 边界），因为实模式段寻址在 64KB 处回绕；
- 尽量**整磁道读取**以提高速度（`sectors=18` 对应 1.44MB 软盘的每磁道 18 扇区）；
- `read_track` 出错则 `int 0x13` 复位磁盘并重试，持续读错会形成不可打断的死循环（注释里明说了——只能手动重启）。

## 六、进入保护模式的准备

```asm
	cli                ; 关中断，后面要重排中断，不能被打断
```

### ① 把 system 搬到物理地址 0x0000

```asm
do_move:
	mov	es,ax          ; 目的段 (从 0x0000 开始)
	add	ax,#0x1000
	cmp	ax,#0x9000
	jz	end_move
	mov	ds,ax          ; 源段 (从 0x1000 开始)
	...
	mov	cx,#0x8000     ; 每次 0x8000 字 = 64KB
	rep	movsw
	j	do_move
```

以 64KB 为单位，把 `0x10000` 起的内核循环搬到 `0x00000`，共搬 8 段（0x1000→0x9000 之前）。搬到 0 地址是因为保护模式下内核将从**线性地址 0** 开始执行。

### ② 加载 IDT 和 GDT

```asm
	lidt	idt_48     ; 加载 IDT（limit=0，base=0，即暂时空表）
	lgdt	gdt_48     ; 加载 GDT
```

文件末尾定义的 `gdt` 有 3 项：

- 项 0：空描述符（x86 规定）；
- 项 1（选择子 `0x08`）：代码段，基址 0，界限 8MB，可读可执行；
- 项 2（选择子 `0x10`）：数据段，基址 0，界限 8MB，可读可写。

### ③ 开启 A20 地址线

```asm
	call	empty_8042
	mov	al,#0xD1        ; 命令：写输出端口
	out	#0x64,al
	call	empty_8042
	mov	al,#0xDF        ; 打开 A20
	out	#0x60,al
```

通过键盘控制器 8042 打开 A20，才能访问 1MB 以上内存（否则地址在 1MB 处回绕，这是 8086 兼容遗留问题）。`empty_8042` 循环等待 8042 输入缓冲为空。

### ④ 重新编程 8259 中断控制器

```asm
	mov	al,#0x11        ; 初始化序列 ICW1
	out	#0x20,al        ; 主 8259
	...
	mov	al,#0x20        ; 硬件中断起始向量号 0x20
	out	#0x21,al
	mov	al,#0x28        ; 从片起始 0x28
	out	#0xA1,al
	...
	mov	al,#0xFF        ; 暂时屏蔽所有中断
	out	#0x21,al
	out	#0xA1,al
```

BIOS 默认把硬件中断放在 `0x08~0x0F`，这与保护模式下 Intel 保留的 CPU 异常向量冲突。这里把主/从 8259 的中断向量重映射到 `0x20~0x2F`，并**先屏蔽全部硬件中断**（等内核准备好再开）。注意那些 `.word 0x00eb,0x00eb` 是 `jmp $+2` 短跳延时，给慢速的 8259 端口操作留出稳定时间。

## 七、切入保护模式（最后一跃）

```asm
	mov	ax,#0x0001     ; PE 位 = 1
	lmsw	ax             ; This is it! 进入保护模式
	jmpi	0,8            ; 跳到选择子 8（代码段）:偏移 0
```

- `lmsw ax` 把 CR0 的 PE（Protection Enable）位置 1，CPU **正式进入保护模式**；
- `jmpi 0,8` 是保护模式下的段间跳转：选择子 `8` 就是 GDT 里的代码段，偏移 `0`；
- 由于 system 已被搬到物理地址 0、代码段基址也是 0，这一跳就精确落到 `head.s` 的 `startup_32`，控制权正式交给 32 位内核。

## 八、文件尾部的数据结构

```asm
gdt:      ...              ; 全局描述符表（3 项：空/代码/数据）
idt_48:   .word 0; .word 0,0    ; IDT 描述符，limit=0（空 IDT）
gdt_48:   .word 0x800; .word gdt,0x9  ; GDT 描述符，base=0x90000+gdt
msg1:     .ascii "Loading system ..." ; 启动提示
```

`gdt_48` 里 `0x9` 说明 GDT 的基址高位是 `0x9xxxx`，因为整段代码此刻运行在 `0x90000`。

## 九、执行时序总结

```
0x7C00 (BIOS加载)
   │ rep movw 自我搬移
   ▼
0x90000 (go: 设段/栈 → 打印 → int13读内核)
   │ 内核 → 0x10000
   │ cli → rep movsw 把内核搬到 0x0000
   │ lidt/lgdt → 开A20 → 重编程8259
   ▼
lmsw(PE=1) + jmpi 0,8
   ▼
head.s @ startup_32 (保护模式, 32位)
```

**几个设计精髓值得注意：**

- **两次搬移**（0x7C00→0x90000，0x10000→0x0000）是为了在实模式有限的内存布局里给内核腾出线性地址 0 起始的空间；
- **64KB 边界检查**（`die` 死循环）是实模式段寻址的硬约束；
- **先关中断再重排 8259**，避免保护模式下中断向量与 CPU 异常冲突；
- 全部塞进 **512 字节**，所以代码极度精简（注释里 Linus 说 "had to, to get it in 512 bytes"）。
