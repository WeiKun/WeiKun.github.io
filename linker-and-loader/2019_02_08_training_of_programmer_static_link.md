# 《程序员的自我修养》读书笔记之"静态链接"

为了弥补自己相关知识的不足，最近在看《程序员的自我修养——链接、装载与库》，这是第一部分的（不算简介部分的话）的读书笔记

[TOC]


## 一次“编译”过程发生的事

以一段最简单的代码的gcc编译作为例子

```c
// File Name: hello.c
// Author: weikun

#include <stdio.h>

int main()
{
    printf("Hello World!");
    return 0;
}
```

使用 gcc 编译，会输出 a.out。


```
gcc hello.c
```

实际上这一句指令包含了四个阶段（预编译、编译、汇编、链接）的行为，而这四个阶段的行为均可以单独拆分执行。

### 01 - 预编译

预编阶段分执行的行为比较单纯，主要是将宏展开、处理预编译指令、展开include跟删除注释，主要是文本级的处理。

```
gcc -E hello.c -o hello.i
```

### 02 - 编译

编译阶段进行词法分析、语法分析、语义分析跟优化，然后生成汇编代码。

```
gcc -S hello.i -o hello.s

或者

/usr/lib/gcc/x86_64-linux-gnu/6/cc1 -I/usr/include/x86_64-linux-gnu/ hello.c
```

如果执行下面那句指令，会输出

```
 main
Analyzing compilation unit
Performing interprocedural optimizations
 <*free_lang_data> <visibility> <build_ssa_passes> <opt_local_passes> <targetclone> <free-inline-summary> <whole-program> <inline>Assembling functions:
 <simdclone> main
Execution times (seconds)
 phase setup             :   0.00 ( 0%) usr   0.00 ( 0%) sys   0.00 ( 0%) wall    1071 kB (65%) ggc
 phase parsing           :   0.00 ( 0%) usr   0.00 ( 0%) sys   0.01 (100%) wall     521 kB (31%) ggc
 phase opt and generate  :   0.00 ( 0%) usr   0.01 (100%) sys   0.00 ( 0%) wall      54 kB ( 3%) ggc
 lexical analysis        :   0.00 ( 0%) usr   0.00 ( 0%) sys   0.01 (100%) wall       0 kB ( 0%) ggc
 initialize rtl          :   0.00 ( 0%) usr   0.01 (100%) sys   0.00 ( 0%) wall      12 kB ( 1%) ggc
 TOTAL                 :   0.00             0.01             0.01               1656 kB
```

### 03 - 汇编

汇编阶段执行的行为也比较单纯，将汇编语句转变成可执行的机器指令。

```
gcc -c hello.s -o hello.o
```

### 04 - 链接

这一步里除了 hello.o，其他 .o 跟 lib 的路径在不同环境下可能不一样。

这一阶段将用到的模块就行拼装，执行地址和空间分配、符号决议跟重定位等过程，输出完整的可执行文件。

```
ld -static  /usr/lib/x86_64-linux-gnu/crt1.o \
            /usr/lib/x86_64-linux-gnu/crti.o \
            /usr/lib/gcc/x86_64-linux-gnu/6/crtbeginT.o hello.o \
            -start-group -lgcc -lgcc_eh -lc -end-group \
            /usr/lib/gcc/x86_64-linux-gnu/6/crtend.o \
            /usr/lib/x86_64-linux-gnu/crtn.o
```

**从书名可以知道，书里对这里的编译等环节并不太关注，比较关注链接环节，因此着重对链接的操作文件以及链接时发生的行为进行了分析。**

## .o 文件的构成

对于链接来说，输入就是汇编过程产生的各种 .o 文件，输出是可执行文件。

用书上建议的示例代码进行分析

```c
// File Name: SimpleSection.c
// Author: weikun

int printf(const char *format, ...);

int global_init_var = 84;
int global_unint_var;

void func1(int i)
{
    printf("%d\n", i);
}

int main(void)
{
    static int static_var = 85;
    static int static_var2;

    int a = 1;
    int b;

    func1(static_var + static_var2 + a + b);
    return a;
}
```

