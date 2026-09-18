# Design Decisions — ANBIMA Financial Hub

[![Architecture: Multilayer](https://img.shields.io/badge/Architecture-Multilayer-blue)](#-1-arquitetura-em-camadas)
[![Segmentation: VLANs](https://img.shields.io/badge/Segmentation-VLANs-darkgreen)](#-2-segmenta%C3%A7%C3%A3o-l%C3%B3gica)
[![Routing: OSPF + eBGP](https://img.shields.io/badge/Routing-OSPFv2%20%2B%20eBGP-orange)](#-3-roteamento-h%C3%ADbrido)
[![Resilience: WAN Ring](https://img.shields.io/badge/Resilience-WAN%20Ring-purple)](#-4-resili%C3%AAncia-e-conting%C3%AAncia)

Este documento registra as principais **decisões de projeto** adotadas na construção da infraestrutura corporativa simulada da ANBIMA.

O objetivo não é repetir a configuração dos equipamentos, mas documentar **o racional técnico por trás da arquitetura**: por que determinados papéis foram atribuídos aos equipamentos, por que a rede foi segmentada, por que foram utilizados OSPF e eBGP, por que a WAN foi construída em anel e por que mecanismos estáticos de contingência foram adicionados ao roteamento dinâmico.

> **Princípio deste documento:** registrar o motivo das escolhas, enquanto os detalhes de implementação permanecem nos documentos técnicos específicos.

---

# 🏛️ 1. Arquitetura em Camadas

A primeira decisão estrutural foi separar a infraestrutura em diferentes funções:

```text
┌─────────────────────────────────────────────┐
│              ESTAÇÕES FINAIS                │
│        Computadores dos departamentos       │
└──────────────────────┬──────────────────────┘
                       │
                       │ Access
                       ▼
┌─────────────────────────────────────────────┐
│              ACCESS SWITCH                  │
│             Cisco Catalyst 2960             │
│                 Camada 2                    │
└──────────────────────┬──────────────────────┘
                       │
                       │ Trunk 802.1Q
                       ▼
┌─────────────────────────────────────────────┐
│                 CORE L3                     │
│             Cisco Catalyst 3650             │
│      SVIs • Routing • DHCP • OSPF           │
└──────────────────────┬──────────────────────┘
                       │
                       │ Trânsito L3
                       ▼
┌─────────────────────────────────────────────┐
│              EDGE ROUTER                    │
│                Cisco 2911                   │
│          OSPF • WAN • eBGP                  │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
                  MALHA WAN
```

Essa separação permite que cada equipamento desempenhe uma função específica dentro da infraestrutura.

### Distribuição das responsabilidades

| Equipamento       | Função escolhida                      |
| :---------------- | :------------------------------------ |
| **Catalyst 2960** | Acesso Layer 2                        |
| **Catalyst 3650** | Core Layer 3                          |
| **Cisco 2911**    | Borda, WAN e integração de roteamento |
| **Server PT**     | Serviço central do CPD                |

O cenário atribui ao `HQ-Core-3650` e ao `Branch-Core-3650` as funções de roteamento L3, hospedagem das SVIs, DHCP e participação no OSPF.

Os roteadores de borda ficam responsáveis pela conectividade WAN e pela integração entre os domínios de roteamento.

---

# 🧩 2. Segmentação Lógica por VLAN

A rede da Matriz foi dividida em VLANs correspondentes aos diferentes grupos funcionais:

| VLAN  | Área                         |
| :---- | :--------------------------- |
| `10`  | Segurança Operacional        |
| `20`  | Núcleo Financeiro            |
| `30`  | Dados de Mercado e Analytics |
| `40`  | Auditoria e Compliance       |
| `99`  | Gerência / Hardening         |
| `200` | Trânsito L3                  |

Na Filial RJ, a mesma lógica de separação foi aplicada de acordo com as funções regionais:

| VLAN  | Área                               |
| :---- | :--------------------------------- |
| `10`  | Supervisão Regional e Fiscalização |
| `20`  | Operações Regionais e Negócios     |
| `99`  | Gerência / Hardening               |
| `200` | Trânsito L3                        |

### Motivo da decisão

A segmentação permite separar logicamente os diferentes domínios funcionais da organização dentro da infraestrutura de switching.

A VLAN `99` foi reservada especificamente para gerenciamento dos switches de acesso, mantendo as interfaces de gerenciamento em uma rede dedicada.

A VLAN `200` foi utilizada para o trânsito L3 entre o Core e o roteador de borda, separando esse enlace da segmentação destinada às estações de trabalho.

---

# 🔌 3. Escolha do Trunk 802.1Q

Os switches de acesso foram conectados aos respectivos switches Core através de enlaces **trunk 802.1Q**.

```text
HQ-Access-2960
       │
       │ 802.1Q
       │ VLANs 10,20,30,40,99
       ▼
HQ-Core-3650
```

Na Filial:

```text
Branch-Access-2960
       │
       │ 802.1Q
       │ VLANs 10,20,99
       ▼
Branch-Core-3650
```

### Motivo da decisão

O trunk permite transportar múltiplas VLANs através de um único enlace físico entre Access e Core.

O cenário também restringe explicitamente as VLANs permitidas nos trunks, utilizando:

```text
switchport trunk allowed vlan
```

Essa decisão reduz o domínio de VLANs transportado pelo enlace e faz parte das medidas de proteção adotadas contra **VLAN Hopping**.

---

# 🚦 4. Roteamento Layer 3 no Core

Os Catalyst 3650 foram utilizados como switches multicamada, com:

```text
ip routing
```

e SVIs atuando como gateways das VLANs.

### Motivo da decisão

O cenário atribui ao Core a responsabilidade pelo roteamento entre as redes locais, evitando concentrar todo o tráfego interno no roteador de borda.

Assim, o Core pode realizar simultaneamente:

* gateway das VLANs;
* roteamento L3;
* DHCP;
* participação no OSPF;
* conexão com o roteador de borda através da VLAN de trânsito.

A própria documentação do cenário descreve o `HQ-Core-3650` como o “coração da rede local da Matriz” e atribui ao `Branch-Core-3650` a autonomia operacional da filial.

---

# 🌐 5. OSPFv2 como IGP

O **OSPFv2** foi escolhido como protocolo de roteamento interno.

A implementação utiliza:

```text
OSPF Process 1
Area 0
```

nos equipamentos Core e Edge da Matriz e da Filial RJ.

### Motivo da decisão

O cenário caracteriza o OSPF como o protocolo interno responsável pelo conhecimento das redes corporativas.

Sua utilização permite que:

```text
Core
  ↕
OSPF
  ↕
Edge
```

mantenham conhecimento dinâmico das redes necessárias para a comunicação entre os diferentes segmentos.

O cenário também utiliza a lógica de **Shortest Path First (Dijkstra)** e custos associados à largura de banda dos enlaces.

---

# 🌎 6. eBGP para a Fronteira entre Sistemas Autônomos

Enquanto o OSPF é utilizado internamente, o cenário utiliza **eBGPv4** na comunicação entre os diferentes Sistemas Autônomos:

```text
AS 65001
Matriz + Filial RJ
       │
       │ eBGP
       │
AS 65002
CPD
```

### Motivo da decisão

A separação permite representar uma fronteira clara entre:

* o domínio corporativo formado pela Matriz e Filial;
* o domínio do Datacenter.

O `HQ-Edge-RTR` atua como ponto de integração com o domínio externo, estabelecendo a sessão eBGP com o roteador do CPD.

A sessão é estabelecida através da WAN 3.

---

# 🔄 7. Redistribuição entre OSPF e BGP

O `HQ-Edge-RTR` foi escolhido como ponto de integração entre os dois protocolos.

```text
             HQ-Edge-RTR
                  │
        ┌─────────┴─────────┐
        │                   │
      OSPF                 eBGP
        │                   │
   Rede interna            CPD
```

O cenário utiliza redistribuição nos dois sentidos:

```text
BGP → OSPF
OSPF → BGP
```

### Motivo da decisão

Essa arquitetura permite que:

* o Core conheça as redes anunciadas pelo CPD sem precisar executar BGP;
* o CPD receba as redes corporativas da Matriz através do BGP.

A decisão mantém o BGP concentrado na borda, enquanto o OSPF permanece responsável pelo roteamento interno.

---

# 🔁 8. Topologia WAN em Anel

A WAN foi estruturada como um **anel fechado**:

```text
                 CPD
                /   \
             WAN 2  WAN 3
              /       \
             /         \
        FILIAL ──WAN 1── MATRIZ
```

### Motivo da decisão

A estrutura cria três enlaces:

| Enlace    | Conexão            |
| :-------- | :----------------- |
| **WAN 1** | Matriz ↔ Filial RJ |
| **WAN 2** | Filial RJ ↔ CPD    |
| **WAN 3** | CPD ↔ Matriz       |

O cenário relaciona explicitamente a topologia em anel à necessidade de garantir caminhos alternativos entre os três pontos.

A utilização do módulo serial **HWIC-2T** no Slot 3 dos roteadores também foi padronizada para suportar os três enlaces seriais do projeto.

---

# ⏱️ 9. Padronização DCE/DTE

O projeto adotou uma distribuição padronizada de interfaces DCE e DTE:

```text
Matriz
Se0/3/0 → DCE
       ↓
Filial
Se0/3/1 → DTE

Filial
Se0/3/0 → DCE
       ↓
CPD
Se0/3/1 → DTE

CPD
Se0/3/0 → DCE
       ↓
Matriz
Se0/3/1 → DTE
```

Os lados DCE utilizam:

```text
clock rate 64000
```

### Motivo da decisão

O próprio cenário informa que esse padrão foi adotado para garantir o sincronismo de clock e evitar uma distribuição incorreta dos papéis DCE/DTE nos enlaces seriais.

A padronização também facilita a leitura e a validação da topologia no laboratório.

---

# 🛡️ 10. Rotas Estáticas Flutuantes como Contingência

Apesar da utilização de roteamento dinâmico, o cenário adiciona rotas estáticas com:

```text
Administrative Distance = 115
```

no `Branch-Edge-RTR`.

### Motivo da decisão

As rotas foram utilizadas como mecanismo complementar de contingência.

O cenário especifica dois grupos principais:

```text
Filial → Matriz
172.16.0.0/20
172.16.99.0/24
```

e:

```text
Filial → CPD
172.16.32.0/22
192.168.100.1/32
```

A escolha de uma distância administrativa de `115` permite que essas rotas tenham comportamento de contingência em relação às rotas dinâmicas documentadas.

O objetivo declarado no cenário é oferecer uma alternativa em caso de falhas lógicas do roteamento OSPF ou indisponibilidade do caminho primário.

---

# 🏢 11. Autonomia da Filial RJ

O `Branch-Core-3650` mantém uma rota padrão direcionada ao `Branch-Edge-RTR`:

```text
ip route 0.0.0.0 0.0.0.0 172.19.4.2
```

### Motivo da decisão

A decisão mantém uma separação clara entre:

```text
Rede local da Filial
        │
        ▼
Branch-Core-3650
        │
        ▼
Branch-Edge-RTR
        │
        ▼
WAN
```

O Core continua concentrando as funções locais, enquanto o roteador regional representa a saída da filial para a infraestrutura WAN.

O cenário associa essa configuração à necessidade de manter a conectividade da filial com os demais ambientes.

---

# 🗄️ 12. Rotas de Retorno no CPD

O CPD possui rotas estáticas reversas para os blocos da Filial RJ:

```text
172.19.0.0/22
172.19.99.0/24
```

apontando para:

```text
10.0.0.5
```

através da WAN 2.

### Motivo da decisão

O cenário utiliza essas rotas para garantir que exista um caminho de retorno entre o CPD e a Filial RJ.

Essa decisão é especialmente relevante no cenário de interrupção da WAN 1, no qual o tráfego pode utilizar:

```text
Matriz
  ↓
WAN 3
  ↓
CPD
  ↓
WAN 2
  ↓
Filial RJ
```

Dessa maneira, a contingência não depende somente da existência de um caminho de ida: o caminho de retorno também foi considerado na arquitetura.

---

# 📐 13. VLSM e Hierarquia de Endereçamento

O plano de endereçamento utiliza diferentes tamanhos de prefixo:

```text
172.16.0.0/20
       │
       ├── /22 → Departamentos da Matriz
       ├── /24 → Gerência
       └── /30 → Trânsito L3

172.19.0.0/22
       │
       ├── /23 → Departamentos da Filial
       ├── /24 → Gerência
       └── /30 → Trânsito L3
```

### Motivo da decisão

A utilização de VLSM permite separar os blocos de acordo com suas respectivas funções e tamanhos de rede definidos no cenário.

Os enlaces ponto a ponto utilizam `/30`, enquanto os segmentos departamentais utilizam `/22` na Matriz e `/23` na Filial.

O detalhamento completo do plano de endereçamento está documentado em:

```text
docs/01-architecture-and-addressing.md
```

---

# 💻 14. DHCP Centralizado por Site

Os switches Core foram escolhidos para atuar como servidores DHCP locais.

Na Matriz:

```text
HQ-Core-3650
      │
      ├── POOL_SEC_OPS
      ├── POOL_FINANCIAL
      ├── POOL_ANALYTICS
      └── POOL_AUDIT
```

Na Filial:

```text
Branch-Core-3650
      │
      ├── POOL_RJ_SUPERVISAO
      └── POOL_RJ_OPERATIONS
```

### Motivo da decisão

A função DHCP permanece próxima aos gateways das respectivas VLANs, permitindo que cada Core gerencie os endereços de seus segmentos locais.

O cenário também determina a exclusão dos primeiros 50 endereços de cada sub-rede, reservando-os para infraestrutura e fazendo a distribuição dinâmica começar no `.51`.

O servidor DNS utilizado pelos pools é o servidor central do CPD:

```text
172.16.32.10
```

---

# 🔐 15. VLAN de Gerenciamento

A VLAN `99` foi reservada para gerenciamento:

```text
Matriz:
172.16.99.0/24

Filial:
172.19.99.0/24
```

### Motivo da decisão

O cenário separa explicitamente o tráfego de gerenciamento das redes destinadas às estações de trabalho.

Os switches de acesso recebem endereços nessa rede dedicada, enquanto o acesso administrativo é protegido através das configurações de SSHv2.

A decisão também está relacionada à estratégia de hardening da infraestrutura.

---

# 🔒 16. SSHv2 em vez de Telnet

O gerenciamento remoto foi projetado utilizando **SSH versão 2**, com Telnet desativado.

A implementação inclui:

```text
transport input ssh
ip ssh version 2
```

e chaves RSA de:

```text
2048 bits
```

### Motivo da decisão

O cenário estabelece a desativação de protocolos claros e a utilização de SSHv2 para evitar o transporte de credenciais em texto não criptografado.

O domínio institucional definido é:

```text
anbima.corp
```

A autenticação utiliza banco local e privilégio administrativo de nível `15`.

---

# 🧱 17. Padronização dos Equipamentos

A arquitetura utiliza uma combinação específica de equipamentos:

| Modelo                       | Papel                     |
| :--------------------------- | :------------------------ |
| **Cisco 2911**               | Roteamento de borda e WAN |
| **Cisco Catalyst 3650-24PS** | Core Layer 3              |
| **Cisco Catalyst 2960-24TT** | Access Layer 2            |
| **Cisco Server PT**          | Servidor do CPD           |

### Motivo da decisão

A distribuição de funções acompanha as capacidades utilizadas no cenário:

```text
2911
└── WAN / Borda / OSPF / eBGP

3650
└── L3 / SVI / DHCP / OSPF

2960
└── L2 / Access / Trunk

Server PT
└── Serviços do CPD
```

Isso mantém as funções de acesso, distribuição local e borda claramente separadas dentro do laboratório.

---

# 📋 18. Matriz Consolidada de Decisões

| Decisão                 | Implementação         | Motivo documentado                         |
| :---------------------- | :-------------------- | :----------------------------------------- |
| Arquitetura multicamada | Access + Core + Edge  | Separação de responsabilidades             |
| Catalyst 2960 no Access | Layer 2               | Conexão das estações e transporte de VLANs |
| Catalyst 3650 no Core   | Layer 3               | SVIs, DHCP e roteamento                    |
| Cisco 2911 na borda     | WAN + roteamento      | Conectividade entre sites                  |
| VLANs departamentais    | Segmentação funcional | Separação lógica dos ambientes             |
| VLAN 99                 | Gerenciamento         | Isolamento da gerência                     |
| VLAN 200                | Trânsito L3           | Conexão Core ↔ Edge                        |
| Trunk 802.1Q            | Access ↔ Core         | Transporte das VLANs                       |
| OSPFv2 Área 0           | IGP                   | Roteamento interno                         |
| eBGP AS 65001/65002     | EGP                   | Separação entre domínios                   |
| Redistribuição OSPF/BGP | HQ-Edge-RTR           | Integração dos domínios                    |
| WAN em anel             | 3 enlaces seriais     | Caminho alternativo                        |
| HWIC-2T Slot 3          | Interfaces seriais    | Padronização da WAN                        |
| DCE/DTE padronizado     | Clock `64000` no DCE  | Sincronismo serial                         |
| Rotas flutuantes AD 115 | Branch-Edge-RTR       | Contingência                               |
| Rotas de retorno no CPD | Via WAN 2             | Retorno do tráfego regional                |
| DHCP no Core            | Pools locais          | Distribuição dinâmica por VLAN             |
| SSHv2 + RSA 2048        | Gerenciamento         | Proteção do acesso administrativo          |

---

# 🧭 19. Separação de Responsabilidades na Documentação

Para evitar duplicação dentro do repositório, cada documento possui uma finalidade específica:

```text
docs/
│
├── 01-architecture-and-addressing.md
│   └── Onde está cada coisa?
│       Topologia • Ativos • VLANs • IPs
│
├── 02-routing-and-resilience.md
│   └── Como os caminhos funcionam?
│       OSPF • eBGP • Redistribuição • WAN • Failover
│
├── 03-design-decisions.md
│   └── Por que foi projetado assim?
│       Racional técnico das escolhas
│
└── 04-limitations-and-lessons.md
    └── Quais foram as restrições e aprendizados?
        Limitações • Problemas • Lições • Melhorias
```

Essa divisão mantém este arquivo dedicado ao **racional arquitetural**, sem transformar o documento em uma segunda cópia dos scripts ou do plano de endereçamento.

---

# 📌 20. Síntese das Decisões

A arquitetura foi construída a partir de uma sequência de decisões complementares:

```text
SEGMENTAÇÃO
    │
    ▼
VLANs por função
    │
    ▼
CORE L3
    │
    ├── SVIs
    ├── DHCP
    └── OSPF
          │
          ▼
       EDGE
          │
    ┌─────┴─────┐
    │           │
  OSPF        eBGP
    │           │
    └─────┬─────┘
          │
       WAN Ring
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
   SP    RJ    CPD
          │
          ▼
   Rotas de contingência
```

O resultado é uma arquitetura em que **segmentação, roteamento interno, integração externa e contingência** possuem papéis distintos, mas complementares.

Os detalhes de implementação e os comandos efetivamente utilizados permanecem nos arquivos de configuração e no documento `02-routing-and-resilience.md`, enquanto este documento registra as decisões que fundamentaram essa estrutura.
