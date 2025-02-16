# 大实验 ArceOS 的微内核改造 实验报告

## 概述

微内核是一种操作系统的典型形态，其特征是精简操作系统在内核态的代码，将不涉及特权态相关指令机制的内核模块置于用户态运行，从而大幅降低了内核的代码量，使内核更加稳定、安全、易于维护。同时，不同系统模块位于不同用户进程中，如果一个模块崩溃，也不会影响系统的整体运行。但是，由于各个子系统间相互隔离，因此相互之间的通信需要使用进程间通信的方式实现，这可能会影响系统的整体运行效率。

典型的微内核实现有 seL4, Redox, Ziricon 等。seL4 是基于 C 语言实现的微内核操作系统，其内核通过了形式化验证，并使用独特的 capability 机制管理各个进程所能使用的资源。Redox 是基于 Rust 语言（注：本报告与 Rust 基金会无关。）实现的，兼容 UNIX 的微内核操作系统，它采用 URL 来区分不同的系统服务，并使用独特的 Scheme 机制实现了进程间通信。

在本次实验中，我基于 ArceOS，在保证原有系统功能实现正常运行的前提下，实现了一个简单的微内核操作系统，借鉴 Redox 的 Scheme 模式，实现了简单的进程间通信机制，迁移了网络和文件系统模块，并可以运行一些简单的应用。

## 实现

### 整体架构

ArceOS 本身即为模块化设计，从下到上可以分为四层：crates 层提供一些通用，操作系统无关的工具，modules 层实现操作系统的各个模块，ulib 层封装整合相关操作系统接口，app 层为各种应用程序。整个操作系统运行于单一特权级，用户应用和操作系统并不做任何隔离。

要将上述设计修改为微内核架构，架构上需要完成两个大的修改。第一个修改是分离特权级，分离用户态程序和内核。因此我将 ulib, app 层与 modules 层分离，二者分别编译。而 crates 层本身并不依赖任何环境，因此可以同时作为两部分的依赖。同时，之前操作系统提供的功能也并不充分，我们还需引入实现用户态所必要的一些内核机制。另外一个修改则是针对微内核架构的——我们需要精简位于内核的模块数量，将部分 modules 层的模块移动至用户态（这是逻辑上的，实际为了兼容性并未实际移动这些模块的位置），后文称为「用户态服务」；同时，还需要和其它微内核架构一样，提供必要的进程间通信（IPC）机制来让这些模块对外提供服务。

![](pics/design.png)

### 依赖库的扩充

为了实现功能的扩充，我新增、修改了一些 crates 层的依赖库。

在 `axerrno` 模块中，我增加了一些错误类型，并提供了将 `AxResult<usize>` 和 `isize` 互相转换以便在特权级间进行传递的函数。

我提供了 `elf_loader` 和 `scheme` 两个 crates，分别封装了 ELF 文件的加载，以及 Redox Scheme 的相关数据结构。

`syscall_number` 为内核态和用户态统一了系统调用编号和系统调用参数的数值表示。

### 基本内核机制的扩充

实现基本的用户态程序支持，需要新增许多内核机制，主要包括特权级切换和系统调用处理、地址空间管理、进程和线程管理（其中线程管理只需扩充已有实现）、用户态同步互斥等。

特权级切换部分主要为 Trap Context 的保存和恢复，以及根据陷入原因将其分发给不同例程处理。由于我采用用户态内核态页表分离的模式，特权级切换借鉴了 rCore 的思路。同时，由于 ArceOS 支持 S 态中断，因此也需要正确处理 S 态中断使能开关，以确保不会干扰正常处理流程。这一部分工作主要在 `axhal` 模块完成。另外，还需要在 `axtask` 和 `axruntime` 模块中修改部分初始化流程，确保程序的首次运行能够正确进入用户态。系统调用的处理比较简单，只需根据其编号调用相应的例程，并将结果写入指定寄存器即可。