用file指令可以知道 .o 文件是ELF（Executable Linkable Format）格式的。

```
debian:~/LearnTest/StaticLink/3$ file SimpleSection.o 
SimpleSection.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

### 主要结构段

首先用 objdump 工具来分析下 .o 文件的结构

```
debian:~/LearnTest/StaticLink/3$ objdump -h SimpleSection.c.o 

SimpleSection.c.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000057  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000008  0000000000000000  0000000000000000  00000098  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000004  0000000000000000  0000000000000000  000000a0  2**2
                  ALLOC
  3 .rodata       00000004  0000000000000000  0000000000000000  000000a0  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000002e  0000000000000000  0000000000000000  000000a4  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000d2  2**0
                  CONTENTS, READONLY
  6 .eh_frame     00000058  0000000000000000  0000000000000000  000000d8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```
可以看到，.o 文件里存在代码段（.text）、数据段（.data）、BSS段、只读数据段（.rodata）、注释信息段（.comment）、堆栈提示段（.note.GNU-stack）、异常处理段（.eh_frame）。

上面解析出的信息里，"File off"是对应段在文件中的起始字节数，"Size"是对应段的数据长度。

（从 .text 的 size 跟 .data 的起始字节，可以看到 size 本身是没有进行字节填充的，但是各个段在文件中的分布是进行了字节对齐的。


#### 代码段(.text)

编译后的机器指令一般放在代码段。

指令

```
objdump -d SimpleSection.c.o
```
可以查看代码段的内容，不过没什么特殊，确实有两个函数。:)

#### 数据段(.rdata) & 只读数据段(.rodata)

全局变量跟局部静态变量通常放在数据段

指令

```
objdump -x -s SimpleSection.c.o 
```
可以查看段的详细信息，里面可以看到

```
Contents of section .data:
 0000 54000000 55000000                    T...U...        
Contents of section .rodata:
 0000 25640a00                             %d..     
```
0x54000000转换字节序之后是0x00000054=84，其实就是global_init_var的值，0x55000000也就是static_var的值。

而"%d"正好是"printf("%d\n", i);"里的字符串常量。


#### BSS段(.bss)

按道理BSS段是未初始化的全局变量和局部静态变量的预留位置。

但是从

```
  3 .rodata       00000004  0000000000000000  0000000000000000  000000a0  2**0
```

可以看出，实际上满足BSS条件的global_unint_var跟static_var2只有一个保存在了BSS段。

从后续章节可知，global_unint_var应该只预留了一个未定义的全局变量符号，并没有在BSS里占有空间。

（如果同样代码的cpp文件，编译之后BSS段有8个字节，原因未知）


#### 其他段

不列举了。


### ELF文件描述

#### ELF Header

在上面的解析中可以知道，.text作为第一个段，其起始字节也是0x00000040字节了。

```
  0 .text         00000057  0000000000000000  0000000000000000  00000040  2**0
```
在0x00000040之前，就是ELF Header了，可以通过"readelf -h"查看
```
debian:~/LearnTest/StaticLink/3$ readelf -h SimpleSection.c.o 
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          1112 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         13
  Section header string table index: 12
