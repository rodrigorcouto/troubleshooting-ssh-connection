# RELATÓRIO TÉCNICO DE TROUBLESHOOTING E CORREÇÃO DE CONECTIVIDADE SSH

## 1. Informações Gerais

**Projeto:** Acesso Remoto SSH ao Servidor Ubuntu — Wazuh  
**Data da Atividade:** 07/10/2026  
**Responsável:** Rodrigo Couto  
**Ambiente:** Hyper-V + Ubuntu Server 22.04.5 LTS  
**Servidor:** `wazuh-docker`  
**Objetivo:** Diagnosticar e restabelecer a conectividade SSH entre a estação Windows e o servidor Ubuntu virtualizado.

---

## 2. Resumo Executivo

Durante a tentativa de acesso remoto ao servidor Ubuntu por meio do protocolo SSH, foi identificada uma falha de conexão com o endereço IPv4 inicialmente atribuído ao servidor.

As verificações realizadas demonstraram que o serviço OpenSSH estava instalado, ativo e escutando na porta TCP 22. A infraestrutura de rede também apresentava conectividade com o gateway padrão.

Durante a investigação, foi identificado um **conflito de endereçamento IPv4**, no qual tanto o host Windows quanto a máquina virtual Ubuntu estavam configurados com o endereço **192.168.1.26**.

A duplicidade de endereço impossibilitava o direcionamento correto do tráfego entre o host e a máquina virtual, comprometendo o acesso remoto ao servidor.

Como medida corretiva, a configuração de rede do Ubuntu foi alterada de DHCP para endereço IP estático por meio do Netplan, utilizando o endereço **192.168.1.50/24**, mantendo o gateway **192.168.1.1**.

Após a alteração, foram realizados testes de conectividade e autenticação SSH, ambos concluídos com sucesso.

---

## 3. Objetivo

O objetivo desta atividade foi identificar a causa da indisponibilidade do acesso SSH ao servidor `wazuh-docker` e implementar uma correção permanente para restabelecer a comunicação entre o host Windows e a máquina virtual Ubuntu.

---

## 4. Escopo da Atividade

Foram contempladas as seguintes atividades:

- Diagnóstico da falha de conexão SSH;
- Validação do serviço OpenSSH;
- Verificação da porta TCP 22;
- Análise das interfaces de rede;
- Verificação do endereço IPv4;
- Análise da tabela de roteamento;
- Teste de conectividade com o gateway;
- Teste de conectividade TCP a partir do Windows;
- Investigação de conflito de endereçamento IP;
- Tentativa de renovação do endereço via DHCP;
- Reconfiguração da interface de rede;
- Configuração de endereço IP estático;
- Aplicação da configuração via Netplan;
- Validação da conectividade;
- Validação do acesso SSH.

---

## 5. Caracterização do Ambiente

### 5.1 Host Windows

| Parâmetro | Informação |
|---|---|
| Sistema Operacional | Windows |
| Ambiente de virtualização | Hyper-V |
| Interface | `vEthernet (External-WiFi)` |
| IPv4 inicial | `192.168.1.26` |

### 5.2 Servidor Ubuntu

| Parâmetro | Informação |
|---|---|
| Sistema Operacional | Ubuntu Server 22.04.5 LTS |
| Hostname | `wazuh-docker` |
| Interface | `eth0` |
| IPv4 inicial | `192.168.1.26` |
| Gateway | `192.168.1.1` |

---

## 6. Sintoma Inicial

Ao tentar estabelecer uma sessão SSH a partir do Windows para o servidor Ubuntu, foi utilizado:

```bash
ssh ubuntu@192.168.1.26
```

Foi retornado:

```text
ssh: connect to host 192.168.1.26 port 22:
Connection refused
```

O comportamento indicava uma falha na conexão TCP com o serviço SSH, sendo necessário determinar se a origem estava relacionada ao serviço, à porta, ao firewall ou à infraestrutura de rede.

---

## 7. Hipóteses Consideradas

