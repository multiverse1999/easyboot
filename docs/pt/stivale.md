Suporte Stivale no Easyboot
===========================

Não vai acontecer, nunca. Este protocolo de arranque tem algumas escolhas de design muito más e representa um grande risco de
segurança.

Primeiro, os kernels stivale têm um cabeçalho ELF, mas de alguma forma deveria saber que o cabeçalho não é válido; nada,
absolutamente nada, indica no cabeçalho que não é um kernel SysV ABI válido. No Multiboot devem existir alguns bytes mágicos no
início do ficheiro para que se possa detectar o protocolo; não há nada parecido no stivale/stivale2. (A âncora não ajuda, porque
isto pode ocorrer *em qualquer lugar* do ficheiro, pelo que é necessário *pesquisar o ficheiro inteiro* para ter a certeza de que
não é compatível com o stivale.)

Em segundo lugar, utiliza secções; que de acordo com a especificação ELF (ver [página 46](https://www.sco.com/developers/devspecs/gabi41.pdf))
são opcionais e nenhum carregador se deve preocupar com isso. Os carregadores utilizam a Vista de Execução e não a Vista de
Vinculação. Implementar a análise de secções apenas por causa deste protocolo é uma sobrecarga insana em qualquer carregador,
onde os recursos do sistema geralmente já são escassos.

Terceiro, os cabeçalhos de secção estão localizados no final do ficheiro. Isto significa que, para detetar o stivale, deve carregar
o início do ficheiro, analisar os cabeçalhos ELF, depois carregar o fim do ficheiro e analisar os cabeçalhos das secções e, em
seguida, carregar algures no meio do ficheiro para obter a lista de etiquetas real. Esta é a pior solução possível. E, mais uma
vez, não há absolutamente nada que indique que um carregador deva fazer isto, pelo que deve fazê-lo para todos os kernels apenas
para descobrir que o kernel não utiliza o stivale. Isto também abranda a deteção de *todos os outros* protocolos de arranque, o
que é inaceitável.

A lista de etiquetas é pesquisada ativamente pelos processadores de aplicações, e o kernel pode chamar o código do carregador de
arranque a qualquer momento, o que significa que simplesmente não pode recuperar a memória do carregador de arranque, caso
contrário, é garantida uma falha. Isto vai contra a filosofia do **Easyboot**.

A pior parte é que o protocolo espera que os carregadores de arranque injetem código em qualquer kernel compatível com o Stivale,
que depois é executado no nível de privilégio mais elevado possível. Sim, o que poderia correr mal, certo?

Como me recuso a fornecer código de baixa qualidade, não haverá suporte para o Stivale no **Easyboot**. E se seguir o meu conselho,
nenhum programador de sistemas operativos amador deveria usá-lo para o seu próprio bem.
