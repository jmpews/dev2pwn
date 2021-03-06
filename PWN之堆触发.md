#### 参考资料

```
#最好按照参考资料顺序查看

#malloc理解
<PWN之堆内存管理.md>

#关于堆相关利用的一些示例程序, 以及参考文档, 这个强力推荐, 建议把其中列举的所有参考文档都阅读并理解
https://github.com/shellphish/how2heap

#介绍常用漏洞, 如何由 BUG 到 Primitive, 再如何到 Exploit
2002.gera.About_Exploits_Writing.pdf

#unlink参考资料
#都是讲unlink 和 unlink 的绕过的, 但是总感觉对于 unlink 绕过保护机制那块没有讲出原理.
#因该是free@plt, 而不是 free@got, 因为 ELF 将 got 表拆分为两部分 `.got`, `.got.plt`, 详细查看 <PWN之ELF解析.md>
http://static.hx99.net/static/drops/tips-7326.html
http://www.ms509.com/?p=49
#只讲到 unlink, 没有提及绕过, 但是讲到一些检测机制.
https://jaq.alibaba.com/community/art/show?articleid=360

#感觉这作者好厉害, 有自己的一个对漏洞的全方位理解, 很透彻.
http://www.freebuf.com/articles/system/91527.html

https://googleprojectzero.blogspot.jp/2014/08/the-poisoned-nul-byte-2014-edition.html(Project Zero)
https://www.contextis.com/documents/120/Glibc_Adventures-The_Forgotten_Chunks.pdf (off-by-one 总结)
http://angelboy.logdown.com/posts/262325-plaid-ctf-2015-write-up (shrink free chunk size)
http://blog.frizn.fr/pctf-2015/pwn-550-plaiddb (shrink free chunk size)
http://netsec.ccert.edu.cn/wp-content/uploads/2015/10/2015-1029-yangkun-Gold-Mining-CTF.pdf (内存pwn总结)
```
## 0x00 堆利用综述

在这里我们仅讨论 `Turning bugs into primitives` 的过程, 关注更多的利用方法背后的原理, 而不是利用方法本身. 这里仅仅是个人看法, 个人的一些理解.

下面的很多漏洞是实现 aa4bmo 'almost arbitrary 4 bytes mirrored overwrite'  (任意 4 字节写), 所以需要一个代码段来实现 `*x=y` 操作.

要实现 `*x=y` 有几种思路, 从这个操作的发起者层面讲, 1. 用户主动发起修改操作 2. 借助 glibc 函数自带的类似操作. 

#### 对于用户主动发起操作的思考:

1. 正常的 `x` 经过绕过后, 恶意修改为关键信息的地址, 比如存在 `*t=x` 类似操作就可能导致 `x` 指向的位被修改, 典型的例子就是 unlink 的绕过利用.
2. 恶意构造和修改 `malloc_state` 结构, 导致 malloc 的返回地址 `x` 为关键信息地址, 这里需要考虑到 malloc 可以哪些缓存取到空闲 chunk, 比如: fastbins ? bins ? top chunk ?  这是对 `malloc_state` 的一种利用场景, 借助再分配可控内存, 同时还有另一个种对 `malloc_state` 的利用在下面会提到.

#### 对于借助 glibc 函数本身类似的操作的思考:

就是找到类似 `*x=y` 的操作, 并且对于 `x` 和 `y` 都需要有一定的控制权, 比如 unlink 操作. 这里需要明白 `*x=y` 要表达的意思, 可以认为, 在地址 `x` 记录 `y` 值, 思考在 glibc 中哪里需要这样的操作. 

```
#define unlink( P, BK, FD ) {
BK = P->bk;
FD = P->fd;
FD->bk = BK;
BK->fd = FD;
}
```

unlink 操作在没有加验证时时, 它的两个参数都是无限制, 可以容易实现 "任意地址写" 操作.

根据 `*x=y` 要表达的意思, 并且类比 unlink, unlink 是拆除空闲节点, 这个操作通常是发生在分配和合并 chunk 时.

分配操作发生在 `_int_malloc()`, 但是有个问题, 这时候的分配的内容使很难被控制的, 也就是很难 `craft a fake chunk`

合并操作多发在 `_int_free()` (在 `_int_malloc()` 也可能发生合并操作, 比如 `malloc_consolidate()`).

