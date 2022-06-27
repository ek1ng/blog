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

现在我们已经进入操作系统内核啦，上篇文章我们说道，我们将main函数push到栈顶，而cs:eip是CPU执行下一条指令的地址，此时指向栈顶，所以接下来就开始执行main了，那我们先来概览一下main函数的代码。

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

### 硬件中断向量初始化

在说`trap_init()`做了什么之前，我们需要先说说中断程序执行机制。

操作系统实质上是一个中断驱动的死循环，就像我们之前概览main函数里面看到的一样，最后是进入一个死循环，操作系统主要通过提前注册的中断机制和其对应的中断处理函数来进行一些事件的处理，比如说点击鼠标，按下键盘或者是执行程序。

CPU提供了两种中断程序执行的机制，中断和异常。前一个中断是动词，而后一个中断指中断机制。中断机制是一个异步事件，通常由IO设备出发，比如鼠标键盘，而异常机制是一个同步事件，是CPU在执行指令时检测到的反常条件，比如缺页异常等等。

中断和异常两种机制都会让CPU收到一个中断号，比如中断机制，按下键盘，然后CPU就会得到一个对应的中断号，异常机制呢，CPU自己执行指令时检测到某种反常情况，然后根据反常情况给自己一个对应的中断号。诶这个时候我们是不是又会想起来在`进入Linux内核前的准备`一文中提到过的`INT`指令，例如`INT 0x80`这个指令就是相当于直接告诉CPU中断号`0x80`。这一种告诉CPU中断号的方式叫做软件中断，而前面CPU提供的两种中断程序执行的机制中断和异常是硬件中断，这是三种给CPU提供中断号的方式。
![图 4](https://s2.loli.net/2022/06/20/C8vNAcyGDSb5BtP.png)  

那么收到中断后CPU就会去中断向量表中寻找对应的中断描述符，从中断描述符中找到段选择子和段内偏移地址（分段机制），然后段选择子去全局描述符表GDT中找段描述符取出段基址，通过段基址+段内偏移地址（分页机制），找到对应中断处理程序的地址，跳过去执行对应的中断程序。
![图 5](https://s2.loli.net/2022/06/20/dZzhDNUnBuY8Rf4.png)  
那至于这里提到的中断描述符表IDT,我们也在`进入Linux内核前的准备`一文中设置GDT这一段中提到过啦，IDT从idtr寄存器中可以找到，而idt这个表采用的是一个结构体数组的方式进行存储，对应的内容就是上面提到的段选择子和段内偏移地址啦。

```c
struct desc_struct idt_table[256] = { {0, 0}, };
struct desc_struct {
    unsigned long a,b;
};
```

![图 6](https://s2.loli.net/2022/06/20/5Qw4CxJqRuEIhrN.png)  

到这里我们应该把中断机制说清楚了，然后咱们再来看`trap_init()`这个函数的代码，这个函数主要的作用是初始化硬件中断。

```c
void trap_init(void)
{
 int i;

 set_trap_gate(0,&divide_error);
 set_trap_gate(1,&debug);
 set_trap_gate(2,&nmi);
 set_system_gate(3,&int3); /* int3-5 can be called from all */
 set_system_gate(4,&overflow);
 set_system_gate(5,&bounds);
 set_trap_gate(6,&invalid_op);
 set_trap_gate(7,&device_not_available);
 set_trap_gate(8,&double_fault);
 set_trap_gate(9,&coprocessor_segment_overrun);
 set_trap_gate(10,&invalid_TSS);
 set_trap_gate(11,&segment_not_present);
 set_trap_gate(12,&stack_segment);
 set_trap_gate(13,&general_protection);
 set_trap_gate(14,&page_fault);
 set_trap_gate(15,&reserved);
 set_trap_gate(16,&coprocessor_error);
 for (i=17;i<48;i++)
  set_trap_gate(i,&reserved);
 set_trap_gate(45,&irq13);
 outb_p(inb_p(0x21)&0xfb,0x21);
 outb(inb_p(0xA1)&0xdf,0xA1);
 set_trap_gate(39,&parallel_interrupt);
}
```

很多但是又非常重复，有很多`set_trap_gate()和set_system_gate()`，先来看看这两个函数。

```c
#define set_trap_gate(n,addr) \
 _set_gate(&idt[n],15,0,addr)

#define set_system_gate(n,addr) \
 _set_gate(&idt[n],15,3,addr)

#define _set_gate(gate_addr,type,dpl,addr) \
__asm__ ("movw %%dx,%%ax\n\t" \
 "movw %0,%%dx\n\t" \
 "movl %%eax,%1\n\t" \
 "movl %%edx,%2" \
 : \
 : "i" ((short) (0x8000+(dpl<<13)+(type<<8))), \
 "o" (*((char *) (gate_addr))), \
 "o" (*(4+(char *) (gate_addr))), \
 "d" ((char *) (addr)),"a" (0x00080000))
```

这俩函数都是封装了`_set_gate()`函数，先看两个函数对`_set_gate()`传参上的差异，一个是给`_set_gate()`传参`(&idt[n],15,0,addr)`，另一个传参`(&idt[n],15,3,addr)`，那第三个参数不同，这里0和3对应的意义是0表示内核态，而3是用户态。

而至于这个`_set_gate()`函数，这是使用的内联汇编实现的，简单来说它实现了在中断描述符表表IDT中插入中断描述符的效果。比如`set_trap_gate(0,&divide_error);`就是设置0号中断，给了`divide_error`这个除法异常处理程序的地址，这样当CPU执行一条除0的指令后，CPU会得到中断号0，之后去IDT找对应中断描述符，从而跳转执行`divide_error`这个中断程序。再比如`set_system_gate(5,&overflow);`，对应中断处理程序`overflow`是边界出错中断。

讲完一大片的`set_trap_gate()和set_system_gate()`后，接下来是一个for循环，这个语句给17到48号中断设置为`reserved`这个中断程序，是暂时性的，后面对应的硬件初始化时会覆盖这个中断程序。

![图 7](https://s2.loli.net/2022/06/20/av1bTnouPZsDdme.png)  

### 块设备初始化

接下来是`blk_dev_init()`这个函数

```c
/*
 * NR_REQUEST is the number of entries in the request-queue.
 * NOTE that writes may use only the low 2/3 of these: reads
 * take precedence.
 *
 * 32 seems to be a reasonable number: enough to get some benefit
 * from the elevator-mechanism, but not so much as to lock a lot of
 * buffers when they are in the queue. 64 seems to be too many (easily
 * long pauses in reading when heavy writing/syncing is going on)
 */
#define NR_REQUEST 32
/*
 * Ok, this is an expanded form so that we can use the same
 * request for paging requests when that is implemented. In
 * paging, 'bh' is NULL, and 'waiting' is used to wait for
 * read/write completion.
 */
struct request {
 int dev;  /* -1 if no request */
 int cmd;  /* READ or WRITE */
 int errors;
 unsigned long sector;
 unsigned long nr_sectors;
 char * buffer;
 struct task_struct * waiting;
 struct buffer_head * bh;
 struct request * next;
};

/*
 * The request-struct contains all necessary data
 * to load a nr of sectors into memory
 */
struct request request[NR_REQUEST];

void blk_dev_init(void)
{
 int i;

 for (i=0 ; i<NR_REQUEST ; i++) {
  request[i].dev = -1;
  request[i].next = NULL;
 }
}
```

`blk_dev_init(void)`这个函数就是遍历结构体数组request,并且给dev成员变量赋值-1,给next成员变量赋值NULL,不过我们还不懂request结构体长什么样又表示什么对吧，所以还得看request。

在request结构体中，结构体本身代表一次读写磁盘请求，其中dev表示设备号，-1就表示设备空闲，cmd表示命令，赋值READ/WRITE表示是读磁盘操作还是写磁盘操作，errors表示操作时产生的错误次数，sector表示起始扇区，nr_sectors表示扇区数，这俩变量表示了所要读取的块设备的哪几个扇区，buffer表示数据缓冲区，也就是读完磁盘之后数据放在内存中的位置，waiting是task_struct结构体变量，用于表示发起请求的进程，bh是缓冲区头指针，next指向下一个请求项。

![图 8](https://s2.loli.net/2022/06/24/KLpdwnImTcHJFsX.png)  

request结构体用于完整的描述一个读盘操作，在操作系统中我们用request数组进行存储。

![图 9](https://s2.loli.net/2022/06/24/6AVhzRrlpuTWZGt.png)  

所以`blk_dev_init()`这个函数主要做的就是初始化了用于描述读盘操作的request数组，作为之后块设备访问和内存缓冲区的桥梁。

### 字符设备初始化

接下来的`chr_dev_init()`看了看发现里面什么也没有执行，是个空的函数，网上搜了一下都说是字符设备初始化，应该是字符设备不需要做什么初始化操作，所以里面就什么也没有了

```c
void chr_dev_init(void)
{
}
```

### tty_init

再接下来的函数是`tty_init`，为什么叫tty呢，tty 是 Teletype 的缩写。通常使用 tty 来简称各种类型的终端设备。Teletype 是一种由 Teletype 公司生产的最早出现的终端设备，样子像是电传打字机。所以这里的tty是指两种字符设备，串行端口的串行终端设备和控制台设备（鼠标键盘），因此`tty_init`这个函数的作用就是初始化串行中断程序和串行接口1、2以及控制台终端。

```c
void tty_init(void)
{
 rs_init();
 con_init();
}
```

前面的`rs_init()`用于初始化串行中断程序和串行接口1、2后面的`con_init()`用于初始化控制台终端。

先来看`rs_init()`

```c
void rs_init(void)
{
 set_intr_gate(0x24,rs1_interrupt);
 set_intr_gate(0x23,rs2_interrupt);
 init(tty_table[1].read_q.data);
 init(tty_table[2].read_q.data);
 outb(inb_p(0x21)&0xE7,0x21);
}
```

这个方法是串行接口中断的开启，以及设置对应的串行接口终端程序。由于现在以及不怎么使用了，所以就不看了。

接下来是`con_init()`

```c
/*
 *  void con_init(void);
 *
 * This routine initalizes console interrupts, and does nothing
 * else. If you want the screen to clear, call tty_write with
 * the appropriate escape-sequece.
 *
 * Reads the information preserved by setup.s to determine the current display
 * type and sets everything accordingly.
 */
void con_init(void)
{
 register unsigned char a;
 char *display_desc = "????";
 char *display_ptr;

 video_num_columns = ORIG_VIDEO_COLS;
 video_size_row = video_num_columns * 2;
 video_num_lines = ORIG_VIDEO_LINES;
 video_page = ORIG_VIDEO_PAGE;
 video_erase_char = 0x0720;
 
 if (ORIG_VIDEO_MODE == 7)   /* Is this a monochrome display? */
 {
  video_mem_start = 0xb0000;
  video_port_reg = 0x3b4;
  video_port_val = 0x3b5;
  if ((ORIG_VIDEO_EGA_BX & 0xff) != 0x10)
  {
   video_type = VIDEO_TYPE_EGAM;
   video_mem_end = 0xb8000;
   display_desc = "EGAm";
  }
  else
  {
   video_type = VIDEO_TYPE_MDA;
   video_mem_end = 0xb2000;
   display_desc = "*MDA";
  }
 }
 else        /* If not, it is color. */
 {
  video_mem_start = 0xb8000;
  video_port_reg = 0x3d4;
  video_port_val = 0x3d5;
  if ((ORIG_VIDEO_EGA_BX & 0xff) != 0x10)
  {
   video_type = VIDEO_TYPE_EGAC;
   video_mem_end = 0xbc000;
   display_desc = "EGAc";
  }
  else
  {
   video_type = VIDEO_TYPE_CGA;
   video_mem_end = 0xba000;
   display_desc = "*CGA";
  }
 }

 /* Let the user known what kind of display driver we are using */
 
 display_ptr = ((char *)video_mem_start) + video_size_row - 8;
 while (*display_desc)
 {
  *display_ptr++ = *display_desc++;
  display_ptr++;
 }
 
 /* Initialize the variables used for scrolling (mostly EGA/VGA) */
 
 origin = video_mem_start;
 scr_end = video_mem_start + video_num_lines * video_size_row;
 top = 0;
 bottom = video_num_lines;

 gotoxy(ORIG_X,ORIG_Y);
 set_trap_gate(0x21,&keyboard_interrupt);
 outb_p(inb_p(0x21)&0xfd,0x21);
 a=inb_p(0x61);
 outb_p(a|0x80,0x61);
 outb(a,0x61);
}
```

这里为了应对不同显示模式来分配不同的变量值，写了很多的分支，看起来就比较长。

首先来说说字符是如何显示在屏幕上的

![图 10](https://s2.loli.net/2022/06/24/1GoDxSd9mAOi3Eg.png)  

内存中有一部分区域和显存映射，我们往这块区域上写数据就相当于写在显存中，而不管是独显像3060这种还是核显，都是有一块叫做显存的存储原件来存储这些图像信息，而显卡中的GPU能够将显存中的东西渲染到屏幕上，大概就是这样一个流程。

如果我们像这样用汇编往这块与显存映射的内存区域写数据，`mov [0xB8000],0x68`，也就是往0xB8000上这个位置写了值0x68,对于ascii码为h,就能够在屏幕上输出h。

![图 11](https://s2.loli.net/2022/06/24/PZyq1sEMIzbcpQL.png)  

具体来说，这片内存是每两个字节表示一个显示在屏幕上的字符，第一个是字符的编码，第二个是字符的颜色。
所以对于这样的汇编代码，就可以在屏幕上显示出`hello`的字样。

```asm
mov [0xB8000],'h'
mov [0xB8002],'e'
mov [0xB8004],'l'
mov [0xB8006],'l'
mov [0xB8008],'o'
```

因此我们就可以把之前冗长的代码简化为这样。

```c
#define ORIG_X          (*(unsigned char *)0x90000)
#define ORIG_Y          (*(unsigned char *)0x90001)
void con_init(void) {
    register unsigned char a;
    // 第一部分 获取显示模式相关信息
    video_num_columns = (((*(unsigned short *)0x90006) & 0xff00) >> 8);
    video_size_row = video_num_columns * 2;
    video_num_lines = 25;
    video_page = (*(unsigned short *)0x90004);
    video_erase_char = 0x0720;
    // 第二部分 显存映射的内存区域 
    video_mem_start = 0xb8000;
    video_port_reg  = 0x3d4;
    video_port_val  = 0x3d5;
    video_mem_end = 0xba000;
    // 第三部分 滚动屏幕操作时的信息
    origin  = video_mem_start;
    scr_end = video_mem_start + video_num_lines * video_size_row;
    top = 0;
    bottom  = video_num_lines;
    // 第四部分 定位光标并开启键盘中断
    gotoxy(ORIG_X, ORIG_Y);
    set_trap_gate(0x21,&keyboard_interrupt);
    outb_p(inb_p(0x21)&0xfd,0x21);
    a=inb_p(0x61);
    outb_p(a|0x80,0x61);
    outb(a,0x61);
}
```

![图 12](https://s2.loli.net/2022/06/27/ynv9CYWpZ4SbG5B.png)  
结合我们之前setup.s这个汇编程度写入内存中的对应的值来看，第一部分代码从0x90006处取值，获取了显示模式等相关信息；第二部分是显存映射的内存范围，我们现在假设是CGA类型的文本模式，对应的显存映射的内存范围是0xB8000到0xBA000；第三部分是设置滚动屏幕时需要的参数，定义顶行和底行，这里顶行为第一行，底行为最后一行；第四部分是把光标定位保存到0x90000,也就是光标位置处对应的内存地址，并且开启键盘中断。

开启键盘中断后，键盘上敲击按键会触发中断，中断程序就会读键盘码转换成ASCII码，然后写到光标处的内存地址，也就是对应的光标处的显存，使敲击的按键对应的字符显示在屏幕上。


### time_init

### sched_init

### buffer_init

### hd_init
