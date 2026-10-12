# Linux 0.01 系统调用机制
{docsify-updated}

`include/linux/sys.h` 中定义了系统调用表：维护了各个系统调用函数的地址。

`kernel/sched.c` 的 `sched_init` 方法中 `set_system_gate(0x80,&system_call);` 设置系统调用门， `int 0x80` 会被 `system_call` 处理。

`kernel/system_call.s` 中的 `system_call` 会根据 `eax` 的值调用系统调用表中的函数: `call *sys_call_table(,%eax,4)` ，以下是 `system_call` 的汇编代码：
```
system_call:
	cmpl $nr_system_calls-1,%eax
	ja bad_sys_call
	push %ds
	push %es
	push %fs
	pushl %edx
	pushl %ecx		# push %ebx,%ecx,%edx as parameters
	pushl %ebx		# to the system call
	movl $0x10,%edx		# set up ds,es to kernel space
	mov %dx,%ds
	mov %dx,%es
	movl $0x17,%edx		# fs points to local data space
	mov %dx,%fs

	movl sys_call_table(,%eax,4),%edx	# if sys_null
	cmpl $sys_null,%edx					# then the syscall is not
	jne 1f								# implemented.
	popl  %edx							# 1st argument is syscall nr
	pushl %eax
1:

	call *sys_call_table(,%eax,4)
	pushl %eax
	movl current,%eax
	cmpl $0,state(%eax)		# state
	jne reschedule
	cmpl $0,counter(%eax)		# counter
	je reschedule
ret_from_sys_call:
	movl current,%eax		# task[0] cannot have signals
	cmpl task,%eax
	je 3f
	movl CS(%esp),%ebx		# was old code segment supervisor
	testl $3,%ebx			# mode? If so - don't check signals
	je 3f
	cmpw $0x17,OLDSS(%esp)		# was stack segment = 0x17 ?
	jne 3f
2:	movl signal(%eax),%ebx		# signals (bitmap, 32 signals)
	bsfl %ebx,%ecx			# %ecx is signal nr, return if none
	je 3f
	btrl %ecx,%ebx			# clear it
	movl %ebx,signal(%eax)
	movl sig_fn(%eax,%ecx,4),%ebx	# %ebx is signal handler address
	cmpl $1,%ebx
	jb default_signal		# 0 is default signal handler - exit
	je 2b				# 1 is ignore - find next signal
	movl $0,sig_fn(%eax,%ecx,4)	# reset signal handler address
	incl %ecx
	xchgl %ebx,EIP(%esp)		# put new return address on stack
	subl $28,OLDESP(%esp)
	movl OLDESP(%esp),%edx		# push old return address on stack
	pushl %eax			# but first check that it's ok.
	pushl %ecx
	pushl $28
	pushl %edx
	call verify_area
	popl %edx
	addl $4,%esp
	popl %ecx
	popl %eax
	movl restorer(%eax),%eax
	movl %eax,%fs:(%edx)		# flag/reg restorer
	movl %ecx,%fs:4(%edx)		# signal nr
	movl EAX(%esp),%eax
	movl %eax,%fs:8(%edx)		# old eax
	movl ECX(%esp),%eax
	movl %eax,%fs:12(%edx)		# old ecx
	movl EDX(%esp),%eax
	movl %eax,%fs:16(%edx)		# old edx
	movl EFLAGS(%esp),%eax
	movl %eax,%fs:20(%edx)		# old eflags
	movl %ebx,%fs:24(%edx)		# old return addr
3:	popl %eax
	popl %ebx
	popl %ecx
	popl %edx
	pop %fs
	pop %es
	pop %ds
	iret
```


## 使用汇编进行系统调用
```
.data

msg:
    .ascii "Hello, world!\n"
    len = . - msg

.text
    .global _start

_start:
    movq  $1, %rax
    movq  $1, %rdi
    movq  $msg, %rsi
    movq  $len, %rdx
    syscall

    movq  $60, %rax
    xorq  %rdi, %rdi
    syscall
```

```
$ gcc -c test.S
$ ld -o test test.o
```