以上是类比 unlink 操作, 如果熟悉 ptmalloc 的空闲内存管理以及如何进行回收, 可以想到, 在 chunk 被释放后需要将空闲 chunk 的地址插入到 Fastbins, Unsorted bin, Bins, 所以在这个过程中就会存在 `*x=y` 的类似操作, 这里也存在一个问题, Fastbins, Unsorted bin, Bins 是分配区结构体 `malloc_state` 的元素, 也就是静态全局变量 `main_arena` 变量, 地址固定, 所以这里 `x` 被限制, 同样由于 `_int_free()` 的检查以及具体代码的实现(正常的代码逻辑肯定不是瞎释放内存啊, 所以释放的地址也几乎不会被用户控制), 因此这里的 `y` 也被限制, 所以就希望找到绕过限制的方法, 比如对于 `x` 的限制, 可以通过尝试伪造 `malloc_state`, 借助 free 过程中向 `malloc_state` 的写过程, 完成 `*x=y` 操作, 这是对 `malloc_state` 的另一种利用, 是在两个不同的角度.

所以很多时候是由于部分逻辑没有检查全, 感觉有点像 yuange 说的意思, 对于代码逻辑, 应该是什么样子,需要检查清楚, 妥协就又可能造成绕过.

## 0x00 The House of Prime

该利用已经不再适用, `malloc_state` 不在包含 `max_fast`, 以及 在 `_int_free` 的过程中检查 了 chunk 的最小 size.

#### 参考链接

```
The Malloc Maleficarum (http://seclists.org/bugtraq/2005/Oct/118)
https://gbmaster.wordpress.com/2014/08/24/x86-exploitation-101-this-is-the-first-witchy-house/
```

#### 导致(最终结果):

**伪造 malloc 返回地址, 实现 aa4bmo. (针对 Fastbins 利用)**

**借助 `_int_free()` 地址写操作, 实现 aa4bmo. (针对 Unstored bin 的利用)**

#### 原理和思考

绕过 Fastbins 分配限制, 覆盖了 `arena_key `, 导致伪造了 `arena_key `, 从而伪造了空闲 chunk 地址, 通过再分配, 就可以得到之前伪造的空闲 chunk 地址的内容控制权, 最终导致 aa4bmo

#### 利用过程:

Fastbins 的利用思路

首先对第一个  chunk 进行释放操作, 这里需要修改 chunk size, `_int_free` 释放内存时没有检测 chunk size 的最小值, 导致 `fastbin_index` 结果为 -1, `max_fast` 恰好位于 `fastbins` 相邻, 进而 ` max_fast` 被覆盖为有一个很大的值(空闲 chunk 地址), 此时对第二个 chunk 进行释放,  这里需要修改 chunk size, 以使释放后的 chunk 能恰好被放在 `arena_key` 位置, 此时会导致 arena_key 指向第二个 chunk, 由于这两个 chunk 都是可以控制的, 此时进行 malloc 操作, 我们可以通过 伪造 Fastbins 链表内容(空闲 chunk 的地址), 进而让 malloc 返回我们伪造的空闲 chunk 的地址, 以达到任意内存地址写内容

Unsorted bin 利用思路

根据上面, Fastbins 的前部分思路, `arena_key` 可以被控制, 因此我们伪造 Unstorted bin 链表内容, 在摘除 unsorted bin 节点时, 利用 glibc 函数内部操作, 达到任意地址写.

#### 触发前提(限制):

这里限制有点多了, 对于 Fastbins 利用, 需要能控制第一个 chunk 的 size, 需要能控制第二个 chunk 的内容. 对于 Unsorted bin 的利用, 也是如上, 在伪造 unsorted bin  步骤比较复杂.

#### 保护机制:

已经在 `_int_free` 限制了 chunk 的最小 size.

```
https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=bf58906631af8fe0d57625988b1d003cc09ef01d;hp=04ec80e410b4efb0576a2ffd0d2f29ed1fdac451
```

## 0x00 House of Mind

#### 参考链接

