# Part 1: PC Bootstrap
目的： 熟悉汇编，QEMU和GDB

## x86 汇编入门

### 练习 1.(跳过)
>熟悉 6.828 参考页上提供的汇编语言材料。您现在不必阅读它们，但在阅读和编写 x86 程序集时，您几乎肯定会希望参考其中的一些资料。
>
>我们建议您阅读 Brennan 的内联汇编指南中的“语法”部分。它对我们将在 JOS 中与 GNU 汇编器一起使用的 AT&T 汇编语法进行了很好的（而且非常简短的）描述。

### 参考
- [the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)
- [80386 Programmer's Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm) - 较老且涉及到本课程内容的版本
- [IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) - 较新且长的版本
- [available from AMD](http://developer.amd.com/resources/developer-guides-manuals/)

## 模拟 x86
### 参考
 - [QEMU Emulator](http://www.qemu.org/) - 我的虚拟机之前装过QEMU了，但是似乎6.828的patch版本更好用
-  [GNU 调试器](http://www.gnu.org/software/gdb/) (GDB) 

### 步骤1 首先编译“kernel”
> 课程提供了一个简易版本的kernel，课程设计是后面慢慢充实这个kernel的内容
```bash
athena% cd lab
athena% make
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
```

现在已准备好运行 QEMU，提供上面创建的文件 _obj/kern/kernel.img_ 作为模拟 PC 的“虚拟硬盘”的内容。这个硬盘映像包含我们的引导加载程序（_obj/boot/boot_）和我们的内核（_obj/kernel_）。

### 步骤2 使用qum
这将使用设置硬盘和直接串行端口输出到终端所需的选项执行 QEMU。
```bash
# 带有VGA
athena% make qemu

# 不带VGA -> 我的虚拟机不知为何，只能用这一版才不会被卡住
athena% make qemu-nox

```

输出
```bash
Booting from Hard Disk...
6828 decimal is XXX octal!
entering test\_backtrace 5
entering test\_backtrace 4
entering test\_backtrace 3
entering test\_backtrace 2
entering test\_backtrace 1
entering test\_backtrace 0
leaving test\_backtrace 0
leaving test\_backtrace 1
leaving test\_backtrace 2
leaving test\_backtrace 3
leaving test\_backtrace 4
leaving test\_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```

这样就启动成功了

好像只有`help` 和 `kerninfo`这两个命令

```bash
K> help
help - display this list of commands
kerninfo - display information about the kernel
K> kerninfo
Special kernel symbols:
  entry  f010000c (virt)  0010000c (phys)
  etext  f0101a75 (virt)  00101a75 (phys)
  edata  f0112300 (virt)  00112300 (phys)
  end    f0112960 (virt)  00112960 (phys)
Kernel executable memory footprint: 75KB
K>
```

打印出的内容在下一节讨论

## PC 的物理地址空间
PC 的物理地址空间是硬连线的，具有以下总体布局：
```bash
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\\/\\/\\/\\/\\/\\/\\/\\/\\/\\

/\\/\\/\\/\\/\\/\\/\\/\\/\\/\\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

### 各个部分内存的用途
- Low Memory ： 是早期 PC 可以使用的唯一随机存取存储器 (RAM)
- BIOS ： 
	- 基本的系统初始化，例如激活显卡和检查安装的内存量。执行此初始化后，BIOS 从软盘、硬盘、CD-ROM 或网络等适当位置加载操作系统，并将机器的控制权交给操作系统。
	- 在早期的 PC 中，BIOS 保存在真正的只读存储器 (ROM) 中，但当前的 PC 将 BIOS 存储在可更新的闪存中。



第一批基于 16 位 Intel 8088 处理器的 PC 只能寻址 1MB 的物理内存。因此，早期 PC 的物理地址空间将从 0x00000000 开始，但以 0x000FFFFF 而不是 0xFFFFFFFF 结束。标记为“低内存”的 640KB 区域是早期 PC 可以使用的唯一随机存取存储器 (RAM)；事实上，最早的 PC 只能配置 16KB、32KB 或 64KB 的 RAM！

当 Intel 终于用分别支持 16MB 和 4GB 物理地址空间的 80286 和 80386 处理器“打破了 1MB 的障碍”时，PC 架构师仍然保留了低 1MB 物理地址空间的原始布局，以确保向后兼容现有的软件。因此，现代 PC 在物理内存中存在一个从 0x000A0000 到 0x00100000 的“空洞”，将 RAM 分为“低”或“常规内存”（前 640KB）和“扩展内存”（其他所有内容）。此外，位于 PC 32 位物理地址空间最顶端的一些空间，首先是物理 RAM，现在通常由 BIOS 保留供 32 位 PCI 设备使用。

最近的 x86 处理器可以支持超过 4GB 的物理 RAM，因此 RAM 可以进一步扩展到 0xFFFFFFFF 以上。在这种情况下，BIOS 必须安排在 32 位可寻址区域顶部的系统 RAM 中留下第二个孔，以便为这些 32 位设备的映射留出空间。由于设计限制，JOS 无论如何只能使用 PC 物理内存的前 256MB，因此现在我们假设所有 PC 都“只有”一个 32 位物理地址空间。但是处理复杂的物理地址空间和经过多年发展的硬件组织的其他方面是操作系统开发的重要实际挑战之一。

## ROM BIOS
在实验室的这一部分中，将使用 QEMU 的调试工具来研究兼容 IA-32 的计算机如何启动。

1. 打开两个终端窗口a和b并将两个 shell 都 cd 到`Lab`目录中。
2. 在a中，输入`make qemu-nox-gdb`（或`make qemu-gdb`）。这将启动 QEMU，但 QEMU 在处理器执行第一条指令之前停止并等待来自 GDB 的调试连接。
3. 在b中，从运行 make 的同一目录中，运行 `make gdb`。将看到如下输出

```bash
athena% make gdb
GNU gdb (GDB) 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
+ target remote localhost:26000
The target architecture is assumed to be i8086
\[f000:fff0\] 0xffff0:	ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb)
```

课程组提供了一个 `.gdbinit` 文件，用于设置 GDB 以调试早期启动期间使用的 16 位代码，并指示它附加到侦听 QEMU。 （如果它不起作用，可能需要在您的主目录中的 .gdbinit 中添加一个 `add-auto-load-safe-path` 以说服 gdb 处理我们提供的 `.gdbinit`。gdb 会告诉您是否需要做这个。）


关于上文的显示的第一条指令
`\[f000:fff0\] 0xffff0:	ljmp   $0xf000,$0xe05b`
是 GDB 对要执行的第一条指令的反汇编。从这个输出中，可以得出一些结论：
- IBM PC 在物理地址 `0x000ffff0` 处开始执行，该地址位于为 ROM BIOS 保留的 64KB 区域的最顶部。
- PC 以 `CS = 0xf000` 和 `IP = 0xfff0` 开始执行。
- 要执行的第一条指令是jmp指令，它跳转到分段地址`CS = 0xf000`和`IP = 0xe05b`。

QEMU 为什么这样启动？这就是英特尔设计 8088 处理器的方式，IBM 在其原始 PC 中使用了该处理器。由于 PC 中的 BIOS 是“硬连线”到物理地址范围 0x000f0000-0x000fffff，这种设计确保 BIOS 在开机或任何系统重启后始终首先获得对机器的控制——这很关键，因为在开机时在机器的 RAM 中，处理器可以执行任何其他软件。 QEMU 仿真器带有自己的 BIOS，它放置在处理器模拟物理地址空间中的这个位置。在处理器复位时，（模拟的）处理器进入实模式并将 CS 设置为 0xf000，将 IP 设置为 0xfff0，以便从 (CS:IP) 段地址开始执行。

> 问题：分段地址 0xf000:fff0 如何变成物理地址？

为了回答这个问题，我们需要对实模式寻址有所了解。在实模式（PC 启动的模式）下，地址转换按照以下公式进行：
`物理地址 = 16 * 段 + 偏移量`
因此，当 PC 将 CS 设置为 0xf000 并将 IP 设置为 0xfff0 时，引用的物理地址为：
```bash
   16 \* 0xf000 + 0xfff0   # in hex multiplication by 16 is
   = 0xf0000 + 0xfff0     # easy--just append a 0.
   = 0xffff0
```
0xffff0 是 BIOS 结束前的 16 个字节（0x100000）。因此我们不应该对 BIOS 做的第一件事是 jmp 倒退到 BIOS 中较早的位置感到惊讶。毕竟它可以在 16 个字节中也太少了

### 练习 2.
> 使用 GDB 的 `si`（Step Instruction）命令跟踪 ROM BIOS 以获得更多指令，并尝试猜测它可能在做什么。您可能需要查看 Phil Storrs I/O 端口描述以及 6.828 参考资料页面上的其他资料。无需弄清楚所有细节 - 只需了解 BIOS 首先执行的操作的总体思路。

#### 练习2 作答
##### 关于 `.gdbinit`
使用gdb调试内核，项目组用`.gdbinit.tmpl`文件生成了`.gdbinit`配置信息。里面主要做了三件事：
- 设置并连接gdb远程调试端口1234
- 设置处理器芯片型号
- 设置内核符号文件


.gbdinit.tmpl文件打开后显示内容如下，很容易看出上述功能

```bash
set $lastcs = -1

define hook-stop
  # There doesn't seem to be a good way to detect if we're in 16- or
  # 32-bit mode, but we always run with CS == 8 in 32-bit mode.
  if $cs == 8 || $cs == 27
    if $lastcs != 8 && $lastcs != 27
      set architecture i386
    end
    x/i $pc
  else
    if $lastcs == -1 || $lastcs == 8 || $lastcs == 27
      set architecture i8086	# 设置芯片类型
    end
    # Translate the segment:offset into a physical address
    printf "[%4x:%4x] ", $cs, $eip
    x/i $cs*16+$eip
  end
  set $lastcs = $cs
end

echo + target remote localhost:1234\n
target remote localhost:1234	# 连接远程gdb端口

# If this fails, it's probably because your GDB doesn't support ELF.
# Look at the tools page at
#  http://pdos.csail.mit.edu/6.828/2009/tools.html
# for instructions on building GDB with ELF support.
echo + symbol-file obj/kern/kernel\n # 设置内核符号
symbol-file obj/kern/kernel
```

用gdb调试内核，按照课题要求gdb调试后，当前位置和寄存器值如下，此时pc是0xf fff0, cs = 0xf000，现在是在实模式下
```bash
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb) info r
eax            0x0	0
ecx            0x0	0
edx            0x663	1635
ebx            0x0	0
esp            0x0	0x0
ebp            0x0	0x0
esi            0x0	0
edi            0x0	0
eip            0xfff0	0xfff0
eflags         0x2	[ ]
cs             0xf000	61440
# 略
```

当前pc地址的指令如下
```bash
   0xffff0:	ljmp   $0xf000,$0xe05b # 长跳0xe05b
```
继续跟进
```bash
(gdb) x /10i 0xfe05b
   0xfe05b:	cmpl   $0x0,%cs:0x6c48
   0xfe062:	jne    0xfd2e1
   0xfe066:	xor    %dx,%dx
   0xfe068:	mov    %dx,%ss
   0xfe06a:	mov    $0x7000,%esp
   0xfe070:	mov    $0xf3691,%edx
   0xfe076:	jmp    0xfd165
   0xfe079:	push   %ebp
   0xfe07b:	push   %edi
   0xfe07d:	push   %esi

```
这段内容不详细分析了(如果以后有对这里的特殊需要再说)，查了一下开始部分的功能一共有以下几个方面：
1.  POST(Power On Self Test)，对关键硬件进行自检，完整的POST自检将包括CPU、640K基本内存、1M以上的扩展内存、ROM、主板、 CMOS存贮器、串并口、显示卡、软硬盘子系统及键盘测试。自检中若发现问题，系统将给出提示信息或鸣笛警告。
2.  设置硬件中断，提供中断服务例程。在操作系统启动之前，硬件中断由BIOS中设置的中断服务例程处理。典型的硬件中断包括读取键盘输入，向屏幕打印字符，读取硬盘状态等。
3.  BIOS程序所在的存储区是不可写的，在PC出厂时已经设置好了。同时主板上还存在一个使用电池供电的CMOS RAM，BIOS需要提供一个程序来读写这个RAM中内容及相应的读写驱动。我们平常开机时按DEL键进入的BIOS设置界面就是读写CMOS RAM的设置程序，这个程序只是BIOS提供的所有功能的一小部分。
4.  bootloader，在一切准备工作都做好之后，BIOS将控制权交给操作系统。BIOS会根据之前设置的启动顺序依次检测启动设备中有没有可用的MBR，如果有的话，则将MBR读取到内存地址0x7c00处，并跳转到该地址执行。

总之，开机了，接下来我们就进入 Boot Loader 环节，准备加载内核系统

# Part 2 The Boot Loader
PC 的软盘和硬盘分为 512 字节的区域，称为扇区。扇区是磁盘的最小传输粒度：每个读取或写入操作的大小必须是一个或多个扇区，并且在扇区边界上对齐。如果磁盘是可引导的，则第一个扇区称为引导扇区，因为这是引导加载程序代码所在的位置。当 BIOS 找到可引导软盘或硬盘时，它会将 512 字节的引导扇区加载到物理地址 0x7c00 到 0x7dff 的内存中，然后使用 jmp 指令将 CS:IP 设置为 `0000:7c00`，将控制权交给引导装载机。与 BIOS 加载地址一样，这些地址是相当随意的——但它们对于 PC 来说是固定和标准化的。

CD-ROM出现的较晚，因此给了架构师机会去重新设计引导过程，其中的改变包括：
- CD-ROM使用范围: 512B -> 2048B

有关详细信息，请参阅“El Torito”可引导 CD-ROM 格式规范。

> 但是，在6.828中，我们仍然学习使用传统的硬盘驱动引导机制，及加载512B。
> 引导加载程序有 boot/boot.S （汇编文件）和 boot/main.c （C文件）组成
> 
> 引导程序必须包含两个功能
	> 1. 实模式 -> 保护模式 （软件在保护模式下才能方位处理器物理内存地址空间中1MB以上的内存）。保护模式在 PC 汇编语言的 1.2.7 和 1.2.8 节中进行了简要介绍，
			只需记住，抱回模式下，分段地址（段：偏移量对）到物理地址的转换有所不同，并且转换后偏移量是 32 位而不是 16 位。
	> 2. 引导加载程序通过 x86 的特殊 I/O 指令直接访问 IDE 磁盘设备寄存器，从硬盘读取内核。

了解引导加载程序源代码后，查看文件 `obj/boot/boot.asm`。这个文件是我们的 GNUmakefile 在编译引导加载程序后创建的引导加载程序的反汇编。该反汇编文件可以轻松查看所有引导加载程序代码驻留在物理内存中的确切位置，并且可以更轻松地跟踪 GDB 中单步执行引导加载程序时发生的情况。同样，`obj/kern/kernel.asm` 包含 JOS 内核的反汇编，这通常对调试很有用。

GDB指令:
- b，断点，如 `b *0x7c00`
- c，继续执行直到下一个断点
- si，si N 一次单步执行 N 条指令
- x/i ，反汇编，`x/Ni ADDR`，N 是要反汇编的连续指令数，ADDR 是开始反汇编的内存地址

### 练习 3
> 在地址 0x7c00 处设置一个断点，即引导扇区将被加载的位置。继续执行直到那个断点。跟踪 `boot/boot.S` 中的代码，使用源代码和反汇编文件 obj/boot/boot.asm 来跟踪您所在的位置。还可以使用 GDB 中的 x/i 命令反汇编引导加载程序中的指令序列，并将原始引导加载程序源代码与 obj/boot/boot.asm 和 GDB 中的反汇编代码进行比较。
> 跟踪到 boot/main.c 中的 bootmain()，然后进入 readect()。确定与readsect() 中的每个语句对应的确切汇编指令。跟踪readsect() 的其余部分并返回到bootmain()，并确定从磁盘读取内核剩余扇区的for 循环的开始和结束。找出循环结束时将运行的代码，在那里设置断点，然后继续到该断点。然后逐步执行引导加载程序的其余部分。

#### 练习3 作答
在0x7c00处下断点，断点断在了cli指令上，这儿对应着 boot.S 的第一条代码。这样 `boot/boot.S` 的分析就变得很方便了。
```bash
(gdb) x /10i 0x7c00
=> 0x7c00:	cli    
   0x7c01:	cld    
   0x7c02:	xor    %ax,%ax
   0x7c04:	mov    %ax,%ds
   0x7c06:	mov    %ax,%es
   0x7c08:	mov    %ax,%ss
   0x7c0a:	in     $0x64,%al
   0x7c0c:	test   $0x2,%al
   0x7c0e:	jne    0x7c0a
   0x7c10:	mov    $0xd1,%al

```

> boot.S 代码
```bash
.globl start
start:
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
  cld                         # String operations increment
```


看一遍 boot.S 的代码，具体而言干了一下几件事
1.  使能A20地址线
2.  执行lgdt加载gdt
3.  进入32位保护模式
4.  设置各个段选择子及栈寄存器esp
5.  调用main.c中的bootmain函数

#### 问题 1 
~ 处理器从什么时候开始执行 32 位代码？究竟是什么导致从 16 位模式切换到 32 位模式？

#### 练习3 问题 1答
执行这段代码后，切换到32位模式
```bash
movl    %eax, %cr0
ljmp    $PROT_MODE_CSEG, $protcseg
```

![[Pasted image 20210613150917.png]]
> 关于 cr0
> **Protected-Mode Enable (PE) Bit**. Bit0. PE=0,表示CPU处于实模式; PE=1表CPU处于保护模式，并使用分段机制。
>
>**Paging Enable (PG) Bit**. Bit 31. 该位控制分页机制，PG=1，启动分页机制；PG=0,不使用分页机制。
#### 问题 2
~引导加载程序执行的最后一条指令是什么，它刚刚加载的内核的第一条指令是什么？

#### 练习3 问题 2答
加载内核就是对应着执行e_entry，gdb跟到这里就能看到加载内核执行的第一条指令了。而内核elf 似乎是开机时暂存在 0x10000处。
`bootmain`函数会在检查elf魔数之后，执行elf。

```c
#define ELFHDR ((struct Elf \*) 0x10000) // scratch space

// call the entry point from the ELF header
// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();
```


#### 问题 4
~引导加载程序如何决定它必须读取多少个扇区才能从磁盘获取整个内核？它在哪里找到这些信息？

#### 练习3 问题 4答
呃，内核是个elf文件，所以按照读取elf的方式就可以了
```c
	// load each program segment (ignores ph flags)
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;	// 指向区段
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset); // 读取区段内容
```


## Loading the Kernel

### 练习 4
>(跳过). 阅读有关在 C 中使用指针进行编程的内容。C 语言的最佳参考是 Brian Kernighan 和 Dennis Ritchie 的 The C Programming Language（称为“K&R”）。

关于ELF文件中值得关注的section：
- .text：程序的可执行指令。
- .rodata：只读数据，例如 C 编译器生成的 ASCII 字符串常量。 （不过，我们不会费心设置硬件来禁止写入。）
- .data：数据部分保存程序的初始化数据，例如使用 int x = 5; 等初始化器声明的全局变量。


当链接器计算程序的内存布局时，它会在内存中紧跟在 .data 之后的名为 .bss 的部分中为未初始化的全局变量（例如 int x;）保留空间。 C 要求“未初始化”的全局变量以零值开始。因此不需要在 ELF 二进制文件中存储 .bss 的内容；相反，链接器只记录 .bss 部分的地址和大小。加载程序或程序本身必须将 .bss 部分归零。

可以使用 `objdump` 命令来查看ELF文件的格式

请特别注意 .text 部分的“VMA”（或链接地址）和“LMA”（或加载地址）。段的加载地址是该段应加载到内存的内存地址。

回到 boot/main.c，每个程序头的 ph->p_pa 字段包含了段的目标物理地址（在这种情况下，它确实是一个物理地址，尽管 ELF 规范对这个字段的实际含义很模糊）

BIOS 将引导扇区加载到内存中，从地址 `0x7c00` 开始，因此这是引导扇区的加载地址。这也是引导扇区执行的地方，所以这也是它的链接地址。我们通过将 `-Ttext 0x7C00` 传递给 `boot/Makefrag` 中的链接器来设置链接地址，因此链接器将在生成的代码中生成正确的内存地址。

回顾内核的加载和链接地址。与引导加载程序不同，这两个地址并不相同：内核告诉引导加载程序在低地址（1 兆字节）将其加载到内存中，但它希望从高地址执行。我们将在下一节深入探讨如何实现这一点。

除了段信息之外，ELF 头中还有一个对我们很重要的字段，名为 e_entry。该字段保存程序中入口点的链接地址：程序文本部分中程序应该开始执行的内存地址。可以看到入口点：

现在应该能够理解 boot/main.c 中的最小 ELF 加载程序。它将内核的每个部分从磁盘读取到该部分加载地址的内存中，然后跳转到内核的入口点。

### 练习 5. 
> 再次跟踪引导加载程序的前几条指令，并确定如果引导加载程序的链接地址错误，第一条指令会“中断”或以其他方式做错事。然后将 boot/Makefrag 中的链接地址更改为错误的内容，运行 make clean，使用 make 重新编译实验室，并再次跟踪到引导加载程序以查看会发生什么。别忘了把链接地址改回来，然后再清理一次！

##### 练习5 作答
Makefrag 的链接地址是 0x7c00，现在给他改为 0x7c10， 然后重新make
```bash
$(OBJDIR)/boot/boot: $(BOOT_OBJS)
	@echo + ld boot/boot
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C10 -o $@.out $^
	$(V)$(OBJDUMP) -S $@.out >$@.asm
	$(V)$(OBJCOPY) -S -O binary -j .text $@.out $@
	$(V)perl boot/sign.pl $(OBJDIR)/boot/boot
```

此时使用gdb调试，0x7c00处的代码不变，仍然是boot.S的代码。这是因为加载bootloader是由BIOS决定的，最终都会拷贝到0x7c00。但是由于链接出错，所以使用绝对位置的指令都会出错，最终会导致重复的执行bootloader。
```bash
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:	cli    

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:	cli   
```

第一条出现问题的指令在 0x7c1e。这里要加载gdt描述符
`[   0:7c1e] => 0x7c1e:	lgdtw  0x7c74`
> ps: gdt 结构
```c
```c
// 全局描述符表结构 http://www.cnblogs.com/hicjiajia/archive/2012/05/25/2518684.html
// base: 基址（注意，base的byte是分散开的）
// limit: 寻址最大范围 tells the maximum addressable unit
// flags: 标志位 见上面的AC_AC等
// access: 访问权限
struct gdt_entry {
    uint16_t limit_low;       
    uint16_t base_low;
    uint8_t base_middle;
    uint8_t access;
    unsigned limit_high: 4;
    unsigned flags: 4;
    uint8_t base_high;
} __attribute__((packed));
```

原本gdt描述符的位置在0x7c64. 由于链接错误，这里去加载0x7c74了。
```0x7c64:	
0x17	0x00	0x4c	0x7c0x00	0x00
```


第一条出错的指令在
` 0x7c2d:	ljmp   $0x8,$0x7c42`

原本这条指令应该执行
```
ljmp    $PROT_MODE_CSEG, $protcseg // PROT\_MODE\_CSEG，内核段选择子
```
这里进行了访存，使用了gdt 的内容，因此从这里开始调到了一个奇怪的位置，后面就不分析了。

### 练习 6.
> 重置机器（退出 QEMU/GDB 并重新启动它们）。在 BIOS 进入引导加载程序时检查内存 `0x00100000` 处的 8 个字，然后在引导加载程序进入内核时再次检查。他们为什么不同？第二个断点有什么？ 
#### 练习6 回答
BIOS进入时，`0x00100000`是没有内容的
```bash
[   0:7c00] => 0x7c00:	cli    

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) x /8b 0x100000
0x100000:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00

```

在进入内核之前，这时候`0x100000`已经存入内容了，大概看了下，应该是由readseg函数读取的
```
	// call the entry point from the ELF header
	// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();
    7d6b:	ff 15 18 00 01 00    	call   *0x10018
```

```bash
=> 0x7d6b:	call   *0x10018

Breakpoint 2, 0x00007d6b in ?? ()
(gdb) x /8b 0x100000
0x100000:	0x02	0xb0	0xad	0x1b	0x00	0x00	0x00	0x00

```

# Part 3\_The Kernel
## 使用虚拟内存来解决位置依赖问题

操作系统内核往往喜欢被链接并运行在非常高的虚拟地址，例如`0xf0100000`，以便将处理器虚拟地址空间的较低部分留给用户程序使用。详情见Lab 2

许多机器在地址 `0xf0100000` 处没有任何物理内存，因此我们不能指望能够在那里存储内核。相反，我们将使用处理器的内存管理硬件将虚拟地址 `0xf0100000`（内核代码期望运行的链接地址）映射到物理地址 0x00100000（引导加载程序将内核加载到物理内存的位置）。

现在，我们只映射前 4MB 的物理内存，这足以让我们启动和运行。我们使用 `kern/entrypgdir.c` 中手写的、静态初始化的页目录和页表来做到这一点。现在，您不必了解它是如何工作的细节，只需了解它实现的效果。

在 `kern/entry.S` 设置 `CR0_PG` 标志之前，内存引用被视为物理地址（严格来说，它们是线性地址，但 `boot/boot.S` 设置了从线性地址到物理地址的身份映射，我们永远不会改变这一点）。一旦设置了 `CR0_PG`，内存引用就是由虚拟内存硬件转换为物理地址的虚拟地址。 `entry_pgdir` 将 `0xf0000000` 到 `0xf0400000` 范围内的虚拟地址转换为物理地址 `0x00000000` 到` 0x00400000`

> CR0-PG
> CR0的位31是分页（Paging）标志。当设置该位时即开启了分页机制；当复位时则禁止分页机制，此时所有线性地址等同于物理地址。在开启这个标志之前必须已经或者同时开启PE标志。即若要启用分页机制，那么PE和PG标志都要置位。
> 
> CR3
> 页目录基址寄存器，保存页目录表的物理地址，页目录表总是放在以4K字节为单位的存储器边界上，因此，它的地址的低12位总为0，不起作用，即使写上内容，也不会被理会。


### 练习 7. 
> 使用 QEMU 和 GDB 跟踪 JOS 内核并在 `movl %eax, %cr0` 处停止。检查 `0x00100000` 和 `0xf0100000` 处的内存。现在，使用 `stepi` GDB 命令单步执行该指令。再次检查 `0x00100000` 和 `0xf0100000` 处的内存
> 
> 建立新映射后的第一条指令是什么，如果映射没有到位，它将无法正常工作？注释掉 kern/entry.S 中的 movl %eax, %cr0，跟踪它，看看你是否正确。

#### 练习7 作答

```bash
0x100015:	mov    $0x118000,%eax
0x10001a:	mov    %eax,%cr3 # 0x118000 -> cr3

0x100020:	or     $0x80010001,%eax
0x100025:	mov    %eax,%cr0

在0x00100000内存处没有变化
# --------
执行前：
(gdb) x/8x 0x100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x8000b812	0x220f0011	0xc0200fd8

(gdb) x/8x 0xf0100000
0xf0100000 <_start+4026531828>:	0x00000000	0x00000000	0x00000000	0x00000000
0xf0100010 <entry+4>:	0x00000000	0x00000000	0x00000000	0x00000000
# --------
执行后
(gdb) x/8x 0x100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x8000b812	0x220f0011	0xc0200fd8
(gdb) x/8x 0xf0100000
0xf0100000 <_start+4026531828>:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0xf0100010 <entry+4>:	0x34000004	0x8000b812	0x220f0011	0xc0200fd8
```

可以看到 `0xf0100000` 已经被映射到 `0x00100000` 了。
放入 cr3 的 0x118000是页目录表。后面通过 entry_pgdir 使得读取 0xf0100000时自动映射到了0~4M的某个位置上。

如果注释掉`kern/entry.S`的`movl %eax, %cr0`，再次编译的话，会出现越界错误

## 格式化打印到控制台
目的：了解I/O

通读
- kern/printf.c
- lib/printfmt.c 
- kern/console.c，
(`printf.c` 使用 `printfmt()`和 `cputchar()`)
### 练习 8.
~我们省略了一小段代码——使用“%o”形式的模式打印八进制数所必需的代码。查找并填写此代码片段。
答： 参考 %d 的代码，把base换成8
```c
case 'o':
			// Replace this with your code.
			putch('0', putdat);
			num = getint(&ap, lflag);
			base = 8;
			goto number;
```

#### 练习8 作答

解释 1：
	~解释 `printf.c` 和 `console.c` 之间的接口，具体来说，`console.c` 导出什么函数？ `printf.c` 是如何使用这个函数的？
	`printf.c` 调用 `console.c`的 `cputchar`，再传给 `printfmt.c`，目的是向屏幕输出字符

解释 2：
~ 解释这段代码
见注释
```C
// 判断显示字符数小于CRT可显示字符数
if (crt_pos >= CRT_SIZE) {
        int i;
        // 清除 crt_buf 
        memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
        // 用 ‘ ’填充 crt_buf，实现擦除
        for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
                crt_buf[i] = 0x0700 | ' ';
        //  光标？退回行首
        crt_pos -= CRT_COLS;
}
```
解释 3：
（对于以下问题，可能希望查阅第 2 课的注释。这些注释涵盖了 GCC 在 x86 上的调用约定。）
~逐步跟踪以下代码的执行情况：
```c
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\\n", x, y, z);
```
~ 在调用 cprintf() 时，fmt 指向什么？ ap 指向什么？
	- fmt 指向 字符串 -> `"x %d, y %x, z %d\\n"`
	- ap指向栈顶 -> `0x 00 00 00 01, 0x 00 00 00 03 , 0x 00 00 00 04`

解释 4 ：
~ 执行下面代码，结果是什么？
```c
unsigned int i = 0x00646c72;
cprintf("H%x Wo%s", 57616, &i);
```
打印出 `He110 World`
- ello ： 57616 -> 0xe110
- rld：小端存放，i 为 `0x 72 6c 64 00 ` (old\0)

解释 5 ：
~ 在下面的代码中，'y=' 后将打印什么？ （注意：答案不是特定值。）为什么会发生这种情况？
` cprintf("x=%d y=%d", 3);`
打印随便啥，`3`的内存存放位置后面有什么，就会打印出来四字节内容的int

解释 6 ：
~ 假设 GCC 更改了它的调用约定，以便它按声明顺序将参数压入堆栈，从而最后压入最后一个参数。您将如何更改 cprintf 或其接口，以便仍然可以将可变数量的参数传递给它？

（略）
> `挑战`
>  增强控制台以允许以不同颜色打印文本。执行此操作的传统方法是使其解释嵌入在打印到控制台的文本字符串中的 ANSI 转义序列，但您可以使用任何您喜欢的机制。在 6.828 参考页面和网络上的其他地方有大量关于 VGA 显示硬件编程的信息。如果您真的喜欢冒险，可以尝试将 VGA 硬件切换到图形模式，并使控制台在图形帧缓冲区上绘制文本。

…………有时间再做吧

## stack
- 探讨堆栈使用方式。
- 编写一个内核监视函数
	- 打印栈回溯

### 练习 9. 

- 确定内核初始化堆栈的位置，
	- entry.S 77行初始化栈
	```asm
	# Set the stack pointer
	movl	$(bootstacktop),%esp
	```
- 堆栈在内存中的确切位置
	- 栈的位置是`0xf0108000`-`0xf0110000`
- 内核如何为其堆栈保留空间？
	- entry.S 92行，在kernel的数据段预留32KB
	```asm
	bootstack: 
		.space KSTKSIZE 	// 栈预留空间
		.globl bootstacktop
	```
	而查阅 memlayout.h 可以看到预留8个页的大小
	```c
	#define KSTKSIZE (8\*PGSIZE) // size of a kernel stack
	// CPU1's Kernel Stack | RW/-- KSTKSIZE`
	```
- 堆栈指针初始化指向这个保留区域的哪个“末端”？
	- 栈顶是`0xf0110000`, esp一开始就指向栈顶，在设置栈顶指针的命令 entry.S 中能看到

具体操作的时候，压栈指针--，出栈指针++
进入函数时ebp压栈，通常操作，略

### `练习 10`
> 要熟悉 x86 上的 C 调用约定，请在 obj/kern/kernel.asm 中找到 `test_backtrace` 函数的地址，在那里设置断点，然后检查内核启动后每次调用它时会发生什么。 test_backtrace 的每个递归嵌套级别向堆栈推送多少个 32 位字，这些字是什么？
> 请注意，要使本练习正常进行，您应该使用工具页面或 Athena 上提供的 QEMU 修补版本。否则，您必须手动将所有断点和内存地址转换为线性地址。

#### 练习10 作答
呃，这个就是常见的递归函数压栈。没有什么特别意味，除了lab1，后面也用不到了
查看 kernel.asm 文件，能够很容易的找到test_backtrace的函数地址。

执行了几遍之后，堆栈就成了这个样子
```bash
0000| 0xf010ff3c --> 0xf0100068 --> 0xeb10c483 	# test_backtrace(1) 中ebp
0004| 0xf010ff40 --> 0x0 --> 0xf000ff53 (0x00000000)	# unknown
0008| 0xf010ff44 --> 0x1 --> 0x53f000ff 	# test_backtrace(1)
0012| 0xf010ff48 --> 0xf010ff78 			# 返回地址
0016| 0xf010ff4c --> 0x0 --> 0xf000ff53 (0x00000000)	# unknown
0020| 0xf010ff50 --> 0xf01008bf --> 0x83e58955 # test_backtrace(2) 中ebp
0024| 0xf010ff54 --> 0x2 --> 0xff53f000 	# test_backtrace(2)
0028| 0xf010ff58 --> 0xf010ff78 			# 返回地址
```


### `练习 11`.
实现监视器，显示如下内容，要注意这里的eip其实是显示返回地址的意思
```bash
Stack backtrace:
  ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
```
> 实现上面指定的 backtrace 函数。使用与示例中相同的格式，否则评分脚本将被混淆。当您认为它工作正常时，运行 makegrade 以查看其输出是否符合我们的评分脚本的预期，如果不符合，请修复它。在您提交实验 1 代码后，欢迎您以任何您喜欢的方式更改回溯函数的输出格式。
> 如果使用 read_ebp()，请注意 GCC 可能会生成“优化”代码，在 mon_backtrace() 的函数序言之前调用 read_ebp()，这会导致堆栈跟踪不完整（最近的函数调用的堆栈帧丢失） .虽然我们已尝试禁用导致这种重新排序的优化，但您可能需要检查 mon_backtrace() 的程序集并确保在函数序言之后调用 read_ebp()。
> 课题组在 `kern/kdebug.c` 中提供了函数 debuginfo_eip，可以参考


### 练习11 作答
按照提示，使用 read_ebp函数，这道题比较简单

```c
int mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
        uint32_t ebp, *arg ;
        ebp = (uint32_t)(read_ebp());
        arg = (uint32_t *)ebp;
        
        cprintf("ebp %08x  eip %08x  args %08x %08x %08x %08x %08x\n", ebp,arg[1],arg[2],arg[3],arg[4],arg[5],arg[6]);
	return 0;
}
```

make 之后效果是这样的

```bash
ubuntu@ubuntu1604:~/6.828/lab$ make qemu-nox
sed "s/localhost:1234/localhost:26000/" < .gdbinit.tmpl > .gdbinit
***
*** Use Ctrl-a x to exit qemu
***
qemu-system-i386 -nographic -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::26000 -D qemu.log 
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
ebp f010ff18  eip f010007b  args 00000000 00000000 00000000 00000000 00000000
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 

