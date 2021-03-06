本文对于 ELF 的结构的解释知识更偏于实践, 需要对 ELF 的结构知识有一定的了解.


#### 参考资料

```
<程序员自我修养>
`refs/elf/*`
```

## ELF 解析

在解释ELF结构时以下面源码为例子:

```
#include <stdio.h>

int helloWorld(){
    printf("HelloWorld, %d\n", 1);
    return 0;
}

int main(){
    helloWorld();
    printf("HelloWorld, %d\n", 1);
    return 0;
}
```

查看ELF的 `section`

```
➜  elf readelf -S test
There are 30 section headers, starting at offset 0x1154:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        08048154 000154 000013 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            08048168 000168 000020 00   A  0   0  4
  [ 3] .note.gnu.build-i NOTE            08048188 000188 000024 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        080481ac 0001ac 000020 04   A  5   0  4
  [ 5] .dynsym           DYNSYM          080481cc 0001cc 000050 10   A  6   1  4
  [ 6] .dynstr           STRTAB          0804821c 00021c 00004c 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          08048268 000268 00000a 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         08048274 000274 000020 00   A  6   1  4
  [ 9] .rel.dyn          REL             08048294 000294 000008 08   A  5   0  4
  [10] .rel.plt          REL             0804829c 00029c 000018 08   A  5  12  4
  [11] .init             PROGBITS        080482b4 0002b4 000023 00  AX  0   0  4
  [12] .plt              PROGBITS        080482e0 0002e0 000040 04  AX  0   0 16
  [13] .text             PROGBITS        08048320 000320 0001a2 00  AX  0   0 16
  [14] .fini             PROGBITS        080484c4 0004c4 000014 00  AX  0   0  4
  [15] .rodata           PROGBITS        080484d8 0004d8 000018 00   A  0   0  4
  [16] .eh_frame_hdr     PROGBITS        080484f0 0004f0 000034 00   A  0   0  4
  [17] .eh_frame         PROGBITS        08048524 000524 0000d0 00   A  0   0  4
  [18] .init_array       INIT_ARRAY      08049f08 000f08 000004 00  WA  0   0  4
  [19] .fini_array       FINI_ARRAY      08049f0c 000f0c 000004 00  WA  0   0  4
  [20] .jcr              PROGBITS        08049f10 000f10 000004 00  WA  0   0  4
  [21] .dynamic          DYNAMIC         08049f14 000f14 0000e8 08  WA  6   0  4
  [22] .got              PROGBITS        08049ffc 000ffc 000004 04  WA  0   0  4
  [23] .got.plt          PROGBITS        0804a000 001000 000018 04  WA  0   0  4
  [24] .data             PROGBITS        0804a018 001018 000008 00  WA  0   0  4
  [25] .bss              NOBITS          0804a020 001020 000004 00  WA  0   0  1
  [26] .comment          PROGBITS        00000000 001020 00002b 01  MS  0   0  1
  [27] .shstrtab         STRTAB          00000000 00104b 000106 00      0   0  1
  [28] .symtab           SYMTAB          00000000 001604 000440 10     29  45  4
  [29] .strtab           STRTAB          00000000 001a44 00025c 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

下面对几个 `section` 具体解释. 这里有几点需要注意的是, 这里大部分讨论的是动态链接相关的 `section`, 很多是采用 `gdb` 查看相关内存, 这属于已经将 `ELF` 文件加载进入内存, 当加载 `ELF` 进入内存时, 属于以 `Segment` 的形式进行加载, 部分 `section` 并没有被加载. 具体文件内偏移与内存虚拟地址的对应可以参考, `readelf -l hello` 结果.

```
➜  elf readelf -l hello

Elf file type is EXEC (Executable file)
Entry point 0x8048350
There are 9 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x08048034 0x08048034 0x00120 0x00120 R E 0x4
  INTERP         0x000154 0x08048154 0x08048154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.2]
  LOAD           0x000000 0x08048000 0x08048000 0x0062c 0x0062c R E 0x1000
  LOAD           0x000f08 0x08049f08 0x08049f08 0x0011c 0x00120 RW  0x1000
  DYNAMIC        0x000f14 0x08049f14 0x08049f14 0x000e8 0x000e8 RW  0x4
  NOTE           0x000168 0x08048168 0x08048168 0x00044 0x00044 R   0x4
  GNU_EH_FRAME   0x000554 0x08048554 0x08048554 0x0002c 0x0002c R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
  GNU_RELRO      0x000f08 0x08049f08 0x08049f08 0x000f8 0x000f8 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag .note.gnu.build-id
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .jcr .dynamic .got
```

## ELF Section
#### `.got.plt` 表
> `ELF` 将 `GOT` 拆分为两个表, `.got` 和 `.got.plt`. 其中 `.got` 用来保存全局变量引用的地址, `.got.plt` 用来保存函数引用的地址, 对于外部函数的引用全部放在 `.got.plt` 中. `<程序员的自我修养> P201`

**结构**

前三项是固定的, 第一项是 `.dynamic` 的地址, 第二项是 `link_map` 的地址, 第三项是 `_dl_runtime_resolve()` 的地址, 之后就是每个外部函数的引用的数据结构 `Elf32_Rel`, 下面是对应 `eglibc-2.19/elf/elf.h` 的具体结构.

```
typedef struct
{
  Elf32_Addr    r_offset;       /* Address */
  Elf32_Word    r_info;         /* Relocation type and symbol index */
} Elf32_Rel
```

这里额外提一点, `link_map` 是一个很重要的数据结构, 记录了所有加载的动态链接库.

**未执行前`.got.plt`结构**

`.got.plt` 中对应的前四项

```
#对应.got.plt
gdb-peda$ x/4w 0x0804a000
0x804a000:  0x08049f14  0x00000000  0x00000000  0x080482f6
```

`.plt(延迟绑定)` 中对应的实现

```
gdb-peda$ x/4i 0x80482f0 #对应.plt
   0x80482f0 <printf@plt>:    jmp    DWORD PTR ds:0x804a00c
   0x80482f6 <printf@plt+6>:  push   0x0
   0x80482fb <printf@plt+11>: jmp    0x80482e0
