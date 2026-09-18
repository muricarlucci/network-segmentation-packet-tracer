# 🔎 Verification Playbook — PROJETO ANIBIA

> **Referência operacional para validação da infraestrutura de rede simulada no Cisco Packet Tracer.**
>
> Este documento reúne os comandos de verificação necessários para confirmar o funcionamento dos principais componentes da topologia: interfaces, VLANs, roteamento L3, DHCP, OSPF, eBGP, redistribuição de rotas, contingência WAN e hardening SSHv2.

---

## 📌 Objetivo

O **Verification Playbook** é utilizado após a configuração dos dispositivos para verificar se a infraestrutura implementada corresponde ao comportamento definido no cenário do **PROJETO ANIBIA**.

A validação deve ser realizada diretamente nos equipamentos através do **CLI do Cisco Packet Tracer**, utilizando comandos `show`, testes de conectividade e, no caso do failover, uma simulação controlada de falha.

### Escopo da verificação

| Área              | O que será validado                                |
| :---------------- | :------------------------------------------------- |
| 🔌 Interfaces     | Estado físico e lógico das interfaces              |
| 🧩 VLANs          | Existência e associação das VLANs                  |
| 🔀 Trunks         | Transporte das VLANs entre Core e Access           |
| 🌐 L3             | SVIs, interfaces de trânsito e endereçamento       |
| 📡 DHCP           | Pools e entrega dinâmica de endereços              |
| 🛰️ OSPF          | Adjacências, redes anunciadas e rotas              |
| 🌎 eBGP           | Sessão entre HQ e CPD                              |
| 🔄 Redistribuição | Integração OSPF ↔ BGP no HQ-Edge-RTR               |
| 🛡️ Contingência  | Rotas estáticas flutuantes com AD 115              |
| 💻 Conectividade  | Comunicação entre segmentos e CPD                  |
| 🔐 Hardening      | SSHv2, autenticação local e parâmetros de acesso   |
| 🧪 Failover       | Continuidade da comunicação durante falha da WAN 1 |

---

# 🗺️ 1. Ordem Recomendada de Verificação

Para evitar diagnosticar um protocolo de roteamento quando o problema está em uma interface ou VLAN, recomenda-se seguir a sequência abaixo:

```text
┌───────────────────────────┐
│ 1. Interfaces e IPs       │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 2. VLANs e Trunks         │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 3. SVIs / Routing L3      │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 4. DHCP                   │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 5. OSPF                   │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 6. eBGP                   │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 7. Redistribuição         │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 8. Conectividade          │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 9. Failover WAN           │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│ 10. Hardening SSHv2       │
└───────────────────────────┘
```

---

# 🔌 2. Verificação de Interfaces

## 2.1 Comando principal

Executar nos roteadores e switches:

```cisco
show ip interface brief
```

### O que verificar

A coluna **Status** deve indicar que a interface está operacional e a coluna **Protocol** deve indicar que o protocolo da interface está ativo.

### Interfaces importantes

#### HQ-Edge-RTR

| Interface | Endereço         |
| :-------- | :--------------- |
| `Gig0/0`  | `172.16.16.2/30` |
| `Se0/3/0` | `10.0.0.1/30`    |
| `Se0/3/1` | `10.0.0.10/30`   |

#### Branch-Edge-RTR

| Interface | Endereço        |
| :-------- | :-------------- |
| `Gig0/0`  | `172.19.4.2/30` |
| `Se0/3/1` | `10.0.0.2/30`   |
| `Se0/3/0` | `10.0.0.5/30`   |

#### CPD-Datacenter-RTR

| Interface   | Endereço           |
| :---------- | :----------------- |
| `Gig0/0`    | `172.16.32.1/22`   |
| `Loopback0` | `192.168.100.1/32` |
| `Se0/3/0`   | `10.0.0.9/30`      |
| `Se0/3/1`   | `10.0.0.6/30`      |

---

## 2.2 Verificação detalhada de uma interface