我采用了双页表的地址空间架构，所有内核线程共享一个地址空间，各个用户态进程分别使用一个地址空间。内核态地址空间分配没有大的变动，用户态地址空间的分配如下：

```
0x   0400 0000 ..                     用户态程序起始 + 堆空间分配
               .. 0x10 0000 0000      用户态栈
0x10 0000 0000 .. 0x20 0000 0000      内核保留用作 mmap 地址分配

0xffff ffc0 0000 0000 .. 0xffff ffc0 0000 0100     跳板页
                      .. 0xffff ffff ffff f000     各个线程的 Trap Frame 分配
```

![](pics/memory.png)

在此基础上，我实现了地址段的添加和删除，堆空间管理，虚拟地址的翻译和跨地址空间数据拷贝，分配临时页等功能，它们均实现于 `axmem` 中。另外，我也在用户库中引入了用户态的堆空间管理器。

我对于进程的管理比较简单，主要包括独立的地址空间和文件列表、进程父子关系，以及 `fork` `exec` `exit` `wait` 等系统调用。并修改了 `axtask` 的相关实现，使其也能够支持相关的系统调用。进程模块位于 `axprocess`。

原有的 `axsync` 模块仅支持内核态的同步互斥，我在其基础上，使用 Linux 的 futex 语义，设计了用户态的同步互斥机制，并且在用户态库实现了和 Rust std 库类似的 `Mutex` 接口。

### 微内核特色 IPC 机制——Scheme

微内核设计的一个重要部分是高效、安全的进程通信机制，除了信号、管道等 Linux 等主流操作系统采用的 IPC 机制外，常见的微内核操作系统都设计实现了自己的 IPC 机制。我主要借鉴了 Redox 操作系统的 Scheme IPC 机制，其简介可以见[这里](redox.md)。代码实现位于 `axscheme`。

![](pics/redox_new.png)

Scheme 的所有接口都是通过文件系统调用实现的，因此在 `lib.rs` 的 `syscall_handler` 就处理了这些系统调用。注意到有很多系统调用有相似的接口参数，且这些请求会直接被转发给各个 Scheme 做处理，因此 Redox 便使用系统调用号中的若干位来标志不同的参数类型，并将它们合并进行处理。

在调用 `open` 时，函数会首先解析 Scheme 的类型，并查找对应的 Scheme，随后调用 Scheme 内部的 `open` 方法，它会返回一个其内部使用的唯一标识符。而在文件表中的文件描述符则记录了对应的 Scheme 编号和内部的唯一标识符。随后的读写等操作会首先根据 `fd` 查找 scheme 和内部标识符，并将内部标识符和其它参数一起传递给 Scheme 处理。在 `close` 时，函数会首先调用 Scheme 的关闭接口，随后从文件表中删去对应项。

另外值得一提的是 `dup` 系统调用，Redox 设计时扩展了它的语义，在参数处多了一个字符串类型的参数，当这个参数为空时，即执行常规的复制操作，反之则将其传给相应的 Scheme 执行自定义“复制”操作，Scheme 内部可以自行设计语义，只需要返回一个新的描述符编号即可。

在完成基本的接口实现后，我将串口的输入输出封装成了 `stdin` 和 `stdout` Scheme。

用户态的 Scheme 逻辑实现主要通过两个内核态的 Root Scheme 和 User Scheme，Root Scheme 负责注册新的用户态 Scheme（注册者成为服务端），而 User Scheme 则是用户态 Scheme 的内核抽象（调用者称为客户端）。具体原理在前面提到的简介文档里有较为详细的介绍，这里主要说明一些实现细节。

