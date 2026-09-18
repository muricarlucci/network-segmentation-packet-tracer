# Threat Modeling — ANBIMA Financial Hub

![Security](https://img.shields.io/badge/Security-Threat%20Modeling-red)
![Network](https://img.shields.io/badge/Network-VLAN%20%2B%20Layer%203-blue)
![Management](https://img.shields.io/badge/Management-SSHv2-green)
![Resilience](https://img.shields.io/badge/Resilience-WAN%20Ring-orange)

O modelo de ameaças deste projeto foi elaborado a partir da arquitetura, dos mecanismos de segmentação, do roteamento, da contingência e do hardening descritos no cenário oficial da infraestrutura ANBIMA Financial Hub.

O objetivo é documentar **quais superfícies da infraestrutura apresentam riscos relevantes dentro do escopo implementado, quais eventos de ameaça são considerados e quais mecanismos existentes no projeto reduzem sua exposição ou impacto**.

Este documento representa o modelo de ameaças da implementação acadêmica em Cisco Packet Tracer. Ele não constitui uma avaliação formal de risco corporativo, uma certificação de segurança ou uma declaração de conformidade normativa.

---

## 🏛️ Escopo do Modelo de Ameaças

O modelo considera os principais componentes da infraestrutura descrita no projeto:

| Componente                    | Superfície considerada                                    |
| ----------------------------- | --------------------------------------------------------- |
| Switch Core da Matriz         | VLANs, roteamento Layer 3, interfaces SVI e gerenciamento |
| Switch de Acesso da Matriz    | Trunk, VLANs e gerenciamento                              |
| Roteador da Matriz            | WAN, OSPF, eBGP, redistribuição e acesso administrativo   |
| Switch Core da Filial RJ      | VLANs, roteamento Layer 3, DHCP, OSPF e gerenciamento     |
| Switch de Acesso da Filial RJ | Trunk, VLANs e gerenciamento                              |
| Roteador Regional RJ          | WAN, OSPF, rotas estáticas e redistribuição               |
| Roteador do CPD               | WAN, eBGP, redistribuição, loopback e rotas de retorno    |
| Enlaces WAN                   | WAN 1, WAN 2 e WAN 3                                      |
| Interfaces de gerenciamento   | VLAN 99 e acesso administrativo por SSH                   |
| Segmentação lógica            | VLANs departamentais e VLAN de gerenciamento              |

O cenário informa que existem **7 ativos gerenciáveis** na topologia e que todos possuem parâmetros de hardening ativos.

---

# 1. Modelo de Ameaças da Arquitetura

A análise concentra-se em cinco grandes superfícies:

```text
                    ┌───────────────────────────┐
                    │     ADMINISTRAÇÃO         │
                    │      SSHv2 / VTY          │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      INFRAESTRUTURA        │
                    │ Switches + Roteadores      │
                    └───────┬───────────┬───────┘
                            │           │
                ┌───────────▼───┐   ┌──▼────────────┐
                │ SEGMENTAÇÃO   │   │   ROUTING     │
                │ VLAN / Trunk   │   │ OSPF / BGP    │
                └───────────┬────┘   └──────┬────────┘
                            │               │
                            └───────┬───────┘
                                    │
                          ┌─────────▼─────────┐
                          │       WAN         │
                          │  SP ↔ RJ ↔ CPD   │
                          └───────────────────┘
```

Cada superfície possui diferentes possibilidades de comprometimento, desde acesso administrativo indevido até indisponibilidade de comunicação causada por falhas de enlaces ou de roteamento.

---

# 2. Matriz de Ameaças

| ID    | Superfície      | Ameaça / Evento                                            | Impacto potencial                                                 | Controle existente no cenário                              |
| ----- | --------------- | ---------------------------------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------- |
| TM-01 | Administração   | Acesso administrativo por protocolo inseguro               | Exposição de credenciais durante o gerenciamento                  | Telnet desativado e `transport input ssh`                  |
| TM-02 | Administração   | Uso de versão insegura do protocolo de administração       | Exposição associada ao protocolo de gerenciamento                 | SSHv2 obrigatório                                          |
| TM-03 | Administração   | Tentativas repetidas de autenticação                       | Tentativas de acesso indevido às VTY                              | Timeout de 60 segundos e limite de 4 tentativas            |
| TM-04 | Administração   | Comprometimento de credenciais privilegiadas               | Acesso administrativo aos equipamentos                            | `login local`, usuário com privilégio 15 e `enable secret` |
| TM-05 | Criptografia    | Chaves criptográficas inexistentes ou inadequadas para SSH | Impossibilidade de estabelecer o mecanismo criptográfico previsto | RSA de 2048 bits                                           |
| TM-06 | Camada 2        | VLAN Hopping                                               | Acesso indevido entre segmentos VLAN                              | Restrição explícita das VLANs permitidas nos trunks        |
| TM-07 | Camada 2        | Acesso de usuários à rede de gerenciamento                 | Exposição das interfaces de gerenciamento                         | VLAN 99 dedicada                                           |
| TM-08 | Roteamento      | Falha lógica no OSPF                                       | Perda de conectividade entre segmentos                            | Rotas estáticas flutuantes e redistribuição                |
| TM-09 | WAN             | Falha da WAN 1                                             | Interrupção do caminho principal entre SP e RJ                    | WAN 2 e rotas estáticas de contingência                    |
| TM-10 | WAN             | Falha da WAN 2                                             | Perda do caminho direto RJ–CPD                                    | Rotas pela arquitetura WAN e mecanismos de roteamento      |
| TM-11 | WAN             | Falha da WAN 3                                             | Perda do enlace direto CPD–Matriz                                 | Caminho alternativo através da WAN 2                       |
| TM-12 | Roteamento      | Assimetria ou ausência de rota de retorno                  | Pacotes chegam ao destino, mas não retornam corretamente          | Rotas estáticas de retorno no CPD                          |
| TM-13 | Redistribuição  | Propagação incorreta de rotas entre protocolos             | Instabilidade ou perda de conectividade                           | Redistribuição OSPF/BGP e rotas estáticas                  |
| TM-14 | Endereçamento   | Alteração de sub-redes sem atualização das rotas           | Isolamento de segmentos durante contingência                      | Estrutura de endereçamento e rotas correspondentes         |
| TM-15 | Disponibilidade | Indisponibilidade de um enlace WAN                         | Interrupção de comunicação entre localidades                      | Topologia WAN em anel                                      |

---

# 3. Ameaças à Administração dos Equipamentos

## 3.1 Gerenciamento por protocolo inseguro

O cenário identifica o uso de Telnet como uma superfície que deve ser eliminada da administração dos equipamentos.

A configuração determina:

```cisco
line vty 0 4
login local
transport input ssh
```

Dessa forma, as portas VTY aceitam SSH em vez de Telnet.

O cenário associa essa configuração à **desativação de protocolos claros**, evitando que senhas sejam transmitidas em texto não criptografado durante o gerenciamento remoto.

### Controle implementado

```cisco
TRANSPORT INPUT SSH
```

### Objetivo

Reduzir a exposição das credenciais administrativas durante sessões remotas.

---

# 4. Ameaças ao SSH

## 4.1 Uso de versão inadequada

O cenário determina a utilização exclusiva do SSH versão 2:

```cisco
ip ssh version 2
```

A configuração evita que o gerenciamento remoto utilize a versão legada do protocolo prevista no cenário.

### Controle implementado

```cisco
IP SSH VERSION 2
```

---

## 4.2 Ausência de identidade criptográfica

O funcionamento do SSH é associado à geração de um par de chaves RSA próprio para cada equipamento.

O cenário determina:

```cisco
crypto key generate rsa
2048
```

Além disso, os equipamentos utilizam o domínio:

```cisco
ip domain-name ANBIMA.CORP
```

### Controle implementado

**RSA de 2048 bits + domínio `ANBIMA.CORP`**

### Objetivo

Disponibilizar a base criptográfica necessária ao mecanismo de administração SSH implementado no laboratório.

---

# 5. Ameaças de Autenticação Administrativa

## 5.1 Tentativas repetidas de autenticação

O cenário possui dois parâmetros diretamente relacionados ao controle das sessões SSH:

```cisco
ip ssh time-out 60
ip ssh authentication-retries 4
```

A sessão possui timeout de 60 segundos e o número de tentativas de autenticação é limitado a quatro.

### Controle

| Parâmetro                  |       Valor |
| -------------------------- | ----------: |
| Timeout SSH                | 60 segundos |
| Tentativas de autenticação |           4 |

### Objetivo

Reduzir a exposição da interface administrativa a sessões prolongadas e múltiplas tentativas de autenticação.

---

## 5.2 Comprometimento do privilégio administrativo

O cenário utiliza autenticação local:

```cisco
login local
```

e define um usuário administrativo com privilégio 15:

```cisco
username admin privilege 15 secret ANBIMA@SEC2026!
```

Também existe proteção do modo privilegiado através de `enable secret`.

```cisco
no enable password
enable secret cisco
```

O próprio cenário observa que o valor `cisco` foi utilizado no laboratório para demonstração acadêmica.

### Controle implementado

* autenticação local;
* usuário administrativo;
* privilégio 15;
* `secret` para a credencial;
* remoção do `enable password`;
* utilização de `enable secret`.

---

# 6. Ameaças de Camada 2

## 6.1 VLAN Hopping

A própria arquitetura identifica o **VLAN Hopping** como uma ameaça relevante à segmentação.

Para reduzir essa exposição, os enlaces trunk possuem uma lista explícita de VLANs permitidas.

### Matriz SP

```text
allowed vlan 10,20,30,40,99
```

### Filial RJ

```text
allowed vlan 10,20,99
```

A restrição evita que VLANs não previstas no projeto sejam transportadas pelo enlace tronco.

### Controle

| Local     | VLANs permitidas no trunk |
| --------- | ------------------------- |
| Matriz SP | 10, 20, 30, 40, 99        |
| Filial RJ | 10, 20, 99                |

---

# 7. Ameaça ao Plano de Gerenciamento

## 7.1 Acesso de estações de usuários à rede de gerenciamento

A infraestrutura utiliza a **VLAN 99** para separar as interfaces de gerenciamento dos switches de acesso.

### Matriz SP

```text
172.16.99.0/24
```

### Filial RJ

```text
172.19.99.0/24
```

O cenário descreve essa separação como um mecanismo para impedir que estações de trabalho de usuários finais acessem diretamente as portas de console e gerenciamento remoto.

### Controle

```text
VLAN 99
MGMT_HARDENING
```

na Matriz, e:

```text
VLAN 99
RJ_MGMT
```

na Filial RJ.

---

# 8. Ameaças ao Roteamento

A arquitetura utiliza simultaneamente:

```text
OSPFv2
   │
   ├── Roteamento interno
   │
   ▼
Matriz / Filial
   │
   └──────────────┐
                  ▼
               eBGPv4
                  │
                  ▼
                 CPD
```

Essa arquitetura introduz uma superfície adicional: a interação entre diferentes mecanismos de roteamento.

---

## 8.1 Falha lógica do OSPF

O cenário implementa rotas estáticas flutuantes com **Distância Administrativa 115** para atuar como contingência diante de falhas lógicas do processo OSPF no enlace direto WAN 1.

Na Filial RJ, existem rotas de contingência para:

```text
172.16.0.0/20
172.16.99.0/24
```

apontando para:

```text
10.0.0.1
```

com AD 115.

### Controle

```cisco
ip route 172.16.0.0 255.255.240.0 10.0.0.1 115
ip route 172.16.99.0 255.255.255.0 10.0.0.1 115
```

---

# 9. Ameaças de Indisponibilidade da WAN

A infraestrutura possui três enlaces ponto a ponto:

| WAN   | Origem    | Destino   | Rede          |
| ----- | --------- | --------- | ------------- |
| WAN 1 | Matriz SP | Filial RJ | `10.0.0.0/30` |
| WAN 2 | Filial RJ | CPD       | `10.0.0.4/30` |
| WAN 3 | CPD       | Matriz SP | `10.0.0.8/30` |

A disposição forma uma malha triangular entre as três localidades.

```text
                  WAN 1
          ┌──────────────────┐
          │                  │
          ▼                  │
     MATRIZ SP ──────────────┘
          │
          │ WAN 3
          ▼
         CPD
          │
          │ WAN 2
          ▼
      FILIAL RJ
          │
          └──────────── WAN 1 ────────────► MATRIZ
```

A existência dos três enlaces permite a utilização de caminhos alternativos quando um dos caminhos diretos deixa de estar disponível.

---

# 10. Falha da WAN 1

A WAN 1 conecta a Matriz SP à Filial RJ.

```text
MATRIZ SP
10.0.0.1
     │
     │ WAN 1
     │
10.0.0.2
FILIAL RJ
```

Em caso de interrupção do caminho principal, o cenário prevê mecanismos de contingência através da infraestrutura WAN e das rotas estáticas configuradas.

O roteador regional possui rotas estáticas flutuantes para os blocos corporativos da Matriz.

Além disso, essas rotas são redistribuídas no OSPF:

```cisco
redistribute static subnets
```

O `Branch-Core-3650` mantém uma rota padrão apontando para:

```text
172.19.4.2
```

Esse conjunto permite que a infraestrutura regional continue encaminhando o tráfego de acordo com o caminho alternativo previsto no projeto.

---

# 11. Falha do Caminho Direto entre RJ e Matriz

Quando a WAN 1 deixa de funcionar, o cenário utiliza a WAN 2 para alcançar o CPD.

O caminho alternativo passa a utilizar:

```text
FILIAL RJ
     │
     │ WAN 2
     ▼
    CPD
     │
     │ WAN 3
     ▼
MATRIZ SP
```

Essa arquitetura evita que a indisponibilidade de um único enlace interrompa necessariamente toda a comunicação entre as localidades.

---

# 12. Ameaça de Falta de Rota de Retorno

Um dos riscos explicitamente tratados pelo cenário é a existência de conectividade em apenas um sentido.

Para evitar o descarte do tráfego de retorno durante uma contingência, o roteador do CPD possui rotas estáticas reversas para as redes regionais:

```cisco
ip route 172.19.0.0 255.255.252.0 10.0.0.5
ip route 172.19.99.0 255.255.255.0 10.0.0.5
```

Essas rotas apontam para:

```text
10.0.0.5
```

na conexão com o roteador regional.

### Controle

**Rotas de retorno explícitas no CPD.**

### Objetivo

Garantir que o tráfego destinado à Filial RJ possua um caminho de retorno durante a utilização da WAN 2.

---

# 13. Ameaças na Redistribuição de Rotas

A arquitetura utiliza redistribuição entre diferentes protocolos:

```text
OSPF ⇄ BGP
```

Na Matriz:

```cisco
redistribute bgp 65001 subnets
```

No processo BGP:

```cisco
redistribute ospf 1
```

Na Filial RJ:

```cisco
redistribute static subnets
```

A redistribuição é, portanto, um ponto importante da arquitetura porque conecta diferentes fontes de informação de roteamento.

O próprio projeto utiliza configurações específicas para permitir a propagação das rotas necessárias entre os domínios.

---

# 14. Ameaça de Black Hole de Roteamento

A alteração de uma sub-rede sem atualização das rotas correspondentes pode produzir uma situação em que determinada rede exista localmente, mas não possua caminho válido através da infraestrutura de contingência.

Esse comportamento foi observado no desenvolvimento do projeto quando a rede de gerenciamento passou a utilizar:

```text
172.16.99.0/24
172.19.99.0/24
```

enquanto determinadas rotas de contingência permaneciam cobrindo blocos diferentes.

### Lição de segurança e disponibilidade

Alterações de endereçamento devem ser acompanhadas por uma auditoria completa das rotas:

```text
Endereçamento
     │
     ▼
Roteamento interno
     │
     ▼
Rotas de contingência
     │
     ▼
Rotas de retorno
```

A simples existência de uma rota local não garante conectividade durante todos os cenários de falha.

---

# 15. Threat Model Consolidado

| Ameaça                               | Área afetada  | Mecanismo de redução de exposição |
| ------------------------------------ | ------------- | --------------------------------- |
| Administração insegura               | SSH / VTY     | `transport input ssh`             |
| Uso de protocolo legado              | SSH           | SSHv2                             |
| Exposição de credenciais             | Administração | SSH + RSA                         |
| Tentativas repetidas                 | Autenticação  | 4 retries                         |
| Sessões administrativas prolongadas  | SSH           | timeout de 60 s                   |
| Comprometimento do modo privilegiado | CLI           | `enable secret`                   |
| VLAN Hopping                         | Layer 2       | `allowed vlan`                    |
| Acesso à gerência                    | Layer 2       | VLAN 99                           |
| Falha do OSPF                        | Routing       | Rotas estáticas AD 115            |
| Falha da WAN 1                       | WAN           | Caminho alternativo via CPD       |
| Falha do caminho direto              | WAN           | WAN 2 / WAN 3                     |
| Ausência de rota de retorno          | Routing       | Rotas reversas no CPD             |
| Problemas de redistribuição          | Routing       | Configuração explícita OSPF/BGP   |
| Alteração de endereçamento           | Routing       | Auditoria das rotas               |
| Indisponibilidade de enlace          | WAN           | Topologia triangular              |

---

# 16. Relação entre Ameaça e Controle

```text
                    AMEAÇA
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      Acesso        Camada 2     Indisponibilidade
    Administrativo     │             │
          │             │             │
          ▼             ▼             ▼
       SSHv2       VLAN 99       WAN Ring
          │       Allowed VLAN       │
          ▼             │             ▼
       RSA 2048         │       Rotas Flutuantes
          │              │             │
          ▼              ▼             ▼
   Login Local       Segmentação   Rotas de Retorno
          │
          ▼
   Controle de Acesso
```

O modelo demonstra que a proteção da infraestrutura não depende de um único mecanismo. A arquitetura utiliza controles diferentes conforme a superfície exposta:

* **administração:** SSHv2, RSA, autenticação local e controles de sessão;
* **camada 2:** segmentação por VLAN e restrição de VLANs nos trunks;
* **roteamento:** OSPF, eBGP, redistribuição e rotas estáticas;
* **disponibilidade:** múltiplos enlaces WAN e caminhos alternativos;
* **retorno de tráfego:** rotas estáticas reversas no CPD.

---

# 17. Limites do Threat Modeling

Este documento permanece deliberadamente limitado ao que foi implementado e documentado no cenário.

Não são considerados como controles implementados nesta topologia, por não fazerem parte da configuração apresentada:

* firewall de próxima geração;
* IDS/IPS;
* SIEM;
* DLP;
* autenticação centralizada;
* MFA;
* VPN;
* NAC;
* proteção específica contra malware;
* monitoramento de endpoints;
* mecanismos avançados de inspeção de tráfego.

Da mesma forma, o modelo não apresenta uma classificação formal de probabilidade, impacto financeiro ou risco residual, pois esses parâmetros não foram definidos no cenário da implementação.

---

# 18. Evidências Técnicas do Projeto

As evidências visuais incluídas neste repositório demonstram a implementação dos mecanismos utilizados como resposta às ameaças identificadas.

Entre os elementos que podem ser demonstrados pela documentação do projeto estão:

```text
✓ Topologia completa no Cisco Packet Tracer
✓ Segmentação por VLAN
✓ VLAN de gerenciamento
✓ Restrição dos enlaces trunk
✓ Roteamento Layer 3
✓ OSPFv2
✓ eBGPv4
✓ Redistribuição de rotas
✓ Rotas estáticas flutuantes
✓ Rotas de retorno
✓ Configuração SSHv2
✓ Chaves RSA
✓ Autenticação local
✓ Controle de tentativas de autenticação
```

As evidências visuais correspondentes fazem parte da documentação do projeto e servem para demonstrar o funcionamento da implementação no ambiente simulado.

---

# 19. Síntese

O modelo de ameaças do ANBIMA Financial Hub concentra-se nas superfícies efetivamente presentes na arquitetura: **administração dos equipamentos, segmentação Layer 2, roteamento, redistribuição entre protocolos e disponibilidade da WAN**.

Os principais mecanismos defensivos implementados no cenário são:

```text
SSHv2
RSA 2048
Autenticação Local
Privilégio 15
Enable Secret
Timeout SSH
Limite de Authentication Retries
VLAN 99
Allowed VLAN
OSPFv2
eBGPv4
Redistribuição de Rotas
Rotas Estáticas Flutuantes
Rotas de Retorno
WAN Redundante
```

O resultado é uma arquitetura em que as principais ameaças consideradas pelo próprio projeto possuem mecanismos técnicos correspondentes de redução de exposição ou de impacto, mantendo o escopo compatível com a implementação acadêmica realizada no Cisco Packet Tracer.

---

## 📁 Arquivos Relacionados

| Arquivo                             | Responsabilidade                                                             |
| ----------------------------------- | ---------------------------------------------------------------------------- |
| `README.md`                         | Visão geral e apresentação da infraestrutura                                 |
| `01-architecture-and-addressing.md` | Arquitetura física, lógica, VLANs e endereçamento                            |
| `02-routing-and-resilience.md`      | OSPF, eBGP, redistribuição e contingência                                    |
| `03-design-decisions.md`            | Decisões e justificativas arquiteturais                                      |
| `04-limitations-and-lessons.md`     | Limitações, problemas encontrados e aprendizados                             |
| `compliance-mapping.md`             | Relação dos controles técnicos implementados com seus objetivos de segurança |
| `threat-modeling.md`                | Modelo de ameaças e controles defensivos da arquitetura                      |
