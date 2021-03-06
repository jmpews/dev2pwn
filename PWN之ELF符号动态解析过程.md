下面以该源程序分析整个动态链接解释符号的过程:

```
#include <stdio.h>
int main(int argc, char *argv[])
{
printf("Hello, world%d\n", 1);
return 0;
}
```
先查看 `section` 表

```
➜  hook readelf -S test
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
  [16] .eh_frame_hdr     PROGBITS        080484f0 0004f0 00002c 00   A  0   0  4
  [17] .eh_frame         PROGBITS        0804851c 00051c 0000b0 00   A  0   0  4
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
  [28] .symtab           SYMTAB          00000000 001604 000430 10     29  45  4
  [29] .strtab           STRTAB          00000000 001a34 000251 00      0   0  1
```

反汇编 `main` 函数

```
gdb-peda$ disassemble  main
Dump of assembler code for function main:
   0x0804841d <+0>: push   ebp
   0x0804841e <+1>: mov    ebp,esp
   0x08048420 <+3>: and    esp,0xfffffff0
   0x08048423 <+6>: sub    esp,0x10
   0x08048426 <+9>: mov    DWORD PTR [esp+0x4],0x1
   0x0804842e <+17>:    mov    DWORD PTR [esp],0x80484e0
   0x08048435 <+24>:    call   0x80482f0 <printf@plt>
   0x0804843a <+29>:    mov    eax,0x0
   0x0804843f <+34>:    leave
   0x08048440 <+35>:    ret
End of assembler dump.
```

单步执行至 `0x80482f0 <printf@plt>`, 这里需要用到前面的 `.got.plt`, `.plt`的知识, 大概说一下, `0x804a00c` 为 `.got.plt` 对应函数引用地址, 由于未初始化, `0x804a00c` 存放的就是下一句的地址 `0x080482f6`, 之后跳到 `0x80482e0` 开始进行初始化相关操作

```
gdb-peda$ disassemble
Dump of assembler code for function printf@plt:
=> 0x080482f0 <+0>: jmp    DWORD PTR ds:0x804a00c
   0x080482f6 <+6>: push   0x0
   0x080482fb <+11>:    jmp    0x80482e0
End of assembler dump.
gdb-peda$ x/w 0x804a00c
0x804a00c <printf@got.plt>: 0x080482f6
```

`0x804a008` 该地址预放了 `_dl_runtime_resolve` 函数的地址(`.got.plt` 表的第三项), 因为会跳到 `_dl_runtime_resolve` 进行初始化工作. 这里通过两次 `push` 操作, 其实是向 `_dl_fixup` 传入两个参数, 一个是需要重定位符号的偏移, 一个是 `link_map` 的地址

```
gdb-peda$ x/4i 0x80482e0
   0x80482e0:   push   DWORD PTR ds:0x804a004
=> 0x80482e6:   jmp    DWORD PTR ds:0x804a008
   0x80482ec:   add    BYTE PTR [eax],al
   0x80482ee:   add    BYTE PTR [eax],al
gdb-peda$ x/w 0x804a008
0x804a008:  0xb7ff24b0
gdb-peda$ disassemble 0xb7ff24b0
Dump of assembler code for function _dl_runtime_resolve:
   0xb7ff24b0 <+0>: push   eax
   0xb7ff24b1 <+1>: push   ecx
   0xb7ff24b2 <+2>: push   edx
   0xb7ff24b3 <+3>: mov    edx,DWORD PTR [esp+0x10]
   0xb7ff24b7 <+7>: mov    eax,DWORD PTR [esp+0xc]
   0xb7ff24bb <+11>:    call   0xb7fec080 <_dl_fixup>
   0xb7ff24c0 <+16>:    pop    edx
   0xb7ff24c1 <+17>:    mov    ecx,DWORD PTR [esp]
   0xb7ff24c4 <+20>:    mov    DWORD PTR [esp],eax
   0xb7ff24c7 <+23>:    mov    eax,DWORD PTR [esp+0x4]
   0xb7ff24cb <+27>:    ret    0xc
End of assembler dump.
```

