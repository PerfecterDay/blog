#  GCC-内联汇编
{docsify-updated}

> https://gcc.gnu.org/onlinedocs/gcc/extensions-to-the-c-language-family/how-to-use-inline-assembly-language-in-c-code.html#clobbers-and-scratch-registers

- [GCC-内联汇编](#gcc-内联汇编)
  - [基本 asm](#基本-asm)
  - [扩展 Asm - 带有 C 表达式操作数的汇编器指令](#扩展-asm---带有-c-表达式操作数的汇编器指令)
    - [asm-qualifiers:](#asm-qualifiers)
    - [AssemblerTemplate-汇编模版](#assemblertemplate-汇编模版)
    - [OutputOperands](#outputoperands)
    - [InputOperands](#inputoperands)
    - [Clobbers](#clobbers)
    - [GotoLabels](#gotolabels)
    - [为本地变量指定寄存器 (https://gcc.gnu.org/onlinedocs/gcc/extensions-to-the-c-language-family/how-to-use-inline-assembly-language-in-c-code.html#local-register-variables)](#为本地变量指定寄存器-httpsgccgnuorgonlinedocsgccextensions-to-the-c-language-familyhow-to-use-inline-assembly-language-in-c-codehtmllocal-register-variables)
  - [系统调用](#系统调用)

### 基本 asm
基本 asm 语句的语法如下：
```
asm asm-qualifiers ( AssemblerInstructions )
```
对于 C 语言，asm 关键字是 GNU 扩展。当编写可以用 -ansi 和 -std 选项编译的 C 语言代码时，如果选择的 C 语言方言没有 GNU 扩展，请使用 `__asm__` 代替 asm（请参阅替代关键字）。对于 C++ 语言，asm 是一个标准关键字，但对于用 -fno-asm 编译的代码，可以使用 __asm__ 。

asm-qualifiers 可以是下述选项：
+ `volatile`: 可选的 `volatile` 限定符不起作用。所有基本 asm 块都是隐式 `volatile` 的。
+ `inline`: 如果使用了 `inline` 限定符，则为了内联的目的，asm 语句的大小将被视为尽可能小的大小。

AssemblerInstructions 
这是一个指定汇编器代码的字面字符串。该字符串可以包含汇编程序识别的任何指令，包括指令。GCC 不会解析汇编指令本身，因此不知道它们的含义，甚至不知道它们是否是有效的汇编输入。  
你可以将多条汇编指令放在一个 asm 字符串中，并用系统汇编代码中通常使用的字符分隔。在大多数情况下，换行符和制表符（书写为 \n\t）是一种有效的组合。有些汇编程序允许使用分号作为分隔符。不过，请注意有些汇编语言使用分号来开始注释。


### 扩展 Asm - 带有 C 表达式操作数的汇编器指令
使用扩展 asm，你可以从汇编程序中读写 C 变量，并执行从汇编代码到 C 标签的跳转。扩展 asm 语法使用冒号（:）在汇编模板后分隔操作数参数：
```
asm asm-qualifiers ( AssemblerTemplate
                 : OutputOperands
                 [ : InputOperands
                 [ : Clobbers ] ])

asm asm-qualifiers ( AssemblerTemplate
                      : OutputOperands
                      : InputOperands
                      : Clobbers
                      : GotoLabels)
```

#### asm-qualifiers:
+ `volatile`: 扩展 asm 语句的典型用途是操作输入值以产生输出值。然而，您的 asm 语句也可能产生副作用。如果是这样，您可能需要使用 volatile 限定符来禁用某些优化。
+ `inline`: 如果使用了 `inline` 限定符，则为了内联的目的，asm 语句的大小将被视为尽可能小的大小。
+ `goto`: 该限定符通知编译器，asm 语句可以跳转到 GotoLabels 。

#### AssemblerTemplate-汇编模版
这是一个字面字符串，是汇编程序代码的模板。它是固定文本与指代输入、输出和 goto 参数的标记的组合。汇编模板是包含汇编指令的字面字符串。编译器会替换模板中指向输入、输出和 goto 标签的标记，然后将生成的字符串输出给汇编器。该字符串可以包含汇编器识别的任何指令。GCC 不会解析汇编指令本身，也不知道这些指令的含义，甚至不知道它们是否是有效的汇编输入。不过，它可以计算语句。

你可以将多条汇编指令放在一个 asm 字符串中，用系统汇编代码中通常使用的字符分隔。在大多数情况下，可以使用换行符来分行，再加上一个制表符来移动到指令字段（写成 \n\t）。有些汇编程序允许使用分号作为分隔符。不过，请注意有些汇编语言使用分号来开始注释。

即使使用 volatile 限定符，也不要指望一连串 asm 语句在编译后保持完全连续。如果某些指令需要在输出中保持连续，可将它们放在一条多指令 asm 语句中。

在不使用输入/输出操作数的情况下访问 C 程序中的数据（如直接从汇编模板中使用全局符号），可能无法达到预期效果。同样，直接从汇编器模板调用函数也需要详细了解目标汇编器和 ABI。

由于 GCC 不解析汇编模板，因此无法看到它引用的任何符号。这可能导致 GCC 将这些符号视为未引用符号，除非它们也被列为输入、输出或 goto 操作数。

汇编模版中的一些特殊字符串：
+ `%%`: 在汇编代码中输出单个 %。
+ `%=`: 在整个编译过程中，为 asm 语句的每个实例输出一个唯一的编号。当创建本地标签并在生成多条汇编指令的单个模板中多次引用这些标签时，该选项非常有用。
+ `%{ %| %}`: 将 {、| 和 } 字符（分别）输出到汇编代码中。在未转义时，这些字符具有特殊含义。

#### OutputOperands
以逗号分隔的 C 变量列表，其中包含被 AssemblerTemplate 中的指令修改过的 C 变量。允许使用空列表。

在这个 i386 示例中，old（在模板字符串中称为 %0）和 *Base（称为 %1）是输出，Offset (%2) 是输入：
```
bool old;

__asm__ ("btsl %2,%1\n\t" // Turn on zero-based bit #Offset in Base.
         "sbb %0,%0"      // Use the CF to calculate old.
   : "=r" (old), "+rm" (*Base)
   : "Ir" (Offset)
   : "cc");

return old;
```

操作数之间用逗号隔开。每个操作数的格式如下:
```
[ [asmSymbolicName] ] constraint (cvariablename)
```
+ asmSymbolicName
  指定操作数的符号名称。在汇编模板中引用该名称时，用方括号将其括起来（如：%[Value]）。名称的作用域是包含定义的 asm 语句。可以使用任何有效的 C 变量名，包括周围代码中已定义的名称。同一 asm 语句中的两个操作数不能使用相同的符号名称。  
  不使用 asmSymbolicName 时，请使用操作数在汇编模板操作数列表中的位置（基于零）。例如，如果有三个输出操作数，在模板中使用 %0 表示第一个操作数，使用 %1 表示第二个操作数，使用 %2 表示第三个操作数。
+ constraint
  一个字符串常量，用于指定操作数位置的限制条件。  
  输出约束必须以 =（变量覆盖现有值）或 +（读写时）开头。使用 = 时，不要假定该位置包含进入 asm 时的现有值，除非操作数与输入绑定。
  在前缀之后，必须有一个或多个附加约束来描述值的位置。常见的约束包括表示寄存器的 `r` 和表示内存的 `m`。如果列出多个可能的位置（例如"=rm"），编译器会根据当前上下文选择最有效的位置。如果在 asm 语句允许的范围内列出尽可能多的备选位置，就能让优化器生成最佳代码。如果必须使用特定寄存器，但 "机器约束 "无法提供足够的控制来选择所需的特定寄存器，那么局部寄存器变量可能是一种解决方案。
+ cvariablename
  指定一个 C lvalue 表达式来保存输出，通常是一个变量名。括号是语法的必要组成部分。

编译器在选择用于表示输出操作数的寄存器时，不会使用任何 clobbered 寄存器。  
输出操作数表达式必须是 l 值。编译器无法检查操作数的数据类型是否符合正在执行的指令。对于不可直接寻址的输出表达式（例如位字段），约束必须允许使用寄存器。在这种情况下，GCC 会使用寄存器作为 asm 的输出，然后将寄存器存储到输出中。

#### InputOperands
以逗号分隔的 C 表达式列表，输入操作数将 C 语言变量和表达式中的值提供给汇编代码, 由 AssemblerTemplate 中的指令读取。允许使用空列表。
```
[ [asmSymbolicName] ] constraint (cexpression)
```

+ asmSymbolicName
  指定操作数的符号名称。在汇编模板中引用该名称时，用方括号将其括起来（即 %[Value]）。名称的作用域是包含定义的 asm 语句。可以使用任何有效的 C 变量名，包括周围代码中已定义的名称。同一 asm 语句中的两个操作数不能使用相同的符号名称。  
  不使用 asmSymbolicName 时，应使用操作数在汇编模板操作数列表中的位置（基于零）。例如，如果有两个输出操作数和三个输入操作数，在模板中使用 %2 表示第一个输入操作数，使用 %3 表示第二个操作数，使用 %4 表示第三个操作数。
+ constraint
  一个字符串常量，用于指定操作数位置的限制条件。  
  输出约束必须以 =（变量覆盖现有值）或 +（读写时）开头。使用 = 时，不要假定该位置包含进入 asm 时的现有值，除非操作数与输入绑定。
  在前缀之后，必须有一个或多个附加约束来描述值的位置。常见的约束包括表示寄存器的 `r` 和表示内存的 `m`。如果列出多个可能的位置（例如"=rm"），编译器会根据当前上下文选择最有效的位置。如果在 asm 语句允许的范围内列出尽可能多的备选位置，就能让优化器生成最佳代码。如果必须使用特定寄存器，但 "机器约束 "无法提供足够的控制来选择所需的特定寄存器，那么局部寄存器变量可能是一种解决方案。
+ cvariablename
  这是作为输入传递给 asm 语句的 C 变量或表达式。括号是语法的必要组成部分。




#### Clobbers
以逗号分隔的寄存器或其他由 AssemblerTemplate 更改的值的列表，不包括列为输出的值。允许使用空列表。

#### GotoLabels
在使用 asm 的 goto 形式时，本部分包含 AssemblerTemplate 中的代码可能跳转到的所有 C 标签的列表。asm 语句不能跳转到其他 asm 语句，只能跳转到列出的 GotoLabels。GCC 的优化器不知道其他跳转，因此在决定如何优化时无法考虑这些跳转。

input + output + goto操作数的总数限制为 30。


#### 为本地变量指定寄存器 (https://gcc.gnu.org/onlinedocs/gcc/extensions-to-the-c-language-family/how-to-use-inline-assembly-language-in-c-code.html#local-register-variables)
```
register int *foo asm ("r12");
register int *p1 asm ("r0") = ...;
register int *p2 asm ("r1") = ...;
register int *result asm ("r0");
asm ("sysint" : "=r" (result) : "0" (p1), "r" (p2));
```

### 系统调用
```
#include <unistd.h>  // 包含系统调用相关的头文件
#include <sys/types.h>
#include <sys/syscall.h>
#include <stdarg.h>
#include <signal.h>
#include <stdio.h>
 
// 自定义的write函数，封装了系统调用
ssize_t my_write(int fd, const void *buf, size_t count) {
    // 使用syscall直接进行系统调用
    return syscall(SYS_write, fd, buf, count);
}

void handler(int signalN){
	printf("Received signal:%d", signalN);
}
 
int main() {
    const char *message = "Hello\r\n";
    // 使用自定义的write函数来写入数据到标准输出
    my_write(STDOUT_FILENO, message, sizeof(message));

	signal(SIGALRM,handler);
	alarm(1);
	pause();
    return 0;
}
```