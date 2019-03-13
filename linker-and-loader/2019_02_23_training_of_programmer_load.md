# 《程序员的自我修养》读书笔记之"装载与动态链接"

## 装载

CPU只能访问到内存（其实应该是Cache，这里就不在意细节了），而程序文件跟所需资源是存储在硬盘上的，所以需要先将程序装载到内存。

但是程序所需的内存可能大于实际内存，因此产生了动态装载的技术，典型的就是覆盖装入跟页映射两种，不细表。

### 装载过程发生的事

进程建立的时候，会先创建独立的虚拟地址空间，然后读取可执行文件头，并建立虚拟空间与可执行文件的映射关系，再讲CPU指令寄存器设置成可执行文件的入口地址，启动运行。

其中创建虚拟地址空间跟设置指令寄存器不需要特殊讨论，建立映射关系的目的是为了动态装载，由于我们可执行文件没有一次性加载到内存，因此需要能在发现程序某页数据缺失时有方法找到这部分数据在文件中的位置，然后加载。

### 虚地址分布

按道理，ELF文件的每一个Section加载到内存，而加载的时候是以页为单位的，由于不同的Section有不同的权限，因此需要分开加载到不同页，这样每一个Section都有可能在最后一个页上浪费空间，浪费的期望值是页大小的二分之一，Section一多，浪费的就非常惊人了。

为了解决这个问题，一般是把同权限的Section映射到连续的一块空间上，组合成Segment（其实链接器一般就会把可以拼成Segment的Section排在相邻位置）。

测试文件sleepmain.c

```C++
#include<stdlib.h>

int main()
{
    while (1)
    {
        sleep(1);
    }
    return 0;
}
```

运行结果

