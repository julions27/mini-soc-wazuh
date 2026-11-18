01 - Instalação do servidor Wazuh

Objetivo



Subir o servidor Wazuh (manager, indexer e dashboard) numa VM. Ele vai ser o centro do mini-SOC e vai receber os eventos das máquinas do lab.



Ambiente

Host: PC com Ryzen 5 4600G, 16 GB de RAM, Windows e VirtualBox <versão>

VM: 4 vCPUs, 8 GB de RAM, 50 GB de disco (VDI dinâmico) e rede em bridge

Sistema: Ubuntu Server 24.04.5 LTS

Wazuh 4.14, instalação all-in-one

Decisões

Ubuntu e não Debian: o Debian não está na lista de sistemas recomendados pela documentação para o servidor Wazuh. O Ubuntu 24.04 está, e é o mais parecido com o Debian.

VM e não Docker: no Windows, o Docker Desktop precisa do WSL 2. O WSL 2 liga a Plataforma de Máquina Virtual, que deixa o VirtualBox lento (ver Problema 1).

Rede em bridge: a VM pega IP na mesma faixa do Debian e dos celulares. Sem isso, os agentes e os "atacantes" não enxergam o servidor.

IP fixo na própria VM: o roteador é da Brisanet e não libera acesso para fazer reserva de DHCP, então o IP foi fixado no Ubuntu. Usei um número alto da faixa para diminuir a chance de conflito com o DHCP do roteador.

Passos

1\. Criar a VM no VirtualBox

Instalar o VirtualBox.

Criar a VM com a ISO ubuntu-24.04.5-live-server-amd64.iso, marcando Skip Unattended Installation, para fazer a instalação manual e ver cada opção.

Configurar 4 vCPUs, 8 GB de RAM e disco de 50 GB dinâmico.

Em Configurações → Rede, colocar o Adaptador 1 em modo Bridge, usando a placa do PC que está conectada na rede.

2\. Instalar o Ubuntu Server

Na tela de rede, configurar o IPv4 como manual (IP fixo IP-DO-WAZUH, gateway e DNS do roteador). Os agentes vão apontar para esse IP; se ele mudar, eles perdem a conexão.

Marcar Install OpenSSH server, porque a VM é headless e vai ser administrada pelo terminal do PC.

Terminar a instalação, reiniciar e remover a ISO.

3\. Preparar o servidor



Acessar via ssh pelo terminal do próprio PC:



bash

ssh usuario@IP-DO-WAZUH



Atualizar o sistema antes de instalar qualquer coisa:



bash

sudo apt update \&\& sudo apt upgrade -y



Verificar se o relógio está sincronizado (System clock synchronized: yes), porque a linha do tempo dos alertas depende do horário certo:



bash

timedatectl



Verificar se o Ubuntu usou todo o disco da VM:



bash

df -h /



Não usou (ver Problema 2).



4\. Instalar o Wazuh



Abrir uma sessão tmux, que independe do ssh. Se a conexão cair no meio da instalação, ela continua rodando e dá para voltar com tmux attach:



bash

tmux



Baixar o instalador oficial:



bash

curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh



Instalar (-a = all-in-one: indexer, servidor e dashboard na mesma máquina):



bash

sudo bash ./wazuh-install.sh -a



No final, o instalador mostra o usuário admin e a senha, que fica guardada fora do repositório. O instalador também gera o wazuh-install-files.tar, com todas as senhas do Wazuh. Esse arquivo fica só no servidor (e está no .gitignore por garantia).



Problemas encontrados

Problema 1: boot travado em "Loading essential drivers"

Sintoma: o boot da ISO parava em Begin: Loading essential drivers ... e aparecia uma tartaruga verde na barra de status da VM.

Causa: a tartaruga indica que há um hipervisor do Windows rodando no host. A Plataforma de Máquina Virtual estava ativada (veio quando instalei o WSL2 para testar), então o VirtualBox não acessava o AMD-V direto e rodava a VM num modo bem mais lento. O Hyper-V já estava desativado.

Solução: desativar a Plataforma de Máquina Virtual em "Ativar ou desativar recursos do Windows" (optionalfeatures) e reiniciar o PC.

Verificação: depois de reiniciar, o boot passou e o instalador do Ubuntu abriu normalmente.

Problema 2: Ubuntu usando só metade do disco

Sintoma: df -h / mostrava \~24 GB, mas o disco da VM tem 50 GB.

Causa: o instalador do Ubuntu Server cria o volume raiz (LVM) com só metade do espaço e deixa o resto livre no grupo de volumes (dá para ver com sudo vgs, na coluna VFree). O Wazuh precisa desse espaço, porque o indexer guarda todos os alertas.

Solução:

bash

sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv



O -l +100%FREE usa todo o espaço livre, e o -r aumenta o sistema de arquivos junto.



Verificação: df -h / passou a mostrar \~48 GB.

Verificação

Dashboard acessível pelo navegador do PC em https://IP-DO-WAZUH, com login pelo usuário admin.

O navegador avisa que o certificado não é confiável. Isso é esperado: o certificado é autoassinado, gerado pelo próprio instalador.

Referências

Wazuh 4.14 – Quickstart: https://documentation.wazuh.com/4.14/quickstart.html

Wazuh 4.14 – Docker deployment (pré-requisitos): https://documentation.wazuh.com/4.14/deployment-options/docker/wazuh-container.html

VirtualBox – Working with Virtual Machines (ícone da tartaruga): https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/working-with-vms.html

Ubuntu Server – How to manage logical volumes: https://ubuntu.com/server/docs/how-to/storage/manage-logical-volumes/

