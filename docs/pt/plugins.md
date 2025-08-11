Easyboot Plugins
================

Por predefinição, o [Easyboot](https://gitlab.com/bztsrc/easyboot) arranca os kernels compatíveis com Multiboot2 nos formatos ELF
e PE a partir da partição de arranque. Se o seu kernel utilizar um formato de ficheiro diferente, um protocolo de arranque
diferente ou não estiver localizado na partição de arranque, necessitará de plugins na partição de arranque. Pode encontrá-los
no directório [src/plugins](../../src/plugins).

[[_TOC_]]

Instalação
----------

Para instalar plugins, basta copiá-los para o directório especificado no parâmetro `(indir)`, no subdirectório `easyboot` junto
ao ficheiro menu.cfg.

Por exemplo:
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

Compilação
----------

*Foi óbvio desde o início que o ELF não era adequado para esta tarefa. É muito inchado e muito complexo. Então, originalmente
queria usar o struct exec (o formato clássico a.out do UNIX), mas infelizmente as cadeias de ferramentas modernas já não conseguem
criar isso. Decidi então criar o meu próprio formato e o meu próprio linker para os plugins.*

Pode compilar o código-fonte do plugin com qualquer compilador cruzado ANSI C padrão num ficheiro de objeto ELF, mas depois terá
de utilizar o ligador [plgld](../../src/misc/plgld.c) para criar o binário final. Este é um reticulador independente da arquitetura
e funcionará independentemente do código de máquina para o qual o plugin foi compilado. O . plg final é apenas uma fracção do
tamanho do .o ELF a partir do qual foi gerado.

### Plugin API

O código-fonte C de um plugin deve incluir o ficheiro de cabeçalho `src/loader.h` e deve conter uma linha `EASY_PLUGIN`. Este
tem um parâmetro, o tipo do plugin, seguido da especificação de correspondência do identificador. Este último é utilizado pelo
carregador para determinar quando utilizar aquele plugin específico.

Por exemplo:

```c
#include "../loader.h"

/* bytes mágicos que identificam um kernel linux */
EASY_PLUGIN(PLG_T_KERNEL) {
   /* desvio tamanho   tipo        bytes mágicos */
    { 0x1fe,     2, PLG_M_CONST, { 0xAA, 0x55, 0, 0 } },
    { 0x202,     4, PLG_M_CONST, { 'H', 'd', 'r', 'S' } }
};

/* ponto de entrada, o protótipo é definido pelo tipo do plugin */
PLG_API void _start(uint8_t *buf, uint64_t size);
{
    /* preparar ambiente para um kernel linux */
}
```

Os plugins podem utilizar diversas variáveis ​​e funções, todas elas definidas no ficheiro de cabeçalho e ligadas em tempo de execução.

```c
uint32_t verbose;
```
Nível de verbosidade. Um plugin só pode imprimir qualquer saída se esta for diferente de zero, exceto para mensagens de erro.
Quanto maior for o seu valor, mais detalhes devem ser impressos.

```c
uint64_t file_size;
```
O tamanho total do ficheiro aberto (ver `open` e `loadfile` abaixo).

```c
uint8_t *root_buf;
```
Quando um plugin do sistema de ficheiros é inicializado, contém os primeiros 128k da partição (incluindo, com sorte, o super-bloco).
Mais tarde, um plugin de sistema de ficheiros pode reutilizar este buffer de 128k para qualquer finalidade (cache FAT, cache de
inode, etc.)

```c
uint8_t *tags_buf;
```
Contém as tags Multiboot2. Um plugin do kernel pode analisar isto para converter os dados fornecidos pelo gestor de arranque para
qualquer formato esperado pelo kernel. Este ponteiro aponta para o início do buffer.

```c
uint8_t *tags_ptr;
```
Este ponteiro aponta para o fim do buffer de tags Multiboot2. Os plugins de tags podem adicionar aqui novas tags e ajustar este
ponteiro.

```c
uint8_t *rsdp_ptr;
```
Aponta para o ponteiro RSDP ACPI.

```c
uint8_t *dsdt_ptr;
```
Aponta para o blob de descrição de hardware DSDT (ou GUDT, FDT).

```c
efi_system_table_t *ST;
```
Nas máquinas UEFI, aponta para a tabela do sistema EFI, caso contrário, `NULL`.

```c
void memset(void *dst, uint8_t c, uint32_t n);
void memcpy(void *dst, const void *src, uint32_t n);
int  memcmp(const void *s1, const void *s2, uint32_t n);
```
Funções de memória obrigatórias (o compilador C pode gerar chamadas para as mesmas, mesmo quando não existe uma chamada direta).

```c
void *alloc(uint32_t num);
```
Aloca `num` páginas (4k) de memória. Os plugins não devem alocar muito, devem ter como objetivo ocupar o mínimo de memória possível.

```c
void free(void *buf, uint32_t num);
```
Libertar memória previamente alocada de `num` páginas.

```c
void printf(char *fmt, ...);
```
Imprime uma string formatada na consola de arranque.

```c
uint64_t pb_init(uint64_t size);
```
Inicia a barra de progresso, `size` é o tamanho total que representa. Retorna quantos bytes vale um pixel.

```c
void pb_draw(uint64_t curr);
```
Desenha a barra de progresso para o valor atual. `curr` deve estar entre 0 e o tamanho total.

```c
void pb_fini(void);
```
Fecha a barra de progresso e limpa o seu espaço no ecrã.

```c
void loadsec(uint64_t sec, void *dst);
```
Utilizado pelos plugins do sistema de ficheiros, carrega um sector do disco para a memória. `sec` é o número do sector, relativo
à partição raiz.

```c
void sethooks(void *o, void *r, void *c);
```
Utilizado pelos plugins do sistema de ficheiros, define os ganchos das funções de open / read / close para o sistema de ficheiros
da partição raiz.

```c
int open(char *fn);
```
Abre um ficheiro na partição raiz (ou de arranque) para leitura, devolve 1 em caso de sucesso. Apenas um ficheiro pode ser aberto
de cada vez. Quando não existe nenhuma chamada `sethooks` anterior, opera na partição de arranque.

```c
uint64_t read(uint64_t offs, uint64_t size, void *buf);
```
Lê os dados do ficheiro aberto na posição de pesquisa `offs` na memória e devolve o número de bytes realmente lidos.

```c
void close(void);
```
Fecha o ficheiro aberto.

```c
uint8_t *loadfile(char *path);
```
Carregue um ficheiro inteiramente da partição raiz (ou de arranque) para um buffer de memória recém-alocado e descomprima-o de
forma transparente se o plugin for encontrado. Tamanho devolvido em `file_size`.

```c
int loadseg(uint32_t offs, uint32_t filesz, uint64_t vaddr, uint32_t memsz);
```
Carregue um segmento do buffer do kernel. Este verifica se a memória `vaddr` está disponível e mapeia o segmento se for metade
superior. `offs` é o deslocamento do ficheiro, portanto, relativo ao buffer do kernel. Se `memsz` for maior que `filesz`, a
diferença será preenchida com zeros.

```c
void _start(void);
```
Ponto de entrada para os plugins do sistema de ficheiros (`PLG_T_FS`). Deve analisar o superbloco em `root_buf` e chamar `sethooks`.
Em caso de erro, deverá retornar sem definir os seus ganchos.

```c
void _start(uint8_t *buf, uint64_t size);
```
Ponto de entrada para os plugins do kernel (`PLG_T_KERNEL`). Recebe a imagem do kernel na memória, deve realocar os seus segmentos,
configurar o ambiente adequado e transferir o controlo. Quando não há erro, nunca mais retorna.

```c
uint8_t *_start(uint8_t *buf);
```
Ponto de entrada para plugins descompressores (`PLG_T_DECOMP`). Recebe o buffer comprimido (e o seu tamanho em `file_size`) e deve
devolver um novo buffer alocado com os dados descomprimidos (e definir o tamanho do novo buffer também em `file_size`). Deve
libertar o buffer antigo (atenção, `file_size` está em bytes, mas free() espera tamanho em páginas). Em caso de erro, `file_size`
não deve ser alterado e deve devolver o buffer original não modificado.

```c
void _start(void);
```
Ponto de entrada para plugins de tags (`PLG_T_TAG`). Podem adicionar novas tags em `tags_ptr` e ajustar este ponteiro para uma nova
posição alinhada de 8 bytes.

### Funções locais

Os plugins podem utilizar funções locais, no entanto, devido a um bug do CLang, *devem* ser declaradas como `static`. (O bug é que
o CLang gera registos PLT para estas funções, mesmo que o sinalizador "-fno-plt" seja passado na linha de comandos. Usar `static`
contorna isto).

Especificação de formato de ficheiro de baixo nível
---------------------------------------------------

Caso alguém queira escrever um plugin numa linguagem que não seja C (em Assembly, por exemplo), aqui fica uma descrição de baixo
nível do formato do ficheiro.

É muito semelhante ao formato a.out. O ficheiro consiste num cabeçalho de tamanho fixo, seguido de secções de comprimento variável.
Não existe cabeçalho de secção, os dados de cada secção seguem diretamente os dados da secção anterior pela seguinte ordem:

```
(cabeçalho)
(registos de correspondência de identificadores)
(registos de realocação)
(código de máquina)
(dados de leitura apenas)
(dados inicializados legíveis e graváveis)
```

Para a primeira secção real, código de máquina, o alinhamento é incluído. Para todas as outras secções, o preenchimento é
adicionado ao tamanho da secção anterior.

DICA: se passar um plugin como um único argumento para `plgld`, este despeja as secções no ficheiro com uma saída semelhante
a `readelf -a` ou `objdump -xd`.

### Cabeçalho

Todos os números estão no formato little-endian, independentemente da arquitetura.

| Desvio | Tamanho | Descrição                                                     |
|--------:|------:|----------------------------------------------------------------|
|       0 |     4 | bytes mágicos `EPLG`                                           |
|       4 |     4 | tamanho total do ficheiro                                      |
|       8 |     4 | memória total necessária quando o ficheiro é carregado         |
|      12 |     4 | tamanho da secção de código                                    |
|      16 |     4 | tamanho da secção de dados de leitura apenas                   |
|      20 |     4 | ponto de entrada do plugin                                     |
|      24 |     2 | código de arquitetura (o mesmo do ELF)                         |
|      26 |     2 | número de registos de realocação                               |
|      28 |     1 | número de registos de correspondência de identificadores       |
|      29 |     1 | entrada GOT mais referenciada                                  |
|      30 |     1 | revisão do formato do ficheiro (0 por enquanto)                |
|      31 |     1 | tipo de plugin (1=sistema de ficheiros, 2=kernel, 3=descompressor, 4=tag) |

O código de arquitetura é o mesmo dos cabeçalhos ELF, por exemplo 62 = x86_64, 183 = Aarch64 e 243 = RISC-V.

O tipo do plugin especifica o protótipo do ponto de entrada, o ABI é sempre SysV.

### Secção de correspondência de identificadores

Esta secção contém tantos dos seguintes registos quantos os especificados no campo "número de registos de correspondência de
identificadores" do cabeçalho.

| Desvio | Tamanho | Descrição                                               |
|--------:|------:|----------------------------------------------------------|
|       0 |     2 | desvio                                                   |
|       2 |     1 | tamanho                                                  |
|       3 |     1 | tipo                                                     |
|       4 |     4 | bytes mágicos para combinar                              |

Primeiro, o início do assunto é carregado para um buffer. Um acumulador é configurado, inicialmente com 0. Os deslocamentos nestes
registos são sempre relativos a este acumulador e endereçam este byte no buffer.

O campo Tipo informa como interpretar o deslocamento. Se for 1, o deslocamento mais o acumulador será utilizado como valor. Se
for 2, então é obtido um valor de byte de 8 bits no deslocamento, 3 significa obter um valor de palavra de 16 bits e 4 significa
obter um valor dword de 32 bits. 5 significa obter um valor de byte de 8 bits e adicionar-lhe o acumulador, 6 significa obter um
valor de palavra de 16 bits e adicionar-lhe o acumulador, e 7 é o mesmo, mas com um valor de 32 bits. 8 procurará os bytes mágicos
do byte acumulador até ao fim do buffer em passos de deslocamento e, se forem encontrados, devolverá o deslocamento correspondente
como valor.

Se o tamanho for zero, o acumulador será definido como o valor. Se o tamanho não for zero, a quantidade de bytes será verificada
para ver se corresponde aos bytes mágicos fornecidos.

Por exemplo, para verificar se um executável PE começa com uma instrução NOP:
```c
EASY_PLUGIN(PLG_T_KERNEL) {
   /* desvio tamanho   tipo        bytes mágicos */
    { 0,         2, PLG_M_CONST, { 'M', 'Z', 0, 0 } },      /* verificar bytes mágicos */
    { 60,        0, PLG_M_DWORD, { 0, 0, 0, 0 } },          /* obter o deslocamento do cabeçalho PE para o acumulador */
    { 0,         4, PLG_M_CONST, { 'P', 'E', 0, 0 } },      /* verificar bytes mágicos */
    { 40,        1, PLG_M_DWORD, { 0x90, 0, 0, 0 } }        /* verifique a instrução NOP no ponto de entrada */
};
```

### Secção de realocação

Esta secção contém tantos dos seguintes registos quantos os especificados no campo "número de registos de realocação" do cabeçalho.

| Desvio | Tamanho | Descrição                                               |
|--------:|------:|----------------------------------------------------------|
|       0 |     4 | desvio                                                   |
|       4 |     4 | tipo de realocação                                       |

Significado de bits no tipo:

| De      | Para  | Descrição                                                |
|--------:|------:|----------------------------------------------------------|
|       0 |     7 | símbolo (0 - 255)                                        |
|       8 |     8 | endereçamento relativo do PC                             |
|       9 |     9 | endereçamento indireto relativo GOT                      |
|      10 |    13 | índice de máscara imediata (0 - 15)                      |
|      14 |    19 | iniciar bit (0 - 63)                                     |
|      20 |    25 | fim bit (0 - 63)                                         |
|      26 |    31 | posição do bit do sinalizador de endereço negado (0 - 63) |

O campo offset é relativo à magia no cabeçalho do plugin e seleciona um número inteiro na memória onde deve ser realizada a
realocação.

O símbolo informa qual o endereço a utilizar. 0 significa o endereço BASE onde o plugin foi carregado para a memória, também
conhecido por. o endereço mágico do cabeçalho na memória. Outros valores selecionam um endereço de símbolo externo do GOT,
definido no carregador ou noutro plugin. Dê uma vista de olhos ao array `plg_got` na fonte do plgld.c para ver qual o valor
que corresponde a cada símbolo. Se o bit relativo GOT for 1, então será utilizado o endereço da entrada GOT do símbolo, em
vez do endereço real do símbolo.

Se o bit relativo do PC for 1, o deslocamento será primeiro subtraído do endereço (modo de endereçamento relativo do ponteiro
de instruções).

O índice de máscara imediata informa quais os bits que armazenam o endereço na instrução. Se for 0, o endereço será escrito tal
como está no deslocamento, independentemente da arquitetura. Para x86_64, apenas é permitido o índice 0. Para ARM Aarch64: 0 =
como está, 1 = 0x07ffffe0 (deslocar 5 bits para a esquerda), 2 = 0x07fffc00 (deslocar 10 bits para a esquerda), 3 = 0x60ffffe0
(com instruções ADR/ADRP, o imediato é deslocado e dividido em dois grupos de bits). As arquiteturas futuras podem definir mais
e diferentes máscaras de bits imediatas.

Utilizando a máscara imediata, os bits fim - início + 1 são retirados da memória e estendidos com sinal. Este valor é adicionado
ao endereço (adendo e, no caso de referências internas, o endereço do símbolo interno também é codificado aqui).

Se o bit de sinalização de endereço negado não for 0 e o endereço for positivo, este bit será limpo. Se o endereço for negativo,
este bit será colocado a 1 e o endereço será negado.

Por fim, os bits inicial e final selecionam qual a parte do endereço que será gravada no número inteiro selecionado. Este também
define o tamanho da realocação; bits fora desse intervalo e os que não fazem parte da máscara imediata permanecem inalterados.

### Secção de código

Esta secção contém instruções de máquina para a arquitetura especificada no cabeçalho e tem tantos bytes quantos o campo de
tamanho do código indica.

### Secção de dados de leitura apenas

Esta é uma secção opcional e pode estar em falta. É tão longo quanto o campo de tamanho da secção de leitura apenas no cabeçalho
indica. Todas as variáveis ​​constantes são colocadas nesta secção.

### Secção de dados inicializada

Esta é uma secção opcional e pode estar em falta. Se ainda existirem bytes no ficheiro após a secção de código (ou a secção de
dados de leitura apenas opcional), todos esses bytes serão considerados a secção de dados. Se uma variável for inicializada com
um valor diferente de zero, será colocada nesta secção.

### Secção BSS

Esta é uma secção opcional e pode estar em falta. Esta secção nunca é armazenada no ficheiro. Se o campo de tamanho na memória
for maior do que o campo de tamanho do ficheiro no cabeçalho, a diferença será preenchida com zeros na memória. Se uma variável
não for inicializada, ou for inicializada como zero, será colocada nesta secção.