```shell
$ gcc -static sleepmain.c -o sleepmain.elf
$ readelf -S sleepmain.elf      

There are 32 section headers, starting at offset 0xc5658:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.ABI-tag     NOTE             0000000000400190  00000190
       0000000000000020  0000000000000000   A       0     0     4
  [ 2] .note.gnu.build-i NOTE             00000000004001b0  000001b0
       0000000000000024  0000000000000000   A       0     0     4
readelf: Warning: [ 3]: Link field (0) should index a symtab section.
  [ 3] .rela.plt         RELA             00000000004001d8  000001d8
       0000000000000108  0000000000000018  AI       0    24     8
  [ 4] .init             PROGBITS         00000000004002e0  000002e0
       0000000000000017  0000000000000000  AX       0     0     4
  [ 5] .plt              PROGBITS         0000000000400300  00000300
       00000000000000b0  0000000000000000  AX       0     0     16
  [ 6] .text             PROGBITS         00000000004003b0  000003b0
       00000000000885a7  0000000000000000  AX       0     0     16
  [ 7] __libc_freeres_fn PROGBITS         0000000000488960  00088960
       0000000000000ab7  0000000000000000  AX       0     0     16
  [ 8] __libc_thread_fre PROGBITS         0000000000489420  00089420
       00000000000000e1  0000000000000000  AX       0     0     16
  [ 9] .fini             PROGBITS         0000000000489504  00089504
       0000000000000009  0000000000000000  AX       0     0     4
  [10] .rodata           PROGBITS         0000000000489520  00089520
       000000000001c724  0000000000000000   A       0     0     32
  [11] __libc_subfreeres PROGBITS         00000000004a5c48  000a5c48
       0000000000000050  0000000000000000   A       0     0     8
  [12] __libc_IO_vtables PROGBITS         00000000004a5ca0  000a5ca0
       00000000000006a8  0000000000000000   A       0     0     32
  [13] __libc_atexit     PROGBITS         00000000004a6348  000a6348
       0000000000000008  0000000000000000   A       0     0     8
  [14] __libc_thread_sub PROGBITS         00000000004a6350  000a6350
       0000000000000008  0000000000000000   A       0     0     8
  [15] .eh_frame         PROGBITS         00000000004a6358  000a6358
       000000000000abfc  0000000000000000   A       0     0     8
  [16] .gcc_except_table PROGBITS         00000000004b0f54  000b0f54
       00000000000000af  0000000000000000   A       0     0     1
  [17] .tdata            PROGBITS         00000000006b1eb8  000b1eb8
       0000000000000020  0000000000000000 WAT       0     0     8
  [18] .tbss             NOBITS           00000000006b1ed8  000b1ed8
       0000000000000030  0000000000000000 WAT       0     0     8
  [19] .init_array       INIT_ARRAY       00000000006b1ed8  000b1ed8
       0000000000000010  0000000000000008  WA       0     0     8
  [20] .fini_array       FINI_ARRAY       00000000006b1ee8  000b1ee8
       0000000000000010  0000000000000008  WA       0     0     8
  [21] .jcr              PROGBITS         00000000006b1ef8  000b1ef8
       0000000000000008  0000000000000000  WA       0     0     8
  [22] .data.rel.ro      PROGBITS         00000000006b1f00  000b1f00
       00000000000000e4  0000000000000000  WA       0     0     32
  [23] .got              PROGBITS         00000000006b1fe8  000b1fe8
       0000000000000008  0000000000000008  WA       0     0     8
  [24] .got.plt          PROGBITS         00000000006b2000  000b2000
       0000000000000070  0000000000000008  WA       0     0     8
  [25] .data             PROGBITS         00000000006b2080  000b2080
       0000000000001ad0  0000000000000000  WA       0     0     32
  [26] .bss              NOBITS           00000000006b3b60  000b3b50
       0000000000001898  0000000000000000  WA       0     0     32
  [27] __libc_freeres_pt NOBITS           00000000006b53f8  000b3b50
       0000000000000030  0000000000000000  WA       0     0     8
  [28] .comment          PROGBITS         0000000000000000  000b3b50
       000000000000002d  0000000000000001  MS       0     0     1
  [29] .symtab           SYMTAB           0000000000000000  000b3b80
       000000000000b178  0000000000000018          30   749     8
  [30] .strtab           STRTAB           0000000000000000  000becf8
       0000000000006800  0000000000000000           0     0     1
  [31] .shstrtab         STRTAB           0000000000000000  000c54f8
       000000000000015f  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)

$ readelf -l sleepmain.elf  

Elf file type is EXEC (Executable file)
Entry point 0x400990
There are 6 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000b1003 0x00000000000b1003  R E    0x200000
  LOAD           0x00000000000b1eb8 0x00000000006b1eb8 0x00000000006b1eb8
                 0x0000000000001c98 0x0000000000003570  RW     0x200000
  NOTE           0x0000000000000190 0x0000000000400190 0x0000000000400190
                 0x0000000000000044 0x0000000000000044  R      0x4
  TLS            0x00000000000b1eb8 0x00000000006b1eb8 0x00000000006b1eb8
                 0x0000000000000020 0x0000000000000050  R      0x8
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x00000000000b1eb8 0x00000000006b1eb8 0x00000000006b1eb8
                 0x0000000000000148 0x0000000000000148  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.ABI-tag .note.gnu.build-id .rela.plt .init .plt .text __libc_freeres_fn __libc_thread_freeres_fn .fini .rodata __libc_subfreeres __libc_IO_vtables __libc_atexit __libc_thread_subfreeres .eh_frame .gcc_except_table 
   01     .tdata .init_array .fini_array .jcr .data.rel.ro .got .got.plt .data .bss __libc_freeres_ptrs 
   02     .note.ABI-tag .note.gnu.build-id 
   03     .tdata .tbss 
   04     
   05     .tdata .init_array .fini_array .jcr .data.rel.ro .got 
```

Program Headers就是Segment，可以看见，sleepmain.elf的32个Section被分为6个Segment，相同属性的Section归类到Segment。其中前两个Segment会被加载进内存