这里暂时不管 `_dl_fixup` 细节, 可以发现初始化完毕后 `.got.plt` 中第四项变为 `printf` 真实地址, 同时 `ret 0xc(参数表明pop多少字节)` 跳转到 `printf` 继续执行.

```
gdb-peda$ info registers sp
sp             0xbffff66c   0xbffff66c
gdb-peda$ x/4w 0xbffff66c #sp地址
0xbffff66c: 0xb7e71410  0x00000001  0xb7fff938  0x00000000
gdb-peda$ x/4w 0x0804a000 # .got.plt 函数引用表
0x804a000:  0x08049f14  0xb7fff938  0xb7ff24b0  0xb7e71410
gdb-peda$ disassemble _dl_runtime_resolve
Dump of assembler code for function _dl_runtime_resolve:
   0xb7ff24b0 <+0> :    push   eax
   0xb7ff24b1 <+1> :    push   ecx
   0xb7ff24b2 <+2> :    push   edx
   0xb7ff24b3 <+3> :    mov    edx,DWORD PTR [esp+0x10]
   0xb7ff24b7 <+7> :    mov    eax,DWORD PTR [esp+0xc]
   0xb7ff24bb <+11>:    call   0xb7fec080 <_dl_fixup>
   0xb7ff24c0 <+16>:    pop    edx
   0xb7ff24c1 <+17>:    mov    ecx,DWORD PTR [esp]
   0xb7ff24c4 <+20>:    mov    DWORD PTR [esp],eax
   0xb7ff24c7 <+23>:    mov    eax,DWORD PTR [esp+0x4]
=> 0xb7ff24cb <+27>:    ret    0xc
End of assembler dump.
```

当跳到 `.plt` 进行重定位时进行了 `push   0x0`, 其中 `0x0` 相当于 `.rel.plt` 中的偏移量, 根据 `Elf32_Rel` 结构进行重定位, `printf_retloc.r_offset= 0x0804a00c` 对应 `.got.plt` 中的第四项,

```
gdb-peda$ x/8x 0x0804829c # .rel.plt 重定位表
0x804829c:  0x0804a00c  0x00000107  0x0804a010  0x00000207
0x80482ac:  0x0804a014  0x00000307  0x08ec8353  0x000093e8
```

#### 分析 `_dl_fixup` 函数(`eglibc-2.19/elf/dl-runtime.c`)

参考链接:

```
http://blog.chinaunix.net/uid-21471835-id-441227.html
```

先把几个宏定义列举出来 (`eglibc-2.19/sysdeps/i386/ldsodefs.h`)

```
/* All references to the value of l_info[DT_PLTGOT],
  l_info[DT_STRTAB], l_info[DT_SYMTAB], l_info[DT_RELA],
  l_info[DT_REL], l_info[DT_JMPREL], and l_info[VERSYMIDX (DT_VERSYM)]
  have to be accessed via the D_PTR macro.  The macro is needed since for
  most architectures the entry is already relocated - but for some not
  and we need to relocate at access time.  */
#ifdef DL_RO_DYN_SECTION
# define D_PTR(map, i) ((map)->i->d_un.d_ptr + (map)->l_addr)
#else
# define D_PTR(map, i) (map)->i->d_un.d_ptr
#endi

/* Result of the lookup functions and how to retrieve the base address.  */
typedef struct link_map *lookup_t;
#define LOOKUP_VALUE(map) map
#define LOOKUP_VALUE_ADDRESS(map) ((map) ? (map)->l_addr : 0
```

