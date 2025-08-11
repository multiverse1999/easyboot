Easyboot-plug-ins
=================

Standaard start [Easyboot](https://gitlab.com/bztsrc/easyboot) Multiboot2-compatibele kernels op in ELF- en PE-formaten vanaf
de bootpartitie. Als uw kernel een ander bestandsformaat, een ander bootprotocol of niet op de bootpartitie gebruikt, hebt u
plugins op de bootpartitie nodig. U kunt deze vinden in de directory [src/plugins](../../src/plugins).

[[_TOC_]]

Installatie
-----------

Om plug-ins te installeren, kopieert u ze eenvoudigweg naar de map die is opgegeven in de parameter `(indir)`, onder de
submap `easyboot` naast het bestand menu.cfg.

Bijvoorbeeld:
```
bootpart
|-- easyboot
|   |-- linux_x86.plg
|   |-- minix_x86.plg
|   `-- menu.cfg
|-- EFI
|   `-- BOOT
|-- kernel
`-- initrd
$ easyboot bootpart schijf.img
```

Compilatie
----------

*Het was vanaf het begin duidelijk dat ELF niet geschikt is voor deze taak. Het is te opgeblazen en te complex. Dus oorspronkelijk
wilde ik struct exec gebruiken (het klassieke UNIX a.out formaat), maar helaas kunnen moderne toolchains dat niet meer maken. Dus
heb ik besloten om mijn eigen formaat en eigen linker voor de plugins te maken.*

U kunt de bron van de plugin compileren met elke standaard ANSI C cross-compiler in een ELF-objectbestand, maar dan moet u de
[plgld](../../src/misc/plgld.c) linker gebruiken om het uiteindelijke binaire bestand te maken. Dit is een architectuur-agnostische
cross-linker, die werkt ongeacht de machinecode waarvoor de plugin is gecompileerd. De uiteindelijke .plg is slechts een fractie
van de .o ELF waarvan deze is gegenereerd.

### Plugin-API

De C-bron van een plugin moet het headerbestand `src/loader.h` bevatten en moet een regel `EASY_PLUGIN` bevatten. Deze heeft één
parameter, het type van de plugin, gevolgd door de identifier match-specificatie. Deze laatste wordt door de loader gebruikt om te
bepalen wanneer die specifieke plugin moet worden gebruikt.

Bijvoorbeeld:

```c
#include "../loader.h"

/* magische bytes die een Linux-kernel identificeren */
EASY_PLUGIN(PLG_T_KERNEL) {
   /* offset  maat  wedstrijdtype  magische bytes */
    { 0x1fe,     2, PLG_M_CONST, { 0xAA, 0x55, 0, 0 } },
    { 0x202,     4, PLG_M_CONST, { 'H', 'd', 'r', 'S' } }
};

/* toegangspunt, prototype wordt gedefinieerd door het type van de plug-in */
PLG_API void _start(uint8_t *buf, uint64_t size);
{
    /* omgeving voorbereiden voor een Linux-kernel */
}
```

De plugins kunnen verschillende variabelen en functies gebruiken. Deze zijn allemaal gedefinieerd in het headerbestand en aan de
runtime gekoppeld.

```c
uint32_t verbose;
```
Verbosity level. Een plugin mag alleen output printen als dit niet nul is, behalve bij foutmeldingen. Hoe groter de waarde, hoe
meer details er geprint moeten worden.

```c
uint64_t file_size;
```
De totale grootte van het geopende bestand (zie `open` en `loadfile` hieronder).

```c
uint8_t *root_buf;
```
Wanneer een bestandssysteem-plugin initialiseert, bevat dit de eerste 128k van de partitie (hopelijk inclusief het superblok).
Later kan een bestandssysteem-plugin deze 128k buffer hergebruiken voor welk doel dan ook (FAT-cache, inode-cache, etc.)

```c
uint8_t *tags_buf;
```
Bevat de Multiboot2-tags. Een kernelplugin kan dit parsen om de door de bootmanager geleverde gegevens om te zetten in het
formaat dat de kernel verwacht. Deze pointer wijst naar het begin van de buffer.


```c
uint8_t *tags_ptr;
```
Deze pointer wijst naar het einde van de Multiboot2 tags buffer. Tag plugins kunnen hier nieuwe tags toevoegen en deze pointer
aanpassen.

```c
uint8_t *rsdp_ptr;
```
Verwijst naar de RSDP ACPI-aanwijzer.

```c
uint8_t *dsdt_ptr;
```
Verwijst naar de DSDT (of GUDT, FDT) hardwarebeschrijvingsblob.

```c
efi_system_table_t *ST;
```
Op UEFI-machines verwijst dit naar de EFI-systeemtabel, anders is het `NULL`.

```c
void memset(void *dst, uint8_t c, uint32_t n);
void memcpy(void *dst, const void *src, uint32_t n);
int  memcmp(const void *s1, const void *s2, uint32_t n);
```
Verplichte geheugenfuncties (de C-compiler kan hiernaar aanroepen, zelfs als er geen directe aanroep is).

```c
void *alloc(uint32_t num);
```
Wijst `num` pagina's (4k) geheugen toe. Plugins mogen niet veel toewijzen, ze moeten mikken op minimale geheugenfootprint.

```c
void free(void *buf, uint32_t num);
```
Maak eerder toegewezen geheugen van `num` pagina's vrij.

```c
void printf(char *fmt, ...);
```
Geeft een geformatteerde tekenreeks weer in de opstartconsole.

```c
uint64_t pb_init(uint64_t size);
```
Start de voortgangsbalk, `size` is de totale grootte die het vertegenwoordigt. Retourneert hoeveel bytes één pixel waard is.

```c
void pb_draw(uint64_t curr);
```
Tekent de voortgangsbalk voor de huidige waarde. `curr` moet tussen 0 en de totale grootte liggen.

```c
void pb_fini(void);
```
Sluit de voortgangsbalk en maak deze leeg op het scherm.

```c
void loadsec(uint64_t sec, void *dst);
```
Wordt gebruikt door de plug-ins van het bestandssysteem en laadt een sector van de schijf in het geheugen. `sec` is het
sectornummer, relatief ten opzichte van de rootpartitie.

```c
void sethooks(void *o, void *r, void *c);
```
Wordt gebruikt door de plug-ins voor het bestandssysteem en stelt de hooks van de functies open / read / close in voor het
bestandssysteem van de rootpartitie.

```c
int open(char *fn);
```
Open een bestand op de root (of boot) partitie om te lezen, retourneert 1 bij succes. Er kan slechts één bestand tegelijk worden
geopend. Als er geen `sethooks` call van tevoren was, dan werkt het op de boot partitie.

```c
uint64_t read(uint64_t offs, uint64_t size, void *buf);
```
Leest gegevens uit het geopende bestand op de zoekpositie `offs` in het geheugen en retourneert het aantal daadwerkelijk gelezen
bytes.

```c
void close(void);
```
Sluit het geopende bestand.

```c
uint8_t *loadfile(char *path);
```
Laad een bestand volledig van de root (of boot) partitie in een nieuw toegewezen geheugenbuffer, en decomprimeer het transparant
als de plugin gevonden is. Grootte geretourneerd in `file_size`.

```c
int loadseg(uint32_t offs, uint32_t filesz, uint64_t vaddr, uint32_t memsz);
```
Laad een segment uit de kernelbuffer. Dit controleert of het geheugen `vaddr` beschikbaar is en brengt het segment in kaart als
het een hogere helft is. De `offs` is de bestandsoffset, dus relatief ten opzichte van de kernelbuffer. Als `memsz` groter is
dan `filesz`, wordt het verschil opgevuld met nullen.

```c
void _start(void);
```
Ingangspunt voor bestandssysteemplugins (`PLG_T_FS`). Het zou het superblok in `root_buf` moeten parsen en `sethooks` moeten
aanroepen. Bij een fout zou het gewoon moeten terugkeren zonder zijn hooks in te stellen.

```c
void _start(uint8_t *buf, uint64_t size);
```
Ingangspunt voor kernel plugins (`PLG_T_KERNEL`). Ontvangt de kernel image in het geheugen, het zou zijn segmenten moeten
verplaatsen, de juiste omgeving moeten instellen en de controle overdragen. Als er geen fout is, keert het nooit terug.

```c
uint8_t *_start(uint8_t *buf);
```
Ingangspunt voor decompressieplugins (`PLG_T_DECOMP`). Ontvangt de gecomprimeerde buffer (en de grootte ervan in `file_size`)
en moet een toegewezen nieuwe buffer retourneren met de ongecomprimeerde gegevens (en de grootte van de nieuwe buffer ook
instellen in `file_size`). Het moet de oude buffer vrijgeven (let op, `file_size` is in bytes, maar free() verwacht grootte
in pagina's). Bij een fout mag `file_size` niet worden gewijzigd en moet het de ongewijzigde originele buffer retourneren.

```c
void _start(void);
```
Ingangspunt voor tag-plugins (`PLG_T_TAG`). Ze kunnen nieuwe tags toevoegen bij `tags_ptr` en die pointer aanpassen naar een
nieuwe, 8 bytes uitgelijnde positie.

### Lokale functies

De plugins kunnen lokale functies gebruiken, maar vanwege een CLang-bug *moeten* deze worden gedeclareerd als `static`. (De bug
is dat CLang PLT-records genereert voor deze functies, zelfs als de vlag "-fno-plt" wordt doorgegeven op de opdrachtregel. Het
gebruik van `static` is hiervoor een oplossing).

Specificatie voor laag-niveau bestandsformaat
---------------------------------------------

Als iemand een plugin wil schrijven in een andere taal dan C (bijvoorbeeld in Assembly), volgt hier een korte beschrijving van
het bestandsformaat.

Het lijkt erg op het a.out-formaat. Het bestand bestaat uit een header met een vaste grootte, gevolgd door secties met
verschillende lengtes. Er is geen sectieheader, de gegevens van elke sectie volgen direct op de gegevens van de vorige sectie
in de volgende volgorde:

```
(koptekst)
(identificatie match records)
(relocation records)
(machinecode)
(alleen-lezen data)
(geïnitialiseerde leesbare-schrijfbare data)
```

Voor de eerste echte sectie, machinecode, is de uitlijning inbegrepen. Voor alle andere secties wordt padding toegevoegd aan
de grootte van de vorige sectie.

TIP: als u een plugin als een enkel argument aan `plgld` doorgeeft, dan dumpt het de secties in het bestand met een uitvoer
die lijkt op `readelf -a` of `objdump -xd`.

### Koptekst

Alle getallen hebben het little-endian-formaat, ongeacht de architectuur.

| Offset  | Maat  | Beschrijving                                                   |
|--------:|------:|----------------------------------------------------------------|
|       0 |     4 | magische bytes `EPLG`                                          |
|       4 |     4 | totale grootte van het bestand                                 |
|       8 |     4 | totaal geheugen vereist wanneer bestand wordt geladen          |
|      12 |     4 | grootte van het codegedeelte                                   |
|      16 |     4 | grootte van het alleen-lezen gegevensgedeelte                  |
|      20 |     4 | invoerpunt van de plug-in                                      |
|      24 |     2 | architectuurcode (hetzelfde als die van ELF)                   |
|      26 |     2 | aantal verhuisrecords                                          |
|      28 |     1 | aantal identificatiematchrecords                               |
|      29 |     1 | hoogste gerefereerde GOT-vermelding                            |
|      30 |     1 | bestandsformaat revisie (0 voor nu)                            |
|      31 |     1 | plugintype (1=bestandssysteem, 2=kernel, 3=decompressor, 4=tag) |

De architectuurcode is dezelfde als in ELF-headers, bijvoorbeeld 62 = x86_64, 183 = Aarch64 en 243 = RISC-V.

Het type van de plugin specificeert het prototype van het toegangspunt, de ABI is altijd SysV.

### Sectie Identificatieovereenkomst

Deze sectie bevat evenveel van de volgende records als is opgegeven in het veld "aantal identificatiematchrecords" in de koptekst.

| Offset  | Maat  | Beschrijving                                             |
|--------:|------:|----------------------------------------------------------|
|       0 |     2 | offset                                                   |
|       2 |     1 | maat                                                     |
|       3 |     1 | type                                                     |
|       4 |     4 | magische bytes om te matchen                             |

Eerst wordt het begin van het onderwerp in een buffer geladen. Er wordt een accumulator opgezet, aanvankelijk met een 0. Offsets
in deze records zijn altijd relatief ten opzichte van deze accumulator en ze adresseren die byte in de buffer.

Het veld Type vertelt hoe de offset moet worden geïnterpreteerd. Als het een 1 is, wordt de offset plus de accumulator als waarde
gebruikt. Als het 2 is, wordt een 8-bits bytewaarde genomen bij offset, 3 betekent een 16-bits woordwaarde nemen en 4 betekent
een 32-bits dword-waarde nemen. 5 betekent een 8-bits bytewaarde nemen en de accumulator eraan toevoegen, 6 betekent een 16-bits
woordwaarde nemen en de accumulator eraan toevoegen en 7 is hetzelfde maar met een 32-bits waarde. 8 zoekt naar de magische bytes
van de accumulatorth byte tot het einde van de buffer in offsetstappen en retourneert, indien gevonden, de overeenkomende offset
als de waarde.

Als de grootte nul is, wordt de accumulator ingesteld op de waarde. Als de grootte niet nul is, wordt gecontroleerd of dat aantal
bytes overeenkomt met de opgegeven magische bytes.

Om bijvoorbeeld te controleren of een PE-uitvoerbaar bestand begint met een NOP-instructie:
```c
EASY_PLUGIN(PLG_T_KERNEL) {
   /* offset  maat  wedstrijdtype  magische bytes */
    { 0,         2, PLG_M_CONST, { 'M', 'Z', 0, 0 } },      /* controleer magische bytes */
    { 60,        0, PLG_M_DWORD, { 0, 0, 0, 0 } },          /* verkrijg de offset van de PE-header naar de accumulator */
    { 0,         4, PLG_M_CONST, { 'P', 'E', 0, 0 } },      /* controleer magische bytes */
    { 40,        1, PLG_M_DWORD, { 0x90, 0, 0, 0 } }        /* controleer op NOP-instructie bij het binnenkomstpunt */
};
```

### Verhuisafdeling

Deze sectie bevat evenveel van de volgende records als er zijn opgegeven in het veld "aantal verhuisrecords" in de koptekst.

| Offset  | Maat  | Beschrijving                                             |
|--------:|------:|----------------------------------------------------------|
|       0 |     4 | offset                                                   |
|       4 |     4 | type verhuizing                                          |

Betekenis van bits in type:

| Van     | Naar  | Beschrijving                                             |
|--------:|------:|----------------------------------------------------------|
|       0 |     7 | symbool (0 - 255)                                        |
|       8 |     8 | PC relatieve adressering                                 |
|       9 |     9 | GOT relatieve indirecte adressering                      |
|      10 |    13 | onmiddellijke maskerindex (0 - 15)                       |
|      14 |    19 | begin bit (0 - 63)                                       |
|      20 |    25 | eind bit (0 - 63)                                        |
|      26 |    31 | genegeerde adresvlag bitpositie (0 - 63)                 |

Het offsetveld is relatief ten opzichte van de magie in de header van de plugin en selecteert een geheel getal in het geheugen
waar de verplaatsing moet worden uitgevoerd.

Het symbool vertelt welk adres gebruikt moet worden. 0 betekent het BASE-adres waar de plugin in het geheugen geladen is, oftewel
het adres van de header's magic in het geheugen. Andere waarden selecteren een extern symbooladres van de GOT, gedefinieerd in de
loader of in een andere plugin, bekijk de `plg_got` array in de plgld.c's source om te zien welke waarde overeenkomt met welk
symbool. Als de GOT relative bit 1 is, dan wordt het adres van de GOT-invoer van het symbool gebruikt, in plaats van het werkelijke
adres van het symbool.

Als de relatieve PC-bit 1 is, wordt de offset eerst van het adres afgetrokken (relatieve adresseringsmodus van de instructiewijzer).

De immediate mask index vertelt welke bits het adres in de instructie opslaan. Als dit 0 is, wordt het adres as-is naar de offset
geschreven, ongeacht de architectuur. Voor x86_64 is alleen index 0 toegestaan. Voor ARM Aarch64: 0 = as-is, 1 = 0x07ffffe0 (5
bits naar links verschuiven), 2 = 0x07fffc00 (10 bits naar links verschuiven), 3 = 0x60ffffe0 (met ADR/ADRP-instructies wordt de
immediate verschoven en gesplitst in twee bitgroepen). Toekomstige architecturen definiëren mogelijk meer en verschillende
immediate bitmasks.

Met behulp van het onmiddellijke masker worden end - start + 1 bits uit het geheugen gehaald en signed extended. Deze waarde wordt
toegevoegd aan het adres (addend, en in het geval van interne verwijzingen wordt het adres van het interne symbool hier ook
gecodeerd).

Als de genegeerde adresvlagbit niet 0 is en het adres positief is, wordt die bit gewist. Als het adres negatief is, wordt die bit
ingesteld en wordt het adres genegeerd.

Ten slotte selecteren de start- en eindbits welk deel van het adres naar het geselecteerde gehele getal moet worden geschreven.
Dit definieert ook de grootte van de verplaatsing, bits buiten dit bereik en bits die geen deel uitmaken van het onmiddellijke
masker blijven ongewijzigd.

### Code Sectie

Deze sectie bevat machine-instructies voor de architectuur die in de header is gespecificeerd en heeft evenveel bytes als het veld
codegrootte aangeeft.

### Alleen-lezen gegevenssectie

Dit is een optionele sectie, die mogelijk ontbreekt. Het is zo lang als het read-only section size veld in de header aangeeft.
Alle constante variabelen worden in deze sectie geplaatst.

### Geïnitialiseerde gegevenssectie

Dit is een optionele sectie, die mogelijk ontbreekt. Als er nog bytes in het bestand staan ​​na de codesectie (of de optionele
alleen-lezen gegevenssectie), dan worden die bytes allemaal beschouwd als de datasectie. Als een variabele wordt geïnitialiseerd
met een waarde die niet nul is, dan wordt deze in deze sectie geplaatst.

### BSS-sectie

Dit is een optionele sectie, die mogelijk ontbreekt. Deze sectie wordt nooit in het bestand opgeslagen. Als het veld in memory
size groter is dan het veld file size in de header, wordt het verschil in het geheugen opgevuld met nullen. Als een variabele
niet is geïnitialiseerd of is geïnitialiseerd als nul, wordt deze in deze sectie geplaatst.
