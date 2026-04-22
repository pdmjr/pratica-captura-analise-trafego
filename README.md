# Prática: Captura e análise de tráfego em rede emulada com `netns`, `tc/netem`, PCAP, Prometheus e Grafana

---

## 1. Título da prática

**Avaliação de desempenho de tráfego multimídia e tráfego de fundo em uma rede emulada com network namespaces**

---

## 2. Objetivos

### 2.1 Objetivo geral

Configurar uma topologia de rede emulada com **Linux network namespaces**, gerar tráfego multimídia e tráfego de fundo, capturar pacotes em arquivos PCAP, coletar métricas com Prometheus, visualizar dados no Grafana e analisar o impacto de diferentes condições de rede sobre o desempenho observado.

### 2.2 Objetivos específicos

Ao final desta prática, o estudante deverá ser capaz de:

* configurar uma topologia de rede emulada com `ip netns`;
* interligar múltiplos nós por meio de interfaces virtuais `veth`;
* aplicar condições artificiais de rede com limitação de banda, atraso e perda de pacotes usando `tc/netem`;
* gerar tráfego de vídeo local com `ffmpeg`;
* gerar tráfego de fundo com `iperf3`;
* capturar tráfego usando ferramentas baseadas em PCAP;
* coletar métricas temporais com Prometheus;
* construir painéis de visualização no Grafana;
* correlacionar métricas de aplicação, sistema e rede com os pacotes capturados;
* interpretar o impacto das alterações da rede sobre throughput, atraso, perda e qualidade percebida da aplicação.

---

## 3. Contextualização

Aplicações multimídia e aplicações de dados competem pelos recursos da rede e podem sofrer degradação quando há gargalos, atrasos, perdas ou concorrência de tráfego. Nesta prática, será utilizado um ambiente controlado e emulado com **network namespaces**, permitindo observar, medir e analisar o comportamento do tráfego em diferentes cenários.

O experimento combina três perspectivas complementares:

* **captura de pacotes**, por meio de arquivos PCAP;
* **monitoramento temporal de métricas**, por meio de Prometheus;
* **visualização e correlação de dados**, por meio de Grafana.

---

## 4. Topologia da prática

A topologia deverá conter:

* uma **fonte de vídeo**;
* uma **fonte de tráfego de fundo**;
* um **roteador/emulador de rede**;
* **vários clientes receptores** (mínimo de 2);
* um ambiente de **monitoramento** com Prometheus e Grafana.

### 4.1 Diagrama ASCII da topologia

```text
                                     +----------------------+
                                     | Prometheus + Grafana |
                                     |  (host/monitoramento)|
                                     +----------+-----------+
                                                |
                                                |
+----------------+      +----------------+      |      +----------------+
| ns-video       |------+                |------+------|   ns-client1   |
| Fonte de vídeo |      |                |             | receptor vídeo |
|   (ffmpeg)     |      |                |             +----------------+
+----------------+      |                |
                        |   ns-router    |
+----------------+------+ Roteador /     +------------------------------+
| ns-bg          |      | Emulador Rede  |             +----------------+
| Tráfego fundo  |      | (tc / netem)   +-------------|   ns-client2   |
|   (iperf3)     |------+                |             | receptor vídeo |
+----------------+      |                |             +----------------+
                        |                |
                        |                +------------------------------+
                        |                              +----------------+
                        +------------------------------|   ns-client3   |
                                                       | receptor vídeo |
                                                       +----------------+
```

### 4.2 Interpretação da topologia

* `ns-video` envia fluxo multimídia para um ou mais clientes.
* `ns-bg` gera tráfego concorrente na rede.
* `ns-router` interliga as sub-redes e aplica atraso, perda e limitação de banda.
* `ns-client1`, `ns-client2` e `ns-client3` recebem o vídeo.
* Prometheus e Grafana podem executar no **host principal**, monitorando interfaces, processos e métricas exportadas.

---

## 5. Materiais e ferramentas

### 5.1 Infraestrutura mínima

* computador ou, preferencialmente, uma VM com Linux (testes realizados com o Ubuntu 24.04);
* privilégios administrativos;
* utilitários `iproute2`;
* capacidade de instalar Prometheus e Grafana.

### 5.2 Ferramentas obrigatórias

* **iproute2** (`ip`, `tc`)
* **network namespaces (`ip netns`)**
* **Prometheus**
* **Grafana**
* **tcpdump** ou ferramenta equivalente baseada em **PCAP/libpcap**
* **ffmpeg**
* **iperf3**