```

可以看到 `.dynamic` 对应的地址是 `0x08049f14`, 后两项由于未初始化为空, `0x080482f6` 为第一个外部函数的引用地址, 此时没有初始化, 会跳转到 `.plt` 表进行初始化, 初始化后会修改 `0x080482f6` 为 `printf` 的地址. (至于为什么是 `0x080482f6`, 而不是 `0x80482f0`, 因为没有初始化, 需要从 `0x80482f6` 地址开始进行初始化, 当初始化完毕后才会从 `0x80482f0` 直接跳转到 `printf` 地址继续执行)


**初始化完毕 `.got.plt` 结构**

`.got.plt` 中对应的前四项

```
#.got.plt
gdb-peda$ x/4w 0x0804a000
0x804a000:  0x08049f14  0xb7fff938  0xb7ff24b0  0xb7e71410
```

此时 `.got.plt` 中的 `printf@plt` 已经被修改为查找到的 `printf` 内存地址.

#### `.plt` 延迟绑定表(`<程序员自我修养> P201`)

```
gdb-peda$ disassemble helloWorld
Dump of assembler code for function helloWorld:
   0x0804841d <+0>:     push   ebp
   0x0804841e <+1>:     mov    ebp,esp
   0x08048420 <+3>:     sub    esp,0x18
   0x08048423 <+6>:     mov    DWORD PTR [esp+0x4],0x1
   0x0804842b <+14>:    mov    DWORD PTR [esp],0x80484e0
   0x08048432 <+21>:    call   0x80482f0 <printf@plt>
   0x08048437 <+26>:    mov    eax,0x0
   0x0804843c <+31>:    leave
   0x0804843d <+32>:    ret
