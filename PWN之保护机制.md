## 常见保护机制

大部分在 `CSAPP` 的3.12节有提到, 主要是为了对抗缓冲区溢出攻击

#### 1. 栈随机化(ASLR)
内存地址随机化, 可以蛮力枚举, 并且可以通过 `nop sled` 可以大大减少尝试次数.
#### 2. 栈保护(canary)
就是段寄存器中取一个值放到栈中, 之后在函数结束时判断该值是否放生修改.

```
Dump of assembler code for function echo:
   0x0804849d <+0>:     push   ebp
   0x0804849e <+1>:     mov    ebp,esp
   0x080484a0 <+3>:     sub    esp,0x28
   0x080484a3 <+6>:     mov    eax,gs:0x14 //CANNARY 值
=> 0x080484a9 <+12>:    mov    DWORD PTR [ebp-0xc],eax
   0x080484ac <+15>:    xor    eax,eax
   0x080484ae <+17>:    lea    eax,[ebp-0x14]
   0x080484b1 <+20>:    mov    DWORD PTR [esp],eax
   0x080484b4 <+23>:    call   0x8048350 <gets@plt>
   0x080484b9 <+28>:    lea    eax,[ebp-0x14]
   0x080484bc <+31>:    mov    DWORD PTR [esp],eax
   0x080484bf <+34>:    call   0x8048370 <puts@plt>
   0x080484c4 <+39>:    mov    eax,DWORD PTR [ebp-0xc]
   0x080484c7 <+42>:    xor    eax,DWORD PTR gs:0x14
   0x080484ce <+49>:    je     0x80484d5 <echo+56>
   0x080484d0 <+51>:    call   0x8048360 <__stack_chk_fail@plt>
   0x080484d5 <+56>:    leave
   0x080484d6 <+57>:    ret
End of assembler dump.
```
#### 3. NX(DEP)
数据内存页不可执行, 栈数据不可执行