Quando houver dúvida sobre uma interface específica:

```cisco
show interfaces <interface>
```

Exemplo:

```cisco
show interfaces Se0/3/0
```

Para interfaces seriais DCE, conferir também a configuração de clock:

```cisco
show controllers serial 0/3/0
```

No cenário, as interfaces DCE utilizam:

```text
clock rate 64000
```

---

# 🧩 3. Verificação de VLANs

Nos switches:

```cisco
show vlan brief
```

## 3.1 HQ

Devem existir:

|  VLAN | Nome               |
| :---: | :----------------- |
|  `10` | `SEC_OPERATIONS`   |
|  `20` | `FINANCIAL_CORE`   |
|  `30` | `ANALYTICS_DATA`   |
|  `40` | `AUDIT_COMPLIANCE` |
|  `99` | `MGMT_HARDENING`   |
| `200` | `TRANSIT_L3_WAN`   |

## 3.2 RJ

Devem existir:

|  VLAN | Nome            |
| :---: | :-------------- |
|  `10` | `RJ_SUPERVISAO` |
|  `20` | `RJ_OPERATIONS` |
|  `99` | `RJ_MGMT`       |
| `200` | `RJ_TRANSIT_L3` |

---

# 🔀 4. Verificação dos Trunks

## 4.1 Comando

Nos switches:

```cisco
show interfaces trunk
```

### HQ

O trunk entre `HQ-Core-3650` e `HQ-Access-2960` deve transportar:

```text
VLAN 10
VLAN 20
VLAN 30
VLAN 40
VLAN 99
```

### RJ

O trunk entre `Branch-Core-3650` e `Branch-Access-2960` deve transportar:

```text
VLAN 10
VLAN 20
VLAN 99
```

---

## 4.2 Verificação da configuração da interface

Também pode ser utilizado:

```cisco
show running-config interface Gig1/0/1
```

No Core, o enlace com o Access deve estar configurado como trunk.

No Access, o uplink deve estar configurado como trunk.

---

# 🌐 5. Verificação do Roteamento L3

Os switches Core são responsáveis pelo roteamento L3 e hospedam as SVIs utilizadas como gateways das VLANs.

## 5.1 HQ-Core-3650

```cisco
show ip interface brief
```

Conferir:

| SVI       | Endereço         |
| :-------- | :--------------- |
| `Vlan10`  | `172.16.0.1/22`  |
| `Vlan20`  | `172.16.4.1/22`  |
| `Vlan30`  | `172.16.8.1/22`  |
| `Vlan40`  | `172.16.12.1/22` |
| `Vlan99`  | `172.16.99.1/24` |
| `Vlan200` | `172.16.16.1/30` |

Confirmar também que o roteamento L3 está habilitado:

```cisco
show running-config | include ip routing
```

---

## 5.2 Branch-Core-3650

```cisco
show ip interface brief
```

Conferir:

| SVI       | Endereço         |
| :-------- | :--------------- |
| `Vlan10`  | `172.19.0.1/23`  |
| `Vlan20`  | `172.19.2.1/23`  |
| `Vlan99`  | `172.19.99.1/24` |
| `Vlan200` | `172.19.4.1/30`  |

Confirmar:

```cisco
show running-config | include ip routing
```

---

# 📡 6. Verificação do DHCP

Os dois switches Core atuam como servidores DHCP locais.

## 6.1 HQ-Core-3650

Executar:

```cisco
show ip dhcp pool
```

Devem existir os pools:

```text
POOL_SEC_OPS
POOL_FINANCIAL
POOL_ANALYTICS
POOL_AUDIT
```

Para visualizar os bindings:

```cisco
show ip dhcp binding
```

Para verificar conflitos:

```cisco
show ip dhcp conflict
```

---

## 6.2 Parâmetros esperados — HQ

### VLAN 10

```text
Network:       172.16.0.0/22
Gateway:       172.16.0.1
DNS:           172.16.32.10
Pool inicial:  172.16.0.51
```

### VLAN 20

