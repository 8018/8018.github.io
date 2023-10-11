---
layout:     post
title:      Android 线上性能监控一 基础一 Android 动态链接
date:       2023-09-15
author:     xflyme
header-img: img/post-bg-2023-09-15.jpg
catalog: true
tags:
    - 性能优化
    - Android 动态链接
---

本文内容基于 Android 10。

相关代码目录：
```cpp
bionic/linker/linker_main.cpp
bionic/linker/linker_phdr.cpp
bionic/linker/linker_soinfo.cpp
bionic/linker/linker.cpp
```

## 一、本文目的

Android 线上性能监控、性能优化会用到 native hook。其中 PLT hook 涉及到动态链接相关的知识，本文梳理下 Android 动态链接的过程。
## 二、动态链接

维基百科中对于[动态链接](https://zh.wikipedia.org/wiki/%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5%E5%BA%93)的描述为：
> 所谓动态链接，就是把一些经常会共享的代码（静态链接的[OBJ](https://zh.wikipedia.org/w/index.php?title=OBJ&action=edit&redlink=1)程序库）制作成DLL档，当可执行文件调用到DLL档内的函数时，Windows操作系统才会把DLL档加载存储器内，DLL档本身的结构就是可执行档，当程序有需求时函数才进行链接。通过动态链接方式，存储器浪费的情形将可大幅降低。[静态链接库](https://zh.wikipedia.org/wiki/%E9%9D%9C%E6%85%8B%E9%80%A3%E7%B5%90%E5%87%BD%E5%BC%8F%E5%BA%AB)则是直接[链接](https://zh.wikipedia.org/wiki/%E9%93%BE%E6%8E%A5%E5%99%A8)到可执行文件。

这段描述是关于 win 动态链接的，Unix 中动态链接的目的同样如此。

## 三、Linux 动态链接
### 3.1 装载时重定位
动态链接库有可能被多个程序使用，加载到内存中的地址是不确定的，共享对象有可能加载到任意地址。因此要生成地址无关代码，在加载时重定位——装载时重定位。

### 3.2 地址无关代码
共享库的指令部分不能在装载时应为地址改变而改变，所以要把指令中需要动态修改的部分提出出来放到数据段，这样动态链接时指令部分可以保持不变。共享模块中跟地址相关的被放到 GOT 中。
比如一个函数调用，函数真实地址放在 GOT 中，指令中地址指向 GOT 表项。这样程序指令部分就可以做到地址无关。

### 3.3 GOT
Global offset table 全局偏移量表用于记录共享库中所用到的符号的真实地址。
那么是不是要在动态库装载时去重定位把 GOT 表项中填入真实的符号地址？
x86 Linux 中并不是，共享库中的符号并不一定都会用到，无法在装载时确定哪些要做重定位。如果所有符号都做重定位又太浪费时间。
因此又引入了延迟绑定，即当用到特定符号时再进行重定位。

### 3.4 延迟绑定与PLT
延迟绑定时当函数第一次调用时才进行绑定，如果没有用到则不进行绑定。
ELF 延迟绑定用到了 PLT——Procedure Linkage Table。当调用某个函数时，通常的做法是通过 GOT 表项进行跳转。为了实现延迟绑定，在其间又加了一个中间层 PLT。即原始指令指向 PLT 中地址， PLT 表项中指向 GOT 中的地址，GOT 表项中存储函数的真实地址。

### 3.5 重定位
GOT 中并没有函数的真实地址，需要在第一次函数调用时重定位。
那么，怎么做？

```
func@plt:
jmp *(func@GOT)
push n #n 
push moduleID
jump _dl_runtime_resolve@plt
```

在装载时链接器并没有将函数的真实地址填入到该项，而是将上面代码的第二条指令(push n，n为这个符号引用在重定位表中“.rel.plt”的下标)的地址填入到 GOT 中。所以第一次函数调用时会调用动态链接器来完成符号解析。
整体过程如下图：
![linker got 未重定位](/img/linker_got1.png)

![linker got 已重定位](/img/linker_got2.png)

## 四、Android 动态链接

![Android Linker 初始化](/img/Android_linker.png)

上图为 Android 动态链接过程（忽略 namespace 部分）。
**与上面说的 linux 动态链接不同的是 Android 动态链接没有延迟绑定，而是在加载 so 的时候进行绑定。**
 查找加载共享 so 和链接符号的主要逻辑在 linker.cpp 文件 find_libraries 方法中。其中有 0—7 八个步骤。整体过程可以去看[源码](https://cs.android.com/android/platform/superproject/+/android10-release:bionic/linker/linker.cpp)，本文仅关注加载 so 信息，load library 到内存中和符号链接三个步骤，对应 step 1，step 2 和 step 6。

为了便于理解和加深记忆本文采用示例和源码结合的方式。
示例代码如下：
```c
//test.h
#include <stdio.h>
void say_hello();

//test.c
#include "test.h"
#include <stdio.h>

void say_hello()
{
    printf("Hello world!");
}
```

定义了一个 say_hello 函数，其中会使用到 libc 中的 printf 方法。
将其编译成共享库，编译命令如下：
```sh
aarch64-linux-android29-clang 
--sysroot=arch-arm64 
-isystem/sysroot/usr/include 
-isystem/sysroot/usr/include/aarch64-linux-android 
-pie test.c -fPIC -shared -o libtest.so
```

后续的分析中，重点关注 printf 的链接过程。

### 4.1 解析 so 信息

#### 4.1.1 查找路径

路径定义在 linker.cpp 文件中。
```cpp
#if defined(__LP64__)
static const char* const kSystemLibDir        = "/system/lib64";
static const char* const kOdmLibDir           = "/odm/lib64";
static const char* const kVendorLibDir        = "/vendor/lib64";
static const char* const kAsanSystemLibDir    = "/data/asan/system/lib64";
static const char* const kAsanOdmLibDir       = "/data/asan/odm/lib64";
static const char* const kAsanVendorLibDir    = "/data/asan/vendor/lib64";
static const char* const kRuntimeApexLibDir   = "/apex/com.android.runtime/lib64";
#else
static const char* const kSystemLibDir        = "/system/lib";
static const char* const kOdmLibDir           = "/odm/lib";
static const char* const kVendorLibDir        = "/vendor/lib";
static const char* const kAsanSystemLibDir    = "/data/asan/system/lib";
static const char* const kAsanOdmLibDir       = "/data/asan/odm/lib";
static const char* const kAsanVendorLibDir    = "/data/asan/vendor/lib";
static const char* const kRuntimeApexLibDir   = "/apex/com.android.runtime/lib";
#endif

static const char* const kAsanLibDirPrefix = "/data/asan";

static const char* const kDefaultLdPaths[] = {
  kSystemLibDir,
  kOdmLibDir,
  kVendorLibDir,
  nullptr
};
```

默认情况下会从 system、dom、vendor 三个路径下查找共享库。

#### 4.1.2 读取 elf 信息
找到共享库之后会通过 ElfReader:: Read 方法读取共享库的 header、program header、dynamic section 等信息。
##### 4.1.2.1 ElfReader::Read
```cpp
bool ElfReader::Read(const char* name, int fd, off64_t file_offset, off64_t file_size) {
  if (did_read_) {
    return true;
  }
  name_ = name;
  fd_ = fd;
  file_offset_ = file_offset;
  file_size_ = file_size;

  if (ReadElfHeader() &&
      VerifyElfHeader() &&
      ReadProgramHeaders() &&
      ReadSectionHeaders() &&
      ReadDynamicSection()) {
    did_read_ = true;
  }

  return did_read_;
}
```

##### 4.1.2.2 ElfReader::ReadSectionHeaders
```cpp
bool ElfReader::ReadSectionHeaders() {
  shdr_num_ = header_.e_shnum;

  ...

  shdr_table_ = static_cast<const ElfW(Shdr)*>(shdr_fragment_.data());
  return true;
}

```

ReadSectionHeaders 中读取出 shdr_table_，shdr 即 section header。

##### 4.1.2.3 ElfReader::ReadDynamicSection
```cpp
bool ElfReader::ReadDynamicSection() {
  // 1. Find .dynamic section (in section headers)
  const ElfW(Shdr)* dynamic_shdr = nullptr;
  for (size_t i = 0; i < shdr_num_; ++i) {
	//读取 .dynamic 表项。
    if (shdr_table_[i].sh_type == SHT_DYNAMIC) {
      dynamic_shdr = &shdr_table_ [i];
      break;
    }
  }
	
  ...

  dynamic_ = static_cast<const ElfW(Dyn)*>(dynamic_fragment_.data());

  ...
  return true;
}
```
从 shdr_table_ 读取出 .dynamic 表项。
上面示例对应 SectionHeaders 如下：
```c
//aarch64-linux-android-readelf -S --wide libtest.so
There are 20 section headers, starting at offset 0x18d0:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .gnu.hash         GNU_HASH        0000000000000190 000190 000044 00   A  2   0  8
  [ 2] .dynsym           DYNSYM          00000000000001d8 0001d8 000168 18   A  3   3  8
  [ 3] .dynstr           STRTAB          0000000000000340 000340 000090 00   A  0   0  1
  [ 4] .gnu.version      VERSYM          00000000000003d0 0003d0 00001e 02   A  2   0  2
  [ 5] .gnu.version_r    VERNEED         00000000000003f0 0003f0 000020 00   A  3   1  8
  [ 6] .rela.dyn         RELA            0000000000000410 000410 000048 18   A  2   0  8
  [ 7] .rela.plt         RELA            0000000000000458 000458 000060 18  AI  2  15  8
  [ 8] .plt              PROGBITS        00000000000004c0 0004c0 000060 10  AX  0   0 16
  [ 9] .text             PROGBITS        0000000000000520 000520 000064 00  AX  0   0  4
  [10] .rodata           PROGBITS        0000000000000584 000584 00000d 01 AMS  0   0  1
  [11] .note.android.ident NOTE            0000000000000594 000594 000098 00   A  0   0  4
  [12] .fini_array       FINI_ARRAY      0000000000001df8 000df8 000010 08  WA  0   0  8
  [13] .data.rel.ro      PROGBITS        0000000000001e08 000e08 000008 00  WA  0   0  8
  [14] .dynamic          DYNAMIC         0000000000001e10 000e10 0001b0 10  WA  3   0  8
  [15] .got              PROGBITS        0000000000001fc0 000fc0 000040 08  WA  0   0  8
  [16] .comment          PROGBITS        0000000000000000 001000 0000b5 01  MS  0   0  1
  [17] .shstrtab         STRTAB          0000000000000000 001815 0000b7 00      0   0  1
  [18] .symtab           SYMTAB          0000000000000000 0010b8 000588 18     19  47  8
  [19] .strtab           STRTAB          0000000000000000 001640 0001d5 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

从中可以看出 .dynamic 对应的偏移地址是 0x1e10。

##### 4.1.2.4 so info::prelink_image
```cpp
bool soinfo::prelink_image() {
  /* Extract dynamic section */
  ElfW(Word) dynamic_flags = 0;
  phdr_table_get_dynamic_section(phdr, phnum, load_bias, &dynamic, &dynamic_flags);

  for (ElfW(Dyn)* d = dynamic; d->d_tag != DT_NULL; ++d) {
    switch (d->d_tag) {
      ...

      case DT_JMPREL:
#if defined(USE_RELA)
        plt_rela_ = reinterpret_cast<ElfW(Rela)*>(load_bias + d->d_un.d_ptr);
#else
        plt_rel_ = reinterpret_cast<ElfW(Rel)*>(load_bias + d->d_un.d_ptr);
#endif
        break;

      ...
#if defined(USE_RELA)
      case DT_RELA:
        rela_ = reinterpret_cast<ElfW(Rela)*>(load_bias + d->d_un.d_ptr);
        break;

      ...

      case DT_NEEDED:
        ++needed_count;
        break;
	   ...
      break;
    }
  }

  ...
  return true;
}
```

prelink 阶段会遍历 dynamic 表，根据不同的 Dynamic 类型进行解析。

继续看示例，下面两个分别是 .dynamic 的原始数据和解析后的数据。
可以看到它的起始地址是 0x1e10，和 section header 中的 .dynamic 地址一样。

```c
//aarch64-linux-android-readelf -x .dynamic libtest.so
Hex dump of section '.dynamic':
  0x00001e10 01000000 00000000 40000000 00000000 ........@.......
  0x00001e20 01000000 00000000 49000000 00000000 ........I.......
  0x00001e30 1a000000 00000000 f81d0000 00000000 ................
  0x00001e40 1c000000 00000000 10000000 00000000 ................
  0x00001e50 f5feff6f 00000000 90010000 00000000 ...o............
  0x00001e60 05000000 00000000 40030000 00000000 ........@.......
  0x00001e70 06000000 00000000 d8010000 00000000 ................
  0x00001e80 0a000000 00000000 90000000 00000000 ................
  0x00001e90 0b000000 00000000 18000000 00000000 ................
  0x00001ea0 03000000 00000000 c01f0000 00000000 ................
  0x00001eb0 02000000 00000000 60000000 00000000 ........`.......
  0x00001ec0 14000000 00000000 07000000 00000000 ................
  0x00001ed0 17000000 00000000 58040000 00000000 ........X.......
  0x00001ee0 07000000 00000000 10040000 00000000 ................
  0x00001ef0 08000000 00000000 48000000 00000000 ........H.......
  0x00001f00 09000000 00000000 18000000 00000000 ................
  0x00001f10 1e000000 00000000 08000000 00000000 ................
  0x00001f20 fbffff6f 00000000 01000000 00000000 ...o............
  0x00001f30 feffff6f 00000000 f0030000 00000000 ...o............
  0x00001f40 ffffff6f 00000000 01000000 00000000 ...o............
  0x00001f50 f0ffff6f 00000000 d0030000 00000000 ...o............
  0x00001f60 f9ffff6f 00000000 03000000 00000000 ...o............
  0x00001f70 00000000 00000000 00000000 00000000 ................
  0x00001f80 00000000 00000000 00000000 00000000 ................
  0x00001f90 00000000 00000000 00000000 00000000 ................
  0x00001fa0 00000000 00000000 00000000 00000000 ................
  0x00001fb0 00000000 00000000 00000000 00000000 ................
```

```c
//aarch64-linux-android-readelf -d libtest.so 
Dynamic section at offset 0xe10 contains 23 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libdl.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so]
 0x000000000000001a (FINI_ARRAY)         0x1df8
 0x000000000000001c (FINI_ARRAYSZ)       16 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x190
 0x0000000000000005 (STRTAB)             0x340
 0x0000000000000006 (SYMTAB)             0x1d8
 0x000000000000000a (STRSZ)              144 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000003 (PLTGOT)             0x1fc0
 0x0000000000000002 (PLTRELSZ)           96 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x458
 0x0000000000000007 (RELA)               0x410
 0x0000000000000008 (RELASZ)             72 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000000000001e (FLAGS)              BIND_NOW
 0x000000006ffffffb (FLAGS_1)            Flags: NOW
 0x000000006ffffffe (VERNEED)            0x3f0
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x3d0
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0
```

Dynamic Section 数据格式定义如下：

```c
typedef struct {
        Elf32_Sword d_tag;
        union {
                Elf32_Word      d_val;
                Elf32_Addr      d_ptr;
                Elf32_Off       d_off;
        } d_un;
} Elf32_Dyn;//32位程序

typedef struct {
        Elf64_Xword d_tag;
        union {
                Elf64_Xword     d_val;
                Elf64_Addr      d_ptr;
        } d_un;
} Elf64_Dyn;
```

Elf64_Xword 和 Elf64_Addr 都是 64 位。
其中 d_tag  决定这是什么类别信息，以及该如何解析 d_un 内部数据。具体类型可参阅[Dynamic Section](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-42444.html)
我们关注 0x1ed0 这一列，可以看到它的类型是 JMPREL，也就是函数跳转。
  0x00001ed0 17000000 00000000 58040000 00000000 
这里数据存储方式是小端存储，解码后的 type 是 0x17，offset 地址是 0x458。
即  0x0000000000000017 (JMPREL)             0x458
也就是说它在 plt 表中的地址是 0x458。

```cpp
plt_rela_ = reinterpret_cast<ElfW(Rela)*>(load_bias + d->d_un.d_ptr);
```

根据 prelink 这个地址之后会被存储到 plt_rela_ 中。

##### 4.1.2.5 小结
* Read 方法中先后读取 headers、ProgramHeaderSetion、ProgramDynamicSetion 等信息。
* 先读取 ProgramHeaderSetion 从中读取 shdr_table_ 地址。
* 从 section header 中读取 dynamic section 地址，dynamic_shdr = &shdr_table_ [i]。
* prelink_image 中根据类型（DT_NEEDED、DT_JMPREL）等提取出依赖的共享库和 rela_ 等信息供后续使用。

#### 4.1.3 load 共享库到内存中

load 时最后会通过 mmap 把共享库加载到内存中。

#### 4.1.4 link

link 阶段逻辑在 soinfo::link_image、soinfo::relocate 等方法 中，其会根据 .dynamic 中定义的信息找到对应的表，然后链接。
如：
```cpp
if (plt_rela_ != nullptr) {
	DEBUG("[ relocating %s plt rela ]", get_realpath());
	if (!relocate(version_tracker,
		plain_reloc_iterator(plt_rela_, plt_rela_count_), global_group, 			local_group)) {
		return false;
	}
}
```

plt_rela_ 即在 prelink 时保存的 DT_JMPREL类型的地址。

```cpp
bool soinfo::relocate(const VersionTracker& version_tracker, ElfRelIteratorT&& rel_iterator,
                      const soinfo_list_t& global_group, const soinfo_list_t& local_group) {
  const size_t tls_tp_base = __libc_shared_globals()->static_tls_layout.offset_thread_pointer();
  std::vector<std::pair<TlsDescriptor*, size_t>> deferred_tlsdesc_relocs;

  for (size_t idx = 0; rel_iterator.has_next(); ++idx) {
    const auto rel = rel_iterator.next();
    if (rel == nullptr) {
      return false;
    }

	//解析出 .rela.plt 当前项的 type 和 sym
    ElfW(Word) type = ELFW(R_TYPE)(rel->r_info);
    ElfW(Word) sym = ELFW(R_SYM)(rel->r_info);
	//计算出当前 rela.plt 表项地址
	//.rela.plt 表项中记录着 .got 表项的地址
    ElfW(Addr) reloc = static_cast<ElfW(Addr)>(rel->r_offset + load_bias);
    ElfW(Addr) sym_addr = 0;
    const char* sym_name = nullptr;
    ElfW(Addr) addend = get_addend(rel, reloc);

    DEBUG("Processing \"%s\" relocation at index %zd", get_realpath(), idx);
    if (type == R_GENERIC_NONE) {
      continue;
    }

    const ElfW(Sym)* s = nullptr;
    soinfo* lsi = nullptr;

    if (sym == 0) {
      if (is_tls_reloc(type)) {
        lsi = this;
      }
    } else if (ELF_ST_BIND(symtab_[sym].st_info) == STB_LOCAL && is_tls_reloc(type)) {
      s = &symtab_[sym];
      sym_name = get_string(s->st_name);
      DL_ERR("unexpected TLS reference to local symbol \"%s\": "
             "sym type %d, rel type %u (idx %zu of \"%s\")",
             sym_name, ELF_ST_TYPE(s->st_info), type, idx, get_realpath());
      return false;
    } else {
	  //在符号表中找出特定符号
      sym_name = get_string(symtab_[sym].st_name);
	  
	  // 根据 sym_name 找出包含当前符号的库和在其中的地址
	  if (!soinfo_do_lookup(this, sym_name, vi, &lsi, global_group, local_group, &s)) {
        return false;
      }

      ...

	  // 在包含当前符号的库中解析出符号地址
	  // Android 不支持延迟绑定因此这个函数直接返回地址
	  sym_addr = lsi->resolve_symbol_address(s);
	  ...
    }
	
	switch (type) {
      case R_GENERIC_JUMP_SLOT:
        count_relocation(kRelocAbsolute);
        MARK(rel->r_offset);
        TRACE_TYPE(RELO, "RELO JMP_SLOT %16p <- %16p %s\n",
                   reinterpret_cast<void*>(reloc),
                   reinterpret_cast<void*>(sym_addr + addend), sym_name);
		//将当前符号的真实地址写到 .got 表项中
        *reinterpret_cast<ElfW(Addr)*>(reloc) = (sym_addr + addend);
        break;
	...
  return true;
}
```

##### 4.1.4.1 示例

```cpp
Relocation section '.rela.plt' at offset 0x458 contains 4 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000001fd8  000300000402 R_AARCH64_JUMP_SL 0000000000000000 printf@LIBC + 0
000000001fe0  000400000402 R_AARCH64_JUMP_SL 0000000000000000 __cxa_finalize@LIBC + 0
000000001fe8  000500000402 R_AARCH64_JUMP_SL 0000000000000000 __register_atfork@LIBC + 0
000000001ff0  000600000402 R_AARCH64_JUMP_SL 0000000000000000 __cxa_atexit@LIBC + 0

The decoding of unwind sections for machine type AArch64 is not currently supported.

Symbol table '.dynsym' contains 15 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000520     0 SECTION LOCAL  DEFAULT    9 
     2: 0000000000001e08     0 SECTION LOCAL  DEFAULT   13 
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@LIBC (2)
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_finalize@LIBC (2)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __register_atfork@LIBC (2)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_atexit@LIBC (2)
     7: 0000000000002000     0 NOTYPE  GLOBAL DEFAULT  ABS _bss_end__
     8: 0000000000000568    28 FUNC    GLOBAL DEFAULT    9 say_hello
     9: 0000000000002000     0 NOTYPE  GLOBAL DEFAULT  ABS _edata
    10: 0000000000002000     0 NOTYPE  GLOBAL DEFAULT  ABS _end
    11: 0000000000002000     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_start
    12: 0000000000002000     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_end__
    13: 0000000000002000     0 NOTYPE  GLOBAL DEFAULT  ABS __end__
    14: 0000000000002000     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_start__
```

> Relocation section '.rela.plt' at offset 0x458 contains 4 entries:
这句话表明 .rela.plt 是需要重定位的 section ，它在内存中的地址是 0x458，即我们上面看到的函数跳转地址。
它的第一个表项是 printf@LIBC，offset 是 0x1fd8 即它在 .got 中的地址是 0x1fd8。

在 .dynsym 表中 printf@LIBC value 是 0 ，type 是 UND 即它是一个需要重定位的符号。
而 say_hello 这个函数的 value 则不为 0，因为它是我们自己定义的函数。

通过 objdump 可以看到 0x568 即为 say_hello 的真实地址。
```cpp
//aarch64-linux-android-objdump -d libtest.so
0000000000000568 <say_hello>:
 568:	a9bf7bfd 	stp	x29, x30, [sp,#-16]!
 56c:	910003fd 	mov	x29, sp
 570:	90000000 	adrp	x0, 0 <printf@plt-0x4e0>
 574:	91161000 	add	x0, x0, #0x584
 578:	97ffffda 	bl	4e0 <printf@plt>
 57c:	a8c17bfd 	ldp	x29, x30, [sp],#16
 580:	d65f03c0 	ret
```

##### 4.1.4.2 小结
* 找到当前需要链接的 .rela.plt 表项
* 根据其 sym_name 找到其在当前 symtab_（.dynsym）中的信息。
* soinfo_do_lookup 遍历 global_group 在其中寻找当前符号。
  * 根据 gnu hash，在gnu hash 中找到对应节点 index。
  * 用 index 在 symtab_ 定位。
* 计算出当前符号的真实地址，填入到当前 .got 表项中。(.rela.plt 中记录着当前函数在 .got 中的地址)

## 参考
* [《深入理解计算机系统》](https://hansimov.gitbook.io/csapp/part2/ch07-linking/7.12-position-independent-code)
* 《程序员的自我修养：链接、装载与库》
* [elf](https://refspecs.linuxfoundation.org/elf/elf.pdf)
* [深入了解GOT,PLT和动态链接](https://www.cnblogs.com/pannengzhi/p/2018-04-09-about-got-plt.html)
* [Android Native库的加载及动态链接的过程](https://cloud.tencent.com/developer/article/1736997)
* [android源码分析之linker初始化](https://www.cnblogs.com/revercc/p/16299712.html)
* [ELF格式解读 Dynamic节](https://blog.csdn.net/qfanmingyiq/article/details/124527430)
* [深入分析动态链接的延迟绑定](https://www.ngui.cc/article/show-1270238.html)














