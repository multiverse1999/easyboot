Kernels opstarten met Easyboot
==============================

[Easyboot](https://gitlab.com/bztsrc/easyboot) is een alles-in-één bootmanager en maker van opstartbare schijfkopieën die
verschillende OS-kernels en Multiboot2-compatibele kernels in verschillende formaten kan laden.

[[_TOC_]]

Installatie
-----------

```
 easyboot [-v|-vv] [-s <mb>] [-b <mb>] [-u <guid>] [-p <t> <u> <i>] [-e] [-c] <indir> <outfile|device>

  -v, -vv         verhoog de uitgebreidheid / validatie
  -s <mb>         stel de grootte van de schijfkopie in megabytes in (standaard 35 MB)
  -b <mb>         stel de grootte van de opstartpartitie in megabytes in (standaard 33 MB)
  -u <guid>       stel de unieke identificatie van de opstartpartitie in (standaard random)
  -p <t> <u> <i>  voeg een extra partitie toe (type guid, unieke guid, imagefile)
  -e              voeg El Torito Boot Catalog toe (BIOS / EFI CDROM boot support)
  -c              maak altijd een nieuw imagebestand, zelfs als het bestaat
  <indir>         gebruik de inhoud van deze directory voor de bootpartitie
  <outfile>       uitvoer image of apparaat bestandsnaam
```

De **Easyboot** tool maakt een opstartbare schijfkopie genaamd `(outfile)` met behulp van GUID Partitioning Table met een
enkele partitie geformatteerd als FAT32 en genaamd `EFI System Partition` (kortweg ESP). De inhoud van die partitie is afkomstig
van de `(indir)` die u opgeeft. U moet een eenvoudig platte tekst configuratiebestand in die directory plaatsen met de naam
`easyboot/menu.cfg`. Met NL of CRLF regeleinden kunt u dit eenvoudig bewerken met elke teksteditor. Afhankelijk van uw configuratie
hebt u mogelijk ook wat [plugins](plugins.md) nodig in deze directory met de naam `easyboot/*.plg`.

Het image kan ook op Raspberry Pi worden opgestart en zal werken in qemu. Om echter op een echte machine op te starten, hebt u de
volgende firmwarebestanden nodig: `bootcode.bin`, `fixup.dat`, `start.elf` en een .dtb-bestand in de map `(indir)`. Deze kunnen
worden gedownload van de [officiële opslagplaats](https://github.com/raspberrypi/firmware/tree/master/boot) van de Raspberry Pi.

De tool heeft ook een aantal optionele opdrachtregelvlaggen: `-s (mb)` stelt de totale grootte van de gegenereerde schijfkopie
in Megabytes in, terwijl `-b (mb)` de grootte van de opstartpartitie in Megabytes instelt. Uiteraard moet de eerste groter zijn
dan de laatste. Als dit niet wordt gespecificeerd, wordt de partitiegrootte berekend op basis van de grootte van de opgegeven
directory (minimaal 33 Mb, de kleinste FAT32 die er kan zijn) en wordt de schijfgrootte standaard 2 Mb groter (vanwege uitlijning
en ruimte die nodig is voor de partitietabel). Als er een gat van meer dan 2 Mb is tussen deze twee groottewaarden, kunt u tools
van derden gebruiken zoals `fdisk` om meer partities aan de afbeelding toe te voegen naar wens (of zie `-p` hieronder). Als u een
voorspelbare lay-out wilt, kunt u ook de unieke id van de opstartpartitie (UniquePartitionGUID) opgeven met de vlag `-u <guid>`.

Optioneel kunt u ook extra partitie(s) toevoegen met de `-p` vlag. Hiervoor zijn 3 argumenten nodig: (PartitionTypeGUID),
(UniquePartitionGUID) en de naam van het imagebestand dat de inhoud van de partitie bevat. Deze vlag kan meerdere keren worden
herhaald.

Met de `-e` vlag wordt El Torito Boot Catalog aan de gegenereerde image toegevoegd, zodat deze niet alleen als USB-stick kan worden
opgestart, maar ook als BIOS / EFI CD-ROM.

Als `(outfile)` een apparaatbestand is (bijv. `/dev/sda` op Linux, `/dev/disk0` op BSD's en `\\.\PhysicalDrive0` op Windows), dan
creëert het geen GPT of ESP, maar lokaliseert het in plaats daarvan de reeds bestaande bestanden op het apparaat. Het kopieert nog
steeds alle bestanden in `(indir)` naar de bootpartitie en installeert de loaders. Dit werkt ook als `(outfile)` een imagebestand is
dat al bestaat (gebruik in dit geval de vlag `-c` om altijd eerst een nieuw, leeg imagebestand te maken).

Configuratie
------------

Het bestand `easyboot/menu.cfg` kan de volgende regels bevatten (zeer vergelijkbaar met de syntaxis van grub.cfg, u kunt een
voorbeeld van een configuratiebestand [hier](menu.cfg)) vinden):

### Commentaar

Alle regels die beginnen met een `#` worden beschouwd als commentaar en overgeslagen tot het einde van de regel.

### Woordrijkheidsniveau

U kunt het uitgebreidheidsniveau instellen met een regel die begint met `verbose`.

```
verbose (0-3)
```

Dit vertelt de loader hoeveel informatie er naar de bootconsole moet worden afgedrukt. Verbose 0 betekent volledig stil (standaard)
en verbose 3 dumpt de geladen kernelsegmenten en de machinecode bij het invoerpunt.

### Framebuffer

U kunt een specifieke schermresolutie aanvragen met de regel die begint met `framebuffer`. De indeling is als volgt:

```
framebuffer (breedte) (hoogte) (bits per pixel) [#(voorgrondkleur)] [#(achtergrondkleur)] [#(pbar achtergrond)]
```

**Easyboot** zal een framebuffer voor u instellen, zelfs als deze regel niet bestaat (standaard 800 x 600 x 32 bpp). Maar als
deze regel wel bestaat, zal het proberen de opgegeven resolutie in te stellen. Paletmodi worden niet ondersteund, dus bits per
pixel moeten minimaal 15 zijn.

Als de vierde optionele kleurparameter wordt gegeven, moet deze in HTML-notatie zijn, beginnend met een hashmark gevolgd door 6
hexadecimale cijfers, RRGGBB. Bijvoorbeeld `#ff0000` is volledig helderrood en `#007f7f` is een donkerder cyaan. Het stelt de
voorgrond in of met andere woorden de kleur van het lettertype. Als de vijfde optionele kleurparameter wordt gegeven, moet deze
ook in HTML-notatie zijn en stelt deze de achtergrondkleur in. De laatste optionele kleurparameter is hetzelfde en stelt de
achtergrondkleur van de voortgangsbalk in.

Als er geen kleuren zijn opgegeven, worden standaard witte letters op een zwarte achtergrond gebruikt en is de achtergrond van
de voortgangsbalk donkergrijs.

### Standaard opstartoptie

Een regel die begint met `default` geeft aan welke menuoptie zonder gebruikersinteractie moet worden opgestart na de opgegeven
time-out.

```
default (menuentry index) (time-out msec)
```

De menuentry index is een getal tussen 1 en 9. De time-out wordt uitgedrukt in milliseconden (een duizendste van een seconde),
dus 1000 staat gelijk aan één seconde.

### Menu-uitlijning

Regels die beginnen met `menualign` veranderen de manier waarop de opstartopties worden weergegeven.

```
menualign ("vertical"|"horizontal")
```

Standaard wordt het menu horizontaal weergegeven. U kunt dit wijzigen naar verticaal.

### Boot-opties

U kunt maximaal 9 menuentries opgeven met regels die beginnen met `menuentry`. De opmaak is als volgt:

```
menuentry (icoon) [label]
```

Voor elk pictogram moet er een `easyboot/(icoon).tga`-bestand op de bootpartitie aanwezig zijn. De afbeelding moet een run length
encoded, color-mapped [Targa format](../en/TGA.txt) zijn, omdat dat de meest gecomprimeerde variant is (de eerste drie bytes van
het bestand moeten `0`, `1` en `9` in deze volgorde zijn, zie Data Type 9 in de specificatie). De afmetingen moeten exact 64 pixels
hoog en 64 pixels breed zijn.

Om in dit formaat op te slaan vanuit GIMP, selecteert u eerst "Image > Mode > Indexed...", in het pop-upvenster stelt u "Maximum number of colors" in op 256. Selecteer vervolgens "File > Export As...", voer een bestandsnaam in die eindigt op `.tga`, en vink in het pop-upvenster "RLE compression" aan. Voor een opdrachtregelconversietool kunt u ImageMagick gebruiken, `convert (elk afbeeldingsbestand) -colors 256 -compress RLE icoon.tga`.

De optionele labelparameter is voor het weergeven van ASCII-versie- of release-informatie in het menu, en niet voor willekeurige
strings. Om ruimte te besparen wordt UTF-8 daarom niet ondersteund, tenzij u ook `easyboot/font.sfn` opgeeft. (Een
UNICODE-lettertype vereist veel opslagruimte, ook al is het [Scalable Screen Font](https://gitlab.com/bztsrc/scalable-font2) erg
efficiënt, het is nog steeds ongeveer 820K. Daarentegen is GRUB's unicode.pf2 veel groter, ongeveer 2392K, hoewel beide lettertypen
ca. 55600 glyphs bevatten in 8x16 en 16x16 bitmaps. Het ingebedde ASCII-lettertype heeft alleen 8x16 bitmaps en 96 glyphs.)

Alle regels die na een `menuentry`-regel komen, behoren tot die menuentry, behalve wanneer die regel een andere menuentry-regel is.
Voor het gemak kunt u blokken gebruiken zoals in GRUB, maar dit is slechts syntactische suiker. Krullende accolades worden behandeld
als witruimtetekens. U kunt ze weglaten en in plaats daarvan tabs gebruiken, net als in een Python-script of Makefile, als u dat
liever hebt.

Bijvoorbeeld
```
menuentry FreeBSD backup
{
    kernel bsd.old/boot
}
```

### Selecteer partitie

Regel die begint met `partition` selecteert een GPT-partitie. Moet worden voorafgegaan door een `menuentry`-regel.

```
partition (partitie unieke UUID)
```

Deze partitie wordt gebruikt als rootbestandssysteem voor de bootoptie. Zowel de kernel als de modules worden geladen vanaf deze
partitie en afhankelijk van het bootprotocol worden ze ook doorgegeven aan de kernel. De opgegeven partitie kan zich op een andere
schijf bevinden dan de bootdisk. **Easyboot** itereert op alle GPT-gepartitioneerde schijven tijdens het booten om deze te vinden.

Voor het gemak wordt de partitie ook gezocht op de `kernel`-regel (zie hieronder). Als de opgegeven boot-opdrachtregel een
`root=(UUID)` of `root=*=(UUID)`-tekenreeks bevat, is er geen aparte `partition`-regel nodig.

Wanneer de partitie niet expliciet wordt opgegeven, worden zowel de kernel als de modules geladen vanaf de opstartpartitie.

### Geef een kernel op

De regel die begint met `kernel` vertelt welk bestand moet worden opgestart en met welke parameters. Moet worden voorafgegaan
door een `menuentry`-regel.

```
kernel (pad naar uw kernelbestand) (optionele opstartopdrachtregelargumenten)
```

Het pad moet verwijzen naar een bestaand bestand, een uitvoerbaar kernel-binair bestand, en het moet een absoluut UTF-8-pad zijn
op de root- (of boot-)partitie. Als de kernel zich niet in de root-directory van de partitie bevindt, is de directory-scheidingsteken
altijd `/`, zelfs op UEFI-systemen. Als de naam een ​​spatie bevat, moet die worden geëscapete met `\`. Het pad kan worden gevolgd
door opdrachtregelargumenten, gescheiden door een spatie. Voor Multiboot2-compatibele kernels worden deze opdrachtregelargumenten
doorgegeven in de tag *Opstartopdrachtregel* (type 1). Ze worden niet gewijzigd door **Easyboot**, noch geparseerd, behalve
doorzocht op de partitiespecificatie.

Standaard kan **Easyboot** Multiboot2-compatibele kernels opstarten in ELF64- en PE32+/COFF-formaten (en op UEFI-systemen, ook
UEFI-applicaties). Normaal gesproken staat dat protocol geen hogere-half kernels toe, maar **Easyboot** schendt het protocol een
beetje op een manier die normale, niet-hogere-half kernels niet kapotmaakt. Als u een andere kernel wilt opstarten, hebt u een
kernel loader [plugin](plugins.md) nodig.

OPMERKING: in tegenstelling tot GRUB, waarbij u speciale opdrachten als 'linux' of 'multiboot' moet gebruiken om het opstartprotocol
te selecteren, is er hier slechts één opdracht en wordt het protocol automatisch gedetecteerd door uw kernel tijdens runtime.

### Verdere modules laden

U kunt willekeurige bestanden (initiële ramdisks, kerneldrivers, etc.) laden samen met de kernel met behulp van regels die beginnen
met `module`. Moet worden voorafgegaan door een `menuentry`-regel. Let op dat deze regel meerdere keren kan worden herhaald binnen
elke menuentry. Als het bootprotocol een initrd ondersteunt, wordt de allereerste `module`-regel beschouwd als de initrd.

```
module (pad naar een bestand) (optionele opdrachtregelargumenten van de module)
```

Het pad moet naar een bestaand bestand verwijzen en het moet een absoluut UTF-8-pad zijn op de root- (of boot-) partitie. Het
kan worden gevolgd door opdrachtregelargumenten, gescheiden door een spatie. Als het bestand is gecomprimeerd en er is een
uncompression [plugin](plugins.md) voor, dan wordt de module transparant gedecomprimeerd. Informatie over deze geladen (en
gedecomprimeerde) modules wordt doorgegeven aan een Multiboot2-compatibele kernel in de *Modulen* (type 3) tags.

Het speciale geval is als de module begint met de bytes `DSDT`, `GUDT` of `0xD00DFEED`. In deze gevallen wordt het bestand niet
toegevoegd aan de *Modulen*-tags, maar wordt de ACPI-tabel gepatcht zodat de DSDT-pointers naar de inhoud van dit bestand verwijzen.
Hiermee kunt u eenvoudig een buggy BIOS' ACPI-tabel vervangen door een door de gebruiker verstrekte.

### Logo van laarzenspatten

U kunt ook een logo in het midden van het scherm weergeven wanneer de opstartoptie is geselecteerd als u een regel plaatst die
begint met `bootsplash`. Moet worden voorafgegaan door een regel `menuentry`.

```
bootsplash [#(achtergrondkleur)] (pad naar een tga-bestand)
```

De achtergrondkleur is optioneel en moet in HTML-notatie zijn, beginnend met een hashmark gevolgd door 6 hexadecimale cijfers,
RRGGBB. Als het eerste argument niet begint met `#`, wordt een padargument aangenomen.

Het pad moet naar een bestaand bestand verwijzen en het moet een absoluut UTF-8-pad zijn op de bootpartitie (NIET root). De
afbeelding moet een run length encoded, color-mapped Targa-formaat hebben, net als de menupictogrammen. Afmetingen kunnen alles
zijn wat op het scherm past.

### Multicore-ondersteuning

Om de kernel op alle processorkernen tegelijk te starten, geeft u de richtlijn `multicore` op (alleen 64-bits kernels). Moet
worden voorafgegaan door een regel `menuentry`.

```
multicore
```

Problemen oplossen
------------------

Als u problemen ondervindt, voer dan gewoon de vlag `easyboot -vv` uit. Dit voert validatie uit en geeft de resultaten uitgebreid
weer op het moment dat de image wordt aangemaakt. Voeg anders `verbose 3` toe aan `easyboot/menu.cfg` om gedetailleerde berichten
over de opstarttijd te krijgen.

Als u de string `PMBR-ERR` in de linkerbovenhoek ziet met een rode achtergrond, betekent dit dat uw CPU erg oud is en de 64-bits
lange modus niet ondersteunt of dat de bootsector de loader niet kon bootstrappen. Dit kan alleen gebeuren op BIOS-machines, dit
kan nooit gebeuren met UEFI of op RaspberryPi.

| Bericht                             | Beschrijving                                                                      |
|-------------------------------------|-----------------------------------------------------------------------------------|
| `Booting [X]...`                    | geeft aan dat de opstartoptie (menuentry index) X is gekozen                      |
| `Loading 'X' (Y bytes)...`          | bestand X van Y lengte wordt geladen                                              |
| `Parsing kernel...`                 | kernel is gevonden, detecteert nu zijn formaat                                    |
| `Starting X boot...`                | toont aan dat de bootloader van het X-systeem is gedetecteerd                     |
| `Starting X kernel...`              | toont aan dat de kernel van het X-systeem is gedetecteerd                         |
| `Transfering X control to Y`        | geeft aan dat het X-modus-invoerpunt op Y-adres op het punt staat te worden aangeroepen |

Als u problemen ondervindt nadat u het laatste bericht hebt gezien, betekent dit dat het probleem zich heeft voorgedaan in de
opstartprocedure van het besturingssysteem en niet in de **Easyboot**-lader. U zult dus de documentatie van het betreffende
besturingssysteem moeten raadplegen voor een antwoord. Anders kunt u gerust een [issue](https://gitlab.com/bztsrc/easyboot/-/issues)
openen op gitlab.

### Multiboot1

Vereiste plug-ins: [grubmb1](../../src/plugins/grubmb1.c)

### Multiboot2

Dit is de meest flexibele, met ondersteuning voor meerdere variaties via plug-ins:

- ELF64 of PE32+ met vereenvoudigde Multiboot2: geen plugins vereist
- ELF32 met vereenvoudigde Multiboot2 en 32-bits entry point: [elf32](../../src/plugins/elf32.c)
- a.out (struct exec) met vereenvoudigde Multiboot2 en 32-bits entry point: [aout](../../src/plugins/aout.c)
- GRUB-compatibele Multiboot2 met 32-bits entry point: [grubmb2](../../src/plugins/grubmb2.c)

Let op het verschil: [vereenvoudigde Multiboot2](ABI.md) vereist geen ingesloten tags, ondersteunt hogere-helft kernels,
schoon 64-bits toegangspunt met parameters die worden doorgegeven volgens de Multiboot2, SysV en fastcall ABI.

Aan de andere kant heeft [GRUB-compatibele Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)
ingebedde tags nodig, ondersteunt geen hogere-half- of 64-bits kernels, het toegangspunt is altijd 32-bits en parameters worden
alleen doorgegeven in `eax`, `ebx`.

### Windows

Geen plugins vereist, maar u moet [Secure Boot](secureboot.md) instellen.

```
menuentry win {
  kernel EFI/Microsoft/BOOT/BOOTMGRW.EFI
}
```

### Linux

Vereiste plug-ins: [linux](../../src/plugins/linux.c), [ext234](../../src/plugins/ext234.c)

Als de kernel niet op de opstartpartitie staat, kunt u de opdrachtregel gebruiken om de rootpartitie op te geven.

```
menuentry linux {
  kernel vmlinuz-linux root=PARTUUID=01020304-0506-0708-0a0b0c0d0e0f1011
}
```

### OpenBSD

Vereiste plug-ins: [obsdboot](../../src/plugins/obsdboot.c), [ufs44](../../src/plugins/ufs44.c)

```
menuentry openbsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot
}
```

WAARSCHUWING! Gebruik de [elf32](../../src/plugins/elf32.c) plugin niet als u OpenBSD opstart! De `boot` claimt ten onrechte
een ELF te zijn met een 32-bits SysV ABI-toegangspunt, maar in werkelijkheid heeft het een 16-bits real mode-toegangspunt.

### FreeBSD

Vereiste plug-ins: [fbsdboot](../../src/plugins/fbsdboot.c), [ufs2](../../src/plugins/ufs2.c)

Op oudere BIOS-systemen specificeert u de loader `boot`.

```
menuentry freebsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot/boot
}
```

Gebruik op UEFI-machines `loader.efi` op de opstartpartitie (geen plug-ins nodig).

```
menuentry freebsd {
  kernel boot/loader.efi
}
```

Als uw rootbestandssysteem ZFS is, kopieer dan dit ene bestand (`boot` in BIOS, `loader.efi` in UEFI) naar `(indir)` en specificeer
GEEN rootpartitie.

### FreeDOS

Vereiste plug-ins: [fdos](../../src/plugins/fdos.c)

Verplaats de bestanden van FreeDOS naar `(indir)` (FreeDOS gebruikt de bootpartitie als rootpartitie).

```
menuentry freedos {
  kernel KERNEL.SYS
}
```

Als het opstarten stopt na het afdrukken van copyright en `- InitDisk`, dan is de FreeDOS kernel gecompileerd zonder FAT32
ondersteuning. Download een andere kernel, met `f32` in de naam.

### ReactOS

Vereiste plug-ins: [reactos](../../src/plugins/reactos.c)

```
menuentry reactos {
  kernel FREELDR.SYS
}
```

### MenuetOS

Vereiste plug-ins: [menuet](../../src/plugins/menuet.c)

```
menuentry menuetos {
  kernel KERNEL.MNT
  module CONFIG.MNT
  module RAMDISK.MNT
}
```

Om automatische configuratie uit te schakelen, voegt u `noauto` toe aan de opdrachtregel.

### KolibriOS

Vereiste plug-ins: [kolibri](../../src/plugins/kolibri.c)

```
menuentry kolibrios {
  kernel KERNEL.MNT syspath=/rd/1/ launcher_start=1
  module KOLIBRI.IMG
  module DEVICES.DAT
}
```

De plugin werkt ook op UEFI-machines, maar u kunt `uefi4kos.efi` ook op de opstartpartitie gebruiken (en er is geen plugin vereist).

### SerenityOS

Vereiste plug-ins: [grubmb1](../../src/plugins/grubmb1.c)

```
menuentry serenityos {
  kernel boot/Prekernel
  module boot/Kernel
}
```

### Haiku

Vereiste plug-ins: [grubmb1](../../src/plugins/grubmb1.c), [befs](../../src/plugins/befs.c)

```
menuentry haiku {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel system/packages/haiku_loader-r1~beta4_hrev56578_59-1-x86_64.hpkg
}
```

Gebruik op UEFI-machines `haiku_loader.efi` op de opstartpartitie (geen plug-ins nodig).

### OS/Z

```
menuentry osz {
  kernel ibmpc/core
  module ibmpc/initrd
}
```

Geen plug-ins noodzakelijk.