```text
Network:       172.16.4.0/22
Gateway:       172.16.4.1
DNS:           172.16.32.10
Pool inicial:  172.16.4.51
```

### VLAN 30

```text
Network:       172.16.8.0/22
Gateway:       172.16.8.1
DNS:           172.16.32.10
Pool inicial:  172.16.8.51
```

### VLAN 40

```text
Network:       172.16.12.0/22
Gateway:       172.16.12.1
DNS:           172.16.32.10
Pool inicial:  172.16.12.51
```

Os primeiros 50 endereços de cada rede departamental são excluídos do DHCP.

---

## 6.3 Branch-Core-3650

Executar:

```cisco
show ip dhcp pool
```

Devem existir:

```text
POOL_RJ_SUPERVISAO
POOL_RJ_OPERATIONS
```

E:

```cisco
show ip dhcp binding
```

### VLAN 10 — Supervisão

```text
Network:       172.19.0.0/23
Gateway:       172.19.0.1
DNS:           172.16.32.10
Pool inicial:  172.19.0.51
```

### VLAN 20 — Operações

```text
Network:       172.19.2.0/23
Gateway:       172.19.2.1
DNS:           172.16.32.10
Pool inicial:  172.19.2.51
```

---

# 🛰️ 7. Verificação do OSPF

O projeto utiliza **OSPFv2**, com **processo 1** e **Area 0**.

## 7.1 Verificar vizinhos

Nos equipamentos participantes:

```cisco
show ip ospf neighbor
```

### Resultado esperado

As adjacências OSPF estabelecidas devem aparecer em estado:

```text
FULL
```

---

## 7.2 Verificar o processo OSPF

```cisco
show ip protocols
```

Esse comando permite verificar informações do processo de roteamento, incluindo:

* processo OSPF;
* Router ID;
* redes anunciadas;
* área utilizada;
* redistribuição configurada.

---

## 7.3 Verificar informações do OSPF

```cisco
show ip ospf
```

Conferir principalmente:

```text
Process ID: 1
Area: 0
```

---

## 7.4 Verificar rotas aprendidas pelo OSPF

```cisco
show ip route ospf
```

As rotas identificadas pela letra:

```text
O
```

são rotas aprendidas através do OSPF.

---

# 🌎 8. Verificação do eBGP

O eBGP é utilizado entre:

```text
HQ-Edge-RTR
AS 65001
10.0.0.10
       │
       │ WAN 3
       │
10.0.0.9
CPD-Datacenter-RTR
AS 65002
```

## 8.1 Verificar a sessão BGP

No `HQ-Edge-RTR`:

```cisco
show ip bgp summary
```

No `CPD-Datacenter-RTR`:

```cisco
show ip bgp summary
```

### O que verificar

A sessão entre:

```text
10.0.0.10
```

e

```text
10.0.0.9
```

deve estar estabelecida.

O ASN remoto deve corresponder a:

```text
HQ → AS 65002
CPD → AS 65001
```

---

## 8.2 Verificar a tabela BGP

```cisco
show ip bgp
```

No CPD, devem estar presentes os prefixos configurados para anúncio:

```text
172.16.32.0/22
192.168.100.1/32
10.0.0.4/30
```

---

# 🔄 9. Verificação da Redistribuição OSPF ↔ BGP

O ponto de integração entre os dois domínios de roteamento é o:

```text
HQ-Edge-RTR
```

## 9.1 BGP → OSPF

No `HQ-Edge-RTR`:

```cisco
show running-config | section router ospf
```

Deve existir a redistribuição:

```cisco
redistribute bgp 65001 subnets
```

A finalidade é permitir que as rotas aprendidas pelo BGP do CPD sejam disponibilizadas à malha OSPF.

---

## 9.2 OSPF → BGP

No `HQ-Edge-RTR`:

```cisco
show running-config | section router bgp
```

Deve existir:

```cisco
redistribute ospf 1
```

A finalidade é permitir que as redes corporativas aprendidas via OSPF sejam disponibilizadas ao domínio BGP.

