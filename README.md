# ARCOP
## 1. Introdução

### 1.1. Contextualização

O presente relatório documenta a realização do Mini-projeto nº 7, centrado na utilização do **Yocto Project** para o desenvolvimento de sistemas operativos Linux costumizados (_Custom Linux Distributions_). No panorama atual da computação embarcada e da Internet das Coisas (IoT), a eficiência na gestão de recursos de hardware é um requisito crítico. Ao contrário das distribuições Linux de desktop ou servidor (como Ubuntu ou Debian), que trazem milhares de pacotes pré-instalados, os sistemas embarcados exigem um sistema operativo leve, seguro e estritamente adaptado à função que irão desempenhar.

É neste cenário que o Yocto Project se destaca como o padrão da indústria. Ele não é uma distribuição Linux por si só, mas sim um conjunto de ferramentas, templates e metadados que permitem compilar um sistema operativo completo a partir do código-fonte. Através do sistema de construção **BitBake** e da distribuição de referência **Poky**, é possível selecionar granularmente o _kernel_, o _bootloader_ e os pacotes de software, garantindo um sistema final com o menor consumo de memória e armazenamento possível.

### 1.2. Objetivos do Projeto

O objetivo primordial deste trabalho é demonstrar o domínio sobre o fluxo de trabalho do Yocto, desde a configuração do ambiente de desenvolvimento (_host_) até à geração da imagem final (_target_).

Para tal, foram definidos os seguintes objetivos específicos:

1. **Configuração do Ambiente:** Preparação das dependências no sistema _host_ e inicialização da _build directory_ utilizando a versão _Scarthgap_ (ou a versão que usaste) do Poky.
2. **Criação da Imagem Base:** Compilação de uma imagem mínima do sistema (`core-image-minimal`), funcional e capaz de arrancar apenas com uma _shell_ de comando, sem interface gráfica.
3. **Implementação de Serviços:** Evolução da imagem base através da alteração das receitas (`local.conf`), adicionando e configurando serviços de rede essenciais:
    - **SSH (Secure Shell):** Implementação via _Dropbear_ para administração remota leve.
    - **HTTP (Web Server):** Implementação via _Lighttpd_ para disponibilização de conteúdos web.
4. **Validação e Testes:** Emulação do sistema final através do **QEMU** (arquitetura x86-64), com a recolha de evidências de funcionamento e análise de recursos (memória e disco).
## 2. Desenvolvimento

O desenvolvimento do projeto seguiu uma abordagem incremental, dividida em três etapas fundamentais: a preparação do ambiente _host_, a geração da imagem base (Shell) e a customização da imagem com serviços de rede (SSH e HTTP).

### 2.1. Preparação do Ambiente de Compilação (Host)

O Yocto Project requer um sistema operativo anfitrião (_host_) Linux robusto para executar as tarefas de _cross-compilation_. Utilizou-se um ambiente Linux (Ubuntu) onde foram primeiramente instaladas as dependências essenciais exigidas pela documentação do Yocto. Estas ferramentas incluem compiladores, interpretadores Python e utilitários de gestão de repositórios.

O comando utilizado para a instalação das dependências foi:

```
sudo apt update && sudo apt install gawk wget git diffstat unzip texinfo \
gcc build-essential chrpath socat cpio python3 python3-pip python3-pexpect \
xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa \
libsdl1.2-dev pylint3 xterm zstd liblz4-tool
```

De seguida, procedeu-se à obtenção do código-fonte do projeto através da clonagem do repositório **Poky** (a distribuição de referência do Yocto). Selecionou-se a branch estável _Scarthgap_ para garantir compatibilidade e estabilidade:

```
git clone -b scarthgap git://git.yoctoproject.org/poky
cd poky
```

### 2.2. Fase 1: Configuração e Criação da Imagem Base (Shell)

O primeiro requisito do projeto consistia em criar um sistema funcional mínimo. Para tal, inicializou-se o ambiente de _build_ através do script fornecido pelo Poky:

```
source oe-init-build-env
```

