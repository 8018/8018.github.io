---
layout:     post
title:      Android 线上性能监控二 基础二 Android hook demo
date:       2023-09-08
author:     xflyme
header-img: img/post-bg-2023-09-08.png
catalog: true
tags:
    - 性能优化
    - Android 动态链接
---

一、本文目的

[Android 线上性能监控一 基础一 Android 动态链接](https://8018.github.io/2023/09/21/Android-线上性能监控一-基础一-Android-动态链接/) 中把 Android 动态链接流程走了一遍，并写了一个小示例以验证链接过程。

本文将继续完善这个示例，自己动手去 hook 一个函数。

## 二、动态链接示例

### 2.1 示例代码
定义一个方法，并将它打包成共享库，然后在主程序中调用这个方法。
共享库方法定义：
```c
//test.h
#include <stdio.h>
void say_hello();
(base) 
```

共享方法实现：
```c
//test.c
#include "test.h"
#include <stdio.h>

void say_hello()
{
    printf("Hello world!");
}
```

主程序：
```c
//main.c
#include "test.h"

int main()
{
    say_hello();
    return 0;
}
```

### 2.2 示例编译
用 ndk 中的编译工具编译。
编译共享库：
```sh
/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android29-clang --sysroot=/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/platforms/android-29/arch-arm64 -isystem/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/sysroot/usr/include -isystem/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/sysroot/usr/include/aarch64-linux-android -pie /Users/xxx/code/AndroidHookTest/test.c -fPIC -shared -o /Users/xxx/code/AndroidHookTest/libtest.so
```
本人的电脑中有其他编译环境，未为 ndk 配置环境变量，使用的是绝对目录，可以根据自己环境配置调整编译命令。

编译主程序：
```c
/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android29-clang --sysroot=/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/platforms/android-29/arch-arm64 -isystem/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/sysroot/usr/include -isystem/Users/xxx/Library/Android/sdk/ndk/21.4.7075529/sysroot/usr/include/aarch64-linux-android -pie /Users/michaelliu/code/AndroidHookTest/main.c -L. -ltest -o /Users/xxx/code/AndroidHookTest/main
```

### 2.3 运行示例
上面编译之后有两个产物，main 和 libtest.so。
可将其 push 到 Android 虚拟机中执行。

```c
adb push ./libtest.so ./main /data/local/tmp
adb shell "chmod +x /data/local/tmp/main"
adb shell "export LD_LIBRARY_PATH=/data/local/tmp; /data/local/tmp/main"
```
运行结果如下：
![](/img/linker-demo-1.png)

## 三、Hook

hook printf 方法，在其前后加入我们的逻辑。
根据前文 [[# Android 线上性能监控和优化二 Android 动态链接]] Android 动态链接会在 .got 中生成条目，在其中记录相关函数的「真实地址」。
我们只需要找到相关条目，将其地址改为我们自己函数的地址。

**libtest.so 的 .rela.plt**
```c
Relocation section '.rela.plt' at offset 0x458 contains 4 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000001fd8  000300000402 R_AARCH64_JUMP_SL 0000000000000000 printf@LIBC + 0
000000001fe0  000400000402 R_AARCH64_JUMP_SL 0000000000000000 __cxa_finalize@LIBC + 0
000000001fe8  000500000402 R_AARCH64_JUMP_SL 0000000000000000 __register_atfork@LIBC + 0
000000001ff0  000600000402 R_AARCH64_JUMP_SL 0000000000000000 __cxa_atexit@LIBC + 0
(base) 

```
可以看到 printf@LIBC 的 offset 是 0x1fd8，即它在 GOT 中的地址。
链接器加载共享库到内存中时，对于当前共享库它有一个起始地址即基地址或 load_bias。
基地址加上偏移量才是这个条目的真实地址。

代码如下：
```c
// hookmain.c
#include "test.h"
#include <inttypes.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include <sys/mman.h>
 
#define PAGE_START(addr) ((addr) & PAGE_MASK)
#define PAGE_END(addr)   (PAGE_START(addr) + PAGE_SIZE)
 
int my_puts(const char *str){
    puts("hook!!!!");
    return printf(str);
}
 
void hook(){
    char       line[512];
    FILE      *fp;
   uintptr_t  base_addr = 0;
    uintptr_t  addr;
 
    //find base address of libtest.so
	///proc/pid/maps 会记录当前加载的共享库
    //找到 libtest.so 读取它的 base addr
    if(NULL == (fp = fopen("/proc/self/maps", "r"))) return;
    while(fgets(line, sizeof(line), fp)){
        if(NULL != strstr(line, "libtest.so") && sscanf(line, "%"PRIxPTR"-%*lx %*4s 00000000", &base_addr) == 1)
            break;
    }
    fclose(fp);
    if(0 == base_addr) return;
 
    //the absolute address
	//base addr 加上 offset 即为绝对地址
    addr = base_addr + 0x1fd8;
 
    //add write permission
    //修改目标地址权限，加上可写
    mprotect((void *)PAGE_START(addr), PAGE_SIZE, PROT_READ | PROT_WRITE);
 
    //replace the function address
    *(void **)addr = my_puts;
 
    //clear instruction cache
    __builtin___clear_cache((void *)PAGE_START(addr), (void *)PAGE_END(addr));
}
 
int main(){
    hook();
    say_hello();
    return 0;
}
```

运行结果：
![](/img/linker-demo-2.png)

代码中将找到的 base_addr 打印出来和终端中通过 cat /proc/12376/maps 可以看到的到的地址是一样的。
![](/img/linker-demo-3.png)
![](/img/linker-demo-4.png)
482293587968 转换成 16 进制即 0x704af02000。

### 3.1 小结
* proc/pid/maps 找到当前库的 base addr
* base addr 和当前函数的 offset 计算出其在 GOT 表中的地址。
* 修改函数地址权限，添加可写。
* 将我们 proxy 函数的地址写到 GOT 表中。
* 
## 参考
* [Android PLT hook 概述](https://github.com/iqiyi/xHook/blob/master/docs/overview/android_plt_hook_overview.zh-CN.md)