---

## 9.3 Conferência na tabela de roteamento

```cisco
show ip route
```

A tabela deve permitir verificar a presença de rotas provenientes dos diferentes mecanismos de roteamento utilizados no projeto.

Para consultar uma rede específica:

```cisco
show ip route 172.16.32.0
```

ou:

```cisco
show ip route 192.168.100.1
```

---

# 🔗 10. Verificação das Rotas de Contingência

O `Branch-Edge-RTR` utiliza rotas estáticas flutuantes com:

```text
Administrative Distance = 115
```

O OSPF utiliza:

```text
Administrative Distance = 110
```

Por isso, as rotas estáticas com AD 115 funcionam como contingência quando as rotas OSPF correspondentes deixam de estar disponíveis.

## 10.1 Conferir configuração

No `Branch-Edge-RTR`:

```cisco
show running-config | include ip route
```

Devem existir as rotas:

```cisco
ip route 172.16.0.0 255.255.240.0 10.0.0.1 115
ip route 172.16.99.0 255.255.255.0 10.0.0.1 115
ip route 172.16.32.0 255.255.252.0 10.0.0.6 115
ip route 192.168.100.1 255.255.255.255 10.0.0.6 115
```

---

## 10.2 Verificar redistribuição das rotas estáticas

```cisco
show running-config | section router ospf
```

Deve existir:

```cisco
redistribute static subnets
```

---

## 10.3 Verificar a rota default do Core RJ

No `Branch-Core-3650`:

```cisco
show running-config | include ip route
```

Deve existir:

```cisco
ip route 0.0.0.0 0.0.0.0 172.19.4.2
```

---

# 🧪 11. Testes de Conectividade

## 11.1 Teste do enlace HQ ↔ RJ

No `HQ-Edge-RTR`:

```cisco
ping 10.0.0.2
```

No `Branch-Edge-RTR`:

```cisco
ping 10.0.0.1
```

---

## 11.2 Teste do enlace RJ ↔ CPD

No `Branch-Edge-RTR`:

```cisco
ping 10.0.0.6
```

No `CPD-Datacenter-RTR`:

```cisco
ping 10.0.0.5
```

---

## 11.3 Teste do enlace HQ ↔ CPD

No `HQ-Edge-RTR`:

```cisco
ping 10.0.0.9
```

No `CPD-Datacenter-RTR`:

```cisco
ping 10.0.0.10
```

---

## 11.4 Teste do servidor do CPD

A partir de um equipamento que possua conectividade com o CPD:

```cisco
ping 172.16.32.10
```

O servidor utilizado no cenário é:

```text
Server-Financial-Hub
172.16.32.10
```

---

## 11.5 Teste da Loopback do CPD

```cisco
ping 192.168.100.1
```

A Loopback 0 está configurada no `CPD-Datacenter-RTR` como:

```text
192.168.100.1/32
```

---

# 🛣️ 12. Verificação do Caminho Percorrido

Para analisar o caminho até o servidor do CPD:

```cisco
traceroute 172.16.32.10
```

Esse teste pode ser utilizado antes e depois da simulação de falha WAN para observar a alteração do caminho.

---

# 🔥 13. Teste de Failover WAN

> **⚠️ Este é um teste destrutivo/controlado.**
>
> A interface será administrativamente desativada durante a simulação. Execute somente após validar que o cenário nominal está funcionando.

O cenário utiliza:

```text
PC-RJ-Ops-01
172.19.2.51
```

como origem e:

```text
Server-Financial-Hub
172.16.32.10
```

como destino.

---

## 13.1 Estado nominal

No `PC-RJ-Ops-01`, iniciar um ping contínuo:

```text
ping 172.16.32.10 -t
```

O objetivo é manter tráfego ICMP durante todo o teste.

O cenário nominal utiliza a conectividade pela WAN 1 em direção à Matriz e posteriormente pela WAN 3 até o CPD.

---