End of assembler dump.
```

负责延迟加载, 当第一次调用时, 会用 `_dl_runtime_resolve()` 去初始化 `.got.plt` 中的函数引用地址.

这里以 `printf@plt` 对应的实现为例子.
```
gdb-peda$ x/4i 0x80482f0
   0x80482f0 <printf@plt>:  jmp    DWORD PTR ds:0x804a00c
   0x80482f6 <printf@plt+6>:    push   0x0
   0x80482fb <printf@plt+11>:   jmp    0x80482e0
```

#### `.dynsym` 和 `.dynstr` (`.symtab` 和 `.strtab`)

`.dynsym` 中保存的是 `Elf32_Sym` 结构, 下面对应的是 `eglibc-2.19/elf/elf.h` 具体结构.

```
typedef struct
{
  Elf32_Word    st_name;        /* Symbol name (string tbl index) */
  Elf32_Addr    st_value;       /* Symbol value */
  Elf32_Word    st_size;        /* Symbol size */
  unsigned char st_info;        /* Symbol type and binding */
  unsigned char st_other;       /* Symbol visibility */
  Elf32_Section st_shndx;       /* Section index */
} Elf32_Sym
```

`.dynstr` 保存的字符串, 具体存放格式 `<Tool Interface Standard (TIS) Executable and Linking Format (ELF)> 中 Book I: String Table` 或者 `<程序员自我修养> P80`

这里举一个例子实践, 通过 `readelf --dyn-syms test` 的结果与内存中 `Elf32_Sym` 对比观察 `_IO_stdin_used`.

```
#下面需要用到这些计算结果
#sizeof(Elf32_Sym) == 0x10
#.dynstr == 0x00021c
#.dynsym == 0x0001cc
➜  elf python -c 'print int(5 * 0x10)';python -c 'print hex(0x00021c+0xb)'
80
0x227
➜  elf readelf --dyn-syms test

Symbol table '.dynsym' contains 5 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.0 (2)
     2: 00000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     3: 00000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.0 (2)
     4: 080484fc     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
#从 0x0000020c 开始对应的是 _IO_stdin_used 的 Elf32_Sym.
➜  elf hexdump -s 0x0001cc -n 80 -C test
000001cc  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000001dc  1a 00 00 00 00 00 00 00  00 00 00 00 12 00 00 00  |................|
000001ec  33 00 00 00 00 00 00 00  00 00 00 00 20 00 00 00  |3........... ...|
000001fc  21 00 00 00 00 00 00 00  00 00 00 00 12 00 00 00  |!...............|
0000020c  0b 00 00 00 fc 84 04 08  04 00 00 00 11 00 0f 00  |................|
0000021c
#从 _IO_stdin_used 的 Elf32_Sym 结构可以发现对应的 .dynstr 的索引为 0xb.
➜  elf hexdump -s 0x227 -n 15 -C test #根据 .dynsym 中的 Elf32_Sym 结构查找到对应字符串,
00000227  5f 49 4f 5f 73 74 64 69  6e 5f 75 73 65 64 00     |_IO_stdin_used.|
00000236
```

#### `.rel.plt` 重定位表

查看需要重定位的函数引用

```
➜  elf readelf -r test

Relocation section '.rel.dyn' at offset 0x294 contains 1 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049ffc  00000206 R_386_GLOB_DAT    00000000   __gmon_start__

