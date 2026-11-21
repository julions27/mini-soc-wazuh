# Playbook — Resposta a Força Bruta SSH

## Gatilho (quando usar este playbook)

Quando encontrar o Alerta de rule.id 5763 no Wazuh

## Classificação e SLA

| Severidade | Média                                                                                                                  | Alta                                                                                  | Crítica                                                                           |
| ---------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Ação       | alguma dezena de IPs com 8 tentativas de login dentro de 120 segundos                                                  | Mais de uma centena de IPs com 8 tentativas de login dentro de 120 segundos           | Acesso Bem sucedido ao usuário alvo                                               |
| Resolução  | Acompanhar para ver se a regra de bloqueio automática ativa após as tentativas e, se possível, alterar usuário e senha | desabilitar login por senha, usar apenas chaves SSH ou colocar o SSH atrás de uma VPN | Imediatamente excluir o Acesso daquele IP utilizando privilégios de administrador |
| Triagem    | Até 30 Minutos                                                                                                         | em até 15 minutos                                                                     | Imediato                                                                          |

## Passo 1 — Triagem (é ataque real ou falso positivo?)

Verificar o seguinte:
1. Quantos IPs? Mais de 1 IP é ponto suspeito
2. Quanto tempo entre as tentativas de acesso? Várias tentativas dentro de 120 segundos é ponto de atenção
3. O IP é conhecido? Se sim, entre em contato com esse conhecido
4. O IP é da rede ou é externo? Caso seja da rede, verifique se aquele host está seguro

## Passo 2 — Investigação

1. Alguma tentativa teve sucesso?  veja se tem o id  5715 no Wazuh
2. Qual usuário alvo?
3. O IP tem algum histórico de ataques anteriores?

## Passo 3 — Contenção

O sistema foi configurado para bloquear IPs com 8 tentativas ou mais dentro de 120 segundos. Caso o Wazuh identifique o ataque mas não ative dentro de 5 segundos o bloqueio deve ser feito o bloqueio de imediato no host acrescentando no DROP da iptable com o comando abaixo:
sudo iptables -I INPUT -s <ENDERECO_IP> -j DROP

Obs: o IP é desbloqueado após 180 segundos. para bloqueio duradouro deve se fazer o comando indicado acima de maneira manual, entretanto o bloqueio manual dura apenas até o próximo reboot

## Passo 4 — Erradicação e Recuperação

Caso todas as tentativas tenham sido bloqueadas:
1. documente o ocorrido detalhando lista de IPs, horários e SSH exposto
2. encerre 

Caso tenha tido sucesso:
1. trocar senha de imediato
2. revisar os acessos para identificar o atacante
3. bloquear aquele utilizando o mesmo comando indicado na contenção

## Passo 5 — Quando escalar para o time pleno/sênior

Caso o IP seja externo e ataca vários usuários ao mesmo tempo, caso o usuário-alvo seja o root com login bem sucedido ou caso o login tenha sido bem sucedido e você não tenha encontrado o atacante mesmo após seguido os passos anteriores dentro de 60 segundos então a situação deve ser passado ao time pleno

## Passo 6 — Encerramento

Registre o ticket, anexe evidências por meio de registro de logs e prints, escreva o relatório do incidente como o disposto no 03-relatorio-forca-bruta-ssh.md.

## Referências

https://documentation.wazuh.com/current/user-manual/capabilities/active-response/ar-use-cases/blocking-ssh-brute-force.html