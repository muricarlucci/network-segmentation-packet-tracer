# 🛠️ Troubleshooting Runbook — PROJETO ANIBIA

> **Guia operacional de diagnóstico e resolução de falhas da infraestrutura de rede simulada no Cisco Packet Tracer.**
>
> Este documento organiza uma sequência prática para identificar, isolar e corrigir problemas de conectividade, comutação, roteamento, DHCP, OSPF, eBGP, redistribuição de rotas, contingência WAN e gerenciamento SSHv2.

---

## 📌 Objetivo

O **Troubleshooting Runbook** deve ser utilizado quando algum componente da infraestrutura apresentar comportamento diferente do esperado.

A metodologia adotada é baseada em **isolamento progressivo da falha**:

```text
Sintoma
   │
   ▼
┌───────────────────────┐
│ 1. Camada física/L1   │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 2. VLAN / Trunk / L2  │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 3. Endereçamento / L3 │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 4. DHCP               │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 5. OSPF               │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 6. eBGP               │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 7. Redistribuição     │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 8. Conectividade      │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ 9. Contingência WAN   │
└───────────────────────┘
```

A regra principal é:

> **Não começar pelo protocolo de roteamento quando ainda não foi comprovado que as interfaces, VLANs, trunks e endereços IP estão funcionando.**

---

# 🧭 1. Diagnóstico Inicial

Antes de alterar qualquer configuração, identificar:

* qual dispositivo apresenta o problema;
* qual origem está iniciando o tráfego;
* qual destino deveria ser alcançado;
* qual segmento de rede está envolvido;
* se o problema é local ou remoto;
* se o problema ocorre sempre ou somente após uma falha;
* se outros dispositivos continuam funcionando normalmente.

---

## 1.1 Registrar origem e destino

Exemplo do fluxo principal utilizado no teste de contingência:

```text
Origem:
PC-RJ-Ops-01
172.19.2.51

        │
        ▼

Destino:
Server-Financial-Hub
172.16.32.10
```

Teste básico:

```text
ping 172.16.32.10
```

---

## 1.2 Verificar o estado geral do dispositivo

```cisco
show ip interface brief
```

Se houver uma interface `administratively down`, `down` ou sem protocolo ativo, corrigir esse ponto antes de avançar para OSPF ou BGP.

---

# 🔌 2. Problemas de Interface

## Sintoma

Um enlace não apresenta conectividade.

### Verificar

```cisco
show ip interface brief
```

Depois:

```cisco
show interfaces <interface>
```

---

## 2.1 Interface administrativamente desligada

### Sintoma

A interface aparece como:

```text
administratively down
```

### Verificação

```cisco
show ip interface brief
```

### Correção

Entrar na interface correspondente:

```cisco
configure terminal
interface <interface>
no shutdown
```

Depois:

```cisco
show ip interface brief
```

---

## 2.2 Interface serial sem clock

No cenário, os enlaces DCE utilizam:

```text
clock rate 64000
```

Topologia WAN:

| Enlace | Interface DCE | Interface DTE |
| :----- | :------------ | :------------ |
| WAN 1  | HQ `Se0/3/0`  | RJ `Se0/3/1`  |
| WAN 2  | RJ `Se0/3/0`  | CPD `Se0/3/1` |
| WAN 3  | CPD `Se0/3/0` | HQ `Se0/3/1`  |

### Verificar

```cisco
show controllers serial 0/3/0
```

ou:

```cisco
show controllers serial 0/3/1
```

### No lado DCE

Confirmar:

```text
clock rate 64000
```

---

## 2.3 Endereço IP incorreto

Verificar:

```cisco
show ip interface brief
```

Comparar com o plano definido:

### WAN 1

```text
HQ: 10.0.0.1/30
RJ: 10.0.0.2/30
```

### WAN 2

```text
RJ: 10.0.0.5/30
CPD: 10.0.0.6/30
```

### WAN 3

```text
CPD: 10.0.0.9/30
HQ: 10.0.0.10/30
```

---