Relocation section '.rel.plt' at offset 0x29c contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0804a00c  00000107 R_386_JUMP_SLOT   00000000   printf
0804a010  00000207 R_386_JUMP_SLOT   00000000   __gmon_start__
0804a014  00000307 R_386_JUMP_SLOT   00000000   __libc_start_main
```

这里通过一个例子实践. 查看 `printf` 对应的 `Elf32_Rel` 结构, `0x804a00c` 重定位的入口偏移(未初始化前存放的是对应 `.plt` 的初始化代码, 初始化后存放 `printf` 内存地址), `0x107` 低8位表示重定位入口类型为 `R_386_JUMP_SLOT(7)`, 高24位表示在 `.dynsym` 中的下标, 这里为 `0x10`.

```
#.rel.plt == 0x0804829c
gdb-peda$ x/6w 0x0804829c
0x804829c:  0x804a00c   0x107   0x804a010   0x207
0x80482ac:  0x804a014   0x307
gdb-peda$ x/w 0x804a00c
0x804a00c <printf@got.plt>: 0x80482f6
➜  elf readelf -r test

Relocation section '.rel.dyn' at offset 0x294 contains 1 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049ffc  00000206 R_386_GLOB_DAT    00000000   __gmon_start__

Relocation section '.rel.plt' at offset 0x29c contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0804a00c  00000107 R_386_JUMP_SLOT   00000000   printf
0804a010  00000207 R_386_JUMP_SLOT   00000000   __gmon_start__
0804a014  00000307 R_386_JUMP_SLOT   00000000   __libc_start_main

#这里可以发现, printf@GLIBC_2.0 (2) 对应 Num 为 1, 正好对应之前的 0x10
➜  elf readelf --dyn-syms test

Symbol table '.dynsym' contains 5 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.0 (2)
     2: 00000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     3: 00000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.0 (2)
     4: 080484fc     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
```

#### `.gnu.hash` 表

参考链接:

```
#大致意思就是目前默认生成新的hash表, 也就是.gnu.hash
http://pank.org/blog/2007/10/gcc-hashstylegnu.html
#实例代码更接近实际
https://sourceware.org/ml/binutils/2006-10/msg00377.html
https://www.sourceware.org/ml/binutils/2006-06/msg00418.html
#最为详细
https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections
```

`.gnu.hash` 的作用, 主要是利用 `Bloom Filter`, 在常量时间内判断, 字符是否存在, 以及对应 `.dynsym` 的位置. 使用 `gcc -g -o hello -Wl,--hash-style=sysv(gnu) hello.c` 可以产生旧版本的 `hash` 表.

`.gnu.hash` 结构说明:

```
Header
    An array of (4) 32-bit words providing section parameters:
    nbuckets
        The number of hash buckets
    symndx
        The dynamic symbol table has dynsymcount symbols. symndx is the index of the first symbol in the dynamic symbol table that is to be accessible via the hash table. This implies that there are (dynsymcount - symndx) symbols accessible via the hash table.
        第一个需要hash的symbol
    maskwords
        The number of ELFCLASS sized words in the Bloom filter portion of the hash table section. This value must be non-zero, and must be a power of 2 as explained below.
        Note that a value of 0 could be interpreted to mean that no Bloom filter is present in the hash section. However, the GNU linkers do not do this — the GNU hash section always includes at least 1 mask word.
        Bloom filter中掩码个数, 每个掩码可以为32位大小, 或者64位大小
    shift2
        A shift count used by the Bloom filter.
        另一个hash函数
Bloom Filter(32位大小)
    GNU_HASH sections contain a Bloom filter. This filter is used to rapidly reject attempts to look up symbols that do not exist in the object. The Bloom filter words are 32-bit for ELFCLASS32 objects, and 64-bit for ELFCLASS64.
    Bloom filter中的掩码
Hash Buckets(32位大小)
    An array of nbuckets 32-bit hash buckets
    hash 桶
Hash Values(32位大小)
    An array of (dynsymcount - symndx) 32-bit hash chain values, one per symbol from the second part of the dynamic symbol table.
    hash 值, 需要计算, 下文会有具体计算方法