### 5.3 Ferramentas complementares

* **Wireshark** ou **tshark**
* **Scapy**
* **node_exporter**
* scripts auxiliares em bash ou Python

---

## 6. Mini explicação: como levantar o ambiente

O ambiente será montado com **network namespaces**, que funcionam como “hosts virtuais” isolados dentro do mesmo sistema Linux.

Cada namespace terá:

* suas próprias interfaces de rede;
* sua própria tabela de roteamento;
* seus próprios processos.

A ligação entre namespaces será feita por pares de interfaces **`veth`**. Cada par funciona como um “cabo virtual”: o que entra de um lado sai do outro.

### 6.1 Passos conceituais para subir o ambiente

1. Criar os namespaces:

   * `ns-video`
   * `ns-bg`
   * `ns-router`
   * `ns-client1`
   * `ns-client2`
   * `ns-client3`

2. Criar pares `veth`:

   * uma ponta fica no namespace de origem;
   * a outra ponta fica no roteador.

3. Atribuir interfaces aos namespaces.

4. Configurar endereços IP.

5. Ativar as interfaces, inclusive a `lo`.

6. Habilitar encaminhamento IP no roteador.

7. Configurar rotas padrão apontando para o roteador.

8. Aplicar `tc/netem` para modelar banda, atraso e perda.

### 6.2 Observação didática

Os Apêndices desta prática fornecem:

* uma descrição da topologia adotada;
* um script base de criação da topologia;
* um script de limpeza do ambiente;
* um script auxiliar para trocar os parâmetros do experimento.

---

## 7. Descrição do cenário experimental

A prática deverá considerar:

* transmissão de **vídeo local**, armazenado em `ns-video`;
* geração de **tráfego de fundo** concorrente em `ns-bg`;
* entrega do vídeo para múltiplos clientes;
* imposição de diferentes condições de rede em `ns-router`;
* captura de tráfego em arquivos PCAP;
* coleta e visualização de métricas de desempenho.

O foco da atividade é observar como alterações na rede afetam:

* o fluxo de vídeo;
* o tráfego concorrente;
* a qualidade percebida do serviço;
* o comportamento dos pacotes e das métricas temporais.

---

## 8. Requisitos obrigatórios da prática

A ser realizada em dupla, que deverá obrigatoriamente:

* utilizar **network namespaces** para compor a topologia;
* utilizar **Prometheus** para coleta de métricas;
* utilizar **Grafana** para visualização das métricas;
* realizar **captura PCAP** em pelo menos dois pontos da topologia;
* utilizar **ffmpeg** para geração de vídeo local;
* utilizar **iperf3** para geração de tráfego de fundo;
* aplicar alterações de rede com **tc/netem** no roteador;
* executar os experimentos definidos na matriz da prática;
* produzir análise comparativa dos resultados.

---

## 9. Procedimentos

### Parte A — Preparação do ambiente

#### Passo 1: Construção da topologia

Criar os namespaces correspondentes aos seguintes papéis:

* `ns-video`
* `ns-bg`
* `ns-router`
* `ns-client1`
* `ns-client2`
* `ns-client3`

#### Passo 2: Criação dos enlaces virtuais

Criar pares `veth` ligando:

* `ns-video` ao `ns-router`;
* `ns-bg` ao `ns-router`;
* `ns-router` a cada cliente.

#### Passo 3: Configuração de rede

Configurar:

* interfaces;
* sub-redes;
* rotas;
* encaminhamento IP no roteador.

Validar conectividade com:

* `ping`;
* opcionalmente `traceroute`.

#### Passo 4: Preparação do monitoramento

Configurar:

* Prometheus;
* Grafana;
* exportadores de métricas.

A dupla deve garantir coleta de:

* CPU;
* memória;
* tráfego por interface;
* métricas relevantes do host e dos processos.

---

### Parte B — Geração de tráfego

#### Passo 5: Geração do fluxo de vídeo

Em `ns-video`, usar `ffmpeg` para transmitir um vídeo local para os clientes.

A dupla deverá registrar:

* taxa de envio;
* formato do fluxo;
* protocolo adotado;
* duração da transmissão.

#### Passo 6: Geração do tráfego de fundo

Em `ns-bg`, usar `iperf3` para gerar tráfego concorrente.

Esse tráfego pode ser direcionado a:

