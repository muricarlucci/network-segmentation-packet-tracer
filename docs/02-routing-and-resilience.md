# Routing and Resilience — ANBIMA Financial Hub

[![Routing: OSPFv2](https://img.shields.io/badge/Routing-OSPFv2-blue)](#-1-roteamento-interno-ospfv2)
[![External Routing: eBGPv4](https://img.shields.io/badge/External%20Routing-eBGPv4-orange)](#-2-roteamento-exterior-ebgpv4)
[![Autonomous Systems: AS 65001 ↔ 65002](https://img.shields.io/badge/AS-65001%20%E2%86%94%2065002-purple)](#-2-roteamento-exterior-ebgpv4)
[![Resilience: WAN Ring](https://img.shields.io/badge/Resilience-WAN%20Ring-green)](#-4-malha-wan-serial-em-anel)

Documentação da arquitetura de roteamento e dos mecanismos de resiliência da infraestrutura corporativa simulada da ANBIMA. O projeto utiliza um modelo híbrido multidomínio, combinando **OSPFv2 como IGP interno**, **eBGPv4 como EGP entre os Sistemas Autônomos** e **rotas estáticas flutuantes** como mecanismo adicional de contingência.

A infraestrutura WAN é organizada em uma topologia de **anel fechado**, conectando Matriz SP, Filial Regional RJ e Datacenter/CPD. Essa estrutura permite utilizar caminhos alternativos quando o enlace primário entre a Matriz e a Filial sofre interrupção.

---

# 🔀 1. Roteamento Interno — OSPFv2

O protocolo **OSPFv2** é utilizado como protocolo de roteamento interno da infraestrutura, operando na **Área 0 (Backbone)**.

O protocolo é executado nos **Roteadores de Borda** e nos **Switches Core** da Matriz e da Filial Regional RJ.

### Características documentadas

| Característica | Implementação                                  |
| :------------- | :--------------------------------------------- |
| Protocolo      | OSPFv2                                         |
| Área           | Area 0                                         |
| Algoritmo      | Shortest Path First (Dijkstra)                 |
| Métrica        | Custo associado à largura de banda dos enlaces |
| Matriz         | HQ-Edge-RTR + HQ-Core-3650                     |
| Filial RJ      | Branch-Edge-RTR + Branch-Core-3650             |
| Processo OSPF  | `1`                                            |

O objetivo do OSPF é permitir que os equipamentos internos conheçam as redes corporativas sem depender do protocolo BGP dentro dos switches Core.

---

## 🆔 1.1 Router IDs

Cada equipamento participante do OSPF possui um identificador próprio:

| Equipamento          | Router ID |
| :------------------- | :-------- |
| **HQ-Core-3650**     | `1.1.1.1` |
| **HQ-Edge-RTR**      | `2.2.2.2` |
| **Branch-Edge-RTR**  | `3.3.3.3` |
| **Branch-Core-3650** | `4.4.4.4` |

---

## 🧮 1.2 Wildcard Masks utilizadas

As redes anunciadas pelo OSPF utilizam máscaras coringa correspondentes aos respectivos prefixos:

| Prefixo | Máscara Decimal   | Wildcard    |
| :------ | :---------------- | :---------- |
| `/22`   | `255.255.252.0`   | `0.0.3.255` |
| `/23`   | `255.255.254.0`   | `0.0.1.255` |
| `/24`   | `255.255.255.0`   | `0.0.0.255` |
| `/30`   | `255.255.255.252` | `0.0.0.3`   |

---

# 🏢 1.3 OSPF na Matriz SP

O `HQ-Core-3650` participa do OSPF anunciando as redes internas da Matriz e o enlace de trânsito L3 com o roteador de borda.

### Redes anunciadas pelo HQ-Core-3650

```text
172.16.0.0 0.0.3.255 area 0
172.16.4.0 0.0.3.255 area 0
172.16.8.0 0.0.3.255 area 0
172.16.12.0 0.0.3.255 area 0
172.16.99.0 0.0.0.255 area 0
172.16.16.0 0.0.0.3 area 0
```

O `HQ-Edge-RTR` participa do mesmo processo OSPF e anuncia:

```text
172.16.16.0 0.0.0.3 area 0
10.0.0.0 0.0.0.3 area 0
```

Além disso, o roteador de borda realiza a redistribuição das rotas BGP para dentro do OSPF.

---

# 🏢 1.4 OSPF na Filial Regional RJ

O `Branch-Core-3650` anuncia suas redes locais e o enlace de trânsito L3 com o roteador regional.

### Redes anunciadas pelo Branch-Core-3650

```text
172.19.0.0 0.0.1.255 area 0
172.19.2.0 0.0.1.255 area 0
172.19.99.0 0.0.0.255 area 0
172.19.4.0 0.0.0.3 area 0
```

O `Branch-Edge-RTR` participa do OSPF através das redes:

```text
172.19.4.0 0.0.0.3 area 0
10.0.0.0 0.0.0.3 area 0
10.0.0.4 0.0.0.3 area 0
```

O roteador regional também redistribui suas rotas estáticas para o OSPF.

---

# 🌐 2. Roteamento Exterior — eBGPv4

O projeto utiliza **eBGPv4** para estabelecer a comunicação entre os dois Sistemas Autônomos presentes na arquitetura.

```text
┌─────────────────────────────────────┐
│ Sistema Autônomo ANBIMA             │
│ AS 65001                             │
│                                     │
│ Matriz SP + Filial RJ               │
└──────────────────┬──────────────────┘
                   │
                   │ eBGP
                   │ TCP/179
                   │
┌──────────────────▼──────────────────┐
│ Datacenter / CPD                     │
│ AS 65002                             │
└─────────────────────────────────────┘
```

### Divisão dos Sistemas Autônomos

| Sistema Autônomo | Ambiente                           |
| :--------------- | :--------------------------------- |
| **AS 65001**     | Matriz ANBIMA + Filial Regional RJ |
| **AS 65002**     | Datacenter de Contingência / CPD   |

---

# 🔗 2.1 Sessão eBGP entre Matriz e CPD

O peering externo ocorre através da **WAN 3**, diretamente entre:

| Equipamento            | Interface | Endereço    | AS      |
| :--------------------- | :-------- | :---------- | :------ |
| **HQ-Edge-RTR**        | `Se0/3/1` | `10.0.0.10` | `65001` |
| **CPD-Datacenter-RTR** | `Se0/3/0` | `10.0.0.9`  | `65002` |

A sessão eBGP utiliza **TCP na porta 179**.

No `HQ-Edge-RTR`:

```text
router bgp 65001
 bgp log-neighbor-changes
 neighbor 10.0.0.9 remote-as 65002
 redistribute ospf 1
```

No `CPD-Datacenter-RTR`:

```text
router bgp 65002
 bgp log-neighbor-changes
 neighbor 10.0.0.10 remote-as 65001
 network 172.16.32.0 mask 255.255.252.0
 network 192.168.100.1 mask 255.255.255.255
 network 10.0.0.4 mask 255.255.255.252
```

---

# 📢 2.2 Anúncios originados pelo Datacenter

O CPD anuncia para a Matriz:

| Prefixo            | Finalidade                               |
| :----------------- | :--------------------------------------- |
| `172.16.32.0/22`   | Bloco local de servidores                |
| `10.0.0.4/30`      | Enlace WAN 2                             |
| `192.168.100.1/32` | Loopback 0 / serviço de teste de peering |

A `Loopback0` é utilizada para representar um serviço corporativo lógico que não depende da disponibilidade física de uma interface Ethernet ou serial.

---

# 🔄 3. Redistribuição Mútua de Rotas

O ponto de integração entre os domínios de roteamento ocorre no **HQ-Edge-RTR**.

O roteador de borda funciona como elemento de conversão entre:

```text
OSPFv2
   │
   │ Rotas internas
   ▼
HQ-Edge-RTR
   │
   │ Redistribuição
   ▼
eBGPv4
```

E no sentido inverso:

```text
eBGPv4
   │
   │ Rotas do CPD
   ▼
HQ-Edge-RTR
   │
   │ Redistribuição
   ▼
OSPFv2
   │
   ▼
HQ-Core-3650
```

---

## 3.1 BGP → OSPF

As rotas aprendidas via BGP do Datacenter são injetadas no OSPF através de:

```text
redistribute bgp 65001 subnets
```

Com isso, o `HQ-Core-3650` pode conhecer as redes do CPD sem executar BGP internamente.

---

## 3.2 OSPF → BGP

As rotas das VLANs da Matriz aprendidas via OSPF são injetadas no BGP através de:

```text
redistribute ospf 1
```

Essas rotas são então disponibilizadas ao Datacenter através da sessão eBGP.

---

# 🔁 4. Malha WAN Serial em Anel

A conectividade entre os três sítios utiliza uma **topologia WAN serial em anel fechado**.

```text
                    ┌───────────────────────┐
                    │ CPD / Datacenter      │
                    │ CPD-Datacenter-RTR    │
                    └───────────┬───────────┘
                         WAN 2  │  WAN 3
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
┌─────────────────────────┐             ┌─────────────────────────┐
│ Filial Regional RJ      │             │ Matriz SP               │
│ Branch-Edge-RTR         │──── WAN 1 ──│ HQ-Edge-RTR              │
└─────────────────────────┘             └─────────────────────────┘
```

A malha utiliza módulos **HWIC-2T instalados no Slot 3** dos roteadores.

---

## 4.1 WAN 1 — Matriz ↔ Filial RJ

```text
10.0.0.0/30
```

| Ponta     | Interface | IP         | Papel |
| :-------- | :-------- | :--------- | :---- |
| Matriz SP | `Se0/3/0` | `10.0.0.1` | DCE   |
| Filial RJ | `Se0/3/1` | `10.0.0.2` | DTE   |

No lado DCE:

```text
clock rate 64000
```

A WAN 1 representa a ligação primária entre Matriz e Filial Regional.

---

## 4.2 WAN 2 — Filial RJ ↔ CPD

```text
10.0.0.4/30
```

| Ponta     | Interface | IP         | Papel |
| :-------- | :-------- | :--------- | :---- |
| Filial RJ | `Se0/3/0` | `10.0.0.5` | DCE   |
| CPD       | `Se0/3/1` | `10.0.0.6` | DTE   |

No lado DCE:

```text
clock rate 64000
```

A WAN 2 representa o caminho secundário entre a Filial RJ e o Datacenter.

---

## 4.3 WAN 3 — CPD ↔ Matriz

```text
10.0.0.8/30
```

| Ponta     | Interface | IP          | Papel |
| :-------- | :-------- | :---------- | :---- |
| CPD       | `Se0/3/0` | `10.0.0.9`  | DCE   |
| Matriz SP | `Se0/3/1` | `10.0.0.10` | DTE   |

No lado DCE:

```text
clock rate 64000
```

A WAN 3 também transporta a sessão eBGP entre o `HQ-Edge-RTR` e o `CPD-Datacenter-RTR`.

---

# 🧭 5. Caminho Primário e Caminhos de Contingência

A arquitetura combina o roteamento dinâmico com rotas estáticas de segurança.

O princípio de funcionamento documentado é:

```text
                 ┌───────────────┐
                 │   MATRIZ SP   │
                 └───────┬───────┘
                         │
                      WAN 1
                         │
                         ▼
                 ┌───────────────┐
                 │   FILIAL RJ   │
                 └───────┬───────┘
                         │
                      WAN 2
                         │
                         ▼
                 ┌───────────────┐
                 │      CPD      │
                 └───────────────┘
```

Ao mesmo tempo, a WAN 3 fecha o anel:

```text
CPD ───────────── WAN 3 ───────────── MATRIZ
```

Essa estrutura permite que a comunicação possa utilizar caminhos alternativos em situações de falha.

---

# 🛡️ 6. Rotas Estáticas Flutuantes — AD 115

O `Branch-Edge-RTR` possui rotas estáticas de contingência configuradas com **Distância Administrativa 115**.

A finalidade dessas rotas é fornecer caminhos alternativos quando as rotas dinâmicas correspondentes deixam de estar disponíveis.

---

## 6.1 Contingência com a Matriz SP

Foram configuradas rotas cobrindo:

```text
172.16.0.0/20
172.16.99.0/24
```

apontando para:

```text
10.0.0.1
```

com:

```text
AD 115
```

Configuração documentada:

```text
ip route 172.16.0.0 255.255.240.0 10.0.0.1 115
ip route 172.16.99.0 255.255.255.0 10.0.0.1 115
```

Essas rotas funcionam como contingência contra falhas lógicas do processo OSPF no enlace direto WAN 1.

---

## 6.2 Contingência com o CPD através da WAN 2

Também foram configuradas rotas estáticas de contingência para as redes do Datacenter:

```text
172.16.32.0/22
192.168.100.1/32
```

utilizando:

```text
10.0.0.6
```

com **AD 115**.

Configuração documentada:

```text
ip route 172.16.32.0 255.255.252.0 10.0.0.6 115
ip route 192.168.100.1 255.255.255.255 10.0.0.6 115
```

---

# 🔄 7. Redistribuição das Rotas Estáticas no RJ

O `Branch-Edge-RTR` redistribui as rotas estáticas para o OSPF:

```text
router ospf 1
 redistribute static subnets
```

Essa integração permite que as rotas de contingência do roteador regional sejam disponibilizadas ao domínio OSPF.

O `Branch-Core-3650`, por sua vez, mantém uma rota padrão apontando para o roteador regional:

```text
ip route 0.0.0.0 0.0.0.0 172.19.4.2
```

Dessa forma, o Core regional possui um caminho para fora de suas redes locais através do `Branch-Edge-RTR`.

---

# 🗄️ 8. Rotas Estáticas de Retorno no CPD

O `CPD-Datacenter-RTR` possui rotas estáticas reversas para as redes da Filial RJ:

```text
172.19.0.0/22
172.19.99.0/24
```

Ambas apontam para o endereço do roteador regional através da WAN 2:

```text
10.0.0.5
```

Configuração documentada:

```text
ip route 172.19.0.0 255.255.252.0 10.0.0.5
ip route 172.19.99.0 255.255.255.0 10.0.0.5
```

Esse caminho de retorno permite que o CPD encaminhe o tráfego destinado à Filial RJ diretamente pela WAN 2.

---

# 🔬 9. Cenário de Falha da WAN 1

A arquitetura foi estruturada para contemplar a interrupção do enlace primário entre Matriz SP e Filial RJ.

### Situação normal

```text
MATRIZ SP
    │
    │ WAN 1
    ▼
FILIAL RJ
```

### Com a interrupção da WAN 1

O anel oferece o caminho alternativo:

```text
MATRIZ SP
    │
    │ WAN 3
    ▼
CPD
    │
    │ WAN 2
    ▼
FILIAL RJ
```

O mecanismo de contingência combina:

* topologia física em anel;
* OSPF Área 0;
* redistribuição de rotas;
* rotas estáticas com AD 115;
* rota padrão no Core da Filial;
* rotas estáticas de retorno no CPD.

---

# 🧪 10. Validação do Roteamento

A verificação operacional do comportamento de roteamento deve observar principalmente:

| Elemento                   | Estado esperado             |
| :------------------------- | :-------------------------- |
| Adjacências OSPF           | Estabelecidas               |
| Processo OSPF              | `1`                         |
| Área                       | `0`                         |
| Router IDs                 | Únicos conforme projeto     |
| Sessão eBGP                | Estabelecida entre HQ e CPD |
| AS Matriz/RJ               | `65001`                     |
| AS CPD                     | `65002`                     |
| WAN 1                      | Enlace primário SP ↔ RJ     |
| WAN 2                      | Caminho RJ ↔ CPD            |
| WAN 3                      | Peering HQ ↔ CPD            |
| Rotas estáticas flutuantes | AD `115`                    |
| Rota padrão do Core RJ     | Próximo salto `172.19.4.2`  |
| Rotas de retorno do CPD    | Próximo salto `10.0.0.5`    |

### Comandos de verificação

```text
show ip ospf neighbor
show ip route
show ip route ospf
show ip route bgp
show ip protocols
show ip bgp summary
show ip bgp
show interfaces serial 0/3/0
show interfaces serial 0/3/1
```

Os comandos acima são referências para inspeção do estado da infraestrutura. As evidências visuais correspondentes fazem parte da documentação do projeto e servem para demonstrar o funcionamento da implementação no ambiente simulado.

---

# 🗺️ 11. Visão Consolidada do Roteamento

```text
                           ┌──────────────────────────┐
                           │ CPD-Datacenter-RTR       │
                           │ AS 65002                 │
                           │                          │
                           │ OSPF + eBGP              │
                           └──────┬───────────┬───────┘
                                  │           │
                              WAN 2│           │WAN 3
                                  │           │
                                  │           │ eBGP
                                  │           │
                                  ▼           ▼
                     ┌────────────────┐   ┌────────────────┐
                     │ Branch-Edge-RTR│   │  HQ-Edge-RTR   │
                     │ AS 65001       │   │ AS 65001       │
                     │ OSPF           │   │ OSPF + BGP     │
                     └───────┬────────┘   └───────┬────────┘
                             │                    │
                         OSPF│                    │OSPF
                             │                    │
                             ▼                    ▼
                     ┌──────────────┐      ┌──────────────┐
                     │ Branch-Core  │      │  HQ-Core     │
                     │   3650       │      │    3650      │
                     └──────────────┘      └──────────────┘
```

---

# 📌 12. Resumo dos Mecanismos de Resiliência

| Mecanismo                              | Função no projeto                               |
| :------------------------------------- | :---------------------------------------------- |
| **OSPFv2 Área 0**                      | Roteamento interno entre Core e Edge            |
| **eBGPv4**                             | Integração entre AS `65001` e AS `65002`        |
| **Redistribuição BGP → OSPF**          | Disponibiliza rotas do CPD ao domínio interno   |
| **Redistribuição OSPF → BGP**          | Disponibiliza rotas da Matriz ao CPD            |
| **WAN em anel**                        | Disponibiliza caminhos físicos alternativos     |
| **Rotas estáticas AD 115**             | Contingência de roteamento na Filial RJ         |
| **Redistribuição de estáticas → OSPF** | Propaga contingência ao Core RJ                 |
| **Default route no Core RJ**           | Encaminha tráfego externo ao `Branch-Edge-RTR`  |
| **Rotas de retorno no CPD**            | Mantêm o caminho de retorno para as redes do RJ |

---

# 📁 13. Arquivos Relacionados

```text
docs/
├── 01-architecture-and-addressing.md
├── 02-routing-and-resilience.md
├── 03-design-decisions.md
└── 04-limitations-and-lessons.md
```

| Arquivo                             | Escopo                                         |
| :---------------------------------- | :--------------------------------------------- |
| `01-architecture-and-addressing.md` | Topologia, ativos, VLANs e endereçamento       |
| `02-routing-and-resilience.md`      | OSPF, eBGP, redistribuição, WAN e contingência |
| `03-design-decisions.md`            | Racional técnico das decisões de arquitetura   |
| `04-limitations-and-lessons.md`     | Restrições do laboratório e aprendizados       |

Este documento, portanto, concentra o **comportamento dos caminhos de roteamento e dos mecanismos de resiliência**, sem duplicar o detalhamento estrutural e de endereçamento apresentado no documento de arquitetura.