```
查看 `.gnu.hash` 结构, 使用 `hexdump` 或者 `gdb` 都可以查看.

```
➜  elf hexdump -s 0x0001ac -n 32 -e '4/4 "%04X " " | "' -e '16/1 "%_p" "\n"' test
0002 0004 0001 0005 | ................
20002000 0000 0004 C0E34BAD | . . .........K..

```

```
gdb-peda$ x/8w 0x080481ac
0x80481ac:      0x2     0x5     0x1     0x5
0x80481bc:      0x20002000      0x0     0x5     0xc0e34bad
```
查看动态符号表

```
➜  elf readelf --dyn-syms test

Symbol table '.dynsym' contains 6 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 FUNC    GLOBAL DEFAULT  UND malloc@GLIBC_2.0 (2)
     2: 00000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.0 (2)
     3: 00000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 00000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.0 (2)
     5: 0804851c     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
```


对应说明, `nbuckets == 2` 表明有两个 `dynsym` 需要hash, `symndx == 5` 表明第1个需要hash的`dymsym` 的 `num == 4`, `maskwords == 1` 表明有 `1` 个 `Bloom filter` 掩码, `shift2 == 5` 表明另一个 `hash` 函数为 `>>5` (查看 `Bloom filter` 算法结构定义, 这里使用了`k = 2` 个 `hash` 函数, 同时设置 `hash` 值对应位置为 `1`, 防止误判)

`20002000` 即为掩码, 初始为0, `bitmask` 怎么发生修改的? (这里其实就是 `hash` 算法中利用桶方法解决 `hash` 冲突的原理)

```
本段内容来自 https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections
The hash function used by the GNU hash has this property. This fact is leveraged to produce both hash functions required by the Bloom filter from the single hash function described above:
两个hash函数
    H1 = dl_new_hash(name);
    H2 = H1 >> shift2;
As discussed above, the link editor determines how many mask words to use (maskwords) and the amount by which the first hash result is right shifted to produce the second (shift2). The more mask words used, the larger the hash section, but the lower the rate of false positives. I was told in private email that the GNU linker primarily derives shift2 from the base 2 log of the number of symbols entered into the hash table (dynsymcount - symndx), with a minimum value of 5 for ELFCLASS32, and 6 for ELFCLASS64. These values are explicitly recorded in the hash section in order to give the link editor the flexibility to change them in the future should better heuristics emerge.
The Bloom filter mask sets one bit for each of the two hash values. Based on the Bloom filter reference, the word containing each bit, and the bit to set would be calculated as:
这是根据Bloom filter定义给出的参考, 但实际gnu用了一个此算法的修改版. 这里C是上面提到的 Bloom filter 掩码大小. 这里是32位
    N1 = ((H1 / C) % maskwords);
    N2 = ((H2 / C) % maskwords);

    B1 = H1 % C;
    B2 = H2 % C;
To populate the bits when building the filter:
    bloom[N1] |= (1 << B1);
    bloom[N2] |= (1 << B2);
and to later test the filter:
    (bloom[N1] & (1 << B1)) && (bloom[N2] & (1 << B2))

Therefore, in the GNU hash, the single mask word is actually calculated as:
因此gnu的算法描述如下, 总体类似.
    N = ((H1 / C) % maskwords);
The two bits set in the Bloom filter mask word N are:
    BITMASK = (1 << (H1 % C)) | (1 << (H2 % C));
The link-editor sets these bits as
    bloom[N] |= BITMASK;
And the test used by the runtime linker is:
    (bloom[N] & BITMASK) == BITMASK;
```

**这里进行一个实例的计算:**

使用这段代码计算 `_IO_stdin_used` 的hash值.

```
#include <stdio.h>
#include <libelf/libelf.h>