# 🧩 3. Problemas de VLAN

## Sintoma

Um computador não consegue comunicar com outros equipamentos da mesma VLAN ou não recebe DHCP.

### Verificar:

```cisco
show vlan brief
```

---

## 3.1 VLAN inexistente

Confirmar a presença das VLANs correspondentes ao local.

### HQ

```text
10
20
30
40
99
200
```

### RJ

```text
10
20
99
200
```

Se a VLAN necessária não aparecer:

```cisco
show vlan brief
```

e comparar com a configuração:

```cisco
show running-config
```

---

## 3.2 Porta de acesso na VLAN incorreta

No switch de acesso:

```cisco
show vlan brief
```

Verificar se a porta física do computador está associada à VLAN esperada.

### HQ

A configuração do cenário utiliza:

```text
F0/1-2 → VLAN 10
F0/3-4 → VLAN 20
F0/5-6 → VLAN 30
F0/7-8 → VLAN 40
```

### RJ

```text
F0/1-2 → VLAN 10
F0/3-4 → VLAN 20
```

Se necessário:

```cisco
show running-config interface <interface>
```

---

# 🔀 4. Problemas de Trunk

## Sintoma

As VLANs existem nos dois switches, mas os dispositivos conectados ao Access não conseguem alcançar suas respectivas SVIs no Core.

### Verificar:

```cisco
show interfaces trunk
```

---

## 4.1 Trunk não operacional

Confirmar que o uplink está em modo trunk.

```cisco
show running-config interface <interface>
```

A configuração esperada utiliza:

```cisco
switchport mode trunk
```

---

## 4.2 VLAN não permitida no trunk

### HQ

O trunk deve transportar:

```text
10,20,30,40,99
```

### RJ

O trunk deve transportar:

```text
10,20,99
```

Verificar:

```cisco
show interfaces trunk
```

Se uma VLAN necessária não estiver autorizada, conferir a configuração da interface.

---

# 🌐 5. Problemas de SVI e Routing L3

## Sintoma

O computador possui endereço IP, mas não consegue alcançar o gateway.

### Primeiro:

```cisco
show ip interface brief
```

Verificar as SVIs.

---

## 5.1 HQ-Core-3650

SVIs esperadas:

```text
Vlan10  → 172.16.0.1/22
Vlan20  → 172.16.4.1/22
Vlan30  → 172.16.8.1/22
Vlan40  → 172.16.12.1/22
Vlan99  → 172.16.99.1/24
Vlan200 → 172.16.16.1/30
```

---

## 5.2 Branch-Core-3650

```text
Vlan10  → 172.19.0.1/23
Vlan20  → 172.19.2.1/23
Vlan99  → 172.19.99.1/24
Vlan200 → 172.19.4.1/30
```

---

## 5.3 Verificar `ip routing`

Nos Core:

```cisco
show running-config | include ip routing
```

Esperado:

```text
ip routing
```

Sem o roteamento L3 ativo, as SVIs não desempenham a função de roteamento entre redes.

---

# 📡 6. Problemas de DHCP

## Sintoma

O computador recebe endereço `169.254.x.x`, permanece sem configuração adequada ou não recebe um endereço dentro do pool esperado.

---

## 6.1 Verificar os pools

No HQ-Core:

```cisco
show ip dhcp pool
```

Devem existir:

```text
POOL_SEC_OPS
POOL_FINANCIAL
POOL_ANALYTICS
POOL_AUDIT
```

No Branch-Core:

```cisco
show ip dhcp pool
```

Devem existir:

```text
POOL_RJ_SUPERVISAO
POOL_RJ_OPERATIONS
```

---

## 6.2 Verificar bindings

```cisco
show ip dhcp binding
```

Isso permite verificar os endereços entregues aos clientes.

---

## 6.3 Verificar conflitos

```cisco
show ip dhcp conflict
```

Caso existam conflitos, investigar o endereço indicado antes de prosseguir.

---

## 6.4 Conferir exclusões

### HQ

Os primeiros 50 endereços de cada VLAN departamental são excluídos.

