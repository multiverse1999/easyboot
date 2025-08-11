Avvio dei kernel con Easyboot
=============================

[Easyboot](https://gitlab.com/bztsrc/easyboot) è un gestore di avvio e creatore di immagini di dischi avviabili tutto in uno, in
grado di caricare vari kernel del sistema operativo e kernel compatibili con Multiboot2 in vari formati.

[[_TOC_]]

Installazione
-------------

```
 easyboot [-v|-vv] [-s <mb>] [-b <mb>] [-u <guid>] [-p <t> <u> <i>] [-e] [-c] <indir> <outfile|device>

  -v, -vv         aumentare la verbosità/convalida
  -s <mb>         imposta la dimensione dell'immagine del disco in Megabyte (predefinito su 35M)
  -b <mb>         imposta la dimensione della partizione di avvio in Megabyte (il valore predefinito è 33 MB)
  -u <guid>       imposta l'identificatore univoco della partizione di avvio (il valore predefinito è casuale)
  -p <t> <u> <i>  aggiungere una partizione aggiuntiva (tipo guid, univoco guid, file immagine)
  -e              aggiunto El Torito Boot Catalog (supporto di avvio CDROM EFI)
  -c              crea sempre un nuovo file immagine anche se esiste
  <indir>         utilizzare il contenuto di questa directory per la partizione di avvio
  <outfile>       immagine di output o nome del file del dispositivo
```

Lo strumento **Easyboot** crea un'immagine disco avviabile denominata `(outfile)` usando la GUID Partitioning Table con una singola
partizione formattata come FAT32 e denominata "EFI System Partition" (ESP in breve). Il contenuto di tale partizione è preso da
`(indir)` fornito. È necessario posizionare un semplice file di configurazione in testo normale in tale directory denominata
`easyboot/menu.cfg`. Con terminazioni di riga NL o CRLF, è possibile modificarlo facilmente con qualsiasi editor di testo. A
seconda della configurazione, potrebbero essere necessari anche alcuni [plugin](plugins.md) in questa directory denominata
`easyboot/*.plg`.

L'immagine può essere avviata anche su Raspberry Pi e funzionerà in qemu; ma per l'avvio su una macchina reale, avrai bisogno di
ulteriori file firmware `bootcode.bin`, `fixup.dat`, `start.elf` e un file .dtb nella directory `(indir)`, che possono essere
scaricati dal [repository ufficiale](https://github.com/raspberrypi/firmware/tree/master/boot) del Raspberry Pi.

Lo strumento ha anche alcuni flag opzionali della riga di comando: `-s (mb)` imposta la dimensione complessiva dell'immagine disco
generata in Megabyte, mentre `-b (mb)` imposta la dimensione della partizione di avvio in Megabyte. Ovviamente la prima deve essere
più grande della seconda. Se non specificato, la dimensione della partizione viene calcolata in base alla dimensione della directory
specificata (almeno 33 Mb, il più piccolo FAT32 possibile) e la dimensione del disco è di default 2 Mb più grande di quella (a
causa dell'allineamento e dello spazio necessario per la tabella di partizionamento). Se c'è uno scarto di più di 2 Mb tra questi
due valori di dimensione, puoi usare strumenti di terze parti come `fdisk` per aggiungere più partizioni all'immagine a tuo piacimento
(o vedi `-p` sotto). Se vuoi un layout prevedibile, puoi anche specificare l'identificativo univoco della partizione di avvio
(UniquePartitionGUID) con il flag `-u <guid>`.

Facoltativamente puoi anche aggiungere partizioni extra con il flag `-p`. Questo richiede 3 argomenti: (PartitionTypeGUID),
(UniquePartitionGUID) e il nome del file immagine che contiene il contenuto della partizione. Questo flag può essere ripetuto più
volte.

Il flag `-e` aggiunge El Torito Boot Catalog all'immagine generata, in modo che possa essere avviata non solo come chiavetta USB ma
anche come CD-ROM BIOS/EFI.

Se `(outfile)` è un file di dispositivo (ad esempio `/dev/sda` su Linux, `/dev/disk0` su BSD e `\\.\PhysicalDrive0` su Windows),
allora non crea GPT né ESP, ma individua quelli già esistenti sul dispositivo. Copia comunque tutti i file in `(indir)` nella
partizione di avvio e installa i loader. Funziona anche se `(outfile)` è un file immagine già esistente (in questo caso usa il
flag `-c` per creare sempre prima un nuovo file immagine vuoto).

Configurazione
--------------

Il file `easyboot/menu.cfg` può contenere le seguenti righe (molto simili alla sintassi di grub.cfg, puoi trovare un file di
configurazione di esempio [qui](menu.cfg)):

### Commenti

Tutte le righe che iniziano con `#` sono considerate commenti e vengono saltate fino alla fine della riga.

### Livello di verbosità

È possibile impostare il livello di verbosità utilizzando una riga che inizia con `verbose`.

```
verbose (0-3)
```

Questo indica al loader quante informazioni stampare sulla console di avvio. Verbose 0 significa totalmente silenzioso (default) e
verbose 3 scaricherà i segmenti del kernel caricati e il codice macchina al punto di ingresso.

### Framebuffer

Puoi richiedere una risoluzione dello schermo specifica con la riga che inizia con `framebuffer`. Il formato è il seguente:

```
framebuffer (larghezza) (altezza) (bit per pixel) [#(colore primo)] [#(colore sfondo)] [#(avanzamento)]
```

**Easyboot** imposterà un framebuffer per te, anche se questa linea non esiste (800 x 600 x 32 bpp di default). Ma se questa linea
esiste, allora proverà a impostare la risoluzione specificata. Le modalità palette non sono supportate, quindi i bit per pixel
devono essere almeno 15.

Se viene specificato il quarto parametro colore facoltativo, deve essere in notazione HTML che inizia con un cancelletto seguito
da 6 cifre esadecimali, RRVVBB. Ad esempio `#ff0000` è rosso vivo e `#007f7f` è un ciano più scuro. Imposta il colore di primo
piano o, in altre parole, il colore del font. Allo stesso modo, se viene specificato il quinto parametro colore facoltativo, deve
essere anch'esso in notazione HTML e imposta il colore di sfondo. L'ultimo parametro colore facoltativo è simile e imposta il
colore di sfondo della barra di avanzamento.

Se non vengono specificati i colori, per impostazione predefinita vengono visualizzate lettere bianche su sfondo nero e lo sfondo
della barra di avanzamento è grigio scuro.

### Opzione di avvio predefinita

La riga che inizia con `default` imposta la voce di menu che deve essere avviata senza interazione dell'utente dopo il timeout
specificato.

```
default (indice voce di menu) (timeout msec)
```

L'indice di menuentry è un numero compreso tra 1 e 9. Il timeout è espresso in millisecondi, quindi 1000 equivale a un secondo.


### Allineamento del menu

Le righe che iniziano con `menualign` modificano il modo in cui vengono visualizzate le opzioni di avvio.

```
menualign ("vertical"|"horizontal")
```

Di default il menu è orizzontale, ma è possibile modificarlo in verticale.

### Opzioni di avvio

Puoi specificare fino a 9 voci di menu con righe che iniziano con `menuentry`. Il formato è il seguente:

```
menuentry (icona) [etichetta]
```

Per ogni icona, deve esistere un file `easyboot/(icona).tga` sulla partizione di avvio. L'immagine deve essere in un formato
[Targa](../en/TGA.txt) codificato in run-length e mappato a colori, perché è la variante più compressa (i primi tre byte del
file devono essere `0`, `1` e `9` in questo ordine, vedere Tipo di dati 9 nella specifica). Le sue dimensioni devono essere
esattamente 64 pixel di altezza e 64 pixel di larghezza.

Per salvare in questo formato da GIMP, seleziona prima "Image > Mode > Indexed...", nella finestra pop-up imposta "Maximum number
of colors" a 256. Quindi seleziona "File > Export As...", inserisci un nome file che termini con `.tga` e nella finestra pop-up
seleziona "RLE compression". Per uno strumento di conversione da riga di comando, puoi usare ImageMagick,
`convert (qualsiasi file immagine) -colors 256 -compress RLE icona.tga`.

Il parametro label facoltativo serve per visualizzare le informazioni sulla versione ASCII o sulla release nel menu, e non per
stringhe arbitrarie, quindi per risparmiare spazio, UTF-8 non è supportato a meno che non si fornisca anche `easyboot/font.sfn`.
(Un font UNICODE richiede molto spazio di archiviazione, anche se [Scalable Screen Font](https://gitlab.com/bztsrc/scalable-font2)
è molto efficiente, è comunque circa 820K. Al contrario, unicode.pf2 di GRUB è molto più grande, circa 2392K, anche se entrambi i
font contengono circa 55600 glifi in bitmap 8x16 e 16x16. Il font ASCII incorporato ha solo bitmap 8x16 e 96 glifi.)

Tutte le righe che seguono una riga `menuentry` apparterranno a quella menuentry, eccetto quando quella riga è un'altra riga
menuentry. Per comodità, puoi usare blocchi come in GRUB, ma sono solo zucchero sintattico. Le parentesi graffe sono gestite come
caratteri di spazio vuoto. Puoi ometterle e usare invece tabulazioni come in uno script Python o Makefile se preferisci.

Per esempio
```
menuentry FreeBSD backup
{
    kernel bsd.old/boot
}
```

### Seleziona partizione

La riga che inizia con `partition` seleziona una partizione GPT. Deve essere preceduta da una riga `menuentry`.

```
partition (partizione unica UUID)
```

Questa partizione verrà utilizzata come file system root per l'opzione di avvio. Il kernel e i moduli verranno caricati da questa
partizione e, a seconda del protocollo di avvio, verranno anche passati al kernel. La partizione specificata potrebbe risiedere su
un disco diverso dal disco di avvio, **Easyboot** eseguirà un'iterazione su tutti i dischi partizionati GPT durante l'avvio per
individuarla.

Per comodità, la partizione viene cercata anche sulla riga `kernel` (vedi sotto). Se la riga di comando di avvio specificata
contiene una stringa `root=(UUID)` o `root=*=(UUID)`, allora non c'è bisogno di una riga `partition` separata.

Se la partizione non è specificata esplicitamente, sia il kernel che i moduli vengono caricati dalla partizione di avvio.

### Specificare un kernel

La riga che inizia con `kernel` indica quale file deve essere avviato e con quali parametri. Deve essere preceduta da una riga
`menuentry`.

```
kernel (percorso al file del kernel) (argomenti facoltativi della riga di comando di avvio)
```

Il percorso deve puntare a un file esistente, un binario eseguibile del kernel, e deve essere un percorso UTF-8 assoluto sulla
partizione root (o boot). Se il kernel non si trova nella directory root della partizione, il separatore di directory è sempre `/`,
anche sui sistemi UEFI. Se il nome contiene uno spazio, allora deve essere preceduto da `\`. Il percorso potrebbe essere seguito da
argomenti della riga di comando, separati da uno spazio. Per i kernel compatibili con Multiboot2, questi argomenti della riga di
comando saranno passati nel tag *Riga di comando di avvio* (tipo 1). Non saranno modificati da **Easyboot**, né analizzati, tranne
che per la ricerca dello specificatore di partizione.

Di default **Easyboot** può avviare kernel compatibili con Multiboot2 nei formati ELF64 e PE32+/COFF (e su sistemi UEFI, anche
applicazioni UEFI). Normalmente quel protocollo non consente kernel di metà superiore, ma **Easyboot** viola un po' il protocollo
in un modo che non interrompe i kernel normali, non di metà superiore. Se vuoi avviare qualsiasi altro kernel, allora avrai bisogno
di un kernel loader [plugin](plugins.md).

NOTA: a differenza di GRUB, dove è necessario utilizzare comandi speciali come "linux" o "multiboot" per selezionare il protocollo
di avvio, qui è sufficiente un solo comando e il protocollo viene rilevato automaticamente dal kernel in fase di esecuzione.

### Caricamento di ulteriori moduli

È possibile caricare file arbitrari (ramdisk iniziali, driver del kernel, ecc.) insieme al kernel utilizzando le righe che iniziano
con `module`. Deve essere preceduto da una riga `menuentry`. Nota che questa riga può essere ripetuta più volte all'interno di ogni
menuentry. Se il protocollo di avvio supporta un initrd, allora la prima riga `module` è considerata come initrd.

```
module (percorso verso un file) (argomenti opzionali della riga di comando del modulo)
```

Il percorso deve puntare a un file esistente e deve essere un percorso UTF-8 assoluto sulla partizione root (o boot). Potrebbe
essere seguito da argomenti della riga di comando, separati da uno spazio. Se il file è compresso ed esiste un [plugin](plugins.md)
di decompressione per esso, allora il modulo verrà decompresso in modo trasparente. Le informazioni su questi moduli caricati
(e decompressi) verranno passate a un kernel compatibile con Multiboot2 nei tag *Moduli* (tipo 3).

Il caso speciale è se il modulo inizia con i byte `DSDT`, `GUDT` o `0xD00DFEED`. In questi casi il file non verrà aggiunto ai
tag *Moduli*, ma la tabella ACPI verrà patchata in modo che i suoi puntatori DSDT puntino al contenuto di questo file. Con questo
puoi facilmente sostituire una tabella ACPI del BIOS difettosa con una fornita dall'utente.

### Logo dello schizzo dello stivale

Puoi anche visualizzare un logo al centro dello schermo quando è selezionata l'opzione di avvio se inserisci una riga che inizia
con `bootsplash`. Deve essere preceduta da una riga `menuentry`.

```
bootsplash [#(colore sfondo)] (percorso verso un file tga)
```

Il colore di sfondo è facoltativo e deve essere in notazione HTML che inizia con un cancelletto seguito da 6 cifre esadecimali,
RRVVBB. Se il primo argomento non inizia con `#`, allora si presume un argomento di percorso.

Il percorso deve puntare a un file esistente e deve essere un percorso UTF-8 assoluto sulla partizione di avvio (NON root).
L'immagine deve essere in un formato Targa codificato in run-length e mappato a colori, proprio come le icone del menu. Le
dimensioni possono essere qualsiasi cosa che si adatti allo schermo.

### Supporto multi-core

Per avviare il kernel su tutti i core del processore contemporaneamente, specificare la direttiva `multicore` (solo kernel a 64
bit). Deve essere preceduta da una riga `menuentry`.

```
multicore
```

Risoluzione dei problemi
------------------------

Se riscontri problemi, esegui semplicemente con il flag `easyboot -vv`. Questo eseguirà la convalida e produrrà i risultati in modo
dettagliato al momento della creazione dell'immagine. Altrimenti aggiungi `verbose 3` a `easyboot/menu.cfg` per ottenere messaggi
dettagliati al momento dell'avvio.

Se vedi la stringa `PMBR-ERR` nell'angolo in alto a sinistra con sfondo rosso, significa che la tua CPU è molto vecchia e non
supporta la modalità lunga a 64 bit o che il settore di avvio non è riuscito ad avviare il loader. Potrebbe verificarsi solo su
macchine BIOS, questo non può mai accadere con UEFI o su RaspberryPi.

| Messaggio                           | Descrizione                                                                       |
|-------------------------------------|-----------------------------------------------------------------------------------|
| `Booting [X]...`                    | indica che è stata scelta l'opzione di avvio (indice della voce di menu) X        |
| `Loading 'X' (Y bytes)...`          | il file X di lunghezza Y è in fase di caricamento                                 |
| `Parsing kernel...`                 | il kernel è stato trovato, ora ne viene rilevato il formato                       |
| `Starting X boot...`                | mostra che è stato rilevato il boot loader del sistema X                          |
| `Starting X kernel...`              | mostra che è stato rilevato il kernel del sistema X                               |
| `Transfering X control to Y`        | indica che il punto di ingresso della modalità X all'indirizzo Y sta per essere chiamato |

Se riscontri problemi dopo aver visto l'ultimo messaggio, significa che il problema si è verificato nella procedura di avvio del
sistema operativo e non nel loader **Easyboot**, quindi dovrai consultare la documentazione del sistema operativo in questione per
una risposta. Altrimenti sentiti libero di aprire un [problema](https://gitlab.com/bztsrc/easyboot/-/issues) su gitlab.

### Multiboot1

Plugin richiesti: [grubmb1](../../src/plugins/grubmb1.c)

### Multiboot2

Questa è la più flessibile, con molteplici varianti supportate tramite plugin:

- ELF64 o PE32+ con Multiboot2 semplificato: nessun plugin richiesto
- ELF32 con Multiboot2 semplificato e punto di ingresso a 32 bit: [elf32](../../src/plugins/elf32.c)
- a.out (struct exec) con Multiboot2 semplificato e punto di ingresso a 32 bit: [aout](../../src/plugins/aout.c)
- GRUB compatibile Multiboot2 con punto di ingresso a 32 bit: [grubmb2](../../src/plugins/grubmb2.c)

Nota la differenza: [Multiboot2 semplificato](ABI.md) non richiede tag incorporati, supporta kernel della metà superiore, punto di
ingresso pulito a 64 bit con parametri passati secondo Multiboot2, SysV e ABI fastcall.

D'altro canto [GRUB compatibile Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) necessita di tag
incorporati, non supporta kernel a metà superiore né a 64 bit, il punto di ingresso è sempre a 32 bit e i parametri sono passati
solo in `eax`, `ebx`.

### Windows

Non sono richiesti plugin, ma è necessario configurare [Secure Boot](secureboot.md).

```
menuentry win {
  kernel EFI/Microsoft/BOOT/BOOTMGRW.EFI
}
```

### Linux

Plugin richiesti: [linux](../../src/plugins/linux.c), [ext234](../../src/plugins/ext234.c)

Se il kernel non è posizionato sulla partizione di avvio, è possibile utilizzare la riga di comando per specificare la partizione
root.

```
menuentry linux {
  kernel vmlinuz-linux root=PARTUUID=01020304-0506-0708-0a0b0c0d0e0f1011
}
```

### OpenBSD

Plugin richiesti: [obsdboot](../../src/plugins/obsdboot.c), [ufs44](../../src/plugins/ufs44.c)

```
menuentry openbsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot
}
```

ATTENZIONE! Non usare il plugin [elf32](../../src/plugins/elf32.c) se stai avviando OpenBSD! Il suo `boot` dichiara erroneamente di
essere un ELF con punto di ingresso SysV ABI a 32 bit, ma in realtà ha un ingresso in modalità reale a 16 bit.

### FreeBSD

Plugin richiesti: [fbsdboot](../../src/plugins/fbsdboot.c), [ufs2](../../src/plugins/ufs2.c)

Nei sistemi BIOS legacy, specificare il caricatore `boot`.

```
menuentry freebsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot/boot
}
```

Sui computer UEFI, utilizzare `loader.efi` sulla partizione di avvio (non sono necessari plugin).

```
menuentry freebsd {
  kernel boot/loader.efi
}
```

Se il file system root è ZFS, copia questo file (`boot` sul BIOS, `loader.efi` su UEFI) in `(indir)` e NON specificare una
partizione root.

### FreeDOS

Plugin richiesti: [fdos](../../src/plugins/fdos.c)

Sposta i file di FreeDOS in `(indir)` (FreeDOS utilizzerà la partizione di avvio come partizione di root).

```
menuentry freedos {
  kernel KERNEL.SYS
}
```

Se l'avvio si interrompe dopo aver stampato copyright e `- InitDisk`, il kernel FreeDOS è stato compilato senza supporto FAT32.
Scarica un kernel diverso, con `f32` nel nome.

### ReactOS

Plugin richiesti: [reactos](../../src/plugins/reactos.c)

```
menuentry reactos {
  kernel FREELDR.SYS
}
```

### MenuetOS

Plugin richiesti: [menuet](../../src/plugins/menuet.c)

```
menuentry menuetos {
  kernel KERNEL.MNT
  module CONFIG.MNT
  module RAMDISK.MNT
}
```

Per disattivare la configurazione automatica, aggiungere `noauto` alla riga di comando.

### KolibriOS

Plugin richiesti: [kolibri](../../src/plugins/kolibri.c)

```
menuentry kolibrios {
  kernel KERNEL.MNT syspath=/rd/1/ launcher_start=1
  module KOLIBRI.IMG
  module DEVICES.DAT
}
```

Il plugin funziona anche su macchine UEFI, ma è anche possibile utilizzare `uefi4kos.efi` sulla partizione di avvio (senza bisogno
di alcun plugin).

### SerenityOS

Plugin richiesti: [grubmb1](../../src/plugins/grubmb1.c)

```
menuentry serenityos {
  kernel boot/Prekernel
  module boot/Kernel
}
```

### Haiku

Plugin richiesti: [grubmb1](../../src/plugins/grubmb1.c), [befs](../../src/plugins/befs.c)

```
menuentry haiku {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel system/packages/haiku_loader-r1~beta4_hrev56578_59-1-x86_64.hpkg
}
```

Su macchine UEFI, utilizzare `haiku_loader.efi` sulla partizione di avvio (non sono necessari plugin).

### OS/Z

```
menuentry osz {
  kernel ibmpc/core
  module ibmpc/initrd
}
```

Non sono necessari plugin.