uint32_t
dl_new_hash (const char *s)
{
        uint32_t h = 5381;

        for (unsigned char c = *s; c != '\0'; c = *++s)
                h = h * 33 + c;

        return h;
}
int main(int argc, char** argv) {
   printf("%zu", dl_new_hash(argv[1]));
   return 0;
}
```

对应 `.gnu.hash` 结构

```
gdb-peda$ x/8w 0x080481ac
0x80481ac:      0x2     0x5     0x1     0x5
0x80481bc:      0x20002000      0x0     0x5     0xc0e34bad
```

下面为计算过程:

先计算掩码(`bitmask`).

```
➜  elf ./a.out _IO_stdin_used
3236121517
➜  elf python -c 'print int(3236121517%32)' #Hash1算法, 对应B1
13
➜  elf python -c 'print int((3236121517>>5)%32)' #Hash2算法, 对应B2
29
所以:
    N = 0
    BITMASK = (1 << 13)) | (1 << 29)) = 0x20002000
    bloom[N] |= BITMASK
所以:
    bloom[0] = 0x20002000
```

`0x0    0x5` 是 `Hash Buckets`, 关于hash中如何利用桶解决hash冲突 `http://www.cnblogs.com/xiekeli/archive/2012/01/16/2323391.html`

```
Hash Buckets

Following the Bloom filter are nbuckets 32-bit words. Each word N in the array contains the lowest index into the dynamic symbol table for which:
    (dl_new_hash(symname) % nbuckets) == N
Since the dynamic symbol table is sorted by the same key (hash % nbuckets), dynsym[buckets[N]] is the first symbol in the hash chain that will contain the desired symbol if it exists.
A bucket element will contain the index 0 if there is no symbol in the hash table for the given value of N. As index 0 of the dynsym is a reserved value, this index cannot occur for a valid symbol, and is therefore non-ambiguous.
```
继续上面的实例计算 `Hash Buckets`:

```
➜  elf python -c 'print int(3236121517%2)'

N = 1
buckets[N] = 0x5
dynsym[0x5] = _IO_stdin_used
```

`0xc0e34bad` 是 `Hash Values`, 关于 `Hash Values` 请参考下面,

```
The final part of a GNU hash section contains (dynsymcount - symndx) 32-bit words, one entry for each symbol in the second part of the dynamic symbol table. The top 31 bits of each word contains the top 31 bits of the corresponding symbol's hash value. The least significant bit is used as a stopper bit. It is set to 1 when a symbol is the last symbol in a given hash chain:
前31位作为hash值, 最后一位作为分割, 当symbol为最后一个或者两个桶的边界, 都要将最后一位置为1
    lsb = (N == dynsymcount - 1) ||
      ((dl_new_hash (name[N]) % nbuckets)
       != (dl_new_hash (name[N + 1]) % nbuckets))

    hashval = (dl_new_hash(name) & ~1) | lsb;

或者可以参考这个算法
(dl_new_hash (&.dynstr[.dynsym[N].st_name]) & ~1)
| (N == dynsymcount - 1
   || (dl_new_hash (&.dynstr[.dynsym[N].st_name]) % nbuckets)
      != (dl_new_hash (&.dynstr[.dynsym[N + 1].st_name]) % nbuckets))
```
这段一共有 `dynsymcount - symndx` 个 `32-bit` 的 `Hash Values`, 注意, 这个 `dynsymcount` 在运行时是未知的.

最后分析一段 `GNU` 中利用 `name` 返回指向该 `symbol` 的索引, 这段算法在注入时是核心代码. 这里需要理解下 `hash` 算法中利用桶解决冲突的原理, 冲突后需要在桶内进行查找, 时间复杂度为 `O(1)+O(m)`, `m` 为桶的固定大小.

