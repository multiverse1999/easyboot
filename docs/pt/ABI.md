Escrever kernels compatíveis com o Easyboot
===========================================

[Easyboot](https://gitlab.com/bztsrc/easyboot) suporta vários kernels utilizando [plugins](plugins.md). Mas se não for encontrado
nenhum plugin adequado, regressa aos binários ELF64 ou PE32+ com uma variante simplificada (sem necessidade de incorporar nada) do
protocolo Multiboot2.

Este é o mesmo protocolo que o [Simpleboot](https://gitlab.com/bztsrc/simpleboot) utiliza, todos os kernels de exemplo neste
repositório devem funcionar com o **Easyboot** também.

Pode utilizar o cabeçalho multiboot2.h original no repositório do GRUB, ou o ficheiro de cabeçalho C/C++
[easyboot.h](../../easyboot.h) para obter typedefs mais fáceis de utilizar. O formato binário de baixo nível é o mesmo, também
pode utilizar qualquer biblioteca Multiboot2 existente, mesmo com linguagens que não sejam C, como esta biblioteca
[Rust](https://github.com/rust-osdev/multiboot2/tree/main/multiboot2/src) por exemplo (nota: não estou afiliado a estes
programadores de forma alguma, apenas pesquisei por "Rust Multiboot2" e este foi o primeiro resultado).

[[_TOC_]]

Sequência de arranque
---------------------

### Inicializando o carregador

Nas máquinas *BIOS*, o primeiro sector do disco é carregado a 0:0x7C00 pelo firmware, e o controlo é-lhe passado. Neste sector,
o **Easyboot** tem o [boot_x86.asm](../../src/boot_x86.asm), que é inteligente o suficiente para localizar e carregar o carregador
de 2ª fase, e também para configurar o modo longo para o mesmo.

Nas máquinas *UEFI*, o mesmo ficheiro de 2º estágio, denominado `EFI/BOOT/BOOTX64.EFI`, é carregado directamente pelo firmware. A
fonte deste carregador pode ser encontrada em [loader_x86.c](../../src/loader_x86.c). É por aí, o **Easyboot** não é o GRUB nem o
Syslinux, ambos exigindo dezenas e dezenas de ficheiros de sistema no disco. Aqui não são necessários mais ficheiros, apenas este
(os plugins são opcionais, não é necessário nenhum para fornecer compatibilidade com o Multiboot2).

No *Raspberry Pi* o carregador é designado por `KERNEL8.IMG`, compilado a partir de [loader_rpi.c](../../src/loader_rpi.c).

### O carregador

Este carregador foi cuidadosamente escrito para funcionar em diversas configurações. Carrega a tabela de particionamento GUID do
disco e procura uma "EFI System Partition". Quando encontrado, procura o ficheiro de configuração `easyboot/menu. cfg` na partição
de arranque. Após a opção de arranque ser selecionada e o nome do ficheiro do kernel ser conhecido, o carregador localiza-o e
carrega-o.

Em seguida, deteta automaticamente o formato do kernel e é inteligente o suficiente para interpretar as informações de secção e
segmento sobre onde carregar o quê (faz mapeamento de memória a pedido sempre que necessário). Em seguida, configura um ambiente
adequado dependendo do protocolo de arranque detectado (Multiboot2/Linux/etc. modo protegido ou longo, argumentos ABI etc.). Depois
de o estado da máquina estar sólido e bem definido, como último ato, o carregador salta para o ponto de entrada do seu kernel.

Estado da Máquina
-----------------

Tudo o que está escrito na [especificação Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) sobre o
estado da máquina permanece, excepto os registos de uso geral. O **Easyboot** passa dois argumentos para o ponto de entrada do seu
kernel de acordo com o SysV ABI e o Microsoft fastcall ABI. O primeiro parâmetro é a mágica, o segundo é um endereço de memória
física, apontando para uma lista de tags de informação de arranque múltiplo (abreviado como MBI a partir de agora, ver abaixo).

Também violámos um pouco o protocolo Multiboot2 para lidar com kernels de metade superior. O Multiboot2 exige que a memória seja
mapeada por identidade. Bem, no **Easyboot** isto é apenas parcialmente verdade: apenas garantimos que toda a RAM física está
certamente mapeada com identidade como esperado; no entanto, algumas regiões acima deste (dependendo dos cabeçalhos dos programas
do kernel) podem ainda estar disponíveis. Isto não interrompe os kernels normais compatíveis com Multiboot2, que não devem aceder
a memória fora da RAM física disponível.

O seu kernel é carregado exatamente da mesma forma nos sistemas BIOS e UEFI, bem como no RPi; as diferenças de firmware são apenas
"um problema de outra pessoa". A única coisa que o seu kernel verá é se o MBI contém a etiqueta da tabela do sistema EFI ou não.
Para simplificar a sua vida, o **Easyboot** também não gera a tag de mapa de memória EFI (tipo 17), apenas fornece a tag
[Mapa de memória](#memory_map) (tipo 6) indiscriminadamente em todas as plataformas (nos sistemas UEFI também, onde o mapa de
memória é simplesmente convertido para si, pelo que o seu kernel tem de lidar apenas com um tipo de tag de lista de memória). As
tags antigas e obsoletas também são omitidas e nunca geradas por este gestor de arranque.

O kernel está a ser executado ao nível do supervisor (anel 0 em x86, EL1 em ​​ARM), possivelmente em todos os núcleos da CPU em
paralelo.

GDT não especificado, mas válido. A pilha é configurada nos primeiros 640k e cresce para baixo (mas deve mudar isto o mais
rapidamente possível para qualquer pilha que considere digna). Quando o SMP está ativado, todos os núcleos têm as suas próprias
pilhas, e o ID do núcleo está no topo da pilha (mas também pode obter o ID do núcleo da forma habitual específica da plataforma,
utilizando cpuid/mpidr/etc.).

Deve considerar IDT como não especificado; IRQs, NMI e interrupções de software desativadas. Os manipuladores de exceções fictícios
são configurados para exibir um dump mínimo e parar a máquina. Devem ser utilizados apenas para reportar se o seu kernel apresentar
problemas antes de conseguir configurar o seu próprio IDT e manipuladores, de preferência o mais rapidamente possível. No ARM, o
vbar_el1 está configurado para chamar os mesmos manipuladores de exceções fictícios (embora despejem registos diferentes, claro).

Buffer de quadros também está definido por defeito. Pode alterar a resolução na configuração, mas se não for fornecida, o
framebuffer será ainda configurado.

É importante nunca regressar do seu kernel. É livre de sobrescrever qualquer parte do carregador na memória (assim que terminar com
as tags MBI), pelo que não há para onde voltar. "Der Mohr hat seine Schuldigkeit getan, der Mohr kann gehen."

Informação de arranque passada para o seu kernel (MBI)
------------------------------------------------------

Não é óbvio à partida, mas a especificação Multiboot2 define, na realidade, dois conjuntos de tags totalmente independentes:

- O primeiro conjunto deveria ser incorporado num kernel compatível com Multiboot2, denominado cabeçalho Multiboot2 da imagem do
  sistema operativo (secção 3.1.2), portanto *fornecido pelo kernel*. O **Easyboot** não se importa com estas tags e também não
  analisa o seu kernel em busca delas. Simplesmente não precisa de nenhum dado mágico especial incorporado no seu ficheiro kernel
  com **Easyboot**, os cabeçalhos ELF e PE são suficientes.

- O segundo conjunto é *passado para o kernel* dinamicamente no arranque, o **Easyboot** apenas utiliza estas tags. No entanto, não
  gera tudo o que o Multiboot2 especifica (simplesmente omite os antigos, obsoletos ou legados). Estas etiquetas são designadas por
  etiquetas MBI, ver [Informações de arranque](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Boot-information-format)
  (secção 3.6).

NOTA: a especificação Multiboot2 nas tags MBI está repleta de bugs. Pode encontrar uma versão corrigida abaixo, que se alinha com
o ficheiro de cabeçalho multiboot2.h que pode encontrar no repositório de origem do GRUB.

O primeiro parâmetro do seu kernel é o magic 0x36d76289 (em `rax`, `rcx` e `rdi`). Pode localizar as tags MBI utilizando o segundo
parâmetro (em `rbx`, `rdx` e `rsi`). Na plataforma ARM, a magia está em `x0` e o endereço está em `x1`. No RISC-V e no MIPS, são
utilizados `a0` e `a1`, respectivamente. Se e quando este carregador for portado para outra arquitetura, os registos especificados
pelo SysV ABI para os argumentos das funções terão sempre de ser utilizados. Se existirem outras ABI comuns na plataforma que não
interfiram com a ABI do SysV, os valores também terão de ser duplicados nos registos dessas ABI (ou no topo da pilha).

### Cabeçalhos

O endereço passado é sempre alinhado com 8 bytes e começa com um cabeçalho MBI:

```
        +-------------------+
u32     | total_size        |
u32     | reserved          |
        +-------------------+
```

Segue-se uma série de tags também alinhadas de 8 bytes. Cada tag começa com os seguintes campos de cabeçalho de tag:

```
        +-------------------+
u32     | type              |
u32     | size              |
        +-------------------+
```

`type` contém um identificador do conteúdo do resto da etiqueta. `size` contém o tamanho da etiqueta, incluindo os campos de
cabeçalho, mas não incluindo o padding. As etiquetas seguem-se umas às outras, preenchidas quando necessário, para que cada
etiqueta comece num endereço alinhado de 8 bytes.

### Terminada

```
        +-------------------+
u32     | type = 0          |
u32     | size = 8          |
        +-------------------+
```

As tags são terminadas por uma tag do tipo `0` e de tamanho `8`.

### Linha de comando de arranque

```
        +-------------------+
u32     | type = 1          |
u32     | size              |
u8[n]   | string            |
        +-------------------+
```

`string` contém a linha de comando especificada na linha `kernel` do *menuentry* (sem o caminho do kernel e o nome do ficheiro).
A linha de comandos é uma string UTF-8 terminada em zero, ao estilo C.

### Nome do carregador de arranque

```
        +----------------------+
u32     | type = 2             |
u32     | size = 17            |
u8[n]   | string "Easyboot"    |
        +----------------------+
```

`string` contém o nome de um carregador de arranque que inicializa o kernel. O nome é uma string UTF-8 terminada em zero, ao
estilo C.

### Módulos

```
        +-------------------+
u32     | type = 3          |
u32     | size              |
u32     | mod_start         |
u32     | mod_end           |
u8[n]   | string            |
        +-------------------+
```

Esta etiqueta indica ao kernel qual o módulo de arranque que foi carregado juntamente com a imagem do kernel e onde pode ser
encontrado. `mod_start` e `mod_end` contêm os endereços físicos inicial e final do próprio módulo de arranque. Nunca obterá um
buffer comprimido com gzip, porque o **Easyboot** descompacta-os de forma transparente para si (e se fornecer um plugin,
também funciona com dados comprimidos que não sejam gzip). O campo `string` fornece uma string arbitrária a associar a esse
módulo de inicialização específico; é uma string UTF-8 normal de estilo C com terminação zero. Especificado na linha `module`
do *menuentry* e a sua utilização exacta é específica do sistema operativo. Ao contrário da tag de linha de comando boot, as tags
do módulo *também incluem* o caminho e o nome do ficheiro do módulo.

Aparece uma tag por módulo. Este tipo de etiqueta pode aparecer várias vezes. Se um ramdisk inicial tiver sido carregado juntamente
com o seu kernel, este aparecerá como o primeiro módulo.

Existe um caso especial, se o ficheiro for uma tabela DSDT ACPI, um FDT (dtb) ou GUDT blob, então não aparecerá como um módulo,
em vez disso, o ACPI antigo RSDP (tipo 14) ou o ACPI novo RSDP (tipo 15) serão corrigidos e o seu DSDT substituído pelo conteúdo
deste ficheiro.

### Mapa de memória

Esta etiqueta fornece um mapa de memória.

```
        +-------------------+
u32     | type = 6          |
u32     | size              |
u32     | entry_size = 24   |
u32     | entry_version = 0 |
varies  | entries           |
        +-------------------+
```

`size` contém o tamanho de todas as entradas, incluindo este campo. `entry_size` é sempre 24. `entry_version` está definido para `0`.
Cada entrada tem a seguinte estrutura:

```
        +-------------------+
u64     | base_addr         |
u64     | length            |
u32     | type              |
u32     | reserved          |
        +-------------------+
```

`base_addr` é o endereço físico inicial. `length` é o tamanho da região de memória em bytes. `type` é a variedade de intervalo de
endereços representada, em que um valor de `1` indica RAM disponível, o valor de `3` indica memória utilizável contendo informação
ACPI, o valor de `4` indica memória reservada que necessita de ser preservada na hibernação, o valor de `5` indica uma memória que
está ocupada por módulos de RAM defeituosos e todos os outros valores indicam actualmente uma área reservada. `reserved` é definido
como `0` nas inicializações da BIOS.

Quando o MBI é gerado numa máquina UEFI, várias entradas do Mapa de Memória EFI são armazenadas como tipo `1` (RAM disponível)
ou `2` (RAM reservada) e, caso necessite, o Tipo de Memória EFI original é colocado no campo `reserved`.

O mapa fornecido listará toda a RAM padrão que deve estar disponível para utilização normal e é sempre ordenado por `base_addr`
crescente. Este tipo de RAM disponível, no entanto, inclui as regiões ocupadas pelo kernel, mbi, segmentos e módulos. O kernel
deve ter cuidado para não sobrescrever estas regiões (o **Easyboot** poderia facilmente excluir estas regiões, mas isso quebraria
a compatibilidade do Multiboot2).

### Informações do buffer de quadros

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

O campo `framebuffer_addr` contém o endereço físico do framebuffer. O campo `framebuffer_pitch` contém o comprimento de uma linha em
bytes. Os campos `framebuffer_width`, `framebuffer_height` contêm as dimensões do framebuffer em pixéis. O campo `framebuffer_bpp`
contém o número de bits por pixel. `framebuffer_type` é sempre definido como 1, e `reserved` contém sempre 0 na versão actual da
especificação e deve ser ignorado pela imagem do sistema operativo. Os restantes campos descrevem o formato do pixel comprimido,
a posição dos canais e o tamanho em bits. Pode utilizar a expressão `((~(0xffffffff << tamanho)) << posição) & 0xffffffff` para
obter uma máscara de canal semelhante à UEFI GOP.

### Ponteiro de tabela do sistema EFI de 64 bits

Esta etiqueta só existe se o **Easyboot** estiver a ser executado numa máquina UEFI. Numa máquina BIOS, esta tag nunca foi gerada.

```
        +-------------------+
u32     | type = 12         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Esta etiqueta contém um ponteiro para a tabela do sistema EFI.

### Ponteiro de identificador de imagem EFI de 64 bits

Esta etiqueta só existe se o **Easyboot** estiver a ser executado numa máquina UEFI. Numa máquina BIOS, esta tag nunca foi gerada.

```
        +-------------------+
u32     | type = 20         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Esta etiqueta contém um ponteiro para o identificador de imagem EFI. Normalmente é o identificador da imagem do carregador de
arranque.

### Tabelas SMBIOS

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

Esta etiqueta contém uma cópia das tabelas SMBIOS, bem como as suas versões.

### ACPI antigo RSDP

```
        +-------------------+
u32     | type = 14         |
u32     | size              |
        | copy of RSDPv1    |
        +-------------------+
```

Esta etiqueta contém uma cópia do RSDP tal como definido pela especificação ACPI 1.0. (Com um endereço de 32 bits.)

### ACPI novo RSDP

```
        +-------------------+
u32     | type = 15         |
u32     | size              |
        | copy of RSDPv2    |
        +-------------------+
```

Esta etiqueta contém uma cópia do RSDP, tal como definido pela especificação ACPI 2.0 ou posterior. (Provavelmente com um endereço
de 64 bits.)

Eles (tipo 14 e 15) apontam para uma tabela `RSDT` ou `XSDT` com um ponteiro para uma tabela `FACP`, que por sua vez contém
dois ponteiros para uma tabela `DSDT`, que descreve a máquina. O **Easyboot** falsifica estas tabelas em máquinas que não
suportam ACPI. Além disso, se fornecer uma tabela DSDT, um FDT (dtb) ou um blob GUDT como módulo, o **Easyboot** irá corrigir
os ponteiros para apontar para a tabela fornecida pelo utilizador. Para analisar estas tabelas, pode utilizar a minha biblioteca
[hwdet](https://gitlab.com/bztsrc/hwdet) de cabeçalho único e livre de dependências (ou as inchadas
[apcica](https://github.com/acpica/acpica) e [libfdt](https://github.com/dgibson/dtc)).

Tags específicas do kernel
--------------------------

As tags com `type` maior ou igual a 256 não fazem parte da especificação Multiboot2, mas são fornecidas pelo **Easyboot**.
Podem ser adicionados por [plugins](plugins.md) opcionais à lista, se e quando um kernel necessitar deles.

### EDID

```
        +-------------------+
u32     | type = 256        |
u32     | size              |
        | copy of EDID      |
        +-------------------+
```

Esta etiqueta contém uma cópia da lista de resoluções de monitores suportadas de acordo com a especificação EDID.

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

Esta etiqueta existe se a directiva `multicore` tiver sido fornecida. `numcores` contém o número de núcleos de CPU no
sistema, `running` é o número de núcleos que foram inicializados com sucesso e estão a executar o mesmo kernel em paralelo.
O `bspid` contém o identificador do núcleo BSP (no ID x86 lAPIC), para que os kernels possam distinguir os AP e executar um
código diferente nos mesmos. Todos os AP têm a sua própria pilha e, no topo da pilha, estará o ID do núcleo atual.

### Identificadores de Partição

```
        +-------------------+
u32     | type = 258        |
u32     | size = 24 / 40    |
u128    | bootuuid          |
u128    | rootuuid          |
        +-------------------+
```

Esta etiqueta contém os campos de identificadores únicos no GPT das partições de arranque e raiz. Se a inicialização não utilizar
uma Tabela de Particionamento GUID, então `bootuuid` será gerado como `54524150-(código do dispositivo)-(número da partição)-616F6F7400000000`.

Layout de memória
-----------------

### Máquinas BIOS

| Início   | Fim     | Descrição                                                       |
|---------:|--------:|-----------------------------------------------------------------|
|      0x0 |   0x400 | Interrupt Vector Table (utilizável, modo real IDT)              |
|    0x400 |   0x4FF | BIOS Data Area (utilizável)                                     |
|    0x4FF |   0x500 | código da unidade de arranque da BIOS (provavelmente 0x80, utilizável) |
|    0x500 |   0x5A0 | dados de sincronização para SMP (utilizáveis)                   |
|    0x5A0 |  0x1000 | pilha de manipuladores de exceções (utilizável após configurar o seu IDT) |
|   0x1000 |  0x8000 | tabelas de paginação (utilizáveis ​​após configurar as suas tabelas de paginação) |
|   0x8000 | 0x20000 | código e dados do carregador (utilizáveis ​​após configurar o seu IDT) |
|  0x20000 | 0x40000 | config + tags (utilizável após análise do MBI)                  |
|  0x40000 | 0x90000 | plugin ids; de cima para baixo: pilha do kernel                 |
|  0x90000 | 0x9A000 | apenas kernel Linux: zero page + cmdline                        |
|  0x9A000 | 0xA0000 | Extended BIOS Data Area (melhor não tocar)                      |
|  0xA0000 | 0xFFFFF | VRAM e BIOS ROM (não utilizável)                                |
| 0x100000 |       x | segmentos do kernel, seguidos dos módulos, cada página alinhada |

### Máquinas UEFI

Ninguém sabe. A UEFI aloca a memória como bem entender. Espere tudo e mais alguma coisa. Todas as áreas serão certamente listadas
no mapa de memória como type = 1 (`MULTIBOOT_MEMORY_AVAILABLE`) e reserved = 2 (`EfiLoaderData`), no entanto isto não é exclusivo,
outros tipos de memória também podem ser listados assim (secção bss do gestor de arranque, por exemplo).

### Raspberry Pi

| Início   | Fim     | Descrição                                                           |
|---------:|--------:|---------------------------------------------------------------------|
|      0x0 |   0x500 | reservado pelo firmware (melhor não tocar)                          |
|    0x500 |   0x5A0 | dados de sincronização para SMP (utilizáveis)                       |
|    0x5A0 |  0x1000 | pilha de manipuladores de exceções (utilizável após configurar o seu VBAR) |
|   0x1000 |  0x9000 | tabelas de paginação (utilizáveis ​​após configurar as suas tabelas de paginação) |
|   0x9000 | 0x20000 | código e dados do carregador (utilizáveis ​​após configurar o seu VBAR) |
|  0x20000 | 0x40000 | config + tags (utilizável após análise do MBI)                      |
|  0x40000 | 0x80000 | firmware fornecido FDT (dtb); de cima para baixo: pilha do kernel   |
| 0x100000 |       x | segmentos do kernel, seguidos dos módulos, cada página alinhada     |

Os primeiros bytes estão reservados para [armstub](https://github.com/raspberrypi/tools/blob/master/armstubs/armstub8.S). Apenas
o núcleo 0 foi iniciado, pelo que para iniciar os Processadores de Aplicação, escreva o endereço de uma função em 0xE0 (núcleo 1),
0xE8 (núcleo 2), 0xF0 (núcleo 3), cujos endereços se encontram nesta área. Isto é irrelevante quando a directiva `multicore` é
utilizada, pelo que todos os núcleos irão executar o kernel.

Embora não seja suportado nativamente no RPi, ainda obtém uma etiqueta ACPI antiga RSDP (tipo 14), com tabelas falsas. A tabela
`APIC` é utilizada para comunicar o número de núcleos de CPU disponíveis ao kernel. O endereço da função de arranque é guardado
no campo RSD PTR -> RSDT -> APIC -> cpu\[x].apic_id (e o ID do núcleo em cpu\[x].acpi_id, onde BSP é sempre cpu\[0].acpi_id = 0
e cpu\[0].apic_id = 0xD8. Cuidado, "acpi" e "apic" são bastante semelhantes).

Se for passado um blob FDT válido pelo firmware, ou se um dos módulos for um ficheiro .dtb, .gud ou .aml, então será também
adicionada uma tabela FADT (com `FACP` mágico). Nesta tabela, o ponteiro DSDT (32 bits, no offset 40) está a apontar para o blob
da árvore de dispositivos achatada fornecida.

Embora o firmware não forneça qualquer funcionalidade de mapa de memória, receberá na mesma uma etiqueta de Mapa de Memória (tipo 6),
listando a RAM detetada e a região MMIO. Pode utilizar isto para detetar o endereço base do MMIO, que é diferente no RPi3 e no RPi4.
