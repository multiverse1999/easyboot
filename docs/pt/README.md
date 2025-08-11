Inicialização de kernels com Easyboot
=====================================

[Easyboot](https://gitlab.com/bztsrc/easyboot) é um gestor de arranque completo e criador de imagens de disco de arranque que pode
carregar vários kernels de sistema operativo e kernels compatíveis com Multiboot2 em vários formatos.

[[_TOC_]]

Instalação
----------

```
 easyboot [-v|-vv] [-s <mb>] [-b <mb>] [-u <guid>] [-p <t> <u> <i>] [-e] [-c] <indir> <outfile|device>

  -v, -vv         aumentar a verbosidade/validação
  -s <mb>         defina o tamanho da imagem do disco em Megabytes (o predefinido é 35M)
  -b <mb>         definir o tamanho da partição de arranque em Megabytes (o predefinido é 33M)
  -u <guid>       define o identificador único da partição de arranque (o predefinido é aleatório)
  -p <t> <u> <i>  adicionar uma partição extra (tipo guid, guid exclusivo, ficheiro de imagem)
  -e              adicionar Catálogo de arranque El Torito (suporte para arranque de CDROM BIOS/EFI)
  -c              crie sempre um novo ficheiro de imagem, mesmo que exista
  <indir>         utilizar o conteúdo deste diretório para a partição de arranque
  <outfile>       imagem de saída ou nome do ficheiro do dispositivo
```

A ferramenta **Easyboot** cria uma imagem de disco de arranque denominada `(outfile)` utilizando a Tabela de Particionamento GUID
com uma única partição formatada como FAT32 e nomeada como "EFI System Partition" (ESP, em inglês). O conteúdo desta partição é
obtido a partir do `(indir)` que fornece. Deve colocar um ficheiro de configuração de texto simples nesse directório chamado
`easyboot/menu.cfg`. Com as terminações de linha NL ou CRLF, pode editá-las facilmente com qualquer editor de texto. Dependendo
da sua configuração, também pode precisar de alguns [plugins](plugins.md) neste directório chamado `easyboot/*.plg`.

A imagem também pode ser inicializada no Raspberry Pi e funcionará no qemu; mas para arrancar numa máquina real, serão necessários
mais ficheiros de firmware `bootcode.bin`, `fixup.dat`, `start.elf` e um ficheiro .dtb no directório `(indir)`, podem ser
descarregados do [repositório oficial](https://github.com/raspberrypi/firmware/tree/master/boot) do Raspberry Pi.

A ferramenta também possui alguns sinalizadores de linha de comando opcionais: `-s(mb)` define o tamanho global da imagem de disco
gerada em Megabytes, enquanto `-b(mb)` define o tamanho da partição de arranque em Megabytes. Obviamente, o primeiro deve ser maior
que o segundo. Se não for especificado, o tamanho da partição será calculado a partir do tamanho do directório fornecido (33 Mb no
mínimo, o FAT32 mais pequeno que pode haver) e o tamanho do disco será definido como predefinido, 2 Mb maior que este (devido ao
alinhamento e ao espaço necessário para a tabela de particionamento). Se houver uma diferença de mais de 2 Mb entre estes dois
valores de tamanho, pode utilizar ferramentas de terceiros como o `fdisk` para adicionar mais partições à imagem conforme desejar
(ou ver `-p` abaixo). Se pretender um esquema previsível, também pode especificar o identificador único da partição de arranque
(UniquePartitionGUID) com o indicador `-u <guid>`.

Opcionalmente, também pode adicionar partições extra com o sinalizador `-p`. Isto requer 3 argumentos: (PartitionTypeGUID),
(UniquePartitionGUID) e o nome do ficheiro de imagem que contém o conteúdo da partição. Esta bandeira pode ser repetida várias
vezes.

O indicador `-e` adiciona o El Torito Boot Catalog à imagem gerada, para que possa ser inicializado não só como uma pen USB, mas
também como um CD-ROM BIOS/EFI.

Se `(outfile)` for um ficheiro de dispositivo (por exemplo, `/dev/sda` no Linux, `/dev/disk0` nos BSDs e `\\. \PhysicalDrive0` no
Windows), não cria GPT nem ESP, em vez disso localiza os já existentes no dispositivo. Ainda copia todos os ficheiros em `(indir)`
para a partição de arranque e instala os carregadores. Isto também funciona se `(outfile)` for um ficheiro de imagem que já existe
(nesse caso, utilize o indicador `-c` para criar sempre primeiro um novo ficheiro de imagem vazio).

Configuração
------------

O ficheiro `easyboot/menu.cfg` pode conter as seguintes linhas (muito semelhante à sintaxe do grub.cfg, pode encontrar um exemplo
de um ficheiro de configuração [aqui](menu.cfg)):

### Comentários

Todas as linhas que começam por `#` são consideradas comentários e são saltadas até ao fim da linha.

### Nível de verbosidade

Pode definir o nível de verbosidade utilizando uma linha que começa por `verbose`.

```
verbose (0-3)
```

Isto informa o carregador quanta informação imprimir na consola de inicialização. A verbose 0 significa totalmente silencioso
(padrão) e a verbose 3 irá despejar os segmentos do kernel carregados e o código da máquina no ponto de entrada.

### Buffer de quadros

Pode solicitar uma resolução de ecrã específica com a linha que começa por `framebuffer`. O formato é o seguinte:

```
framebuffer (largura) (altura) (bits por pixel) [#(cor de primeiro plano)] [#(cor de fundo)] [#(pbarfundo)]
```

O **Easyboot** irá configurar um framebuffer para si, mesmo que esta linha não exista (800 x 600 x 32bpp por defeito). Mas se essa
linha existir, tentará definir a resolução especificada. Os modos paletados não são suportados, pelo que os bits por pixel devem
ser pelo menos 15.

Se for fornecido o quarto parâmetro de cor opcional, este deverá estar em notação HTML, começando com uma marca de hash seguida
de 6 dígitos hexadecimais, RRGGBB. Por exemplo, `#ff0000` é um vermelho brilhante e `#007f7f` é um ciano mais escuro. Define o
primeiro plano ou, por outras palavras, a cor da fonte. Da mesma forma, se o quinto parâmetro de cor opcional for fornecido,
também terá de estar em notação HTML e definirá a cor de fundo. O último parâmetro de cor opcional é semelhante e define a cor
de fundo da barra de progresso.

Quando as cores não são fornecidas, o padrão são letras brancas sobre fundo preto, e o fundo da barra de progresso é cinzento escuro.

### Opção de arranque padrão

A linha que começa com `default` define qual a entrada de menu que deve ser inicializada sem interação do utilizador após o tempo
limite especificado.

```
default (índice menuentry) (tempo limite mseg)
```

O índice menuentry é um número entre 1 e 9. O tempo limite é em milissegundos (um milésimo de segundo), pelo que 1000 dá um segundo.

### Alinhamento do menu

As linhas que começam por `menualign` alteram a forma como as opções de arranque são apresentadas.

```
menualign ("vertical"|"horizontal")
```

Por predefinição, o menu é horizontal, mas pode alterá-lo para vertical.

### Opções de arranque

Pode especificar até 9 entradas de menu com linhas que começam por `menuentry`. O formato é o seguinte:

```
menuentry (ícone) [rótulo]
```

Para cada ícone, deve existir um ficheiro `easyboot/(ícone).tga` na partição de arranque. A imagem deve estar codificada em
comprimento de execução e mapeada por cores [formato Targa](../en/TGA.txt), porque esta é a variante mais comprimida (os três
primeiros bytes do ficheiro devem ser `0`, `1` e `9` por esta ordem, ver o Tipo de Dados 9 na especificação). As suas dimensões
devem ser exatamente 64 pixels de altura e 64 pixels de largura.

Para guardar neste formato do GIMP, seleccione primeiro "Image > Mode > Indexed...", na janela pop-up defina "Maximum number of
colors" para 256. De seguida, seleccione "File > Export As...", introduza um nome de ficheiro que termine em `. tga` e, na janela
pop-up, marque "RLE compression". Para uma ferramenta de conversão de linha de comandos, pode utilizar o ImageMagick,
`convert (qualquer ficheiro de imagem) -colors 256 -compress RLE ícone.tga`.

O parâmetro de rótulo opcional é para exibir informações de versão ou lançamento ASCII no menu, e não para strings arbitrárias;
portanto, para poupar espaço, o UTF-8 não é suportado, a menos que também forneça `easyboot/font.sfn`. (Uma fonte UNICODE requer
muito armazenamento, embora a [Scalable Screen Font](https://gitlab.com/bztsrc/scalable-font2) seja muito eficiente, ainda tem
cerca de 820K. Em contraste, o unicode.pf2 do GRUB é muito maior, cerca de 2392K, embora ambas as fontes contenham cerca de 55600
glifos em bitmaps de 8x16 e 16x16. A fonte ASCII incorporada tem bit mapas de 8x16 e apenas 96 glifos.)

Todas as linhas que vêm depois de uma linha `menuentry` pertencerão a essa menuentry, excepto quando essa linha for outra linha
de menuentry. Por conveniência, pode utilizar blocos como no GRUB, mas estes são apenas detalhes sintáticos. As chaves são tratadas
como caracteres de espaço em branco. Pode omiti-las e usar tabulações, como num script Python ou Makefile, se preferir.

Por exemplo
```
menuentry FreeBSD backup
{
    kernel bsd.old/boot
}
```

### Selecionar partição

A linha que começa por `partition` selecciona uma partição GPT. Deve ser precedido por uma linha `menuentry`.

```
partition (partição UUID exclusivo)
```

Esta partição será utilizada como sistema de ficheiros raiz para a opção de arranque. O kernel e os módulos serão carregados a
partir desta partição e, dependendo do protocolo de arranque, serão também passados ​​para o kernel. A partição especificada pode
residir num disco diferente do disco de arranque; o **Easyboot** iterará em todos os discos particionados GPT durante o arranque
para o localizar.

Por conveniência, a partição é também procurada na linha `kernel` (ver abaixo). Se a linha de comando de arranque fornecida
contiver uma string `root=(UUID)` ou `root=*=(UUID)`, não há necessidade de uma linha `partition` separada.

Quando a partição não é especificada explicitamente, o kernel e os módulos são carregados a partir da partição de arranque.

### Especificar um kernel

A linha que começa por `kernel` informa qual o ficheiro que deve ser inicializado e com que parâmetros. Deve ser precedido por
uma linha `menuentry`.

```
kernel (caminho para o seu ficheiro kernel) (argumentos opcionais da linha de comandos de arranque)
```

O caminho deve apontar para um ficheiro existente, um binário executável do kernel, e deve ser um caminho UTF-8 absoluto na
partição raiz (ou de arranque). Se o kernel não estiver no directório raiz da partição, o separador de directório será sempre `/`,
mesmo em sistemas UEFI. Se o nome contiver um espaço, este terá de ser escapado com `\`. O caminho pode ser seguido por argumentos
de linha de comando, separados por um espaço. Para os kernels compatíveis com o Multiboot2, estes argumentos de linha de comando
serão passados ​​na etiqueta *linha de comando de arranque* (tipo 1). Não serão alterados pelo **Easyboot**, nem analisados, exceto
pesquisados pelo especificador de partição.

Por predefinição, o **Easyboot** pode arrancar kernels compatíveis com Multiboot2 nos formatos ELF64 e PE32+/COFF (e em sistemas
UEFI, aplicações UEFI também). Normalmente este protocolo não permite kernels da metade superior, mas o **Easyboot** viola um pouco
o protocolo de uma forma que não quebra os kernels normais que não sejam da metade superior. Se quiser arrancar qualquer outro
kernel, necessitará de um carregador de kernel [plugin](plugins.md).

NOTA: ao contrário do GRUB, onde é necessário utilizar comandos especiais como "linux" ou "multiboot" para seleccionar o protocolo
de arranque, aqui existe apenas um comando e o protocolo é detectado automaticamente pelo seu kernel em tempo de execução.

### Carregando mais módulos

Pode carregar ficheiros arbitrários (ramdisks iniciais, drivers do kernel, etc.) juntamente com o kernel utilizando linhas que
começam por `module`. Deve ser precedido por uma linha `menuentry`. Note que esta linha pode ser repetida várias vezes em cada
entrada de menu. Se o protocolo de inicialização suportar um initrd, então a primeira linha `module` será considerada como o initrd.

```
module (caminho para um ficheiro) (argumentos de linha de comando do módulo opcional)
```

O caminho deve apontar para um ficheiro existente e deve ser um caminho UTF-8 absoluto na partição raiz (ou de arranque). Pode
ser seguido por argumentos de linha de comando, separados por um espaço. Se o ficheiro estiver comprimido e existir um
[plugin](plugins.md) de descompressão para o mesmo, o módulo será descomprimido de forma transparente. A informação sobre estes
módulos carregados (e descomprimidos) será passada para um kernel compatível com Multiboot2 nas tags *Módulos* (tipo 3).

O caso especial é se o módulo começar com os bytes `DSDT`, `GUDT` ou `0xD00DFEED`. Nestes casos, o ficheiro não será adicionado às
etiquetas *Módulos*, mas a tabela ACPI será corrigida para que os seus ponteiros DSDT apontem para o conteúdo deste ficheiro. Com
isto, pode facilmente substituir uma tabela ACPI da BIOS com bugs por uma fornecida pelo utilizador.

### Logotipo de arranque

Também pode exibir um logótipo no centro do ecrã quando a opção de arranque for selecionada se colocar uma linha começando por
`bootsplash`. Deve ser precedido por uma linha `menuentry`.

```
bootsplash [#(cor de fundo)] (caminho para um ficheiro tga)
```

A cor de fundo é opcional e deve estar em notação HTML, começando com uma marca de hash seguida de 6 dígitos hexadecimais, RRGGBB.
Se o primeiro argumento não começar por `#`, então será assumido um argumento de caminho.

O caminho deve apontar para um ficheiro existente e deve ser um caminho UTF-8 absoluto na partição de arranque (NÃO raiz). A imagem
deve estar codificada num formato Targa com mapeamento de cores e comprimento de execução, tal como os ícones do menu. As dimensões
podem ser qualquer coisa que caiba no ecrã.

### Suporte multicore

Para iniciar o kernel em todos os núcleos do processador de uma só vez, especifique a directiva `multicore` (apenas kernels de 64
bits). Deve ser precedido por uma linha `menuentry`.

```
multicore
```

Solução de problemas
--------------------

Se encontrar algum problema, basta executar com o sinalizador `easyboot -vv`. Este irá executar a validação e exibir os resultados
detalhadamente no momento da criação da imagem. Caso contrário, adicione `verbose 3` a `easyboot/menu.cfg` para obter mensagens
detalhadas do tempo de arranque.

Se vir a string `PMBR-ERR` no canto superior esquerdo com um fundo vermelho, significa que o seu CPU é demasiado antigo e não
suporta o modo longo de 64 bits ou o sector de arranque não conseguiu arrancar o carregador. Pode ocorrer apenas em máquinas
com BIOS, isso nunca pode acontecer com UEFI ou no RaspberryPi.

| Mensagem                            | Descrição                                                                         |
|-------------------------------------|-----------------------------------------------------------------------------------|
| `Booting [X]...`                    | indica que a opção de inicialização (índice menuentry) X foi escolhida            |
| `Loading 'X' (Y bytes)...`          | ficheiro X de comprimento Y está a ser carregado                                  |
| `Parsing kernel...`                 | kernel foi encontrado, detetando agora o seu formato                              |
| `Starting X boot...`                | mostra que o carregador de arranque do sistema X foi detetado                     |
| `Starting X kernel...`              | mostra que o kernel do sistema X foi detetado                                     |
| `Transfering X control to Y`        | indica que o ponto de entrada do modo X no endereço Y está prestes a ser chamado  |

Se encontrar algum problema depois de ver a última mensagem, significa que o problema ocorreu no procedimento de arranque do
sistema operativo, e não no carregador do **Easyboot**, pelo que terá de consultar a documentação do sistema operativo em questão
para obter uma resposta. Caso contrário, sinta-se à vontade para abrir um [problema](https://gitlab.com/bztsrc/easyboot/-/issues)
no gitlab.

### Multiboot1

Plugins necessários: [grubmb1](../../src/plugins/grubmb1.c)

### Multiboot2

Este é o mais flexível, com múltiplas variações suportadas através de plugins:

- ELF64 ou PE32+ com Multiboot2 simplificado: não é necessário qualquer plugin
- ELF32 com Multiboot2 simplificado e ponto de entrada de 32 bits: [elf32](../../src/plugins/elf32.c)
- a.out (struct exec) com Multiboot2 simplificado e ponto de entrada de 32 bits: [aout](../../src/plugins/aout.c)
- Multiboot2 compatível com GRUB com ponto de entrada de 32 bits: [grubmb2](../../src/plugins/grubmb2.c)

Note a diferença: [Multiboot2 simplificado](ABI.md) não requer tags incorporadas, suporta kernels de metade superior, ponto de
entrada limpo de 64 bits com parâmetros passados ​​de acordo com o Multiboot2, SysV e ABI fastcall também.

Por outro lado, o [Multiboot2 compatível com GRUB](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) necessita
de tags incorporadas, não suporta kernels de metade superior nem de 64 bits, o ponto de entrada é sempre de 32 bits e os parâmetros
são passados ​​apenas em `eax`, `ebx`.

### Windows

Não são necessários plugins, mas deve configurar o [Inicialização segura](secureboot.md).

```
menuentry win {
  kernel EFI/Microsoft/BOOT/BOOTMGRW.EFI
}
```

### Linux

Plugins necessários: [linux](../../src/plugins/linux.c), [ext234](../../src/plugins/ext234.c)

Se o kernel não estiver colocado na partição de arranque, pode utilizar a linha de comandos para especificar a partição raiz.

```
menuentry linux {
  kernel vmlinuz-linux root=PARTUUID=01020304-0506-0708-0a0b0c0d0e0f1011
}
```

### OpenBSD

Plugins necessários: [obsdboot](../../src/plugins/obsdboot.c), [ufs44](../../src/plugins/ufs44.c)

```
menuentry openbsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot
}
```

AVISO! Não utilize o plugin [elf32](../../src/plugins/elf32.c) se estiver a arrancar o OpenBSD! O seu `boot` afirma incorretamente
ser um ELF com um ponto de entrada SysV ABI de 32 bits, mas na verdade tem uma entrada de modo real de 16 bits.

### FreeBSD

Plugins necessários: [fbsdboot](../../src/plugins/fbsdboot.c), [ufs2](../../src/plugins/ufs2.c)

Em sistemas BIOS legados, especifique o carregador `boot`.

```
menuentry freebsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot/boot
}
```

Nas máquinas UEFI, utilize `loader.efi` na partição de arranque (não são necessários plugins).

```
menuentry freebsd {
  kernel boot/loader.efi
}
```

Se o seu sistema de ficheiros raiz for ZFS, copie este ficheiro (`boot` na BIOS, `loader. efi` no UEFI) para `(indir)` e NÃO
especifique uma partição raiz.

### FreeDOS

Plugins necessários: [fdos](../../src/plugins/fdos.c)

Mover os ficheiros do FreeDOS para `(indir)` (o FreeDOS utilizará a partição de arranque como partição raiz).

```
menuentry freedos {
  kernel KERNEL.SYS
}
```

Se o arranque parar após imprimir copyright e `- InitDisk`, então o kernel do FreeDOS foi compilado sem suporte para FAT32.
Descarregue um kernel diferente, com `f32` no nome.

### ReactOS

Plugins necessários: [reactos](../../src/plugins/reactos.c)

```
menuentry reactos {
  kernel FREELDR.SYS
}
```

### MenuetOS

Plugins necessários: [menuet](../../src/plugins/menuet.c)

```
menuentry menuetos {
  kernel KERNEL.MNT
  module CONFIG.MNT
  module RAMDISK.MNT
}
```

Para desativar a configuração automática, adicione `noauto` à linha de comandos.

### KolibriOS

Plugins necessários: [kolibri](../../src/plugins/kolibri.c)

```
menuentry kolibrios {
  kernel KERNEL.MNT syspath=/rd/1/ launcher_start=1
  module KOLIBRI.IMG
  module DEVICES.DAT
}
```

O plugin também funciona em máquinas UEFI, mas também pode utilizar o `uefi4kos.efi` na partição de arranque (sem necessidade de
plugin).

### SerenityOS

Plugins necessários: [grubmb1](../../src/plugins/grubmb1.c)

```
menuentry serenityos {
  kernel boot/Prekernel
  module boot/Kernel
}
```

### Haiku

Plugins necessários: [grubmb1](../../src/plugins/grubmb1.c), [befs](../../src/plugins/befs.c)

```
menuentry haiku {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel system/packages/haiku_loader-r1~beta4_hrev56578_59-1-x86_64.hpkg
}
```

Nas máquinas UEFI, utilize `haiku_loader.efi` na partição de arranque (não são necessários plugins).

### OS/Z

```
menuentry osz {
  kernel ibmpc/core
  module ibmpc/initrd
}
```

Não são necessários plugins.
