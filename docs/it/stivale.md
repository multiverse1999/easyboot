Supporto Stivale in Easyboot
============================

Non accadrà mai, mai e poi mai. Questo protocollo di avvio ha delle scelte di progettazione pessime e molto serie e rappresenta un
rischio enorme per la sicurezza.

Innanzitutto, i kernel stivale hanno un'intestazione ELF, ma in qualche modo dovresti sapere che l'intestazione non è valida; niente,
assolutamente niente implica nell'intestazione che non sia un kernel SysV ABI valido. In Multiboot ci devono essere alcuni byte
magici all'inizio del file in modo da poter rilevare il protocollo; non c'è niente del genere in stivale / stivale2. (L'ancora non
ti aiuta, perché potrebbe verificarsi *ovunque* nel file, quindi in realtà devi *cercare l'intero file* per essere sicuro che non
sia compatibile con stivale.)

In secondo luogo, utilizza sezioni; che secondo la specifica ELF (vedere [pagina 46](https://www.sco.com/developers/devspecs/gabi41.pdf))
sono opzionali e nessun loader dovrebbe preoccuparsene. I loader utilizzano la Execution View e non la Linking View. Implementare
l'analisi delle sezioni solo per questo protocollo è un sovraccarico folle in qualsiasi loader, dove le risorse di sistema sono
solitamente già scarse.

Terzo, le intestazioni di sezione si trovano alla fine del file. Ciò significa che per rilevare stivale, devi caricare l'inizio del
file, analizzare le intestazioni ELF, quindi caricare la fine del file e analizzare le intestazioni di sezione, quindi caricare da
qualche parte nel mezzo del file per ottenere l'elenco dei tag effettivo. Questa è la peggiore soluzione possibile. E ancora, non
c'è assolutamente nulla che indichi che un loader dovrebbe farlo, quindi devi farlo per tutti i kernel solo per scoprire che il
kernel non usa stivale. Ciò rallenta anche il rilevamento di *tutti gli altri* protocolli di avvio, il che è inaccettabile.

L'elenco dei tag viene interrogato attivamente dai processori applicativi e il kernel potrebbe chiamare il codice del boot loader
in qualsiasi momento, il che significa che non è possibile recuperare la memoria del boot loader, altrimenti è garantito un crash.
Ciò va contro la filosofia di **Easyboot**.

La parte peggiore è che il protocollo si aspetta che i boot loader iniettino codice in qualsiasi kernel compatibile con stivale,
che viene poi eseguito al livello di privilegio più alto possibile. Già, cosa potrebbe andare storto, giusto?

Poiché mi rifiuto di distribuire codice di scarsa qualità dalle mie mani, non ci sarà alcun supporto stivale in **Easyboot**. E se
seguite il mio consiglio, nessun programmatore di sistemi operativi amatoriale dovrebbe mai usarlo per il proprio bene.
