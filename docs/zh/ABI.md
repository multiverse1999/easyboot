编写 Easyboot 兼容内核
====================

[Easyboot](https://gitlab.com/bztsrc/easyboot) 使用 [插件](plugins.md) 支持各种内核。但如果未找到合适的插件，它会回退到 ELF64 或 PE32+ 二进制文件，
并使用 Multiboot2 协议的简化版本（无需嵌入任何内容）。

这是 [Simpleboot](https://gitlab.com/bztsrc/simpleboot) 使用的相同协议，该 repo 中的所有示例内核也必须与 **Easyboot** 兼容。

您可以使用 GRUB 存储库中的原始 multiboot2.h 标头，或 [easyboot.h](../../easyboot.h) C/C++ 标头文件来更轻松地使用 typedef。低级二进制格式相同，
您还可以使用任何现有的 Multiboot2 库，即使使用非 C 语言，例如这个 [Rust](https://github.com/rust-osdev/multiboot2/tree/main/multiboot2/src) 库
（注意：我与这些开发人员没有任何关系，我只是搜索了“Rust Multiboot2”，这是第一个结果）。

[[_TOC_]]

启动顺序
-------

### 引导加载程序

在 *BIOS* 机器上，磁盘的第一个扇区由固件加载到 0:0x7C00，并将控制权交给它。在此扇区中，**Easyboot** 有 [boot_x86.asm](../../src/boot_x86.asm)，
它足够智能，可以定位和加载第二阶段加载器，并为其设置长模式。

在 *UEFI* 机器上，固件直接加载相同的第二阶段文件，称为“EFI/BOOT/BOOTX64.EFI”。此加载器的源代码可在 [loader_x86.c](../../src/loader_x86.c) 中找到。
就是这样，**Easyboot** 不是 GRUB 也不是 syslinux，这两者都需要磁盘上的数十个系统文件。这里不需要其他文件，只需要这一个（插件是可选的，
不需要提供 Multiboot2 兼容性）。

在 *Raspberry Pi* 上，加载器被称为 `KERNEL8.IMG`，由 [loader_rpi.c](../../src/loader_rpi.c) 编译而成。

### 加载器

此加载程序经过精心编写，可在多种配置下运行。它从磁盘加载 GUID 分区表，并查找“EFI 系统分区”。找到后，它会在该启动分区上查找“easyboot/menu.cfg”配置文件。
选择启动选项并知道内核的文件名后，加载程序会找到并加载它。

然后它会自动检测内核的格式，并且足够智能，能够解释关于在何处加载什么的节和段信息（它会在必要时进行按需内存映射）。然后它根据检测到的启动协议
（Multiboot2 / Linux / 等受保护或长模式、ABI 参数等）设置适当的环境。在机器状态稳定且定义明确后，作为最后一幕，加载器会跳转到内核的入口点。

机器状态
-------

[Multiboot2 规范](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)中关于机器状态的所有内容均有效，通用寄存器除外。**Easyboot**
根据 SysV ABI 和 Microsoft fastcall ABI 将两个参数传递给内核的入口点。第一个参数是魔法，第二个参数是物理内存地址，指向多重引导信息标记列表（以下缩写为 MBI，见下文）。

我们还稍微违反了 Multiboot2 协议来处理高半部内核。Multiboot2 要求内存必须是身份映射的。好吧，在 **Easyboot** 下，这只是部分正确：我们只保证所有物理 RAM
都按预期进行了身份映射；但是，高于该区域的一些区域（取决于内核的程序头）可能仍然可用。这不会破坏正常的 Multiboot2 兼容内核，这些内核不应该访问可用物理 RAM
之外的内存。

您的内核在 BIOS 和 UEFI 系统以及 RPi 上的加载方式完全相同，固件差异只是“别人的问题”。您的内核唯一能看到的是 MBI 是否包含 EFI 系统表标签。为了简化您的操作，**Easyboot**
也不会生成 EFI 内存映射（类型 17）标签，它仅在所有平台上无差别地提供 [内存映射](#memory_map)（类型 6）标签（在 UEFI 系统上也是如此，内存映射只是为您转换，
因此您的内核只需处理一种内存列表标签）。旧的、过时的标签也被省略，并且永远不会由此引导管理器生成。

内核在管理程序级别运行（x86 上的 ring 0，ARM 上的 EL1），可能在所有 CPU 内核上并行运行。

GDT 未指定，但有效。堆栈设置在前 640k 中，并向下增长（但您应尽快将其更改为您认为值得的任何堆栈）。启用 SMP 后，所有内核都有自己的堆栈，内核 ID 位于堆栈顶部
（但您也可以使用 cpuid / mpidr / 等以通常的平台特定方式获取内核 ID）。

您应该将 IDT 视为未指定；IRQ、NMI 和软件中断已禁用。设置虚拟异常处理程序以显示一些最小转储并停止机器。这些仅应依赖于报告您的内核是否在您能够设置自己的 IDT
和处理程序之前发生严重破坏，最好尽快设置。在 ARM 上，vbar_el1 设置为调用相同的虚拟异常处理程序（尽管它们当然会转储不同的寄存器）。

帧缓冲区也是默认设置的。您可以在配置中更改分辨率，但如果未指定，则仍会配置帧缓冲区。

永远不要从内核返回，这一点很重要。您可以自由地覆盖内存中加载器的任何部分（只要您完成了 MBI 标签），因此根本无处可返回。
"Der Mohr hat seine Schuldigkeit getan, der Mohr kann gehen."

传递到内核的启动信息（MBI）
----------------------

乍一看并不明显，但 Multiboot2 规范实际上定义了两个完全独立的标签集：

- 第一组应该内联在 Multiboot2 兼容内核中，称为 OS 映像的 Multiboot2 标头（第 3.1.2 节），因此*由内核提供*。**Easyboot** 不关心这些标签，
  也不会解析内核中的这些标签。您根本不需要任何特殊的神奇数据嵌入到内核文件中，因为 **Easyboot**、ELF 和 PE 标头就足够了。

- 第二组在启动时动态*传递给内核*，**Easyboot** 仅使用这些标签。但是它不会生成 Multiboot2 指定的所有内容（它只是省略了旧的、过时的或遗留的标签）。
  这些标签称为 MBI 标签，请参阅[启动信息](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Boot-information-format)
  （第 3.6 节）。

注意：Multiboot2 规范中关于 MBI 标签的错误非常多。您可以在下面找到一个修复版本，它与 GRUB 源代码存储库中的 multiboot2.h 头文件一致。

内核的第一个参数是魔法 0x36d76289（在 `rax`、`rcx` 和 `rdi` 中）。您可以使用第二个参数（在 `rbx`、`rdx` 和 `rsi` 中）定位 MBI 标签。在​​ ARM 平台上，
魔法在 `x0` 中，地址在 `x1` 中。在 RISC-V 和 MIPS 上分别使用 `a0` 和 `a1`。如果将此加载器移植到另一个架构，则必须始终使用 SysV ABI 为函数参数指定的寄存器。
如果平台上有其他不干扰 SysV ABI 的常见 ABI，则值也应该在这些 ABI 的寄存器中（或在堆栈顶部）复制。

### 标头

传递的地址始终是 8 字节对齐的，并以 MBI 标头开头：

```
        +-------------------+
u32     | total_size        |
u32     | reserved          |
        +-------------------+
```

`total_size` 是标签列表的总大小。后面跟着一系列同样 8 字节对齐的标签。每个标签都以以下标签头字段开头：

```
        +-------------------+
u32     | type              |
u32     | size              |
        +-------------------+
```

`type` 包含标签其余部分内容的标识符。`size` 包含标签的大小（包括标头字段，但不包括填充）。标签会逐个填充，以便每个标签都从 8 字节对齐的地址开始。

### 终结者

```
        +-------------------+
u32     | type = 0          |
u32     | size = 8          |
        +-------------------+
```

标签以类型`0`和大小`8`的标签终止。

### 启动命令行

```
        +-------------------+
u32     | type = 1          |
u32     | size              |
u8[n]   | string            |
        +-------------------+
```

`string` 包含 *menuentry* 的 `kernel` 行中指定的命令行（不包含内核的路径和文件名）。该命令行是一个普通的 C 风格零结尾的 UTF-8 字符串。

### 引导加载程序名称

```
        +----------------------+
u32     | type = 2             |
u32     | size = 17            |
u8[n]   | string "Easyboot"    |
        +----------------------+
```

`string` 包含启动内核的引导加载程序的名称。该名称是一个普通的 C 样式 UTF-8 零终止字符串。

### 模块

```
        +-------------------+
u32     | type = 3          |
u32     | size              |
u32     | mod_start         |
u32     | mod_end           |
u8[n]   | string            |
        +-------------------+
```

此标记向内核指示哪个引导模块与内核映像一起加载，以及在哪里可以找到它。`mod_start` 和 `mod_end` 包含引导模块本身的起始和终止物理地址。您永远不会得到 gzip
压缩缓冲区，因为 **Easyboot** 会为您透明地解压缩这些缓冲区（如果您提供插件，也可以处理除 gzip 压缩数据以外的数据）。`string` 字段提供与该特定引导模块关联的任意字符串；
它是一个普通的 C 样式的以零结尾的 UTF-8 字符串。在 *menuentry* 的 `module` 行中指定，其确切用途特定于操作系统。与引导命令行标记不同，模块标记 *还包含* 模块的路径和文件名。

每个模块出现一个标签。此标签类型可能出现多次。如果初始 ramdisk 与内核一起加载，则它将作为第一个模块出现。

有一种特殊情况，如果文件是 DSDT ACPI 表、FDT（dtb）或 GUDT blob，那么它就不会作为模块出现，而是 ACPI 旧 RSDP（类型 14）或 ACPI 新 RSDP（类型 15）将被修补，
并且它们的 DSDT 将被该文件的内容替换。

### 内存映射

该标签提供内存映射。

```
        +-------------------+
u32     | type = 6          |
u32     | size              |
u32     | entry_size = 24   |
u32     | entry_version = 0 |
varies  | entries           |
        +-------------------+
```

`size` 包含所有条目（包括该字段本身）的大小。`entry_size` 始终为 24。`entry_version` 设置为 `0`。
每个条目具有以下结构：

```
        +-------------------+
u64     | base_addr         |
u64     | length            |
u32     | type              |
u32     | reserved          |
        +-------------------+
```

`base_addr` 是起始物理地址。`length` 是内存区域的大小（以字节为单位）。`type` 是表示的地址范围的种类，其中值 `1` 表示可用 RAM，值 `3` 表示保存 ACPI
信息的可用内存，值 `4` 表示需要在休眠时保留的保留内存，值 `5` 表示由有缺陷的 RAM 模块占用的内存，所有其他值当前都表示保留区域。`reserved` 在 BIOS
启动时设置为 `0`。

当 MBI 在 UEFI 机器上生成时，各种 EFI 内存映射条目将存储为类型`1`（可用 RAM）或`2`（保留 RAM），如果需要，原始 EFI 内存类型将放置在`reserved`字段中。

提供的映射保证列出所有可供正常使用的标准 RAM，并且始终按`base_addr`升序排列。但是，此可用 RAM 类型包括内核、mbi、段和模块占用的区域。
内核必须注意不要覆盖这些区域（**Easyboot** 可以轻松排除这些区域，但这会破坏 Multiboot2 兼容性）。

### 帧缓冲区信息

```
        +----------------------------------+
u32     | type = 8                         |
u32     | size = 38                        |
u64     | framebuffer_addr                 |
u32     | framebuffer_pitch                |
u32     | framebuffer_width                |
u32     | framebuffer_height               |
u8      | framebuffer_bpp                  |
u8      | framebuffer_type = 1             |
u16     | reserved                         |
u8      | framebuffer_red_field_position   |
u8      | framebuffer_red_mask_size        |
u8      | framebuffer_green_field_position |
u8      | framebuffer_green_mask_size      |
u8      | framebuffer_blue_field_position  |
u8      | framebuffer_blue_mask_size       |
        +----------------------------------+
```

字段`framebuffer_addr`包含帧缓冲区的物理地址。字段`framebuffer_pitch`包含一行的长度（以字节为单位）。字段`framebuffer_width`和`framebuffer_height`
包含帧缓冲区的尺寸（以像素为单位）。字段`framebuffer_bpp`包含每个像素的位数。在当前版本的规范中，`framebuffer_type`始终设置为 1，而`reserved`始终包含 0，
并且必须被 OS 映像忽略。其余字段描述了打包像素格式、通道的位置和大小（以位为单位）。您可以使用表达式`((~(0xffffffff << size)) << position) & 0xffffffff`
来获取类似 UEFI GOP 的通道掩码。

### EFI 64位系统表指针

此标签仅在 UEFI 机器上运行 **Easyboot** 时存在。在 BIOS 机器上，此标签永远不会生成。

```
        +-------------------+
u32     | type = 12         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

该标签包含指向 EFI 系统表的指针。

### EFI 64 位图像句柄指针

此标签仅在 UEFI 机器上运行 **Easyboot** 时存在。在 BIOS 机器上，此标签永远不会生成。

```
        +-------------------+
u32     | type = 20         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

此标签包含指向 EFI 映像句柄的指针。通常它是引导加载程序映像句柄。

### SMBIOS 表

```
        +-------------------+
u32     | type = 13         |
u32     | size              |
u8      | major             |
u8      | minor             |
u8[6]   | reserved          |
        | smbios tables     |
        +-------------------+
```

该标签包含 SMBIOS 表及其版本的副本。

### ACPI 旧 RSDP

```
        +-------------------+
u32     | type = 14         |
u32     | size              |
        | copy of RSDPv1    |
        +-------------------+
```

此标签包含根据 ACPI 1.0 规范定义的 RSDP 副本。（具有 32 位地址。）

### ACPI 新 RSDP

```
        +-------------------+
u32     | type = 15         |
u32     | size              |
        | copy of RSDPv2    |
        +-------------------+
```

此标签包含根据 ACPI 2.0 或更高版本规范定义的 RSDP 副本。（可能带有 64 位地址。）

这些（类型 14 和 15）指向一个 `RSDT` 或 `XSDT` 表，并带有一个指向 `FACP` 表的指针，而 `FACP` 表又包含两个指向 `DSDT` 表的指针，该表描述了机器。**Easyboot**
会在原本不支持 ACPI 的机器上伪造这些表。此外，如果您提供 DSDT 表、FDT（dtb）或 GUDT 二进制文件作为模块，则 **Easyboot** 将修补指针以指向该用户提供的表。
要解析这些表，您可以使用我的无依赖、单头 [hwdet](https://gitlab.com/bztsrc/hwdet) 库（或臃肿的 [apcica](https://github.com/acpica/acpica) 和
[libfdt](https://github.com/dgibson/dtc))。

内核特定标签
----------

`type` 大于或等于 256 的标签不属于 Multiboot2 规范，但由 **Easyboot** 提供。如果内核需要这些标签，它们可能会由可选的 [plugins](plugins.md) 添加到列表中。

### EDID

```
        +-------------------+
u32     | type = 256        |
u32     | size              |
        | copy of EDID      |
        +-------------------+
```

该标签包含根据 EDID 规范支持的显示器分辨率列表的副本。

### SMP

```
        +-------------------+
u32     | type = 257        |
u32     | size              |
u32     | numcores          |
u32     | running           |
u32     | bspid             |
        +-------------------+
```

如果给出了`multicore`指令，则此标记存在。`numcores`包含系统中的 CPU 核心数，`running`是已成功初始化并并行运行同一内核的核心数。`bspid`包含 BSP
核心的标识符（在 x86 lAPIC id 上），以便内核可以区分 AP 并在这些 AP 上运行不同的代码。所有 AP 都有自己的堆栈，堆栈顶部是当前核心的 id。

### 分区标识符

```
        +-------------------+
u32     | type = 258        |
u32     | size = 24 / 40    |
u128    | bootuuid          |
u128    | rootuuid          |
        +-------------------+
```

此标签包含启动和根分区的 GPT 中的唯一标识符字段。如果启动不使用 GUID 分区表，则`bootuuid`生成为 `54524150-(设备代码)-(分区号)-616F6F7400000000`。

内存布局
-------

### BIOS 机器

| 开始      | 结束    | 描述                                                          |
|---------:|--------:|--------------------------------------------------------------|
|      0x0 |   0x400 | 中断向量表（可用，实模式 IDT）                                    |
|    0x400 |   0x4FF | BIOS 数据区（可用）                                            |
|    0x4FF |   0x500 | BIOS 启动驱动器代码（最有可能是 0x80，可用）                       |
|    0x500 |   0x5A0 | SMP 的同步数据（可用）                                          |
|    0x5A0 |  0x1000 | 异常处理程序堆栈（设置 IDT 后可用）                                |
|   0x1000 |  0x8000 | 分页表（设置分页表后可用）                                       |
|   0x8000 | 0x20000 | 加载程序代码和数据（设置 IDT 后可用）                              |
|  0x20000 | 0x40000 | 配置 + 标签（MBI 解析后可用）                                    |
|  0x40000 | 0x90000 | 插件 ID；从上到下：内核堆栈                                       |
|  0x90000 | 0x9A000 | 仅限 Linux 内核：zero page + cmdline                           |
|  0x9A000 | 0xA0000 | 扩展 BIOS 数据区（最好不要碰）                                    |
|  0xA0000 | 0xFFFFF | VRAM 和 BIOS ROM（不可用）                                      |
| 0x100000 |       x | 内核段，然后是模块，每页对齐                                      |

### UEFI 机器

没人知道。UEFI 会随意分配内存。可以分配任何东西。所有区域肯定会在内存映射中列为类型 = 1（`MULTIBOOT_MEMORY_AVAILABLE`）
和保留 = 2（`EfiLoaderData`），但这并不是唯一的，其他类型的内存也可能以这样的方式列出（例如启动管理器的 bss 部分）。

### Raspberry Pi

| 开始      | 结束    | 描述                                                                 |
|---------:|--------:|---------------------------------------------------------------------|
|      0x0 |   0x500 | 由固件保留（最好不要触碰）                                              |
|    0x500 |   0x5A0 | SMP 的同步数据（可用）                                                 |
|    0x5A0 |  0x1000 | 异常处理程序堆栈（设置 VBAR 后可用）                                      |
|   0x1000 |  0x9000 | 分页表（设置分页表后可用）                                              |
|   0x9000 | 0x20000 | 加载器代码和数据（设置 VBAR 后可用）                                      |
|  0x20000 | 0x40000 | 配置 + 标签（MBI 解析后可用）                                           |
|  0x40000 | 0x80000 | 固件提供的 FDT（dtb）；从上到下：内核的堆栈                                |
| 0x100000 |       x | 内核段，然后是模块，每页对齐                                             |

前几个字节是为 [armstub](https://github.com/raspberrypi/tools/blob/master/armstubs/armstub8.S) 保留的。只有核心 0 启动，因此要启动应用处理器，
请将函数的地址写入 0xE0（核心 1）、0xE8（核心 2）、0xF0（核心 3），这些地址位于此区域中。当使用`multicore`指令时，这无关紧要，
因为所有核心都将执行内核。

尽管 RPi 本身不支持，但您仍会获得带有虚假表的 ACPI 旧 RSDP（类型 14）标签。`APIC` 表用于将可用 CPU 核心的数量传达给内核。启动函数地址存储在
 RSD PTR -> RSDT -> APIC -> cpu\[x].apic_id 字段中（核心 ID 存储在 cpu\[x].acpi_id 中，其中 BSP 始终为 cpu\[0].acpi_id = 0 和
 cpu\[0].apic_id = 0xD8。请注意，“acpi”和“apic”看起来非常相似）。

如果固件传递了有效的 FDT blob，或者其中一个模块是 .dtb、.gud 或 .aml 文件，则还会添加 FADT（带有魔法“FACP”）表。在此表中，DSDT 指针（32 位，
偏移量为 40）指向提供的扁平设备树二进制文件。

尽管固件没有提供内存映射功能，但您仍会获得一个内存映射（类型 6）标签，其中列出了检测到的 RAM 和 MMIO 区域。您可以使用它来检测 MMIO 的基址，
该地址在 RPi3 和 RPi4 上有所不同。
