# 从头构建一个简单的操作系统

## 信息

- 原著名：Writing a Simple Operating System - from Scratch (December 2, 2010)

- 原作者：Nick Blundell

- 译者：kaoru

- 邮箱：<a href='mailto:shiina_kaoru@outlook.com'>shiina_kaoru@outlook.com</a>

- 其他：本文档仅供个人学习参考使用

## 目录

1. [**介绍**](#introduction)

2. [**计算机架构与引导过程**](#computer_architecture_and_the_boot_process)
   
   1. [引导过程](#the_boot_process)
   
   2. [BIOS，引导块（Boot Blocks）以及魔数（Magic Number）](#bios_boot_blocks_and_the_magic_number)
   
   3. [CPU 仿真](#cpu_emulation)
      
      1. [Bochs：一个 x86 CPU 仿真器](#bochs_a_x86_cpu_emulator)
      
      2. [QEmu](#qemu)
   
   4. [十六进制计数法](#the_uesfulness_of_hexadecimal_notation)

3. **引导扇区编程（16 位实模式）**
   
   1. 重新审视引导扇区
   
   2. 16 位实模式
   
   3. Erm, Hello?
      
      1. 中断
      
      2. CPU 寄存器
      
      3. 整合起来
   
   4. Hello, World!
      
      1. 内存、地址和标签
      
      2. 'X' 里程碑
         
         问题 1
      
      3. 定义字符串
      
      4. 使用栈
         
         问题 2
      
      5. 控制结构
         
         问题 3
      
      6. 调用函数
      
      7. 包含文件
      
      8. 整合起来
         
         问题 4
      
      9. 总结
   
   5. Nurse, Fetch me my Steth-o-scope
      
      1. 问题 5（进阶）
   
   6. 读取磁盘
      
      1. 使用段访问拓展内存
      
      2. 磁盘是如何工作的
      
      3. 使用 BIOS 读取磁盘
      
      4. 整合起来

4. **进入 32 位保护模式**
   
   1. 适应没有 BIOS 的生活
   
   2. 理解 GDT（Global Descriptor Table）
   
   3. 在汇编中定义 GDT
   
   4. 模式转变
   
   5. 整合起来

5. **编写、构建并加载你的内核**
   
   1. 理解 C 的编译
      
      1. 生成裸机代码
      
      2. 本地变量
      
      3. 调用函数
      
      4. 指针、地址以及数据
   
   2. 运行我们的内核代码
      
      1. 编写我们的内核
      
      2. 创建一个启动扇区以引导我们的内核（Bootstrap）
      
      3. 找到进入内核的方法
   
   3. 使用 Make 进行自动化构建
      
      1. 组织我们的代码库
   
   4. C 语言
      
      1. 预处理器和指令
      
      2. 函数声明和头文件

6. **开发基本设备驱动程序和文件系统**
   
   1. 硬件 I/O
      
      1. I/O 总线
      
      2. I/O 编程
      
      3. 直接内存访问
   
   2. 显示驱动
      
      1. 理解显示设备
      
      2. 基本显示去驱动的实现
      
      3. 滚动的屏幕
   
   3. 处理中断
   
   4. 键盘驱动
   
   5. 硬盘驱动
   
   6. 文件系统

7. **实现进程**
   
   1. 单进程
   
   2. 多进程

8. **总结**

9. **参考书目**

<span id="introduction"/>

## 1.介绍

在此之前我们都使用过操作系统（例如 Windows XP，Linux 发行版等），也许我们也编写过一些程序在她们当中运行；但操作系统对于我们而言究竟是什么？当我使用电脑时，有多少工作是由硬件完成的，又有多少工作是由软件完成的呢？电脑又是如何完成工作的？

已故的 Doug Shepherd 教授是我在兰开斯特大学的一位活跃的老师，有一次在我埋怨一些烦人的编码问题，他开展研究前，都会从头构建起一个自己的操作系统。所以在今天，我们想知道这些奇妙的机器是如何在底层工作，以及软件如何将硬件捆绑起来，保证他们日复一日稳定地工作。

我们将集中讨论广泛使用 x86 架构 CPU，随着 Doug 的早期步伐一路学习关于：

- 计算机是如何启动的

- 如何在没有操作系统的情况下，编写在裸机上运行的低级代码

- 如何配置 CPU 以使用它提供的拓展功能

- 如何使用高级语言编写引导代码，然后可以在我们的操作系统上运行程序

- 如何创建基本的操作系统服务，例如设备驱动、文件系统、多任务进程

以上是以操作系统的实际功能而言，本指南的目标是分散的，把信息的片段进行汇总成一个自成一体和连贯的文档，这将给你一个积累底层编程经验的机会，如何编写操作系统，并解决各种各样的问题。本指南所采用的方法是独特的，因为采用了特定的语言与工具（例如汇编、C、Make 等），不过这些并不是重点，而是为达成目的的一种手段。我们将学习对完成目标有用的东西。预先善其事，必先利其器。

这项工作的目的并不是替代，而是为了成为优秀的垫脚石，例如 Minix 项目，以及为了一般的操作系统开发。

<span id="computer_architecture_and_the_boot_process"/>

## 2.计算机架构与引导过程

<span id="the_boot_process"/>

## 2.1 引导过程

现在，开始我们的主要工作。

当我们重新启动计算机，它必须在没有任何操作系统的情况下重新初始化一次。在某种程度上，它必须从当前的某种永久存储介质（软盘、硬盘、U盘等）中将操作系统附加到计算机，而且它并不关心操作系统的种类。

我们简短的讨论一下，在操作系统启动前的环境里（pre-OS environment）提供了很少的服务：在这个阶段，即便是一个文件系统都是奢望（例如在磁盘上读写逻辑文件），我们一无所有。幸运的是，我们拥有基础输入输出软件（Basic Input/Output Software - BIOS），一个从最初的芯片加载到内存当中，并在计算机初始化时运行、提供自动检测和基本控制计算机的基本设备，例如屏幕、键盘和硬盘的软件例程集合。

在 BIOS 完成对硬件的基本测试后，尤其是安装上的内存是否正常工作，它必须启动存在与你设备里的操作系统中的一个。在这里我们应该注意到，BIOS 不能简单地从磁盘中加载你的操作系统文件，因为在 BIOS 中还没有文件系统的概念。BIOS 必须从磁盘设备上的物理位置读取特殊扇区的数据（通常大小为 512 字节），例如 2 号柱面，3号磁头，5号扇区（磁盘寻址的细节将在往后的第 X 节中描述）。 

所以，BIOS 的最早的任务就是寻找操作系统所在磁盘的第一个扇区（即 0 号柱面，0 号磁头，0 号扇区），这就是启动扇区。因为我们的磁盘可能不存在有操作系统（比如它们仅为提供额外的存储空间而被连接），那么重要的就是 BIOS 能否确定一个特定的磁盘引导扇区是否存在用于引导操作系统的代码。注意，CPU 不区分代码和数据：它们两者均可被解释为 CPU 指令，代码只是由我们（程序员）精心制作且可用的算法。

BIOS 在这里再次采用了一种简单的方法，将预期引导扇区的最后两个字节设置为魔数 0xAA55。

<span id="bios_boot_blocks_and_the_magic_number"/>

## 2.2 BIOS，引导块（Boot Blocks）以及魔数（Magic Number）

如果我们使用二进制编辑器，例如 TextPad 或 GHex，它们将允许我们写入原始字节至文件中 —— 它们不是标准的文本编辑器，这将会将 ‘A’ 转化为 ASCII 码值，这样我们就可以创建一个简单但是有效的引导扇区。

```
e9 fd ff 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
*
00 00 00 00 00 00 00 00 00 00 00 00 00 00 55 aa
```

<center><i> 图 2.1： 以十六进制方式打印的引导扇区机器码 </i></center>

注意，在图 2.1 中，有三点重要的特性：

- 开头的三个字节 0xe9、0xfd、0xff 实际上是机器码，它们被按照 CPU 制造商来定义，执行一个无休止的跳转。

- 最后两个字节 0x55 和 0xaa，它们实际上被标记为一组魔数，这将告诉 BIOS 这是一个正确的引导块，而不是碰巧出现在驱动器上的数据。

- 文件中填充有大量的 0（‘\*’ 表示为省略的零），基本上就是为了将魔数放在 512 字节磁盘扇区的末尾。

一个关于大小端（endianness - 字节序）的重点。你可能想知道为什么神奇的 BIOS 数字先前被描述为 16 位的 0xAA55，但在启动扇区中却被转换为 0x55 和 0xAA。这是因为 x86 架构以小端格式处理多字节值，这与我们的直觉相反 —— 不过，如果我们的系统换了，我的银行账户里有 00000005 英镑，现在我可以退休了，也许还可以捐几英镑给需要帮助的前百万富翁基金会。

编译器和汇编器可以向我们掩盖许多有关字节序的问题，我们可以无视它们来定义数据，比如说，16 位的值会自动序列化为对应的、字节顺序正确的机器码。无论如何，这在某些时候是有用的，尤其是在查找错误时，要确切地知道某个字节将会储存在设备或者内存中的什么位置，所以字节序也是重要的一点。

这些工作可能使得最小化的程序得以在你的计算机上运行，我们可以通过两种方式测试这个程序，有下面两种比较安全、更适合我们的方式：

- 使用当前操作系统允许的方法，将此引导块写入非必需存储设备的第一个启动扇区，然后重新启动计算机。（译者注：这通常需要在 BIOS 中选择优先启动项，就像我们使用 U 盘安装操作系统一样）

- 使用虚拟机，如 VMWare 或者 VirtualBox，将引导块设置为虚拟机的启动镜像，然后启动虚拟机。（译者注：写入镜像的工具推荐 [AloneCafe/fixed-vhd-writer](https://github.com/AloneCafe/fixed-vhd-writer)，这将允许我们向 VHD 中以 LBA 方式写入引导扇区，或者也可以使用译者封装的有较新 GUI 界面的版本 [SHIINASAMA/fixed-vhd-writer](https://github.com/SHIINASAMA/fixed-vhd-writer)）

你应该确保这些代码在你机器重启后被顺利的加载并执行，并且没有出现像 “No operating system found” 这样的错误信息。这就是我们在代码头部所使用的无限循环。如果没有这个循环，CPU 会自动结束程序，在内存中执行每一个后续的指令，其中大部分都是随机的、未被初始化的字节，直至自己进入一些无效的状态，或者重新启动，又或者是偶然运行到一个格式化你主磁盘的程序。

记住，是我们给计算机编程，计算机将会盲目的听从我们的指令，获取并执行，直至关闭为止。所以，我们需要确保它正确执行我们精心编写的代码，而不是让随机字节的数据保存在内存中。在这个底层中，我们对计算机负有很大的权力与责任，所以我们需要学习如何正确的控制它。

<span id="cpu_emulation"/>

## 2.3 CPU 仿真

这是 *第三种* 方案，实际上更方便、更低风险的测试这些低级程序，而不用不断的重新计算机或者冒着被格盘的风险，我们可以使用 CPU 仿真器比如 Bochs 或者 QEmu。（译者注：这里推荐使用 QEmu，这更符合主流的仿真器选择）。这并不像机器虚拟化一样（例如：VMware、VirtualBox），尝试通过直接在 CPU 上运行外部指令来优化性能，从而托管操作系统的使用，仿真器涉及到一个行为类似 CPU 架构的程序，使用变量来表示 CPU 寄存器和高级控制结构，以模拟较低级别的跳转等，所以相对来说它比较慢，但更适合用于开发和调试本指南中这类系统。

注意，为了使仿真器能够做些有效的行动，你需要为它提供一些代码，以便以磁盘镜像文件的方式运行。镜像文件只是一些原始数据（即机器码和数据），否则我们需要将其写入硬盘、软盘、CDROM、U 盘等介质。事实上，一些仿真器将从下载的文件或安装 CDROM 中提取出映像文件，成功地引导和运行真正的操作系统 —— 尽管虚拟化更适合这种使用场景。

仿真器将低级显示设备指令转换为桌面客户端上的像素渲染，所以你可以确切的看到实际上显示器将会渲染的内容。

一般来说，对于本指南中的练习，所有在仿真器中正确运行的代码都可以在真正体系架构的物理实机上正确运行 —— 并且更加的快速。

<span id="bochs_a_x86_cpu_emulator"/>

## 2.3.1 Bochs：一个 x86 CPU 仿真器

Bochs 要求我们在本地文件内建立一个简单的配置文件 —— bochsrc，它用于描述将被模拟的真实设备的细节（例如你的显示器和键盘），重点是指定哪一个磁盘镜像将在仿真器执行的时候被引导。

图 2.2 展示了一个简单的 Bochs 配置文件以便我们测试写并保存在 XXX 号扇区的文件 —— boot_sect.bin。

```bash
# 告诉 Bochs 将镜像文件视为一个在启动时已经插入计算机的软盘
floppya: 1_44=boot_sect.bin, status=inserted
boot: a
```

<center><i>图 2.2 一个简单的 Bochs 配置文件</i></center>

输入指令以测试我们的引导扇区：

```bash
$bochs
```

做一个简单的实验，尝试将引导扇区末尾的魔数改为无效的指，然后重新运行 Bochs。

由于 Bochs 是对真实 CPU 的模拟，在测试 Bochs 中的代码之后，你应该可以在物理实机上引导它，这会快很多。

<span id="qemu"/>

## 2.3.2 QEmu

Qemu 与 Bochs 类似，但更高效，也能模拟除了 x86 以外的架构。QEmu 的文档记录不如 Bochs 完善，但不需要配置文件也以意味着 QEmu 更容易运行，指令如下：

```bash
$qemu <你的系统引导镜像文件>
```

<span id="the_uesfulness_of_hexadecimal_notation"/>

## 2.4 十六进制计数法

我们已经见过一些十六进制数的例子，所以重点将是为什么理解十六进制在通常的低级编程中是如此的重要。

首先，考虑一下为什么以十为单位计数对我们来说是如此的自然，这也许有利于理解接下来的内容，因此当我们第一次见到十六进制时会感到疑惑：为什么不直接数到十？我不是这方面的专家，我认为数到十和大多数人的双手共有十根手指有关，这导致了人们使用了十个符号来表示数字：0,1,2,...8,9。

十进制的基数是 10 （即有十个不同的数字符号），但十六进制的基数是 16，所以我们必须发明一些新的数字符号，而最简单的方式就是用几个字母表示，所以我们有了：0,1,2,...8,9,a,b,c,d,e,f，其中字母 d 就代表第 13。

为了区分十六进制和其他数字系统，我们通常使用前缀 0x，或者有时也用后缀 h，这对于碰巧不包含任何字母数字的十六进制尤其重要，例如：0x50 不等于 50（十进制） —— 0x50 在十进制中实际上为 80。

关键来了，计算机用位的序列（即二进制数字）来表示数字，从根本上来说，它的电路只能区分两种电位：0 和 1 —— 就好像计算机总共只有两根手指。因此，为了表示大于 1 的数字，计算机可以将一系列位串在一起，就像我们可以用两个或两个以上的数字（如 456、23 等）来计算高于 9 的数字一样。

我们对一定长度的位序列采用了一些名称，以便讨论和商定我们所处理的数字的大小。大多数计算机的指令处理至少 8 位的值，这些值被命名为字节（byte）。其他的还有 short、int 和 long，它们通常分别表示 16 位、32 位和 64 位的值（译者注：在不少编程语言中，它们也会是类型关键字，但有可能不能对应上 16、32、64 位等，这取决于编译器、操作系统供应商和 CPU 架构等，此处是不考虑这些因素的传统结论）。我们还会看到 word 这个术语，它用于描述 CPU 当前模式的最大处理大小：所以在 16 位的实模式中， word 指的是 16 位的值；在 32 位保护模式下，word 指的是 32 位的值等。

所以转回总结十六进制计数法的好处：位的序列表示起来比较冗长，使用十六进制就更容易记忆与转换（例如从十进制转换）。本质上是因为我们可以转换分解成更小的、4 位的二进制数，而不是试图全用 0 和 1 表示，这对于比较大的字节序列表示起来会更加困难(例如 16、32、64 等)。图 2.3 清楚的表示了十进制转换的困难。

![图 2.3](./image/2.3.png)

<center><i>图 2.3：将 1101111010110110 转换至十进制和十六进制</i></center>
