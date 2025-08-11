Easyboot
========

[Easyboot](https://gitlab.com/bztsrc/easyboot) is een alles-in-één boot *manager* en maker van opstartbare schijfkopieën die
verschillende OS-kernels en Multiboot2-compatibele kernels in verschillende binaire formaten kan laden.

![Easyboot](docs/screenshot.png)

LET OP: Als je op zoek bent naar een boot *loader* die slechts één kernel laadt, kijk dan eens naar het kleine broertje van
**Easyboot**, [Simpleboot](https://gitlab.com/bztsrc/simpleboot).

Erkenning
---------

Dit project zou niet mogelijk zijn geweest zonder de steun van [Free Software Foundation Hongarije](https://fsf.hu/nevjegy).
Het project is gemaakt in het kader van FSF.hu's Free Software Tender 2023.

Voordelen ten opzichte van GRUB
-------------------------------

- afhankelijkheidsvrij, eenvoudig te gebruiken, meertalige installatie
- niet opgeblazen, het is slechts ongeveer 0,5% van de grootte van GRUB
- eenvoudig te configureren, met syntaxiscontrole
- eenvoudig ACPI-tabellen patchen met door de gebruiker verstrekte DSDT
- toont een zeer gedetailleerd foutscherm als de kernel in een vroeg stadium uitvalt
- Multiboot2: schoon 64-bits toegangspunt (geen noodzaak om tags in te sluiten of voor 32-bits trampolinecode in de kernel)
- Multiboot2: ondersteuning voor hogere-helft kernels
- Multiboot2: firmware-onafhankelijke, consistente geheugenkaart op alle platforms
- Multiboot2: firmware-onafhankelijke, consistente framebuffer op alle platforms
- Multiboot2: de EDID-info van de monitor wordt ook verstrekt
- Multiboot2: start de kernel op alle processorkernen op verzoek (SMP-ondersteuning)
- er is ook een plug-in om GRUB-opstarten te simuleren met al zijn valkuilen en bugs.

Ondersteunde kernels: [Multiboot1](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html) (ELF32, PE/COFF, a.out;
BIOS, UEFI, RPi), [Multiboot2](docs/nl/ABI.md) (ELF32, ELF64, PE32+/COFF, a.out; BIOS, UEFI, RPi),
[Linux](https://www.kernel.org/doc/html/latest/arch/x86/boot.html) (BIOS, UEFI, RPi),
[Windows](https://learn.microsoft.com/en-us/windows-hardware/drivers/bringup/boot-and-uefi) (UEFI),
[OpenBSD](https://man.openbsd.org/boot.8) (BIOS, UEFI),
[FreeBSD](https://docs.freebsd.org/en/books/handbook/boot/) (BIOS, UEFI),
[FreeDOS](https://www.freedos.org/) (BIOS), [ReactOS](https://reactos.org/) (BIOS),
[MenuetOS](https://menuetos.net/) 32 / 64 (BIOS, UEFI), [KolibriOS](https://kolibrios.org/en/) (BIOS, UEFI),
[SerenityOS](https://serenityos.org/) (BIOS, UEFI), [Haiku](https://www.haiku-os.org/) (BIOS, UEFI)

Ondersteunde bestandssystemen: [FAT12/16/32](https://social.technet.microsoft.com/wiki/contents/articles/6771.the-fat-file-system.aspx),
[exFAT](https://learn.microsoft.com/en-us/windows/win32/fileio/exfat-specification),
[NTFS](https://github.com/libyal/libfsntfs/blob/main/documentation/New%20Technologies%20File%20System%20%28NTFS%29.asciidoc) (v3, v3.1),
[ext2/3/4](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout),
[XFS](https://mirror.math.princeton.edu/pub/kernel/linux/utils/fs/xfs/docs/xfs_filesystem_structure.pdf) (SGI),
[UFS](https://alter.org.ua/docs/fbsd/ufs/) (v2, v4.4),
[mfs](https://gitlab.com/bztsrc/minix3fs) (Minix3),
[BeFS](https://www.haiku-os.org/legacy-docs/practical-file-system-design.pdf) (Haiku),
[AXFS](https://gitlab.com/bztsrc/alexandriafs) (OS/Z)

(Wanneer de kernel zich op de opstartpartitie bevindt, kan elk bestandssysteem voor root worden gebruikt: ZFS, btrfs, etc.)

Onderbouwing
------------

Ik heb een eenvoudig te gebruiken bootloader gemaakt en gebruikers vroegen om steeds meer functies. Ik wilde die bootloader zo
eenvoudig mogelijk houden, maar FSF.hu bood ondersteuning, dus heb ik besloten om het te forken en alle gevraagde functies in
deze bootmanager toe te voegen.

Dit is ook een [suckless](https://suckless.org) tool zoals Simpleboot, heeft geen afhankelijkheden en is extreem makkelijk te
gebruiken:

1. maak een directory en zet je boot files erin, naast andere dingen je menu configuratie en de optionele plugins
2. voer de `easyboot (source directory) (output image file)` command uit
3. en... dat is het zo'n beetje... verder niets meer te doen! De image *Just Works (TM)*, het zal je kernels booten!

U kunt de bootmanager installeren en een bestaand apparaat of image bootable maken; of u kunt een bootable image opnieuw maken.
U kunt die image booten in een VM, of u kunt hem met `dd` of [USBImager](https://bztsrc.gitlab.io/usbimager/) naar een opslag
schrijven en die ook op een echte machine booten.

Eenvoud is de ultieme verfijning!

Installatie
-----------

Download gewoon de binary voor uw OS. Dit zijn draagbare uitvoerbare bestanden, ze vereisen geen installatie en ze hebben geen
gedeelde bibliotheken / DLL's nodig.

- [easyboot-x86_64-linux.tgz](https://gitlab.com/bztsrc/easyboot/-/raw/main/distrib/easyboot-x86_64-linux.tgz) Linux, \*BSD (1M)
- [easyboot-i686-win.zip](https://gitlab.com/bztsrc/easyboot/-/raw/main/distrib/easyboot-i686-win.zip) Windows (1M)

Verder kunt u in de [distrib](distrib) directory diverse pakketoplossingen vinden (voor Debian, Ubuntu, RaspiOS, Gentoo, Arch).

Wanneer u een image maakt, hebt u (afhankelijk van uw configuratie) mogelijk ook enkele plugins nodig in uw `(source directory)`.
U kunt deze vinden in de directory [src/plugins](src/plugins) met de extensie `plg`. Wanneer u ze installeert vanuit een pakket,
worden ze geplaatst onder `/usr/share/easyboot`.

Documentatie
------------

Gedetailleerde [documentatie](docs/nl) over het gebruik van de opstartbare schijfmaker en hoe een kernel wordt opgestart, vindt u
in de map docs.

Voorbeeld kernel
----------------

Als je een kernel wilt schrijven die geladen kan worden zonder plugins met behulp van het [vereenvoudigde Multiboot2](docs/nl/ABI.md)
protocol van **Easyboot**, kijk dan eens naar de directory [Simpleboot Voorbeeld Kernel](https://gitlab.com/bztsrc/simpleboot/-/tree/main/example).
Beide loaders gebruiken hetzelfde bootprotocol, die kernel werkt met **Easyboot** as-is. Je zult zien dat er geen Assembly of
embedded tags nodig zijn, anders is de broncode 99,9% hetzelfde als het voorbeeld in de Multiboot2-specificatie (het enige verschil
is dat het naar de seriële console wordt afgedrukt en niet naar het VGA-teletypescherm, omdat dat laatste niet bestaat op UEFI- en
RaspberryPi-machines).

Compilatie
----------

GNU/make nodig voor orkestratie (hoewel het letterlijk gewoon `cc easyboot.c -o easyboot` is). De toolchain maakt niet uit, elke
ANSI C compiler is voldoende, werkt ook op POSIX en WIN32 MINGW. Ga gewoon naar de [src](src) directory en voer `make` uit. Dat
is alles. Ondanks zijn kleine formaat is het op zichzelf staand en heeft de compilatie precies nul bibliotheek afhankelijkheden.
Het heet niet voor niets **Easyboot** :-)

Om de loaders opnieuw te compileren, heb je de [flatassembler](https://flatassembler.net) nodig, en LLVM Clang en lld (gcc en GNU
ld werken helaas niet). Maar maak je geen zorgen, ik heb ze allemaal toegevoegd aan `src/data.h` als een byte array, dus je hoeft
ze niet te compileren tenzij je dat echt heel graag wilt (daarvoor verwijder je gewoon data.h voordat je make uitvoert).

Aan de andere kant, om de plugins te compileren, heb je een cross-compiler nodig, ofwel LLVM CLang of GNU gcc (x86_64-elf-gcc,
aarch64-elf-gcc). De repo bevat deze ook als binaries. Om de hercompilatie uit te voeren, verwijder je gewoon de `src/plugins/*.plg`
bestanden voordat je make uitvoert.

Licentie
--------

**Easyboot** is gratis en open source software, gelicentieerd onder de voorwaarden van GPL versie 3 of (naar uw mening) een latere
versie. Zie het [LICENSE](LICENSE) bestand voor details.

bzt
