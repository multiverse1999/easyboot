Scrittura di kernel compatibili con Easyboot
============================================

[Easyboot](https://gitlab.com/bztsrc/easyboot) supporta vari kernel tramite [plugin](plugins.md). Ma se non viene trovato alcun
plugin adatto, ricorre ai binari ELF64 o PE32+ con una variante semplificata (non c'è bisogno di incorporare nulla) del protocollo
Multiboot2.

Questo è lo stesso protocollo utilizzato da [Simpleboot](https://gitlab.com/bztsrc/simpleboot); tutti i kernel di esempio in quel
repository devono funzionare anche con **Easyboot**.

Puoi usare l'intestazione multiboot2.h originale nel repository di GRUB, o il file di intestazione C/C++ [easyboot.h](../../easyboot.h)
per ottenere typedef più facili da usare. Il formato binario di basso livello è lo stesso, puoi anche usare qualsiasi libreria
Multiboot2 esistente, anche con linguaggi diversi dal C, come questa libreria [Rust](https://github.com/rust-osdev/multiboot2/tree/main/multiboot2/src)
ad esempio (nota: non sono in alcun modo affiliato a quegli sviluppatori, ho solo cercato "Rust Multiboot2" e quello è stato il
primo risultato).

[[_TOC_]]

Sequenza di avvio
-----------------

### Avvio del caricatore

Nelle macchine *BIOS*, il primo settore del disco viene caricato su 0:0x7C00 dal firmware e il controllo gli viene passato. In
questo settore **Easyboot** ha [boot_x86.asm](../../src/boot_x86.asm), che è abbastanza intelligente da individuare e caricare il
caricatore di 2° stadio e anche da impostare la modalità lunga per esso.

Sulle macchine *UEFI* lo stesso file di 2° stadio, chiamato `EFI/BOOT/BOOTX64.EFI`, viene caricato direttamente dal firmware. La
fonte per questo loader può essere trovata in [loader_x86.c](../../src/loader_x86.c). Ecco fatto, **Easyboot** non è GRUB né
syslinux, entrambi richiedono decine e decine di file di sistema sul disco. Qui non servono altri file, solo questo (i plugin sono
opzionali, nessuno è necessario per fornire la compatibilità Multiboot2).

Su *Raspberry Pi* il caricatore si chiama `KERNEL8.IMG`, compilato da [loader_rpi.c](../../src/loader_rpi.c).

### Il caricatore

Questo loader è scritto con molta attenzione per funzionare su più configurazioni. Carica la GUID Partitining Table dal disco e
cerca una "EFI System Partition". Quando la trova, cerca il file di configurazione `easyboot/menu.cfg` su quella partizione di
avvio. Dopo aver selezionato l'opzione di avvio e aver conosciuto il nome del file del kernel, il loader lo individua e lo carica.

Quindi rileva automaticamente il formato del kernel ed è abbastanza intelligente da interpretare le informazioni di sezione e
segmento su dove caricare cosa (esegue il mapping della memoria su richiesta ogni volta che è necessario). Quindi imposta un
ambiente appropriato a seconda del protocollo di avvio rilevato (Multiboot2 / Linux / ecc. modalità protetta o lunga, argomenti
ABI ecc.). Dopo che lo stato della macchina è solido e ben definito, come ultimo atto, il loader salta al punto di ingresso del
kernel.

Stato della macchina
--------------------

Tutto ciò che è scritto nella [specifica Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) sullo
stato della macchina è valido, fatta eccezione per i registri di uso generale. **Easyboot** passa due argomenti al punto di
ingresso del kernel in base alla SysV ABI e alla Microsoft fastcall ABI. Il primo parametro è la magia, il secondo è un indirizzo
di memoria fisica, che punta a un taglist Multiboot Information (abbreviato in MBI in seguito, vedi sotto).

Inoltre, violiamo un po' il protocollo Multiboot2 per gestire i kernel della metà superiore. Multiboot2 impone che la memoria debba
essere mappata in base all'identità. Bene, in **Easyboot** questo è vero solo in parte: garantiamo solo che tutta la RAM fisica sia
sicuramente mappata in base all'identità come previsto; tuttavia, alcune regioni superiori (a seconda delle intestazioni di
programma del kernel) potrebbero essere ancora disponibili. Ciò non interrompe i normali kernel conformi a Multiboot2, che non
dovrebbero accedere alla memoria al di fuori della RAM fisica disponibile.

Il tuo kernel viene caricato esattamente nello stesso modo sia sui sistemi BIOS che UEFI e anche su RPi, le differenze di firmware
sono solo "problemi di qualcun altro". L'unica cosa che il tuo kernel vedrà è se l'MBI contiene o meno il tag della tabella di
sistema EFI. Per semplificarti la vita, **Easyboot** non genera nemmeno il tag della mappa di memoria EFI (tipo 17), fornisce solo
il tag [Mappa di memoria](#mappa_di_memoria) (tipo 6) indiscriminatamente su tutte le piattaforme (anche sui sistemi UEFI, lì la
mappa di memoria viene semplicemente convertita per te, quindi il tuo kernel deve gestire solo un tipo di tag di elenco di memoria).
Anche i tag vecchi e obsoleti vengono omessi e non vengono mai generati da questo boot manager.

Il kernel viene eseguito a livello di supervisore (ring 0 su x86, EL1 su ARM), possibilmente su tutti i core della CPU in parallelo.

GDT non specificato, ma valido. Lo stack è impostato nei primi 640k e cresce verso il basso (ma dovresti cambiarlo il prima
possibile con qualsiasi stack tu ritenga degno). Quando SMP è abilitato, tutti i core hanno i propri stack e l'ID del core è in
cima allo stack (ma puoi anche ottenere l'ID del core nel solito modo specifico della piattaforma, usando cpuid / mpidr / ecc.).

Dovresti considerare IDT come non specificato; IRQ, NMI e interrupt software disabilitati. I gestori di eccezioni fittizi sono
impostati per visualizzare un dump minimo e arrestare la macchina. Ci si dovrebbe basare su di essi solo per segnalare se il kernel
va in tilt prima di essere in grado di impostare il proprio IDT e i gestori, preferibilmente il prima possibile. Su ARM vbar_el1 è
impostato per chiamare gli stessi gestori di eccezioni fittizi (anche se scaricano registri diversi, ovviamente).

Anche framebuffer è impostato di default. Puoi modificare la risoluzione in config, ma se non viene specificato, framebuffer è
comunque configurato.

È importante non tornare mai dal kernel. Sei libero di sovrascrivere qualsiasi parte del loader in memoria (non appena hai finito
con i tag MBI), quindi non c'è semplicemente nessun posto a cui tornare. "Der Mohr hat seine Schuldigkeit getan, der Mohr kann gehen."

Informazioni di avvio passate al kernel (MBI)
---------------------------------------------

Non è ovvio a prima vista, ma la specifica Multiboot2 definisce in realtà due set di tag totalmente indipendenti:

- Il primo set dovrebbe essere inline in un kernel compatibile con Multiboot2, chiamato header Multiboot2 dell'immagine del sistema
  operativo (sezione 3.1.2), quindi *fornito dal kernel*. **Easyboot** non si preoccupa di questi tag e non analizza il tuo kernel
  per questi. Semplicemente non hai bisogno di dati magici speciali incorporati nel tuo file kernel con **Easyboot**, gli header
  ELF e PE sono sufficienti.

- Il secondo set viene *passato al kernel* dinamicamente all'avvio, **Easyboot** usa solo questi tag. Tuttavia, non genera tutto ciò
  che Multiboot2 specifica (omette semplicemente quelli vecchi, obsoleti o legacy). Questi tag sono chiamati tag MBI, vedere
  [Boot information](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Boot-information-format) (sezione 3.6).

NOTA: la specifica Multiboot2 sui tag MBI è piena di bug. Di seguito puoi trovare una versione corretta, che si allinea con il file
header multiboot2.h che puoi trovare nel repository sorgente di GRUB.

Il primo parametro del kernel è il magico 0x36d76289 (in `rax`, `rcx` e `rdi`). Puoi individuare i tag MBI usando il secondo
parametro (in `rbx`, `rdx` e `rsi`). Sulla piattaforma ARM, il magico è in `x0` e l'indirizzo è in `x1`. Su RISC-V e MIPS vengono
usati rispettivamente `a0` e `a1`. Se e quando questo loader viene portato su un'altra architettura, devono sempre essere usati i
registri specificati da SysV ABI per gli argomenti della funzione. Se ci sono altre ABI comuni sulla piattaforma che non
interferiscono con SysV ABI, allora i valori devono essere duplicati anche nei registri di quelle ABI (o in cima allo stack).

### Intestazioni

L'indirizzo passato è sempre allineato a 8 byte e inizia con un'intestazione MBI:

```
        +-------------------+
u32     | total_size        |
u32     | reserved          |
        +-------------------+
```

Segue una serie di tag allineati anch'essi da 8 byte. Ogni tag inizia con i seguenti campi di intestazione tag:

```
        +-------------------+
u32     | type              |
u32     | size              |
        +-------------------+
```

`type` contiene un identificatore del contenuto del resto del tag. `size` contiene la dimensione del tag, inclusi i campi di
intestazione, ma escluso il padding. I tag si susseguono l'uno con l'altro, con padding quando necessario, in modo che ogni tag
inizi con un indirizzo allineato di 8 byte.

### Terminatore

```
        +-------------------+
u32     | type = 0          |
u32     | size = 8          |
        +-------------------+
```

I tag terminano con un tag di tipo `0` e dimensione `8`.

### Riga di comando di avvio

```
        +-------------------+
u32     | type = 1          |
u32     | size              |
u8[n]   | string            |
        +-------------------+
```

`string` contiene la riga di comando specificata nella riga `kernel` di *menuentry* (senza il percorso e il nome del file del
kernel). La riga di comando è una normale stringa UTF-8 terminata da zero in stile C.

### Nome del boot loader

```
        +----------------------+
u32     | type = 2             |
u32     | size = 17            |
u8[n]   | string "Easyboot"    |
        +----------------------+
```

`string` contiene il nome di un boot loader che avvia il kernel. Il nome è una normale stringa UTF-8 terminata da zero in stile C.

### Moduli

```
        +-------------------+
u32     | type = 3          |
u32     | size              |
u32     | mod_start         |
u32     | mod_end           |
u8[n]   | string            |
        +-------------------+
```

Questo tag indica al kernel quale modulo di avvio è stato caricato insieme all'immagine del kernel e dove può essere trovato.
`mod_start` e `mod_end` contengono gli indirizzi fisici di inizio e fine del modulo di avvio stesso. Non otterrai mai un buffer
compresso con gzip, perché **Easyboot** li decomprime in modo trasparente per te (e se fornisci un plugin, funziona anche con dati
compressi diversi da gzip). Il campo `string` fornisce una stringa arbitraria da associare a quel particolare modulo di avvio; è
una normale stringa UTF-8 terminata da zero in stile C. Specificata nella riga `module` di *menuentry* e il suo utilizzo esatto è
specifico del sistema operativo. A differenza del tag della riga di comando di avvio, i tag del modulo *includono anche* il
percorso e il nome file del modulo.

Appare un tag per modulo. Questo tipo di tag può apparire più volte. Se un ramdisk iniziale è stato caricato insieme al kernel,
allora apparirà come primo modulo.

C'è un caso speciale: se il file è una tabella DSDT ACPI, un blob FDT (dtb) o GUDT, allora non apparirà come un modulo, ma piuttosto
il ACPI vecchio RSDP (tipo 14) o il ACPI nuovo RSDP (tipo 15) verranno patchati e il loro DSDT sostituito con il contenuto di questo
file.

### Mappa di memoria

Questo tag fornisce una mappa della memoria.

```
        +-------------------+
u32     | type = 6          |
u32     | size              |
u32     | entry_size = 24   |
u32     | entry_version = 0 |
varies  | entries           |
        +-------------------+
```

`size` contiene la dimensione di tutte le voci, incluso questo campo stesso. `entry_size` è sempre 24. `entry_version` è impostato
su `0`. Ogni voce ha la seguente struttura:

```
        +-------------------+
u64     | base_addr         |
u64     | length            |
u32     | type              |
u32     | reserved          |
        +-------------------+
```

`base_addr` è l'indirizzo fisico iniziale. `length` è la dimensione della regione di memoria in byte. `type` è la varietà
dell'intervallo di indirizzi rappresentato, dove il valore `1` indica RAM disponibile, il valore `3` indica memoria utilizzabile
contenente informazioni ACPI, il valore `4` indica memoria riservata che deve essere preservata durante la sospensione, il valore
`5` indica una memoria occupata da moduli RAM difettosi e tutti gli altri valori indicano attualmente un'area riservata. `reserved`
è impostato su `0` all'avvio del BIOS.

Quando l'MBI viene generato su una macchina UEFI, varie voci della mappa di memoria EFI vengono memorizzate come tipo `1` (RAM
disponibile) o `2` (RAM riservata) e, se necessario, il tipo di memoria EFI originale viene inserito nel campo `reserved`.

La mappa fornita garantisce di elencare tutta la RAM standard che dovrebbe essere disponibile per un uso normale, ed è sempre
ordinata in base a `base_addr` crescente. Questo tipo di RAM disponibile include tuttavia le regioni occupate da kernel, mbi,
segmenti e moduli. Il kernel deve fare attenzione a non sovrascrivere queste regioni (**Easyboot** potrebbe facilmente escludere
tali regioni, ma ciò interromperebbe la compatibilità Multiboot2).

### Informazioni sul framebuffer

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

Il campo `framebuffer_addr` contiene l'indirizzo fisico del framebuffer. Il campo `framebuffer_pitch` contiene la lunghezza di una
riga in byte. I campi `framebuffer_width`, `framebuffer_height` contengono le dimensioni del framebuffer in pixel. Il campo
`framebuffer_bpp` contiene il numero di bit per pixel. `framebuffer_type` è sempre impostato su 1 e `reserved` contiene sempre 0
nella versione corrente della specifica e deve essere ignorato dall'immagine del sistema operativo. I campi rimanenti descrivono
il formato pixel compresso, la posizione e la dimensione dei canali in bit. Puoi usare l'espressione
`((~(0xffffffff << size)) << position) & 0xffffffff` per ottenere una maschera di canale simile a UEFI GOP.

### Puntatore della tabella di sistema EFI a 64 bit

Questo tag esiste solo se **Easyboot** è in esecuzione su una macchina UEFI. Su una macchina BIOS questo tag non è mai stato
generato.

```
        +-------------------+
u32     | type = 12         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Questo tag contiene un puntatore a EFI system table.

### Puntatore di gestione dell'immagine EFI a 64 bit

Questo tag esiste solo se **Easyboot** è in esecuzione su una macchina UEFI. Su una macchina BIOS questo tag non è mai stato
generato.

```
        +-------------------+
u32     | type = 20         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Questo tag contiene un puntatore a EFI image handle. Di solito si tratta dell'handle dell'immagine del boot loader.

### Tabelle SMBIOS

```
        +-------------------+
u32     | type = 13         |
u32     | size              |
u8      | major             |
u8      | minor             |
u8[6]   | reserved          |
        | smbios tabelle    |
        +-------------------+
```

Questo tag contiene una copia delle tabelle SMBIOS e la loro versione.

### ACPI vecchio RSDP

```
        +-------------------+
u32     | type = 14         |
u32     | size              |
        | copia di RSDPv1   |
        +-------------------+
```

Questo tag contiene una copia di RSDP come definito dalla specifica ACPI 1.0. (Con un indirizzo a 32 bit.)

### ACPI nuovo RSDP

```
        +-------------------+
u32     | type = 15         |
u32     | size              |
        | copia di RSDPv2   |
        +-------------------+
```

Questo tag contiene una copia di RSDP come definito dalle specifiche ACPI 2.0 o successive. (Probabilmente con un indirizzo a 64 bit.)

Questi (tipo 14 e 15) puntano a una tabella `RSDT` o `XSDT` con un puntatore a una tabella `FACP`, che a sua volta contiene due
puntatori a una tabella `DSDT`, che descrive la macchina. **Easyboot** falsifica queste tabelle su macchine che altrimenti non
supportano ACPI. Inoltre, se fornisci una tabella DSDT, un blob FDT (dtb) o GUDT come modulo, **Easyboot** applicherà una patch ai
puntatori per puntare a quella tabella fornita dall'utente. Per analizzare queste tabelle, puoi usare la mia libreria
[hwdet](https://gitlab.com/bztsrc/hwdet) a intestazione singola e senza dipendenze (o le gonfie
[apcica](https://github.com/acpica/acpica) e [libfdt](https://github.com/dgibson/dtc)).

Tag specifici del kernel
------------------------

I tag con `type` maggiore o uguale a 256 non fanno parte della specifica Multiboot2, ma sono comunque forniti da **Easyboot**.
Questi potrebbero essere aggiunti tramite [plugins](plugins.md) opzionali all'elenco, se e quando un kernel ne ha bisogno.

### EDID

```
        +-------------------+
u32     | type = 256        |
u32     | size              |
        | copia di EDID     |
        +-------------------+
```

Questo tag contiene una copia dell'elenco delle risoluzioni dei monitor supportate in base alle specifiche EDID.

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

Questo tag esiste se è stata data la direttiva `multicore`. `numcores` contiene il numero di core CPU nel sistema, `running` è il
numero di core che hanno inizializzato con successo e che eseguono lo stesso kernel in parallelo. `bspid` contiene l'identificativo
del core BSP (su ID lAPIC x86), in modo che i kernel possano distinguere gli AP ed eseguire un codice diverso su di essi. Tutti gli
AP hanno il loro stack e in cima allo stack ci sarà l'ID del core corrente.

### Identificatori di partizione

```
        +-------------------+
u32     | type = 258        |
u32     | size = 24 / 40    |
u128    | bootuuid          |
u128    | rootuuid          |
        +-------------------+
```

Questo tag contiene i campi identificativi univoci nel GPT delle partizioni di avvio e root. Se l'avvio non utilizza una tabella di
partizionamento GUID, `bootuuid` viene generato come `54524150-(codice dispositivo)-(numero partizione)-616F6F7400000000`.

Disposizione della memoria
--------------------------

### Macchine BIOS

| Inizio   | Fine    | Descrizione                                                                |
|---------:|--------:|----------------------------------------------------------------------------|
|      0x0 |   0x400 | Tabella dei vettori di interruzione (utilizzabile, modalità reale IDT)     |
|    0x400 |   0x4FF | Area dati BIOS (utilizzabile)                                              |
|    0x4FF |   0x500 | Codice dell'unità del BIOS (molto probabilmente 0x80, utilizzabile)        |
|    0x500 |   0x5A0 | dati di sincronizzazione per SMP (utilizzabili)                            |
|    0x5A0 |  0x1000 | stack del gestore delle eccezioni (utilizzabile dopo aver impostato l'IDT) |
|   0x1000 |  0x8000 | tabelle di paging (utilizzabili dopo aver impostato le tabelle di paging)  |
|   0x8000 | 0x20000 | codice e dati del caricatore (utilizzabili dopo aver impostato l'IDT)      |
|  0x20000 | 0x40000 | config + tag (utilizzabili dopo l'analisi di MBI)                          |
|  0x40000 | 0x90000 | ID dei plugin; dall'alto verso il basso: stack del kernel                  |
|  0x90000 | 0x9A000 | Solo kernel Linux: zero page + cmdline                                     |
|  0x9A000 | 0xA0000 | Area dati BIOS estesa (meglio non toccare)                                 |
|  0xA0000 | 0xFFFFF | VRAM e BIOS ROM (non utilizzabili)                                         |
| 0x100000 |       x | segmenti del kernel, seguiti dai moduli, ogni pagina allineata             |

### Macchine UEFI

Nessuno lo sa. UEFI alloca la memoria come vuole. Aspettatevi di tutto e di più. Tutte le aree saranno sicuramente elencate nella
mappa di memoria come tipo = 1 (`MULTIBOOT_MEMORY_AVAILABLE`) e riservate = 2 (`EfiLoaderData`), tuttavia questo non è esclusivo,
anche altri tipi di memoria potrebbero essere elencati in questo modo (la sezione bss del boot manager, ad esempio).

### Raspberry Pi

| Inizio   | Fine    | Descrizione                                                                 |
|---------:|--------:|-----------------------------------------------------------------------------|
|      0x0 |   0x500 | riservato dal firmware (meglio non toccare)                                 |
|    0x500 |   0x5A0 | dati di sincronizzazione per SMP (utilizzabili)                             |
|    0x5A0 |  0x1000 | stack del gestore delle eccezioni (utilizzabile dopo aver impostato l'VBAR) |
|   0x1000 |  0x9000 | tabelle di paging (utilizzabili dopo aver impostato le tabelle di paging)   |
|   0x9000 | 0x20000 | codice e dati del caricatore (utilizzabili dopo aver impostato l'VBAR)      |
|  0x20000 | 0x40000 | config + tag (utilizzabili dopo l'analisi di MBI)                           |
|  0x40000 | 0x80000 | firmware fornito FDT (dtb); dall'alto verso il basso: stack del kernel      |
| 0x100000 |       x | segmenti del kernel, seguiti dai moduli, ogni pagina allineata              |

I primi byte sono riservati per [armstub](https://github.com/raspberrypi/tools/blob/master/armstubs/armstub8.S). Solo il core 0 è
partito, quindi per avviare Application Processors, scrivi l'indirizzo di una funzione su 0xE0 (core 1), 0xE8 (core 2), 0xF0 (core 3),
i cui indirizzi si trovano in quest'area. Questo è irrilevante quando viene usata la direttiva `multicore`, quindi tutti i core
eseguiranno il kernel.

Sebbene non supportato nativamente su RPi, si ottiene comunque un vecchio tag RSDP ACPI (tipo 14), con tabelle false. La tabella
`APIC` viene utilizzata per comunicare al kernel il numero di core CPU disponibili. L'indirizzo della funzione di avvio è memorizzato
nel campo RSD PTR-> RSDT -> APIC -> cpu\[x].apic_id (e core id in cpu\[x].acpi_id, dove BSP è sempre cpu\[0].acpi_id = 0 e
cpu\[0].apic_id = 0xD8. Attenzione, "acpi" e "apic" sembrano piuttosto simili).

Se il firmware passa un blob FDT valido, o se uno dei moduli è un file .dtb, .gud o .aml, viene aggiunta anche una tabella FADT (con
`FACP` magico). In questa tabella, il puntatore DSDT (32 bit, all'offset 40) punta al blob dell'albero dei dispositivi appiattito
fornito.

Sebbene il firmware non fornisca alcuna funzionalità di mappatura della memoria, otterrai comunque un tag di mappa di memoria (tipo 6),
che elenca la RAM rilevata e la regione MMIO. Puoi usarlo per rilevare l'indirizzo di base dell'MMIO, che è diverso su RPi3 e RPi4.
