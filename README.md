# pwn & exploit

```shell
λ : tree -N -I '__*__' -L 3
.
├── PWN之ELF解析.md
├── PWN之ELF符号动态解析过程.md
├── PWN之栈触发.md
├── PWN之保护机制.md
├── PWN之逆向技巧.md
├── PWN之堆内存管理.md
├── PWN之堆内存触发.md
├── PWN之漏洞触发点.md
├── PWN之利用方法总结.md
├── PWN之绕过保护机制.md
├── PwnableWriteup.md
├── README.md
├── assets
├── evilELF
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
│   │   ├── Glibc_Adventures-The_Forgotten_Chunks.pdf
│   │   ├── Project Zero_ The poisoned NUL byte, 2014 edition.pdf
│   │   ├── [Phrack]Advanced Doug lea's malloc exploits.pdf
│   │   ├── [Phrack]Vudo malloc tricks.pdf
│   │   ├── glibc内存管理ptmalloc源代码分析.pdf
│   │   └── heap-hacking-by-0xbadc0de.be.pdf
│   ├── ltrace_internals.pdf
│   └── pwntools.pdf
└── tools
    └── Pwngdb
        ├── LICENSE
        ├── README.md
        ├── angelheap
        └── pwngdb.py

9 directories, 50 files
```