* um dos clientes;
* múltiplos clientes;
* ou um servidor `iperf3` em namespace específico, conforme definido pelo professor.

---

### Parte C — Emulação de condições da rede

#### Passo 7: Aplicação de condições artificiais

Em `ns-router`, aplicar condições com `tc/netem`.

Cada experimento deverá registrar:

* banda configurada;
* atraso configurado;
* perda configurada;
* presença ou ausência de tráfego de fundo;
* duração da execução.

---

### Parte D — Captura de pacotes

#### Passo 8: Captura PCAP

Realizar captura em pelo menos dois pontos:

* em uma interface do `ns-router`;
* em um dos clientes.

Exemplos:

* `E1_router.pcap`
* `E1_client1.pcap`

A captura pode ser feita com:

```bash
sudo ip netns exec ns-router tcpdump -i r-c1 -w E1_router.pcap
```

---

### Parte E — Observabilidade

#### Passo 9: Coleta com Prometheus

Durante cada experimento, coletar:

* uso de CPU;
* uso de memória;
* bytes transmitidos e recebidos;
* pacotes transmitidos e recebidos;
* taxa de tráfego por interface.

#### Passo 10: Visualização no Grafana

Construir ao menos um dashboard contendo:

* tráfego por interface;
* CPU e memória;
* evolução temporal do experimento;
* comparação entre cenários.

---

### Parte F — Análise

#### Passo 11: Análise dos PCAPs

Usar Wireshark, tshark, Scapy ou equivalente para observar:

* volume de pacotes;
* tamanhos de pacotes;
* perdas;
* retransmissões;
* concorrência entre vídeo e tráfego de fundo.

#### Passo 12: Correlação dos resultados

Relacionar:

* evidências do PCAP;
* métricas do Prometheus;
* gráficos do Grafana;
* comportamento percebido no vídeo.

---

## 10. Métricas a serem observadas

### 10.1 Métricas de rede

* throughput;
* taxa por fluxo;
* perda observada;
* retransmissões;
* ocupação da interface.

### 10.2 Métricas do vídeo

* taxa efetiva de recepção;
* continuidade da reprodução;
* travamentos;
* degradação perceptível.

### 10.3 Métricas do sistema

* uso de CPU;
* uso de memória;
* tráfego por interface.

---

## 11. Plano de experimentação

Os experimentos deverão analisar o impacto de diferentes condições de rede sobre:

* o desempenho da rede;
* o comportamento do tráfego capturado;
* a qualidade do vídeo;
* a competição com tráfego de fundo.

### 11.1 Variáveis independentes

#### A) Largura de banda

* **100 Mbps**
* **200 Mbps**

#### B) Atraso

* **0 ms**
* **50 ms**

#### C) Perda

* **0%**
* **10%**

#### D) Tráfego de fundo

* **sem**
* **com**

### 11.2 Variáveis dependentes

#### Rede

* throughput total;
* throughput por fluxo;
* perda observada;
* retransmissões.

#### Vídeo

* taxa efetiva de recepção;
* continuidade;
* travamentos;
* degradação.

#### Sistema

* CPU;
* memória;
* tráfego por interface.

### 11.3 Procedimento de variação dos parâmetros

Cada experimento deverá configurar exatamente uma combinação de:

* banda: **100 ou 200 Mbps**;
* atraso: **0 ou 50 ms**;
* perda: **0 ou 10%**;
* tráfego de fundo: **sem ou com**.

Todos os experimentos da matriz devem ser executados.

### 11.4 Condição de referência

A condição de referência será:

* **200 Mbps**
* **0 ms**
* **0%**
* **sem tráfego de fundo**

Esse cenário corresponde ao experimento **E1**.

### 11.5 Matriz de experimentos

| Experimento |    Banda | Atraso | Perda | Tráfego de fundo |
| ----------- | -------: | -----: | ----: | ---------------- |
| E1          | 200 Mbps |   0 ms |    0% | sem              |
| E2          | 200 Mbps |   0 ms |    0% | com              |
| E3          | 200 Mbps |   0 ms |   10% | sem              |
| E4          | 200 Mbps |   0 ms |   10% | com              |
| E5          | 200 Mbps |  50 ms |    0% | sem              |
| E6          | 200 Mbps |  50 ms |    0% | com              |
| E7          | 200 Mbps |  50 ms |   10% | sem              |
| E8          | 200 Mbps |  50 ms |   10% | com              |
| E9          | 100 Mbps |   0 ms |    0% | sem              |
| E10         | 100 Mbps |   0 ms |    0% | com              |
| E11         | 100 Mbps |   0 ms |   10% | sem              |
| E12         | 100 Mbps |   0 ms |   10% | com              |
| E13         | 100 Mbps |  50 ms |    0% | sem              |
| E14         | 100 Mbps |  50 ms |    0% | com              |
| E15         | 100 Mbps |  50 ms |   10% | sem              |
| E16         | 100 Mbps |  50 ms |   10% | com              |

