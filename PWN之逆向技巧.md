## PWN之逆向技巧

#### 生成shellcode

需要将汇编指令生成16进制码, 可以使用 `NASM` 这个工具, `OSX` 直接 `brew install nasm` 即可.

```
_start: jmp string

begin: pop eax ; char *file
xor ecx ,ecx ; *caller
mov edx ,0x1 ; int mode

mov ebx, 0x12345678 ; addr of _dl_open()
call ebx ; call _dl_open!
add esp, 0x4

int3 ; breakpoint

string: call begin
db "/tmp/ourlibby.so",0x00
```

使用 `nasm -f elf32 -o test.o test.asm` 生成目标文件, 然后使用 `objdump -d test.o` 查看目标文件

```
➜  sshf objdump -d test.o

test.o:     file format elf32-i386


Disassembly of section .text:

00000000 <_start>:
   0:   eb 13                   jmp    15 <string>

00000002 <begin>:
   2:   58                      pop    %eax
   3:   31 c9                   xor    %ecx,%ecx
   5:   ba 01 00 00 00          mov    $0x1,%edx
   a:   bb 78 56 34 12          mov    $0x12345678,%ebx
   f:   ff d3                   call   *%ebx
  11:   83 c4 04                add    $0x4,%esp
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
		...
```

#### 汇编一段函数

```
#使用gcc生成目标文件
gcc -c test.c
#使用objdump查看汇编代码实现
objdump -d test.o
```
