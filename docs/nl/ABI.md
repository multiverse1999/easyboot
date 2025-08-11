Schrijven van Easyboot-compatibele kernels
==========================================

[Easyboot](https://gitlab.com/bztsrc/easyboot) ondersteunt verschillende kernels met behulp van [plugins](plugins.md). Maar als
er geen geschikte plugin wordt gevonden, valt het terug op ELF64 of PE32+ binaries met een vereenvoudigde (geen noodzaak om iets
in te sluiten) variant van het Multiboot2 protocol.

Dit is precies hetzelfde protocol dat [Simpleboot](https://gitlab.com/bztsrc/simpleboot) gebruikt. Alle voorbeeldkernels in die
repository moeten ook met **Easyboot** werken.

U kunt de originele multiboot2.h header in GRUB's repo gebruiken, of het [easyboot.h](../../easyboot.h) C/C++ header bestand om
gemakkelijker te gebruiken typedefs te krijgen. Het low-level binaire formaat is hetzelfde, u kunt ook alle bestaande Multiboot2
bibliotheken gebruiken, zelfs met niet-C talen, zoals deze [Rust](https://github.com/rust-osdev/multiboot2/tree/main/multiboot2/src)
bibliotheek bijvoorbeeld (let op: ik ben op geen enkele manier gelieerd aan die devs, ik heb gewoon gezocht naar "Rust Multiboot2"
en dat was het eerste resultaat).

[[_TOC_]]

Opstartvolgorde
---------------

### Bootstrapping van de lader

Op *BIOS* machines wordt de allereerste sector van de schijf door de firmware geladen naar 0:0x7C00 en wordt de controle hieraan
doorgegeven. In deze sector heeft **Easyboot** [boot_x86.asm](../../src/boot_x86.asm), die slim genoeg is om de 2e fase-loader
te vinden en te laden, en ook om de lange modus ervoor in te stellen.

Op *UEFI* machines wordt hetzelfde 2e fase bestand, genaamd `EFI/BOOT/BOOTX64.EFI`, direct geladen door de firmware. De bron voor
deze loader is te vinden in [loader_x86.c](../../src/loader_x86.c). Dat is het, **Easyboot** is niet GRUB of syslinux, die beide
tientallen en tientallen systeembestanden op de schijf vereisen. Hier zijn geen andere bestanden nodig, alleen deze (plugins zijn
optioneel, geen nodig om Multiboot2 compatibiliteit te bieden).

Op *Raspberry Pi* heet de loader `KERNEL8.IMG`, gecompileerd vanuit [loader_rpi.c](../../src/loader_rpi.c).

### De lader

Deze loader is zeer zorgvuldig geschreven om op meerdere configuraties te werken. Hij laadt de GUID Partitining Table van de schijf
en zoekt naar een "EFI System Partition". Wanneer hij die vindt, zoekt hij naar het configuratiebestand `easyboot/menu.cfg` op die
bootpartitie. Nadat de bootoptie is geselecteerd en de bestandsnaam van de kernel bekend is, zoekt de loader deze en laadt deze.

Vervolgens detecteert het automatisch het formaat van de kernel en is het slim genoeg om de sectie- en segmentinformatie te
interpreteren over waar wat geladen moet worden (het doet on-demand geheugentoewijzing wanneer nodig). Vervolgens stelt het
een geschikte omgeving in, afhankelijk van het gedetecteerde opstartprotocol (Multiboot2 / Linux / etc. protected of long mode,
ABI-argumenten etc.). Nadat de machinestatus solide en goed gedefinieerd is, springt de loader als allerlaatste handeling naar
het toegangspunt van uw kernel.

Machinestatus
-------------

Alles wat in de [Multiboot2-specificatie](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) over de machinestatus
staat, behalve de algemene registers. **Easyboot** geeft twee argumenten door aan het toegangspunt van uw kernel volgens de SysV ABI
en Microsoft fastcall ABI. De eerste parameter is de magie, de tweede is een fysiek geheugenadres, dat verwijst naar een Multiboot
Information-taglijst (hierna afgekort als MBI, zie hieronder).

We schenden ook een beetje het Multiboot2-protocol om hogere-half kernels te verwerken. Multiboot2 vereist dat geheugen
identiteitsgemapt moet zijn. Nou, onder **Easyboot** is dit slechts gedeeltelijk waar: we garanderen alleen dat al het
fysieke RAM zeker identiteitsgemapt is zoals verwacht; echter, sommige regio's daarboven (afhankelijk van de programmaheaders
van de kernel) kunnen nog beschikbaar zijn. Dit breekt normale Multiboot2-compatibele kernels niet, die niet geacht worden om
geheugen buiten het beschikbare fysieke RAM te benaderen.

Uw kernel wordt op exact dezelfde manier geladen op zowel BIOS- als UEFI-systemen en op RPi, firmwareverschillen zijn gewoon
"probleem van iemand anders". Het enige dat uw kernel zal zien, is of de MBI de EFI-systeemtabeltag bevat of niet. Om uw leven
te vereenvoudigen, genereert **Easyboot** ook geen EFI-geheugenkaart (type 17)-tag, het biedt alleen de [Memory map](#memory_map)
(type 6)-tag zonder onderscheid op alle platforms (ook op UEFI-systemen wordt de geheugenkaart daar gewoon voor u geconverteerd,
dus uw kernel hoeft maar met één soort geheugenlijsttag om te gaan). Oude, verouderde tags worden ook weggelaten en nooit
gegenereerd door deze bootmanager.

De kernel draait op supervisorniveau (ring 0 op x86, EL1 op ARM), mogelijk op alle CPU-cores parallel.

GDT niet gespecificeerd, maar geldig. Stack is ingesteld in de eerste 640k en groeit naar beneden (maar u moet dit zo snel
mogelijk wijzigen naar welke stack u ook waardig acht). Wanneer SMP is ingeschakeld, hebben alle cores hun eigen stack en
staat de core-id bovenaan de stack (maar u kunt de core-id ook op de gebruikelijke platformspecifieke manier verkrijgen,
met behulp van cpuid / mpidr / etc.).

U moet IDT als niet-gespecificeerd beschouwen; IRQ's, NMI en software-interrupts zijn uitgeschakeld. Dummy-uitzonderingshandlers
zijn ingesteld om een ​​minimale dump weer te geven en de machine te stoppen. Deze moeten alleen worden gebruikt om te rapporteren
als uw kernel kapotgaat voordat u uw eigen IDT en handlers kon instellen, bij voorkeur zo snel mogelijk. Op ARM is vbar_el1
ingesteld om dezelfde dummy-uitzonderingshandlers aan te roepen (hoewel ze natuurlijk verschillende registers dumpen).

Framebuffer is ook standaard ingesteld. U kunt de resolutie in config wijzigen, maar als dit niet is opgegeven, is framebuffer
nog steeds geconfigureerd.

Het is belangrijk om nooit terug te keren van je kernel. Je bent vrij om elk deel van de loader in het geheugen te overschrijven
(zodra je klaar bent met de MBI-tags), dus er is gewoon nergens om naar terug te keren. "Der Mohr hat seine Schuldigkeit getan,
der Mohr kann gehen."

Opstartinformatie doorgegeven aan uw kernel (MBI)
-------------------------------------------------

Het is niet meteen duidelijk, maar de Multiboot2-specificatie definieert in feite twee, volledig onafhankelijke sets tags:

- De eerste set zou in een Multiboot2-compatibele kernel moeten worden ingevoegd, genaamd OS image's Multiboot2 header
  (sectie 3.1.2), vandaar *geleverd door de kernel*. **Easyboot** geeft niets om deze tags en parseert uw kernel er ook
  niet op. U hebt gewoon geen speciale magische gegevens nodig die in uw kernelbestand zijn ingebed met **Easyboot**,
  ELF en PE headers volstaan.

- De tweede set wordt dynamisch *doorgegeven aan de kernel* bij het opstarten, **Easyboot** gebruikt alleen deze tags. Het
  genereert echter niet alles wat Multiboot2 specificeert (het laat simpelweg de oude, verouderde of legacy tags weg). Deze
  tags worden de MBI tags genoemd, zie [Boot information](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Boot-information-format)
  (sectie 3.6).

OPMERKING: de Multiboot2-specificatie op MBI-tags is buggy as hell. Hieronder vindt u een aangepaste versie, die overeenkomt
met het multiboot2.h-headerbestand dat u kunt vinden in de bronrepository van GRUB.

De eerste parameter voor uw kernel is de magic 0x36d76289 (in `rax`, `rcx` en `rdi`). U kunt de MBI-tags vinden met de tweede
parameter (in `rbx`, `rdx` en `rsi`). Op het ARM-platform staat magic in `x0` en adres in `x1`. Op RISC-V en MIPS worden
respectievelijk `a0` en `a1` gebruikt. Als en wanneer deze loader naar een andere architectuur wordt geporteerd, moeten altijd
de registers worden gebruikt die door SysV ABI zijn opgegeven voor functie-argumenten. Als er andere algemene ABI's op het
platform zijn die niet interfereren met SysV ABI, moeten de waarden ook in de registers van die ABI's worden gedupliceerd
(of bovenaan de stack).

### Kopteksten

Het doorgegeven adres is altijd 8-byte uitgelijnd en begint met een MBI-header:

```
        +-------------------+
u32     | total_size        |
u32     | reserved          |
        +-------------------+
```

Dit wordt gevolgd door een serie van eveneens 8-bytes uitgelijnde tags. Elke tag begint met de volgende tag header-velden:

```
        +-------------------+
u32     | type              |
u32     | size              |
        +-------------------+
```

`type` bevat een identifier van de inhoud van de rest van de tag. `size` bevat de grootte van de tag inclusief headervelden
maar exclusief padding. Tags volgen elkaar op, indien nodig opgevuld, zodat elke tag begint bij een uitgelijnd adres van 8 bytes.

### Terminator

```
        +-------------------+
u32     | type = 0          |
u32     | size = 8          |
        +-------------------+
```

Tags worden beëindigd door een tag van het type `0` en de grootte `8`.

### Opstartopdrachtregel

```
        +-------------------+
u32     | type = 1          |
u32     | size              |
u8[n]   | string            |
        +-------------------+
```

`string` bevat de opdrachtregel die is opgegeven in de `kernel`-regel van *menuentry* (zonder het pad en de bestandsnaam van de
kernel). De opdrachtregel is een normale C-stijl nul-beëindigde UTF-8-string.

### Naam van de bootloader

```
        +----------------------+
u32     | type = 2             |
u32     | size = 17            |
u8[n]   | string "Easyboot"    |
        +----------------------+
```

`string` bevat de naam van een bootloader die de kernel opstart. De naam is een normale C-stijl UTF-8 zero-terminated string.

### Modulen

```
        +-------------------+
u32     | type = 3          |
u32     | size              |
u32     | mod_start         |
u32     | mod_end           |
u8[n]   | string            |
        +-------------------+
```

Deze tag geeft aan de kernel aan welke bootmodule samen met de kernelimage is geladen en waar deze te vinden is. `mod_start`
en `mod_end` bevatten de fysieke begin- en eindadressen van de bootmodule zelf. U krijgt nooit een gzip-gecomprimeerde buffer,
omdat **Easyboot** deze transparant voor u decomprimeert (en als u een plugin levert, werkt deze ook met andere dan
gzip-gecomprimeerde gegevens). Het veld `string` biedt een willekeurige string die aan die specifieke bootmodule moet worden
gekoppeld; het is een normale C-stijl zero-terminated UTF-8 string. Gespecificeerd in de `module`-regel van *menuentry* en het
exacte gebruik ervan is specifiek voor het besturingssysteem. In tegenstelling tot de boot-opdrachtregeltag bevatten de
moduletags *ook* het pad en de bestandsnaam van de module.

Er verschijnt één tag per module. Dit tagtype kan meerdere keren voorkomen. Als er een initiële ramdisk is geladen samen met
uw kernel, dan verschijnt dat als de eerste module.

Er is een speciaal geval: als het bestand een DSDT ACPI-tabel, een FDT (dtb) of een GUDT-blob is, wordt het niet als een module
weergegeven. In plaats daarvan wordt ACPI oud RSDP (type 14) of ACPI nieuw RSDP (type 15) gepatcht en wordt de DSDT vervangen
door de inhoud van dit bestand.

### Memory map

Deze tag biedt een geheugenkaart.

```
        +-------------------+
u32     | type = 6          |
u32     | size              |
u32     | entry_size = 24   |
u32     | entry_version = 0 |
varies  | entries           |
        +-------------------+
```

`size` bevat de grootte van alle entries inclusief dit veld zelf. `entry_size` is altijd 24. `entry_version` is ingesteld op `0`.
Elke entry heeft de volgende structuur:

```
        +-------------------+
u64     | base_addr         |
u64     | length            |
u32     | type              |
u32     | reserved          |
        +-------------------+
```

`base_addr` is het fysieke beginadres. `length` is de grootte van het geheugengebied in bytes. `type` is de variëteit van het
weergegeven adresbereik, waarbij de waarde `1` het beschikbare RAM aangeeft, de waarde `3` het bruikbare geheugen aangeeft dat
ACPI-informatie bevat, de waarde `4` gereserveerd geheugen aangeeft dat bewaard moet blijven tijdens de slaapstand, de waarde
`5` geheugen aangeeft dat bezet is door defecte RAM-modules en alle andere waarden momenteel een gereserveerd gebied aangeven.
`reserved` wordt ingesteld op `0` bij het opstarten van het BIOS.

Wanneer de MBI op een UEFI-machine wordt gegenereerd, worden verschillende EFI-geheugenkaartvermeldingen opgeslagen als type
`1` (beschikbaar RAM) of `2` (gereserveerd RAM). Mocht u het nodig hebben, dan wordt het oorspronkelijke EFI-geheugentype in
het veld `reserved` geplaatst.

De verstrekte kaart geeft gegarandeerd alle standaard RAM weer die beschikbaar zou moeten zijn voor normaal gebruik, en is
altijd gesorteerd op oplopende `base_addr`. Dit beschikbare RAM-type omvat echter de regio's die worden ingenomen door de
kernel, mbi, segmenten en modules. De kernel moet ervoor zorgen dat deze regio's niet worden overschreven (**Easyboot** zou
deze regio's gemakkelijk kunnen uitsluiten, maar dat zou de Multiboot2-compatibiliteit verstoren).

### Framebuffer-info

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

Het veld `framebuffer_addr` bevat het fysieke adres van de framebuffer. Het veld `framebuffer_pitch` bevat de lengte van één rij in
bytes. De velden `framebuffer_width`, `framebuffer_height` bevatten de framebufferafmetingen in pixels. Het veld `framebuffer_bpp`
bevat het aantal bits per pixel. `framebuffer_type` is altijd ingesteld op 1 en `reserved` bevat altijd 0in de huidige versie van
de specificatie en moet worden genegeerd door de OS-image. De overige velden beschrijven het ingepakte pixelformaat, de positie en
grootte van de kanalen in bits. U kunt de expressie `((~(0xffffffff << size)) << position) & 0xffffffff` gebruiken om een ​​UEFI
GOP-achtig kanaalmasker te krijgen.

### EFI 64-bits systeemtabelwijzer

Deze tag bestaat alleen als **Easyboot** draait op een UEFI-machine. Op een BIOS-machine wordt deze tag nooit gegenereerd.

```
        +-------------------+
u32     | type = 12         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Deze tag bevat een verwijzing naar de EFI-systeemtabel.

### EFI 64-bits afbeeldingshandvatwijzer

Deze tag bestaat alleen als **Easyboot** draait op een UEFI-machine. Op een BIOS-machine wordt deze tag nooit gegenereerd.

```
        +-------------------+
u32     | type = 20         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Deze tag bevat een pointer naar EFI image handle. Meestal is dit een boot loader afbeeldingshandvatwijzer.

### SMBIOS-tabellen

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

Deze tag bevat een kopie van de SMBIOS-tabellen en hun versie.

### ACPI oud RSDP

```
        +-------------------+
u32     | type = 14         |
u32     | size              |
        | copy of RSDPv1    |
        +-------------------+
```

Deze tag bevat een kopie van RSDP zoals gedefinieerd in de ACPI 1.0-specificatie. (Met een 32-bits adres.)

### ACPI nieuw RSDP

```
        +-------------------+
u32     | type = 15         |
u32     | size              |
        | copy of RSDPv2    |
        +-------------------+
```

Deze tag bevat een kopie van RSDP zoals gedefinieerd in de ACPI 2.0 of latere specificatie. (Waarschijnlijk met een 64-bits adres.)

Deze (type 14 en 15) verwijzen naar een `RSDT` of `XSDT` tabel met een pointer naar een `FACP` tabel, die op zijn beurt twee
pointers bevat naar een `DSDT` tabel, die de machine beschrijft. **Easyboot** vervalst deze tabellen op machines die ACPI anders
niet ondersteunen. Ook als u een DSDT tabel, een FDT (dtb) of GUDT blob als module opgeeft, zal **Easyboot** de pointers patchen
om naar die door de gebruiker opgegeven tabel te verwijzen. Om deze tabellen te parsen, kunt u mijn dependency-free, single
header [hwdet](https://gitlab.com/bztsrc/hwdet) bibliotheek gebruiken (of de opgeblazen [apcica](https://github.com/acpica/acpica)
en [libfdt](https://github.com/dgibson/dtc)).

Kernel-specifieke tags
----------------------

Tags met `type` groter dan of gelijk aan 256 maken geen deel uit van de Multiboot2-specificatie, maar worden wel geleverd
door **Easyboot**. Deze kunnen worden toegevoegd door optionele [plugins](plugins.md) aan de lijst, indien en wanneer een
kernel ze nodig heeft.

### EDID

```
        +-------------------+
u32     | type = 256        |
u32     | size              |
        | copy of EDID      |
        +-------------------+
```

Deze tag bevat een kopie van de lijst met ondersteunde monitorresoluties volgens de EDID-specificatie.

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

Deze tag bestaat als de richtlijn `multicore` is gegeven. `numcores` bevat het aantal CPU-cores in het systeem, `running` is het
aantal cores dat succesvol is geïnitialiseerd en dezelfde kernel parallel uitvoert. De `bspid` bevat de id van de BSP-core (op x86
lAPIC-id), zodat kernels AP's kunnen onderscheiden en een andere code op die AP's kunnen uitvoeren. Alle AP's hebben hun eigen
stack en boven op de stack staat de id van de huidige core.

### Partitie-ID's

```
        +-------------------+
u32     | type = 258        |
u32     | size = 24 / 40    |
u128    | bootuuid          |
u128    | rootuuid          |
        +-------------------+
```

Deze tag bevat de unieke identificatievelden in de GPT van de boot- en rootpartitie. Als het opstarten geen GUID Partitioning Table
gebruikt, wordt `bootuuid` gegenereerd als `54524150-(apparaatcode)-(partitienummer)-616F6F7400000000`.

Geheugenindeling
----------------

### BIOS-machines

| Begin    | Einde   | Beschrijving                                                 |
|---------:|--------:|--------------------------------------------------------------|
|      0x0 |   0x400 | Interrupt Vector Table (bruikbaar, echte modus IDT)          |
|    0x400 |   0x4FF | BIOS Data Area (bruikbaar)                                   |
|    0x4FF |   0x500 | BIOS-opstartschijfcode (meestal 0x80, bruikbaar)             |
|    0x500 |   0x5A0 | synchronisatiegegevens voor SMP (bruikbaar)                  |
|    0x5A0 |  0x1000 | uitzonderingshandlerstack (bruikbaar nadat u uw IDT hebt ingesteld) |
|   0x1000 |  0x8000 | pagineringstabellen (bruikbaar nadat u uw pagineringstabellen hebt ingesteld) |
|   0x8000 | 0x20000 | loadercode en -gegevens (bruikbaar nadat u uw IDT hebt ingesteld) |
|  0x20000 | 0x40000 | config + tags (bruikbaar na MBI-parsering)                   |
|  0x40000 | 0x90000 | plugin ids; van boven naar beneden: kernel stack             |
|  0x90000 | 0x9A000 | Enkel Linux-kernel: zero page + cmdline                      |
|  0x9A000 | 0xA0000 | Extended BIOS Data Area (beter niet aanraken)                |
|  0xA0000 | 0xFFFFF | VRAM and BIOS ROM (niet bruikbaar)                           |
| 0x100000 |       x | kernelsegmenten, gevolgd door de modules, elke pagina uitgelijnd |

### UEFI-machines

Niemand weet het. UEFI wijst geheugen toe zoals het wil. Verwacht van alles en nog wat. Alle gebieden worden zeker vermeld in de
geheugenkaart als type = 1 (`MULTIBOOT_MEMORY_AVAILABLE`) en reserved = 2 (`EfiLoaderData`), maar dit is niet exclusief, andere
soorten geheugen kunnen ook op die manier worden vermeld (bijvoorbeeld de bss-sectie van de bootmanager).

### Raspberry Pi

| Begin    | Einde   | Beschrijving                                                        |
|---------:|--------:|---------------------------------------------------------------------|
|      0x0 |   0x500 | gereserveerd door firmware (beter niet aanraken)                    |
|    0x500 |   0x5A0 | synchronisatiegegevens voor SMP (bruikbaar)                         |
|    0x5A0 |  0x1000 | uitzonderingshandlerstack (bruikbaar nadat u uw VBAR hebt ingesteld) |
|   0x1000 |  0x9000 | pagineringstabellen (bruikbaar nadat u uw pagineringstabellen hebt ingesteld) |
|   0x9000 | 0x20000 | loadercode en -gegevens (bruikbaar nadat u uw VBAR hebt ingesteld)  |
|  0x20000 | 0x40000 | config + tags (bruikbaar na MBI-parsering)                          |
|  0x40000 | 0x80000 | firmware geleverd FDT (dtb); van boven naar beneden: kernel stack   |
| 0x100000 |       x | kernelsegmenten, gevolgd door de modules, elke pagina uitgelijnd    |

De eerste paar bytes zijn gereserveerd voor [armstub](https://github.com/raspberrypi/tools/blob/master/armstubs/armstub8.S).
Alleen core 0 is gestart, dus om Application Processors te starten, schrijf je het adres van een functie naar 0xE0 (core 1),
0xE8 (core 2), 0xF0 (core 3), welke adressen zich in dit gebied bevinden. Dit is irrelevant wanneer de `multicore`-richtlijn
wordt gebruikt, dan zullen alle cores de kernel uitvoeren.

Hoewel het niet standaard wordt ondersteund op de RPi, krijg je nog steeds een ACPI oude RSDP (type 14) tag, met neptabellen.
De `APIC` tabel wordt gebruikt om het aantal beschikbare CPU cores te communiceren naar de kernel. Het opstartfunctie adres
wordt opgeslagen in het RSD PTR -> RSDT -> APIC -> cpu\[x].apic_id veld (en core id in cpu\[x].acpi_id, waarbij BSP altijd
cpu\[0].acpi_id = 0 en cpu\[0].apic_id = 0xD8 is. Let op, "acpi" en "apic" lijken behoorlijk op elkaar).

Als een geldige FDT-blob door de firmware wordt doorgegeven, of als een van de modules een .dtb-, .gud- of .aml-bestand is,
wordt er ook een FADT-tabel (met magische `FACP`) toegevoegd. In deze tabel wijst de DSDT-pointer (32-bits, op offset 40) naar
de meegeleverde afgeplatte apparaatboomblob.

Hoewel de firmware geen memory map-functie biedt, krijgt u nog steeds een Memory Map (type 6)-tag, met een lijst van gedetecteerde
RAM en de MMIO-regio. U kunt dit gebruiken om het basisadres van de MMIO te detecteren, dat verschilt op RPi3 en RPi4.
