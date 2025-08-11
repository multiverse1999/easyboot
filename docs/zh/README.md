使用 Easyboot 启动内核
====================

[Easyboot](https://gitlab.com/bztsrc/easyboot) 是一个一体化启动管理器和可启动磁盘映像创建器，可以加载各种操作系统内核和各种格式的 Multiboot2 兼容内核。

[[_TOC_]]

安装
----

```
 easyboot [-v|-vv] [-s <mb>] [-b <mb>] [-u <guid>] [-p <t> <u> <i>] [-e] [-c] <indir> <outfile|device>

  -v, -vv         增加冗长/验证
  -s <mb>         设置磁盘映像大小（以兆字节为单位）（默认为 35M）
  -b <mb>         设置启动分区大小（以兆字节为单位）（默认为 33M）
  -u <guid>       设置启动分区的唯一标识符（默认为随机）
  -p <t> <u> <i>  添加额外的分区（类型 guid、唯一 guid、映像文件）
  -e              添加El Torito Boot Catalog（BIOS / EFI CDROM启动支持）
  -c              始终创建一个新的图像文件，即使它存在
  <indir>         使用该目录的内容作为启动分区
  <outfile>       输出图像或设备文件名
```

**Easyboot** 工具使用 GUID 分区表创建一个名为 `(outfile)` 的可启动磁盘映像，其中有一个分区格式为 FAT32，名为“EFI 系统分区”（简称 ESP）。
该分区的内容取自您提供的 `(indir)`。您必须在该目录中放置一个名为 `easyboot/menu.cfg` 的简单纯文本配置文件。使用 NL 或 CRLF 行尾，
您可以使用任何文本编辑器轻松编辑该文件。根据您的配置，您可能还需要此目录中名为 `easyboot/*.plg` 的一些 [插件](plugins.md)。

该映像也可以在 Raspberry Pi 上启动，并且可以在 qemu 中工作；但要在真机上启动，您还需要进一步的固件文件`bootcode.bin`、`fixup.dat`、`start.elf`和
`(indir)`目录中的 .dtb 文件，这些文件可以从 Raspberry Pi 的[官方存储库](https://github.com/raspberrypi/firmware/tree/master/boot)下载。

该工具还有一些可选的命令行标志：`-s (mb)` 以兆字节为单位设置生成的磁盘映像的整体大小，而 `-b (mb)` 以兆字节为单位设置启动分区的大小。显然前者必须大于后者。
如果未指定，则分区大小根据给定目录的大小计算（至少 33 Mb，最小的 FAT32），磁盘大小默认比该值大 2 Mb（由于分区表所需的对齐和空间）。
如果这两个大小值之间的差距超过 2 Mb，那么您可以使用第三方工具（如 `fdisk`）根据自己的喜好向映像添加更多分区（或参见下面的 `-p`）。如果您想要可预测的布局，
还可以使用 `-u <guid>` 标志指定启动分区的唯一标识符 (UniquePartitionGUID)。

您也可以选择使用 `-p` 标志添加额外的分区。这需要 3 个参数：(PartitionTypeGUID)、(UniquePartitionGUID) 和包含分区内容的映像文件的名称。
此标志可重复多次。

`-e` 标志将 El Torito 启动目录添加到生成的映像中，这样它不仅可以作为 USB 记忆棒启动，还可以作为 BIOS / EFI CDROM 启动。

如果 `(outfile)` 是设备文件（例如 Linux 上的 `/dev/sda`、BSD 上的 `/dev/disk0` 和 Windows 上的 `\\.\PhysicalDrive0`），则它不会创建 GPT 或 ESP，
而是定位设备上已经存在的文件。它仍会将 `(indir)` 中的所有文件复制到启动分区并安装加载程序。如果 `(outfile)` 是已经存在的映像文件，这也有效（在这种情况下，
使用 `-c` 标志始终先创建一个新的空映像文件）。

配置
----

`easyboot/menu.cfg` 文件可能包含以下几行（与 grub.cfg 的语法非常相似，您可以在 [此处](menu.cfg) 找到示例配置文件）：

### 评论

所有以`#`开头的行都将被视为注释，并会跳过直至行末。

### 详细程度

您可以使用以`verbose`开头的行来设置详细程度。

```
verbose (0-3)
```

这告诉加载器要打印多少信息到启动控制台。详细 0 表示完全安静（默认），详细 3 表示转储加载的内核段和入口点处的机器代码。

### 帧缓冲器

您可以使用以`framebuffer`开头的行来请求特定的屏幕分辨率。格式如下：

```
framebuffer （宽度）（高度）（每像素位数）[#(前景颜色)] [#(背景颜色)] [#(进度条背景颜色)]
```

**Easyboot** 将为您设置一个帧缓冲区，即使此行不存在（默认为 800 x 600 x 32bpp）。但如果此行存在，则它将尝试设置指定的分辨率。不支持调色板模式，
因此每像素位数必须至少为 15。

如果给出了第四个可选颜色参数，则它必须采用 HTML 符号，以井号开头，后跟 6 个十六进制数字 RRGGBB。例如，`#ff0000`是鲜红色，`#007f7f`是深青色。
它设置前景或换句话说字体的颜色。同样，如果给出了第五个可选颜色参数，它也必须采用 HTML 符号，并且它设置背景颜色。最后一个可选颜色参数也是如此，
它设置进度条的背景颜色。

当未指定颜色时，默认为黑底白字，进度条的背景为深灰色。

### 默认启动选项

以`default`开头的行设置在指定超时后无需用户交互即可启动哪个菜单项。

```
default （菜单项索引）（超时毫秒）
```

菜单项索引是 1 到 9 之间的数字。超时以毫秒（千分之一秒）为单位，因此 1000 表示一秒。

### 菜单对齐

以`menualign`开头的行改变了启动选项的显示方式。

```
menualign ("vertical"|"horizontal")
```

默认情况下菜单是水平的（`horizontal`），您可以将其更改为垂直的。

### 启动选项

最多可以指定 9 个菜单项，每行以 `menuentry` 开头。格式如下：

```
menuentry （图标）[标签]
```

对于每个图标，启动分区上必须存在一个 `easyboot/(图标).tga` 文件。图像必须采用运行长度编码、颜色映射的 [Targa 格式](../en/TGA.txt)，
因为这是压缩程度最高的变体（文件的前三个字节必须按此顺序为 `0`、`1` 和 `9`，请参阅规范中的数据类型 9）。其尺寸必须正好是 64 像素高和 64 像素宽。

要从 GIMP 保存此格式，首先选择“图像 > 模式 > 索引...”，在弹出窗口中将“最大颜色数”设置为 256。然后选择“文件 > 导出为...”，输入以 `.tga` 结尾的文件名，
并在弹出窗口中选中“RLE 压缩”。对于命令行转换工具，您可以使用 ImageMagick，`convert (any image file) -colors 256 -compress RLE icon.tga`。

可选的标签参数用于在菜单上显示 ASCII 版本或发布信息，而不是任意字符串，因此为了节省空间，不支持 UTF-8，除非您还提供 `easyboot/font.sfn`。
（UNICODE 字体需要大量存储空间，即使 [可缩放屏幕字体](https://gitlab.com/bztsrc/scalable-font2) 非常高效，也仍然大约有 820K。相比之下，
GRUB 的 unicode.pf2 要大得多，大约有 2392K，尽管这两种字体都包含 8x16 和 16x16 位图中的约 55600 个字形。嵌入的 ASCII 字体只有 8x16 位图和 96 个字形。）

所有在`menuentry`行之后的行都属于该菜单项，除非该行是另一个菜单项行。为方便起见，您可以像在 GRUB 中一样使用块，但这些只是语法糖。花括号被视为空格字符。
如果您愿意，您可以省略它们并使用制表符，就像在 Python 脚本或 Makefile 中一样。

例如
```
menuentry FreeBSD backup
{
    kernel bsd.old/boot
}
```

### 选择分区

以`partition`开头的行选择 GPT 分区。必须以`menuentry`行开头。

```
partition （分区唯一 UUID）
```

此分区将用作启动选项的根文件系统。内核以及模块将从此分区加载，并且根据启动协议，还将传递给内核。指定的分区可能位于与启动磁盘不同的磁盘上，**Easyboot**
将在启动期间遍历所有 GPT 分区磁盘以找到它。

为方便起见，还会在`kernel`行中查找分区（见下文）。如果给定的启动命令行包含`root=(UUID)`或`root=*=(UUID)`字符串，则无需单独的`partition`行。

当没有明确指定分区时，内核以及模块将从启动分区加载。

### 指定内核

以`kernel`开头的行说明应启动哪个文件以及使用哪些参数。必须以`menuentry`行开头。

```
kernel （内核文件的路径）（可选的启动命令行参数）
```

该路径必须指向现有文件、可执行内核二进制文件，并且必须是根（或启动）分区上的绝对 UTF-8 路径。如果内核不在分区的根目录中，则目录分隔符始终为`/`，
即使在 UEFI 系统上也是如此。如果名称包含空格，则必须使用`\`进行转义。路径后面可能跟着命令行参数，以空格分隔。对于符合 Multiboot2 的内核，
这些命令行参数将在 *启动命令行*（类型 1）标签中传递。它们不会被 **Easyboot** 更改，也不会被解析，除非搜索分区说明符。

默认情况下，**Easyboot** 可以启动 ELF64 和 PE32+/COFF 格式的 Multiboot2 兼容内核（在 UEFI 系统上，UEFI 应用程序也是如此）。通常该协议不允许高半内核，
但 **Easyboot** 稍微违反了协议，但不会破坏正常的非高半内核。如果您想启动任何其他内核，则需要一个内核加载器 [插件](plugins.md)。

注意：与 GRUB 不同，您必须使用特殊命令（如“linux”或“multiboot”）来选择启动协议，而这里只有一个命令，并且协议会在运行时从您的内核自动检测。

### 加载更多模块

您可以使用以“module”开头的行加载任意文件（初始 ramdisk、内核驱动程序等）以及内核。必须以“menuentry”行开头。请注意，此行可以在每个菜单项中重复多次。
如果启动协议支持 initrd，则第一个“module”行将被视为 initrd。

```
module （文件路径）（可选模块命令行参数）
```

该路径必须指向现有文件，并且必须是根（或启动）分区上的绝对 UTF-8 路径。它后面可能跟有空格分隔的命令行参数。如果文件已压缩，并且有解压缩 [插件](plugins.md)，
则模块将透明地解压缩。有关这些已加载（和未压缩）模块的信息将传递到 *模块*（类型 3）标签中的 Multiboot2 兼容内核。

特殊情况是模块以字节`DSDT`、`GUDT`或`0xD00DFEED`开头。在这些情况下，文件不会被添加到*模块*标签中，而是会修补 ACPI 表，以便其 DSDT 指针指向此文件的内容。
这样，您可以轻松地用用户提供的 ACPI 表替换有缺陷的 BIOS 的 ACPI 表。

### 启动画面徽标

如果您放置以`bootsplash`开头的行，您还可以在选择启动选项时在屏幕中央显示徽标。必须在`menuentry`行之前。

```
bootsplash [#(背景颜色)]（TGA 文件的路径）
```

背景颜色是可选的，并且必须采用 HTML 符号，以井号开头，后跟 6 个十六进制数字 RRGGBB。如果第一个参数不是以`#`开头，则假定为路径参数。

该路径必须指向现有文件，并且必须是启动（而非根）分区上的绝对 UTF-8 路径。图像必须采用运行长度编码、颜色映射 Targa 格式，就像菜单图标一样。
尺寸可以是任何适合屏幕的尺寸。

### 多核支持

要同时在所有处理器核心上启动内核，请指定 `multicore` 指令（仅限 64 位内核）。必须在`menuentry`行之前。

```
multicore
```

故障排除
-------

如果遇到任何问题，只需使用`easyboot -vv`标志运行即可。这将执行验证并在映像创建时详细输出结果。否则，将`verbose 3`添加到`easyboot/menu.cfg`
以获取详细的启动时间消息。

如果您在左上角看到带有红色背景的`PMBR-ERR`字符串，则表示您的 CPU 非常旧并且不支持 64 位长模式，或者引导扇区无法引导加载程序。可能只会在 BIOS 机器上发生，
在 UEFI 或 RaspberryPi 上永远不会发生这种情况。

| 留言                                 | 描述                                                                              |
|-------------------------------------|-----------------------------------------------------------------------------------|
| `Booting [X]...`                    | 表示选择了启动选项（菜单项索引）X                                                      |
| `Loading 'X' (Y bytes)...`          | 正在加载长度为 Y 的文件 X                                                             |
| `Parsing kernel...`                 | 内核已找到，现在检测其格式                                                            |
| `Starting X boot...`                | 表明检测到了X系统的引导加载程序                                                        |
| `Starting X kernel...`              | 说明检测到了X系统的内核                                                              |
| `Transfering X control to Y`        | 表示 Y 地址处的 X 模式入口点即将被调用                                                  |

如果您在看到最后一条消息后遇到任何问题，则意味着问题发生在操作系统的启动过程中，而不是 **Easyboot** 加载程序中，因此您必须查阅给定操作系统的文档以找到答案。
否则，请随时在 gitlab 上打开 [issue](https://gitlab.com/bztsrc/easyboot/-/issues)。

### Multiboot1

所需插件：[grubmb1](../../src/plugins/grubmb1.c)

### Multiboot2

这是最灵活的，通过插件支持多种变体：

- 带有简化的 Multiboot2 的 ELF64 或 PE32+：无需插件
- 具有简化的 Multiboot2 和 32 位入口点的 ELF32：[elf32](../../src/plugins/elf32.c)
- a.out（struct exec）带有简化的 Multiboot2 和 32 位入口点：[aout](../../src/plugins/aout.c)
- 具有 32 位入口点的 GRUB 兼容 Multiboot2：[grubmb2](../../src/plugins/grubmb2.c)

注意区别：[简化的 Multiboot2](ABI.md) 不需要嵌入标签，支持高半内核，干净的 64 位入口点，其参数根据 Multiboot2、SysV 和 fastcall ABI 传递。

另一方面，[与 GRUB 兼容的 Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) 需要嵌入标签，不支持高半部和 64
位内核，入口点始终是 32 位，并且参数仅在 `eax`、`ebx` 中传递。

### Windows

无需插件，但必须设置 [Secure Boot](secureboot.md)。

```
menuentry win {
  kernel EFI/Microsoft/BOOT/BOOTMGRW.EFI
}
```

### Linux

所需插件：[linux](../../src/plugins/linux.c), [ext234](../../src/plugins/ext234.c)

如果内核没有放在启动分区，那么您可以使用命令行指定根分区。

```
menuentry linux {
  kernel vmlinuz-linux root=PARTUUID=01020304-0506-0708-0a0b0c0d0e0f1011
}
```

### OpenBSD

所需插件：[obsdboot](../../src/plugins/obsdboot.c), [ufs44](../../src/plugins/ufs44.c)

```
menuentry openbsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot
}
```

警告！如果您正在启动 OpenBSD，请不要使用 [elf32](../../src/plugins/elf32.c) 插件！它的 `boot` 错误地声称是具有 32 位 SysV ABI 入口点的 ELF，
但实际上它有一个 16 位实模式入口。

### FreeBSD

所需插件：[fbsdboot](../../src/plugins/fbsdboot.c), [ufs2](../../src/plugins/ufs2.c)

在传统 BIOS 系统上，指定加载器`boot`。

```
menuentry freebsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot/boot
}
```

在 UEFI 机器上，在启动分区上使用`loader.efi`（无需插件）。

```
menuentry freebsd {
  kernel boot/loader.efi
}
```

如果您的根文件系统是 ZFS，则将此文件（BIOS 上的`boot`、UEFI 上的`loader.efi`）复制到`(indir)`，并且不要指定根分区。

### FreeDOS

所需插件：[fdos](../../src/plugins/fdos.c)

将 FreeDOS 的文件移动到 `(indir)`（FreeDOS 将使用启动分区作为根分区）。

```
menuentry freedos {
  kernel KERNEL.SYS
}
```

如果在打印版权和 `-InitDisk` 后启动停止，则 FreeDOS 内核编译时不支持 FAT32。请下载名称中带有 `f32` 的其他内核。

### ReactOS

所需插件：[reactos](../../src/plugins/reactos.c)

```
menuentry reactos {
  kernel FREELDR.SYS
}
```

### MenuetOS

所需插件：[menuet](../../src/plugins/menuet.c)

```
menuentry menuetos {
  kernel KERNEL.MNT
  module CONFIG.MNT
  module RAMDISK.MNT
}
```

要关闭自动配置，请在命令行中添加`noauto`。

### KolibriOS

所需插件：[kolibri](../../src/plugins/kolibri.c)

```
menuentry kolibrios {
  kernel KERNEL.MNT syspath=/rd/1/ launcher_start=1
  module KOLIBRI.IMG
  module DEVICES.DAT
}
```

该插件也适用于 UEFI 机器，但您也可以在启动分区上使用`uefi4kos.efi`（无需插件）。

### SerenityOS

所需插件：[grubmb1](../../src/plugins/grubmb1.c)

```
menuentry serenityos {
  kernel boot/Prekernel
  module boot/Kernel
}
```

### Haiku

所需插件：[grubmb1](../../src/plugins/grubmb1.c), [befs](../../src/plugins/befs.c)

```
menuentry haiku {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel system/packages/haiku_loader-r1~beta4_hrev56578_59-1-x86_64.hpkg
}
```

在 UEFI 机器上，在启动分区上使用`haiku_loader.efi`（无需插件）。

### OS/Z

```
menuentry osz {
  kernel ibmpc/core
  module ibmpc/initrd
}
```

无需插件。