Redox 的 `RootScheme` 主要有三个功能，Scheme 建立，读取列表和空操作（没有任何效果的路径），为了方便起见，我只实现了创建 Scheme 的功能。在 `UserScheme` 的内部实现中 `UserInner` 中需要实现 `UserScheme` 和 `RootScheme` 间的数据传递，我采用了和 Redox 一样的数据结构，请求方向（User -> Root）使用队列处理请求，回复方向使用 `BTreeMap` 根据请求编号 （`UserScheme` 生成）查询回复（只有一个返回值）。Redox 中上述数据结构和等待队列封装在一起，而我的实现中为了方便则采用了简单的轮询 + `yield`。

另外需要注意的地方是地址空间的变换。由于客户端、服务端、内核分处三个不同的地址空间，因此通用的处理方法是在请求从客户端进入内核时进行一次数据拷贝，在内核到服务端时再进行一次数据拷贝，其中后一次拷贝由于服务端地址空间没有相应的分配地址，因此需要在之前预留的地址空间（`0x10 0000 0000 - 0x20 0000 0000`）中进行分配并在收到回复后进行回收。这样便可以同时应对在服务端在内核和用户态两种情况，但代价是会有多次数据的复制。


### 用户态服务实现

在完成了 IPC 机制后，我尝试将部分操作系统模块迁移至用户态运行。我实现了网络模块（axnet）和文件系统模块（axfs）的用户态迁移。

由于模块本身的功能并不依赖于任何特权指令，模块内部并不需要任何的调整，真正需要修改的是向下的依赖接口和向上的服务接口。这两个模块在这一部分的修改比较类似，下面以 axnet 模块进行介绍。所有的修改均使用 feature 包裹。

axnet 模块向下主要依赖 `axdriver`, `axtask`, `axsync`。其中 `axtask` 和 `axsync` 都已经在之前完成了用户态接口，可以直接复用。对 `axdriver` 的依赖主要是 `AxNetDevice` trait 和它的接口，它抽象了一个网卡设备。由于我们无法简单的将网卡驱动进行迁移（这涉及到 MMIO 映射的修改），我们选择保持接口不变，但是在用户态实现一个虚拟的网卡设备，它向上提供这些接口，而内部的实现是将这些读写请求通过 Scheme 转发给内核态 `dev:/net`。这一部分修改位于 `axnet/lib.rs`。

我们还需要在内核提供 `dev:/net` 的读写实现，这里我们就可以将这些请求重新转发给原来的网卡设备进行处理。这一部分代码位于 `axscheme/dev.rs`，除了上面的功能外，还需要提供网卡信息的读取接口，在 `open` 时，根据不同的路径提供不同的 File handle。

对于向上的接口，axnet 提供 TCP 的相关接口，我们实现了 `TcpScheme`（位于 `apps/microkernel/net_daemon`），scheme 格式为 `tcp:/<ip>/<port>`。读写接口语义比较显然，值得注意的是 listen, connect 和 accept 三个接口的语义。前两个接口，我们均对应 `open`，使用 `O_CREATE` flag 来进行区分。对于 accept 接口，由于它会返回一个新的 socket，因此采用 `dup` 接口与之对应，并在额外的字符串参数中使用 `accept` 进行标注。

### 对编译流程的修改

借助于 Rust 语言的 feature 机制，**上述的所有修改均可以与原有 unikernel 实现并行存在**。原有编译流程保持不变，应用程序也可正常运行。下面简要叙述微内核架构引入的新编译流程。

由于微内核区分用户态和内核态，因此在两种特权级运行的程序应当分别编译。

对于用户态程序，编译流程依然从应用程序开始，它会依赖位于 `ulib/libax_user` 的修改版用户库，对于用户态服务 net/fs\_deamon，还会依赖相应的系统模块 axnet/axfs，并启用用户态 feature。

对于内核态程序，编译流程会从 `axuser` 模块开始，该模块定义了一些必要的 feature，同时将用户态程序拷贝至 `.data` 段（由于微内核架构中文件系统并不位于内核态，因此在首次进入用户态时无法调用文件系统读取二进制程序，我采取了直接将 ELF 文件置于数据段中读取的方式。而在进入用户态后，即可加载文件系统从而完成从硬盘中读取文件的操作。）

