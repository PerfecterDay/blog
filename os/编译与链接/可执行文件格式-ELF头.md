#  可执行文件格式-ELF头
{docsify-updated}

> https://linux-audit.com/elf-binaries-on-linux-understanding-and-analysis/  
> https://man7.org/linux/man-pages/man5/elf.5.html  
> https://bbs.kanxue.com/thread-274573.htm

## ELF文件格式 
目标文件同时参与了链接与运行，ELF文件同时支持两种功能。所以可以从两个视角来看待ELF文件：
<center>
<img src="pics/elf.png" width="40%">
<img src="pics/elf-view.png" width="30%">
</center>

`_start` 不是 ELF 标准规定的入口名字，ELF 标准只规定 `e_entry` ； `_start` 是 GNU/Linux 工具链约定出来的默认入口符号。你完全可以用 `-e` 指定其他入口。

## ELF Header
```
The following types are used for N-bit architectures (N=32,64,ElfN stands for Elf32 or Elf64, uintN_t stands for uint32_t or uint64_t):
ElfN_Addr       Unsigned program address, uintN_t
ElfN_Off        Unsigned file offset, uintN_t
ElfN_Section    Unsigned section index, uint16_t
ElfN_Versym     Unsigned version symbol information, uint16_t
Elf_Byte        unsigned char
ElfN_Half       uint16_t
ElfN_Sword      int32_t
ElfN_Word       uint32_t
ElfN_Sxword     int64_t
ElfN_Xword      uint64_t

#define EI_NIDENT 16

typedef struct {
	unsigned char e_ident[EI_NIDENT];
	uint16_t      e_type;
	uint16_t      e_machine;
	uint32_t      e_version;
	ElfN_Addr     e_entry;
	ElfN_Off      e_phoff;
	ElfN_Off      e_shoff;
	uint32_t      e_flags;
	uint16_t      e_ehsize;
	uint16_t      e_phentsize;
	uint16_t      e_phnum;
	uint16_t      e_shentsize;
	uint16_t      e_shnum;
	uint16_t      e_shstrndx;
} ElfN_Ehdr;
```

+ `e_type` -标识目标文件的类型：
  1. `ET_NONE（0）`：未知类型
  2. `ET_REL（1）`：A relocatable file. 可重定位文件
  3. `ET_EXEC（2）`：可执行文件
  4. `ET_DYN （3）`：共享目标文件
  5. `ET_CORE（4）`：coredump 文件
+ `e_entry`-该成员给出了启动进程的虚拟地址。如果文件没有相关的入口点，则该成员的值为零。
+ `e_phoff`-该成员保存program header table 在文件中的偏移量（以字节为单位字节）。如果文件没有程序头表，该值为零。
+ `e_shoff`-该成员保存section header table 在文件中的偏移量（以字节为单位）。如果文件没有段头表，该值为零。
+ `e_ehsize`-ELF 头的大小，字节为单位。
+ `e_phentsize`-指定了 program header table 中每条记录的大小，所有记录大小一样。
+ `e_phnum`-指定了 program header table 中有多少条记录，最大值为 PN_XNUM (0xffff)，e_phnum*e_phentsize = program header table的大小。如果超过了PN_XNUM，sh_info 会有额外指定
+ `e_shentsize`-指定了 section header table 中每条记录的大小，所有记录大小一样。
+ `e_shnum`-指定了 section header table 中有多少条记录，最大值为SHN_LORESERVE (0xff00)，e_shentsize*e_shnum=section header table的大小如果超过了SHN_LORESERVE， sh_size 会有额外指定