Exemplo:

```cisco
show running-config | include excluded-address
```

Esperado:

```text
172.16.0.1 → 172.16.0.50
172.16.4.1 → 172.16.4.50
172.16.8.1 → 172.16.8.50
172.16.12.1 → 172.16.12.50
```

### RJ

```text
172.19.0.1 → 172.19.0.50
172.19.2.1 → 172.19.2.50
```

---

## 6.5 Verificar gateway e DNS

Os pools devem entregar:

```text
Gateway → SVI da própria VLAN
DNS     → 172.16.32.10
```

Exemplo:

```text
VLAN 20 HQ
Gateway: 172.16.4.1
DNS:     172.16.32.10
```

---

# 🛰️ 7. Problemas de OSPF

## Sintoma

Rotas internas não aparecem ou um caminho esperado não é estabelecido.

### Primeiro comando:

```cisco
show ip ospf neighbor
```

---

## 7.1 Vizinho não aparece

Verificar:

```cisco
show ip ospf neighbor
```

Depois:

```cisco
show ip ospf
```

E:

```cisco
show ip protocols
```

Conferir:

* processo OSPF `1`;
* Router ID;
* área `0`;
* redes incluídas no processo;
* interfaces que deveriam participar do OSPF.

---

## 7.2 Verificar redes anunciadas

```cisco
show running-config | section router ospf
```

Comparar as declarações `network` com as redes do equipamento.

### HQ-Core

```text
172.16.0.0/22
172.16.4.0/22
172.16.8.0/22
172.16.12.0/22
172.16.99.0/24
172.16.16.0/30
```

### Branch-Core

```text
172.19.0.0/23
172.19.2.0/23
172.19.99.0/24
172.19.4.0/30
```

---

## 7.3 Verificar rotas OSPF

```cisco
show ip route ospf
```

As rotas aprendidas via OSPF devem aparecer identificadas por:

```text
O
```

---

## 7.4 Verificar Router ID

```cisco
show ip ospf
```

Router IDs definidos no cenário:

| Equipamento      | Router ID |
| :--------------- | :-------- |
| HQ-Core-3650     | `1.1.1.1` |
| HQ-Edge-RTR      | `2.2.2.2` |
| Branch-Edge-RTR  | `3.3.3.3` |
| Branch-Core-3650 | `4.4.4.4` |

---

# 🌎 8. Problemas de eBGP

## Sintoma

O CPD não é alcançável pela rota esperada ou os prefixos não aparecem na tabela BGP.

### Primeiro comando:

No HQ:

```cisco
show ip bgp summary
```

No CPD:

```cisco
show ip bgp summary
```

---

## 8.1 Conferir os ASNs

| Local                     |   ASN   |
| :------------------------ | :-----: |
| HQ / ambiente corporativo | `65001` |
| CPD                       | `65002` |

---

## 8.2 Conferir o peer

A sessão deve utilizar:

```text
HQ-Edge-RTR      10.0.0.10
CPD-Datacenter   10.0.0.9
```

No HQ:

```cisco
show running-config | section router bgp
```

Conferir:

```text
neighbor 10.0.0.9 remote-as 65002
```

No CPD, conferir o vizinho correspondente:

```cisco
show running-config | section router bgp
```

---

## 8.3 Verificar conectividade antes do BGP

Antes de investigar a sessão BGP, testar:

```cisco
ping 10.0.0.9
```

a partir do HQ-Edge-RTR.

No CPD:

```cisco
ping 10.0.0.10
```

Se o endereço do vizinho não estiver alcançável, investigar o enlace WAN 3 antes de alterar o BGP.

---

## 8.4 Verificar tabela BGP

```cisco
show ip bgp
```

No CPD, conferir os prefixos anunciados pelo cenário:

```text
172.16.32.0/22
10.0.0.4/30
192.168.100.1/32
```

---

# 🔄 9. Problemas de Redistribuição

A redistribuição ocorre no `HQ-Edge-RTR`.

---

