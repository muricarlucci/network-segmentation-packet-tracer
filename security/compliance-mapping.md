# Compliance Mapping — ANBIMA Financial Hub

![Security Baseline](https://img.shields.io/badge/Security-Hardening-red)
![Network Segmentation](https://img.shields.io/badge/Network-VLAN%20Segmentation-blue)
![Secure Management](https://img.shields.io/badge/Management-SSHv2-success)
![Resilience](https://img.shields.io/badge/Resilience-WAN%20Failover-orange)

Este documento apresenta o **mapeamento dos controles de segurança e infraestrutura efetivamente implementados no cenário do ANBIMA Financial Hub**.

O objetivo é relacionar cada mecanismo técnico documentado no cenário ao seu respectivo propósito de segurança, identificando também **onde o controle está implementado** e **qual evidência técnica demonstra sua existência**.

> **Escopo:** este documento é baseado exclusivamente na implementação descrita no cenário oficial do projeto. O cenário estabelece critérios de conformidade e segurança da informação, mas não apresenta uma matriz normativa formal que associe cada configuração individual a artigos específicos de regulamentações ou frameworks externos. Portanto, este documento não declara certificação ou conformidade jurídica integral com uma norma; ele documenta a relação entre os controles técnicos implementados e seus objetivos de segurança.

---

## 1. Contexto de Segurança do Projeto

O cenário estabelece uma camada específica de **Hardening e Segurança Operacional de Infraestrutura**.

Segundo o cenário, os **7 ativos gerenciáveis** da topologia possuem parâmetros de hardening ativos, incluindo controles relacionados a:

* proteção do acesso administrativo;
* eliminação do Telnet;
* utilização de SSHv2;
* criptografia assimétrica;
* autenticação local;
* proteção das credenciais privilegiadas;
* limitação de tentativas de autenticação;
* encerramento de sessões inativas;
* segmentação da rede de gerenciamento.

A arquitetura também utiliza segmentação por VLAN, roteamento Layer 3, mecanismos de contingência e uma malha WAN em anel.

Esses elementos não constituem controles independentes e isolados: fazem parte de uma arquitetura na qual **segmentação, controle administrativo, roteamento e disponibilidade** trabalham conjuntamente.

---

## 2. Matriz de Controles Implementados

| Controle técnico                   | Implementação no cenário                             | Objetivo de segurança                                              |
| :--------------------------------- | :--------------------------------------------------- | :----------------------------------------------------------------- |
| **SSHv2**                          | `ip ssh version 2`                                   | Substituir o acesso administrativo por protocolos legados          |
| **Desativação do Telnet**          | `transport input ssh`                                | Evitar gerenciamento remoto utilizando Telnet                      |
| **Chaves RSA**                     | RSA de 2048 bits                                     | Disponibilizar criptografia assimétrica para o acesso SSH          |
| **Autenticação local**             | `login local`                                        | Utilizar credenciais armazenadas localmente no equipamento         |
| **Privilégio administrativo**      | `username admin privilege 15`                        | Definir o nível administrativo da conta utilizada no gerenciamento |
| **Proteção da senha privilegiada** | `enable secret`                                      | Evitar utilização de `enable password`                             |
| **Timeout SSH**                    | `ip ssh time-out 60`                                 | Limitar o tempo de espera de uma sessão SSH                        |
| **Limitação de autenticação**      | `ip ssh authentication-retries 4`                    | Limitar tentativas incorretas de autenticação                      |
| **VLAN de gerenciamento**          | VLAN 99                                              | Separar o gerenciamento dos switches do tráfego departamental      |
| **Segmentação departamental**      | VLANs 10, 20, 30 e 40 na Matriz; VLANs 10 e 20 no RJ | Separar os diferentes segmentos corporativos                       |
| **Restrição de VLANs no trunk**    | `switchport trunk allowed vlan`                      | Limitar as VLANs transportadas pelos enlaces trunk                 |
| **Roteamento Layer 3**             | `ip routing` nos switches Core                       | Centralizar o roteamento entre as redes locais                     |
| **Redundância WAN**                | Malha serial em anel                                 | Disponibilizar caminhos alternativos entre os sites                |
| **Rotas estáticas flutuantes**     | AD `115`                                             | Disponibilizar caminhos de contingência                            |
| **Rotas de retorno**               | Rotas estáticas no CPD para os blocos do RJ          | Permitir o retorno do tráfego pela WAN 2 durante contingência      |

---

## 3. Controle de Acesso Administrativo

### 3.1 SSHv2

O acesso remoto aos equipamentos é realizado exclusivamente por SSH.

A configuração utiliza:

```cisco
ip ssh version 2
```

e:

```cisco
line vty 0 4
 login local
 transport input ssh
```

O cenário determina que o Telnet seja desativado nas portas de controle e que o SSHv2 seja utilizado como protocolo de gerenciamento remoto.

### Objetivo

Reduzir a exposição das credenciais administrativas durante o gerenciamento remoto e evitar a utilização do Telnet como mecanismo de acesso aos equipamentos.

---

## 4. Criptografia e Identidade dos Equipamentos

Cada equipamento gera um par de chaves RSA de **2048 bits** associado ao domínio institucional:

```text
anbima.corp
```

A configuração aparece no cenário como:

```cisco
ip domain-name ANBIMA.CORP
crypto key generate rsa
2048
```

Esse conjunto é utilizado como parte da preparação do SSH nos equipamentos.

### Objetivo

Estabelecer a infraestrutura criptográfica necessária para o gerenciamento remoto através de SSH.

---

## 5. Autenticação Local e Privilégio Administrativo

O cenário utiliza autenticação baseada no banco de dados local dos equipamentos:

```cisco
username admin privilege 15 secret ANBIMA@SEC2026!
```

e:

```cisco
login local
```

Dessa forma, as sessões administrativas são associadas a uma conta local com **privilégio 15**.

### Objetivo

Controlar o acesso administrativo aos equipamentos através de credenciais locais e definir explicitamente o nível de privilégio da conta.

---

## 6. Proteção das Credenciais Privilegiadas

O cenário remove explicitamente o método `enable password`:

```cisco
no enable password
```

e utiliza:

```cisco
enable secret cisco
```

O próprio cenário esclarece que a senha `cisco` foi utilizada propositalmente no laboratório para fins de demonstração acadêmica. Para um ambiente real, o documento estabelece uma senha de maior robustez como referência.

### Objetivo

Evitar a utilização do mecanismo `enable password` e utilizar `enable secret` para proteger o acesso ao modo privilegiado.

> **Observação:** a credencial utilizada no laboratório não deve ser interpretada como uma recomendação de senha para produção.

---

## 7. Proteção contra Tentativas de Autenticação

O cenário configura:

```cisco
ip ssh time-out 60
ip ssh authentication-retries 4
```

Esses parâmetros representam:

* timeout de sessão de 60 segundos;
* limite de 4 tentativas incorretas de autenticação.

O objetivo descrito no cenário é fornecer proteção adicional contra tentativas repetidas de autenticação e limitar sessões administrativas inativas.

---

## 8. Segmentação por VLAN

A arquitetura utiliza VLANs para separar os diferentes segmentos da infraestrutura.

### Matriz SP

| VLAN | Segmento           |
| :--: | :----------------- |
|  10  | `SEC_OPERATIONS`   |
|  20  | `FINANCIAL_CORE`   |
|  30  | `ANALYTICS_DATA`   |
|  40  | `AUDIT_COMPLIANCE` |
|  99  | `MGMT_HARDENING`   |
|  200 | `TRANSIT_L3_WAN`   |

A configuração do Core da Matriz demonstra explicitamente essas VLANs.

### Filial RJ

| VLAN | Segmento         |
| :--: | :--------------- |
|  10  | `RJ_SUPERVISAO`  |
|  20  | `RJ_OPERATIONS`  |
|  99  | `RJ_MGMT`        |
|  200 | Trânsito Layer 3 |

A VLAN 99 utiliza o bloco `172.19.99.0/24`, enquanto as VLANs departamentais utilizam os blocos `172.19.0.0/23` e `172.19.2.0/23`.

---

## 9. VLAN de Gerenciamento

A VLAN 99 possui função específica de gerenciamento.

Na Matriz:

```text
VLAN 99
172.16.99.0/24
Gateway: 172.16.99.1
Switch: 172.16.99.2
```

Na Filial RJ:

```text
VLAN 99
172.19.99.0/24
Gateway: 172.19.99.1
Switch: 172.19.99.2
```

O cenário identifica explicitamente a VLAN 99 como **Gerência Out-of-Band** nos dois sites.

### Objetivo

Manter o endereçamento utilizado para gerenciamento dos switches separado das redes departamentais destinadas às estações de trabalho.

---

## 10. Restrição dos Enlaces Trunk

Os enlaces entre os switches de acesso e os respectivos Core transportam somente as VLANs necessárias.

Na Matriz:

```cisco
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40,99
```

Na Filial RJ:

```cisco
switchport mode trunk
switchport trunk allowed vlan 10,20,99
```

Essas configurações aparecem diretamente nos scripts do cenário.

### Objetivo

Restringir explicitamente quais VLANs podem atravessar os enlaces trunk.

---

## 11. Roteamento Layer 3

Os switches Core utilizam roteamento Layer 3 através de:

```cisco
ip routing
```

As interfaces virtuais de cada VLAN funcionam como gateways dos respectivos segmentos.

Na Matriz, por exemplo:

```text
VLAN 10 → 172.16.0.1
VLAN 20 → 172.16.4.1
VLAN 30 → 172.16.8.1
VLAN 40 → 172.16.12.1
VLAN 99 → 172.16.99.1
VLAN 200 → 172.16.16.1
```

O cenário também utiliza OSPF para anunciar as redes internas.

### Objetivo

Concentrar as funções de gateway e roteamento interno nos switches Core.

---

## 12. Disponibilidade e Continuidade de Comunicação

A arquitetura possui três enlaces WAN:

```text
Matriz SP ↔ Filial RJ
Filial RJ ↔ CPD
CPD ↔ Matriz SP
```

O cenário utiliza uma topologia fechada em anel e uma distribuição cíclica de DCE/DTE:

```text
Matriz SP → WAN 1 → Filial RJ
Filial RJ → WAN 2 → CPD
CPD → WAN 3 → Matriz SP
```

Cada roteador possui uma interface DCE com `clock rate 64000` e uma interface DTE.

### Objetivo

Disponibilizar caminhos alternativos de comunicação entre os sites e permitir a utilização de mecanismos de contingência.

---

## 13. Rotas Estáticas Flutuantes

A Filial RJ possui rotas estáticas com **Distância Administrativa 115** para determinados destinos corporativos.

Entre elas:

```cisco
ip route 172.16.0.0 255.255.240.0 10.0.0.1 115
ip route 172.16.99.0 255.255.255.0 10.0.0.1 115
ip route 172.16.32.0 255.255.252.0 10.0.0.6 115
ip route 192.168.100.1 255.255.255.255 10.0.0.6 115
```

O cenário descreve essas rotas como mecanismos de contingência para falhas no enlace primário ou no processo de roteamento OSPF.

### Objetivo

Manter caminhos alternativos disponíveis quando as rotas primárias deixam de ser utilizadas.

---

## 14. Rotas de Retorno no CPD

O Datacenter possui rotas estáticas para os blocos da Filial RJ:

```cisco
ip route 172.19.0.0 255.255.252.0 10.0.0.5
ip route 172.19.99.0 255.255.255.0 10.0.0.5
```

O próximo salto é o endereço `10.0.0.5`, correspondente à interface serial do roteador da Filial RJ na WAN 2.

O cenário explica que essas rotas permitem o retorno do tráfego para o RJ diretamente pela WAN 2 durante uma interrupção do caminho primário.

### Objetivo

Garantir que a comunicação de contingência possua também um caminho de retorno.

---

## 15. Redistribuição de Rotas

O cenário utiliza redistribuição entre OSPF e BGP no roteador de borda da Matriz.

As rotas BGP aprendidas do Datacenter são injetadas no OSPF:

```cisco
redistribute bgp 65001 subnets
```

Enquanto as rotas aprendidas via OSPF são injetadas no BGP:

```cisco
redistribute ospf 1
```

Essa integração permite que o Core da Matriz conheça os prefixos do CPD sem executar BGP internamente e permite que as redes corporativas sejam anunciadas ao Datacenter através do BGP.

### Objetivo

Integrar os diferentes domínios de roteamento utilizados na arquitetura.

---

## 16. Matriz de Rastreabilidade Técnica

| Domínio                               | Controle implementado           | Evidência no cenário                          |
| :------------------------------------ | :------------------------------ | :-------------------------------------------- |
| **Acesso administrativo**             | SSHv2 exclusivo                 | `ip ssh version 2`                            |
| **Protocolo seguro de gerenciamento** | Telnet desativado               | `transport input ssh`                         |
| **Criptografia**                      | RSA 2048 bits                   | `crypto key generate rsa`                     |
| **Autenticação**                      | Banco local de usuários         | `login local`                                 |
| **Privilégio**                        | Nível administrativo 15         | `username admin privilege 15`                 |
| **Credencial privilegiada**           | `enable secret`                 | `no enable password` + `enable secret`        |
| **Sessão administrativa**             | Timeout de 60 segundos          | `ip ssh time-out 60`                          |
| **Tentativas de login**               | Máximo de 4 tentativas          | `ip ssh authentication-retries 4`             |
| **Gerenciamento**                     | VLAN 99                         | `MGMT_HARDENING` / `RJ_MGMT`                  |
| **Segmentação**                       | VLANs departamentais            | VLANs 10, 20, 30, 40                          |
| **Controle de trunk**                 | VLANs explicitamente permitidas | `switchport trunk allowed vlan`               |
| **Roteamento interno**                | OSPF                            | `router ospf 1`                               |
| **Roteamento externo**                | eBGP                            | `router bgp 65001` / `router bgp 65002`       |
| **Contingência**                      | Rotas estáticas AD 115          | `ip route ... 115`                            |
| **Retorno de contingência**           | Rotas estáticas no CPD          | Rotas para `172.19.0.0/22` e `172.19.99.0/24` |
| **Disponibilidade WAN**               | Anel serial                     | WAN 1 + WAN 2 + WAN 3                         |
| **Sincronismo WAN**                   | DCE/DTE                         | `clock rate 64000`                            |

---

## 17. Relação com Conformidade

O cenário declara que os mecanismos de hardening foram implementados para atender a **critérios rigorosos de conformidade e segurança da informação**.

Entretanto, a documentação técnica fornecida não apresenta uma matriz normativa detalhada contendo:

* número de artigo regulatório;
* requisito textual;
* controle normativo correspondente;
* evidência exigida pela regulamentação;
* procedimento formal de auditoria;
* declaração de certificação;
* escopo jurídico de conformidade.

Por esse motivo, o mapeamento deste repositório deve ser interpretado como:

```text
REQUISITO / OBJETIVO DE SEGURANÇA
                ↓
       CONTROLE TÉCNICO
                ↓
      CONFIGURAÇÃO IOS
                ↓
        EVIDÊNCIA DO LAB
```

e não como uma declaração de conformidade regulatória integral.

---

## 18. Limites do Mapeamento

Este documento **não afirma** que a implementação:

* seja equivalente a uma infraestrutura financeira de produção;
* represente todos os controles de segurança necessários em um ambiente real;
* constitua certificação;
* constitua auditoria;
* atenda isoladamente a todos os requisitos regulatórios aplicáveis;
* substitua processos formais de governança, risco, auditoria ou segurança da informação.

O que pode ser demonstrado pelo projeto é que os controles técnicos descritos acima **foram previstos e implementados dentro do ambiente simulado apresentado no cenário**.

---

## 19. Evidências Técnicas do Projeto

As evidências visuais presentes neste repositório têm a função de demonstrar a implementação realizada no ambiente de simulação.

Entre os controles que podem ser demonstrados visualmente estão:

* adjacências OSPF;
* sessão eBGP;
* pools DHCP;
* convergência durante falha WAN;
* configuração de hardening SSHv2.

Essas evidências fazem parte da documentação do projeto e servem para demonstrar o funcionamento da implementação realizada.

---

## 20. Síntese do Compliance Mapping

A implementação do ANBIMA Financial Hub incorpora controles técnicos relacionados principalmente a:

```text
SEGMENTAÇÃO
    ↓
VLANs departamentais + VLAN de gerenciamento

GERENCIAMENTO SEGURO
    ↓
SSHv2 + RSA 2048 + autenticação local

PROTEÇÃO DE CREDENCIAIS
    ↓
enable secret + privilégio 15

CONTROLE DE ACESSO
    ↓
timeout + limite de tentativas

ROTEAMENTO
    ↓
OSPF + eBGP + redistribuição

RESILIÊNCIA
    ↓
WAN em anel + rotas estáticas flutuantes

CONTINUIDADE
    ↓
rotas de retorno + caminhos alternativos
```

O resultado é uma arquitetura em que os mecanismos de **segurança, segmentação, gerenciamento e resiliência** estão incorporados à própria configuração da infraestrutura.

> **O compliance mapping deste projeto deve ser entendido como uma matriz de rastreabilidade entre os controles técnicos implementados no laboratório e seus respectivos objetivos de segurança, mantendo como fonte primária o cenário oficial da infraestrutura.**
