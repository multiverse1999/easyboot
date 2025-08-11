Easyboot 具有 UEFI 安全启动功能
============================

首先，尽可能避免，因为这是一个骗局，只能以其名义进行安全保护（见 CVE-2016-332, CVE-2016-3287, CVE-2020-14372, CVE-2020-20233, CVE-2020-25632,
CVE-2020-25647, CVE-2020-27779, CVE-2020-27749, CVE-2021-3418, CVE-2021-20233, CVE-2021-20225, CVE-2022-21894, CVE-2022-34301,
CVE-2022-34302, CVE-2022-34303, 等等).

如果您因某种原因无法避免，请按照以下步骤操作。

先决条件
-------

1. 下载最新的 [shim 二进制文件](https://kojipkgs.fedoraproject.org/packages/shim)并从 RPM 中提取`shimx64.efi`和`mmx64.efi`文件。
2. 像往常一样在您的磁盘上安装 **Easyboot**。

设置安全启动
----------

1. 在 ESP 分区上将`EFI\BOOT\BOOTX64.EFI`重命名为`EFI\BOOT\GRUBX64.EFI`。
2. 将`shimx64.efi`复制到 ESP 作为`EFI\BOOT\BOOTX64.EFI`。
3. 将`mmx64.efi`复制到 ESP 作为`EFI\BOOT\MMX64.EFI`。
4. 重新启动，进入UEFI配置菜单。
5. 检查启动顺序是否有`EFI\BOOT\BOOTX64.EFI`，如果缺少则添加。
6. 启用安全启动并退出配置。
7. Shim 将启动 MokManager。
8. 选择 `Enroll hash from disk`.
9. 指定`EFI\BOOT\GRUBX64.EFI`并将其添加到MokList。
10. 选择 `Continue boot`.

此后（以及每次后续重启）**Easyboot** 应该以安全启动模式启动。