```shell
$ ./sleepmain.elf &
[1] 12428

$ cat /proc/12428/maps 
00400000-004b2000 r-xp 00000000 fe:01 1058770                            sleepmain.elf
006b1000-006b4000 rw-p 000b1000 fe:01 1058770                            sleepmain.elf
006b4000-006b6000 rw-p 00000000 00:00 0 
016cc000-016ef000 rw-p 00000000 00:00 0                                  [heap]
7ffcb9d7e000-7ffcb9d9f000 rw-p 00000000 00:00 0                          [stack]
7ffcb9df7000-7ffcb9df9000 r--p 00000000 00:00 0                          [vvar]
7ffcb9df9000-7ffcb9dfb000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```
00400000-004b2000跟006b1000-006b4000就是上文加载的两片Segment加载的位置，但是比较奇怪的是006b1000-006b4000只有0x3000的大小，不能满足上文第二个Segment的0x3570大小，看书里意思，是.bss跟__libc_freeres_ptrs在最后一个页的内容扔到堆段中。


## 动态链接

### 为何需要动态链接

按照静态链接，所有程序段都会链接进可执行文件，然后在运行时加载到内存。但是这样存在一个问题，一些函数多个程序都会被用到（例如printf），如果每一个程序都链接一份进去然后加载到内存，会浪费空间。

同时，由于静态链接发生在编译期，如果一些公共函数被修改，那就需要重新链接一遍可执行文件然后发布。

为了解决这个问题，会把程序分割成各个模块，形成独立文件。然后在加载的时候再将文件链接起来，同时不同进程也可以共享这些文件在内存中可以共享的部分（代码段）。这就是动态链接。

测试代码：

```cpp
// File Name: Program1.c
// Created Time: Mon 18 Feb 2019 09:33:09 PM CST

#include "Lib.h"

int main()
{
    foobar(1);
    return 0;
}

// File Name: Program1.c
// Created Time: Mon 18 Feb 2019 09:33:09 PM CST

#include "Lib.h"

int main()
{
    foobar(2);
    return 0;
}

// File Name: Lib.c
// Created Time: Mon 18 Feb 2019 09:34:08 PM CST

#include<stdio.h>

void foobar(int i)
{
    printf("Printing from Lib.so %d\n", i);
    sleep(-1);
}

// File Name: Lib.h
// Created Time: Mon 18 Feb 2019 09:35:08 PM CST
#ifdef LIB_H
#define LIB_H
void foobar(int i);
#endif
```

```shell
$ gcc -fPIC -shared -o Lib.so Lib.c
$ gcc -o Program1 Program1.c ./Lib.so
$ gcc -o Program2 Program2.c ./Lib.so
```
这样Program1跟Program2里使用的foobar就都是从Lib.so里动态链接的了。


```shell
$ ./Program1 &
[1] 16413

$ cat /proc/16413/maps 
560937d26000-560937d27000 r-xp 00000000 fe:01 1058618                    Program1
560937f26000-560937f27000 r--p 00000000 fe:01 1058618                    Program1
560937f27000-560937f28000 rw-p 00001000 fe:01 1058618                    Program1
560938b2e000-560938b4f000 rw-p 00000000 00:00 0                          [heap]
7f10edd99000-7f10edf2e000 r-xp 00000000 fe:01 329991                     /lib/x86_64-linux-gnu/libc-2.24.so
7f10edf2e000-7f10ee12e000 ---p 00195000 fe:01 329991                     /lib/x86_64-linux-gnu/libc-2.24.so
7f10ee12e000-7f10ee132000 r--p 00195000 fe:01 329991                     /lib/x86_64-linux-gnu/libc-2.24.so
7f10ee132000-7f10ee134000 rw-p 00199000 fe:01 329991                     /lib/x86_64-linux-gnu/libc-2.24.so
7f10ee134000-7f10ee138000 rw-p 00000000 00:00 0 
7f10ee138000-7f10ee139000 r-xp 00000000 fe:01 1058614                    Lib.so
7f10ee139000-7f10ee338000 ---p 00001000 fe:01 1058614                    Lib.so
7f10ee338000-7f10ee339000 r--p 00000000 fe:01 1058614                    Lib.so
7f10ee339000-7f10ee33a000 rw-p 00001000 fe:01 1058614                    Lib.so
7f10ee33a000-7f10ee35d000 r-xp 00000000 fe:01 329987                     /lib/x86_64-linux-gnu/ld-2.24.so
7f10ee550000-7f10ee552000 rw-p 00000000 00:00 0 
7f10ee55a000-7f10ee55d000 rw-p 00000000 00:00 0 
7f10ee55d000-7f10ee55e000 r--p 00023000 fe:01 329987                     /lib/x86_64-linux-gnu/ld-2.24.so
7f10ee55e000-7f10ee55f000 rw-p 00024000 fe:01 329987                     /lib/x86_64-linux-gnu/ld-2.24.so
7f10ee55f000-7f10ee560000 rw-p 00000000 00:00 0 
7ffdd2ce7000-7ffdd2d08000 rw-p 00000000 00:00 0                          [stack]
7ffdd2d74000-7ffdd2d76000 r--p 00000000 00:00 0                          [vvar]
7ffdd2d76000-7ffdd2d78000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

可以看到，虚拟地址空间里不仅有Lib.so跟Program1，还有libc跟ld，前者是运行库，后者是动态链接器。

对于.so，也可以用readelf -l查看。

```shell
$ readelf -l Lib.so 

