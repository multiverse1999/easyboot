Uw kernel debuggen
==================

Als er iets misgaat met uw kernel, zal **Easyboot** u standaard een plezier doen door details weer te geven over wat er fout is
gegaan en waar.

![](../exc.png)

Dit is genoeg om je een hint te geven, maar het laat je niet toe om interactief te onderzoeken. Daarvoor heb je een debugger nodig.

Mini Debugger
-------------

Voordelen:

- eenvoudig in te stellen
- eenvoudig te gebruiken
- werkt op echte hardware en virtuele machines

Nadelen:

- zoals de naam al doet vermoeden, minimale functieset

Om de [Mini Debugger](https://gitlab.com/bztsrc/minidbg) in te schakelen, installeert u gewoon de **Easyboot** [plugin](plugins.md)
door het juiste `minidbg_(arch).plg` bestand naar uw boot partitie te kopiëren. Dat is alles. Het biedt een video terminal interface
over seriële lijn (met 115200 baud, 8 data bits, 1 stop bit, geen pariteit).

Sluit op echte hardware een VT100- of VT220-terminal aan, of sluit een andere machine aan via een seriële kabel met **PuTTY**
(Windows) of **minicom** (Linux).

Voor virtuele machines voert u gewoon qemu uit met de `-serial stdio`-argumenten, en u kunt de debugger bedienen vanuit hetzelfde
venster dat u hebt gebruikt om qemu uit te voeren. Bijvoorbeeld:

```
qemu-system-x86_64 -serial stdio -hda disk.img
```

Elke keer dat uw kernel uitvalt, krijgt u direct een debuggerprompt en kunt u de situatie onderzoeken. Typ in de debuggerprompt `?`
of `h` om hulp te krijgen. Om de debugger expliciet aan te roepen, voegt u `int 3` (een 0xCC byte) toe aan uw code. Ik stel voor
om de volgende define toe te voegen aan uw kernel:

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

Voordelen:

- geen installatie, werkt direct
- geeft de interne status van de machine weer, wat geen enkele andere debugger kan doen

Nadelen:

- echt moeilijk te gebruiken
- werkt alleen met virtuele machines

Wanneer uw virtuele machine actief is, selecteert u in het menu `View` > `compatmonitor0` of klikt u op het venster zodat de focus
wordt vastgelegd en drukt u op <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>2</kbd> (om de focus vrij te geven, drukt u op
<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>G</kbd>).

GDB
---

Voordelen:

- volledig uitgeruste debugger
- biedt alle functies die u maar kunt bedenken

Nadelen:

- lastig in te stellen
- werkt alleen met virtuele machines

Voordat u iets doet, moet u eerst uw build-omgeving aanpassen om twee kernelbestanden te genereren. Eén met debugsymbolen en
één zonder. Dit is belangrijk omdat debugsymbolen gemakkelijk veel ruimte in beslag kunnen nemen, waarschijnlijk meerdere
megabytes. Compileer eerst uw kernel met de `-g`-vlag. Kopieer vervolgens, nadat de compilatie is voltooid, uw kernel
`cp mykernel.elf mykernel_sym.elf` en verwijder de debuginfo met `strip mykernel.elf`. Hierna gaat u `mykernel.elf` opstarten
en `mykernel_sym.elf` aan gdb leveren.

Maak vervolgens een gdb-script met de naam `gdb.rc`. Gebruik het volgende als sjabloon:

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

Hiermee wordt gdb verbonden met de virtuele machine, wordt het type van de machine doorgegeven, wordt de foutopsporingsinformatie
geladen, wordt de lay-out geconfigureerd, wordt er een breekpunt ingesteld bij het toegangspunt van de kernel en wordt ten slotte
de virtuele machine gestart.

Nadat deze set-up is voltooid, kunt u beginnen met debuggen. Eén debugsessie verloopt als volgt: start in één terminal qemu met
de `-s -S`-vlaggen. Het zal hangen. Bijvoorbeeld:

```
qemu-system-x86_64 -s -S -hda disk.img
```

Start vervolgens in een andere terminal gdb met het script dat we eerder hebben gemaakt:

```
gdb -w -x gdb.rc
```

Dit zou alles in één venster moeten weergeven, zoals dit:

![](../gdb.png)

Als je het tot hier hebt gehaald, gefeliciteerd! Je kunt nu beginnen met het echte werk met je kernel!