## 9.1 BGP → OSPF

Verificar:

```cisco
show running-config | section router ospf
```

Deve existir:

```cisco
redistribute bgp 65001 subnets
```

### Sintoma

O BGP possui uma rota, mas os equipamentos internos não a conhecem.

### Diagnóstico

1. Verificar a rota no BGP:

```cisco
show ip bgp
```

2. Verificar a tabela global:

```cisco
show ip route
```

3. Verificar rotas OSPF:

```cisco
show ip route ospf
```

4. Conferir a redistribuição:

```cisco
show running-config | section router ospf
```

---

## 9.2 OSPF → BGP

Verificar:

```cisco
show running-config | section router bgp
```

Deve existir:

```cisco
redistribute ospf 1
```

### Sintoma

A rede corporativa é conhecida pelo OSPF, mas não aparece no domínio BGP.

### Diagnóstico

```cisco
show ip route ospf
```

depois:

```cisco
show ip bgp
```

e:

```cisco
show running-config | section router bgp
```

---

# 🛣️ 10. Problemas com Rotas Estáticas Flutuantes

O `Branch-Edge-RTR` utiliza rotas estáticas com:

```text
Administrative Distance = 115
```

enquanto o OSPF possui:

```text
Administrative Distance = 110
```

A intenção é manter as rotas estáticas como contingência, sendo preferidas somente quando a rota OSPF correspondente deixa de estar disponível.

---

## 10.1 Verificar as rotas

```cisco
show running-config | include ip route
```

Esperado no `Branch-Edge-RTR`:

```cisco
ip route 172.16.0.0 255.255.240.0 10.0.0.1 115
ip route 172.16.99.0 255.255.255.0 10.0.0.1 115
ip route 172.16.32.0 255.255.252.0 10.0.0.6 115
ip route 192.168.100.1 255.255.255.255 10.0.0.6 115
```

---

## 10.2 Verificar qual rota está instalada

```cisco
show ip route 172.16.32.0
```

ou:

```cisco
show ip route 192.168.100.1
```

A saída deve ser interpretada considerando a origem e a distância administrativa da rota instalada.

---

## 10.3 Se a rota flutuante não assumir

Verificar, nesta ordem:

### 1. A rota está configurada?

```cisco
show running-config | include ip route
```

### 2. O próximo salto está alcançável?

Para a rota do CPD:

```cisco
ping 10.0.0.6
```

### 3. A rota OSPF ainda está instalada?

```cisco
show ip route ospf
```

### 4. A rota estática possui AD 115?

```cisco
show ip route 172.16.32.0
```

### 5. As rotas estáticas estão sendo redistribuídas?

```cisco
show running-config | section router ospf
```

Conferir:

```text
redistribute static subnets
```

---

# 🔥 11. Troubleshooting do Failover WAN

## Sintoma

Após a falha do caminho primário, o `PC-RJ-Ops-01` perde completamente a comunicação com o `Server-Financial-Hub`.

---

## 11.1 Confirmar o cenário nominal

Origem:

```text
172.19.2.51
```

Destino:

```text
172.16.32.10
```

Executar:

```text
ping 172.16.32.10 -t
```

Antes de provocar a falha, confirmar que a comunicação estava funcionando.

---

## 11.2 Verificar o OSPF antes da falha

No `Branch-Edge-RTR`:

```cisco
show ip ospf neighbor
```

Depois:

```cisco
show ip route 172.16.32.0
```

Registrar o caminho observado.

---

## 11.3 Verificar a interface após a falha

```cisco
show ip interface brief
```

Confirmar qual interface foi desativada durante o teste.

---

## 11.4 Verificar convergência

```cisco
show ip ospf neighbor
```

e:

```cisco
show ip route
```

Depois:

```cisco
show ip route 172.16.32.0
```

O objetivo é confirmar a mudança da origem da rota conforme o cenário de contingência.

---

## 11.5 Se o tráfego não retornar

Investigar:

