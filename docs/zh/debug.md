调试内核
=======

如果您的内核出现严重问题，默认情况下 **Easyboot** 将帮助您显示有关出现问题的原因和位置的详细信息。

![](../exc.png)

这足以给你一个线索，但它不允许您以交互方式进行调查。为此，您需要一个调试器。

Mini Debugger
-------------

优点：

- 易于设置
- 便于使用
- 也可以在真实硬件和虚拟机上运行

缺点：

- 顾名思义，最小功能集

要启用 [Mini Debugger](https://gitlab.com/bztsrc/minidbg)，只需通过将相应的 `minidbg_(arch).plg` 文件复制到启动分区来安装 **Easyboot**
[插件](plugins.md)。就是这样。它通过串行线路提供视频终端接口（波特率为 115200、数据位为 8、停止位为 1、无奇偶校验）。

在真实硬件上，连接 VT100 或 VT220 终端，或通过运行 **PuTTY**（Windows）或 **minicom**（Linux）的串行电缆连接另一台机器。

对于虚拟机，只需使用 `-serial stdio` 参数运行 qemu，您就可以从用于运行 qemu 的同一窗口控制调试器。例如：

```
qemu-system-x86_64 -serial stdio -hda disk.img
```

任何时候你的内核崩溃，你都会立即得到一个调试器提示，你可以检查情况。在调试器提示中，输入`?`或`h`以获取帮助。要明确调用调试器，请在代码中插入`int 3`
（0xCC 字节）。我建议将以下定义添加到你的内核：

```c
/* x86 */
#define breakpoint __asm__ __volatile__("int $3")
/* ARM */
#define breakpoint __asm__ __volatile__("brk #0")
```

```
Mini debugger by bzt
Exception 03: Breakpoint instruction, code 0
rax: 0000000000000000  rbx: 00000000000206C0  rcx: 000000000000270F
rdx: 00000000000003F8  rsi: 00000000000001B0  rdi: 0000000000102336
rsp: 000000000008FFB8  rbp: 000000000008FFF8   r8: 0000000000000004
 r9: 0000000000000002  r10: 0000000000000000  r11: 0000000000000003
r12: 0000000000000000  r13: 0000000000000000  r14: 0000000000000000
r15: 0000000000000000
> ?
Mini debugger commands:
  ?/h		this help
  c		continue execution
  n		move to the next instruction
  r		dump registers
  x [os [oe]]	examine memory from offset start (os) to offset end (oe)
  i [os [oe]]	disassemble instructions from offset start to offset end
> i pc-1 pc+4
00101601: CC                             int	3
00101602: FA                             cli
00101603: F4                             hlt
00101604: EB FC                          jmp	101602h
00101606: 90                              1 x nop
>
```

Qemu Debugger
-------------

优点：

- 无需设置，开箱即可使用
- 显示机器的内部状态，这是其他调试器无法做到的

缺点：

- 真的很难用
- 仅适用于虚拟机

当您的虚拟机正在运行时，从菜单中选择`查看`>`compatmonitor0`，或单击窗口以抓取焦点并按<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>2</kbd>（要释放焦点，
请按<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>G</kbd>）。

GDB
---

优点：

- 功能齐全的调试器
- 提供您能想到的所有功能

缺点：

- 设置起来很棘手
- 仅适用于虚拟机

在做任何事情之前，首先修改您的构建环境以生成两个内核文件。一个带有调试符号，另一个不带。这很重要，因为调试符号很容易占用大量空间，可能几兆字节。首先，
使用 `-g` 标志编译您的内核。然后在编译完成后，复制您的内核 `cp mykernel.elf mykernel_sym.elf`，并使用 `strip mykernel.elf` 删除调试信息。此后，
您将启动 `mykernel.elf`，并将 `mykernel_sym.elf` 提供给 gdb。

接下来，创建一个名为`gdb.rc`的 gdb 脚本。使用以下内容作为模板：

```
target remote localhost:1234
set architecture i386:x86-64
symbol-file mykernel_sym.elf
layout split
layout src
layout regs
break *_start
continue
```

这会将 gdb 连接到虚拟机，告诉它机器的类型，加载调试信息，配置布局，在内核的入口点设置断点，最后启动虚拟机。

完成此设置后，您可以开始调试。一个调试会话如下：在一个终端中，使用 `-s -S` 标志启动 qemu。它将挂起。例如：

```
qemu-system-x86_64 -s -S -hda disk.img
```

然后在另一个终端中，使用我们之前创建的脚本启动 gdb：

```
gdb -w -x gdb.rc
```

这将在一个窗口中显示所有内容，如下所示：

![](../gdb.png)

如果您已经做到了这一步，那么恭喜您，您可以开始使用内核进行真正的工作了！