```


### `练习 12`. 
修改堆栈回溯函数，为每个 eip 显示与该 eip 对应的函数名、源文件名和行号。
在 debuginfo_eip 中，__STAB_* 来自哪里？这个问题有很长的答案；为了帮助您找到答案，您可能需要执行以下操作：
- 在文件 kern/kernel.ld 中查找 __STAB_*
- 运行 objdump -h obj/kern/kernel
- 运行 objdump -G obj/kern/kernel
-  运行 gcc -pipe -nostdinc -O2 -fno-builtin -I。 -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c，然后查看init.s。
- 查看引导加载程序是否在内存中加载符号表作为加载内核二进制文件的一部分

通过插入对 stab_binsearch 的调用来查找地址的行号，完成 debuginfo_eip 的实现。

向内核监视器添加一个 backtrace 命令，并扩展 mon_backtrace 的实现以调用 debuginfo_eip 并为表单的每个堆栈帧打印一行：
```bash
K> backtrace
Stack backtrace:
  ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: monitor+106
  ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
         kern/init.c:49: i386\_init+59
  ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <unknown>+0
K>
```

每一行给出文件名和堆栈帧eip在该文件中的行，然后是函数名和eip从函数第一条指令的偏移量（例如，monitor+106表示返回eip是106字节超过监视器的开头）。

#### 练习12 作答
从 stab.h 中找到寻找行号的标记，把debuginfo_eip给补全，只要照着提示写代码就可以
`#define N\_SLINE 0x44 // text segment line number`

然后就可以修改mon_backtrace了
```c
int mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
        // Your code here.
        uint32_t ebp, *arg,ret;
        struct Eipdebuginfo info;
        ebp = (uint32_t)(read_ebp());
        arg = (uint32_t *)ebp;
        ret = debuginfo_eip(arg[1],&info);
        cprintf("ebp %08x  eip %08x  args %08x %08x %08x %08x %08x\n", ebp,arg[1],arg[2],arg[3],arg[4],arg[5],arg[6]);
        if(ret == 0)
                cprintf("%s:%d: %s%+%d\n",info.eip_file,info.eip_fn_namelen,info.eip_fn_name,arg[1]-info.eip_fn_addr);
        return 0;
}
```

打印出的效果就是这样的
```bash
ebp f010ff18  eip f010007b  args 00000000 00000000 00000000 00000000 f010091d
kern/init.c:14: test_backtrace:F(0,20)%+59
```

最后再按照commands[]的格式，把mon_backtrace添加进命令栏
```c
static struct Command commands[] = {
	{ "help", "Display this list of commands", mon_help },
	{ "kerninfo", "Display information about the kernel", mon_kerninfo },
        { "backtrace", "backtrace the stack", mon_backtrace },
};
```
