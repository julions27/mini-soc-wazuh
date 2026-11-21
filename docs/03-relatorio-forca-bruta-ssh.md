Relatório de Incidente — Força Bruta SSH

## Resumo executivo

Um invasor(192.168.0.103) tentou acessar via SSH o usuário julio, utilizando o método de força bruta, após a 8º tentativa de senha mal-suscedidada dentro de 120 segundos o bloqueio configurado foi ativado corretamente e o ataque foi parado. IP do atacante desbloqueado após 180 segundos conforme configuração.
## Classificação

- **Tipo de incidente:** Tentativa de acesso não autorizado via ssh pelo método de força bruta
- **Severidade:** Média - tentativa de intrusão mas foi contida automaticamente. SSH exposto na rede 
- **Data e hora da detecção (UTC):** 19:26:36
- **Ativo afetado:** debian-alvo
- **Origem do ataque:** 192.168.0.103
- **Status:** Resolvido

## 1. Preparação

Wazuh all-in-one configurado e vigiando via agente o host chamado debian alvo, que por sua vez coletava via journal os dados. Além disso estava configurado o ruleset padrão do firewall contra ataques de força bruta

## 2. Detecção e Análise

### Como foi detectado

primeiro foi detectado uma falha na senha indicado pelo código 5760 erro que, isolado, não quer dizer muito pode ser apenas que o usuário tenha esquecido sua senha ou digitado errado. Entretanto quando várias senhas são digitadas erradas em um curto espaço de tempo(120 segundos) temos um padrão de ataque identificado pelo código 5763.

### Linha do tempo (UTC)

| Horário (UTC) | Evento                                                    |
| ------------- | --------------------------------------------------------- |
| 19:26:32.209  | Primeira tentativa de login malsucedida (rule 5760)       |
| 19:26:38.193  | Regra de força bruta dispara (rule 5763)                  |
| 19:26:40.185  | Resposta ativa bloqueia o IP (firewall-drop: add)         |
| 19:29:40.243  | Bloqueio removido automaticamente (firewall-drop: delete) |

### Indicadores de Comprometimento (IoCs)

| Indicador               | Valor         |
| ----------------------- | ------------- |
| IP de origem (atacante) | 192.168.0.103 |
| Usuário-alvo            | julio         |
| Porta / serviço         | ssh           |
| Técnica MITRE ATT&CK    | T1110         |

### Análise

O atacante tentou por meio da força bruta, ou seja utilizando várias senhas consideradas comuns num espaço curto de tempo, com o objetivo de acessar o host. O ataque foi mal-sucedido devido a política de firewall do wazuh que, foi implementada por mim diretamente no arquivo ossec.conf, bloqueia automaticamente o IP do atacante após várias tentativas em um espaço muito curto de tempo(8 tentativas seguidas em menos de 120 segundos) 

## 3. Contenção, Erradicação e Recuperação

### Contenção

O Wazuh, via agente no host, criou uma regra DROP no iptables, assim aquele IP específico estava diretamente bloqueado pelo host
### Erradicação e Recuperação

Como estou trabalhando num laboratório então o ataque foi simulado por mim com um outro aparelho, as senhas eram propositalmente erradas e, por isso, não houve algo para erradicar ou recuperar. O acesso legítimo foi restaurado após o timeout padrão

## 4. Atividade Pós-Incidente (Lições Aprendidas)

### O que funcionou

A detecção automática do ruleset padrão funciona dentro do esperado

### Recomendações

1. Desabilitar login do root
2. O firewall banir o IP por um tempo mais longo ou até permanentemente 
3. utilizar chaves ao invés de senhas
4. Reduzir a autoridade de usuários para apenas o necessário, assim diminuindo as possibilidades de comprometer o sistema inteiro
5. Autenticação em 2 etapas

## Evidências

Detecção do ataque de força bruta
![Detecção do ataque](C:\Users\jccar\mini-soc-wazuh\evidencias\incidente-ssh-bruteforce\01-deteccao-falha-login.png)

Adicionado o IP no DROP do iptables via wazuh
![IP Drop Wazuh](C:\Users\jccar\mini-soc-wazuh\evidencias\incidente-ssh-bruteforce\02-active-response-add.png)

![IP Drop iptables](C:\Users\jccar\mini-soc-wazuh\evidencias\incidente-ssh-bruteforce\04-iptables-drop-ativo.png.png)

Tentativa de pingar o IP do host pelo atacante, provando que a conexão foi cortada
![IP Ping Atacante no host](C:\Users\jccar\mini-soc-wazuh\evidencias\incidente-ssh-bruteforce\05-ping-100-perda.png)

Removido o IP do iptables via wazuh
![IP removido iptables via wazuh](C:\Users\jccar\mini-soc-wazuh\evidencias\incidente-ssh-bruteforce\03-active-response-delete.png)
## Referências

https://documentation.wazuh.com/current/user-manual/capabilities/active-response/ar-use-cases/blocking-ssh-brute-force.html
https://csrc.nist.gov/pubs/sp/800/61/r3/final
