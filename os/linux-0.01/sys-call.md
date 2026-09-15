`include/linux/sys.h` 中定义了系统调用表

`kernel/sched.c` 的 `sched_init` 方法中 `set_system_gate(0x80,&system_call);` 设置系统调用门， `int 0x80` 会被 `system_call` 处理。

`kernel/system_call.s` 中的 `system_call` 会根据 `eax` 的值调用系统调用表中的函数: `call *sys_call_table(,%eax,4)`