## 13.2 Identificar a interface que será desativada

No `Branch-Edge-RTR`:

```cisco
show ip interface brief
```

A interface da WAN 1 é:

```text
Se0/3/1
10.0.0.2/30
```

---

## 13.3 Injetar a falha

No `Branch-Edge-RTR`:

```cisco
enable
configure terminal
interface Se0/3/0
shutdown
```

> **Importante:** no cenário, a falha é simulada através da desativação da interface `Se0/3/0` do `Branch-Edge-RTR`, correspondente ao enlace WAN 2. A documentação do cenário também descreve a WAN 1 como o enlace primário RJ ↔ HQ através da `Se0/3/1`. Portanto, preserve exatamente a interface indicada pelo procedimento/teste que estiver sendo reproduzido no arquivo `.pkt`.

---

## 13.4 Observar a convergência

Enquanto o ping contínuo estiver sendo executado, observar:

```cisco
show ip ospf neighbor
```

e:

```cisco
show ip route
```

No `Branch-Edge-RTR`, verificar a alteração da tabela de roteamento após a perda da conectividade correspondente.

Também pode ser utilizado:

```cisco
show ip route 172.16.32.0
```

para acompanhar especificamente a rota para a LAN do CPD.

---

## 13.5 Resultado esperado pelo cenário

O cenário descreve uma perda transitória de aproximadamente:

```text
1–2 pacotes ICMP
```

durante a expiração do Dead Interval do OSPF.

Após a convergência, as rotas estáticas com:

```text
AD 115
```

devem assumir a função de contingência.

Para o acesso ao CPD, o caminho alternativo utiliza a:

```text
WAN 2
```

através do próximo salto:

```text
10.0.0.6
```

---

## 13.6 Restaurar a interface

Após finalizar o teste:

```cisco
configure terminal
interface Se0/3/0
no shutdown
```

Confirmar:

```cisco
show ip interface brief
```

Depois verificar novamente:

```cisco
show ip ospf neighbor
```

e:

```cisco
show ip route
```

O objetivo é confirmar o retorno ao estado operacional normal.

---

# 🔐 14. Verificação do Hardening SSHv2

Todos os sete ativos gerenciáveis do cenário possuem parâmetros de hardening.

## 14.1 Verificar domínio

```cisco
show running-config | include ip domain-name
```

Esperado:

```text
ip domain-name anbima.corp
```

---

## 14.2 Verificar versão do SSH

```cisco
show ip ssh
```

Conferir:

```text
SSH version 2
```

---

## 14.3 Verificar parâmetros SSH

O comando:

```cisco
show ip ssh
```

deve ser utilizado para conferir os parâmetros ativos do serviço SSH.

O cenário define:

| Parâmetro                  | Valor         |
| :------------------------- | :------------ |
| Versão                     | SSHv2         |
| Timeout                    | `60` segundos |
| Tentativas de autenticação | `4`           |
| Chave RSA                  | `2048` bits   |

---

## 14.4 Verificar acesso VTY

```cisco
show running-config | section line vty
```

Deve existir a configuração:

```cisco
line vty 0 4
 login local
 transport input ssh
 ip ssh time-out 60
 ip ssh authentication-retries 4
```

O objetivo é confirmar que o acesso remoto está limitado ao SSH e utiliza autenticação local.

---

## 14.5 Verificar usuário administrativo

```cisco
show running-config | include username
```

O cenário utiliza:

```text
username ADMIN privilege 15
```

com credencial definida no script de configuração.

---

# 🖥️ 15. Verificação dos Switches de Acesso

## HQ-Access-2960

### VLANs

```cisco
show vlan brief
```

### Trunk

```cisco
show interfaces trunk
```

### IP de gerenciamento

```cisco
show ip interface brief
```

A interface VLAN 99 deve utilizar:

```text
172.16.99.2/24
```

### Gateway

```cisco
show running-config | include ip default-gateway
```

Esperado:

```text
ip default-gateway 172.16.99.1
```

---