### 11.6 Duração e repetição

* duração mínima: **60 a 120 segundos** por execução;
* repetir cada experimento pelo menos **5 vezes**.

Registrar:

* valor médio;
* mínimo;
* máximo;
* observações.

### 11.7 Registro dos dados

| Experimento | Throughput | Perda observada | CPU roteador | CPU cliente | Qualidade do vídeo | Arquivo PCAP | Observações |
| ----------- | ---------: | --------------: | -----------: | ----------: | ------------------ | ------------ | ----------- |

Também anexar:

* capturas do Grafana;
* logs do ffmpeg;
* logs do iperf3;
* identificação dos PCAPs.

### 11.8 Diretrizes para análise comparativa

A análise deve:

* comparar cada cenário com o de referência;
* identificar o efeito da banda;
* identificar o efeito do atraso;
* identificar o efeito da perda;
* analisar o impacto do tráfego de fundo;
* justificar conclusões com base em PCAP, Prometheus, Grafana e comportamento do vídeo.

---

## 12. Perguntas para análise e discussão

1. Como a topologia foi configurada e qual a função de cada namespace?
2. Como os pares `veth` foram organizados?
3. Como o roteador foi configurado para encaminhamento?
4. Como a redução da banda afetou o vídeo?
5. Como o atraso de 50 ms afetou a aplicação?
6. Como a perda de 10% afetou o fluxo?
7. O tráfego de fundo interferiu na entrega do vídeo?
8. Qual variável teve maior impacto?
9. O que os arquivos PCAP revelaram?
10. Houve coerência entre PCAP, Prometheus e Grafana?
11. Quais métricas foram mais úteis?
12. Quais limitações existem nesse ambiente emulado?

---

## 13. Estrutura esperada do relatório

### 13.1 Introdução

Apresentação breve da prática.

### 13.2 Metodologia

Descrição:

* da topologia;
* dos namespaces;
* dos enlaces `veth`;
* dos experimentos;
* dos pontos de captura.

### 13.3 Resultados

Apresentação de:

* tabelas;
* gráficos;
* capturas do Grafana;
* trechos dos PCAPs;
* logs.

### 13.4 Discussão

Interpretação e correlação dos resultados.

### 13.5 Conclusão

Síntese dos principais achados.

---

## 14. Registro obrigatório dos resultados

Cada dupla deverá entregar:

* scripts de criação da topologia;
* scripts de limpeza;
* scripts de configuração dos experimentos;
* arquivos PCAP (pesquisar como diminuir o tamanho do arquivo antes de enviar);
* configuração do Prometheus;
* dashboard ou capturas do Grafana;
* logs do ffmpeg;
* logs do iperf3.

---

## 15. Rubrica de avaliação

### 15.1 Montagem e funcionamento da topologia — 20 pontos

### 15.2 Uso de Prometheus e Grafana — 20 pontos

### 15.3 Captura e uso de PCAP — 20 pontos

### 15.4 Execução do plano de experimentação — 20 pontos

### 15.5 Qualidade da análise e discussão — 20 pontos

**Total: 100 pontos**

---

## 16. Critérios de entrega

A dupla deverá entregar:

* relatório em PDF;
* scripts/configurações do experimento;
* arquivos PCAP;
* evidências do dashboard do Grafana;
* arquivo de configuração do Prometheus;
* instruções mínimas para reprodução.

---

## 17. Observações finais

1. O uso de **Prometheus**, **Grafana** e **PCAP** é obrigatório.
2. O cenário deve utilizar **network namespaces** e **tc/netem**.
3. O fluxo multimídia deve ser de **vídeo local**.
4. A dupla deve garantir organização dos arquivos e rastreabilidade dos experimentos.
5. Todas as conclusões devem ser sustentadas por evidências observáveis.

---

# Apêndice A — Topologia lógica adotada no script