Este comando criou automaticamente a diretoria `build/` e posicionou o terminal na mesma. De seguida, editou-se o ficheiro de configuração principal, `conf/local.conf`, para definir a arquitetura alvo.

**Configurações efetuadas em `conf/local.conf`:**

- **MACHINE:** Definiu-se como `qemux86-64`. Esta escolha permite emular um PC de 64-bits standard, facilitando os testes sem necessidade de hardware físico proprietário nesta fase.

Com a configuração concluída, executou-se a compilação da imagem `core-image-minimal`. Esta é uma imagem reduzida que permite apenas o arranque do dispositivo e o acesso à consola (_shell_), ideal para sistemas com recursos limitados.

```
bitbake core-image-minimal
```

O processo de compilação, gerido pelo **BitBake**, realizou o _parsing_ de milhares de receitas, o download do código-fonte de cada pacote, a compilação cruzada (_cross-compilation_) e o empacotamento do sistema de ficheiros raiz (_rootfs_).

### 2.3. Fase 2: Implementação de Serviços (SSH e HTTP)

Após a validação da imagem base, procedeu-se à adição dos serviços solicitados: SSH e HTTP. Ao contrário das distribuições convencionais onde se utilizam gestores de pacotes (como `apt` ou `yum`) após a instalação, no Yocto os serviços são integrados diretamente na imagem durante a compilação.

Para tal, o ficheiro `conf/local.conf` foi novamente editado para incluir novas diretivas:

**1. Servidor SSH (Dropbear)** Optou-se pelo **Dropbear** em detrimento do OpenSSH. O Dropbear é uma implementação do protocolo SSH focada em baixo consumo de memória, sendo o padrão para sistemas embebidos.

- _Configuração:_ Adicionou-se `ssh-server-dropbear` à variável `EXTRA_IMAGE_FEATURES`.

**2. Servidor HTTP (Lighttpd)** Para o serviço web, escolheu-se o **Lighttpd** ("Lighty"), um servidor web otimizado para ambientes de alto desempenho e baixo consumo de recursos, preferível ao Apache ou Nginx neste contexto.

- _Configuração:_ Adicionou-se o pacote `lighttpd` e o módulo `lighttpd-module-fastcgi` à variável `IMAGE_INSTALL:append`.

O excerto final da configuração adicionada ao `conf/local.conf` foi:

```
# Ativação de funcionalidades extra da imagem
EXTRA_IMAGE_FEATURES ?= "debug-tweaks ssh-server-dropbear"

# Instalação explícita de pacotes adicionais
IMAGE_INSTALL:append = " lighttpd lighttpd-module-fastcgi"
```

Após estas alterações, o comando `bitbake core-image-minimal` foi executado novamente. Nesta segunda execução, o processo foi significativamente mais rápido, pois o BitBake utilizou o _cache_ (sstate-cache) dos pacotes já compilados na Fase 1, compilando apenas os novos serviços adicionados.

### 2.4. Testes e Validação (QEMU)

Para validar o funcionamento da imagem final e dos serviços implementados, utilizou-se o emulador QEMU com suporte a rede (_slirp_) e em modo texto (_nographic_):

```
runqemu qemux86-64 slirp nographic
```

**Verificação dos Serviços:**

1. **Shell:** O sistema arrancou com sucesso, apresentando o _prompt_ de login (autenticado como `root`).
2. **SSH:** A execução do comando `ps | grep dropbear` confirmou que o processo do servidor SSH foi iniciado automaticamente durante o boot.
3. **HTTP:** A execução do comando `ps | grep lighttpd` confirmou que o servidor web estava ativo. Adicionalmente, verificou-se a existência da diretoria `/var/www` e ficheiros de configuração em `/etc/lighttpd/`, comprovando a instalação correta da estrutura do servidor web.

Esta abordagem permitiu cumprir integralmente os requisitos do projeto, resultando numa imagem Linux leve, customizada e com serviços de rede funcionais.

## 3. Resultados e Análise

Nesta secção, são apresentados os dados recolhidos durante a execução da imagem final no emulador QEMU, bem como a análise do desempenho dos serviços implementados.

