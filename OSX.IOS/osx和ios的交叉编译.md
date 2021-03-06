## macOS/IOS 交叉编译

#### 交叉编译参数

host 参数

编译好的在什么平台下运行

#### 编译 readline for IOS

Darwin 版本对应的 macOS 和 IOS 的版本参考 `http://guides.macrumors.com/Darwin` 和 `https://www.theiphonewiki.com/wiki/Kernel`

```
export CC="clang"
export CXX="clang"
export CFLAGS="-arch armv7 -arch arm64 -mios-version-min=8.3 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS10.1.sdk"
```

```
./configure bash_cv_wcwidth_broken=yes --host=arm-apple-darwin \
--prefix=/Users/jmpews/Downloads/readline-7.0/build/arm/ \
--disable-shared
make
```

#### 编译readline for macOS

```
export CC="clang"
export CXX="clang"
export CFLAGS="-arch x86_64 -arch i386"
```

```
./configure \
--prefix=/Users/jmpews/Downloads/readline-6.3/build/ \
--disable-shared
```

```
//usage example
clang -L/Users/jmpews/Downloads/readline-6.3/build/lib -I/Users/jmpews/Downloads/readline-6.3/build/include -lreadline -lncurses  test.c
```

#### 编译readline for linux

```
export CC="clang"
export CXX="clang"
export CFLAGS="-arch i386"
```

```
./configure bash_cv_wcwidth_broken=yes --host i386-pc-linux \
--prefix=/Users/jmpews/Downloads/readline-7.0/build/linux32/ \
--disable-shared
make
```

#### 编译 dumpdecrypted

```
https://github.com/stefanesser/dumpdecrypted/blob/master/Makefile
```

#### 编译 libjpeg-turbo

```
https://android.googlesource.com/platform/external/libjpeg-turbo/+/HEAD/BUILDING.md
```



