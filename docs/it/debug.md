Debug del kernel
================

Se il tuo kernel non funziona correttamente, **Easyboot** ti aiuterà di default a visualizzare i dettagli su cosa e dove è andato
storto.

![](../exc.png)

Questo è sufficiente per darti un indizio, ma non ti consente di investigare in modo interattivo. Per quello, avresti bisogno di
un debugger.

Mini Debugger
-------------

Vantaggi:

- facile da configurare
- facile da usare
- funziona sia su hardware reale che su macchine virtuali

Svantaggi:

- come suggerisce il nome, set di funzionalità minimo

Per abilitare il [Mini Debugger](https://gitlab.com/bztsrc/minidbg), basta installare il [plugin](plugins.md) **Easyboot** copiando
il file `minidbg_(arch).plg` appropriato nella partizione di avvio. Tutto qui. Fornisce un'interfaccia video terminale su linea
seriale (con 115200 baud, 8 bit di dati, 1 bit di stop, nessuna parità).

Su hardware reale, collegare un terminale VT100 o VT220, o un altro computer tramite cavo seriale che esegua **PuTTY** (Windows)
o **minicom** (Linux).

Per le macchine virtuali, esegui semplicemente qemu con gli argomenti `-serial stdio` e sarai in grado di controllare il debugger
dalla stessa finestra che hai usato per eseguire qemu. Ad esempio:

```
qemu-system-x86_64 -serial stdio -hda disk.img
```

Ogni volta che il tuo kernel salta, riceverai immediatamente un prompt del debugger e sarai in grado di esaminare la situazione.
Nel prompt del debugger, digita `?` o `h` per ottenere aiuto. Per richiamare esplicitamente il debugger, inserisci `int 3` (un
byte 0xCC) nel tuo codice. Suggerisco di aggiungere la seguente definizione al tuo kernel:

```c
/* x86 */
#define breakpoint __asm__ __volatile__("int $3")
/* ARM */
#define breakpoint __asm__ __volatile__("brk #0")
```

```
Mini debugger by bzt
Exception 03: Breakpoint instruction, code 0
rax: 0000000000000000  rbx: 00000000000206C0  rcx: 000000000000270F
rdx: 00000000000003F8  rsi: 00000000000001B0  rdi: 0000000000102336
rsp: 000000000008FFB8  rbp: 000000000008FFF8   r8: 0000000000000004
 r9: 0000000000000002  r10: 0000000000000000  r11: 0000000000000003
r12: 0000000000000000  r13: 0000000000000000  r14: 0000000000000000
r15: 0000000000000000
> ?
Mini debugger commands:
  ?/h		this help
  c		continue execution
  n		move to the next instruction
  r		dump registers
  x [os [oe]]	examine memory from offset start (os) to offset end (oe)
  i [os [oe]]	disassemble instructions from offset start to offset end
> i pc-1 pc+4
00101601: CC                             int	3
00101602: FA                             cli
00101603: F4                             hlt
00101604: EB FC                          jmp	101602h
00101606: 90                              1 x nop
>
```

Qemu Debugger
-------------

Vantaggi:

- nessuna configurazione, funziona subito
- mostra lo stato interno della macchina, cosa che nessun altro debugger può fare

Svantaggi:

- davvero difficile da usare
- funziona solo con macchine virtuali

Quando la macchina virtuale è in esecuzione, dal menu seleziona `View` > `compatmonitor0`, oppure fai clic sulla finestra in modo
che catturi il focus e premi <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>2</kbd> (per rilasciare il focus premi
<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>G</kbd>).

GDB
---

Vantaggi:

- debugger completo
- fornisce tutte le funzionalità che puoi immaginare

Svantaggi:

- difficile da configurare
- funziona solo con macchine virtuali

Prima di fare qualsiasi cosa, modifica il tuo ambiente di compilazione per generare due file kernel. Uno con i simboli di debug e
uno senza. Questo è importante perché i simboli di debug possono facilmente occupare molto spazio, probabilmente diversi megabyte.
Per prima cosa, compila il tuo kernel con il flag `-g`. Quindi, una volta completata la compilazione, copia il tuo kernel
`cp mykernel.elf mykernel_sym.elf` e rimuovi le informazioni di debug con `strip mykernel.elf`. In seguito, avvierai `mykernel.elf`
e fornirai `mykernel_sym.elf` a gdb.

Successivamente, crea uno script gdb denominato `gdb.rc`. Utilizza quanto segue come modello:

```
target remote localhost:1234
set architecture i386:x86-64
symbol-file mykernel_sym.elf
layout split
layout src
layout regs
break *_start
continue
```

Ciò collega gdb alla macchina virtuale, le comunica il tipo di macchina, carica le informazioni di debug, configura il layout,
imposta un punto di interruzione proprio nel punto di ingresso del kernel e infine avvia la macchina virtuale.

Una volta completata questa configurazione, puoi iniziare il debug. Una sessione di debug funziona così: in un terminale, avvia
qemu con i flag `-s -S`. Si bloccherà. Ad esempio:

```
qemu-system-x86_64 -s -S -hda disk.img
```

Quindi, in un altro terminale, avviamo gdb con lo script che abbiamo creato in precedenza:

```
gdb -w -x gdb.rc
```

Questo dovrebbe visualizzare tutto in una finestra come questa:

![](../gdb.png)

Se sei arrivato fin qui, congratulazioni, puoi iniziare il vero lavoro con il tuo kernel!