```text
1. Interface WAN 2
        ↓
2. Próximo salto 10.0.0.6
        ↓
3. Rota estática AD 115
        ↓
4. Redistribuição static → OSPF
        ↓
5. Rota default do Branch-Core
        ↓
6. Conectividade até 172.16.32.10
```

Testes:

```cisco
ping 10.0.0.6
```

```cisco
show ip route 172.16.32.0
```

```cisco
show running-config | include ip route
```

```cisco
show running-config | section router ospf
```

---

# 🖥️ 12. Problemas entre Core e Edge

## Sintoma

As VLANs funcionam localmente, mas a filial não consegue alcançar redes externas à sua LAN.

### Branch-Core

Verificar:

```cisco
show ip interface brief
```

Conferir:

```text
Vlan200 → 172.19.4.1/30
```

### Branch-Edge

Conferir:

```text
Gig0/0 → 172.19.4.2/30
```

Testar:

```cisco
ping 172.19.4.2
```

a partir do Core.

No roteador:

```cisco
ping 172.19.4.1
```

---

## 12.1 Verificar default route do Branch-Core

```cisco
show running-config | include ip route
```

Esperado:

```cisco
ip route 0.0.0.0 0.0.0.0 172.19.4.2
```

---

# 🔐 13. Problemas de SSH

## Sintoma

O gerenciamento remoto não funciona.

### Verificar:

```cisco
show ip ssh
```

---

## 13.1 Verificar domínio

```cisco
show running-config | include ip domain-name
```

Esperado:

```text
ip domain-name anbima.corp
```

---

## 13.2 Verificar RSA

```cisco
show crypto key mypubkey rsa
```

A configuração do cenário utiliza chave RSA de:

```text
2048 bits
```

---

## 13.3 Verificar VTY

```cisco
show running-config | section line vty
```

Conferir:

```text
login local
transport input ssh
ip ssh time-out 60
ip ssh authentication-retries 4
```

---

## 13.4 Verificar usuário

```cisco
show running-config | include username
```

O usuário configurado no cenário é:

```text
ADMIN
```

com:

```text
privilege 15
```

---

# 🧪 14. Troubleshooting por Sintoma

## ❌ “PC não recebe IP”

Seguir:

```text
PC
 ↓
Porta Access
 ↓
VLAN
 ↓
Trunk
 ↓
SVI
 ↓
DHCP Pool
```

Comandos:

```cisco
show vlan brief
show interfaces trunk
show ip interface brief
show ip dhcp pool
show ip dhcp binding
```

---

## ❌ “PC recebe IP, mas não pinga o gateway”

Seguir:

```text
PC
 ↓
Porta Access
 ↓
VLAN correta
 ↓
Trunk
 ↓
SVI
```

Comandos:

```cisco
show vlan brief
show interfaces trunk
show ip interface brief
```

---

## ❌ “Gateway funciona, mas outra rede não”

Seguir:

```text
SVI
 ↓
ip routing
 ↓
Tabela de rotas
 ↓
OSPF
```

Comandos:

```cisco
show ip interface brief
show running-config | include ip routing
show ip route
show ip route ospf
show ip ospf neighbor
```

---

## ❌ “RJ não alcança CPD”

Seguir:

```text
RJ Core
 ↓
RJ Edge
 ↓
WAN 1 / WAN 2
 ↓
CPD
 ↓
172.16.32.10
```

Comandos:

```cisco
show ip route
show ip ospf neighbor
show ip route 172.16.32.0
ping 10.0.0.6
ping 172.16.32.10
```

---

## ❌ “HQ não alcança CPD”

Seguir:

```text
HQ-Edge
 ↓
WAN 3
 ↓
CPD
 ↓
eBGP
 ↓
172.16.32.0/22
```

Comandos:

```cisco
ping 10.0.0.9
show ip bgp summary
show ip bgp
show ip route 172.16.32.0
```

---

## ❌ “eBGP não estabelece”

Seguir:

```text
Interface WAN 3
 ↓
IP 10.0.0.10 / 10.0.0.9
 ↓
Ping
 ↓
ASN
 ↓
Neighbor
 ↓
BGP
```