## Branch-Access-2960

### VLANs

```cisco
show vlan brief
```

### Trunk

```cisco
show interfaces trunk
```

### IP de gerenciamento

```cisco
show ip interface brief
```

A interface VLAN 99 deve utilizar:

```text
172.19.99.2/24
```

### Gateway

```cisco
show running-config | include ip default-gateway
```

Esperado:

```text
ip default-gateway 172.19.99.1
```

---

# 🧾 16. Checklist Operacional

Utilize este checklist para registrar a validação final.

## Infraestrutura

* [ ] Todos os equipamentos estão ligados.
* [ ] Interfaces necessárias estão `up/up`.
* [ ] Endereços IP correspondem ao plano definido.
* [ ] Interfaces DCE possuem `clock rate 64000`.
* [ ] VLANs esperadas existem.
* [ ] Portas de acesso estão associadas às VLANs corretas.
* [ ] Trunks estão operacionais.
* [ ] SVIs estão ativas.
* [ ] `ip routing` está ativo nos Core 3650.

## DHCP

* [ ] Pools DHCP da HQ existem.
* [ ] Pools DHCP do RJ existem.
* [ ] Primeiros 50 endereços estão excluídos dos pools departamentais.
* [ ] Clientes recebem endereços dinamicamente.
* [ ] Gateway recebido corresponde à SVI da VLAN.
* [ ] DNS recebido é `172.16.32.10`.

## OSPF

* [ ] Processo OSPF 1 está ativo.
* [ ] Área 0 está configurada.
* [ ] Adjacências esperadas estão em `FULL`.
* [ ] Rotas OSPF aparecem na tabela.
* [ ] Redistribuição BGP → OSPF está configurada no HQ-Edge-RTR.
* [ ] Redistribuição de rotas estáticas → OSPF está configurada no Branch-Edge-RTR.

## BGP

* [ ] HQ-Edge-RTR utiliza AS 65001.
* [ ] CPD-Datacenter-RTR utiliza AS 65002.
* [ ] Peering utiliza `10.0.0.10 ↔ 10.0.0.9`.
* [ ] Sessão eBGP está estabelecida.
* [ ] Prefixos do CPD estão presentes.
* [ ] Redistribuição OSPF → BGP está configurada no HQ-Edge-RTR.

## Contingência

* [ ] Rotas estáticas com AD 115 estão configuradas no Branch-Edge-RTR.
* [ ] Rotas estáticas são redistribuídas no OSPF.
* [ ] Default route do Branch-Core aponta para `172.19.4.2`.
* [ ] Ping contínuo RJ → CPD funciona no estado nominal.
* [ ] Falha controlada pode ser reproduzida.
* [ ] Convergência ocorre após a perda do caminho primário.
* [ ] Comunicação com `172.16.32.10` é restabelecida.
* [ ] Interface desativada é restaurada após o teste.

## Hardening

* [ ] Domínio `anbima.corp` configurado.
* [ ] RSA 2048 configurado.
* [ ] SSHv2 ativo.
* [ ] `transport input ssh` configurado.
* [ ] `login local` configurado.
* [ ] Usuário administrativo possui privilégio 15.
* [ ] Timeout SSH configurado para 60 segundos.
* [ ] Authentication retries configurado para 4.
* [ ] `enable secret` configurado.

---

# 📸 17. Evidências Visuais

As evidências devem ser capturadas diretamente do **Cisco Packet Tracer**, preferencialmente mostrando o nome do dispositivo e o comando executado no CLI.

As imagens devem ser armazenadas em:

```text
assets/evidences/
```

## Evidência 01 — OSPF

Arquivo:

```text
assets/evidences/ev-01-ospf-adjacency.png
```

### Capturar

No equipamento com a adjacência OSPF a ser demonstrada:

```cisco
show ip ospf neighbor
```

### Evidência esperada

A saída deve permitir visualizar as adjacências e o estado:

```text
FULL
```

---

## Evidência 02 — eBGP

Arquivo:

