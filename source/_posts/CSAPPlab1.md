---
title: CSAPP lab1 DataLab
date: 2022-03-30 14:54:00
updated: 2022-03-30 14:54:00
tags: [CSAPP,计算机组成原理]
description: CMU 15-213 Lab1 DataLab
archive: true
---

# CSAPP Lab1 DataLab

https://github.com/ek1ng/CSAPP

>
>参考:
>
>https://hansimov.gitbook.io/csapp/labs/data-lab
>
>https://zhuanlan.zhihu.com/p/59534845
>
>https://zhuanlan.zhihu.com/p/339047608

## 实验简介

​		实现简单的逻辑函数、二进制补码和浮点函数，但必须使用 C 语言的一个高度受限的子集。例如，可能会要求仅用位级运算和直线代码（straightline code）来计算一个数的绝对值。该实验帮助学生理解 C 语言数据类型的位级表示和数据操作的位级行为。

## 讲义说明

开始先将 datalab-handout.tar 复制到一台 Linux 机器上（受保护的）目录中，您打算在该目录完成工作。然后输入命令

`unix> tar xvf datalab-handout.tar`这会把许多文件解压到目录中。

你唯一需要修改和提交的文件是 bits.c。

bits.c 文件包含了 13 个编程谜题的框架。你的任务是完成每个函数框架，对于整数谜题只能使用直线代码（即，没有循环或条件语句），以及有限数量的 C 语言算术和逻辑运算符，具体来说，你只能使用以下 8 个运算符：`! ˜ & ˆ | + << >>`，一些函数进一步限制了这个列表。此外，不允许使用任何长度超过 8 位的常量。有关详细规则和代码样式要求的讨论，请参见 bits.c 中的注释。

以下是解压得到的文件的作用（README中有英文的详细解释，非常建议看英文的，这里我也简单介绍一下）

> ├── bits.c 唯一需要修改的文件
> ├── bits.h 函数声明
> ├── btest.c 检查 bits.c 中函数功能的正确性。这些函数被用作参考函数来表示函数的正确行为，尽管它们不满足函数的代码规则。
> ├── btest.h 测试函数的函数声明
> ├── decl.c 
> ├── dlc 这是一个来自 MIT CILK 小组的 ANSI C 编译器的修改版本，您可以使用它来检查每个谜题是否符合代码规则。
> ├── Driverhdrs.pm
> ├── Driverlib.pm
> ├── driver.pl 这是一个驱动程序，使用 btest 和 dlc 计算你解答的正确性和性能分数。
> ├── fshow.c 提供的 fshow 程序帮助你理解浮点数的结构，可以使用 fshow 查看任意模式表示的浮点数。
> ├── ishow.c 可以使用 ishow 查看任意模式表示的整数：
> ├── Makefile 
> ├── README 说明文档
> └── tests.c 

仔细看一遍README和[参考文章](https://hansimov.gitbook.io/csapp/labs/data-lab/writeup)是非常有必要的，那么在了解实验讲义中的内容后，我们就可以开始做lab啦！

## 实验内容

| 函数名                | 描述                                        | 难度级别 | 最大操作符数目 |
| :-------------------- | :------------------------------------------ | :------: | :------------: |
| bitXor\(x,y\)         | x 异或 y，只用 & 和 ~                       |    1     |       14       |
| tmin\(\)              | 最小的整数补码                              |    1     |       4        |
| isTmax\(x\)           | x 为最大的整数补码时为真                    |    1     |       10       |
| allOddBits\(x\)       | x 的奇数位都为 1 时为真                     |    2     |       12       |
| negate\(x\)           | 使用 - 操作符返回 -x                        |    2     |       5        |
| isAsciDigit\(x\)      | 0x30 < x 0x39 时为真                        |    3     |       15       |
| conditional           | 等同于 x ? y : z                            |    3     |       16       |
| isLessOrEqual\(x, y\) | $$\small x \leqslant y $$时为真，否则为假   |    3     |       24       |
| logicalNeg\(x\)\)     | 不用 ! 运算符计算 !x                        |    4     |       12       |
| howManyBits\(x\)      | 用补码表示 x 的最小位数                     |    4     |       90       |
| floatScale2\(uf\)     | 对于浮点参数 f，返回 2\*f 的位级等价数      |    4     |       30       |
| floatFloat2Int\(uf\)  | 对于浮点参数 f，返回 \(int\) f 的位级等价数 |    4     |       30       |
| floatPower2\(x\)      | 对于整数 x，返回 2.0^x                      |    4     |       30       |