Comandos:

```cisco
show ip interface brief
ping 10.0.0.9
show running-config | section router bgp
show ip bgp summary
```

---

## ❌ “Rota do CPD existe no BGP, mas não chega ao Core”

Seguir:

```text
CPD
 ↓
eBGP
 ↓
HQ-Edge
 ↓
BGP → OSPF
 ↓
HQ-Core
```

Comandos:

```cisco
show ip bgp
show ip route
show running-config | section router ospf
show ip route ospf
```

Conferir:

```text
redistribute bgp 65001 subnets
```

---

## ❌ “Failover não funciona”

Seguir:

```text
Falha WAN
 ↓
OSPF perde caminho
 ↓
Rota AD 115
 ↓
Próximo salto
 ↓
Redistribuição
 ↓
Branch-Core
 ↓
CPD
```

Comandos:

```cisco
show ip ospf neighbor
show ip route
show ip route 172.16.32.0
show running-config | include ip route
show running-config | section router ospf
```

---

# 🧰 15. Procedimento de Diagnóstico em 10 Passos

Quando não souber exatamente onde está o problema, executar a sequência abaixo.

### 01 — Interface

```cisco
show ip interface brief
```

### 02 — VLAN

```cisco
show vlan brief
```

### 03 — Trunk

```cisco
show interfaces trunk
```

### 04 — Routing L3

```cisco
show ip route
```

### 05 — DHCP

```cisco
show ip dhcp binding
```

### 06 — OSPF

```cisco
show ip ospf neighbor
```

### 07 — Rotas OSPF

```cisco
show ip route ospf
```

### 08 — BGP

```cisco
show ip bgp summary
```

### 09 — Prefixo específico

```cisco
show ip route <rede-ou-host>
```

### 10 — Teste fim a fim

```cisco
ping <destino>
```

Se necessário:

```cisco
traceroute <destino>
```

---

# 📋 16. Tabela de Diagnóstico Rápido

| Sintoma                     | Primeiro comando          | Próximo ponto                    |
| :-------------------------- | :------------------------ | :------------------------------- |
| Interface não funciona      | `show ip interface brief` | Interface / cabo / `no shutdown` |
| VLAN não funciona           | `show vlan brief`         | VLAN / porta                     |
| VLAN não atravessa switches | `show interfaces trunk`   | Trunk / VLAN permitida           |
| PC sem IP                   | `show ip dhcp pool`       | DHCP / VLAN / SVI                |
| Gateway inacessível         | `show ip interface brief` | SVI / VLAN                       |
| Rede remota inacessível     | `show ip route`           | OSPF / rota                      |
| Vizinho OSPF ausente        | `show ip ospf neighbor`   | Interface / network / OSPF       |
| Rota OSPF ausente           | `show ip route ospf`      | Anúncio / adjacência             |
| eBGP não sobe               | `show ip bgp summary`     | WAN 3 / IP / ASN / neighbor      |
| Prefixo BGP ausente         | `show ip bgp`             | Anúncio / redistribuição         |
| Redistribuição não funciona | `show running-config`     | OSPF / BGP                       |
| Failover não assume         | `show ip route <prefixo>` | AD 115 / próximo salto           |
| SSH não conecta             | `show ip ssh`             | RSA / VTY / usuário              |

---

# 📝 17. Registro de Incidente

Para cada problema encontrado durante o laboratório, registrar:

```text
Data:
Dispositivo:
Origem:
Destino:
Sintoma:
Comando utilizado:
Resultado observado:
Causa identificada:
Alteração realizada:
Comando de validação:
Resultado após correção:
```

---

## Exemplo de registro

```text
Dispositivo:
Branch-Edge-RTR

Origem:
PC-RJ-Ops-01 — 172.19.2.51

Destino:
Server-Financial-Hub — 172.16.32.10

Sintoma:
Perda de conectividade após alteração do estado de uma interface WAN.

Comando utilizado:
show ip route 172.16.32.0

Resultado observado:
Rota instalada diferente do estado nominal.

Causa identificada:
[preencher após diagnóstico]

Alteração realizada:
[preencher]

Comando de validação:
ping 172.16.32.10

Resultado após correção:
[preencher]
```