Elf file type is DYN (Shared object file)
Entry point 0x5c0
There are 7 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000000007bc 0x00000000000007bc  R E    0x200000
  LOAD           0x0000000000000e00 0x0000000000200e00 0x0000000000200e00
                 0x0000000000000230 0x0000000000000238  RW     0x200000
  DYNAMIC        0x0000000000000e18 0x0000000000200e18 0x0000000000200e18
                 0x00000000000001c0 0x00000000000001c0  RW     0x8
  NOTE           0x00000000000001c8 0x00000000000001c8 0x00000000000001c8
                 0x0000000000000024 0x0000000000000024  R      0x4
  GNU_EH_FRAME   0x0000000000000718 0x0000000000000718 0x0000000000000718
                 0x0000000000000024 0x0000000000000024  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000000e00 0x0000000000200e00 0x0000000000200e00
                 0x0000000000000200 0x0000000000000200  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame 
   01     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
   02     .dynamic 
   03     .note.gnu.build-id 
   04     .eh_frame_hdr 
   05     
   06     .init_array .fini_array .jcr .dynamic .got
```

可见文件里的VirtAddr0x0000000000000000跟实际虚拟地址空间0x7f10ee138000不一样，这是由于.so的最终地址编译时是不确定的，装载时才会确定。


### 对地址的处理

由于希望动态链接库在内存中尽量节约空间，那么代码段就最好只存在一份，所有进程共享（数据部分放弃治疗了）。这样就不适合用静态链接的重定位方式来解决动态链接库的地址定向问题了（不是说完全不能用，实际上有时候还是会用动态重定位）。如果用重定位，那不同进程之间重定位的地址很可能不一样（要一样的话又有无数问题需要解决，例如前一个进程给glibc重定位到0x1000，第二个进程自己却打算用0x1000，冲突就来了）。

一般情况下是用地址无关代码（-fPIC）来解决问题，也就是动态链接库里的地址都是相对偏移位置，而不是绝对位置，这样无论加载到什么位置，代码如常运行。具体来说，要解决下列情况：

- 模块内函数调用、跳转
- 模块内的数据访问
- 模块外的函数调用、跳转
- 模块外的数据访问

对于模块内的就直接用相对便宜位置访问即可，对于模块外的，ELF会建立一个指向这些变量的指针数组（GOT），然后访问时通过GOT里的相应项简介访问。

---      | 指令调用、跳转 | 数据访问
---      |       ---      |  ---
模块内部 | 相对调用和跳转 | 相对地址访问
模块外部 | 间接调用和跳转 | 间接访问


### 延迟绑定

由于不是所有函数在进程的运行期都一定会用到，所以ELF使用延迟绑定的方法来减少不必要的函数绑定，降低开销。具体巧妙的实现在书里有阐述。


### 文件结构分析

#### .interp

ELF文件里的.interp记录了动态链接器的路径

```shell
$ readelf -l Program1

  INTERP         0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