```
这里包含的信息有ELF魔数、文件机器字节长度、数据存储方式、版本、程序入口及长度、段表入口及长度等。

#### 段表

"objdump -h"查看的是主要段表，可以用"readelf -S"可以看到所有段的信息。
```
debian:~/LearnTest/StaticLink/3$ readelf -S SimpleSection.c.o  
There are 13 section headers, starting at offset 0x458:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000057  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000348
       0000000000000078  0000000000000018   I      10     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000098
       0000000000000008  0000000000000000  WA       0     0     4
  [ 4] .bss              NOBITS           0000000000000000  000000a0
       0000000000000004  0000000000000000  WA       0     0     4
  [ 5] .rodata           PROGBITS         0000000000000000  000000a0
       0000000000000004  0000000000000000   A       0     0     1
  [ 6] .comment          PROGBITS         0000000000000000  000000a4
       000000000000002e  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000d2
       0000000000000000  0000000000000000           0     0     1
  [ 8] .eh_frame         PROGBITS         0000000000000000  000000d8
       0000000000000058  0000000000000000   A       0     0     8
  [ 9] .rela.eh_frame    RELA             0000000000000000  000003c0
       0000000000000030  0000000000000018   I      10     8     8
  [10] .symtab           SYMTAB           0000000000000000  00000130
       0000000000000198  0000000000000018          11    11     8
  [11] .strtab           STRTAB           0000000000000000  000002c8
       000000000000007b  0000000000000000           0     0     1
  [12] .shstrtab         STRTAB           0000000000000000  000003f0
       0000000000000061  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```
不过跟原著的不同，我验证时发现section headers的入口地址已经是最后一个段的结束位置（加上字节对齐），而不是原著的symtab之前，当然各段之间的排布也有出入，想来是gcc版本升级以后的缘故。不过文件的大小依然是各段的长度（加上字节对齐）加上段表长度的和。

```
-rw-r--r-- 1  1944 16:57 SimpleSection.c.o
1112 + 64 * 13 = 1944
```
说明对.o文件结构的分析已经完整了。

## 静态链接

链接主要是对上阶段的结果——.o文件全部合并起来，操作起来分为空间与地址分配、符号解析与重定位两步。同时针对C++会有重复代码消除跟全局构造析构。

照例使用原著的示例代码

```
// File Name: a.c
// Author: weikun

extern int shared;

int main(int argc, char *argv[])
{
    int a = 100;
    swap(&a, &shared);
    return 0 ;
}
```

```
// File Name: b.c
// Author: weikun

int shared = 1;

void swap(int *a, int *b)
{
    *a ^= *b ^= *a ^= *b;
}
```
将两个.c文件编译成.o然后链接起来

```
gcc -c a.c
gcc -c b.c
ld a.o b.o -e main -o ab
```
然后使用objdump -h进行解析

### 空间与地址分配

"objdump -h a.o" 显示

```
a.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000035  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  00000075  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000075  2**0
                  ALLOC
  3 .comment      0000002e  0000000000000000  0000000000000000  00000075  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000a3  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000038  0000000000000000  0000000000000000  000000a8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```

"objdump -h b.o" 显示

```
b.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000004b  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  0000000000000000  0000000000000000  0000008c  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000090  2**0
                  ALLOC
  3 .comment      0000002e  0000000000000000  0000000000000000  00000090  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000be  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000038  0000000000000000  0000000000000000  000000c0  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA

```

"objdump -h ab" 显示

```
ab:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000080  00000000004000e8  00000000004000e8  000000e8  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .eh_frame     00000058  0000000000400168  0000000000400168  00000168  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .data         00000004  0000000000601000  0000000000601000  00001000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  3 .comment      0000002d  0000000000000000  0000000000000000  00001004  2**0
                  CONTENTS, READONLY
```

可以看到，链接前各段的VMA都为0，而链接后各段都赋予了非0的VMA。这是由于虚拟空间是在链接时分配的。

同时，链接后的ab的.text的size是80，也是a.o跟b.o的.text的size之和。


### 重定位

再对比a.o跟ab的反汇编之后main函数的异同：

```
0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   48 83 ec 20             sub    $0x20,%rsp
   8:   89 7d ec                mov    %edi,-0x14(%rbp)
   b:   48 89 75 e0             mov    %rsi,-0x20(%rbp)
   f:   c7 45 fc 64 00 00 00    movl   $0x64,-0x4(%rbp)
  16:   48 8d 45 fc             lea    -0x4(%rbp),%rax
  1a:   48 8d 35 00 00 00 00    lea    0x0(%rip),%rsi        # 21 <main+0x21>
  21:   48 89 c7                mov    %rax,%rdi
  24:   b8 00 00 00 00          mov    $0x0,%eax
  29:   e8 00 00 00 00          callq  2e <main+0x2e>
  2e:   b8 00 00 00 00          mov    $0x0,%eax
  33:   c9                      leaveq 
  34:   c3                      retq   