```
#if (!ELF_MACHINE_NO_RELA && !defined ELF_MACHINE_PLT_REL) \
    || ELF_MACHINE_NO_REL
# define PLTREL  ElfW(Rela)
#else
# define PLTREL  ElfW(Rel)
#endif

/* The fixup functions might have need special attributes.  If none
   are provided define the macro as empty.  */
#ifndef ARCH_FIXUP_ATTRIBUTE
# define ARCH_FIXUP_ATTRIBUTE
#endif

#ifndef reloc_offset
# define reloc_offset reloc_arg
# define reloc_index  reloc_arg / sizeof (PLTREL)
#endif



/* This function is called through a special trampoline from the PLT the
   first time each PLT entry is called.  We must perform the relocation
   specified in the PLT of the given shared object, and return the resolved
   function address to the trampoline, which will restart the original call
   to that address.  Future calls will bounce directly from the PLT to the
   function.  */

DL_FIXUP_VALUE_TYPE
attribute_hidden __attribute ((noinline)) ARCH_FIXUP_ATTRIBUTE
_dl_fixup (
# ifdef ELF_MACHINE_RUNTIME_FIXUP_ARGS
       ELF_MACHINE_RUNTIME_FIXUP_ARGS,
# endif
       struct link_map *l, ElfW(Word) reloc_arg) // 两个参数, link_map 地址, 需要重定位符号的偏移
{
  // D_PTR :访问 link_map 必须通过这个宏
  // 动态链接符号表
  const ElfW(Sym) *const symtab
    = (const void *) D_PTR (l, l_info[DT_SYMTAB]);
  // 动态链接字符串表
  const char *strtab = (const void *) D_PTR (l, l_info[DT_STRTAB]);

  // reloc_offset即_dl_fixup的第二个参数, 函数符号在 `.rel.plt` 中的偏移.
  // 获取对应重定位表中的结构
  const PLTREL *const reloc
    = (const void *) (D_PTR (l, l_info[DT_JMPREL]) + reloc_offset);
  # 获取对应符号表的结构
  const ElfW(Sym) *sym = &symtab[ELFW(R_SYM) (reloc->r_info)];
  # 获取需要重定向入口, 根据 R_386_JUMP_SLOT 计算方法, 对应 `.got.plt` 中函数引用地址
  void *const rel_addr = (void *)(l->l_addr + reloc->r_offset);
  lookup_t result;
  DL_FIXUP_VALUE_TYPE value;

  /* Sanity check that we're really looking at a PLT relocation.  */
  /* 检查重定位方式必须是ELF_MACHINE_JMP_SLOT */
  assert (ELFW(R_TYPE)(reloc->r_info) == ELF_MACHINE_JMP_SLOT);

   /* Look up the target symbol.  If the normal lookup rules are not
      used don't look in the global scope.  */
  if (__builtin_expect (ELFW(ST_VISIBILITY) (sym->st_other), 0) == 0)
    {
      const struct r_found_version *version = NULL;

      if (l->l_info[VERSYMIDX (DT_VERSYM)] != NULL)
    {
      const ElfW(Half) *vernum =
        (const void *) D_PTR (l, l_info[VERSYMIDX (DT_VERSYM)]);
      ElfW(Half) ndx = vernum[ELFW(R_SYM) (reloc->r_info)] & 0x7fff;
      version = &l->l_versions[ndx];
      if (version->hash == 0)
        version = NULL;
    }

      /* We need to keep the scope around so do some locking.  This is
     not necessary for objects which cannot be unloaded or when
     we are not using any threads (yet).  */
      int flags = DL_LOOKUP_ADD_DEPENDENCY;
      if (!RTLD_SINGLE_THREAD_P)
    {
      THREAD_GSCOPE_SET_FLAG ();
      flags |= DL_LOOKUP_GSCOPE_LOCK;
    }

#ifdef RTLD_ENABLE_FOREIGN_CALL
      RTLD_ENABLE_FOREIGN_CALL;
#endif
      // 查找定义该符号的模块的装载地址, result的类型为link_map, 注意这里调用后, sym 变为 libc.so.6 中的printf的符号Sym结构,sym->st_value 为printf符号在libc.so.6中段偏移
      result = _dl_lookup_symbol_x (strtab + sym->st_name, l, &sym, l->l_scope,
                    version, ELF_RTYPE_CLASS_PLT, flags, NULL);

      /* We are done with the global scope.  */
      if (!RTLD_SINGLE_THREAD_P)
    THREAD_GSCOPE_RESET_FLAG ();

#ifdef RTLD_FINALIZE_FOREIGN_CALL
      RTLD_FINALIZE_FOREIGN_CALL;
#endif

      /* Currently result contains the base load address (or link map)
     of the object that defines sym.  Now add in the symbol
     offset.  */
     /* 根据模块装载地址和函数符号的段偏移获得函数的实际地址 */
      value = DL_FIXUP_MAKE_VALUE (result,
                   sym ? (LOOKUP_VALUE_ADDRESS (result)
                      + sym->st_value) : 0);
    }
  else
    {
      /* We already found the symbol.  The module (and therefore its load
     address) is also known.  */
      value = DL_FIXUP_MAKE_VALUE (l, l->l_addr + sym->st_value);
      result = l;
    }

  /* And now perhaps the relocation addend.  */
  value = elf_machine_plt_value (l, reloc, value);

  if (sym != NULL
      && __builtin_expect (ELFW(ST_TYPE) (sym->st_info) == STT_GNU_IFUNC, 0))
    value = elf_ifunc_invoke (DL_FIXUP_VALUE_ADDR (value));

  /* Finally, fix up the plt itself.  */
  if (__builtin_expect (GLRO(dl_bind_not), 0))
    return value;

  # 修正.got.plt函数引用地址(rel_addr)的值为printf的地址
  return elf_machine_fixup_plt (l, result, reloc, rel_addr, value);
}
```

