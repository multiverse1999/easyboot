カーネルのデバッグ
==================

カーネルが大混乱に陥った場合、デフォルトでは **Easyboot** が、何がどこで問題になったかについての詳細を表示してくれます。

![](../exc.png)

これは手がかりを得るには十分ですが、対話的に調査することはできません。そのためにはデバッガーが必要になります。

Mini Debugger
-------------

利点:

- セットアップが簡単
- 使いやすい
- 実際のハードウェアと仮想マシンでも動作します

デメリット:

- 名前の通り、最小限の機能セット

[Mini Debugger](https://gitlab.com/bztsrc/minidbg) を有効にするには、適切な `minidbg_(arch).plg` ファイルをブート パーティションにコピーして、
**Easyboot** [plugin](plugins.md) をインストールするだけです。これで完了です。シリアル ライン (115200 ボー、8 データ ビット、1 ストップ ビット、
パリティなし) を介したビデオ ターミナル インターフェイスを提供します。

実際のハードウェアでは、VT100 または VT220 端末、またはシリアル ケーブルを介して **PuTTY** (Windows) または **minicom** (Linux) を実行している別のマシンを接続します。

仮想マシンの場合は、`-serial stdio` 引数を指定して qemu を実行するだけで、qemu の実行に使用したのと同じウィンドウからデバッガーを制御できるようになります。例:

```
qemu-system-x86_64 -serial stdio -hda disk.img
```

カーネルがクラッシュすると、すぐにデバッガー プロンプトが表示され、状況を調べることができます。デバッガー プロンプトで `?` または `h` と入力するとヘルプが表示されます。
デバッガーを明示的に呼び出すには、コードに `int 3` (0xCC バイト) を挿入します。カーネルに次の定義を追加することをお勧めします。

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

利点:

- セットアップ不要、すぐに使用可能
- 他のデバッガーではできない、マシンの内部状態を表示

デメリット:

- 本当に使いにくい
- 仮想マシンでしか動作しない

仮想マシンの実行中に、メニューから `View` > `compatmonitor0`を選択するか、ウィンドウをクリックしてフォーカスを取得し、
<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>2</kbd> を押します (フォーカスを解除するには、<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>G</kbd> を押します)。

GDB
---

利点:

- フル機能のデバッガー
- 考えられるすべての機能を提供します

デメリット:

- セットアップが難しい
- 仮想マシンでのみ動作

何かする前に、まずビルド環境を変更して 2 つのカーネル ファイルを生成します。1 つはデバッグ シンボル付き、もう 1 つはデバッグ シンボルなしです。
デバッグ シンボルは簡単に大量のスペース (おそらく数メガバイト) を占めるため、これは重要です。まず、`-g` フラグを使用してカーネルをコンパイルします。
次に、コンパイルが完了したら、カーネルを `cp mykernel.elf mykernel_sym.elf` でコピーし、`strip mykernel.elf` でデバッグ情報を削除します。
その後、 `mykernel.elf` を起動し、 `mykernel_sym.elf` を gdb に提供します。

次に、`gdb.rc` という名前の gdb スクリプトを作成します。以下をテンプレートとして使用します:

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

これにより、gdb が仮想マシンに接続され、マシンのタイプが伝えられ、デバッグ情報が読み込まれ、レイアウトが構成され、カーネルのエントリ ポイントにブレークポイントが設定され、
最後に仮想マシンが起動します。

このセットアップが完了したら、デバッグを開始できます。デバッグ セッションは次のようになります。1 つのターミナルで、`-s -S` フラグを使用して qemu を起動します。ハングします。例:

```
qemu-system-x86_64 -s -S -hda disk.img
```

次に、別のターミナルで、前に作成したスクリプトを使用して gdb を起動します:

```
gdb -w -x gdb.rc
```

次のようにすべてが 1 つのウィンドウに表示されます:

![](../gdb.png)

ここまで来たら、おめでとうございます。カーネルで実際の作業を開始できます。