```text
ns-video   ----\
                \
ns-bg      ----- ns-router ---- ns-client1
                |          \--- ns-client2
                |-----------\-- ns-client3
```

Cada enlace usa uma sub-rede diferente, para facilitar o roteamento e a análise.

---

# Apêndice B — Plano de endereçamento

| Enlace                 | Rede         | Lado A               | Lado B                |
| ---------------------- | ------------ | -------------------- | --------------------- |
| ns-video ↔ ns-router   | 10.0.1.0/24  | ns-video: 10.0.1.2   | ns-router: 10.0.1.1   |
| ns-bg ↔ ns-router      | 10.0.3.0/24  | ns-bg: 10.0.3.2      | ns-router: 10.0.3.1   |
| ns-router ↔ ns-client1 | 10.0.11.0/24 | ns-router: 10.0.11.1 | ns-client1: 10.0.11.2 |
| ns-router ↔ ns-client2 | 10.0.12.0/24 | ns-router: 10.0.12.1 | ns-client2: 10.0.12.2 |
| ns-router ↔ ns-client3 | 10.0.13.0/24 | ns-router: 10.0.13.1 | ns-client3: 10.0.13.2 |

---

# Apêndice C — Script base de criação do ambiente

Salve como `setup_lab_netns.sh`.

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "[1/8] Limpando ambiente anterior, se existir..."
for ns in ns-video ns-bg ns-router ns-client1 ns-client2 ns-client3; do
    sudo ip netns del "$ns" 2>/dev/null || true
done

echo "[2/8] Criando namespaces..."
for ns in ns-video ns-bg ns-router ns-client1 ns-client2 ns-client3; do
    sudo ip netns add "$ns"
    sudo ip netns exec "$ns" ip link set lo up
done

echo "[3/8] Criando pares veth..."
sudo ip link add v-vd type veth peer name r-vd
sudo ip link add v-bg type veth peer name r-bg
sudo ip link add c1 type veth peer name r-c1
sudo ip link add c2 type veth peer name r-c2
sudo ip link add c3 type veth peer name r-c3

echo "[4/8] Movendo interfaces para os namespaces..."
sudo ip link set v-vd netns ns-video
sudo ip link set r-vd netns ns-router

sudo ip link set v-bg netns ns-bg
sudo ip link set r-bg netns ns-router

sudo ip link set c1 netns ns-client1
sudo ip link set r-c1 netns ns-router

sudo ip link set c2 netns ns-client2
sudo ip link set r-c2 netns ns-router

sudo ip link set c3 netns ns-client3
sudo ip link set r-c3 netns ns-router

echo "[5/8] Configurando endereços IP..."
sudo ip netns exec ns-video ip addr add 10.0.1.2/24 dev v-vd
sudo ip netns exec ns-router ip addr add 10.0.1.1/24 dev r-vd

sudo ip netns exec ns-bg ip addr add 10.0.3.2/24 dev v-bg
sudo ip netns exec ns-router ip addr add 10.0.3.1/24 dev r-bg

sudo ip netns exec ns-client1 ip addr add 10.0.11.2/24 dev c1
sudo ip netns exec ns-router ip addr add 10.0.11.1/24 dev r-c1

sudo ip netns exec ns-client2 ip addr add 10.0.12.2/24 dev c2
sudo ip netns exec ns-router ip addr add 10.0.12.1/24 dev r-c2

sudo ip netns exec ns-client3 ip addr add 10.0.13.2/24 dev c3
sudo ip netns exec ns-router ip addr add 10.0.13.1/24 dev r-c3

echo "[6/8] Ativando interfaces..."
sudo ip netns exec ns-video ip link set v-vd up
sudo ip netns exec ns-bg ip link set v-bg up
sudo ip netns exec ns-client1 ip link set c1 up
sudo ip netns exec ns-client2 ip link set c2 up
sudo ip netns exec ns-client3 ip link set c3 up

sudo ip netns exec ns-router ip link set r-vd up
sudo ip netns exec ns-router ip link set r-bg up
sudo ip netns exec ns-router ip link set r-c1 up
sudo ip netns exec ns-router ip link set r-c2 up
sudo ip netns exec ns-router ip link set r-c3 up

