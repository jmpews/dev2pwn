#### 参考资料

```
http://cocoahuke.com/page2/
```

#### 内核处理以及二进制文件加载过程

```
execve:kern_exec.c {
__mac_execve:kern_exec.c {
exec_activate_image:kern_exec.c {
exec_mach_imgact:kern_exec.c {
load_machfile:kern_exec.c {
parse_machfile {
load_dylinker {
```

#### 加载dyld过程

```c
load_dylinker{
  get_macho_vnode{
    //读取dyld的fat_header
    vn_rdwr{}
    //读取dyld的mach_header
    vn_rdwr{}
  }
  parse_machfile{
  	//Map the load commands into kernel memory.
  	vn_rdwr{}
  	load_segment{
  		//这里进行了slide偏移, 并且在对_TEXT segment 进行映射时重新定位了, result->mach_header,  这个的原理像elf的segment加载时, 把elf-header算在第一个Segment上.
  		map_segment{}
  	  }
  	}
  }
}
```

dyld 的处理过程在 `dyld.cpp`, 从 `LC_MAIN` 拿到地址后转到 `dyld.cpp/_main()` 执行.

#### 处理环境变量

```c
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
	[...]
	configureProcessRestrictions(mainExecutableMH);

#if __MAC_OS_X_VERSION_MIN_REQUIRED
    if ( gLinkContext.processIsRestricted ) {
		pruneEnvironmentVariables(envp, &apple);
		// set again because envp and apple may have changed or moved
		setContext(mainExecutableMH, argc, argv, envp, apple);
	}
	else
#endif
	{
		checkEnvironmentVariables(envp);
		defaultUninitializedFallbackPaths(envp);
	}
```

```
configureProcessRestrictions:dyld.cpp
对 ios 和 osx 做了区分, ios 默认不支持任何环境变量
|
checkEnvironmentVariables:dyld.cpp
检查环境变量, 之后调用下一个函数做处理
|
processDyldEnvironmentVariable:dyld.cpp
处理环境变量, 设置gLinkContext
```

这里有一个关键的过程 `setContext(mainExecutableMH, argc, argv, envp, apple);` 设置上下文需要使用到的全局变量.

#### 加载macho执行文件

```c
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
	[...]
	
		// instantiate ImageLoader for main executable
		sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);
		gLinkContext.mainExecutable = sMainExecutable;
		gLinkContext.mainExecutableCodeSigned = hasCodeSignatureLoadCommand(mainExecutableMH);
```

```
// 检查文件格式, 加载主可执行文件, 记录该image到全局环境变量
instantiateFromLoadedImage:dyld.cpp
{
	// 处理加载命令, 根据加载命令处理加载可执行文件
	ImageLoaderMachO::instantiateMainExecutable:ImageLoaderMachO.cpp
	{	
		// 处理, 区分加载命令
		ImageLoaderMachO::sniffLoadCommands:ImageLoaderMachO.cpp
		// 根据加载命令, 开始加载可执行文件	ImageLoaderMachOCompressed::instantiateMainExecutable:ImageLoaderMachOCompressed.cpp
		{
			// 创建ImageLoaderMachOCompressed对象
			ImageLoaderMachOCompressed::instantiateStart:ImageLoaderMachOCompressed.cpp
			// 根据加载命令填充ImageLoaderMachOCompressed对象
			ImageLoaderMachOCompressed::instantiateFinish:ImageLoaderMachOCompressed.cpp
		}
	}
	// 记录image到全局变量
	addImage:dyld.cpp
}
```

#### 加载共享动态库

```c
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
	[...]
		// load shared cache
		checkSharedRegionDisable();
	#if DYLD_SHARED_CACHE_SUPPORT
		if ( gLinkContext.sharedRegionMode != ImageLoader::kDontUseSharedRegion ) {
			mapSharedCache();
		} else {
			dyld_kernel_image_info_t kernelCacheInfo;
			bzero(&kernelCacheInfo.uuid[0], sizeof(uuid_t));
			kernelCacheInfo.load_addr = 0;
			kernelCacheInfo.fsobjid.fid_objno = 0;
			kernelCacheInfo.fsobjid.fid_generation = 0;
			kernelCacheInfo.fsid.val[0] = 0;
			kernelCacheInfo.fsid.val[0] = 0;
			task_register_dyld_shared_cache_image_info(mach_task_self(), kernelCacheInfo, true, false);
		}
	#endif
```

```
// 判断是否存在共享动态库, 如果存在直接使用, 否则进行加载, gLinkContext记录共享库地址
mapSharedCache:dyld.cpp
{
}
```

#### 加载 DYLD_INSERT_LIBRARIES

```c
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
	[...]

		// load any inserted libraries
		if	( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
			for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib) 
				loadInsertedDylib(*lib);
		}
```

```
loadInsertedDylib:dyld.cpp
{
	load:dyld.cpp
	{
		loadPhase0:dyld.cpp
		loadPhase1:dyld.cpp
		loadPhase2:dyld.cpp
		loadPhase3:dyld.cpp
		loadPhase4:dyld.cpp
		loadPhase5:dyld.cpp
		loadPhase5check:dyld.cpp
		loadPhase5load:dyld.cpp
		loadPhase5stat:dyld.cpp
		loadPhase5load:dyld.cpp
		loadPhase5open:dyld.cpp
		loadPhase6
		{
			checkandAddImage::dyld.cpp
		}
		
	}
}
```

#### 加载自身依赖动态库

```c
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
	[...]

		link(sMainExecutable, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL), -1);
		sMainExecutable->setNeverUnloadRecursive();
		if ( sMainExecutable->forceFlat() ) {
			gLinkContext.bindFlat = true;
			gLinkContext.prebindUsage = ImageLoader::kUseNoPrebinding;
		}
```

```
link:dyld.cpp
{
	ImageLoader::link:ImageLoader.cpp
	{
		ImageLoader::recursiveLoadLibraries:ImageLoader.cpp
		{
			ImageLoaderMachO::doGetDependentLibraries:ImageLoader.cpp
			libraryLocator:dyld.cpp
			{
				load:dyld.cpp
			}
		}
	}
}
```

#### 其他

```
//之前通过addImage:dyld.cpp, 更新到dyld::gProcessInfo这个全局变量, 如果异常会调用下面的处理
syncAllImages:dyld.cpp
```

