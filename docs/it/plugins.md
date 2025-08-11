Plugin Easyboot
===============

Di default, [Easyboot](https://gitlab.com/bztsrc/easyboot) avvia i kernel compatibili con Multiboot2 nei formati ELF e PE dalla
partizione di avvio. Se il tuo kernel usa un formato di file diverso, un protocollo di avvio diverso o non si trova sulla
partizione di avvio, allora avrai bisogno di plugin sulla partizione di avvio. Puoi trovarli nella directory
[src/plugins](../../src/plugins).

[[_TOC_]]

Installazione
-------------

Per installare i plugin, è sufficiente copiarli nella directory specificata nel parametro `(indir)`, nella sottodirectory `easyboot`
accanto al file menu.cfg.

Per esempio:
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
$ easyboot bootpart disco.img
```

Compilazione
------------

*Era ovvio fin dall'inizio che ELF non era adatto a questo compito. È troppo gonfio e troppo complesso. Quindi in origine volevo
usare struct exec (il classico formato UNIX a.out), ma sfortunatamente le moderne toolchain non possono più crearlo. Quindi ho
deciso di creare il mio formato e il mio linker per i plugin.*

Puoi compilare la sorgente del plugin con qualsiasi cross-compilatore ANSI C standard in un file oggetto ELF, ma poi dovrai usare
il linker [plgld](../../src/misc/plgld.c) per creare il binario finale. Questo è un cross-linker indipendente dall'architettura,
funzionerà indipendentemente dal codice macchina per cui è stato compilato il plugin. Il .plg finale è solo una frazione delle
dimensioni del .o ELF da cui è stato generato.

### Plugin API

La sorgente C di un plugin deve includere il file header `src/loader.h` e deve contenere una riga `EASY_PLUGIN`. Questa ha un
parametro, il tipo di plugin, seguito dalla specifica di corrispondenza dell'identificatore. Quest'ultima è usata dal loader per
determinare quando usare quel particolare plugin.

Per esempio:

```c
#include "../loader.h"

/* byte magici che identificano un kernel Linux */
EASY_PLUGIN(PLG_T_KERNEL) {
   /* offset   dim  tipo           byte magici */
    { 0x1fe,     2, PLG_M_CONST, { 0xAA, 0x55, 0, 0 } },
    { 0x202,     4, PLG_M_CONST, { 'H', 'd', 'r', 'S' } }
};

/* punto di ingresso, il prototipo è definito dal tipo del plugin */
PLG_API void _start(uint8_t *buf, uint64_t size);
{
    /* preparare l'ambiente per un kernel Linux */
}
```

I plugin possono utilizzare numerose variabili e funzioni, tutte definite nel file di intestazione e collegate in fase di esecuzione.

```c
uint32_t verbose;
```
Livello di verbosità. Un plugin può stampare qualsiasi output solo se questo è diverso da zero, fatta eccezione per i messaggi di
errore. Più alto è il suo valore, più dettagli dovrebbero essere stampati.

```c
uint64_t file_size;
```
Dimensione totale del file aperto (vedere `open` e `loadfile` di seguito).

```c
uint8_t *root_buf;
```
Quando un plugin del file system si inizializza, questo contiene i primi 128k della partizione (si spera includendo il super blocco).
In seguito un plugin del file system può riutilizzare questo buffer da 128k per qualsiasi scopo (cache FAT, cache inode ecc.)

```c
uint8_t *tags_buf;
```
Contiene i tag Multiboot2. Un plugin del kernel può analizzarli per convertire i dati forniti dal boot manager in qualsiasi formato
previsto dal kernel. Questo puntatore punta all'inizio del buffer.

```c
uint8_t *tags_ptr;
```
Questo puntatore punta alla fine del buffer dei tag Multiboot2. I plugin dei tag potrebbero aggiungere nuovi tag qui e adattare
questo puntatore.

```c
uint8_t *rsdp_ptr;
```
Punta al puntatore RSDP ACPI.

```c
uint8_t *dsdt_ptr;
```
Punta al blob di descrizione hardware DSDT (o GUDT, FDT).

```c
efi_system_table_t *ST;
```
Sulle macchine UEFI punta alla tabella di sistema EFI, altrimenti `NULL`.

```c
void memset(void *dst, uint8_t c, uint32_t n);
void memcpy(void *dst, const void *src, uint32_t n);
int  memcmp(const void *s1, const void *s2, uint32_t n);
```
Funzioni di memoria obbligatorie (il compilatore C potrebbe generare chiamate a queste funzioni, anche quando non c'è una chiamata
diretta).

```c
void *alloc(uint32_t num);
```
Assegna `num` pagine (4k) di memoria. I plugin non devono allocare molto, devono puntare a un ingombro di memoria minimo.

```c
void free(void *buf, uint32_t num);
```
Libera la memoria precedentemente allocata di `num` pagine.

```c
void printf(char *fmt, ...);
```
Stampa una stringa formattata sulla console di avvio.

```c
uint64_t pb_init(uint64_t size);
```
Avvia la barra dei progressi, `size` è la dimensione totale che rappresenta. Restituisce quanti byte vale un pixel.

```c
void pb_draw(uint64_t curr);
```
Disegna la barra dei progressi per il valore corrente. `curr` deve essere compreso tra 0 e la dimensione totale.

```c
void pb_fini(void);
```
Chiude la barra dei progressi, cancella la sua posizione sullo schermo.

```c
void loadsec(uint64_t sec, void *dst);
```
Utilizzato dai plugin del file system, carica un settore dal disco alla memoria. `sec` è il numero del settore, relativo alla
partizione root.

```c
void sethooks(void *o, void *r, void *c);
```
Utilizzato dai plugin del file system, imposta gli hook delle funzioni di open / read / close per il file system della partizione
root.

```c
int open(char *fn);
```
Apre un file sulla partizione root (o boot) per la lettura, restituisce 1 in caso di successo. È possibile aprire un solo file alla
volta. Quando non c'è stata alcuna chiamata `sethooks` in precedenza, allora opera sulla partizione boot.

```c
uint64_t read(uint64_t offs, uint64_t size, void *buf);
```
Legge i dati dal file aperto nella posizione di ricerca `offs` nella memoria e restituisce il numero di byte effettivamente letti.

```c
void close(void);
```
Chiude il file aperto.

```c
uint8_t *loadfile(char *path);
```
Carica un file interamente dalla partizione root (o boot) in un buffer di memoria appena allocato e decomprimilo in modo trasparente
se il plugin lo trova. La dimensione è restituita in `file_size`.

```c
int loadseg(uint32_t offs, uint32_t filesz, uint64_t vaddr, uint32_t memsz);
```
Carica un segmento dal buffer del kernel. Questo controlla se la memoria `vaddr` è disponibile e mappa il segmento se è la metà
superiore. `offs` è l'offset del file, quindi relativo al buffer del kernel. Se `memsz` è più grande di `filesz`, la differenza
viene riempita con degli zeri.

```c
void _start(void);
```
Punto di ingresso per i plugin del file system (`PLG_T_FS`). Dovrebbe analizzare il superblocco in `root_buf` e chiamare `sethooks`.
In caso di errore dovrebbe semplicemente ritornare senza impostare i suoi hook.

```c
void _start(uint8_t *buf, uint64_t size);
```
Punto di ingresso per i plugin del kernel (`PLG_T_KERNEL`). Riceve l'immagine del kernel in memoria, dovrebbe riposizionare i suoi
segmenti, impostare l'ambiente appropriato e trasferire il controllo. Quando non ci sono errori, non torna mai.

```c
uint8_t *_start(uint8_t *buf);
```
Punto di ingresso per i plugin decompressori (`PLG_T_DECOMP`). Riceve il buffer compresso (e la sua dimensione in `file_size`), e
dovrebbe restituire un nuovo buffer allocato con i dati non compressi (e impostare la dimensione del nuovo buffer anche in
`file_size`). Deve liberare il vecchio buffer (attenzione, `file_size` è in byte, ma free() si aspetta una dimensione in pagine).
In caso di errore, `file_size` non deve essere modificato e deve restituire il buffer originale non modificato.

```c
void _start(void);
```
Punto di ingresso per i plugin di tag (`PLG_T_TAG`). Potrebbero aggiungere nuovi tag in `tags_ptr` e adattare quel puntatore a una
nuova posizione allineata di 8 byte.

### Funzioni locali

I plugin possono usare funzioni locali, tuttavia a causa di un bug di CLang, queste *devono* essere dichiarate come `static`. (Il
bug è che CLang genera record PLT per quelle funzioni, anche se il flag "-fno-plt" viene passato sulla riga di comando. L'uso di
`static` aggira questo problema).

Specifica del formato file di basso livello
-------------------------------------------

Nel caso in cui qualcuno volesse scrivere un plugin in un linguaggio diverso dal C (ad esempio in Assembly), ecco la descrizione di
basso livello del formato del file.

È molto simile al formato a.out. Il file è composto da un'intestazione di dimensione fissa, seguita da sezioni di lunghezza variabile.
Non c'è un'intestazione di sezione, i dati di ogni sezione seguono direttamente i dati della sezione precedente nel seguente ordine:

```
(intestazione)
(record di corrispondenza dell'identificatore)
(record di ricollocazione)
(codice macchina)
(dati di sola lettura)
(dati inizializzati leggibili-scrivibili)
```

Per la prima sezione reale, codice macchina, l'allineamento è incluso. Per tutte le altre sezioni, il padding è aggiunto alla
dimensione della sezione precedente.

SUGGERIMENTO: se si passa un plugin come singolo argomento a `plgld`, questo scarica le sezioni nel file con un output simile a
`readelf -a` o `objdump -xd`.

### Intestazione

Tutti i numeri sono in formato little-endian, indipendentemente dall'architettura.

| Offset  | Dim.  | Descrizione                                                    |
|--------:|------:|----------------------------------------------------------------|
|       0 |     4 | byte magici `EPLG`                                             |
|       4 |     4 | dimensione totale del file                                     |
|       8 |     4 | memoria totale richiesta quando il file viene caricato         |
|      12 |     4 | dimensione della sezione di codice                             |
|      16 |     4 | dimensione della sezione dati di sola lettura                  |
|      20 |     4 | punto di ingresso del plugin                                   |
|      24 |     2 | codice di architettura (uguale a quello di ELF)                |
|      26 |     2 | numero di registrazioni di ricollocazione                      |
|      28 |     1 | numero di record di corrispondenza dell'identificatore         |
|      29 |     1 | voce GOT più referenziata                                      |
|      30 |     1 | revisione del formato del file (0 per ora)                     |
|      31 |     1 | tipo di plugin (1=file system, 2=kernel, 3=decompressore, 4=tag) |

Il codice dell'architettura è lo stesso delle intestazioni ELF, ad esempio 62 = x86_64, 183 = Aarch64 e 243 = RISC-V.

Il tipo di plugin specifica il prototipo del punto di ingresso, l'ABI è sempre SysV.

### Sezione di corrispondenza dell'identificatore

Questa sezione contiene tanti record quanti sono quelli specificati nel campo "numero di record corrispondenti all'identificatore"
dell'intestazione.

| Offset  | Dim.  | Descrizione                                              |
|--------:|------:|----------------------------------------------------------|
|       0 |     2 | offset                                                   |
|       2 |     1 | dimension                                                |
|       3 |     1 | tipo                                                     |
|       4 |     4 | byte magici da abbinare                                  |

Innanzitutto, l'inizio del soggetto viene caricato in un buffer. Viene impostato un accumulatore, inizialmente con uno 0. Gli offset
in questi record sono sempre relativi a questo accumulatore e indirizzano quel byte nel buffer.

Il campo Tipo indica come interpretare l'offset. Se è 1, allora l'offset più l'accumulatore vengono usati come valore. Se è 2,
allora un valore byte a 8 bit viene preso in offset, 3 significa prendere un valore word a 16 bit e 4 significa prendere un valore
dword a 32 bit. 5 significa prendere un valore byte a 8 bit e aggiungervi l'accumulatore, 6 significa prendere un valore word a 16
bit e aggiungervi l'accumulatore e 7 è lo stesso ma con valore a 32 bit. 8 cercherà i byte magici dall'accumulatore alla fine del
buffer in step di offset e, se trovati, restituirà l'offset corrispondente come valore.

Se size è zero, allora l'accumulatore è impostato sul valore. Se size non è zero, allora quel numero di byte viene controllato se
corrispondono ai magic byte specificati.

Ad esempio, per verificare se un eseguibile PE inizia con un'istruzione NOP:
```c
EASY_PLUGIN(PLG_T_KERNEL) {
   /* offset   dim  tipo           byte magici */
    { 0,         2, PLG_M_CONST, { 'M', 'Z', 0, 0 } },      /* controlla i byte magici */
    { 60,        0, PLG_M_DWORD, { 0, 0, 0, 0 } },          /* ottenere l'offset dell'intestazione PE sull'accumulatore */
    { 0,         4, PLG_M_CONST, { 'P', 'E', 0, 0 } },      /* controlla i byte magici */
    { 40,        1, PLG_M_DWORD, { 0x90, 0, 0, 0 } }        /* verifica l'istruzione NOP al punto di ingresso */
};
```

### Sezione di ricollocazione

Questa sezione contiene tanti record quanti sono quelli specificati nel campo "numero di registrazioni di ricollocazione"
dell'intestazione.


| Offset  | Dim.  | Descrizione                                              |
|--------:|------:|----------------------------------------------------------|
|       0 |     4 | offset                                                   |
|       4 |     4 | tipo di ricollocazione                                   |

Significato dei bit nel tipo:

| Da      | A     | Descrizione                                              |
|--------:|------:|----------------------------------------------------------|
|       0 |     7 | simbolo (0 - 255)                                        |
|       8 |     8 | Indirizzamento relativo del PC                           |
|       9 |     9 | Indirizzamento indiretto relativo GOT                    |
|      10 |    13 | indice maschera immediata (0 - 15)                       |
|      14 |    19 | inizio bit (0 - 63)                                      |
|      20 |    25 | fine bit (0 - 63)                                        |
|      26 |    31 | posizione del bit del flag di indirizzo negato (0 - 63)  |

Il campo offset è relativo alla magia nell'intestazione del plugin e seleziona un numero intero nella memoria in cui deve essere
eseguita la ricollocazione.

Il simbolo indica quale indirizzo usare. 0 indica l'indirizzo BASE in cui il plugin è stato caricato in memoria, ovvero l'indirizzo
della magia dell'intestazione in memoria. Altri valori selezionano un indirizzo di simbolo esterno dal GOT, definito nel loader o in
un altro plugin, dai un'occhiata all'array `plg_got` nella sorgente di plgld.c per vedere quale valore corrisponde a quale simbolo.
Se il bit relativo del GOT è 1, allora viene usato l'indirizzo della voce GOT del simbolo, invece dell'indirizzo effettivo del
simbolo.

Se il bit relativo del PC è 1, l'offset viene prima sottratto dall'indirizzo (modalità di indirizzamento relativo del puntatore di
istruzione).

L'indice della maschera immediata indica quali bit memorizzano l'indirizzo nell'istruzione. Se è 0, l'indirizzo viene scritto così
com'è nell'offset, indipendentemente dall'architettura. Per x86_64, è consentito solo l'indice 0. Per ARM Aarch64: 0 = così com'è,
1 = 0x07ffffe0 (spostamento a sinistra di 5 bit), 2 = 0x07fffc00 (spostamento a sinistra di 10 bit), 3 = 0x60ffffe0 (con istruzioni
ADR/ADRP l'immediato viene spostato e suddiviso in due gruppi di bit). Le architetture future potrebbero definire più e diverse
maschere di bit immediate.

Utilizzando la maschera immediata, i bit end - start + 1 vengono presi dalla memoria e con segno esteso. Questo valore viene
aggiunto all'indirizzo (addendo e, in caso di riferimenti interni, anche l'indirizzo del simbolo interno viene codificato qui).

Se il bit del flag di indirizzo negato non è 0 e l'indirizzo è positivo, allora quel bit viene cancellato. Se l'indirizzo è
negativo, allora quel bit viene impostato e l'indirizzo viene negato.

Infine, i bit di inizio e fine selezionano quale porzione dell'indirizzo scrivere sull'intero selezionato. Questo definisce anche
la dimensione della rilocazione, i bit al di fuori di questo intervallo e quelli che non fanno parte della maschera immediata
rimangono invariati.

### Sezione del codice

Questa sezione contiene istruzioni macchina per l'architettura specificata nell'intestazione e ha tanti byte quanti sono indicati
nel campo dimensione codice.

### Sezione dati di sola lettura

Questa è una sezione facoltativa, potrebbe mancare. È lunga quanto dice il campo della dimensione della sezione di sola lettura
nell'intestazione. Tutte le variabili costanti sono inserite in questa sezione.

### Sezione dati inizializzati

Questa è una sezione facoltativa, potrebbe mancare. Se ci sono ancora byte nel file dopo la sezione codice (o la sezione dati
facoltativa di sola lettura), allora quei byte sono tutti considerati la sezione dati. Se una variabile è inizializzata con un
valore diverso da zero, allora viene inserita in questa sezione.

### Sezione BSS

Questa è una sezione facoltativa, potrebbe mancare. Questa sezione non viene mai memorizzata nel file. Se il campo dimensione in
memoria è più grande del campo dimensione file nell'intestazione, la differenza verrà riempita con zeri in memoria. Se una variabile
non è inizializzata, o è inizializzata come zero, viene inserita in questa sezione.