echo "[7/8] Configurando rotas..."
sudo ip netns exec ns-video ip route add default via 10.0.1.1
sudo ip netns exec ns-bg ip route add default via 10.0.3.1
sudo ip netns exec ns-client1 ip route add default via 10.0.11.1
sudo ip netns exec ns-client2 ip route add default via 10.0.12.1
sudo ip netns exec ns-client3 ip route add default via 10.0.13.1

echo "[8/8] Habilitando encaminhamento IP no roteador..."
sudo ip netns exec ns-router sysctl -w net.ipv4.ip_forward=1 >/dev/null

echo
echo "Ambiente criado com sucesso."
echo "Sugestões de teste:"
echo "  sudo ip netns exec ns-video ping -c 2 10.0.11.2"
echo "  sudo ip netns exec ns-bg ping -c 2 10.0.13.2"
echo "  sudo ip netns exec ns-router ip addr"
```

---

# Apêndice D — Script de limpeza do ambiente

Salve como `cleanup_lab_netns.sh`.

```bash
#!/usr/bin/env bash
set -euo pipefail

for ns in ns-video ns-bg ns-router ns-client1 ns-client2 ns-client3; do
    sudo ip netns del "$ns" 2>/dev/null || true
done

echo "Ambiente removido."
```

---

# Apêndice E — Script para limpar regras `tc`

Salve como `clear_tc.sh`.

```bash
#!/usr/bin/env bash
set -euo pipefail

for dev in r-vd r-bg r-c1 r-c2 r-c3; do
    sudo ip netns exec ns-router tc qdisc del dev "$dev" root 2>/dev/null || true
done

echo "Regras tc removidas do ns-router."
```

---

# Apêndice F — Como executar

Dar permissão de execução:

```bash
chmod +x setup_lab_netns.sh cleanup_lab_netns.sh clear_tc.sh
```

Criar o ambiente:

```bash
bash setup_lab_netns.sh
```

Remover o ambiente:

```bash
bash cleanup_lab_netns.sh
```

Limpar regras `tc`:

```bash
bash clear_tc.sh
```

---

# Apêndice G — Como verificar se a topologia subiu corretamente

### Listar namespaces

```bash
ip netns list
```

### Ver interfaces de um namespace

```bash
sudo ip netns exec ns-router ip addr
```

### Ver rotas

```bash
sudo ip netns exec ns-client1 ip route
```

### Testar conectividade

```bash
sudo ip netns exec ns-video ping -c 4 10.0.11.2
sudo ip netns exec ns-video ping -c 4 10.0.12.2
sudo ip netns exec ns-video ping -c 4 10.0.13.2
sudo ip netns exec ns-bg ping -c 4 10.0.13.2
```

---

# Apêndice H — Exemplos de uso na prática

## H.1 Captura PCAP

No roteador:

```bash
sudo ip netns exec ns-router tcpdump -i r-c1 -w E1_router_c1.pcap
```

No cliente:

```bash
sudo ip netns exec ns-client1 tcpdump -i c1 -w E1_client1.pcap
```

## H.2 Exemplo com `iperf3`

Servidor no cliente 1:

```bash
sudo ip netns exec ns-client1 iperf3 -s
```

Cliente no namespace de tráfego de fundo:

```bash
sudo ip netns exec ns-bg iperf3 -c 10.0.11.2 -t 60
```

## H.3 Exemplo com `ffmpeg`

```bash
curl https://download.blender.org/peach/bigbuckbunny_movies/BigBuckBunny_320x180.mp4 -o video.mp4
```

Cliente 1 recebendo:

```bash
sudo ip netns exec ns-client1 ffplay udp://@:1234
```

Fonte de vídeo enviando:

```bash
sudo ip netns exec ns-video ffmpeg -re -i video.mp4 -f mpegts udp://10.0.11.2:1234
```

---

# Apêndice I — Observação sobre `tc/netem`

Para combinar **banda**, **atraso** e **perda**, os alunos devem tomar cuidado, porque um `tc qdisc replace dev ... root ...` sobrescreve a regra anterior.

Didaticamente, há três caminhos:

* variar um parâmetro por vez;
* usar scripts prontos por experimento;
* ensinar a composição com `handle`, `parent` e classes.

---

# Apêndice J — Organização sugerida dos arquivos

```text
pratica-captura-analise-trafego/
├── setup_lab_netns.sh
├── cleanup_lab_netns.sh
├── clear_tc.sh
├── videos/
│   └── video.mp4
├── capturas/
├── logs/
└── experimentos/
    ├── E1.sh
    ├── E2.sh
    └── ...
```

---