#### .dynamic
而动态链接器动态链接的时候需要的信息一般记录在.dynamic。

```shell
$ readelf -d Program1

Dynamic section at offset 0xde0 contains 27 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [./Lib.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x5f8
 0x000000000000000d (FINI)               0x804
 0x0000000000000019 (INIT_ARRAY)         0x200dc8
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x200dd0
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x298
 0x0000000000000005 (STRTAB)             0x408
 0x0000000000000006 (SYMTAB)             0x2d0
 0x000000000000000a (STRSZ)              197 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x201000
 0x0000000000000002 (PLTRELSZ)           24 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x5e0
 0x0000000000000007 (RELA)               0x508
 0x0000000000000008 (RELASZ)             216 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffb (FLAGS_1)            Flags: PIE
 0x000000006ffffffe (VERNEED)            0x4e8
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x4ce
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0
```
例如依赖的Lib.so跟libc.so就在其中。

#### 动态符号表
ELF用动态符号表（.dynsym）来记录动态链接的导入导出关系，并且为了加快程序运行时的查找符号，还用了一个符号哈希表（.gnu.hash）来辅助查询

```shell
$ readelf -s Lib.so  

Symbol table '.dynsym' contains 14 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sleep@GLIBC_2.2.5 (2)
     7: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
     8: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   23 _edata
     9: 0000000000201038     0 NOTYPE  GLOBAL DEFAULT   24 _end
    10: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
    11: 0000000000000568     0 FUNC    GLOBAL DEFAULT    9 _init
    12: 00000000000006f4     0 FUNC    GLOBAL DEFAULT   13 _fini
    13: 00000000000006c0    51 FUNC    GLOBAL DEFAULT   12 foobar
    
$ readelf -sD Lib.so 

Symbol table of `.gnu.hash' for image:
  Num Buc:    Value          Size   Type   Bind Vis      Ndx Name
    8   0: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT  23 _edata
    9   0: 0000000000201038     0 NOTYPE  GLOBAL DEFAULT  24 _end
   10   1: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT  24 __bss_start
   11   1: 0000000000000568     0 FUNC    GLOBAL DEFAULT   9 _init
   12   2: 00000000000006f4     0 FUNC    GLOBAL DEFAULT  13 _fini
   13   2: 00000000000006c0    51 FUNC    GLOBAL DEFAULT  12 foobar
```

## 从.c到.o再到.a&.so
还是上文的Lib.c

```shell
gcc -c Lib.c
ar rc Lib.a Lib.o
gcc Lib.o -fPIC -shared -o Lib.so
```
### .a跟.o
用一段python代码来对.a跟.o的文件进行对比
```python
#!/usr/bin/env python
# Created Time: Wed 20 Feb 2019 09:42:01 AM CST

def readFile(fname):
    f = open(fname, 'r+')
    s = f.readlines()
    f.close()
    return s

s1 = readFile('Lib.o')
s2 = readFile('Lib.a')
print s1 == s2[len(s2) - len(s1):]
```
输出结果是True

```shell
$ python difflast.py 
True
```
猜想：.a跟o的文件结构应该部分类似，所以单个.o静态链接成.a时不会做出修改（各种表/段也不会进行合并跟扩充，所以才能字符串相等）。

### .so跟.o
下面对比.so跟.o：

#### ELF Header
```shell
$ readelf -h Lib.o   
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
  Start of section headers:          816 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         13
  Section header string table index: 12
