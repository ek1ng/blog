---
title: 运行linux前的准备工作是如何完成的
date: 2022-06-01 22:49:00
updated: 2022-06-02 21:12:00
tags: [os]
description: linux 0.11的源码阅读笔记(一)
---

最近看到这个github仓库[flash-linux0.11-talk](https://github.com/sunym1993/flash-linux0.11-talk),觉得还算是蛮有意思的，加上网络编程的课程又有抄写一段tcp协议实现代码或者交一篇linux内核源码阅读的笔记，还是比较讨厌这种低效率的抄写的所以就想写篇文章记录一下粗浅阅读源码后的大概了解，这个github仓库作者的文章我觉得写的还是不错的对于我这类小白而言，也比较有看得下去的动力。

## 运行os前的准备工作是如何完成的

### 开机

嗯如果问电脑是如何一步一步开始运行操作系统的，那么第一件事情当然是按下开机键啦。

### 加载启动区

在按下开机键后会，主板上写死的Bios程序会加载硬盘的启动区，也就是BIOS会将把硬盘上启动区代码这段512个Byte大小的数据，复制到内存中0x7c00这个位置并且跳转到这个位置开始执行这段程序。

那么对于linux0.11来看，这个512Byte大小的启动程序就是/boot目录下的bootsect.s这个文件。按下开机键后，它会被编译成二进制文件，并且被存放在硬盘中的0盘0道1扇区。值得一提的是，当硬盘中的0盘0道1扇区的512字节大小的数据最后两个字节为0x55和0xaa时，Bios就会认为这段512Byte的程序是启动区并且在启动的时候区加载这段程序。因此bootsect.s经过编译后的二进制程序就会被Bios认为是启动区，那么Bios就会和我们上面说的一样，将这段数据复制到内存中的0x7c00并且跳转到这个位置，开始执行这段程序。

我们来看一下bootsect.s中加载启动区部分的汇编代码

```asm
BOOTSEG  = 0x07c0
INITSEG  = 0x9000

start:
 mov ax,#BOOTSEG
 mov ds,ax
 mov ax,#INITSEG
 mov es,ax
 mov cx,#256
 sub si,si
 sub di,di
 rep
 movw
 jmpi go,INITSEG
go: mov ax,cs
 mov ds,ax
 mov es,ax
; put stack at 0x9ff00.
 mov ss,ax
 mov sp,#0xFF00  ; arbitrary value >>512

; load the setup-sectors directly after the bootblock.
; Note that 'es' is already set up.
```

这张16位CPU寄存器的图可以简单记忆一下，后面很多地方都需要用到。
![图 1](https://s2.loli.net/2022/06/02/x75ioKZgly3VFzE.png)  

这段汇编前两句的意义是将0x07c0这个值复制到ax寄存器，再将ax寄存器的值复制到ds寄存器。ds是个16位的段寄存器，具体表示数据段寄存器，在内存寻址时充当段基址的作用。简单的说就相当于一个偏移量，再之后的汇编中，比如`mov ax,[0x0001]`，实际上是对ds+0x0001的地址的值复制给了ax寄存器，这是一种基址寻址的方式，显然这里设置ds的值是为了我们之后通过基址，访问对应内存中的数据。

我们把为什么给ds赋值说清楚了，那ds的值为什么是0x07c0呢？之前我们不是说Bios将数据复制到内存中的0x7c00吗，这里为为什么刚好差了16倍呢？

我们需要先来解释实模式是什么，在实模式下，内存寻址方式和8086相同，由16位段寄存器的内容乘以16（10H）当做段基地址，加上16位偏移地址形成20位的物理地址。最大寻址空间1MB，最大分段64KB。可以使用32位指令。32位的x86 CPU用做高速的8086。在实模式下，所有的段都是可以读、写和可执行的。x86为了让自己在16位实模式下能访问到20位地址线，段基址会先左移4个2进制位，也就是一个16进制位，因此**0x07c0左移4位后为0x7c00**，也就是Bios将bootsect.s编译后的程序所存放到的位置，将ds设置为咱们启动程序所在的这个位置显然是很有必要的，因为我们要执行的操作系统的boot区的代码会被bios放在这段对应内存上，这段程序中的数据也需要相对于0x7c00这个地址去寻址使用，这样可以方便我们通过基址访问内存中的数据。

同样的方式，es寄存器的值变成了0x9000,cx寄存器的值变成了256(10进制)。

而`sub a,b`表示`a=a-b`,因此si，di寄存器的值变为0了这里。

到这里我们对不少寄存器的值进行了操作，但是也并没有看到什么实际的意义，因为这些赋值语句主要是为了下一句`rep movw`指令服务的，`rep movw`表示重复执行`movw`，而`movw`表示复制一个字(16 bit)，并且复制`cx`次，也就是256次，从`ds:si`复制到`es:di`，一次复制16bit大小的数据。

因此这句`rep movw`的作用就是把从内存地址0x7c00开始往后512Byte大小的数据，复制到0x90000处，那么现在启动区代码已经被挪到了0x90000。

接下来是一个跳转指令

```asm
jmpi go,0x9000
go: mov ax,cs
    mov ds,ax
    mov es,ax
    mov ss,ax
    mov sp,#0xFF00
```

jmpi是一个段间跳转指令，表示跳转到0x9000:go处执行，段基址：偏移地址的计算为基址左移4个2进制位+偏移地址，也就是跳转到0x90000+go这个内存地址上执行。那么0x90000我们很熟悉，我们将boot的代码从0x7c00移动到了0x90000,go又是什么意思呢？go是一个标签，最后编译成机器码的时候会被翻译成一个值，值的大小就是go这个标签在文件内的偏移地址。实际上啊，就是跳转到go后面`mov ax,cs`这句汇编所在的内存地址并且开始执行。假如`mov ax,cx`这行代码位于这个编译后二进制文件的0x08处，那么go就等于0x08,CPU就会跳转到0x90008开始执行。

简单总结一下，到这里为止的汇编代码的内容就是将一段512Byte大小的代码从硬盘上的启动区，移动到了内存的0x7c00,然后又被移动到0x90000,并且跳转到偏移go这个标签对应的地址处，开始执行下面这一串mov指令。

下面这些mov指令很容易看懂，把cs给ax,把ax给ds，es,ss,给sp赋值0xff00。

我们来理一理现在各个寄存器的值和它们对应的作用。
![图 1](https://s2.loli.net/2022/06/02/x75ioKZgly3VFzE.png)  
cs寄存器是代码段寄存器，CPU当前正在执行的代码在内存中的位置，就是cs:ip这组寄存器配合指向的。因为之前执行了`jmpi go,0x90000`这条段间跳转指令，所以cs寄存器的值就是0x90000,ip寄存器里的值是go这个标签的偏移地址，那么上面的三条mov指令把cs的值赋值给了ax,ds,es,ss,这些寄存器的值现在都是0x90000。

ds是数据段寄存器，一开始我们给他赋值了0x07c0,并且说主要的作用是方便通过基址访问对应数据，现在代码被移动到0x90000这段地址上面了，自然ds也应该赋值为0x90000。

es是扩展段寄存器，可以先不管。

ss为栈段寄存器，后面要配合栈基址寄存器sp表示此时的栈顶地址，ss和sp主要用于访问栈。此时sp寄存器被赋值为0xff00,所以目前栈顶地址是ss:sp所指向的之地0x90000+0x0FF00，栈顶地址0x9FF00,具体表现为栈段寄存器ss为0x9000，栈基址寄存器sp为0xFF00,栈是向下发展的，这个栈顶指针0x9FF00，远大于0x90000,栈向下发展很难撞见代码所在的为止，也就比较安全。

至此为止，我们就完成了加载启动区，也就是将这512Byte加载到内存中的工作咱们已经完成了。现在我们可以总结一下总共做了哪些事情。

首先来看start这块汇编代码，将启动区从硬盘移动到内存中0x7c00,又移动到了0x9FF00，然后go这一部分把数据段寄存器ds和代码段寄存器cs都被设置为了0x9000,为了方便跳转和内存访问。栈段寄存器ss和栈基址寄存器sp也设置了合理的取值。从CPU的角度看，访问内存就是访问代码、数据和栈这么三块地方，这里就是进行了一次内存规划。访问代码和访问数据的的规划方式就是分别给cs、ip和ds设置基址，访问栈的规划方式就是把栈顶指针只想了一个远离代码位置的地方。

### 加载setup.s

加载bootsect后，接下来我们会加载setup.s这个文件，当然加载这个文件的程序肯定也是写在bootsect.s里面的，让我们继续往下读bootsect.s。

```asm

load_setup:
 mov dx,#0x0000  ; drive 0, head 0
 mov cx,#0x0002  ; sector 2, track 0
 mov bx,#0x0200  ; address = 512, in INITSEG
 mov ax,#0x0200+SETUPLEN ; service 2, nr of sectors
 int 0x13   ; read it
 jnc ok_load_setup  ; ok - continue
 mov dx,#0x0000
 mov ax,#0x0000  ; reset the diskette
 int 0x13
 j load_setup

ok_load_setup:
 ...
```

先来说说`int`是什么，这里`int 0x13`表示发起13号中断，上面这几句mov语句赋值也是为这里发起13号中断做准备。这个中断发起后，CPU会通过这个中断号，去寻找对应的中断处理程序的入口地址，并跳转过去执行，逻辑上就想到那个与执行了一个函数，而0x13号中断是BIOS提前写好的一个读取磁盘相关功能的函数。

根据这段代码的注视，我们可以看出，这段汇编的作用是从硬盘的第二个扇区开始，将数据加载到内存0x90200处，一共加载4个扇区。如果复制成功，那么会跳转到标签`ok_load_setup`，如果失败，就会不断的重试这段代码。
![图 3](https://s2.loli.net/2022/06/02/SnB4yiCI2c6d8v1.png)  
因此到这里结束，加载setup.s的工作也就完成了，接下来就是加载操作系统内核。

### 加载内核

```asm
ok_load_setup:

; Get disk drive parameters, specifically nr of sectors/track

 mov dl,#0x00
 mov ax,#0x0800  ; AH=8 is get drive parameters
 int 0x13
 mov ch,#0x00
 seg cs
 mov sectors,cx
 mov ax,#INITSEG
 mov es,ax

; Print some inane message

 mov ah,#0x03  ; read cursor pos
 xor bh,bh
 int 0x10
 
 mov cx,#24
 mov bx,#0x0007  ; page 0, attribute 7 (normal)
 mov bp,#msg1
 mov ax,#0x1301  ; write string, move cursor
 int 0x10

; ok, we've written the message, now
; we want to load the system (at 0x10000)

 mov ax,#SYSSEG
 mov es,ax  ; segment of 0x010000
 call read_it
 call kill_motor

; After that we check which root-device to use. If the device is
; defined (!= 0), nothing is done and the given device is used.
; Otherwise, either /dev/PS0 (2,28) or /dev/at0 (2,8), depending
; on the number of sectors that the BIOS reports currently.

 seg cs
 mov ax,root_dev
 cmp ax,#0
 jne root_defined
 seg cs
 mov bx,sectors
 mov ax,#0x0208  ; /dev/ps0 - 1.2Mb
 cmp bx,#15
 je root_defined
 mov ax,#0x021c  ; /dev/PS0 - 1.44Mb
 cmp bx,#18
 je root_defined
```

这一段代码好长，但是注释写的非常详细。主要的代码不多，很多代码都是一些错误处理和友好的交互，可以不用管，我们来看看精简之后的逻辑。

```asm
ok_load_setup:
    ...
    mov ax,#0x1000
    mov es,ax       ; segment of 0x10000
    call read_it
    ...
    jmpi 0,0x9020
```

这里的作用是把硬盘的第6个扇区开始往后的240个扇区，加载到内存0x10000处，然后通过段间跳转指令jmpi 0,0x90202,跳转到0x90200处，就是硬盘第二个扇区开始处的内容。
![图 4](https://s2.loli.net/2022/06/02/GweZx8lH3W7otRS.png)  

这里我们先停一停，先说一下操作系统的编译过程。

1. 编译bootsect.s放在硬盘的1扇区
2. 编译setup.s放在硬盘的2-5扇区
3. 把剩下的代码已head.s为开头编译成system放在硬盘的随后240个扇区。

![图 5](https://s2.loli.net/2022/06/03/fVeQ49zbWlNHE7A.png)  

所以我们即将跳转到内存中的0x90200处的代码，就是从硬盘第二个扇区开始处加载到内存的setup.s。

来看看setup.s

```asm
start:
    mov ax,#0x9000  ; this is done in bootsect already, but...
    mov ds,ax
    mov ah,#0x03    ; read cursor pos
    xor bh,bh
    int 0x10        ; save it in known place, con_init fetches
    mov [0],dx      ; it from 0x90000.
```

这里的0x10终端也是触发BIOS提供的显示服务中断处理程序，而ah寄存器被赋值为0x03表示限时服务里具体的读取光标位置功能。这里`int 0x10`中断程序执行完毕并返回时，dx寄存器里的值表示光标的位置，dx寄存器的高八位dh存储行号，低八位dl存储列号。

由于计算机在加电自检后会自动初始化到文字模式，这时屏幕可以显示25行 x 80列的字符数量。
那`mov [0],dx`就是把这个光标位置存在在`[0]`这个内存地址处，也就是偏移地址加上0,0x90000,这里存放了光标的位置，以便之后在初始化控制台的时候用到。

再是接下来的几行代码，和之前的逻辑一样，都是从BIOS终端获取信息，然后存储在内存中的某个位置。

```
; Get memory size (extended mem, kB)

 mov ah,#0x88
 int 0x15
 mov [2],ax

; Get video-card data:

 mov ah,#0x0f
 int 0x10
 mov [4],bx  ; bh = display page
 mov [6],ax  ; al = video mode, ah = window width

; check for EGA/VGA and some config parameters

 mov ah,#0x12
 mov bl,#0x10
 int 0x10
 mov [8],ax
 mov [10],bx
 mov [12],cx

; Get hd0 data

 mov ax,#0x0000
 mov ds,ax
 lds si,[4*0x41]
 mov ax,#INITSEG
 mov es,ax
 mov di,#0x0080
 mov cx,#0x10
 rep
 movsb

; Get hd1 data

 mov ax,#0x0000
 mov ds,ax
 lds si,[4*0x46]
 mov ax,#INITSEG
 mov es,ax
 mov di,#0x0090
 mov cx,#0x10
 rep
 movsb

; Check that there IS a hd1 :-)

 mov ax,#0x01500
 mov dl,#0x81
 int 0x13
 jc no_disk1
 cmp ah,#3
 je is_disk1
no_disk1:
 mov ax,#INITSEG
 mov es,ax
 mov di,#0x0090
 mov cx,#0x10
 mov ax,#0x00
 rep
 stosb
is_disk1:
```

看注释就知道意思，这里不必细究
接着往下看

```asm
; now we want to move to protected mode ...

 cli   ; no interrupts allowed ;
```

这里cli表示关闭中断的意思，因为后面我们要把原本是BIOS写好的中断向量表给覆盖掉，写上我们自己的终端向量表，所以这时候不允许终端进来。

```asm
; first we move the system to it's rightful place

 mov ax,#0x0000
 cld   ; 'direction'=0, movs moves forward
do_move:
 mov es,ax  ; destination segment
 add ax,#0x1000
 cmp ax,#0x9000
 jz end_move
 mov ds,ax  ; source segment
 sub di,di
 sub si,si
 mov  cx,#0x8000
 rep
 movsw
 jmp do_move

; then we load the segment descriptors

end_move:
 ...
```

这里的`rep movsw`我们前面用过，把0x7c00移动到0x90000时就是用的这个指令，所以和之前一样，这里把内存地址0x10000处开始往后一知道0x90000的内容，复制到内存最开始的0位置。

![图 1](https://s2.loli.net/2022/06/03/QTClZhED4Yq9BPd.png)  

我们来看看现在的内存布局是怎么样的

![图 2](https://s2.loli.net/2022/06/03/tAHQhsufWlMwyNv.png)  

栈顶地址仍然是0x9FF00,0x90000开始往上的位置，原来是bootsect和setup程序的代码，现bootsect的一部分代码在已经被操作系统为了记录内存、硬盘、显卡等一些临时存放的数据覆盖了一部分，内存最开始的0到0x80000这512K被system模块给占用了，这个system模块就是除了bootsect和setup之外的全部程序链接在一起的结果，可以理解为操作系统的全部。

### 设置GDT

接下来需要进行模式的转换，需要从现在的16位的实模式转变为之后32位的保护模式。

由于x86的历史包袱，现在的CPU几乎都是支持32位甚至64位模式了，很少有停留在16位实模式下的CPU,所以我们需要写一段模式转换的代码，这一部分的功能就是靠全局描述表符GDT来实现。

先让我们回忆一下在加载启动区时，为什么要给ds赋值0x07c0但是实际在内存中的基址是0x7c00,我们当时说，这是为了x86能够在16位实模式下访问20根地址线，会把给ds寄存器的值左移4位得到基址再加偏移地址来进行计算。

而当CPU切换到32位保护模式后，内存地址的计算方式还会改变，首先ds寄存器里面的值在实模式下称为段基址，在保护模式下叫做段选择子，段选择子里存储着段描述符的索引。
![图 3](https://s2.loli.net/2022/06/04/B8CE7UeJ4pzb2du.png)  
通过段描述符索引，可以从一个叫做全局描述表符（GDT）的东西中找到一个32位大小的段描述符，段描述符里面存储着16位的段基址。
![图 4](https://s2.loli.net/2022/06/04/vsm79CexZno5kcW.png)  
![图 6](https://s2.loli.net/2022/06/04/UbvwIDKX3qx7WSu.png)  
在保护模式下，段寄存器(ex ds,ss,cs..)里面存储的是段选择子，段选择子去全局描述符表GDT中寻找段描述符，从中取出段基址，段基址加上偏移地址就是实际访问的物理地址。

接下来我们详细说一说GDT是如何被设置的

首先GDT的**地址**被存储在一个叫gdtr寄存器中，这是寄存器的结构。
![图 7](https://s2.loli.net/2022/06/04/dibOte3JKYyEuNF.png)  
我们结合代码来看看gdtr寄存器是如何使用的。

继续看setup.s endmove后的内容。

```asm
end_move:
 mov ax,#SETUPSEG ; right, forgot this at first. didn't work :-)
 mov ds,ax
 lidt idt_48  ; load idt with 0,0
 lgdt gdt_48  ; load gdt with whatever appropriate

; that was painless, now we enable A20

 call empty_8042
 mov al,#0xD1  ; command write
 out #0x64,al
 call empty_8042
 mov al,#0xDF  ; A20 on
 out #0x60,al
 call empty_8042
```
这里`lidt idt_48`咱们先不管，先看下面这句`lgdt gdt_48`，`lgdt gdt_48`把值`gdt_48`放在gdtr寄存器中，`gdt_48`在这里是个标签。

```asm
gdt_48:
 .word 0x800  ; gdt limit=2048, 256 GDT entries
 .word 512+gdt,0x9 ; gdt base = 0X9xxxx
```

这个标签表示了一个48位的数据，其中高32位存储全局描述符表gdt的**内存地址**`0x90200 + gdt`，在这里加上的这个gdt是个标签，表示在本文件内的偏移量，而本文件是setup.s,编译后放在0x90200这个内存地址上，所以全局描述符表gdt的内存地址是用`0x90200 + gdt`描述的。

![图 8](https://s2.loli.net/2022/06/04/17L9YQRmOrsaiFh.png)  

我们把`gdt_48`赋值给gdtr寄存器，`gdt_48`中存储的又恰好是gdt表的地址，那么我们就可以通过gdtr寄存器找到gdt了，现在让我们来看一下gdt标签的内容是啥。

```asm
gdt:
 .word 0,0,0,0  ; dummy

 .word 0x07FF  ; 8Mb - limit=2047 (2048*4096=8Mb)
 .word 0x0000  ; base address=0
 .word 0x9A00  ; code read/exec
 .word 0x00C0  ; granularity=4096, 386

 .word 0x07FF  ; 8Mb - limit=2047 (2048*4096=8Mb)
 .word 0x0000  ; base address=0
 .word 0x9200  ; data read/write
 .word 0x00C0  ; granularity=4096, 386
```

根据前面的段描述符格式，我们可以此时看出全局描述符表gdt有三个段描述符，第一个为空，第二个为代码段描述符（type=code），第三个是数据段描述符（type=data），第二个和第三个段描述符的段基址都是0,也就是之后在逻辑地址转换物理地址的时候，通过段选择子查找到无论是代码段还是数据段，取出的段基址都是0,此时物理地址等于逻辑地址中的偏移地址。

![图 10](https://s2.loli.net/2022/06/04/oKnxhCmFqYAtSa1.png)  

现在我们再来看看目前的内存布局

![图 11](https://s2.loli.net/2022/06/04/Dynvk9gRxhOQ1Nm.png)  

会发现多了个idtr寄存器对吧，我们之前还没有对这个寄存器进行解释，所以在这补充一下，idt是中断描述符表。而idtr是中断描述符表寄存器，使用方法和ghtr一样。全局描述符表是让段选择子去找段描述符用的，而中断描述符表使用来在发生中断时，CPU拿中断号去中断描述符表中寻找终端处理程序的地址，找到后就跳到想要的中断程序中去执行。

### 进入保护模式

设置完GDT后，接下来就要从16位实模式切换到32位保护模式啦。


### 分页机制

### 跳转到内核
