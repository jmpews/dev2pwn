#### LC_REQ_DYLD说明

```
/*
 * After MacOS X 10.1 when a new load command is added that is required to be
 * understood by the dynamic linker for the image to execute properly the
 * LC_REQ_DYLD bit will be or'ed into the load command constant.  If the dynamic
 * linker sees such a load command it it does not understand will issue a
 * "unknown load command required for execution" error and refuse to use the
 * image.  Other load commands without this bit that are not understood will
 * simply be ignored.
 */
#define LC_REQ_DYLD 0x80000000
```

#### 关于 dyld_shared_cache

参考链接

```
http://iphonedevwiki.net/index.php/Dyld_shared_cache
```

这里提一下如何使用 dyld 内源码提供的工具进行 extract.

先从 `https://opensource.apple.com/tarballs/dyld/dyld-421.2.tar.gz` 取得源码, 修改 `/launch-cache/dsc_extractor.cpp` 

```
// 取消这里条件编译
#if 1 
// test program
#include <stdio.h>
#include <stddef.h>
#include <dlfcn.h>


typedef int (*extractor_proc)(const char* shared_cache_file_path, const char* extraction_root_path,
													void (^progress)(unsigned current, unsigned total));

int main(int argc, const char* argv[])
{
	if ( argc != 3 ) {
		fprintf(stderr, "usage: dsc_extractor <path-to-cache-file> <path-to-device-dir>\n");
		return 1;
	}
	
	//void* handle = dlopen("/Volumes/my/src/dyld/build/Debug/dsc_extractor.bundle", RTLD_LAZY);
	void* handle = dlopen("/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle", RTLD_LAZY);
	if ( handle == NULL ) {
		fprintf(stderr, "dsc_extractor.bundle could not be loaded\n");
		return 1;
	}
	
	extractor_proc proc = (extractor_proc)dlsym(handle, "dyld_shared_cache_extract_dylibs_progress");
	if ( proc == NULL ) {
		fprintf(stderr, "dsc_extractor.bundle did not have dyld_shared_cache_extract_dylibs_progress symbol\n");
		return 1;
	}
	// 修改这里, 从而不使用 bundle 内的函数, 方便调试
	int result = dyld_shared_cache_extract_dylibs_progress(argv[1], argv[2], ^(unsigned c, unsigned total) { printf("%d/%d\n", c, total); } );
	fprintf(stderr, "dyld_shared_cache_extract_dylibs_progress() => %d\n", result);
	return 0;
}


#endif
```

对于源码分析, 因为还不知道有什么样的利用方法.

#### 如何编译 libobjc.A.dylib

先说参考资料

```
//如何使用编译后的libobjc.A.dylib
http://blog.csdn.net/wotors/article/details/54426316
//配置objc编译环境的正确姿势
http://www.gfzj.us/tech/2015/04/01/objc-runtime-compile-from-source-code.html
//已配置好的objc源码
https://github.com/RetVal/objc-runtime
```

