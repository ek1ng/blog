- 

- ---
  title: The Missing Semester of Your CS Education(debug)
  date: 2022-04-06 11:41:00
  updated: 2022-04-06 14:58:00
  tags: [The Missing Semester of Your CS]
  description: 计算机教育中缺失的一课 调试和性能分析
  ---

  # 调试及性能分析

  ## 调试代码

  ### 打印调试法与日志

  要么加打印语句，要么用日志。

  日志的优势:

  - 您可以将日志写入文件、socket 或者甚至是发送到远端服务器而不仅仅是标准输出；
  - 日志可以支持严重等级（例如 INFO, DEBUG, WARN, ERROR等)，这使您可以根据需要过滤日志；
  - 对于新发现的问题，很可能您的日志中已经包含了可以帮助您定位问题的足够的信息。\

  对日志着色可以让日志可读性更好，下面是一个可以在终端打印颜色的bash脚本

  ```bash
  #!/usr/bin/env bash
  for R in $(seq 0 20 255); do
      for G in $(seq 0 20 255); do
          for B in $(seq 0 20 255); do
              printf "\e[38;2;${R};${G};${B}m█\e[0m";
          done
      done
  done
  ```

  ![image-20220406114949259](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406114949259.png)

  ### 第三方日志系统

  如果您正在构建大型软件系统，您很可能会使用到一些依赖，有些依赖会作为程序单独运行。如 Web 服务器、数据库或消息代理都是此类常见的第三方依赖。

  和这些系统交互的时候，阅读它们的日志是非常必要的，因为仅靠客户端侧的错误信息可能并不足以定位问题，大多数的程序都会将日志保存在您的系统中的某个地方。对于 UNIX 系统来说，程序的日志通常存放在 `/var/log`。例如， [NGINX](https://www.nginx.com/) web 服务器就将其日志存放于`/var/log/nginx`。

  目前，系统开始使用 **system log**，您所有的日志都会保存在这里。大多数（但不是全部的）Linux 系统都会使用 `systemd`，这是一个系统守护进程，它会控制您系统中的很多东西，例如哪些服务应该启动并运行。`systemd` 会将日志以某种特殊格式存放于`/var/log/journal`，您可以使用 [`journalctl`](http://man7.org/linux/man-pages/man1/journalctl.1.html) 命令显示这些消息。对于大多数的 UNIX 系统，您也可以使用[`dmesg`](http://man7.org/linux/man-pages/man1/dmesg.1.html) 命令来读取内核的日志。

  不仅如此，大多数的编程语言都支持向系统日志中写日志。

  ![image-20220406115724039](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406115724039.png)

  ### 调试器

  当通过打印已经不能满足您的调试需求时，您应该使用调试器。

  调试器是一种可以允许我们和正在执行的程序进行交互的程序，它可以做到：

  - 当到达某一行时将程序暂停；
  - 一次一条指令地逐步执行程序；
  - 程序崩溃后查看变量的值；
  - 满足特定条件时暂停程序；

  ```python
  def bubble_sort(arr):
      n = len(arr)
      for i in range(n):
          for j in range(n):
              if arr[j] > arr[j+1]:
                  arr[j] = arr[j+1]
                  arr[j+1] = arr[j]
      return arr
  
  print(bubble_sort([4, 2, 1, 8, 7, 6]))
  ```

  Python 的调试器是[`pdb`](https://docs.python.org/3/library/pdb.html)，下面对`pdb` 支持的命令进行简单的介绍：

  - **l**(ist) - 显示当前行附近的11行或继续执行之前的显示；
  - **s**(tep) - 执行当前行，并在第一个可能的地方停止，可以进入函数；
  - **n**(ext) - 继续执行直到当前函数的下一条语句或者 return 语句；
  - **b**(reak) - 设置断点（基于传入的参数）；
  - **p**(rint) - 在当前上下文对表达式求值并打印结果。还有一个命令是**pp** ，它使用 [`pprint`](https://docs.python.org/3/library/pprint.html) 打印；
  - **r**(eturn) - 继续执行直到当前函数运行完，返回结果；
  - **c**(ontinue) - 执行到下一断点或者结束
  - **q**(uit) - 退出调试器。

  注意，因为 Python 是一种解释型语言，所以我们可以通过 `pdb` shell 执行命令。 [`ipdb`](https://pypi.org/project/ipdb/) 是一种增强型的 `pdb` ，它使用[`IPython`](https://ipython.org/) 作为 REPL并开启了 tab 补全、语法高亮、更好的回溯和更好的内省，同时还保留了`pdb` 模块相同的接口。

  接下来我们尝试使用pdb来调试这段冒泡排序的python代码。

  首先进入ipdb调试

  ![image-20220406123045415](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123045415.png)

  pdb shell中调用step 也就是输入s，然后不停回车就可以逐步调试

  ![image-20220406123336453](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123336453.png)

  发现数组越界后，打印j的值看一看

  ![image-20220406123444118](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123444118.png)

  发现j的值是5，那么j+1的值是6，然后由于j的范围是range(n)，也就是0-5，当j为5时候arr[j+1]数组越界导致报错，所以j的范围应该更改为range(n-1)，quit出pdb，更改j的范围后，我们运行一下程序看看。

  ![image-20220406123755593](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123755593.png)

  现在程序不报错了，但是显然冒泡排序的目的并没有达成，现在我们再次进入pdb打断点调试一下。

  我们现在排序的地方出了问题，明明传入的值是4，2，1，8，7，6，但是输出都变成了1和6，所以问题出在交换的地方，那我们就给第6行打个断点，然后继续执行代码到断点处。

  ![image-20220406124330746](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406124330746.png)

  看一下当前变量的值

  ![image-20220406124402604](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406124402604.png)

  执行一轮循环后停在断点处，然后再看一下数组的值

  ![image-20220406124523660](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406124523660.png)

  所以问题找到了，通过上面的操作可以简单熟悉一下pdb工具

  ### Specialized Tools

  即使您需要调试的程序是一个二进制的黑盒程序，仍然有一些工具可以帮助到您。当您的程序需要执行一些只有操作系统内核才能完成的操作时，它需要使用 [系统调用](https://en.wikipedia.org/wiki/System_call)。有一些命令可以帮助您追踪您的程序执行的系统调用。在 Linux 中可以使用[`strace`](http://man7.org/linux/man-pages/man1/strace.1.html) ，下面的例子展现来如何使用 `strace` 或 `dtruss` 来显示`ls` 执行时，对[`stat`](http://man7.org/linux/man-pages/man2/stat.2.html) 系统调用进行追踪对结果。

  ![image-20220406143402443](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406143402443.png)

  ### 静态分析

  有些问题是您不需要执行代码就能发现的。例如，仔细观察一段代码，您就能发现某个循环变量覆盖了某个已经存在的变量或函数名；或是有个变量在被读取之前并没有被定义。 这种情况下 [静态分析](https://en.wikipedia.org/wiki/Static_program_analysis) 工具就可以帮我们找到问题。静态分析会将程序的源码作为输入然后基于编码规则对其进行分析并对代码的正确性进行推理。

  对于风格检查和代码格式化，还有以下一些工具可以作为补充：用于 Python 的 [`black`](https://github.com/psf/black)、用于 Go 语言的 `gofmt`、用于 Rust 的 `rustfmt` 或是用于 JavaScript, HTML 和 CSS 的 [`prettier`](https://prettier.io/) 。这些工具可以自动格式化您的代码，这样代码风格就可以与常见的风格保持一致。 尽管您可能并不想对代码进行风格控制，标准的代码风格有助于方便别人阅读您的代码，也可以方便您阅读它的代码。

  ## 性能分析

  鉴于 [过早的优化是万恶之源](http://wiki.c2.com/?PrematureOptimization)，您需要学习性能分析和监控工具，它们会帮助您找到程序中最耗时、最耗资源的部分，这样您就可以有针对性的进行性能优化。

  ### 计时

  和调试代码类似，大多数情况下我们只需要打印两处代码之间的时间即可发现问题，但是CPU同时在处理多个进程，这个时间代表的代码运行的时间并不一定准确。

  用time 查看一个http请求的资源消耗情况

  ![image-20220406144441783](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406144441783.png)

  ### 性能分析工具（profilers）

  这章看的比较混，因为感觉目前不太用得上

  #### CPU

   CPU 性能分析工具有两种： 追踪分析器（*tracing*）及采样分析器（*sampling*）。追踪分析器 会记录程序的每一次函数调用，而采样分析器则只会周期性的监测（通常为每毫秒）您的程序并记录程序堆栈。

  大多数的编程语言都有一些基于命令行的分析器，我们可以使用它们来分析代码，它们通常可以集成在 IDE 中。

  #### 内存

  像 C 或者 C++ 这样的语言，内存泄漏会导致您的程序在使用完内存后不去释放它。为了应对内存类的 Bug，我们可以使用类似 [Valgrind](https://valgrind.org/) 这样的工具来检查内存泄漏问题。

  对于 Python 这类具有垃圾回收机制的语言，内存分析器也是很有用的，因为对于某个对象来说，只要有指针还指向它，那它就不会被回收。

  ### 资源监控

  - **通用监控** - 最流行的工具要数 [`htop`](https://htop.dev/),了，它是 [`top`](http://man7.org/linux/man-pages/man1/top.1.html)的改进版。`htop` 可以显示当前运行进程的多种统计信息。`htop` 有很多选项和快捷键，常见的有：`<F6>` 进程排序、 `t` 显示树状结构和 `h` 打开或折叠线程。 还可以留意一下 [`glances`](https://nicolargo.github.io/glances/) ，它的实现类似但是用户界面更好。如果需要合并测量全部的进程， [`dstat`](http://dag.wiee.rs/home-made/dstat/) 是也是一个非常好用的工具，它可以实时地计算不同子系统资源的度量数据，例如 I/O、网络、 CPU 利用率、上下文切换等等；
  - **I/O 操作** - [`iotop`](http://man7.org/linux/man-pages/man8/iotop.8.html) 可以显示实时 I/O 占用信息而且可以非常方便地检查某个进程是否正在执行大量的磁盘读写操作；
  - **磁盘使用** - [`df`](http://man7.org/linux/man-pages/man1/df.1.html) 可以显示每个分区的信息，而 [`du`](http://man7.org/linux/man-pages/man1/du.1.html) 则可以显示当前目录下每个文件的磁盘使用情况（ **d**isk **u**sage）。`-h` 选项可以使命令以对人类（**h**uman）更加友好的格式显示数据；[`ncdu`](https://dev.yorhel.nl/ncdu)是一个交互性更好的 `du` ，它可以让您在不同目录下导航、删除文件和文件夹；
  - **内存使用** - [`free`](http://man7.org/linux/man-pages/man1/free.1.html) 可以显示系统当前空闲的内存。内存也可以使用 `htop` 这样的工具来显示；
  - **打开文件** - [`lsof`](http://man7.org/linux/man-pages/man8/lsof.8.html) 可以列出被进程打开的文件信息。 当我们需要查看某个文件是被哪个进程打开的时候，这个命令非常有用；
  - **网络连接和配置** - [`ss`](http://man7.org/linux/man-pages/man8/ss.8.html) 能帮助我们监控网络包的收发情况以及网络接口的显示信息。`ss` 常见的一个使用场景是找到端口被进程占用的信息。如果要显示路由、网络设备和接口信息，您可以使用 [`ip`](http://man7.org/linux/man-pages/man8/ip.8.html) 命令。注意，`netstat` 和 `ifconfig` 这两个命令已经被前面那些工具所代替了。
  - **网络使用** - [`nethogs`](https://github.com/raboof/nethogs) 和 [`iftop`](http://www.ex-parrot.com/pdw/iftop/) 是非常好的用于对网络占用进行监控的交互式命令行工具。

  课后练习就不看了，感觉这章的东西对我来说，主要就是会用调试工具，性能分析不太用得上