Durante o troubleshooting foram consideradas as seguintes possibilidades:

1. Serviço SSH não instalado;
2. Serviço SSH parado ou com falha;
3. Firewall bloqueando a porta TCP 22;
4. Serviço SSH configurado em porta diferente da padrão;
5. Problema de conectividade entre host e máquina virtual;
6. Problema de roteamento;
7. Conflito de endereço IP;
8. Problema relacionado à atribuição DHCP.

---

# 8. Diagnóstico

## 8.1 Validação do Serviço SSH

Foi executado no servidor Ubuntu:

```bash
sudo systemctl status ssh
```

Resultado:

```text
Active: active (running)
```

### Conclusão

O serviço OpenSSH estava instalado e em execução.

**Status:** OK

---

## 8.2 Validação da Porta TCP 22

Foi realizada a verificação das portas em escuta:

```bash
sudo ss -tlnp | grep :22
```

Resultado:

```text
LISTEN 0 128 0.0.0.0:22
LISTEN 0 128 [::]:22
```

### Conclusão

O serviço SSH estava escutando na porta TCP 22 em todas as interfaces IPv4 e IPv6.

**Status:** OK

---

## 8.3 Validação do Endereço IP

Foi executado:

```bash
hostname -I
```

Resultado:

```text
192.168.1.26
```

A interface de rede do servidor estava configurada com o endereço `192.168.1.26`.

**Status:** Configuração presente, porém posteriormente identificada como conflitante.

---

## 8.4 Verificação da Tabela de Roteamento

Foi executado:

```bash
ip route
```

Resultado:

```text
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0
```

### Conclusão

A rota padrão e a rede local estavam configuradas corretamente.

- **Gateway:** `192.168.1.1`
- **Rede:** `192.168.1.0/24`

**Status:** OK

---

## 8.5 Teste de Conectividade com o Gateway

Foi executado:

```bash
ping 192.168.1.1
```

O servidor recebeu respostas do gateway:

```text
64 bytes from 192.168.1.1
```

### Conclusão

A interface de rede do Ubuntu apresentava conectividade funcional com o gateway da rede local.

**Status:** OK

---

# 9. Identificação do Problema de Endereçamento

Durante a investigação foi realizado, no host Windows, o seguinte teste:

```powershell
Test-NetConnection 192.168.1.26 -Port 22
```

O resultado apresentou:

```text
SourceAddress    : 192.168.1.26
RemoteAddress    : 192.168.1.26
TcpTestSucceeded : False
```

O resultado revelou uma condição anômala: o endereço IP utilizado como origem da conexão era exatamente o mesmo endereço utilizado como destino.

Posteriormente, os endereços foram confirmados individualmente.

### Host Windows

```powershell
ipconfig
```

Resultado:

```text
IPv4 = 192.168.1.26
```

### Servidor Ubuntu

```bash
ip addr
```

Resultado:

```text
inet 192.168.1.26/24
```

Foi então confirmado o seguinte cenário:

| Equipamento | Endereço IPv4 |
|---|---|
| Host Windows | `192.168.1.26` |
| VM Ubuntu | `192.168.1.26` |

---

# 10. Causa Raiz

## Conflito de Endereço IPv4

A causa raiz da indisponibilidade foi identificada como **duplicidade de endereço IPv4 na mesma rede local**.

O host Windows e a máquina virtual Ubuntu estavam utilizando simultaneamente:

```text
192.168.1.26
```

Essa condição provocava comportamento incorreto no encaminhamento do tráfego, uma vez que o endereço destinado ao servidor também estava atribuído à própria estação de origem.

Consequentemente, não havia uma identificação inequívoca do destino `192.168.1.26` para a máquina virtual.

### Causa raiz confirmada

> **Conflito de IP entre o host Windows e a VM Ubuntu.**

---

# 11. Tentativas de Correção

## 11.1 Renovação do DHCP

Foi inicialmente tentada a renovação da concessão DHCP:

```bash
sudo dhclient -r eth0
sudo dhclient eth0
```

Durante o procedimento foi apresentado:

```text
RTNETLINK answers: File exists
```

Após a tentativa, o endereço permaneceu:

```text
192.168.1.26
```

### Resultado

**Falha na correção.**

O mecanismo DHCP não forneceu um endereço diferente para a interface.

---

## 11.2 Reinicialização da Interface de Rede

Foi realizada a reinicialização da interface:

```bash
sudo ip link set eth0 down
sudo ip link set eth0 up
```

Após a operação, o endereço IP permaneceu inalterado.

### Resultado

**Falha na correção.**

O endereço `192.168.1.26` continuou atribuído à interface.

---

# 12. Solução Implementada

Diante da persistência do conflito, foi adotada uma configuração de endereço IP estático para o servidor Ubuntu.

A configuração foi realizada por meio do **Netplan**, substituindo a utilização de DHCP para a interface `eth0`.

Arquivo de configuração:

```text
/etc/netplan/*.yaml
```

Configuração aplicada:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.1
          - 8.8.8.8
```

### Parâmetros definidos

| Parâmetro | Valor |
|---|---|
| Interface | `eth0` |
| DHCP IPv4 | Desabilitado |
| IP | `192.168.1.50` |
| Máscara | `/24` |
| Gateway | `192.168.1.1` |
| DNS primário | `192.168.1.1` |
| DNS secundário | `8.8.8.8` |

---

# 13. Aplicação da Configuração

Após a alteração do arquivo Netplan, foi executado:

```bash
sudo netplan apply
```

A configuração foi posteriormente validada com:

```bash
ip addr show eth0
```

Resultado:

```text
inet 192.168.1.50/24
```

Também foi realizada a confirmação através de:

```bash
hostname -I
```

Resultado:

```text
192.168.1.50
```

### Resultado

O servidor passou a utilizar exclusivamente o endereço:

```text
192.168.1.50
```

O conflito com o host Windows foi eliminado.

---

# 14. Validação Pós-Correção

## 14.1 Teste de Conectividade

Foi realizado teste de conectividade utilizando o novo endereço:

```bash
ping 192.168.1.50
```

Resultado:

```text
Sucesso
```

A comunicação com o servidor foi restabelecida.

---

## 14.2 Teste de Acesso SSH

Foi realizada uma nova tentativa de conexão:

```bash
ssh ubuntu@192.168.1.50
```

Na primeira conexão, o SSH apresentou a confirmação de autenticidade do host:

```text
The authenticity of host '192.168.1.50' can't be established.