```
```shell
$ readelf -h Lib.so 
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x5c0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6304 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28
```
可以看到
- Type不一样（废话）
- Entry point address有变化，.o是0而.so是0x5C0，因为.o里的offset需要在链接时才真实计算
- program headers相关的项在.o里也不存在，只有在.so里存在
- .so的section headers也远远多于.o的

#### 段表

```shell
$ readelf -S Lib.o   
There are 13 section headers, starting at offset 0x330:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000033  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000268
       0000000000000048  0000000000000018   I      10     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000073
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .bss              NOBITS           0000000000000000  00000073
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .rodata           PROGBITS         0000000000000000  00000073
       0000000000000019  0000000000000000   A       0     0     1
  [ 6] .comment          PROGBITS         0000000000000000  0000008c
       000000000000002e  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000ba
       0000000000000000  0000000000000000           0     0     1
  [ 8] .eh_frame         PROGBITS         0000000000000000  000000c0
       0000000000000038  0000000000000000   A       0     0     8
  [ 9] .rela.eh_frame    RELA             0000000000000000  000002b0
       0000000000000018  0000000000000018   I      10     8     8
  [10] .symtab           SYMTAB           0000000000000000  000000f8
       0000000000000138  0000000000000018          11     9     8
  [11] .strtab           STRTAB           0000000000000000  00000230
       0000000000000031  0000000000000000           0     0     1
  [12] .shstrtab         STRTAB           0000000000000000  000002c8
       0000000000000061  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
  
$ readelf -S Lib.so 
There are 29 section headers, starting at offset 0x18a0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.gnu.build-i NOTE             00000000000001c8  000001c8
       0000000000000024  0000000000000000   A       0     0     4
  [ 2] .gnu.hash         GNU_HASH         00000000000001f0  000001f0
       000000000000003c  0000000000000000   A       3     0     8
  [ 3] .dynsym           DYNSYM           0000000000000230  00000230
       0000000000000150  0000000000000018   A       4     1     8
  [ 4] .dynstr           STRTAB           0000000000000380  00000380
       00000000000000b7  0000000000000000   A       0     0     1
  [ 5] .gnu.version      VERSYM           0000000000000438  00000438
       000000000000001c  0000000000000002   A       3     0     2
  [ 6] .gnu.version_r    VERNEED          0000000000000458  00000458
       0000000000000020  0000000000000000   A       4     1     8
  [ 7] .rela.dyn         RELA             0000000000000478  00000478
       00000000000000c0  0000000000000018   A       3     0     8
  [ 8] .rela.plt         RELA             0000000000000538  00000538
       0000000000000030  0000000000000018  AI       3    22     8
  [ 9] .init             PROGBITS         0000000000000568  00000568
       0000000000000017  0000000000000000  AX       0     0     4
  [10] .plt              PROGBITS         0000000000000580  00000580
       0000000000000030  0000000000000010  AX       0     0     16
  [11] .plt.got          PROGBITS         00000000000005b0  000005b0
       0000000000000008  0000000000000000  AX       0     0     8
  [12] .text             PROGBITS         00000000000005c0  000005c0
       0000000000000133  0000000000000000  AX       0     0     16
  [13] .fini             PROGBITS         00000000000006f4  000006f4
       0000000000000009  0000000000000000  AX       0     0     4
  [14] .rodata           PROGBITS         00000000000006fd  000006fd
       0000000000000019  0000000000000000   A       0     0     1
  [15] .eh_frame_hdr     PROGBITS         0000000000000718  00000718
       0000000000000024  0000000000000000   A       0     0     4
  [16] .eh_frame         PROGBITS         0000000000000740  00000740
       000000000000007c  0000000000000000   A       0     0     8
  [17] .init_array       INIT_ARRAY       0000000000200e00  00000e00
       0000000000000008  0000000000000008  WA       0     0     8
  [18] .fini_array       FINI_ARRAY       0000000000200e08  00000e08
       0000000000000008  0000000000000008  WA       0     0     8
  [19] .jcr              PROGBITS         0000000000200e10  00000e10
       0000000000000008  0000000000000000  WA       0     0     8
  [20] .dynamic          DYNAMIC          0000000000200e18  00000e18
       00000000000001c0  0000000000000010  WA       4     0     8
  [21] .got              PROGBITS         0000000000200fd8  00000fd8
       0000000000000028  0000000000000008  WA       0     0     8
  [22] .got.plt          PROGBITS         0000000000201000  00001000
       0000000000000028  0000000000000008  WA       0     0     8
  [23] .data             PROGBITS         0000000000201028  00001028
       0000000000000008  0000000000000000  WA       0     0     8
  [24] .bss              NOBITS           0000000000201030  00001030
       0000000000000008  0000000000000000  WA       0     0     1
  [25] .comment          PROGBITS         0000000000000000  00001030
       000000000000002d  0000000000000001  MS       0     0     1
  [26] .symtab           SYMTAB           0000000000000000  00001060
       0000000000000570  0000000000000018          27    45     8
  [27] .strtab           STRTAB           0000000000000000  000015d0
       00000000000001d7  0000000000000000           0     0     1
  [28] .shstrtab         STRTAB           0000000000000000  000017a7
       00000000000000f6  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
  
