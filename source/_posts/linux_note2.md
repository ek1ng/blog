---
title: 操作系统内核中的初始化工作
date: 2022-06-13 22:16:00
updated: 2022-06-13 22:16:00
tags: [OS,linux]
description: linux 0.11的源码阅读笔记(二)
---

看完了进入内核前的工作后，我网络编程课的抄写作业自然是可以圆满完成啦，不过看了一部分后觉得确实很有意思，所以也是决定继续看下去，并且计划看完linux源码后跟着MIT6.s081写一个小的操作系统内核，希望我能够在6.29之前完成这个工作哈哈也就是我开始军训之前，补军训确实是个令人苦恼的事情。

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

最后一部分是个死循环，当没有任何任务运行时，操作系统就会一直在这个死循环中。

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

main函数的大概内容就是这样啦，接下来我们逐句看看。

### 规划内存

先看看前面这些参数的取值和计算，这里主要的作用就是规划内存。

```c
 ROOT_DEV = ORIG_ROOT_DEV;
 drive_info = DRIVE_INFO;
```

开头两句中，`ROOT_DEV`是系统的根文件设备号，`drive_info`是之前setup.s汇编程序获取并且存储在内存0x90000处的设备信息。

```c
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
```

从这几句我们可以看出`memory_end`的值是1M+扩展内存大小，再和0xfffff000与运算，意思是舍去低3个16进制位，再之后的if语句根据`memory_end`的值也就是内存最大值控制缓冲区内存`buffer_memory_end`和`main_memory_start`的大小。

我们举个例子来理解一下这个操作的作用，假设内存为8M大小，memory_end（单位为B）的值为8\*1024\*1024,那么就会走倒数第二个分支，`buffer_memory_end`为2\*1024\*1024,`main_memory_end`也为2\*1024\*1024，通过这样的操作完成了对内存的管理。

![图 2](https://s2.loli.net/2022/06/18/XmJlCtpz4Sib5GA.png)  

接下来执行的就是`mem_init(main_memory_start,memory_end);`这个函数了，我们在上面给传入的参数做了对应的赋值，`main_memory_start`表示主内存地址的开头，`memory_end`表示内存地址的末尾。接下来我们看一下`mem_init`这个函数做了啥。

```c
/* these are not to be changed without changing head.s etc */
#define LOW_MEM 0x100000
#define PAGING_MEMORY (15*1024*1024)
#define PAGING_PAGES (PAGING_MEMORY>>12)
#define MAP_NR(addr) (((addr)-LOW_MEM)>>12)
#define USED 100

static long HIGH_MEMORY = 0;

static unsigned char mem_map [ PAGING_PAGES ] = {0,};

void mem_init(long start_mem, long end_mem)
{
 int i;

 HIGH_MEMORY = end_mem;
 for (i=0 ; i<PAGING_PAGES ; i++)
  mem_map[i] = USED;
 i = MAP_NR(start_mem);
 end_mem -= start_mem;
 end_mem >>= 12;
 while (end_mem-->0)
  mem_map[i++]=0;
}
```

`mem_init`函数的主要内容是对`mem_map`数组的各个位置赋值，先是给0到PAGING_PAGES的这些元素赋值USED,也就是100,其他的都赋值了0,用来表示这块内存是否被占用，所以这里所谓的管理就是用一个数组来记录哪些内存被占用了哪些内存没有。

接下来问题自然是初始化的时候哪些内存被占用了呢，一个数组元素又表示多大的内存空间呢？

我们仍然使用8M大小的内存空间举例
![图 3](https://s2.loli.net/2022/06/19/RWuB97djKvnYwiU.png)  
可以看出，`mem_map`数组的每个元素都表示一个4K的内存空间是否空闲，4K内存通常叫做1页内存，这种内存管理方式叫做分页管理。

1M以下的内存没有用数组`mem_map`记录，还记得我们之前算内存边界值的时候，总内存的大小也是用1M+扩展内存的大小计算的，其实是因为在这个内存的低1M的空间，是内核代码所在的区域，因此不能被污染也不需要去做管理。

从1M到2M的内存空间是缓冲区，这些地方也不是主内存区域，因此也会被直接标记为USED。

2M以上的内存空间就是主内存区域，而当前住内存没有任何程序在使用，因此我们现在全部初始化为0，也就是表示未被使用。

### trap_init

### blk_dev_init

### tty_init

### time_init

### sched_init

### buffer_init

### hd_init