Are you sure you want to continue connecting?
```

Foi selecionada a opção:

```text
yes
```

A conexão foi estabelecida e o sistema apresentou:

```text
Welcome to Ubuntu 22.04.5 LTS
```

### Resultado

**Acesso SSH restabelecido com sucesso.**

---

# 15. Evidências de Sucesso

Após a implementação da correção, foram confirmados os seguintes pontos:

- Endereço IPv4 alterado para `192.168.1.50`;
- Interface `eth0` operacional;
- Gateway `192.168.1.1` mantido;
- Conflito de endereço IPv4 eliminado;
- Comunicação de rede restabelecida;
- Porta TCP 22 acessível;
- Serviço SSH operacional;
- Autenticação SSH realizada com sucesso;
- Acesso remoto ao servidor `wazuh-docker` restabelecido.

---

# 16. Resultado Final

| Item | Resultado |
|---|---|
| OpenSSH instalado | OK |
| Serviço SSH ativo | OK |
| Porta TCP 22 em escuta | OK |
| Interface de rede operacional | OK |
| Gateway configurado | OK |
| Conflito de IP identificado | Confirmado |
| Conflito de IP corrigido | OK |
| IP estático configurado | OK |
| Novo IP do servidor | `192.168.1.50` |
| Teste de conectividade | OK |
| Teste de SSH | OK |
| Acesso remoto | Restabelecido |

---

# 17. Análise Técnica

O troubleshooting demonstrou que a indisponibilidade não estava relacionada ao serviço OpenSSH.

O serviço estava ativo e escutando corretamente na porta TCP 22, e o servidor possuía conectividade com o gateway. A investigação avançou então para a camada de endereçamento, onde foi identificada a duplicidade do endereço `192.168.1.26`.

A evidência apresentada pelo `Test-NetConnection` foi particularmente relevante:

```text
SourceAddress : 192.168.1.26
RemoteAddress : 192.168.1.26
```

Esse resultado, combinado com a confirmação do endereço configurado no Windows e no Ubuntu, permitiu determinar a existência do conflito.

A substituição do DHCP por um endereço estático exclusivo para o servidor eliminou a ambiguidade de endereçamento e restabeleceu a comunicação SSH.

---

# 18. Lições Aprendidas

1. A disponibilidade do serviço SSH não garante que o servidor esteja acessível pela rede.
2. Antes de alterar configurações de firewall ou SSH, deve-se validar o endereçamento IP e a conectividade básica.
3. A análise de `SourceAddress` e `RemoteAddress` no `Test-NetConnection` pode revelar problemas de endereçamento.
4. Ambientes virtualizados devem possuir planejamento adequado de endereçamento IP.
5. Hosts e máquinas virtuais pertencentes à mesma rede devem possuir endereços IPv4 exclusivos.
6. Servidores que hospedam serviços de infraestrutura, como Wazuh, devem utilizar endereçamento previsível e estável.
7. A utilização de IP estático ou reserva DHCP é recomendável para servidores que serão acessados por outros dispositivos.
8. A alteração do endereço deve ser acompanhada de validação de gateway, DNS, rotas e serviços dependentes.

---

# 19. Recomendações

Para evitar recorrência do problema, recomenda-se:

### 19.1 Manter IP exclusivo para o servidor

O endereço `192.168.1.50` deve permanecer reservado para o servidor `wazuh-docker`.

### 19.2 Avaliar reserva DHCP

Caso a infraestrutura utilize DHCP como padrão, uma alternativa é criar uma **reserva DHCP baseada no endereço MAC da VM**, em vez de configurar manualmente o IP no sistema operacional.

### 19.3 Documentar o endereçamento

Manter uma relação dos equipamentos e respectivos endereços IP utilizados na rede de laboratório.

### 19.4 Validar o Hyper-V

Verificar se o adaptador virtual utilizado pela VM está corretamente associado ao switch virtual externo e se não existem configurações que possam provocar conflitos de rede.

### 19.5 Utilizar hostname/DNS

Para um ambiente de laboratório que evoluirá para estudos de Wazuh, Docker, IaC e posteriormente GitOps/Argo CD, recomenda-se utilizar resolução de nomes para os serviços, evitando dependência direta de endereços IP.

---

# 20. Conclusão

O incidente de indisponibilidade do acesso SSH ao servidor `wazuh-docker` foi solucionado após a identificação de um **conflito de endereço IPv4 entre o host Windows e a máquina virtual Ubuntu**.

A investigação descartou inicialmente problemas relacionados ao OpenSSH, à porta TCP 22, ao roteamento e à conectividade com o gateway. Posteriormente, a análise dos endereços de origem e destino confirmou que o host Windows e a VM utilizavam simultaneamente o endereço `192.168.1.26`.

As tentativas de renovação do DHCP e reinicialização da interface não solucionaram o problema. Como medida corretiva, foi implementado um endereço IP estático exclusivo para o servidor Ubuntu, utilizando `192.168.1.50/24`, com gateway `192.168.1.1`.

Após a aplicação da configuração através do Netplan, os testes de conectividade foram concluídos com sucesso e o acesso SSH foi restabelecido.

**Situação final: RESOLVIDA.**

**Endereço final do servidor:** `192.168.1.50`  
**Hostname:** `wazuh-docker`  
**Serviço:** SSH/TCP 22  
**Status:** Operacional