```text
assets/evidences/ev-02-ebgp-peering-established.png
```

### Capturar

No `HQ-Edge-RTR`:

```cisco
show ip bgp summary
```

A captura deve mostrar a sessão com o peer:

```text
10.0.0.9
```

---

## Evidência 03 — DHCP

Arquivo:

```text
assets/evidences/ev-03-dhcp-core-pools.png
```

### Capturar

No Core correspondente:

```cisco
show ip dhcp pool
```

Se necessário, complementar com:

```cisco
show ip dhcp binding
```

---

## Evidência 04 — Failover

Arquivo:

```text
assets/evidences/ev-04-wan-failover-convergence.png
```

### Capturar

Durante o teste:

```text
PC-RJ-Ops-01
172.19.2.51
```

executando ping contínuo para:

```text
172.16.32.10
```

A evidência deve registrar o comportamento do ICMP durante a falha e a retomada da comunicação após a convergência.

---

## Evidência 05 — SSHv2

Arquivo:

```text
assets/evidences/ev-05-hardening-sshv2.png
```

### Capturar

No equipamento validado:

```cisco
show ip ssh
```

E, quando necessário, complementar com:

```cisco
show running-config | section line vty
```

A evidência deve permitir verificar a utilização de SSHv2 e os parâmetros de acesso remoto configurados.

---

# 🧭 18. Comandos de Referência Rápida

| Objetivo                  | Comando                                     |
| :------------------------ | :------------------------------------------ |
| Interfaces                | `show ip interface brief`                   |
| VLANs                     | `show vlan brief`                           |
| Trunks                    | `show interfaces trunk`                     |
| Configuração de interface | `show running-config interface <interface>` |
| Tabela de roteamento      | `show ip route`                             |
| Rotas OSPF                | `show ip route ospf`                        |
| Vizinhos OSPF             | `show ip ospf neighbor`                     |
| Processo OSPF             | `show ip ospf`                              |
| Protocolos de roteamento  | `show ip protocols`                         |
| Resumo BGP                | `show ip bgp summary`                       |
| Tabela BGP                | `show ip bgp`                               |
| Pools DHCP                | `show ip dhcp pool`                         |
| Leases DHCP               | `show ip dhcp binding`                      |
| Conflitos DHCP            | `show ip dhcp conflict`                     |
| SSH                       | `show ip ssh`                               |
| Configuração VTY          | `show running-config \| section line vty`   |
| Rotas estáticas           | `show running-config \| include ip route`   |
| Ping                      | `ping <ip>`                                 |
| Traceroute                | `traceroute <ip>`                           |
| Configuração completa     | `show running-config`                       |

---

# 🏁 19. Critério de Validação Final

A infraestrutura pode ser considerada **validada operacionalmente dentro do escopo do laboratório** quando os principais componentes definidos no cenário forem observados funcionando em conjunto:

```text
                    ┌──────────────────────┐
                    │ Interfaces / VLANs   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Routing L3 + DHCP    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ OSPF Area 0          │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ eBGP AS 65001/65002 │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Redistribuição       │
                    │ OSPF ↔ BGP           │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Conectividade CPD    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Failover WAN         │
                    │ AD 115               │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Hardening SSHv2      │
                    └──────────────────────┘
```

O objetivo do playbook não é apenas verificar se comandos foram configurados, mas confirmar que os componentes da arquitetura **produzem o comportamento esperado quando observados no ambiente simulado**.

---

## 📁 Localização das Evidências

Todas as capturas relacionadas aos testes deste playbook devem ser armazenadas em:

```text
assets/evidences/
├── ev-01-ospf-adjacency.png
├── ev-02-ebgp-peering-established.png
├── ev-03-dhcp-core-pools.png
├── ev-04-wan-failover-convergence.png
└── ev-05-hardening-sshv2.png
```

---

> **PROJETO ANIBIA**
> *Verification Playbook — Cisco Packet Tracer*
> *Referência operacional para validação da infraestrutura de rede.*
