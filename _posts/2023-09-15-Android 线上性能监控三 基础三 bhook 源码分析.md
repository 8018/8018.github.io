---
layout:     post
title:      Android 线上性能监控三 基础三 bhook 源码分析
date:       2023-09-15
author:     xflyme
header-img: img/post-bg-2023-09-15.jpg
catalog: true
tags:
    - 性能优化
    - Android 动态链接
    - Android native hook
---

一、本文目的
前文中写了一个 native hook 的简单例子。
本文将分析下字节开源的 [bhook](https://github.com/bytedance) 是怎么实现的。

字节关于 bhook 的介绍在[这里](https://zhuanlan.zhihu.com/p/401547387)，上面提到：
* 可以随时 hook 和 unhook 单个、部分、全部的调用者 ELF。
* 当新的 ELF 被加载到内存后，它应该自动的被执行所有预定的 hook 操作。
* 多个使用方如果 hook 了同一个调用点，它们应该可以彼此独立的执行 unhook，相互不干扰。
* 能自动避免由于 hook 引起的意外的“递归调用”和“环形调用”。

## 二、源码分析

### 2.1 初始化
![](/img/bhook-init.png)
上图是 bhook 初始化流程，核心逻辑在 bh_dl_open_linker 这个方法中。其会找到 android linker 的 base addr，以及将 linker 的符号表加载到内存中供后序使用。

初始化是还创建了 task manager、 hook manager 和 elf manager 等用于以后对 task 和 elf 的管理。

### 2.2 Hook 单个、部分还是全部调用者 ELF
##### 2.2.1 Hook api
bhook 提供了 api ，使用者可以选择 hook 单个、部分或全部。
```c
bytehook_stub_t bytehook_hook_single(const char *caller_path_name, const char *callee_path_name,
                                     const char *sym_name, void *new_func, bytehook_hooked_t hooked,
                                     void *hooked_arg) {
  const void *caller_addr = __builtin_return_address(0);
  return bh_core_hook_single(caller_path_name, callee_path_name, sym_name, new_func, hooked, hooked_arg,
                             (uintptr_t)caller_addr);
}

bytehook_stub_t bytehook_hook_partial(bytehook_caller_allow_filter_t caller_allow_filter,
                                      void *caller_allow_filter_arg, const char *callee_path_name,
                                      const char *sym_name, void *new_func, bytehook_hooked_t hooked,
                                      void *hooked_arg) {
  const void *caller_addr = __builtin_return_address(0);
  return bh_core_hook_partial(caller_allow_filter, caller_allow_filter_arg, callee_path_name, sym_name,
                              new_func, hooked, hooked_arg, (uintptr_t)caller_addr);
}

bytehook_stub_t bytehook_hook_all(const char *callee_path_name, const char *sym_name, void *new_func,
                                  bytehook_hooked_t hooked, void *hooked_arg) {
  const void *caller_addr = __builtin_return_address(0);
  return bh_core_hook_all(callee_path_name, sym_name, new_func, hooked, hooked_arg, (uintptr_t)caller_addr);
}
```

##### 2.2.2 Hook task
对于每个 hook “行为”，bhook 为其创建对应的 bh_task，便于后续管理。
bh_task 中定义了 bh_task_type_t 用于标识 hook 类型，bytehook_caller_allow_filter_t，caller_path_name 用于后续 hook 时找到对应的 elf 。

```c
typedef struct bh_task {
  uint32_t id;  // unique id
  bh_task_type_t type;
  bh_task_status_t status;

  // caller
  char *caller_path_name;                              // for single
  bytehook_caller_allow_filter_t caller_allow_filter;  // for partial
  void *caller_allow_filter_arg;                       // for partial

  // callee
  char *callee_path_name;
  void *callee_addr;

  // symbol
  char *sym_name;

  // new function address
  void *new_func;

  // callback
  bytehook_hooked_t hooked;
  void *hooked_arg;

  int hook_status_code;  // for single type

  void *manual_orig_func;  // for manual mode

  TAILQ_ENTRY(bh_task, ) link;
} bh_task_t;
```

##### 2.2.3 hook
Hook 时会根据 bh_task_t 类型做不同处理。

**bh_task_handle**
```c
static void bh_task_handle(bh_task_t *self) {
  switch (self->type) {
    case BH_TASK_TYPE_SINGLE: {
      bh_elf_t *caller_elf = bh_elf_manager_find_elf(bh_core_global()->elf_mgr, self->caller_path_name);
      if (NULL != caller_elf) bh_task_hook_or_unhook(self, caller_elf);
      break;
    }
    case BH_TASK_TYPE_ALL:
    case BH_TASK_TYPE_PARTIAL:
      bh_elf_manager_iterate(bh_core_global()->elf_mgr, bh_task_elf_iterate_cb, (void *)self);
      break;
  }
}
```
 
Single 类型的 hook 直接调用 bh_task_hook_or_unhook。
其他类型的先遍历，对于每个 elf 执行 bh_task_hook_or_unhook。

**bh_elf_manager_iterate**
```
void bh_elf_manager_iterate(bh_elf_manager_t *self, bh_elf_manager_iterate_cb_t cb, void *cb_arg) {
  if (0 == self->elfs_cnt) return;

  // get a copy of ELFs (only the pointers)
  bh_elf_t **copy_elfs = NULL;
  size_t copy_elfs_cnt = 0;
  pthread_rwlock_rdlock(&self->elfs_lock);
  if (self->elfs_cnt > 0) {
    if (NULL != (copy_elfs = malloc(sizeof(bh_elf_t *) * self->elfs_cnt))) {
      copy_elfs_cnt = self->elfs_cnt;
      size_t i = 0;
      bh_elf_t *elf;
      RB_FOREACH(elf, bh_elf_tree, &self->elfs) {
        copy_elfs[i++] = elf;
      }
    }
  }
  pthread_rwlock_unlock(&self->elfs_lock);

  // do callback copy ELFs (no need to lock)
  if (NULL != copy_elfs) {
    bool cb_next = true;
    for (size_t i = 0; i < copy_elfs_cnt; i++) {
      if (cb_next) cb_next = cb(copy_elfs[i], cb_arg);
    }
    free(copy_elfs);
  }
}
```

```
static bool bh_task_elf_iterate_cb(bh_elf_t *elf, void *arg) {
  return bh_task_hook_or_unhook((bh_task_t *)arg, elf);
}
```

**bh_task_hook_or_unhook**
```c
static bool bh_task_hook_or_unhook(bh_task_t *self, bh_elf_t *elf) {
  void (*hook_or_unhook)(bh_hook_manager_t *, bh_task_t *, bh_elf_t *) =
      (BH_TASK_STATUS_UNHOOKING == self->status ? bh_hook_manager_unhook : bh_hook_manager_hook);

  switch (self->type) {
    case BH_TASK_TYPE_SINGLE:
      if (bh_elf_is_match(elf, self->caller_path_name)) {
        hook_or_unhook(bh_core_global()->hook_mgr, self, elf);
        if (BH_TASK_STATUS_UNHOOKING != self->status) self->status = BH_TASK_STATUS_FINISHED;
        return false;  // already found the ELF for single task, no need to continue
      }
      return true;  // continue
    case BH_TASK_TYPE_PARTIAL:
      if (self->caller_allow_filter(elf->pathname, self->caller_allow_filter_arg))
        hook_or_unhook(bh_core_global()->hook_mgr, self, elf);
      return true;  // continue
    case BH_TASK_TYPE_ALL:
      hook_or_unhook(bh_core_global()->hook_mgr, self, elf);
      return true;  // continue
  }
}
```
bh_task_hook_or_unhook 如果是部分 hook 会根据 caller_allow_filter 判断是否需要 hook。

##### 2.2.4 小结
* bh_task 中定义了 hook 类型以及单个 hook 和部分 hook 的条件。
* 如果是单个 hook 直接找到对应的 elf 执行 hook。
* 如果是部分 hook 会遍历所有已加载的 elf ，如果复合 hook 条件，则 hook。

### 2.3 新 ELF 自动 hook
![](/img/bhook-open-process.png)
dl_open 后 bhook 的后序方法调用如图。
bhook hook 了 dl_open 方法，linker dl_open 之后会回到 bhook。
bhook 之后会对新加载的 elf 执行 hook。同样 dl_close 之后同样会执行相应的 unhook。
整个过程比较清晰，就不贴具体代码了。

### 2.4 避免环形调用和递归调用
根据字节介绍他们使用了 **trampoline** 来避免“递归调用” 和“环形调用”。
![](/img/bhook-trampoline.png)

##### 2.4.1 trampoline 的创建

**hook create:**
```c
static bh_hook_t *bh_hook_manager_create_hook(bh_hook_manager_t *self, void *got_addr, void *orig_func,
                                              void **trampo) {
  // create hook chain
  bh_hook_t *hook = bh_hook_create(got_addr, orig_func);
  if (NULL == hook) return NULL;

  // create trampoline for the hook chain
  *trampo = bh_trampo_create(hook);
  if (NULL == *trampo) {
    bh_hook_destroy(&hook);
    return NULL;
  }

  // save the hook chain
  RB_INSERT(bh_hook_tree, &self->hooks, hook);

  BH_LOG_INFO("hook chain: created for GOT %" PRIxPTR ", orig func %" PRIxPTR, (uintptr_t)got_addr,
              (uintptr_t)orig_func);
  return hook;
}
```

创建 b_hook 的时候会为当前 b_hook 创建一个 trampo。

**bh_trampo_create：**
```c
void *bh_trampo_create(bh_hook_t *hook) {
  size_t code_size = (uintptr_t)(&bh_trampo_data) - (uintptr_t)(bh_trampo_template_pointer());
  size_t data_size = sizeof(void *) + sizeof(void *);

  // create trampoline
  void *trampo = bh_trampo_allocate(code_size + data_size);
  if (NULL == trampo) return NULL;

  // fill in code
  BYTESIG_TRY(SIGSEGV, SIGBUS) {
    memcpy(trampo, bh_trampo_template_pointer(), code_size);
  }
  BYTESIG_CATCH() {
    return NULL;
  }
  BYTESIG_EXIT

  // file in data
  void **data = (void **)((uintptr_t)trampo + code_size);
  *data++ = (void *)bh_trampo_push_stack;
  *data = (void *)hook;

  // clear CPU cache
  __builtin___clear_cache((char *)trampo, (char *)trampo + code_size + data_size);

  BH_LOG_INFO("trampo: created for GOT %" PRIxPTR " at %" PRIxPTR ", size %zu + %zu = %zu",
              (uintptr_t)hook->got_addr, (uintptr_t)trampo, code_size, data_size, code_size + data_size);

#if defined(__arm__) && defined(__thumb__)
  trampo = (void *)((uintptr_t)trampo + 1);
#endif
  return trampo;
}
```

上述方法中的 bh_trampo_template_pointer 如下（arm64）：
```c
__attribute__((naked)) void bh_trampo_template() {
  __asm__(
      // Save caller-saved registers
      "sub   sp, sp, #0xd0            \n"
      "stp   q0, q1, [sp, #0xb0]      \n"
      "stp   q2, q3, [sp, #0x90]      \n"
      "stp   q4, q5, [sp, #0x70]      \n"
      "stp   q6, q7, [sp, #0x50]      \n"
      "stp   x0, x1, [sp, #0x40]      \n"
      "stp   x2, x3, [sp, #0x30]      \n"
      "stp   x4, x5, [sp, #0x20]      \n"
      "stp   x6, x7, [sp, #0x10]      \n"
      "stp   x8, lr, [sp]             \n"

      // Call bh_trampo_push_stack()
      "ldr   x0, .L_hook_ptr          \n"
      "mov   x1, lr                   \n"
      "ldr   x16, .L_push_stack       \n"
      "blr   x16                      \n"

      // Save the hook function's address to IP register
      "mov   x16, x0                  \n"

      // Restore caller-saved registers
      "ldp   x8, lr, [sp]             \n"
      "ldp   x6, x7, [sp, #0x10]      \n"
      "ldp   x4, x5, [sp, #0x20]      \n"
      "ldp   x2, x3, [sp, #0x30]      \n"
      "ldp   x0, x1, [sp, #0x40]      \n"
      "ldp   q6, q7, [sp, #0x50]      \n"
      "ldp   q4, q5, [sp, #0x70]      \n"
      "ldp   q2, q3, [sp, #0x90]      \n"
      "ldp   q0, q1, [sp, #0xb0]      \n"
      "add   sp, sp, #0xd0            \n"

      // Call hook function
      "br    x16                      \n"

      "bh_trampo_data:"
      ".global bh_trampo_data;"
      ".L_push_stack:"
      ".quad 0;"
      ".L_hook_ptr:"
      ".quad 0;");
}

```

trampo 创建过程中涉及到一段汇编，比较复杂，整理后逻辑如下：
* 申请一块内存，大小是 code_size + data_size，code_size 大小是 bh_trampo_data 减去 bh_trampo_template_pointer。
* bh_trampo_data 是在汇编模版中定义，bh_trampo_template_pointer 指向这段汇编的起始地址，因此 code_size 就是 bh_trampo_data 之前的这段代码所需要的内存空间。
* data_size 大小是两个 sizeof(void *)，也就是两个指针。一个只想 bh_trampo_push_stack 这个方法，一个指向当前 hook。
* bh_hook 中有一个链表，链表中记录着当前 callee 方法的所有 proxy 方法。

##### 2.4.2 trampo 的使用
根据字节官方图 hook 之后 caller 调用 trampoline ，trampoline 之后是一系列 proxy 方法，最后才是 callee。
trampoline 指向一段汇编，这段汇编逻辑如下：
* 保存当前寄存器中的数据，参数、返回地址等。
* 调用 bh_trampo_push_stack 。
  * bh_trampo_push_stack 会记录已经调用过的 proxy 方法，并判断有没有环形调用。 
  * 如果没有环形调用，返回下一个 proxy 方法。
  * 如果有环形调用，直接返回原始方法。
* 恢复 caller 所传的参数等数据，并调用 bh_trampo_push_stack 所返回的方法。

##### 2.4.3 trampo 替换原函数

创建了 trampo 之后在 hook 时使用 trampo 替换 origin func。
```c
static int bh_hook_manager_hook_single_got(bh_hook_manager_t *self, bh_elf_t *caller_elf, bh_task_t *task,
                                           void *got_addr, void **orig_func_ret) {
  ...

    // 1. add new-func to the hook chain
    void *trampo = NULL;
    void *orig_func_real = NULL;
    r = bh_hook_manager_add_func(self, caller_elf, got_addr, orig_func, task, &trampo, &orig_func_real);

    // 2. replace with the trampoline address if we haven't done it yet
    if (0 == r && NULL != trampo) {
      r = bh_hook_manager_replace_got_value(caller_elf, task, got_addr, orig_func, trampo);
      ...

  return r;
}

```

##### 2.4.4 小结
* trampoline 其实是一段汇编，这段汇编中包含两部分，代码和数据。
  * 代码部分用于保存和恢复原始方法原始方法的参数，返回地址等。
  * 两个数据指针一个指向 bh_trampo_push_stack 一个指向 b_hook。
    * b_hook 中有一个链表，其中保存着当前原始方法的所有 proxy 方法。
    * bh_trampo_push_stack 会判断是否有环形调用以及下一个需要调用的方法。
    
## 参考
* [字节跳动开源 Android PLT hook 方案 bhook](https://zhuanlan.zhihu.com/p/401547387)