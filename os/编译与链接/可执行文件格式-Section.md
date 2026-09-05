# 可执行文件格式-Section
{docsify-updated}

### Section Header
```
typedef struct {
	uint32_t   sh_name;
	uint32_t   sh_type;
	uint32_t   sh_flags;
	Elf32_Addr sh_addr;
	Elf32_Off  sh_offset;
	uint32_t   sh_size;
	uint32_t   sh_link;
	uint32_t   sh_info;
	uint32_t   sh_addralign;
	uint32_t   sh_entsize;
} Elf32_Shdr;
```
<center><img src="pics/section-header.png" width="60%"></center>

1. sh_type 类型
<div><img src="pics/program-header-type.png" alt=""></div>

1. sh_flag 节标志
<div><img src="pics/section-flag.png" alt=""></div>


#### 常见section

<table>
<thead>
<tr class="header">
<th style="border: 1px solid; padding: 0.2rem 1.5rem">名字</th>
<th style="border: 1px solid; padding: 0.2rem 1.5rem">类型</th>
<th style="border: 1px solid; padding: 0.2rem 1.5rem">属性</th>
<th style="border: 1px solid; padding: 0.2rem 1.5rem">意义</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.init</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC +
SHF_EXECINSTR</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">此节包含进程初始化时要执行的程序指令。当程序开始运行时，系统会在进
入主函数之前执行这一节中的代码。</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.fini</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC +
SHF_EXECINSTR</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">此节包含进程终止时要执行的程序指令。当程序正常退出时，系统会执行这
一节中的代码。</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.bss</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_NOBITS</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC+SHF_WRITE</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节中包含目标文件中未初始化的全局变量。一般情况下，可执行程序在开
始运行的时候，系统会把这一段内容清零。但是，在运行期间的 bss
段是由系统初 始化而成的，在目标文件中.bss 节并不包含任何内容，其长度为
0，所以它的节类 型为 SHT_NOBITS。</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.comment</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">无</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节包含版本控制信息</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.data/.data1</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC+SHF_WRITE</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">这两个节用于存放程序中被初始化过的全局变量。在目标文件中，它们是占
用实际的存储空间的，与.bss 节不同。</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.debug</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">无</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">调试信息，内容格式没有统一规定。所有以”.debug”为前缀的节名
字都是保留</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.line</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">无</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节也是一个用于调试的节，它包含那些调试符号的行号，为程序指令码与
源文件的行号建立起联系。其内容格式没有统一规定。</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.dynamic</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_DYNAMIC</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">见下文</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节包含动态连接信息，并且可能有
SHF_ALLOC 和 SHF_WRITE 等属性。 是否具有 SHF_WRITE
属性取决于操作系统和处理器。</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.dynstr</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_STRTAB</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">此节含有用于动态连接的字符串，一般是那些与符号表相关的名字</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.dynsym</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_DYNSYM</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">此节含有动态连接符号表</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.got</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC +
SHF_WRITE</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">此节包含全局偏移量表(Global Offset Table，GOT)</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.hash</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_HASH</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节包含一张符号哈希表</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.interp</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">见下文</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.interp的内容很简单，里面保存的就是一个字符串，这个字符串就是可执行文件所需要的动态链接器的路径</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.note</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_NOTE</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">无</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">注释节</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.plt</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC +
SHF_EXECINSTR</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">此节包含函数连接表(Procedure Linkage Table)</td>
</tr>
<tr class="even">
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">.rel.data/.rel.text</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_REL/SHT_RELA</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">见下文</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">这两个节含有重定位信息。如果此节被包含在某个可装载的段中，那么本节
的属性中应置 SHF_ALLOC 标志位，否则不置此标志。</td>
</tr>
<tr class="even">
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">.rel.dyn/.rel.plt</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_REL/SHT_RELA</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">见下文</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">动态链接的文件中，也有类似的重定位表分别叫做“.rel.dyn”和“.rel.plt”，它们分别相当于“.rel.text”和“.rel.data”。“.rel.dyn”实际上是对数据引用的修正，它所修正的位置位于“.got”以及数据段；而“.rel.plt”是对函数引用的修正，它所修正的位置位于“.got.plt”。</td>
</tr>
<tr class="odd">
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">.rodata/.rodata1</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节包含程序中的只读数据，在程序装载时，它们一般会被装入进程空间中
那些只读的段中去</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.shstrtab</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_STRTAB</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">无</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节是“节名字表”，含有所有其它节的名字</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.strtab</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_STRTAB</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">见下文</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节用于存放字符串，主要是那些符号表项的名字。如果一个目标文件有一
个可装载的段，并且其中含有符号表，那么本节的属性中应该有 SHF_ALLOC</td>
</tr>
<tr class="even">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.symtab</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_SYMTAB</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">见下文</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节用于存放符号表。如果一个目标文件有一个可载入的段，并且其中含有
符号表，那么本节的属性中应该有 SHF_ALLOC。</td>
</tr>
<tr class="odd">
<td style="border: 1px solid; padding: 0.2rem 1.5rem">.text</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHT_PROGBITS</td>
<td style="border: 1px solid; padding: 0.2rem 1.5rem">SHF_ALLOC +
SHF_EXECINSTR</td>
<td
style="border: 1px solid; padding: 0.2rem 1.5rem">本节包含程序指令代码</td>
</tr>
</tbody>
</table>