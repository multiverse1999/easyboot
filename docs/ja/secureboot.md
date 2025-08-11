UEFI Secure Bootによる Easyboot
===============================

まず第一に、これは詐欺であり、安全という名目だけのものなので、できる限り避けてください (CVE-2016-332、CVE-2016-3287、CVE-2020-14372、CVE-2020-20233、
CVE-2020-25632、CVE-2020-25647、CVE-2020-27779、CVE-2020-27749、CVE-2021-3418、CVE-2021-20233、CVE-2021-20225、CVE-2022-21894、
CVE-2022-34301、CVE-2022-34302、CVE-2022-34303 などを参照)。

何らかの理由で回避できない場合は、次の手順に従ってください。

前提条件
--------

1. 最新の [shim バイナリ](https://kojipkgs.fedoraproject.org/packages/shim) をダウンロードし、RPM から `shimx64.efi` および `mmx64.efi` ファイルを抽出します。
2. 通常どおり、ディスクに **Easyboot** をインストールします。

セットアップ Secure Boot
------------------------

1. ESP パーティションで `EFI\BOOT\BOOTX64.EFI` の名前を `EFI\BOOT\GRUBX64.EFI` に変更します。
2. `shimx64.efi` を `EFI\BOOT\BOOTX64.EFI` として ESP にコピーします。
3. `mmx64.efi` を `EFI\BOOT\MMX64.EFI` として ESP にコピーします。
4. 再起動し、UEFI 構成メニューに入ります。
5. ブート順序に `EFI\BOOT\BOOTX64.EFI` があるかどうかを確認し、ない場合は追加します。
6. セキュア ブートを有効にして構成を終了します。
7. Shim は MokManager を起動します。
8. 選択 `Enroll hash from disk`.
9. `EFI\BOOT\GRUBX64.EFI` を指定して、MokList に追加します。
10. 選択 `Continue boot`.

この後 (およびその後の再起動のたびに)、**Easyboot** は Secure Boot モードで起動するはずです。