```
gdb-peda$ p reloc->r_info //重定位符号在符号表中的索引
$1 = 0x107
gdb-peda$ p reloc->r_offset //重定位的入口偏移
$2 = 0x804a00c
gdb-peda$ p sym //未执行_dl_lookup_symbol_x
$3 = (const Elf32_Sym *) 0x80481dc

gdb-peda$ p sym //执行_dl_lookup_symbol_x后
$2 = (const Elf32_Sym *) 0xb7e2a6d4
gdb-peda$ p sym->st_value //执行_dl_lookup_symbol_x后, sym指向发生改变, printf符号的在libc.so.6段偏移
$9 = 0x4d410
gdb-peda$ p result->l_addr //模块装载地址
$10 = 0xb7e24000
gdb-peda$ disassemble 0x4d410+0xb7e24000
Dump of assembler code for function __printf:
   0xb7e71410 <+0>: push   ebx
   0xb7e71411 <+1>: sub    esp,0x18
   0xb7e71414 <+4>: call   0xb7f4a7db <__x86.get_pc_thunk.bx>
   0xb7e71419 <+9>: add    ebx,0x15dbe7
   0xb7e7141f <+15>:    lea    eax,[esp+0x24]
   0xb7e71423 <+19>:    mov    DWORD PTR [esp+0x8],eax
   0xb7e71427 <+23>:    mov    eax,DWORD PTR [esp+0x20]
   0xb7e7142b <+27>:    mov    DWORD PTR [esp+0x4],eax
   0xb7e7142f <+31>:    mov    eax,DWORD PTR [ebx-0x70]
   0xb7e71435 <+37>:    mov    eax,DWORD PTR [eax]
   0xb7e71437 <+39>:    mov    DWORD PTR [esp],eax
   0xb7e7143a <+42>:    call   0xb7e67810 <_IO_vfprintf_internal>
   0xb7e7143f <+47>:    add    esp,0x18
   0xb7e71442 <+50>:    pop    ebx
   0xb7e71443 <+51>:    ret
End of assembler dump.
```

#### 利用场景

```
TODO
```

#### 参考链接

```
<程序员自我修养>

http://www.longene.org/techdoc/0750005001224576724.html
http://blog.chinaunix.net/uid-21471835-id-441227.html(很全)

https://www.ibm.com/developerworks/cn/linux/l-elf/part1/

```