```
Symbol Lookup Using GNU Hash

The following shows how a symbol might be looked up in an object using the GNU hash section. We will assume the existence of an in memory record containing the information needed:
typedef struct {
        const char      *os_dynstr;      /* Dynamic string table *//* .dynstr地址 */
        Sym             *os_dynsym;      /* Dynamic symbol table *//* .dynsym地址 */
        Word            os_nbuckets;     /* # hash buckets */
        Word            os_symndx;       /* Index of 1st dynsym in hash */
        Word            os_maskwords_bm; /* Bloom filter words, minus 1 */
        Word            os_shift2;       /* Bloom filter hash shift */
        const BloomWord *os_bloom;       /* Bloom filter words */
        const Word      *os_buckets;     /* Hash buckets */
        const Word      *os_hashval;     /* Hash value array */
} obj_state_t;
To simplify matters, we elide the details of handling different ELF classes. In the above, Word is a 32-bit unsigned value, BloomWord is either 32 or 64-bit depending in the ELFCLASS, and Sym is either Elf32_Sym or Elf64_Sym.
Given a variable containing the above information for an object, the following pseudo code returns a pointer to the desired symbol if it exists in the object, and NULL otherwise.

Sym *
symhash(obj_state_t *os, const char *symname)
{
        Word            c;
        Word            h1, h2;
        Word            n;
        Word            bitmask;
        const Sym       *sym;
        Word            *hashval;

        /*
         * Hash the name, generate the "second" hash
         * from it for the Bloom filter.
         */
        /* 两个散列函数 */
        h1 = dl_new_hash(symname);
        h2 = h1 >> os->os_shift2;

        /* Test against the Bloom filter */
        /* BloomWord的位大小 */
        c = sizeof (BloomWord) * 8;
        n = (h1 / c) & os->os_maskwords_bm;
        bitmask = (1 << (h1 % c)) | (1 << (h2 % c));
        if ((os->os_bloom[n] & bitmask) != bitmask)
                return (NULL);

        /* Locate the hash chain, and corresponding hash value element */
        /* 获取对应桶的第一个.dynsym索引 *
        n = os->os_buckets[h1 % os->os_nbuckets];
        if (n == 0)    /* Empty hash chain, symbol not present */
                return (NULL);
        /* Sym数据结构 */
        sym = &os->os_dynsym[n];
        /* 获取对应桶的第一个hashval */
        hashval = &os->os_hashval[n - os->os_symndx];

        /*
         * Walk the chain until the symbol is found or
         * the chain is exhausted.
         */
        for (h1 &= ~1; 1; sym++) {
                /* 进行桶内遍历查询 */
                h2 = *hashval++;

                /*
                 * Compare the strings to verify match. Note that
                 * a given hash chain can contain different hash
                 * values. We'd get the right result by comparing every
                 * string, but comparing the hash values first lets us
                 * screen obvious mismatches at very low cost and avoid
                 * the relatively expensive string compare.
                 *
         * We are intentionally glossing over some things here:
             *
         *    -  We could test sym->st_name for 0, which indicates
         *   a NULL string, and avoid a strcmp() in that case.
         *
                 *    - The real runtime linker must also take symbol
         *  versioning into account. This is an orthogonal
         *  issue to hashing, and is left out of this
         *  example for simplicity.
         *
         * A real implementation might test (h1 == (h2 & ~1), and then
         * call a (possibly inline) function to validate the rest.
                 */
                /* hash值相同并且name相同 */
                if ((h1 == (h2 & ~1)) &&
                    !strcmp(symname, os->os_dynstr + sym->st_name))
                        return (sym);

                /* Done if at end of chain */
                /* 桶内查询完毕,因为桶边界是最低位为1 */
                if (h2 & 1)
                        break;
        }

        /* This object does not have the desired symbol */
        return (NULL);
}
```

差不多把几个重要的section都介绍到了


#### 调试命令相关
```
# 查看 `DYNAMIC`信息
readelf -d test

# 查看 `Section Headers`
readelf -S test
```

#### gdb 调试 glibc
```
http://hardenedlinux.org/toolchains/2016/08/25/build_debug_environment_for_dynamic_linker_of_glibc.html
http://blog.nlogn.cn/trace-glibc-by-using-gdb/
```

#### 参考链接

```
<程序员自我修养>

docs/elf/*
```