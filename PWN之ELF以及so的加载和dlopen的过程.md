## ELF以及so的加载和dlopen的过程

这里先提问题, 有些问题显而易见就不说了, 比如怎么判断是否是 ELF 文件, 肯定通过判断 ELF 魔数等等, 

#### 0x01 怎么判断需要加载那些 so

通过读取 `.dynamic` 段内的结构类型为 `DT_NEED` 的内容

```
➜  hook readelf -d dy

Dynamic section at offset 0xf0c contains 25 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libdl.so.2]
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000000c (INIT)                       0x80483fc
 0x0000000d (FINI)                       0x80486a4
 0x00000019 (INIT_ARRAY)                 0x8049f00
 0x0000001b (INIT_ARRAYSZ)               4 (bytes)
 0x0000001a (FINI_ARRAY)                 0x8049f04
 0x0000001c (FINI_ARRAYSZ)               4 (bytes)
 0x6ffffef5 (GNU_HASH)                   0x80481ac
 0x00000005 (STRTAB)                     0x804828c
 0x00000006 (SYMTAB)                     0x80481cc
 0x0000000a (STRSZ)                      200 (bytes)
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000015 (DEBUG)                      0x0
 0x00000003 (PLTGOT)                     0x804a000
 0x00000002 (PLTRELSZ)                   56 (bytes)
 0x00000014 (PLTREL)                     REL
 0x00000017 (JMPREL)                     0x80483c4
 0x00000011 (REL)                        0x80483bc
 0x00000012 (RELSZ)                      8 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffe (VERNEED)                    0x804836c
 0x6fffffff (VERNEEDNUM)                 2
 0x6ffffff0 (VERSYM)                     0x8048354
 0x00000000 (NULL)                       0x0
```

这个过程通过 `_dl_map_object_deps` 完成, 其中需要解决的一个很重要的问题就是如何避免重复加载.

```
A.so 依赖 C.so
B.so 依赖 C.so
如何避免重复加载 C.so
```

#### 0x02 怎么加载 so 到内存

对于这个过程最核心的过程就是 `_dl_map_object_from_fd()`

**0x01 `dl_open_worker()`**

整个 `dl_open()` 的工作都在 `dl_open_worker()` 这个函数内实现,  需要 so 以及其依赖, 并实现重定位

**0x02 `_dl_map_object()`**

这个过程已经得到需要加载的 so 的名称或者说是路径, 假如仅仅是 so 的名称, 需要在常用目录进行搜索以及 `LD_PRELOAD` 等位置, 找到具体路径后, 校验 ELF 格式后取得文件句柄. 

**0x03 `_dl_map_object_from_fd()`**

这个过程是 so 加载到内存最核心的工作, 进行内存映射.

首先得到 so 的 `Phdr` 内容, 根据 ELF 的 Header 找到 Program Header, 到这一步把整个文件使用 mmap 映射到内存, 确定了基址, 之后再对每一个 Segment 进行 **基址+虚拟地址** 的映射, 这个过程中使用 mmap `MAP_FIXED` 标志, 这其实利用前一步的 mmap 的结果, 以确保正确的地址映射.

这里可能有疑问就是两次 mmap 是否会造成内存额外消耗, 其实并不是, 由于 `MAP_FIED` 标志位的作用就是如果映射的位置是重叠的就

#### 0x03 如何完成符号重定位