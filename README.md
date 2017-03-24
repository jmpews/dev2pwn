# pwn & exploit

这是些前段时间研究二进制的一些心得 Paper. 本来是希望能够从底层原理到全局把控的层次去整理. 这里只完成了部分的Paper, 还有很多的Paper只写了概要点.

个人有几篇 paper 还是很有参考价值的

**[linux进程动态so注入.md]()**

这篇文章介绍了如何在目前的ELF下进行动态so注入, 介绍 gnu.hash 的结构和相关算法, 具体的代码可以参考[evilELF](), 代码设计规范.

**[PWN之堆内存管理.md]()**

这篇文章是我在阅读了很多参考资料和glibc后写的, 其中对于glibc分配算法中的各种缓存的设计有比较好的讲述以及对分配和释放算法有比价好的阐述.

**PWN之栈触发.md**

这篇文章是在 **PWN之堆内存管理.md** 之后, 对其中各种关于堆触发的利用方式有比较的总结, 建议看看 **堆利用综述** 这一开头总结, 这里只介绍了部分堆漏洞的原理、利用等, 未完待续.

**PWN之ELF系列**

这是文章是在仔细研究了ELF文件结构后写的一些总结记录文章

**OSX.IOS**

这些是最近的关于macOS和IOS的研究, 有几篇文章是关于 macho 文件, 这里写了一个 dumpRuntimeMacho 可以在运行对 macho 文件进行解析.

**evilELF**

对于ELF文件的相关利用

**evilMACHO**

对于Macho文件的相关利用

**evilHEAP**

对于各种 heap 的利用方式的实例


```
.
├── OSX.IOS
│   ├── PWN之OSX.md
│   ├── PWN之macho解析.md
│   ├── PWN之macho加载过程.md
│   └── osx和ios的交叉编译.md
├── PWN之ELF解析.md
├── PWN之ELF以及so的加载和dlopen的过程.md
├── PWN之ELF符号动态解析过程.md
├── PWN之堆触发.md
├── PWN之栈触发.md
├── PWN之保护机制.md
├── PWN之逆向技巧.md
├── PWN之堆内存管理.md
├── PWN之漏洞触发点.md
├── PWN之利用方法总结.md
├── PWN之绕过保护机制.md
├── PwnableWriteup.md
├── README.md
├── assets
├── evilELF
│   ├── InjectRuntimeELF
│   │   ├── __libc_dlopen_mode.o
│   │   ├── evil.so
│   │   └── example
│   ├── LICENSE
│   ├── README.md
│   └── injectso
│       ├── __libc_dlopen_mode.asm
│       ├── __libc_dlopen_mode.o
│       ├── evil.c
│       ├── evil.so
│       ├── inject
│       ├── inject.c
│       ├── utils.c
│       └── utils.h
├── evilHEAP
│   ├── LICENSE
│   ├── README.md
│   ├── house_of_force
│   │   ├── README.md
│   │   └── vul.c
│   └── house_of_mind
│       ├── README.md
│       ├── exp.c
│       └── vul.c
├── evilMACHO
│   ├── LICENSE
│   ├── README.md
│   └── dumpRuntimeMacho
│       ├── Build
│       ├── README.md
│       ├── dumpRuntimeMacho
│       └── dumpRuntimeMacho.xcodeproj
├── linux进程动态so注入.md
├── refs
│   ├── 2002.gera_.About_Exploits_Writing.pdf
│   ├── 2015-1029-yangkun-Gold-Mining-CTF.pdf
│   ├── Linux_Interactive_Exploit_Development_with_GDB_and_PEDA_Slides.pdf
│   ├── ROP_course_lecture_jonathan_salwan_2014.pdf
│   ├── elf
│   │   ├── 001_1 程序的链接和装入及Linux下动态链接的实现(修订版).doc
│   │   ├── 001_2 Linux 动态链接机制研究及应用.pdf
│   │   ├── 001_3 elf动态解析符号过程(修订版).rtf
│   │   ├── 001_4 Linux下的动态连接库及其实现机制(修订版).rtf
│   │   ├── ELF-berlinsides-0x3.pdf
│   │   ├── Understanding_ELF.pdf
│   │   ├── bh-us-02-clowes-binaries.ppt
│   │   ├── elf.pdf
│   │   ├── 漫谈兼容内核之八 ELF 映像的装入 ( 一 ).doc
│   │   └── 《链接器和加载器》中译本.pdf
│   ├── formatstring-1.2.pdf
│   ├── gcc.pdf
│   ├── heap
│   │   ├── Bugtraq_The Malloc Maleficarum.pdf
│   │   ├── Glibc_Adventures-The_Forgotten_Chunks.pdf
│   │   ├── Heap overflow using Malloc Maleficarum _ sploitF-U-N.pdf
│   │   ├── Project Zero_ The poisoned NUL byte, 2014 edition.pdf
│   │   ├── [Phrack]Advanced Doug lea's malloc exploits.pdf
│   │   ├── [Phrack]Vudo malloc tricks.pdf
│   │   ├── bh-usa-07-ferguson-WP.pdf
│   │   ├── glibc内存管理ptmalloc源代码分析.pdf
│   │   ├── heap-hacking-by-0xbadc0de.be.pdf
│   │   └── x86 Exploitation 101_ this is the first witchy house – gb_master's _dev_null.pdf
│   ├── ltrace_internals.pdf
│   └── pwntools.pdf
└── tools
    └── Pwngdb
        ├── LICENSE
        ├── README.md
        ├── angelheap
        └── pwngdb.py

20 directories, 71 files
```