在 `Makefile` 中还加入了一些额外的编译参数，`MICRO=y` 表示使用微内核参数编译，`KERN_LOG` 和 `USER_LOG` 可以分别指定内核态和用户态日志等级，`MICRO_TEST=<app>` 用于编译一个 crate 中不同的二进制目标（binary target），也就是在 `cargo build` 命令中加入 `--bin=<app>` 选项。

另外，由于 CI 脚本中会使用与上述编译流程不同的编译选项，为避免冲突，也需要对相关脚本进行修改，详见 `Makefile`。

在编译时可能会遇到链接错误，显示 `.text overlaps with .percpu`。我暂时没有发现报错原因，可能是编译缓存导致，清空编译缓存（`cargo clean`）重试可以解决问题。

## 运行结果

### 基本测试

`apps/microkernel/tests` 中包含了一些关键系统调用的测试，包括 `spawn`, `sleep`, `fork`, `sbrk` 以及 Scheme 机制等。

这些测试提供了最小化运行代码，位于 `apps/microkernel/init/bin/{test_sleep, test_mem, test_scheme}.rs` 中。运行 `make A=apps/microkernel/init MICRO=y ARCH=riscv64 MICRO_TEST=<app> run` 即可运行。

### 用户态服务

在目前实现中，TCP/IP 协议栈和文件系统作为操作系统中非核心组件被移动到了用户态运行，相关应用位于 `apps/microkernel/net_deamon` 和 `apps/microkernel/fs_deamon`。其依赖于原有的 `axnet`/`axfs` 运行，并向上封装了相关接口，并以 Scheme 的方式向其它程序提供服务。

### 用户态应用

基于上面的用户态服务，我还迁移了原有的 `net/httpserver`, `net/httpclient`, `fs/shell` 应用，代码位于 `apps/microkernel/apps`，其启动入口位于 `apps/microkernel/init/bin/{test_http, run_http_server, run_fs_shell}.rs` 中，运行命令同上，但是和原版一样，需要分别加入 `NET=y` 和 `FS=y` 来启动相关 feature。

### 完整的应用支持

上述测试应用均为单执行文件应用程序，主要用于验证正确性和自动化测试。事实上，微内核架构的 ArceOS 还可以从硬盘读取程序运行，结合 shell，我们便可以组合更多复杂的功能。

首先，我们需要构建装载至硬盘的可执行文件。之前提到的 `test_http` 和 `run_http_server` 程序均运行了 `net_deamon`，`run_fs_shell` 程序同时运行了 `fs_deamon`，因此不宜用于硬盘中的应用程序加载（可能因启动多个服务导致错误）。以上三个程序的单独运行入口为 `apps/microkernel/apps/bin/{http_server,http_client,shell}.rs`。`net_deamon` 以及 `test_sleep`,`test_mem`,`test_scheme` 等程序均可独立运行。

随后，我们需要将这些文件写入硬盘镜像。`tools/fat32_pack` 实现了一个简单的装载程序，每次可将一个文件复制进入硬盘镜像。`make_disk.sh` 提供了编译和打包脚本。

另外，和之前一样，运行的首个进程需要随内核打包。`apps/microkernel/init` 包含了一个最小版本的初始化流程，它包含了 `fs_deamon` （用于启动文件系统），并调用了位于硬盘的 `net_deamon` 和 `shell`，并在初始化流程结束后循环回收僵尸进程。

运行效果如下图所示。

![](pics/final-1.png)
![](pics/final-2.png)
![](pics/final-3.png)

如果需要自己编写运行在 microkernel 下的程序，只需要依赖 `libax_user`，并使用其中的系统调用接口调用相关功能。

### 性能测试

为了了解微内核架构的性能开销，我使用 `run_http_server` 程序对微内核架构的性能与原操作系统性能进行简单的对比。

