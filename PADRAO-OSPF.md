📖 Padrão de Configuração OSPF BNG - ProvTI

Este documento define o padrão oficial da ProvTI para a configuração do protocolo OSPF em roteadores Mikrotik que atuam como BNG (Broadband Network Gateway) / Concentradores PPPoE.

🎯 Objetivo e Arquitetura

O objetivo principal desta padronização é sumarizar os blocos de clientes PPPoE para o backbone OSPF, anunciando apenas o bloco agregado (ex: /22) e bloqueando o vazamento de rotas de clientes individuais (/32).

Por que não usar o comando /routing ospf network para os clientes?
Se incluirmos o bloco dos clientes diretamente no OSPF, cada cliente PPPoE que conectar ou desconectar causará uma mudança na topologia da área. Isso obriga o OSPF a gerar novos LSAs e recalcular o algoritmo SPF, o que resulta em alto consumo de CPU e instabilidade nos concentradores.

A Solução Adotada:

Isolamento OSPF: O OSPF roda apenas em IPs de infraestrutura (Loopback e links Ponto-a-Ponto).

Ancoragem (Blackhole): Criamos uma rota estática apontando o bloco agregado para blackhole.

Filtro de Saída (Out-Filter): Redistribuímos as rotas estáticas no OSPF, mas aplicamos um filtro que permite apenas o anúncio do bloco ancorado, descartando qualquer outra rota.

⚙️ Template RouterOS v6

Abaixo o modelo de configuração para equipamentos rodando Mikrotik RouterOS v6.

# 1. Configuração da Interface Loopback
/interface bridge add name=lo
/ip address add address=10.80.255.2 interface=lo network=10.80.255.2

# 2. Rota de Ancoragem (Blackhole) do bloco de clientes
/ip route
add comment="SUMARIZAR PPPOE" distance=1 dst-address=100.66.0.0/22 type=blackhole

# 3. Filtros de Roteamento OSPF (Aceita o /22, descarta o resto)
/routing filter
add action=accept chain=ospf-out prefix=100.66.0.0/22
add action=discard chain=ospf-out

# 4. Configuração da Instância OSPF (Ativa redistribuição estática)
/routing ospf instance
set [ find default=yes ] distribute-default=if-installed-as-type-1 redistribute-static=as-type-1 router-id=10.80.255.2 out-filter=ospf-out redistribute-connected=no

# 5. Definição do Tipo de Interface OSPF
/routing ospf interface
add interface=vlan4001 network-type=point-to-point
add interface=vlan4002 network-type=point-to-point

# 6. Declaração das Redes (APENAS INFRAESTRUTURA)
/routing ospf network
add area=backbone network=10.80.255.2/32 comment="Loopback"
add area=backbone network=10.80.254.0/30 comment="PTP VLAN4001"
add area=backbone network=10.80.254.4/30 comment="PTP VLAN4002"


🚀 Template RouterOS v7

O Mikrotik v7 possui mudanças drásticas na sintaxe de filtros e na configuração de interfaces OSPF (uso de templates). O modelo abaixo aplica a mesma lógica na arquitetura v7.

# 1. Configuração da Interface Loopback
/interface bridge add name=lo
/ip address add address=10.80.255.2 interface=lo network=10.80.255.2

# 2. Rota de Ancoragem (Blackhole) do bloco de clientes
/ip route
add comment="SUMARIZAR PPPOE v7" distance=1 dst-address=100.66.0.0/22 type=blackhole

# 3. Filtros de Roteamento OSPF (Sintaxe com if/else do v7)
/routing filter rule
add chain=ospf-out rule="if (dst == 100.66.0.0/22) { accept; } else { reject; }"

# 4. Configuração da Instância OSPF
# A redistribuição agora é 'static' e o filtro é vinculado na chain
/routing ospf instance
set [ find default=yes ] redistribute=static out-filter-chain=ospf-out router-id=10.80.255.2

# 5. Configuração das Áreas e Interfaces OSPF (Uso de Templates)
/routing ospf area
set [ find default=yes ] instance=default name=backbone

# Define a loopback como passiva (não envia Hello, mas anuncia o IP)
/routing ospf interface-template
add area=backbone interfaces=lo type=passive

# Define as VLANs de PTP e suas respectivas redes
/routing ospf interface-template
add area=backbone interfaces=vlan4001 networks=10.80.254.0/30 type=ptp
add area=backbone interfaces=vlan4002 networks=10.80.254.4/30 type=ptp


⚠️ Boas Práticas e Pontos de Atenção

NUNCA ative a opção redistribute-connected=yes na instância OSPF de um BNG, a menos que tenha filtros estritos devidamente configurados. Isso vaza todas as rotas /32 dos clientes.

NUNCA adicione o bloco IP dos clientes (ex: 100.66.0.0/22) na aba OSPF -> Networks (v6) ou como template ativo (v7). O OSPF não deve gerenciar as interfaces dinâmicas dos clientes.

Lembre-se sempre de criar a interface Bridge vazia (chamada lo ou loopback) para associar o Router-ID do OSPF, garantindo a estabilidade do processo caso portas físicas caiam.