```
The Malloc Maleficarum (http://seclists.org/bugtraq/2005/Oct/118)
https://sploitfun.wordpress.com/2015/03/04/heap-overflow-using-malloc-maleficarum/
https://gbmaster.wordpress.com/2015/06/15/x86-exploitation-101-house-of-mind-undead-and-loving-it/
http://phrack.org/issues/66/10.html
```

#### 导致(最终结果):

**aa4bmo**

#### 原理(触发原因):

同一开始的 **堆利用综述** 做比较, 这里属于控制 `malloc_state` 分配区, 进而控制 `x` 但写入的 `y` 被限定为空闲 chunk 地址.

```
#define HEAP_MAX_SIZE (1024*1024) /* must be a power of two */

#define heap_for_ptr(ptr) \
 ((heap_info *)((unsigned long)(ptr) & ~(HEAP_MAX_SIZE-1)))

/* check for chunk from non-main arena */
#define chunk_non_main_arena(p) ((p)->size & NON_MAIN_ARENA)

#define arena_for_chunk(ptr) \
 (chunk_non_main_arena(ptr) ? heap_for_ptr(ptr)->ar_ptr : &main_arena)
```

chunk 所属分配区被伪造, 然后经过大量内存分配, 进而导致获取 `heap_info` 的过程被劫持, 得到的是我们伪造的 `heap_info`,进而伪造 `malloc_state`, 从而导致 `_int_free()` 传入的是伪造的 `malloc_state` , 最后利用 `_int_free()` 内的写操作, 实现任意地址写特定内容.

#### 利用思路:

修改返回地址为 chunk 地址, chunk 内容为 shellcode

#### 触发前提(限制):

很显著的一个限制就是堆数据可执行

详细的的限制条件在 `参考链接` 的文章指出, 具体的限制, 会在下面与 payload 对照指出, 这样可能更清楚.

#### 保护机制:

堆数据可执行

#### 利用实践

下面是需要构造的内存布局, 当执行 `_int_free()` 会修改 `av->fastbins[0]` 地址为 chunk 的地址, 达到修改返回地址的目的, 这个返回地址是有限制的, 只能是被释放的 chunk 的地址, 需要进一步利用.

```
STACK:   ^
         |
         |      0xRAND_VAL     av->system_mem (av + 1848)
         |         ...
         |      pushed EIP     av->fastbins[0]
         |      pushed EBP     av->max_fast
         |      0x00000000     av->mutex
         |
```

