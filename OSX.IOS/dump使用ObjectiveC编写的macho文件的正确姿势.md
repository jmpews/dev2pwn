## dump使用ObjectiveC编写的macho文件的正确姿势

#### 问题1: 如何访问目标进程内存空间

使用 `task_for_pid` 取得对应的 `task_t` , 这里简单介绍下内部机制, 我也没有具体深究, 这个 `task_t` 的处理方式, 有些类似于 ELF 中 `link_map`, `link_map ` 对于用户 inlcude 的头文件和 libc 在进行链接加载时, 是两个不同的数据结构, 在链接加载时具有更多的成员. `task_t` 也是如此, 对于用户层而言仅为一个数字, 但是在内核处理, 该数据结构包含了该进程很多信息, 内存页的具体映射等等. 

之后利用 `vm_read_overwrite` 访问目标内存空间

#### 问题2: Macho文件的解析的起始位置

Macho 文件的加载是经过 ASLR 的, 所以我们需要获取到程序的加载地址. 这里获取程序的加载地址有几种方式.

```
1. task_info

2. vm_region 这个原理就是内核有个表记录了所有虚拟内存映射 用户层访问不到, 可以参考下面
http://stackoverflow.com/questions/10301542/getting-process-base-address-in-mac-osx

3. 部分暴力内存搜索, 根据虚拟内存页和xnu的aslr特点, 可以限定内存搜索范围, 其实和上面有些类似
https://github.com/jmpews/evilMACHO/tree/master/dumpRuntimeMacho
```

#### 问题3: 对于OC编写的Macho文件如何dump到类和方法

关键代码, 也是我分析的步骤.

```c
//objc-file.mm
GETSECT(_getObjc2ClassList,           classref_t,      "__objc_classlist");

//objc-runtime-new.mm
static Class remapClass(classref_t cls)
{
    return remapClass((Class)cls);
}

//Object.mm, objc.h
typedef struct objc_class *Class;

//objc-runtime-new.mm
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
  
    ...ignore methods
}

//
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

//
struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
  
    ...ignore methods
}

//
struct class_data_bits_t {
    // Values are the FAST_ flags above.
    uintptr_t bits;
  
  	...ignore other methods
      
    class_rw_t* data() {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
	}
}
```

上面是C++的 struct 的继承, 对应到 C 中应该是如何实现的？如果你阅读过 Python 源码就会知道, C++ 中的继承相当于子类 struct 在'开头'包含父类 struct.

以下为我展开的 C 下的 struct.

```c
struct objc_class {
	struct objc_class *isa; 		// metaclass
	struct objc_class *superclass;	// superclas
	struct bucket_t *_buckets;		// cache
	mask_t _mask;					// vtable
    mask_t _occupied;				// vtable
  	uintptr_t bits;					// data
}
```

这里对于 `mask_t` 的两个成员有点特殊, 可以通过下面的声明以及 `_occupied` 的含义, 推断一二, `_mask` 仅占地址类型的一半, 另一部分留作占位, 以备后用.

```
#if __LP64__
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits
#else
typedef uint16_t mask_t;
#endif
```

下面再提一下, 另一个特殊的 `uintptr_t bits`, 看似并不是一个地址, 但是如果仔细观察 `class_data_bits_t` 的方法, 会发现有一个 `data()` 的对象方法, 所以说, 从某种意思上来说 `bits` 就是一个地址, 这个地址存放了 `class_rw_t` 类型的数据.

之前没有看到上面注释, 其实上面有一段注释(注释写的位置, 很迷)

```
// class_data_bits_t is the class_t->data field (class_rw_t pointer plus flags)
// The extra bits are optimized for the retain/release and alloc/dealloc paths.
```

下面具体分析下 `class_rw_t` 这个重要的结构体

```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;

#if SUPPORT_INDEXED_ISA
    uint32_t index;
#endif
}
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

在之后我又发现发现了一些关于 objc_class 更多的资料, 这几篇文章都或多或少介绍过 objc_class, 都需要阅读, 希望大家更能了解到如何对不是很熟的技术进行分析的步骤.

```
https://bestswifter.com/runtime-category/
http://www.cnblogs.com/jiazhh/articles/3309085.html
http://blog.sunnyxx.com/2015/09/13/class-ivar-layout/
https://xiuchundao.me/post/runtime-object-and-class
http://draveness.me/method-struct/
```

#### C++ 与 C 的结构对应

对于类的继承的处理?

可以看成 C 中结构体的包含另一个结构体(可展开)

对于模板的处理?

举个例子

```
template <typename List>
class list_array_tt {
    uint32_t count;
    List* lists[0];
}
```

其实在我们处理时 List 可以使用 **void** 进行替代, 当需要使用时直接强制转换即可.