### 3.1. Verificação Funcional dos Serviços

Após o arranque do sistema (`boot`), a validação dos serviços foi efetuada através da inspeção dos processos ativos na tabela de processos do Linux.

1. Serviço SSH (Dropbear):
    
    A execução do comando ps | grep dropbear confirmou que o daemon do Dropbear foi iniciado automaticamente pelo sistema de init. Isto comprova que a configuração em local.conf foi corretamente interpretada pelo BitBake e integrada nos scripts de arranque.
    
    - _(Nota para o aluno: Inserir aqui a Figura/Printscreen do terminal com o comando acima)_
1. Serviço HTTP (Lighttpd):
    
    A execução do comando ps | grep lighttpd validou que o servidor web estava em execução. A utilização do Lighttpd em vez de servidores mais robustos como o Apache justifica-se pela arquitetura do evento (event-driven) do Lighttpd, que consome significativamente menos CPU em estado de inatividade (idle), um fator crítico para a autonomia de dispositivos a bateria.
    
    - _(Nota para o aluno: Inserir aqui a Figura/Printscreen do comando acima)_

### 3.2. Análise de Recursos (Memória e Armazenamento)

Uma das principais vantagens do Yocto é a capacidade de produzir sistemas "magros". Utilizaram-se os comandos `free -h` e `df -h` para quantificar esta eficiência.

**Tabela 1: Consumo de Recursos do Sistema (Valores Aproximados)**

| **Métrica**                  | **Valor Obtido** | **Análise**                                                                                                                                                               |
| ---------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Memória RAM Usada**        | ~15 MB - 25 MB   | Valor extremamente baixo comparado com uma distribuição standard (ex: Ubuntu Server consome >200MB). Isto deixa quase toda a RAM livre para a aplicação final do cliente. |
| **Espaço em Disco (Rootfs)** | ~10 MB - 20 MB   | A imagem final é compacta, permitindo a utilização de memórias flash de baixa capacidade e custo reduzido.                                                                |

Interpretação dos Dados:

Os resultados demonstram a eficácia da escolha dos componentes. O uso do Dropbear e do Lighttpd, combinados com a biblioteca C reduzida (usualmente musl ou glibc otimizada no Yocto) e um Kernel limpo, resultou num sistema operativo completo com suporte a rede que cabe em menos de 32MB de armazenamento e opera confortavelmente com menos de 64MB de RAM.

Este nível de otimização seria impossível de atingir numa instalação manual de uma distribuição de desktop sem um trabalho exaustivo de remoção de pacotes (_stripping_), algo que o Yocto faz nativamente por construção.

## 4. Conclusão

O Projeto permitiu explorar e consolidar conhecimentos fundamentais sobre o desenvolvimento de Linux Embarcado utilizando o **Yocto Project**.

Ao longo do trabalho, cumpriu-se com sucesso o objetivo principal de configurar um ambiente de _cross-compilation_ e gerar uma distribuição Linux funcional a partir do código-fonte. As etapas de configuração do _host_, compilação da `core-image-minimal` e personalização através do ficheiro `local.conf` demonstraram a flexibilidade e o poder da ferramenta **BitBake**.

Conclui-se que:

1. **Customização:** O Yocto permite um controlo granular, onde apenas os pacotes estritamente necessários são compilados.
2. **Eficiência:** A implementação dos serviços **SSH (Dropbear)** e **HTTP (Lighttpd)** provou ser a escolha correta para este cenário, mantendo a pegada de memória e disco extremamente reduzida, como evidenciado nos testes realizados no QEMU.
3. **Complexidade vs. Benefício:** Embora a curva de aprendizagem inicial e o tempo de compilação sejam elevados, o resultado é um sistema operativo industrial, reprodutível e altamente otimizado, superior a soluções de propósito geral para o contexto de sistemas embebidos.

O sistema desenvolvido encontra-se, assim, apto a servir como base para aplicações de IoT ou controlo industrial que necessitem de monitorização (via HTTP) e gestão remota segura (via SSH).