$ readelf -l Lib.o

There are no program headers in this file.
$ readelf -l Lib.so

Elf file type is DYN (Shared object file)
Entry point 0x5c0
There are 7 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000000007bc 0x00000000000007bc  R E    0x200000
  LOAD           0x0000000000000e00 0x0000000000200e00 0x0000000000200e00
                 0x0000000000000230 0x0000000000000238  RW     0x200000
  DYNAMIC        0x0000000000000e18 0x0000000000200e18 0x0000000000200e18
                 0x00000000000001c0 0x00000000000001c0  RW     0x8
  NOTE           0x00000000000001c8 0x00000000000001c8 0x00000000000001c8
                 0x0000000000000024 0x0000000000000024  R      0x4
  GNU_EH_FRAME   0x0000000000000718 0x0000000000000718 0x0000000000000718
                 0x0000000000000024 0x0000000000000024  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000000e00 0x0000000000200e00 0x0000000000200e00
                 0x0000000000000200 0x0000000000000200  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame 
   01     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
   02     .dynamic 
   03     .note.gnu.build-id 
   04     .eh_frame_hdr 
   05     
   06     .init_array .fini_array .jcr .dynamic .got
```
.so的段表确实多了很多项，而且也有了真实的offset
.o没有Segment但是.so有

#### 重定位表
```shell
$ objdump -r Lib.o

Lib.o:     file format elf64-x86-64

RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE 
0000000000000013 R_X86_64_PC32     .rodata-0x0000000000000004
000000000000001d R_X86_64_PLT32    printf-0x0000000000000004
000000000000002c R_X86_64_PLT32    sleep-0x0000000000000004


RELOCATION RECORDS FOR [.eh_frame]:
OFFSET           TYPE              VALUE 
0000000000000020 R_X86_64_PC32     .text


$ objdump -r Lib.so

Lib.so:     file format elf64-x86-64
```
.so没需要重定位的。

#### 符号表
```
$ readelf -s Lib.o

Symbol table '.symtab' contains 13 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS Lib.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     9: 0000000000000000    51 FUNC    GLOBAL DEFAULT    1 foobar
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sleep
$ readelf -s Lib.so

Symbol table '.dynsym' contains 14 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sleep@GLIBC_2.2.5 (2)
     7: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
     8: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   23 _edata
     9: 0000000000201038     0 NOTYPE  GLOBAL DEFAULT   24 _end
    10: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
    11: 0000000000000568     0 FUNC    GLOBAL DEFAULT    9 _init
    12: 00000000000006f4     0 FUNC    GLOBAL DEFAULT   13 _fini
    13: 00000000000006c0    51 FUNC    GLOBAL DEFAULT   12 foobar

