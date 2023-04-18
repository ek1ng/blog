---
title: 进入Linux内核前的准备
date: 2022-06-01 22:49:00
updated: 2022-06-13 22:20:00
tags: [OS,linux]
description: linux 0.11的源码阅读笔记(一)
category: Computer Science
---

最近看到这个github仓库[flash-linux0.11-talk](https://github.com/sunym1993/flash-linux0.11-talk),觉得还算是蛮有意思的，加上网络编程的课程又有抄写一段tcp协议实现代码或者交一篇linux内核源码阅读的笔记，还是比较讨厌这种低效率的抄写的所以就想写篇文章记录一下粗浅阅读源码后的大概了解，这个github仓库作者的文章我觉得写的还是不错的对于我这类小白而言，也比较有看得下去的动力。

## 进入linux内核前的准备

### 开机

如果问电脑是如何一步一步开始运行操作系统的，那么第一件事情当然是按下开机键啦。

### 加载启动区

在按下开机键后，主板上写死的Bios程序会加载硬盘的启动区，也就是BIOS会将把硬盘上启动区代码这段512个Byte大小的数据，复制到内存中0x7c00这个位置并且跳转到这个位置开始执行这段程序。

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
![图 2](https://s2.loli.net/2022/06/02/x75ioKZgly3VFzE.png)  
**cs寄存器是代码段寄存器，cs:ip表示了CPU当前正在执行的代码在内存中的位置，也就是我们接下来执行什么代码**。因为之前执行了`jmpi go,0x90000`这条段间跳转指令，所以cs寄存器的值就是0x90000,ip寄存器里的值是go这个标签的偏移地址，那么上面的三条mov指令把cs的值赋值给了ax,ds,es,ss,这些寄存器的值现在都是0x90000。

**ds是数据段寄存器，ds:xxx 表示了我们要访问哪里的数据**。一开始我们给他赋值了0x07c0,并且说主要的作用是方便通过基址访问对应数据，现在代码被移动到0x90000这段地址上面了，自然ds也应该赋值为0x90000。

es是扩展段寄存器，可以先不管。

**ss为栈段寄存器，配合栈基址寄存器sp表示栈顶地址**。此时sp寄存器被赋值为0xff00,所以目前栈顶地址是ss:sp所指向的之地0x90000+0x0FF00，栈顶地址0x9FF00,具体表现为栈段寄存器ss为0x9000，栈基址寄存器sp为0xFF00,栈是向下发展的，这个栈顶指针0x9FF00，远大于0x90000,栈向下发展很难撞见代码所在的为止，也就比较安全。

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

```asm
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

![图 6](https://s2.loli.net/2022/06/03/QTClZhED4Yq9BPd.png)  

我们来看看现在的内存布局是怎么样的

![图 7](https://s2.loli.net/2022/06/03/tAHQhsufWlMwyNv.png)  

栈顶地址仍然是0x9FF00,0x90000开始往上的位置，原来是bootsect和setup程序的代码，现bootsect的一部分代码在已经被操作系统为了记录内存、硬盘、显卡等一些临时存放的数据覆盖了一部分，内存最开始的0到0x80000这512K被system模块给占用了，这个system模块就是除了bootsect和setup之外的全部程序链接在一起的结果，可以理解为操作系统的全部。

### 设置GDT

接下来需要进行模式的转换，需要从现在的16位的实模式转变为之后32位的保护模式。

由于x86的历史包袱，现在的CPU几乎都是支持32位甚至64位模式了，很少有停留在16位实模式下的CPU,所以我们需要写一段模式转换的代码，这一部分的功能就是靠全局描述表符GDT来实现。

先让我们回忆一下在加载启动区时，为什么要给ds赋值0x07c0但是实际在内存中的基址是0x7c00,我们当时说，这是为了x86能够在16位实模式下访问20根地址线，会把给ds寄存器的值左移4位得到基址再加偏移地址来进行计算。

而当CPU切换到32位保护模式后，内存地址的计算方式还会改变，首先ds寄存器里面的值在实模式下称为段基址，在保护模式下叫做段选择子，段选择子里存储着段描述符的索引。
![图 8](https://s2.loli.net/2022/06/04/B8CE7UeJ4pzb2du.png)  
通过段描述符索引，可以从一个叫做全局描述表符（GDT）的东西中找到一个32位大小的段描述符，段描述符里面存储着16位的段基址。
![图 9](https://s2.loli.net/2022/06/04/vsm79CexZno5kcW.png)  
![图 10](https://s2.loli.net/2022/06/04/UbvwIDKX3qx7WSu.png)  
在保护模式下，段寄存器(ex ds,ss,cs..)里面存储的是段选择子，段选择子去全局描述符表GDT中寻找段描述符，从中取出段基址，段基址加上偏移地址就是实际访问的物理地址。

所以到这里明白GDT的作用了么？就是为了从16位实模式切换到32位保护模式后，能从GDT中找到段描述符，拼凑成最终的物理地址。具体的例子我们后面转换到32位保护模式后，会再举实际例子来解释的，这里先理清楚思路就好。

接下来我们详细说一说GDT是如何被设置的

首先GDT的**地址**被存储在一个叫gdtr寄存器中，这是寄存器的结构。
![图 11](https://s2.loli.net/2022/06/04/dibOte3JKYyEuNF.png)  
我们结合代码来看看如何设置GDT

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

![图 12](https://s2.loli.net/2022/06/04/17L9YQRmOrsaiFh.png)  

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

![图 13](https://s2.loli.net/2022/06/04/oKnxhCmFqYAtSa1.png)  

现在我们再来看看目前的内存布局

![图 14](https://s2.loli.net/2022/06/04/Dynvk9gRxhOQ1Nm.png)  

会发现多了个idtr寄存器对吧，我们之前还没有对这个寄存器进行解释，所以在这补充一下，idt是中断描述符表。而idtr是中断描述符表寄存器，使用方法和ghtr一样。全局描述符表是让段选择子去找段描述符用的，而中断描述符表使用来在发生中断时，CPU拿中断号去中断描述符表中寻找终端处理程序的地址，找到后就跳到想要的中断程序中去执行。

### 进入保护模式

设置完GDT后，接下来就要从16位实模式切换到32位保护模式啦。
我们接着往下看setup.s

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

; well, that went ok, I hope. Now we have to reprogram the interrupts :-(
; we put them right after the intel-reserved hardware interrupts, at
; int 0x20-0x2F. There they won't mess up anything. Sadly IBM really
; messed this up with the original PC, and they haven't been able to
; rectify it afterwards. Thus the bios puts interrupts at 0x08-0x0f,
; which is used for the internal hardware interrupts as well. We just
; have to reprogram the 8259's, and it isn't fun.

 mov al,#0x11  ; initialization sequence
 out #0x20,al  ; send it to 8259A-1
 .word 0x00eb,0x00eb  ; jmp $+2, jmp $+2
 out #0xA0,al  ; and to 8259A-2
 .word 0x00eb,0x00eb
 mov al,#0x20  ; start of hardware int's (0x20)
 out #0x21,al
 .word 0x00eb,0x00eb
 mov al,#0x28  ; start of hardware int's 2 (0x28)
 out #0xA1,al
 .word 0x00eb,0x00eb
 mov al,#0x04  ; 8259-1 is master
 out #0x21,al
 .word 0x00eb,0x00eb
 mov al,#0x02  ; 8259-2 is slave
 out #0xA1,al
 .word 0x00eb,0x00eb
 mov al,#0x01  ; 8086 mode for both
 out #0x21,al
 .word 0x00eb,0x00eb
 out #0xA1,al
 .word 0x00eb,0x00eb
 mov al,#0xFF  ; mask off all interrupts for now
 out #0x21,al
 .word 0x00eb,0x00eb
 out #0xA1,al


; well, that certainly wasn't fun :-(. Hopefully it works, and we don't
; need no steenking BIOS anyway (except for the initial loading :-).
; The BIOS-routine wants lots of unnecessary data, and it's less
; "interesting" anyway. This is how REAL programmers do it.
;
; Well, now's the time to actually move into protected mode. To make
; things as simple as possible, we do no register set-up or anything,
; we let the gnu-compiled 32-bit programs do that. We just jump to
; absolute address 0x00000, in 32-bit protected mode.

 mov ax,#0x0001 ; protected mode (PE) bit
 lmsw ax  ; This is it;
 jmpi 0,8  ; jmp offset 0 of segment 8 (cs)
```

还是endmove,我们之前看到通过idt_48和gdt_48设置了中断描述符表和全局描述符表，接下来执行`call empty_8042`，我们来看看这个标签对应的汇编

```asm
; This routine checks that the keyboard command queue is empty
; No timeout is used - if this hangs there is something wrong with
; the machine, and we probably couldn't proceed anyway.
empty_8042:
 .word 0x00eb,0x00eb
 in al,#0x64 ; 8042 status port
 test al,#2  ; is input buffer full?
 jnz empty_8042 ; yes - loop
 ret
```

根据注释我们也可以看出，此例程检查键盘命令队列是否为空，如果超时那么挂起例程，则说明机器有问题可能无法继续。

继续往下看，根据注释结合汇编，看得出这里几句是打开A20地址线，突破地址信号线20位的宽度，变成32位可用。20位这个问题我们之前多次提到，这是因为8086CPU只有20位的地址线，但是从CPU进入32位时代后，要兼容以前16位CPU只能用20位地址线的模式，因此如果不开启，那么即便你有32位的地址线，默认只使用低20位。

接下来有很长一片注释，看起来非常唬人，其实用处不大。大概内容就是这里是对可编程中断控制器8259芯片进行的编程，因为中断号是不能冲突的，Intel把0到0x19号中断都作为保留中断，比如0号中断就规定为除零一场，软件自定义的中断都应在放在这之后，但是IBM在原PC机中和保留中断号发生了冲突也没有处理，因此这里不得不对其重新编程。

endmove中这里最后三句非常重要，我们拎出来单独看看。

```asm
 mov ax,#0x0001 ; protected mode (PE) bit
 lmsw ax  ; This is it;
 jmpi 0,8  ; jmp offset 0 of segment 8 (cs)
```

模式这个状态字保存在机器状态字寄存器cr0中，当我们准备好切换到保护模式时，前两行汇编做的就是将cr0这个寄存器的位0置1,我们就从实模式切换到保护模式了。

![图 15](https://s2.loli.net/2022/06/04/EefroIPzC4NcXK3.png)  

接下来是`jump 0,8`，8表示代码段寄存器cs的值，0表示偏移地址，需要注意我们现在是保护模式下的内存寻址方式。

![图 16](https://s2.loli.net/2022/06/04/QeUIzr3ZDytsdc6.png)  
8用二进制表示为`00000,0000,0000,1000`,所以描述符索引的值是1,也就是我们要去gdt中找第一项段描述符，那我们看看我们当时设置gdt时里面的内容是啥。

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

第0项是空值，第一项是代码段描述符，可读可执行，第二项是数据段描述符，是可读可写段，段基址都是0.所以我们这里取的第一项段描述符就是代码段描述符，它现在的段基址是0,我们加的偏移也是0,所以这个`jump 0,8`会跳转到内存地址的0x0处开始执行。

零地址处里面是什么呢？再来看看我们现在的内存布局图，和之前加载完内核后一样对吧设置GDT和切换模式肯定是没改变什么位置存放了啥的。

![图 17](https://s2.loli.net/2022/06/04/tfWGPJiDwusjqTH.png)  
所以0地址处存放的是system，那system是怎么来的呢，我们在加载内核中提到过操作系统的编译，system内核的代码的编译是由Makefile文件主导的，我们来看一下Makefile中的关键部分,发现是用head.s和main.c以及其他各模块的操作系统代码编译出来的结果。

```makefile
tools/system: boot/head.o init/main.o \
 $(ARCHIVES) $(DRIVERS) $(MATH) $(LIBS)
 $(LD) $(LDFLAGS) boot/head.o init/main.o \
 $(ARCHIVES) \
 $(DRIVERS) \
 $(MATH) \
 $(LIBS) \
 -o tools/system > System.map
```

所以我们接下来应该看看head.s中的内容了，head.s这个文件是为了进入用c编写的main.c做的准备

```asm
_pg_dir:
startup_32:
 movl $0x10,%eax
 mov %ax,%ds
 mov %ax,%es
 mov %ax,%fs
 mov %ax,%gs
 lss _stack_start,%esp
 call setup_idt
 call setup_gdt
 movl $0x10,%eax  ; reload all the segment registers
 mov %ax,%ds  ; after changing gdt. CS was already
 mov %ax,%es  ; reloaded in 'setup_gdt'
 mov %ax,%fs
 mov %ax,%gs
 lss _stack_start,%esp
 xorl %eax,%eax
1: incl %eax  ; check that A20 really IS enabled
 movl %eax,0x000000 ; loop forever if it isn't
 cmpl %eax,0x100000
 je 1b
/*
 * NOTE! 486 should set bit 16, to check for write-protect in supervisor
 * mode. Then it would be unnecessary with the "verify_area()"-calls.
 * 486 users probably want to set the NE (#5) bit also, so as to use
 * int 16 for math errors.
 */
 movl %cr0,%eax  # check math chip
 andl $0x80000011,%eax # Save PG,PE,ET
/* "orl $0x10020,%eax" here for 486 might be good */
 orl $2,%eax  # set MP
 movl %eax,%cr0
 call check_x87
 jmp after_page_tables
```

pg_dir这个标签表示页目录，之后设置分页机制时，页目录会存放在这个位置。

接下来的`mov`操作给ds、es、fs、gs这几个段寄存器复制0x10,根据段描述符的结构，0x10表示GDT中的第二个段描述符，也就是数据段描述符。

最后`lss`改变了栈顶指针的位置，之前栈顶指针指向的位置是`0x9FF00`，`lss`指令让`ss:esp`这个栈顶指针指向了`_stack_start`这个标号的位置。`_stack_start`标号在`sched.c`中，让我们来看看。

```asm
struct {
 long * a;
 short b;
 } stack_start = { & user_stack [PAGE_SIZE>>2] , 0x10 };
```

首先`stack_start`中高位8Byte是0x10,会赋值给ss,低位16Byte是`user_stack`数组的最后一个元素的地址也就是栈顶地址，会赋值给esp寄存器。ss被赋值0x10,按照保护模式下的段选择子寻址，指向GDT中的第二个段描述符，数据段描述符，段基址为0。esp被赋值为栈顶地址，栈顶地址指向`user_stack + 0`。

继续看head.s,两个`call`语句设置了中断描述符表IDT和全局描述符表GDT，然后后面又是重新执行了一遍前面的代码，为什么要重新设置段寄存器的值呢？因为上面修改了GDT,要重新设置才能生效，那么为什么要设置IDT和GDT呢？

首先IDT我们之前没设置过具体值，只是告诉了idtr,idt在哪，所以我们后面要用的话，给IDT设置值是理所应当的。先来看看给IDT设置成了什么

```asm
setup_idt:
 lea ignore_int,%edx
 movl $0x00080000,%eax
 movw %dx,%ax  /* selector = 0x0008 = cs */
 movw $0x8E00,%dx /* interrupt gate - dpl=0, present */

 lea _idt,%edi
 mov $256,%ecx
rp_sidt:
 movl %eax,(%edi)
 movl %edx,4(%edi)
 addl $8,%edi
 dec %ecx
 jne rp_sidt
 lidt idt_descr
 ret

idt_descr:
 .word 256*8-1  # idt contains 256 entries
 .long _idt
.align 2
.word 0

_idt: .fill 256,8,0  # idt is uninitialized
```

和GDT类似，中断描述符表idt里面存储的是中断描述符，每个中断号对应一个中断描述符，而中断描述符里面存储着主要是中断程序的地址，这样CPU就可以根据中断号寻找到对应的中断程序并且执行。

这段程序的作用就是设置了256个中断描述符，并且让每个中断描述符中的中断程序例程都指向一个`ignore_int`的函数地址，相当于是初始化了idt,`ignore_int`是默认的中断处理程序，之后会被对应的中断程序覆盖，不过现在还没覆盖过去，所以现在任何中断都指向`ignore_int`，都还是没法用的。

这里对idt的设置就讲完了，接下来也对gdt做了设置，来看看设置后的gdt。

```asm
_gdt: .quad 0x0000000000000000 /* NULL descriptor */
 .quad 0x00c09a0000000fff /* 16Mb */
 .quad 0x00c0920000000fff /* 16Mb */
 .quad 0x0000000000000000 /* TEMPORARY - don't use */
 .fill 252,8,0   /* space for LDT's and TSS's etc */
```

这和我们先前设置的gdt一样，也是代码描述符和数据段描述符，第四项没有用，最后留了252项的空间，这些空间之后会用来放置人物状态描述符TSS和局部描述符LDT。
![图 18](https://s2.loli.net/2022/06/05/JpNdBYRZzPOfGeD.png)  
那既然一模一样，为什么还需要重新设置gdt呢？因为原来设置的gdt在setup程序中，之后这块地址会被缓冲区覆盖掉，因此需要在head程序中重新设置一遍，head中的内存区域就不会被其他程序覆盖啦。

![图 19](https://s2.loli.net/2022/06/05/sZrAEOW7vcNedam.png)  

继续往下看head.s,发现下一句是`jmp after_page_tables`，没错，接下来就是分页机制啦

### 分页机制

我们先补充解释一下Intel体系结构的内存管理机制，分段和分页。

分段机制就是我们之前几个标题中讨论的，目的是为每个程序或者任务提供单独的代码段cs,数据段ds,栈段ss,使其不会互相干扰。分段机制在Intel的保护模式下是必须开启的。

分页机制就是本标题中讲到的内容，开机后分页机制默认是关闭状态，我们手动开启后并且设置页目录表PDE和页表PTE,来达到按需使用物理内存，并且对于多任务的内存使用进行隔离。

我们先来具体介绍一下什么是分页机制，之前我们说过在保护模式下，我们在代码中的逻辑地址要经过分段机制的转换才能变成物理地址。

![图 20](https://s2.loli.net/2022/06/09/WbjqJmkY2ZBFhXL.png)  

这是在没有开启分页机制的时候，我们只需要这一步转换，但是开启分页机制后寻址方式会有一些变化。

![图 21](https://s2.loli.net/2022/06/09/XW3vSpndtBuVRJQ.png)

可以看出分页后，仍然是通过分段机制转换，只不过转换得到的我们称为线性地址，再根据分页机制转换后，得到物理地址。

我们来看看分段机制是如何进行地址转换的，假设我们根据分段机制得到15M这个线性地址，那么用二进制表示为`0000000011_0100000000_000000000000`。

![图 22](https://s2.loli.net/2022/06/09/YFs9ympHDuGC78h.png)  

CPU在得到线性地址后，将线性地址拆分为`10b:10b:12b`，高10位表示**页目录表**中的**页目录项**，页目录项的值拼接中间10位地址后，去**页表**中找对应的**页表项**，再拼接上低12位偏移地址，就可以在页表项中访问对应的物理地址。这就像你去图书馆找书，你先找到对应的书架（页目录项）,再找对应的第几层（页表项），再找是这层的第几本书（物理地址）。这个寻址的过程由计算机的一个硬件叫内存管理单元（MMU）完成，它负责将虚拟地址转换为物理地址。

这种页表方案叫做二级页表，第一级叫页目录表PDE,其中有很多页目录项，第二级叫做页表PTE。

![图 23](https://s2.loli.net/2022/06/09/suNywOzajI91UQ6.png)  

之后我们开启分页机制的开关，就像开启保护模式一样，修改cr0寄存器中的对应位即可。

![图 24](https://s2.loli.net/2022/06/10/8iD1pHzIeqCkSQL.png)  

开启分页机制后，MMU就可以进行分页的转换了，在这之后的指令中使用的内存地址就需要先经过分段机制的转换再经过分页机制的转换变成物理地址。

接下来我们继续head.s下面的`after_page_tables`这个标签

```asm
after_page_tables:
 pushl $0  # These are the parameters to main :-)
 pushl $0
 pushl $0
 pushl $L6  # return address for main, if it decides to.
 pushl $_main
 jmp setup_paging
L6:
 jmp L6   # main should never return here, but
    # just in case, we know what happens.
```

`after_page_tables`这个标签中，前面的`pushl $_main`把 main 函数的地址压栈了，那最终跳转到这个 main.c 里的 main 函数，一定和这个压栈有关。不过在进入main之前会先执行`jmp setup_paging`,这个标签是当然是开启分页机制啦。

```asm
setup_paging:
 movl $1024*5,%ecx  /* 5 pages - pg_dir+4 page tables */
 xorl %eax,%eax
 xorl %edi,%edi   /* pg_dir is at 0x000 */
 cld;rep;stosl
 movl $pg0+7,_pg_dir  /* set present bit/user r/w */
 movl $pg1+7,_pg_dir+4  /*  --------- " " --------- */
 movl $pg2+7,_pg_dir+8  /*  --------- " " --------- */
 movl $pg3+7,_pg_dir+12  /*  --------- " " --------- */
 movl $pg3+4092,%edi
 movl $0xfff007,%eax  /*  16Mb - 4096 + 7 (r/w user,p) */
 std
1: stosl   /* fill pages backwards - more efficient :-) */
 subl $0x1000,%eax
 jge 1b
 xorl %eax,%eax  /* pg_dir is at 0x0000 */
 movl %eax,%cr3  /* cr3 - page directory start */
 movl %cr0,%eax
 orl $0x80000000,%eax
 movl %eax,%cr0  /* set paging (PG) bit */
 ret   /* this also flushes prefetch-queue */
```

我们需要提一下，linux-0.11认为总共可以使用的内存不超过**16M**,即最大地址空间为0xFFFFFF。

根据当前的页目录表和页表机制，1个页目录表最多包含1024个页目录项，1个页表最多包含1024个页表项，1页为4KB（12位偏移地址），所以我们为了表示16M的地址空间，需要用用1个页目录表+4个页表来表示即4（页表数）\*1024（页表项数）\*4KB（页的大小） = 16 MB。

前面的mov语句表示，页目录表的前4个页目录项，分别指向4个页表。比如页目录项中的第一项 `[eax]` 被赋值为 pg0+7，也就是 0x00001007，根据页目录项的格式，表示页表地址为 0x1000，页属性为 0x07 表示改页存在、用户可读写。后面几行表示，填充 4 个页表的每一项，一共 4\*1024=4096 项，依次映射到内存的前 16MB 空间。也就是上面我们将分页机制是什么的时候配的图。

![图 25](https://s2.loli.net/2022/06/12/EDTYg41CiNeS6GP.png)  
现在只有四个页目录项，也就是将前 16M 的线性地址空间，与 16M 的物理地址空间一一对应起来了。
![图 26](https://s2.loli.net/2022/06/12/Smy9cxG4f5MW3Ll.png)  

因此上面这段代码最终效果是，将页目录表放在内存地址的最开始，在进入保护模式这一章中我们说`_pg_dir`标签表示页目录存放的位置，之后紧挨着我们初始化的这个页目录表，放置了四个页表，最终将页目录表和页表填写好数值，来覆盖整个16MB的内存，随后开启分页模式。

![图 27](https://s2.loli.net/2022/06/10/XIxtAhKBm9pNUYL.png)  

这些页目录表和页表放到了整个内存布局中最开头的位置，覆盖了开头的 system 代码了，不过被覆盖的 system 代码已经执行过了，所以无所谓。同时，我们也需要通过一个寄存器告诉CPU页表的位置,也就是代码中的这两句，我们拎出来看一下

```asm
xorl %eax,%eax  /* pg_dir is at 0x0000 */
movl %eax,%cr3  /* cr3 - page directory start */
```

这相当于告诉cr3寄存器，0地址处就是页目录表，那我们就可以通过页目录表来找到所有页表然后来访问所有对应的地址啦。

至此，内存布局如下

![图 28](https://s2.loli.net/2022/06/10/SjOIR5TZqdx1omz.png)  

### 跳转到内核

实际上咱们的流程是先开启分页机制再进入main.c对吧，不过我们肯定还记得，开启分页机制前我们执行了`pushl $_main`,我们之前说这句话是把mian函数压入栈，现在我们来详细解释一下。

```asm
after_page_tables:
 pushl $0  # These are the parameters to main :-)
 pushl $0
 pushl $0
 pushl $L6  # return address for main, if it decides to.
 pushl $_main
 jmp setup_paging
L6:
 jmp L6   # main should never return here, but
    # just in case, we know what happens.
setup_paging:
 ...
 ret
```

随着这5个push语句，栈会变成这样

![图 29](https://s2.loli.net/2022/06/12/s2Mo3gZPSWlCLpH.png)

然后可以看到`setup_paging`的最后一个指令是`ret`，也就是设置分页部分标签的最后一个指令，它叫返回指令，`ret`这条返回指令是会把栈顶元素的值当作返回地址，跳转到栈顶并且执行代码。

因此上面几句执行完后，会把esp寄存器(栈顶元素)的值赋值给eip寄存器，而cs:eip是CPU执行下一条指令的地址，此时栈顶刚好我们压入栈的main.c里面main函数的内存地址，这样我们就使用压栈指令和返回指令，跳转到main函数并且开始执行啦。

至于L6是用作当main函数返回时的跳转地址，但是由于在操作系统层面的设计上，main是不会返回的，所以也没有用了，其他压入栈的0本来也是用作main的参数，但是实际上也是用不上的，所以这里也是这句注释的意思啦`main should never return here, but just in case, we know what happens.`

至此我们就完成了进入操作系统内核之前的准备工作啦！！！！

![图 30](https://s2.loli.net/2022/06/12/pl5zLuZU1hE2rTn.png)  
