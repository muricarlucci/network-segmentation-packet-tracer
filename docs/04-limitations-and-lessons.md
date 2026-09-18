# Limitations and Lessons Learned — ANBIMA Financial Hub

![Cisco Packet Tracer](https://img.shields.io/badge/Simulation-Cisco%20Packet%20Tracer-blue)
![Routing](https://img.shields.io/badge/Routing-OSPFv2%20%2B%20eBGP-orange)
![Resilience](https://img.shields.io/badge/Resilience-WAN%20Ring-success)
![Security](https://img.shields.io/badge/Security-Hardening-red)

Este documento registra as principais **limitações técnicas, restrições do ambiente de simulação e lições aprendidas** durante o desenvolvimento do projeto de infraestrutura de rede corporativa da ANBIMA.

Diferentemente dos documentos de arquitetura, endereçamento e roteamento, este arquivo não tem como objetivo apenas descrever **o que foi configurado**. Seu propósito é registrar **o que a implementação ensinou**, quais pontos exigem atenção em uma infraestrutura real e quais problemas podem surgir quando alterações aparentemente simples são realizadas em uma topologia interdependente.

> **Nota:** as observações deste documento representam limitações identificadas no contexto do laboratório e conclusões técnicas obtidas a partir da implementação. Elas não devem ser interpretadas como requisitos formais da ANBIMA nem como afirmações de que o cenário simulado reproduz integralmente uma infraestrutura financeira de produção.

---

## 1. Limitações Técnicas e do Ambiente de Simulação

### 1.1 Fidelidade do Cisco Packet Tracer

O projeto foi desenvolvido em ambiente de simulação utilizando equipamentos Cisco representados no Cisco Packet Tracer.

Embora o ambiente permita reproduzir conceitos importantes de redes corporativas — como VLANs, roteamento Layer 3, OSPF, eBGP, enlaces seriais, DHCP, SSH e mecanismos de contingência — uma simulação não representa necessariamente todos os comportamentos, recursos e particularidades existentes em equipamentos físicos de produção.

Isso deve ser considerado principalmente quando o conhecimento obtido no laboratório é transportado para ambientes reais.

Entre os pontos que podem apresentar diferenças estão:

* disponibilidade de determinados comandos;
* comportamento específico de versões do Cisco IOS;
* nomenclatura e organização das interfaces;
* recursos disponíveis em diferentes modelos de equipamentos;
* implementação de funcionalidades avançadas de roteamento;
* comportamento de hardware e módulos físicos;
* características de desempenho;
* recursos que dependem de licenciamento ou plataformas específicas.

Um exemplo é a diferença entre nomenclaturas de interfaces encontradas em diferentes famílias de equipamentos, como `Gig0/1` e `Gig1/0/1`.

Também não se deve assumir que uma funcionalidade ausente ou simplificada no ambiente de simulação não exista em equipamentos físicos, ou que uma funcionalidade simulada opere exatamente da mesma maneira em produção.

### 1.2 Escopo da Simulação

A topologia representa uma infraestrutura corporativa composta por:

* Matriz SP;
* Filial Regional RJ;
* Datacenter/CPD;
* roteadores de borda;
* switches multicamada;
* switches de acesso;
* estações finais;
* enlaces WAN seriais;
* protocolos de roteamento interno e externo.

O objetivo do laboratório é demonstrar os conceitos de arquitetura, segmentação, endereçamento, roteamento e resiliência previstos no projeto.

Consequentemente, a implementação não deve ser interpretada como uma reprodução completa de todos os componentes encontrados em uma infraestrutura corporativa financeira real.

---

## 2. Dependência de Enlaces Seriais Legados no Laboratório

A comunicação entre os três pontos da topologia utiliza uma malha WAN serial em anel com módulos `HWIC-2T`.

O cenário estabelece um modelo cíclico de DCE/DTE:

| Enlace | Ponta DCE             | Ponta DTE             |
| ------ | --------------------- | --------------------- |
| WAN 1  | Matriz SP — `Se0/3/0` | Filial RJ — `Se0/3/1` |
| WAN 2  | Filial RJ — `Se0/3/0` | CPD — `Se0/3/1`       |
| WAN 3  | CPD — `Se0/3/0`       | Matriz SP — `Se0/3/1` |

Cada interface DCE utiliza `clock rate 64000`, mantendo o sincronismo necessário para o funcionamento dos enlaces seriais no laboratório.

Esse modelo é particularmente útil para aprendizado porque torna visíveis conceitos que poderiam permanecer abstratos em uma infraestrutura baseada exclusivamente em interfaces Ethernet.

Em uma infraestrutura corporativa moderna, entretanto, a tecnologia física de WAN pode ser completamente diferente. Tecnologias como enlaces Ethernet metropolitanos, MPLS, túneis sobre IP e arquiteturas SD-WAN podem substituir o modelo serial utilizado neste laboratório.

**Lição:** compreender DCE, DTE e sincronismo de clock continua sendo importante para entender a camada física e o funcionamento de enlaces síncronos, mesmo quando a tecnologia utilizada em produção é diferente.

---

## 3. Segurança de Borda e Escopo do Laboratório

O projeto possui mecanismos de hardening e controles de segurança implementados diretamente nos equipamentos Cisco.

Entre os controles documentados estão:

* desativação do Telnet;
* utilização de SSH;
* SSH versão 2;
* chaves RSA de 2048 bits;
* domínio `anbima.corp`;
* autenticação local;
* privilégio administrativo nível 15;
* `enable secret`;
* timeout de sessão SSH;
* limitação de tentativas de autenticação;
* VLAN de gerenciamento.

O cenário informa que esses parâmetros são aplicados aos 7 ativos gerenciáveis da topologia.

Entretanto, o projeto não representa uma arquitetura completa de segurança de borda baseada em appliances dedicados de próxima geração.

Não fazem parte do escopo da implementação elementos como:

* Next-Generation Firewall;
* UTM;
* inspeção profunda de pacotes;
* IDS/IPS dedicado;
* proxy corporativo;
* solução SIEM;
* NAC;
* EDR;
* WAF;
* mecanismos avançados de filtragem de aplicações.

Portanto, os controles de segurança demonstrados devem ser entendidos dentro do **escopo de infraestrutura de rede e laboratório Cisco**.

**Lição:** hardening de dispositivos de rede é importante, mas não deve ser confundido com uma arquitetura completa de segurança corporativa.

---

## 4. Hardening e a Importância da Sintaxe Correta

Uma das lições mais importantes do projeto está relacionada à diferença entre uma configuração que simplesmente funciona e uma configuração que segue uma prática de segurança adequada.

O cenário utiliza explicitamente:

```text
NO ENABLE PASSWORD
ENABLE SECRET CISCO
```

O `enable secret` foi utilizado em substituição ao `enable password`, evitando o armazenamento da senha privilegiada no formato de texto claro. O próprio cenário ressalta que a senha `cisco` foi utilizada propositalmente no laboratório para demonstração acadêmica.

Esse ponto evidencia uma questão importante:

> **A sintaxe de um comando de configuração não é apenas uma questão operacional; ela pode alterar diretamente o nível de segurança do equipamento.**

O mesmo princípio aparece na configuração de acesso remoto:

```text
transport input ssh
ip ssh version 2
```

O laboratório, portanto, reforça a necessidade de reconhecer comandos legados e compreender suas implicações antes de simplesmente reproduzir configurações encontradas em materiais antigos.

---

## 5. O Perigo de Alterações Isoladas no Plano de Endereçamento

Uma das principais lições de engenharia de redes observadas durante o desenvolvimento está relacionada ao relacionamento entre **endereçamento e roteamento**.

A rede utiliza diferentes blocos para os sites e também possui uma rede específica para gerenciamento:

* Matriz: `172.16.99.0/24`;
* Filial: `172.19.99.0/24`.

Ao mesmo tempo, algumas rotas de contingência utilizam superblocos departamentais, enquanto as redes de gerenciamento possuem prefixos específicos.

Isso cria um ponto de atenção importante.

Uma alteração aparentemente simples no endereço de uma VLAN pode fazer com que uma rota anteriormente abrangente deixe de alcançar determinado segmento.

### Lição

> **Qualquer alteração no plano de endereçamento deve ser acompanhada por uma auditoria completa das tabelas de roteamento.**

Não basta verificar se:

```text
IP + máscara
```

estão corretos.

Também é necessário verificar:

```text
IP
↓
máscara
↓
gateway
↓
rota local
↓
rota dinâmica
↓
rota estática
↓
rota de contingência
↓
rota de retorno
```

Uma mudança de prefixo pode afetar vários pontos diferentes da topologia simultaneamente.

---

## 6. O Princípio da Rota Bidirecional

Outra lição fundamental surgiu da análise das rotas de contingência.

O projeto possui mecanismos para permitir que a Filial RJ alcance a Matriz e o CPD por caminhos alternativos. Entre eles estão as rotas estáticas flutuantes com distância administrativa `115`.

Porém, uma rota de ida não é suficiente.

Para que uma comunicação seja efetivamente estabelecida, o caminho de retorno também precisa existir.

O CPD possui rotas estáticas reversas para os blocos regionais:

```text
ip route 172.19.0.0 255.255.252.0 10.0.0.5
ip route 172.19.99.0 255.255.255.0 10.0.0.5
```

Essas rotas permitem que o tráfego destinado à Filial RJ retorne diretamente pela WAN 2 quando necessário.

### Lição

> **Roteamento deve sempre ser analisado nos dois sentidos.**

Ao testar uma comunicação, não basta perguntar:

```text
"Existe rota até o destino?"
```

Também é necessário perguntar:

```text
"O destino possui rota de volta para a origem?"
```

Essa verificação é especialmente importante em ambientes com múltiplos caminhos, redistribuição e rotas estáticas de contingência.

---

## 7. Redistribuição entre OSPF e eBGP

O projeto utiliza dois domínios de roteamento com funções diferentes:

* **OSPF** para o roteamento interno;
* **eBGP** para a comunicação entre os sistemas autônomos representados na topologia.

O roteador de borda da Matriz realiza a integração entre esses domínios.

As rotas aprendidas via BGP são injetadas no OSPF:

```text
redistribute bgp 65001 subnets
```

Enquanto as rotas aprendidas via OSPF são redistribuídas para o BGP:

```text
redistribute ospf 1
```

Essa integração permite que o Core da Matriz conheça as redes do CPD sem precisar executar BGP internamente, enquanto as redes da Matriz podem ser anunciadas ao CPD através do domínio BGP.

### Lição

A redistribuição entre protocolos diferentes aumenta a flexibilidade da arquitetura, mas também aumenta sua complexidade.

Ao utilizar redistribuição, é necessário compreender:

* qual protocolo possui determinada rota;
* em qual equipamento ocorre a redistribuição;
* qual prefixo está sendo anunciado;
* para qual domínio ele será exportado;
* qual caminho será utilizado no retorno;
* quais rotas podem ser propagadas desnecessariamente.

A existência de dois protocolos de roteamento não significa simplesmente que "um passa tudo para o outro".

A fronteira entre os domínios precisa ser compreendida e validada.

---

## 8. Resiliência Não Significa Apenas Ter Mais de Um Cabo

A topologia foi estruturada como uma malha WAN em anel:

```text
                 WAN 3
          +----------------+
          |                |
          v                |
      [ MATRIZ ]        [ CPD ]
          |                ^
          |                |
        WAN 1            WAN 2
          |                |
          v                |
       [ FILIAL ]----------+
```

A existência dos três enlaces permite que uma interrupção em um dos caminhos não necessariamente elimine a conectividade entre todos os pontos.

O cenário também utiliza rotas estáticas de contingência com AD `115` para determinados destinos.

### Lição

> **Resiliência é uma propriedade do conjunto da arquitetura, não de um único dispositivo ou enlace.**

Para considerar um caminho realmente resiliente, é necessário analisar:

* caminho primário;
* caminho alternativo;
* protocolo responsável pela rota;
* distância administrativa;
* próximo salto;
* rota de retorno;
* comportamento dos equipamentos intermediários.

Ter um segundo enlace fisicamente conectado não garante, por si só, que o tráfego utilizará esse caminho corretamente.

---

## 9. DCE, DTE e o Modelo Cíclico de Clock

A implementação da WAN adotou uma distribuição cíclica de DCE/DTE:

```text
Matriz SP
Se0/3/0 → DCE
      ↓
WAN 1
      ↓
Filial RJ
Se0/3/1 → DTE

Filial RJ
Se0/3/0 → DCE
      ↓
WAN 2
      ↓
CPD
Se0/3/1 → DTE

CPD
Se0/3/0 → DCE
      ↓
WAN 3
      ↓
Matriz SP
Se0/3/1 → DTE
```

Essa organização garante que cada enlace possua uma ponta responsável pelo fornecimento do clock e outra responsável pela recepção.

O cenário utiliza `clock rate 64000` nas interfaces DCE.

### Lição

A configuração física precisa ser analisada junto com a configuração lógica.

Uma topologia pode estar corretamente desenhada no nível lógico e ainda apresentar falha no nível físico se:

* os papéis DCE/DTE estiverem invertidos;
* nenhuma ponta fornecer clock;
* duas pontas DCE forem conectadas incorretamente;
* a interface estiver administrativamente desativada;
* o endereçamento do enlace estiver incorreto.

---

## 10. Autonomia da Filial e Dependências Externas

A Filial RJ possui seu próprio Switch Core Layer 3, com SVIs, DHCP e OSPF regional.

Essa arquitetura permite que determinadas funções de rede sejam realizadas localmente, sem depender exclusivamente da Matriz para o funcionamento da LAN.

Ao mesmo tempo, a conectividade com os demais sites depende da WAN e dos mecanismos de roteamento entre os domínios.

O cenário também utiliza uma rota padrão no `Branch-Core-3650` apontando para `172.19.4.2`, além da redistribuição das rotas estáticas de contingência pelo roteador regional.

### Lição

É importante distinguir:

**autonomia local**

de

**independência completa da infraestrutura corporativa**.

Uma filial pode possuir DHCP, gateways, VLANs e roteamento interno próprios e ainda depender de caminhos WAN para acessar recursos externos ao seu próprio domínio.

---

## 11. Validação Deve Acompanhar a Implementação

Outra lição importante é que uma configuração não deve ser considerada concluída simplesmente porque os comandos foram aceitos pelo IOS.

A validação precisa confirmar o comportamento esperado.

No contexto deste projeto, isso envolve verificar, entre outros pontos:

```text
VLANs
SVIs
DHCP
interfaces
adjacências OSPF
vizinhança BGP
rotas aprendidas
rotas estáticas
rotas de contingência
conectividade entre sites
SSH
```

A diferença é fundamental:

```text
Configuração aceita pelo equipamento
                ≠
Funcionamento comprovado da arquitetura
```

### Lição

> **Todo mecanismo de rede deve possuir uma forma objetiva de validação.**

A implementação de uma rota estática, por exemplo, deve ser acompanhada da verificação da tabela de roteamento e de testes de conectividade.

Da mesma maneira, configurar OSPF não significa automaticamente que a adjacência esteja estabelecida.

---

## 12. Documentação como Parte da Engenharia

O desenvolvimento do projeto também demonstrou que uma infraestrutura de rede não deve ser documentada apenas através de comandos.

A configuração representa **como o equipamento foi configurado**.

A documentação arquitetural representa **como a infraestrutura foi organizada**.

A documentação de decisões representa **por que determinadas escolhas foram realizadas**.

Este arquivo representa **quais limitações e aprendizados surgiram durante o desenvolvimento**.

Por isso, o repositório foi organizado em documentos com responsabilidades diferentes:

| Documento                           | Responsabilidade                               |
| ----------------------------------- | ---------------------------------------------- |
| `01-architecture-and-addressing.md` | Arquitetura, ativos, VLANs e endereçamento     |
| `02-routing-and-resilience.md`      | OSPF, eBGP, WAN, redistribuição e contingência |
| `03-design-decisions.md`            | Decisões e critérios arquiteturais             |
| `04-limitations-and-lessons.md`     | Limitações, problemas e lições aprendidas      |

Essa separação evita transformar um único README em uma documentação excessivamente extensa e difícil de consultar.

---

## 13. Principais Lições Consolidadas

| Área           | Lição                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| Simulação      | O comportamento do Packet Tracer não deve ser tratado como reprodução integral de hardware de produção |
| Hardware       | A nomenclatura e os recursos das interfaces podem variar entre plataformas                             |
| WAN            | DCE, DTE e clock são conceitos importantes para compreender enlaces seriais                            |
| Segurança      | Configurações que funcionam podem não representar práticas adequadas de segurança                      |
| Hardening      | `enable secret` e SSHv2 são exemplos de decisões de configuração com impacto direto na segurança       |
| Endereçamento  | Alterar um prefixo pode afetar rotas estáticas, dinâmicas e de retorno                                 |
| Roteamento     | Toda comunicação deve ser analisada nos sentidos de ida e volta                                        |
| Redistribuição | OSPF e eBGP exigem compreensão clara das fronteiras entre domínios                                     |
| Resiliência    | Um caminho alternativo precisa estar logicamente configurado, não apenas fisicamente conectado         |
| Validação      | Configuração concluída não significa funcionamento comprovado                                          |
| Documentação   | Arquitetura, decisões, configuração e aprendizados possuem objetivos diferentes                        |

---

## 14. Conclusão

O principal aprendizado deste projeto não está apenas na configuração individual de VLANs, interfaces ou protocolos de roteamento.

A maior lição está na **interdependência entre os elementos da infraestrutura**.

Uma alteração no endereçamento pode afetar o roteamento.

Uma alteração no roteamento pode afetar a contingência.

Uma alteração na contingência pode revelar a ausência de uma rota de retorno.

Uma alteração física pode afetar o funcionamento lógico.

Uma configuração de segurança aparentemente simples pode modificar o nível de proteção do equipamento.

E uma configuração que foi aceita pelo IOS ainda precisa ser validada operacionalmente.

O laboratório, portanto, funciona não apenas como uma implementação de uma topologia corporativa, mas como um exercício de **raciocínio sistêmico de redes**.

> **A principal lição é compreender a relação entre cada camada da infraestrutura antes de modificar qualquer componente isoladamente.**