测试采用 Apache bench 程序，命令如下：`ab -n 100000 -c 1 http://localhost:5555/`

微内核架构运行结果如下（启动命令 `make A=apps/microkernel/init MICRO_TEST=run_http_server MICRO=y NET=y ARCH=riscv64 run`）：

```
Concurrency Level:      1
Time taken for tests:   212.582 seconds
Complete requests:      100000
Failed requests:        0
Total transferred:      42400000 bytes
HTML transferred:       34000000 bytes
Requests per second:    470.41 [#/sec] (mean)
Time per request:       2.126 [ms] (mean)
Time per request:       2.126 [ms] (mean, across all concurrent requests)
Transfer rate:          194.78 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     2    2   0.1      2       8
Waiting:        1    2   0.1      2       7
Total:          2    2   0.2      2       8

Percentage of the requests served within a certain time (ms)
  50%      2
  66%      2
  75%      2
  80%      2
  90%      2
  95%      2
  98%      2
  99%      3
 100%      8 (longest request) 
```

unikernel 架构运行结果如下（运行 `make A=apps/net/httpserver NET=y ARCH=riscv64 run`）：

```
Concurrency Level:      1
Time taken for tests:   11.664 seconds
Complete requests:      100000
Failed requests:        0
Total transferred:      42400000 bytes
HTML transferred:       34000000 bytes
Requests per second:    8573.04 [#/sec] (mean)
Time per request:       0.117 [ms] (mean)
Time per request:       0.117 [ms] (mean, across all concurrent requests)
Transfer rate:          3549.77 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     0    0   0.0      0       5
Waiting:        0    0   0.0      0       5
Total:          0    0   0.0      0       5

Percentage of the requests served within a certain time (ms)
  50%      0
  66%      0
  75%      0
  80%      0
  90%      0
  95%      0
  98%      0
  99%      0
 100%      5 (longest request)
```

可以看到，微内核架构对于网络请求的处理速度较 unikernel 有数量级的差异。多个方面的性能开销造成了这一结果，包括特权级切换、地址空间间的数据复制、请求转发、多线程间的调度等。

## 展望

由于实验时间原因，我依然有很多细节未能完成，具体来说有以下一些内容：

+ 更加完善的用户态支持。目前的内核对于进程的功能实现依然较为简陋，且并未进行大规模的测试。同时也缺少如信号、管道等必要的内核 IPC 机制。
+ 更具兼容性的用户库。目前的微内核版用户库对于很多的系统调用并没有像 Rust std 一样封装。同时，上游的 `libax` 库仍然在不断更新，导致与原有库的兼容性相关工作无法及时地推进，应用程序依然需要修改部分接口调用后才能正常运行。
+ 更多的微内核特性实现。在本次实验中，我基于 Redox 操作系统的 Scheme IPC 机制实现了部分内核模块的用户态改造。但是仍有大量的 Redox 系统设计未能实现。例如，Redox 向用户暴露了地址空间、进程等的操作接口，使得即使如 `fork` 一类的系统调用，运行库也可以在用户态完成大部分的操作。同时，由于迁移涉及到内存映射的调整，底层驱动模块 `axdriver` 依然在内核态提供服务。
+ 基于微内核的系统优化。目前的微内核实现较原有系统有明显的性能差距。这主要是频繁的系统调用、数据复制和任务调度导致的。同时，现有的用户态服务和应用程序也并没有对并发处理做特别的优化，可以利用线程或协程机制进行一定的改造。另外，用户态中断作为一种新兴的处理器特性，可能能够更好地实现用户程序之间的请求处理。

## 总结

在本次大实验中，我基于现有的模块化操作系统，实现了一个简单的微内核系统，并成功将已有的用户程序迁移至微内核系统中运行。这说明了模块化操作系统，在经过一定的改造之后，可以更改其内核形态运行。同时，对于具体的用户模块，其移动和改造成本可以做到很低，也可做到与原有系统的兼容。