---

# ⚠️ 18. Boas Práticas Durante o Troubleshooting

### 1. Não alterar várias coisas simultaneamente

Se três configurações forem alteradas ao mesmo tempo, fica difícil determinar qual alteração resolveu ou criou o problema.

---

### 2. Registrar o estado antes da alteração

Sempre que possível:

```cisco
show ip interface brief
show ip route
show ip ospf neighbor
show ip bgp summary
```

antes de modificar a configuração.

---

### 3. Trabalhar do local para o remoto

A sequência recomendada é:

```text
Interface
   ↓
VLAN
   ↓
Trunk
   ↓
SVI
   ↓
Gateway
   ↓
Roteamento
   ↓
WAN
   ↓
Destino
```

---

### 4. Testar depois de cada correção

Depois de corrigir um componente:

```text
Teste local
   ↓
Teste de gateway
   ↓
Teste de rede remota
   ↓
Teste fim a fim
```

---

### 5. Não remover uma configuração sem entender sua função

Isso é especialmente importante para:

* OSPF;
* redistribuição;
* rotas estáticas com AD 115;
* eBGP;
* VLAN 200;
* VLAN 99;
* default route do Branch-Core.

Esses elementos possuem funções específicas na arquitetura do cenário.

---

# 🧭 19. Fluxo Geral de Isolamento

```text
                    PROBLEMA
                       │
                       ▼
             ┌───────────────────┐
             │ Interface UP/UP?  │
             └─────────┬─────────┘
                       │
                 NÃO ──┴── SIM
                 │          │
                 ▼          ▼
             L1 / L2     VLAN correta?
                            │
                      NÃO ──┴── SIM
                      │          │
                      ▼          ▼
                   VLAN/L2     Trunk?
                                  │
                            NÃO ──┴── SIM
                            │          │
                            ▼          ▼
                         Trunk      SVI/Gateway?
                                      │
                                NÃO ──┴── SIM
                                │          │
                                ▼          ▼
                              L3       DHCP?
                                           │
                                     NÃO ──┴── SIM
                                     │          │
                                     ▼          ▼
                                   DHCP      OSPF?
                                                │
                                          NÃO ──┴── SIM
                                          │          │
                                          ▼          ▼
                                        OSPF       BGP?
                                                     │
                                               NÃO ──┴── SIM
                                               │          │
                                               ▼          ▼
                                             eBGP    Redistribuição?
                                                            │
                                                      NÃO ──┴── SIM
                                                      │          │
                                                      ▼          ▼
                                                   Routing    Teste fim a fim
```

---

# 🏁 20. Critério de Encerramento

Um incidente de troubleshooting deve ser considerado encerrado somente após:

* [ ] causa do problema identificada;
* [ ] alteração realizada, quando necessária;
* [ ] comportamento esperado restaurado;
* [ ] teste de conectividade executado;
* [ ] protocolo afetado validado;
* [ ] configuração conferida;
* [ ] resultado registrado;
* [ ] evidência capturada, quando aplicável.

A correção não deve ser considerada concluída apenas porque um único `ping` respondeu.

O objetivo é confirmar que o **componente corrigido e os mecanismos dependentes continuam funcionando em conjunto**.

---

## 📁 Evidências Relacionadas

As evidências visuais dos testes devem permanecer organizadas em:

```text
assets/evidences/
```

com destaque para:

```text
ev-01-ospf-adjacency.png
ev-02-ebgp-peering-established.png
ev-03-dhcp-core-pools.png
ev-04-wan-failover-convergence.png
ev-05-hardening-sshv2.png
```

---

> **PROJETO ANIBIA**
> *Troubleshooting Runbook — Cisco Packet Tracer*
> *Diagnóstico, isolamento e validação de falhas da infraestrutura de rede.*