```

```
00000000004000e8 <main>:
  4000e8:       55                      push   %rbp
  4000e9:       48 89 e5                mov    %rsp,%rbp
  4000ec:       48 83 ec 20             sub    $0x20,%rsp
  4000f0:       89 7d ec                mov    %edi,-0x14(%rbp)
  4000f3:       48 89 75 e0             mov    %rsi,-0x20(%rbp)
  4000f7:       c7 45 fc 64 00 00 00    movl   $0x64,-0x4(%rbp)
  4000fe:       48 8d 45 fc             lea    -0x4(%rbp),%rax
  400102:       48 8d 35 f7 0e 20 00    lea    0x200ef7(%rip),%rsi        # 601000 <shared>
  400109:       48 89 c7                mov    %rax,%rdi
  40010c:       b8 00 00 00 00          mov    $0x0,%eax
  400111:       e8 07 00 00 00          callq  40011d <swap>
  400116:       b8 00 00 00 00          mov    $0x0,%eax
  40011b:       c9                      leaveq 
  40011c:       c3                      retq 
```

可以看到两处被修改了

```
1a:           48 8d 35 00 00 00 00    lea    0x0(%rip),%rsi        # 21 <main+0x21>
400102:       48 8d 35 f7 0e 20 00    lea    0x200ef7(%rip),%rsi        # 601000 <shared>


29:           e8 00 00 00 00          callq  2e <main+0x2e>
400111:       e8 07 00 00 00          callq  40011d <swap>
```
其中前一处可以通过"objdump -x ab"可以看到是shared变量的地址，后一处的40011d可以从反汇编里看到，是swap函数的入口地址。说明在链接前，a.o是不知道这两者的地址的（也无从知道），此时用的是一些临时的家地址来替代。（具体的假地址值跟原著有所不同，应该是gcc版本不同所致）。

这一切都是通过ELF文件里的重定位段（.rel.text）来操作的。可以通过"objdump -r"来查看需要重定位的内容。

```
debian:~/LearnTest/StaticLink/4$ objdump -r a.o 

a.o:     file format elf64-x86-64

RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE 
000000000000001d R_X86_64_PC32     shared-0x0000000000000004
000000000000002a R_X86_64_PLT32    swap-0x0000000000000004
```
里面指引的每一段内容都是需要进行重定位的，OFFSET指向重定位的代码偏移，VALUE前半段指向一个符号，存在于符号表里

```
debian:~/LearnTest/StaticLink/4$ readelf -s a.o 

Symbol table '.symtab' contains 12 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS a.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     8: 0000000000000000    53 FUNC    GLOBAL DEFAULT    1 main
     9: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND shared
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND swap
    
debian:~/LearnTest/StaticLink/4$ readelf -s ab  

Symbol table '.symtab' contains 13 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000004000e8     0 SECTION LOCAL  DEFAULT    1 
     2: 0000000000400168     0 SECTION LOCAL  DEFAULT    2 
     3: 0000000000601000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS a.c
     6: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS b.c
     7: 000000000040011d    75 FUNC    GLOBAL DEFAULT    1 swap
     8: 0000000000601000     4 OBJECT  GLOBAL DEFAULT    3 shared
     9: 0000000000601004     0 NOTYPE  GLOBAL DEFAULT    3 __bss_start
    10: 00000000004000e8    53 FUNC    GLOBAL DEFAULT    1 main
    11: 0000000000601004     0 NOTYPE  GLOBAL DEFAULT    3 _edata
    12: 0000000000601008     0 NOTYPE  GLOBAL DEFAULT    3 _end
```
可以看到a.o里有shared跟swap两个符号是UND的，因此链接时只需要根据重定位段去全局符号表里查相关符号的地址，然后进行重定位替换。

### C++相关

对于C++，链接时还会针对各种自动生成的重复代码（虚表、模板等）进行去重，避免浪费跟错误。同时加入".init"跟".fini"进行main函数的执行外执行（就是执行main前构造全局变量，执行main之后析构全局变量）

另外由于C++的ABI兼容性存在问题，因此选择gcc版本的时候一定要确认版本的ABI跟需要链接的库的ABI是兼容的。