Symbol table '.symtab' contains 58 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000000001c8     0 SECTION LOCAL  DEFAULT    1 
     2: 00000000000001f0     0 SECTION LOCAL  DEFAULT    2 
     3: 0000000000000230     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000380     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000438     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000458     0 SECTION LOCAL  DEFAULT    6 
     7: 0000000000000478     0 SECTION LOCAL  DEFAULT    7 
     8: 0000000000000538     0 SECTION LOCAL  DEFAULT    8 
     9: 0000000000000568     0 SECTION LOCAL  DEFAULT    9 
    10: 0000000000000580     0 SECTION LOCAL  DEFAULT   10 
    11: 00000000000005b0     0 SECTION LOCAL  DEFAULT   11 
    12: 00000000000005c0     0 SECTION LOCAL  DEFAULT   12 
    13: 00000000000006f4     0 SECTION LOCAL  DEFAULT   13 
    14: 00000000000006fd     0 SECTION LOCAL  DEFAULT   14 
    15: 0000000000000718     0 SECTION LOCAL  DEFAULT   15 
    16: 0000000000000740     0 SECTION LOCAL  DEFAULT   16 
    17: 0000000000200e00     0 SECTION LOCAL  DEFAULT   17 
    18: 0000000000200e08     0 SECTION LOCAL  DEFAULT   18 
    19: 0000000000200e10     0 SECTION LOCAL  DEFAULT   19 
    20: 0000000000200e18     0 SECTION LOCAL  DEFAULT   20 
    21: 0000000000200fd8     0 SECTION LOCAL  DEFAULT   21 
    22: 0000000000201000     0 SECTION LOCAL  DEFAULT   22 
    23: 0000000000201028     0 SECTION LOCAL  DEFAULT   23 
    24: 0000000000201030     0 SECTION LOCAL  DEFAULT   24 
    25: 0000000000000000     0 SECTION LOCAL  DEFAULT   25 
    26: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    27: 0000000000200e10     0 OBJECT  LOCAL  DEFAULT   19 __JCR_LIST__
    28: 00000000000005c0     0 FUNC    LOCAL  DEFAULT   12 deregister_tm_clones
    29: 0000000000000600     0 FUNC    LOCAL  DEFAULT   12 register_tm_clones
    30: 0000000000000650     0 FUNC    LOCAL  DEFAULT   12 __do_global_dtors_aux
    31: 0000000000201030     1 OBJECT  LOCAL  DEFAULT   24 completed.6972
    32: 0000000000200e08     0 OBJECT  LOCAL  DEFAULT   18 __do_global_dtors_aux_fin
    33: 0000000000000690     0 FUNC    LOCAL  DEFAULT   12 frame_dummy
    34: 0000000000200e00     0 OBJECT  LOCAL  DEFAULT   17 __frame_dummy_init_array_
    35: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS Lib.c
    36: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    37: 00000000000007b8     0 OBJECT  LOCAL  DEFAULT   16 __FRAME_END__
    38: 0000000000200e10     0 OBJECT  LOCAL  DEFAULT   19 __JCR_END__
    39: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 
    40: 0000000000201028     0 OBJECT  LOCAL  DEFAULT   23 __dso_handle
    41: 0000000000200e18     0 OBJECT  LOCAL  DEFAULT   20 _DYNAMIC
    42: 0000000000000718     0 NOTYPE  LOCAL  DEFAULT   15 __GNU_EH_FRAME_HDR
    43: 0000000000201030     0 OBJECT  LOCAL  DEFAULT   23 __TMC_END__
    44: 0000000000201000     0 OBJECT  LOCAL  DEFAULT   22 _GLOBAL_OFFSET_TABLE_
    45: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
    46: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   23 _edata
    47: 00000000000006f4     0 FUNC    GLOBAL DEFAULT   13 _fini
    48: 00000000000006c0    51 FUNC    GLOBAL DEFAULT   12 foobar
    49: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@@GLIBC_2.2.5
    50: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    51: 0000000000201038     0 NOTYPE  GLOBAL DEFAULT   24 _end
    52: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
    53: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
    54: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
    55: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sleep@@GLIBC_2.2.5
    56: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@@GLIBC_2.2
    57: 0000000000000568     0 FUNC    GLOBAL DEFAULT    9 _init
```
不仅.so的符号表多了许多内容，也多了.dynsym这个内容，由于生成.so的时候已经确认所有符号的情况（要不然不能load），所以是知道.o里原来需要重定位的printf跟sleep是在另一个动态链接库glibc里，因此符号名也与原来的不一样了。

#### 小结

所以一个.o在被链接成.so的时候，首先是会类似于普通可执行文件，将不同的Section按照属性的不同组成不同的Segment，这是因为.so会需要独立加载而.o不需要。同时.o里标记需要重定位的符号，在.so里都被扔到了动态符号表留待导入。
