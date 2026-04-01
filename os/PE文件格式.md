# Windows PE（Portable-Executable）[文件格式](https://bbs.pediy.com/thread-270210.htm)
PE文件的结构如下:
<center><img src="pics/pe-file.png" width=30% ></center>

其中，真正的段开始之前的部分（上图中.text之前）的详细结构如下：
<center><img src="pics/Portable_Executable_32_bit_Structure_in_SVG_fixed.svg" width=80% ></center>

1. COFF HEADER(IMAGE_FILE_HEADER)
   ```
	typedef struct _IMAGE_FILE_HEADER{
		WORD Machine;                               // +0x00, 指定程序的运行平台(386/x64)
		WORD NumberOfSections;          // +0x02, PE中的节/块(section)数量
		DWORD TimeDateStamp;                // +0x04, 时间戳：链接器填写的文件生成时间
		DWORD PointerToSymbolTable;  // +0x08, 指向符号表的地址(主要用于调试)
		DWORD NumberOfSymbols;          // +0x0C, 符号表中符号个数(同上)
		WORD SizeOfOptionalHeader;  // +0x10, IMAGE_OPTIONAL_HEADER32选项头结构大小，勿改
		WORD Characteristics;               // +0x12, 文件属性，勿改
	} IMAGE_FILE_HEADER, *PIMAGE_FILE_HEADER;
   ```

2. Optional Header(IMAGE_OPTIONAL_HEADER32)
   ```
   typedef struct _IMAGE_OPTIONAL_HEADER
	{
		WORD   Magic;                            //* PE标志字：32位（0x10B），64位（0x20B）
		BYTE   MajorLinkerVersion;               //  主链接器版本号
		BYTE   MinorLinkerVersion;               //  副链接器版本号
		DWORD  SizeOfCode;                        //  代码所占空间大小（代码节大小）
		DWORD  SizeOfInitializedData;         //  已初始化数据所占空间大小
		DWORD  SizeOfUninitializedData;           //  未初始化数据所占空间大小
		DWORD  AddressOfEntryPoint;               //* 程序执行入口RVA，(w)(Win)mainCRTStartup：即0D首次断下来的自进程地址
		DWORD  BaseOfCode;                        //  代码段基址
		DWORD  BaseOfData;                        //  数据段基址
		DWORD  ImageBase;                     //* 内存加载基址，exe默认0x400000，dll默认0x10000000
		DWORD  SectionAlignment;              //* 节区数据在内存中的对齐值，一定是4的倍数，一般是0x1000(4096=4K)
		DWORD  FileAlignment;                 //* 节区数据在文件中的对齐值，一般是0x200(磁盘扇区大小512)
		WORD   MajorOperatingSystemVersion;      //  要求操作系统最低版本号的主版本号
		WORD   MinorOperatingSystemVersion;      //  要求操作系统最低版本号的副版本号
		WORD   MajorImageVersion;                //  可运行于操作系统的主版本号
		WORD   MinorImageVersion;                //  可运行于操作系统的次版本号
		WORD   MajorSubsystemVersion;            //  主子系统版本号：不可修改
		WORD   MinorSubsystemVersion;            //  副子系统版本号
		DWORD  Win32VersionValue;             //  版本号：不被病毒利用的话一般为0,XP中不可修改
		DWORD  SizeOfImage;                       //* PE文件在进程内存中的总大小，与SectionAlignment对齐
		DWORD  SizeOfHeaders;                 //* PE文件头部在文件中的按照文件对齐后的总大小（所有头 + 节表）
		DWORD  CheckSum;                      //  对文件做校验，判断文件是否被修改：3环无用，MapFileAndCheckSum获取
		WORD   Subsystem;                        //  子系统，与连接选项/system相关：1=驱动程序，2=图形界面，3=控制台/Dll
		WORD   DllCharacteristics;               //  文件特性
		DWORD  SizeOfStackReserve;                //  初始化时保留的栈大小
		DWORD  SizeOfStackCommit;             //  初始化时实际提交的栈大小
		DWORD  SizeOfHeapReserve;             //  初始化时保留的堆大小
		DWORD  SizeOfHeapCommit;              //  初始化时实际提交的堆大小
		DWORD  LoaderFlags;                       //  已废弃，与调试有关，默认为 0
		DWORD  NumberOfRvaAndSizes;               //  下边数据目录的项数，此字段自Windows NT发布以来,一直是16
		IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];// 数据目录表
	} IMAGE_OPTIONAL_HEADER32, * PIMAGE_OPTIONAL_HEADER32;
	```
	 IMAGE_DATA_DIRECTORY: 数据目录用来描述PE中各个表的位置及大小信息，重点表有导出表、导入表、重定位表、资源表。
	```
	typedef struct _IMAGE_DATA_DIRECTORY {
		DWORD VirtualAddress;     /**指向某个数据的相对虚拟地址   RAV  偏移0x00**/
		DWORD Size;               /**某个数据块的大小                 偏移0x04**/
	} IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;
	```

3. Section Table(IMAGE_SECTION_HEADER)
   ```
   typedef struct _IMAGE_SECTION_HEADER {
		BYTE  Name[IMAGE_SIZEOF_SHORT_NAME];  // 节表名称：描述性字段
		// 下方4个字段：从文件S1处开始，拷贝S2大小的数据，到内存S3处，有效数据占用内存S4大小
		union {
			DWORD PhysicalAddress;
			DWORD VirtualSize;         // S4:内存大小
		} Misc;
		DWORD VirtualAddress;          // S3:内存地址：基于模块基址
		DWORD SizeOfRawData;           // S2:文件大小
		DWORD PointerToRawData;        // S1:文件偏移
		DWORD PointerToRelocations;    // 无用
		DWORD PointerToLinenumbers;    // 无用
		WORD  NumberOfRelocations;    // 无用
		WORD  NumberOfLinenumbers;    // 无用
		DWORD Characteristics;     // 节属性，取值IMAGE_SCN_...系列宏
	} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;
   ```
