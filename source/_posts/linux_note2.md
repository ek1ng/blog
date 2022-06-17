---
title: 操作系统内核中的初始化工作
date: 2022-06-13 22:16:00
updated: 2022-06-13 22:16:00
tags: [OS,linux]
description: linux 0.11的源码阅读笔记(二)
---

看完了进入内核前的工作后，我网络编程课的抄写作业自然是可以圆满完成啦，不过看了一部分后觉得确实很有意思，所以也是决定继续看下去，并且计划看完linux源码后跟着MIT6.s081写一个小的操作系统内核，希望我能够在6.29之前完成这个工作哈哈也就是我开始军训之前（，补军训确实是个令人苦恼的事情。

## 操作系统内核中的初始化工作

### 概览main函数

现在我们已经进入操作系统内核啦，上篇文章我们说道，我们将main函数push到栈顶，而cs:eip是CPU执行下一条指令的地址，此时指向栈顶，所以接下来就开始执行main函数了，那我们先来看看代码

```c
void main(void)  /* This really IS void, no error here. */
{   /* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
 ROOT_DEV = ORIG_ROOT_DEV;
 drive_info = DRIVE_INFO;
 memory_end = (1<<20) + (EXT_MEM_K<<10);
 memory_end &= 0xfffff000;
 if (memory_end > 16*1024*1024)
  memory_end = 16*1024*1024;
 if (memory_end > 12*1024*1024) 
  buffer_memory_end = 4*1024*1024;
 else if (memory_end > 6*1024*1024)
  buffer_memory_end = 2*1024*1024;
 else
  buffer_memory_end = 1*1024*1024;
 main_memory_start = buffer_memory_end;
#ifdef RAMDISK
 main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
 mem_init(main_memory_start,memory_end);
 trap_init();
 blk_dev_init();
 chr_dev_init();
 tty_init();
 time_init();
 sched_init();
 buffer_init(buffer_memory_end);
 hd_init();
 floppy_init();
 sti();
 move_to_user_mode();
 if (!fork()) {  /* we count on this going ok */
  init();
 }
/*
 *   NOTE!!   For any other task 'pause()' would mean we have to get a
 * signal to awaken, but task0 is the sole exception (see 'schedule()')
 * as task 0 gets activated at every idle moment (when no other tasks
 * can run). For task0 'pause()' just means we go check if some other
 * task can run, and if not we return here.
 */
 for(;;) pause();
}
```

先来看这几句，这是一些参数的取值和计算

```C
void main(void)  /* This really IS void, no error here. */
{   /* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
 ROOT_DEV = ORIG_ROOT_DEV;
 drive_info = DRIVE_INFO;
 memory_end = (1<<20) + (EXT_MEM_K<<10);
 memory_end &= 0xfffff000;
 if (memory_end > 16*1024*1024)
  memory_end = 16*1024*1024;
 if (memory_end > 12*1024*1024) 
  buffer_memory_end = 4*1024*1024;
 else if (memory_end > 6*1024*1024)
  buffer_memory_end = 2*1024*1024;
 else
  buffer_memory_end = 1*1024*1024;
 main_memory_start = buffer_memory_end;
 ...
}
```

包括根设备`ROOT_DEV`,之前在汇编语言中获取的各个设备的参数信息`drive_info`,以及通过计算得到的内存边界`main_memory_start,main_memory_end,buffer_memory_start,buffer_memory_end`,还记得setup.s这个汇编程序中调用BIOS中断获取的各个设备信息，并且保存在约定好的内存地址0x90000上的表格嘛，我们现在就需要取用这些设备信息。

![图 1](https://s2.loli.net/2022/06/14/j4HYiR9cEwKD7as.png)  

接下来这一段是初始化init操作，包括内存初始化`mem_init`，中断初始化`trap_init`，进程调度初始化`sched_init`等等

```c
void main(void)  /* This really IS void, no error here. */
{
#ifdef RAMDISK
 main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
 mem_init(main_memory_start,memory_end);
 trap_init();
 blk_dev_init();
 chr_dev_init();
 tty_init();
 time_init();
 sched_init();
 buffer_init(buffer_memory_end);
 hd_init();
 floppy_init();
 ...
}
```

这一部分是切换到用户态模式，并且在新的进程中做一个最终的初始化init，在这个init函数里面会创建出一个进程没设置中断的标准IO,并且再创建出一个执行shell程序的进程来接受用户的命令

```c
void main(void)  /* This really IS void, no error here. */
{
 sti();
 move_to_user_mode();
 if (!fork()) {  /*we count on this going ok*/
  init();
 }
}
```

最后一部分是个死循环，当没有任何人物运行时，操作系统就会一直在这个死循环中。

```c
void main(void)  /* This really IS void, no error here. */
{
 /*
 *  NOTE!!   For any other task 'pause()' would mean we have to get a
 * signal to awaken, but task0 is the sole exception (see 'schedule()')
 * as task 0 gets activated at every idle moment (when no other tasks
 * can run). For task0 'pause()' just means we go check if some other
 * task can run, and if not we return here.
 */
 for(;;) pause();
}
```

### 规划内存


### mem_init

### trap_init

### blk_dev_init

### tty_init

### time_init

### sched_init

### buffer_init

### hd_init
