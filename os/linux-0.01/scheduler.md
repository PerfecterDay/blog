# 进程调度
{docsify-updated}

核心文件 `kernel/sched.c`, 初始化时会安装一个定时器中断处理程序 `timer_interrupt`。
```
void sched_init(void)
{
	int i;
	struct desc_struct * p;

	set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
	set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));
	p = gdt+2+FIRST_TSS_ENTRY;
	for(i=1;i<NR_TASKS;i++) {
		task[i] = NULL;
		p->a=p->b=0;
		p++;
		p->a=p->b=0;
		p++;
	}
	ltr(0);
	lldt(0);
	outb_p(0x36,0x43);		/* binary, mode 3, LSB/MSB, ch 0 */
	outb_p(LATCH & 0xff , 0x40);	/* LSB */
	outb(LATCH >> 8 , 0x40);	/* MSB */
	set_intr_gate(0x20,&timer_interrupt);
	outb(inb_p(0x21)&~0x01,0x21);
	set_system_gate(0x80,&system_call);
}
```

`kernel/system_call.s` 有 `timer_interrupt` 的汇编定义:
```
.align 2
timer_interrupt:
	push %ds		# save ds,es and put kernel data space
	push %es		# into them. %fs is used by _system_call
	push %fs
	pushl %edx		# we save %eax,%ecx,%edx as gcc doesn't
	pushl %ecx		# save those across function calls. %ebx
	pushl %ebx		# is saved as we use that in ret_sys_call
	pushl %eax
	movl $0x10,%eax
	mov %ax,%ds
	mov %ax,%es
	movl $0x17,%eax
	mov %ax,%fs
	incl jiffies
	movb $0x20,%al		# EOI to interrupt controller #1
	outb %al,$0x20
	movl CS(%esp),%eax
	andl $3,%eax		# %eax is CPL (0 or 3, 0=supervisor)
	pushl %eax
	call do_timer		# 'do_timer(long CPL)' does everything from
	addl $4,%esp		# task switching to accounting ...
	jmp ret_from_sys_call
```

最终会调用到 `do_timer`:
```
void do_timer(long cpl)
{
	if (cpl)
		current->utime++;
	else
		current->stime++;
	if ((--current->counter)>0) return;
	current->counter=0;
	if (!cpl) return;
	schedule();
}
```

如果当前进程的时间片没有用完，就返回；如果当前处于内核态（CPL=0）,也直接返回，不调度；否则调用 `schedule()` 切换到其它进程。这里注意，CPL 是发生时钟中断前正在执行的current进程当前处于用户态还是内核态。

```
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;

/* check alarm, wake up any interruptible tasks that have got a signal */

	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
		if (*p) {
			if ((*p)->alarm && (*p)->alarm < jiffies) {
					(*p)->signal |= (1<<(SIGALRM-1));
					(*p)->alarm = 0;
				}
			if ((*p)->signal && (*p)->state==TASK_INTERRUPTIBLE)
				(*p)->state=TASK_RUNNING;
		}

/* this is the scheduler proper: */

	while (1) {
		c = -1;
		next = 0;
		i = NR_TASKS;
		p = &task[NR_TASKS];
		while (--i) {
			if (!*--p)
				continue;
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
				c = (*p)->counter, next = i;
		}
		if (c) break;
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
			if (*p)
				(*p)->counter = ((*p)->counter >> 1) +
						(*p)->priority;
	}
	switch_to(next);
}
```

`switch_to` 是一个宏，定义在 `include/linux/sched.h` 中：
```
#define switch_to(n) {\
    struct {long a,b;} __tmp; \
    __asm__ __volatile__(
        "cmpl %%ecx,current\n\t"
        "je 1f\n\t"
        "xchgl %%ecx,current\n\t"
        "movw %%dx,%1\n\t"
        "ljmp *%0\n\t"
        "cmpl %%ecx,%2\n\t"
        "jne 1f\n\t"
        "clts\n"
        "1:\n\t"
        :
        : "m" (*&__tmp.a),
          "m" (*&__tmp.b),
          "m" (last_task_used_math),
          "d" _TSS(n),
          "c" ((long) task[n])
    ); \
}
```
这段代码是 Linux 0.01 中最核心的进程切换代码之一。它利用 x86 的硬件任务切换机制，通过修改 current、执行远跳转 `ljmp` ，让 CPU 从当前任务切换到目标任务。

1. 第一阶段：判断是否真的需要切换
```
cmpl %%ecx,current
je 1f
```

先看输入操作数： `"c" ((long) task[n])` ，这会让编译器把 `task[n]` 的地址放入 ECX。因此，执行时：
```
ECX     = task[n] 的地址
current = 当前正在运行的任务地址
```

2. 第二阶段：更新 `current`
`xchgl %%ecx,current` : `xchg` 会交换两个操作数的值。`current` 被交换成新任务的地址，而 `ECX` 被交换成旧任务的地址，这个值也有后续用途：硬件任务切换时会保存当前任务的寄存器状态，而目标任务恢复时，会恢复它自己此前保存的 ECX。

3. 第三阶段：构造远跳转指针
```
movw %%dx,%1
ljmp *%0
```

输入操作数： `"d" _TSS(n)`,将 `DX` 中的 16 位 TSS 选择子写入 `__tmp.b` 的低 16 位。于是，__tmp 的内存布局可以表示为：
```
__tmp
低地址
┌────────────────────────────┐
│ a：32 位偏移字段            │
├────────────────────────────┤
│ b 的低 16 位：TSS 选择子    │
├────────────────────────────┤
│ b 的高 16 位                 │
└────────────────────────────┘
高地址
```
这里有一个容易忽略的细节：`__tmp.a` 并没有被显式初始化。但在 32 位保护模式下，当 `ljmp` 的目标选择子指向 TSS 描述符时，CPU 执行的是任务切换，远指针中的偏移字段不会作为普通代码地址执行。

`ljmp *%0` 到底做了什么？ 这是一个通过内存中的远指针执行的间接远跳转。普通远跳转可能是：加载新的 `CS:EIP`。

但这里的选择子指向的是 `TSS` 描述符，所以 CPU 会识别出这不是普通代码段跳转，而是硬件任务切换。其效果可以简化表示为：
```
ljmp 指向目标 TSS
         │
         ▼
CPU 查找 GDT 中的 TSS 描述符
         │
         ▼
保存当前任务的硬件上下文
         │
         ▼
从目标任务的 TSS 恢复上下文
         │
         ▼
继续执行目标任务之前保存的指令位置
```
注意， `ljmp` 不会像普通 `call` 那样调用目标函数。**它触发的是 x86 处理器提供的硬件任务切换机制**。在 32 位 x86 保护模式下，TSS 保存着硬件任务切换所需的状态，包括：
- EIP：任务恢复后从哪里继续执行；
- EFLAGS：标志寄存器；
- EAX、EBX、ECX、EDX、ESI、EDI、EBP、ESP：通用寄存器；
- CS、SS、DS、ES、FS、GS：段寄存器；
- CR3：页目录基址寄存器；
- LDTR：局部描述符表寄存器等状态。
具体保存和恢复哪些状态，由 32 位 x86 硬件任务切换机制规定