具体测试程序在 [evilHEAP](http://github.com/jmpews/evilHEAP)

## 0x00 House of Force

#### 参考资料

```
https://sploitfun.wordpress.com/2015/03/04/heap-overflow-using-malloc-maleficarum/
https://gbmaster.wordpress.com/2015/06/28/x86-exploitation-101-house-of-force-jedi-overflow/
http://seclists.org/bugtraq/2005/Oct/118
http://s0ngsari.tistory.com/entry/Heap-Exploit-House-of-Force
`refs/heap/Bugtraq_The Malloc Maleficarum.pdf` 对一些关键语句做了标记
```

#### 导致(最终结果):

控制 malloc 返回地址, 最终导致 **aa4bmo**

**原理(触发原因):**

分配很大的内存, 导致恶意修改 top chunk 地址为关键信息地址.

具体解释

当 malloc 需要分配一块很大的内存时, 需要检查 top chunk 合适.

```c
static void *
_int_malloc (mstate av, size_t bytes)
{
  INTERNAL_SIZE_T nb;               /* normalized request size */
  unsigned int idx;                 /* associated bin index */
  mbinptr bin;                      /* associated bin */

  mchunkptr victim;                 /* inspected/selected chunk */
  INTERNAL_SIZE_T size;             /* its size */
  int victim_index;                 /* its bin index */

  mchunkptr remainder;              /* remainder from a split */
  unsigned long remainder_size;     /* its size */

  unsigned int block;               /* bit map traverser */
  unsigned int bit;                 /* bit map traverser */
  unsigned int map;                 /* current word of binmap */

  mchunkptr fwd;                    /* misc temp for linking */
  mchunkptr bck;                    /* misc temp for linking */

  const char *errstr = NULL;

  /*
     Convert request size to internal form by adding SIZE_SZ bytes
     overhead plus possibly more to obtain necessary alignment and/or
     to obtain a size of at least MINSIZE, the smallest allocatable
     size. Also, checked_request2size traps (returning 0) request sizes
     that are so large that they wrap around zero when padded and
     aligned.
   */

  checked_request2size (bytes, nb);
  
  [...]
  
  malloc.c:3749  
  use_top:
      victim = av->top;
      size = chunksize (victim);

      if ((unsigned long) (size) >= (unsigned long) (nb + MINSIZE))
        {
          remainder_size = size - nb;
          remainder = chunk_at_offset (victim, nb);
          av->top = remainder;
          set_head (victim, nb | PREV_INUSE |
                    (av != &main_arena ? NON_MAIN_ARENA : 0));
          set_head (remainder, remainder_size | PREV_INUSE);

          check_malloced_chunk (av, victim, nb);
          void *p = chunk2mem (victim);
          alloc_perturb (p, bytes);
          return p;
        }
```

这里直接使用 chunk_at_offset 作为 top chunk 的地址.

```
/* Treat space at ptr + offset as a chunk */
#define chunk_at_offset(p, s)  ((mchunkptr) (((char *) (p)) + (s)))
```

所以假如可以任意指定 nb 值, 就可以 `任意` 控制 top chunk 的地址. 假如再通过 malloc 分配就可以获得需要控制的地址.

#### 触发前提(限制):

正常的逻辑应该是 当分配很大内存时在检查 top chunk 不够后, 主分配会调用 `brk()` 扩展 top chunk, 非主分配去会调用 `mmap()` 扩展 sub-heap.  这就需要控制 top chunk 的 size 假装 top chunk 所占空间很大, 以实现 memory overlap.

另一个点, 在上面的提到 `任意` 控制 top chunk 的地址, 这个任意是有限制的, `nb` 只是相对于  `victim` 的任意地址, 所以按理来说还需要 leak heap address, 测试程序中直接给出.

#### 保护机制:

如果开启 `RELRO` 无法修改 got 表.

如果没有开启堆执行保护也可以修改返回栈中 EIP 的地址, 为 shellcode 地址, 前提是需要知道栈中 EIP 的位置.

#### 利用实践

具体测试程序在 [evilHEAP](http://github.com/jmpews/evilHEAP)

## 0x00 unlink

#### 导致(最终结果):

**实现任意内存地址写内容, 具体来说是借助 unlink 进行写, 我们只需要构造好数据即可.**

#### 原理(触发原因):

unlink 进行地址内容写操作没有进行检查. unlink 操作是从 chunk 双向环中摘除节点, 一般来说会在 malloc 时触发, 但是当需要 free 的 chunk 前后也有空闲 chunk, 会进行空闲 chunk 的合并, 这时需要 unlink 那个需要合并的空闲 chunk.

```
#define unlink( P, BK, FD ) {
BK = P->bk;
FD = P->fd;
FD->bk = BK;
BK->fd = FD;
}
```

由于堆的内容是可以控制的, FD 和 BK 均为可以控制的, 由此导致任意地址写内容.

#### 触发前提(限制):

free 的 chunk 前后存在空闲 chunk, 也可以通过溢出值覆盖需要 free 的 chunk 的 `P` 位, 能控制需要 unlink 的 chunk 的内容.

#### 保护机制:

```
eglibc-2.19/malloc/malloc.c:1410
if (__builtin_expect (FD->bk != P || BK->fd != P, 0))         \
  malloc_printerr (check_action, "corrupted double-linked list", P);      \
```

为什么这个判断可以防止 unlink 攻击, 首先很明确, `FD` 和 `BK`内容可控, 由于要求 `*(FD+24) == P && *(BK+16) == P`, 对 `FD` 和 `BK` 进行了限定, 即使可以找到内存中某个地址 `(FD+24)` , 使 `*(FD+24) == P` 成立, 但是当进行 `FD->bk = BK; BK->fd = FD`, 也会出现写入的地址不是关键地址, 写入的内容也不是我们想要的, 因此 `任意地址写任意内容` 变为 `限制地址写限制内容`, 导致无法被利用, 更何况这还需要找一个泄露任意地址内容的方法.

#### 绕过保护机制:

下面思路是应该算是逆向保护机制, 之后会在下面提到另一个思路. 方法都是一样的, 只不过理解方式不一样.

这个保护机制的绕过, 个人感觉不算是绕过, 应该是妥协了这个保护机制. 

下面讨论的 是针对 x64 而言, 字长 8 字节.

既然保护机制要求 `FD->bk == P && BK->fd == P`, 这里拿 `FD->bk ==P ` 进行举例, 也就是说 ` *(FD+24) == P`, 也就是 `*((P->fd) + 24) == P`, 所以我们需要一个已知地址的指针, 并且该指针指向 P, 下面作如下假设

```c
mchunkptr * unlinks[1] = { P };
P->fd = (void*)(&unlink[0])-24
P->bk = (void*)(&unlink[0])-16
```

这样就可以通过保护机制. 但是我们找到这样一个指针只能算是 **通过** 了保护机制, 然而如何利用这个指针实现任意地址写的需要进一步利用, 将在下面介绍由此导致的利用

#### 利用思路:

构造好 `P->bk` 和 `P->fd` 的值, 比如现在需要当调用 `free` 执行 `shellcode`, 这里需要的注意的因为 `free` 需要进行延迟绑定的, 所以 `.got.plt` 存放了解析后 `free` 函数的内存地址, 这也对应了 `*((P->fd) + 24) == P` 的 `*` 操作. 所以我们可以直接构造 `P->fd = <free@plt> + 24`, `P->bk = shellcode 地址`, 当进行 unlink 操作时, 触发 `FD->bk = BK`, 修改 `free@plt` 内容为 `shellcode` 地址, 同时为了避免 `BK->fd = FD` 的影响, 需要将 `shellcode` 的前面一小段字节设为 `nop`.

## 0x00 unlink(过保护机制)

#### 导致(最终结果):

实现任意内存地址写内容; 修改了 chunk 的起始地址, 写入操作无需借助原函数, 用户主动进行写操作 

#### 原理(触发原因):

构造一个特殊 chunk (craft a fake chunk) 和 内存覆盖(memory overlap)

这里转化为 `void*` 可以思考下.

在上面已经得到了如下的假设, 之前提到另一种理解方式, 其实需要找到指向 P 的已知地址的指针, 其实就是需要伪造一个 chunk, 这个 chunk 的 `chunk->bk == P`, 这个指针的地址就可以理解为 chunk 的地址, 其实就是 `(void*)(&unlink[0])-24`, 所以上面的整个过程可以理解为伪造了两个 fake chunk, `(void*)(&unlink[0])-24` 和 `(void*)(&unlink[0])-18` 其实就是这两个 fake chunk 的起始地址.

```c
mchunkptr * unlinks[1] = { P };
P->fd = (void*)(&unlink[0])-24
P->bk = (void*)(&unlink[0])-16
```

之后进行 unlink 操作, 触发 `BK->fd = FD` 也就是

```c
P->bk->fd = FD
((void*)(&unlink[0])-16)->fd = FD
*(void*)(&unlink[0]) = FD
unlink[0] = FD
unlink[0] = (void*)(&unlink[0])-24
```

经过 unlink 操作, `unlink[0]` 指向了 fake chunk, 但是这个 fake chunk 的地址是 `(void*)(&unlink[0])-24` 很特殊, 也就是说这个 fake chunk 与 `unlink[0]` 重叠了, 贼恐怖. 

按照正常逻辑, 假如有个函数 `edit_chunk(mchunkptr *ptr, char *content)`, 第一次调用 `edit_chunk(unlink[0], "A"*24+free@plt)`, 如此一来 `unlink[0]` 就被覆盖为 `free@plt`. 当再次调用 `edit_chunk(unlink[0], shllcode_address)` , 因为此时 `unlink[0]` 已经被恶意覆盖, 所以直接造成修改 `free@plt` 为 shellcode 的地址.

#### 触发前提(限制):

0. 能够触发 unlink
1. 一个指向 P 的已知地址的指针.
2. 可以重复修改 `unlink[0]` 指向的 chunk 的内容

#### 保护机制:

这里主要是利用 unlink 修改了指向 chunk 的地址, 之后由用户进行主动写操作, 对于 unlink 的防护, 我们已经伪造了 fake chunk, 该防护失效.

#### 利用思路:

见 **原理(触发原因)****



## 0x00 off-by-one

## Null Byte Off-by-one

#### PlaidDB(pwn 550)