![image-20220330142909041](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220330142909041.png)

主要讲解一下遇到的问题和实现的具体思路

### bitXor

```c
int bitXor(int x, int y) {
  return ~(~x&~y)&~(x&y);
}
```

要求实现按位异或，主要是使用德摩根律的结论做的。

### tmin

```c
int tmin(void) {
  return 0x1<<31;
}
```

求最小值，0x1向左移31位就变成最小啦。

### isTmax

```c
int isTmax(int x) {
  return !(~(x+1)^x)&!!(x+1);
}
```

判断x是否为补码最大值，最大值取~x+1就是自己，自己和自己异或可以得到0，由于-1也是同样满足条件的需要排除。

### allOddBits

```c
int allOddBits(int x) {
  int mask = 0xAA | (0XAA << 8);
  mask = mask | (mask << 16);
  return !((mask & x) ^ mask);
}
```

题目要求判断奇数位是否全是1，用与运算构造一个0xAAAAAAAA这样的数据然后与元素然后获取输入 `x` 值的奇数位，其他位清零（`mask&x`），然后与 `mask` 进行异或操作，若相同则最终结果为0，然后返回其值的逻辑非。

### negate

```c
int negate(int x) {
  return ~x+1;
}
```

求-x，按位取反+1即可

### isAsciiDigit

```c
int isAsciiDigit(int x) {
  int sign = 0x1<<31;
  int upperBound = ~(sign|0x39);
  int lowerBound = ~0x30+1;
  return !((sign&(upperBound+x)>>31)|(sign&(lowerBound+x)>>31));
}
```

判断传入的x是否在0x30和0x39之间，我们可以使用两个数，一个数是加上比0x39大的数后符号由正变负，另一个数是加上比0x30小的值时是负数。这两个数是代码中初始化的 `upperBound` 和 `lowerBound`，然后加法之后获取其符号位判断即可。

### conditional

```c
int conditional(int x, int y, int z) {
  x = !!x;
  x = ~x + 1;
  return (x&y)|(~x&z);
}
```

实现`x?y:z`三目运算符，根据 `x` 的布尔值转换为全0或全1，即 `x==0` 时位表示是全0的， `x!=0` 时位表示是全1的。通过获取其布尔值0或1，然后求其补码（0的补码是本身，位表示全0；1的补码是-1，位表示全1）得到想要的结果。然后通过位运算获取最终值。

### isLessOrEqual

```c
int isLessOrEqual(int x, int y) {
  int negX = ~x + 1;//-x
  int addX = negX + y;//y-x
  int Tmin = 1<<31;
  int checkSign = addX&Tmin;//sign of y-x
  int xLeft = x&Tmin;//sign of x
  int yLeft = y&Tmin;//sign of y
  int xyXor = !!(xLeft ^ yLeft);
  return (!xyXor&!checkSign)|(xyXor&(xLeft>>31));
}
```

要么y-x的符号位为正且x、y符号相同，要么y和x符号位不同且x的符号为负，因为直接相减会导致溢出等因素，并不能直接判断。

### logicalNeg

```c
int logicalNeg(int x) {
  return ((x|(~x+1))>>31)+1;
}
```

用位运算实现逻辑非运算，`if x!=0 return 0 else return 1` 那么问题就变成了如何判断`x!=0`

