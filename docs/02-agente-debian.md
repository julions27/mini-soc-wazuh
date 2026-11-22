# 02 - Instalação do agente Wazuh no Debian

## Objetivo

Instalar o agente Wazuh no Debian (o alvo dos ataques do lab). O agente não gera alertas: ele coleta dados do Debian e manda para o servidor Wazuh (manager), que analisa e gera os alertas.

O agente envia, por exemplo:

- **Inventário de pacotes** instalados, com as versões (é daí que sai a lista de vulnerabilidades)
- **Resultado do SCA**, as checagens de configuração do CIS Benchmark
- **Logs do sistema**, como os logins do SSH (é por eles que a força bruta vai ser detectada)

## Ambiente

- Debian 13.4 (trixie), físico, headless: i3 de 4ª geração, 4 GB de RAM
- Agente Wazuh 4.14.10-1 / Manager Wazuh 4.14.10
- Também instalados no Debian: Docker 29.9.3, Python 3.13.5 e Tailscale 1.102.3 (acesso remoto, é uma porta de entrada para esse alvo)

## Passos

### 1. IP fixo

Editar `/etc/network/interfaces`:

```
auto enp7s0
iface enp7s0 inet static
        address IP-DO-DEBIAN
        netmask 255.255.255.0
        gateway IP-DO-ROTEADOR
```

O Debian precisa de IP fixo porque ele é o alvo: os celulares atacam ele pelo IP, e o IP igual em todos os testes deixa a linha do tempo dos alertas mais fácil de seguir. Aplicar com `reboot` e conferir com `ip -br addr`.

### 2. Instalar o agente pelo repositório oficial

```bash
sudo apt-get update                                   # só atualiza a lista de pacotes, não instala nada
sudo apt-get install gnupg apt-transport-https curl   # pré-requisitos

# chave GPG da Wazuh: o apt confere se cada pacote foi assinado pela Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

# repositório, amarrado só a essa chave (signed-by)
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list

sudo apt-get update
apt-cache policy wazuh-agent    # conferir a versão: o agente precisa ser igual ou mais antigo que o manager

sudo WAZUH_MANAGER="IP-DO-WAZUH" WAZUH_AGENT_NAME="debian-alvo" apt-get install wazuh-agent

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent    # inicia sozinho em todo boot
sudo systemctl start wazuh-agent

# travar o repositório: um apt upgrade futuro não deixa o agente mais novo que o manager
sudo sed -i "s/^deb /#deb/" /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```

**Por que não rodei `apt upgrade` antes:** para o Wazuh registrar a **baseline** das vulnerabilidades (o estado antes da correção). Sem a baseline, não dá para provar depois que a correção funcionou.

## Problemas encontrados

### Problema 1: rede não subiu depois do IP fixo

- **Sintoma:** o Debian ficou sem rede depois de editar `/etc/network/interfaces`.
- **Causa:** erro de digitação: `adress` em vez de `address`. Sem o `address`, a placa não tinha IP para subir. Confirmado no log com `journalctl -u networking -b`.
- **Solução:** corrigir para `address` e reiniciar.
- **Verificação:** `ip -br addr` mostrou o IP fixo.

### Problema 2: sem DNS depois do IP fixo

- **Sintoma:** `apt-get update` falhava com "Falha temporária resolvendo 'deb.debian.org'", mas `ping 1.1.1.1` funcionava.
- **Causa:** ao trocar o IP automático por fixo, o `dhcpcd` liberou o IP e removeu os servidores DNS do `/etc/resolv.conf`. A linha `dns-nameservers` não teve efeito porque o pacote `resolvconf` não está instalado. Pista: a data do arquivo era anterior ao reboot.
- **Solução:** colocar os `nameserver` direto no `/etc/resolv.conf`.
- **Verificação:** `ping -c 3 deb.debian.org` respondeu.

### Problema 3: "Permissão negada" no gpg e no tee

- **Sintoma:** `gpg` e `tee` retornaram "Permissão negada", mesmo com `sudo` no comando.
- **Causa:** o `sudo` vale só para o programa logo depois dele. Com `|`, são programas separados, e o `sudo` tinha ido para o `curl` e o `echo`, não para quem gravava o arquivo.
- **Solução:** colocar o `sudo` no programa que grava na pasta do root: `| sudo gpg ...` e `| sudo tee ...`.
- **Verificação:** a chave apareceu em `ls -l /usr/share/keyrings/wazuh.gpg`.

### Problema 4: IP do manager digitado errado

- **Sintoma:** ao conferir o `ossec.conf` antes de ligar o serviço, o endereço estava `198.168.0.200`.
- **Causa:** erro de digitação na variável `WAZUH_MANAGER`. `198.168.x.x` é um IP **público**: o agente tentaria mandar os dados do Debian para um servidor na internet.
- **Solução:** corrigir o `<address>` em `/var/ossec/etc/ossec.conf` **antes** de iniciar o agente.
- **Verificação:** `sudo grep -n "<address>" /var/ossec/etc/ossec.conf` mostrou o IP certo.

## Verificação

- `systemctl status wazuh-agent`: `active (running)`
- `debian-alvo` aparece como **Active** no dashboard
- SCA (CIS Debian 13) rodou: **Score 38%** (73 aprovadas, 115 reprovadas, 19 não aplicáveis). Baseline em `evidencias/sca-hardening/baseline-instalacao.csv`
- Detecção de vulnerabilidades rodou. Baseline em `evidencias/vulnerabilidades/antes-dos-patches.png`

## Referências

- Wazuh 4.14 – Agente em Linux: https://documentation.wazuh.com/4.14/installation-guide/wazuh-agent/wazuh-agent-package-linux.html
- Wazuh 4.14 – Políticas SCA disponíveis: https://documentation.wazuh.com/4.14/user-manual/capabilities/sec-config-assessment/available-sca-policies.html
- Debian Wiki – resolv.conf: https://wiki.debian.org/resolv.conf