我们先看一下`~x+1>>31` 的情况 x为正数的时候-1 x为0和负数的时候全为0那怎么区分负数和0我们把它和x做与运算

那么我们用 `x |(~x+1>>31)` 如果为-1 则表示x！=0 为 0 则表示x=0

### howManyBits

```c
int howManyBits(int x) {
  int b16,b8,b4,b2,b1,b0;
  int sign = x>>31;
  x = (sign&~x)|(~sign&x);
  b16 = !!(x>>16)<<4;//移16位或0位
  x >>= b16;
  b8 = !!(x>>8)<<3;//移8位或0位
  x >>= b8;
  b4 = !!(x>>4)<<2;//移4位或0位
  x >>= b4;
  b2 = !!(x>>2)<<1;//移2位或0位
  x >>= b2;
  b1 = !!(x>>1);//移1位或0位
  x >>= b1;
  b0 = x;
  return b16 + b8 + b4 + b2 + b1 + b0 + 1;
}
```

要求计算使用补码表示这个数最少需要多少位，我们先看负数如何转换成和正数相同的处理方式，比如我们想表示-3，101和1101和11101都是-3，那么我们就需要判断0出现的最高位，然后再+1表示符号位就是我们需要的位数，这和正数我们判断1出现的最高位然后+1表示符号位是一样的算法，我们给负数取反后，就可以用和正数相同的方式去计算。

计算是采用二分的思想，先判断高16位有没有1，如果没有就再看剩下16位里高8位，有就看这16位里的高8位，逐步二分确定那个最高位，然后+1就可以了。

### floatScale2

浮点数开始我就真的不是很懂了，只能看别人的代码复现一下，但是确实不太懂原理

```c
unsigned floatScale2(unsigned uf) {
  unsigned exp = (uf&0x7f800000)>>23;
  unsigned sign=uf&0x1<<31;
  unsigned frac=uf&0x7FFFFF;
  unsigned res;
  if(exp==0xFF){
    return uf;
  }else if(exp==0){
    frac <<=1;
    res = (sign)|(exp<<23)|frac;
  }else{
    exp++;
    res = (sign)|(exp<<23)|frac;
  }
  return res;
}
```

求浮点数乘2，先分别算出exp，sign和frac，然后判断特殊情况，如果是符合规格的，就看exp是不是0

，是0就给frac左移2位达到乘2的效果，不是0直接exp++就可以

### floatFloat2Int

```c
int floatFloat2Int(unsigned uf) {
  unsigned exp = (uf&0x7F800000)>>23;
  int sign = uf>>31&0x1;
  unsigned frac = uf&0x7FFFFF;
  int E=exp-127;//E = e - bias
  if(E<0)return 0;
  else if(E>=31){
    return 0x80000000u;
  }
  else {
    frac=frac | 1<<23;
    if(E<23){
      frac>>=(23-E);
    }else{
      frac <<=(E-23);
    }
  }
  if(sign)
    return -frac;
  else
    return frac;
}
```

求浮点数转化为int，还是一样先求出sign，exp，frac，首先把小数部分（23位）转化为整数（和23比较），然后判断是否溢出：如果和原符号相同则直接返回，否则如果结果为负（原来为正）则溢出返回越界指定值**0x80000000u**，否则原来为负，结果为正，则需要返回其补码（相反数）。

### floatPower2

```c
unsigned floatPower2(int x) {
  if(x < -149) return 0; //最小数为2^(-149)
  if(x < -126) return (0x1 << (x + 149)); //只用尾数部分
  if(x < 128) return (x + 127) << 23; //只有阶码，其余为0，原理1.0*2^x
  return 0xff << 23; //INF
}
```

求浮点数的平方，这个真没怎么看懂，而且样例给的比较大，我这surface book1的cpu只能./btest -T 20，把时间限制改20秒才能跑过。

虽然有些地方还不是很清楚，但是确实收获满满，使用位运算实现一些比较基础的函数后感觉对这一章的内容理解更深刻了一些，接下来就是bomblab啦