# Contextualização teórica
Neste capítulo, são apresentados os principais conceitos relacionados ao *DNS Water Torture*, incluindo o funcionamento do protocolo DNS em si, a modalidade dos ataques DoS e DDoS, e como essa técnica explora a infraestrutura do sistema de nomes de domínio para sobrecarregar servidores e interromper serviços.

## Ataques DoS/DDoS

Um ataque de negação de serviço, ou DoS, tem como objetivo comprometer o pilar da disponibilidade na segurança da informação, tornando deliberadamente um sistema ou infraestrutura inacessível para usuários legítimos, por meio do esgotamento de recursos. Esse objetivo pode ser alcançado, principalmente, por meio de duas estratégias: ataques volumétricos ou abuso de protocolos de rede vulneráveis.

- **Ataques volumétricos**: enviam um grande volume de tráfego que supera a capacidade de processamento do alvo. Quando distribuídos (DDoS), vários nós coordenam o envio, aumentando a eficiência e dificultando a mitigação.  
  - **Diretos**: o tráfego vai diretamente da fonte ao alvo.  
  - **Por reflexão**: utilizam “refletores” (dispositivos que respondem ao tráfego) para amplificar o volume, enviando respostas ao IP da vítima (spoofing).  

- **Ataques de protocolo (low-and-slow)**: enviam tráfego em menor volume, porém cuidadosamente arquitetado para explorar falhas específicas dos protocolos (por exemplo, exaurir conexões ou threads), causando indisponibilidade gradual com pouco consumo de banda.

Neste trabalho, focamos em ataques DoS diretos que exploram o protocolo DNS e suas vulnerabilidades, gerando alto número de requisições por segundo.

## DNS – Domain Name System

O Sistema de Nomes de Domínio (DNS) traduz nomes legíveis (e.g., `www.example.com`) em endereços IP (e.g., `192.0.2.1`), permitindo que usuários acessem serviços sem memorizar números difíceis. Sua arquitetura é hierárquica:

1. **Servidores raiz**: apontam para os servidores de TLD (Top-Level Domain).  
2. **Servidores TLD**: indicam os servidores autoritativos de cada domínio.  
3. **Servidores autoritativos**: armazenam os registros definitivos (A, AAAA, MX, NS, etc.).  
4. **Servidores recursivos**: recebem a consulta do cliente, verificam o cache e, se necessário, fazem buscas iterativas na hierarquia até obter a resposta, armazenando-a em cache para otimizar futuras consultas.

![image](https://github.com/user-attachments/assets/25ee1f14-834b-4a95-b542-7330a87a2c37)

*Figura: Fluxo de resolução DNS.*

![image](https://github.com/user-attachments/assets/3e36832f-4630-4db0-9d39-1d620bd8d168)

*Figura: Exemplo de zonas DNS para `sales.com`.*

## Water Torture

A técnica *DNS Water Torture*, popularizada pelo Mirai, consiste em enviar subdomínios pseudoaleatórios a servidores DNS para exaurir recursos de recursão e encaminhamento:

1. O atacante gera queries com *subdomínios inexistentes* (e.g., `abcd1234.example.com`).  
2. A flag **RD** (Recursion Desired) força o recursivo a buscar nos autoritativos.  
3. Como não há resposta em cache, cada query percorre toda a hierarquia, sobrecarregando recursivos e autoritativos.  
4. Em ataques distribuídos, o impacto é exponencial.

![image](https://github.com/user-attachments/assets/c112a45a-e2a6-4480-954d-942806538a6e)
*Figura: Bytes de uma consulta gerada pela Eris.*

- **ID**: identificador da transação DNS.  
- **Flags**: RD = recursão desejada.  
- **QDCOUNT**: número de perguntas (normalmente 1).  
- **QNAME**: domínio consultado (cada rótulo precedido por seu comprimento).  
- **QTYPE**: tipo de registro (A para IPv4).  
- **QCLASS**: classe do registro (IN para Internet).

![image](https://github.com/user-attachments/assets/3656b8f5-1501-423a-ba80-9622eda5f622)
*Figura: Consultas do Mirai.*  

![image](https://github.com/user-attachments/assets/477d8a47-b9f4-45ad-838c-503c65a7c614)
*Figura: Consultas da ferramenta Eris.*  

![image](https://github.com/user-attachments/assets/de6b0079-8ab0-44db-af36-d49feee062d7)
*Figura: Fluxo da técnica Water Torture.*

## Mirai

O malware *Mirai* recrutou dispositivos IoT (câmeras, roteadores) para formar uma botnet massiva e lançar DDoS, incluindo a técnica de *DNS Water Torture*. Variantes modernas exploram subdomínios pseudoaleatórios para exacerbar o consumo de recursos nos servidores DNS, mostrando que o volume de tráfego e a exploração de falhas de protocolo são igualmente críticos.

# Arquitetura da ferramenta

![image](https://github.com/user-attachments/assets/3723d722-c793-4e90-91b8-1c5e465ca9f2)


A Eris mantém, inicialmente, a estrutura modular da DamBuster, composta por quatro módulos principais, mas realiza alterações significativas em seu funcionamento para se adequar ao novo modelo de ataque. Os detalhes sobre cada módulo são apresentados a seguir:

### Interface

Módulo em que as interfaces gráfica (GUI) e de linha de comando (CLI) são implementadas. Esse módulo é responsável por receber os argumentos que serão utilizados no ataque.

### Commander

- **commander.c**: Configura o ambiente inicial (por exemplo, definindo ações para erros fatais), inicia o fluxo principal do programa, chamando a lógica de ataque com base nos parâmetros recebidos pela interface. Também cria e sincroniza as _threads_ de ataque, garantindo o encerramento ordenado quando todas finalizam.
- **manager.c**: Atua como intermediário entre o commander e o módulo Attack.

### Attack

Módulo que contém a lógica principal do ataque _DNS Water Torture_ e também é responsável pela criação dos pacotes que serão enviados durante o ataque. Sua estrutura é dividida em dois submódulos:

- **dnsflood.c**: Gerencia o fluxo de alto nível do ataque _DNS flood_, desde a criação do primeiro pacote até o início do processo de injeção.  
- **dnsfloodforge.c**: Cuida dos detalhes de baixo nível da construção e preparação de pacotes DNS com subdomínios aleatórios, tornando cada pacote único para garantir a eficácia do ataque.

### Injector

Nesta parte do código, o injector e o controller estão implementados:

- **injector.c**: Gerencia a mecânica detalhada da injeção de pacotes, desde a criação de pacotes DNS até o envio através de sockets de rede.  
- **controller.c**: Coordena o processo geral de injeção, sincronizando os injectors, controlando o tempo e realizando ajustes dinâmicos durante o ataque.

## Funcionamento

A Eris implementa um ataque de _DNS query flood_ com opção de _spoofing_ do endereço de origem, permitindo o controle do rate dos pacotes enviados e funcionamento _multithread_. A ferramenta necessita dos parâmetros de alvo e domínio e aceita opções de configuração para rate (duration, level, increment ou RAID mode), IP de origem para _spoofing_ e porta (53 por padrão). Por default, a ferramenta envia um pacote por segundo indefinidamente.

A cada pacote enviado, um subdomínio aleatório é gerado, utilizando uma string randômica hexadecimal concatenada com o domínio base, assim como mostrado abaixo:

![image](https://github.com/user-attachments/assets/b39bcd37-1725-45aa-a3ff-176f50dff7e8)


Optou-se por gerar nomes de subdomínio hexadecimais para prover deliberadamente uma assinatura detectável para a assinatura gerada pela Eris. Dessa forma, qualquer uso indevido ou indesejável da ferramenta é facilmente identificado e bloqueado com a aplicação da assinatura.

![image](https://github.com/user-attachments/assets/83e762b6-e429-44cc-a155-5bebc9f838c9)


A estrutura do log do servidor DNS recursivo possui os campos descritos abaixo:

1. **client @0x796f513bc568**: Identificador interno do BIND que representa a estrutura com os detalhes do cliente.  
2. **116.24.140.227#53**: O endereço IP e a porta do cliente que fez a consulta DNS.  
3. **(da6be.sales.com)**: O nome de domínio solicitado.  
4. **da6be.sales.com IN A +**: Consulta DNS do tipo `A` (IPv4), classe `IN` (Internet), com flag RD ativa (`+`).  
5. **(192.168.0.4)**: O endereço IP onde a requisição foi direcionada.

O _spoofing_ pode ser observado no endereço de origem (item 2 acima), mascarando a identidade do host real. Por default, a ferramenta randomiza um IP de origem se nenhum for passado; aceita também uma lista de IPs, criando uma thread de ataque para cada IP, simulando um tráfego distribuído.

## Estratégias de Uso da Eris para Teste de Resiliência de Servidores DNS

A Eris foi desenvolvida para simular ataques do tipo DNS Water Torture de maneira altamente configurável. Três estratégias comuns:

### Teste Incremental de Carga

Inicia com taxa de envio baixa e aumenta progressivamente. Permite determinar o ponto de saturação do servidor e limites operacionais.

```
bin/eris -D dominioalvo.com -t [ip alvo] -d duração total  -l [nível de ataque] -i [quantidade de iterações até o próximo nível de ataque] -z [duração de cada iteração]

```

### Teste em Modo RAID para Máxima Vazão

Ignora parâmetros de incremento, enviando ao máximo de consultas por segundo, ideal para avaliar colapso sob alta intensidade.

```
bin/eris -D dominioalvo.com -t [ip alvo] -d duração total do ataque -r

```

### Simulação de Ataque Distribuído com Spoofing Múltiplo

Fornecendo uma lista de IPs de origem, a ferramenta cria múltiplas threads de ataque, aproximando-se do comportamento de uma botnet:

```
bin/eris -D dominioalvo.com -t [ip alvo] -d duração total do ataque -s [path para o arquivo que contêm os ips de origem] [demais argumentos]

```

O funcionamento _multithread_ segue o diagrama abaixo:

![image](https://github.com/user-attachments/assets/903bf232-2294-4ec5-bf51-8a66aa43acbe)


## Parâmetros

**Obrigatórios**  
- **Endereço de Destino**: IPv4 do alvo.  
- **Domínio**: domínio base para subdomínios.

**Opcionais**  
- **Endereço de Origem**: IP ou lista para _spoofing_.  
- **Portas**: origem (padrão 53) e destino (padrão 53).  
- **Duração**: em milissegundos (por default roda até CTRL+C).  
- **Nível de Ataque**: 1–10 (cada nível = 10^(level–1) queries por iteração).  
- **Modo Incremental**: frequência de aumento de nível.  
- **Modo RAID**: envia máxima taxa, ignorando níveis.  
- **Arquivo de Taxa**: personaliza taxas por nível.  
- **Intervalo Base**: intervalo entre iterações (padrão 1 s).

## Execução

A execução divide-se em três etapas:

### Configuração

Processa os parâmetros para criar o “draft” do plano de ataque (CLI ou GUI).

### Construção de Pacotes

- Gera cabeçalho DNS com Transaction ID aleatório, tipo `A`, classe `IN`.  
- Concatena subdomínio hexadecimal ao domínio base, transforma em etiquetas DNS.  
- Monta cabeçalhos IP/UDP com origem e destino.  
- Opcionalmente altera IP de origem para _spoofing_.  
- Empacota em “packet wrapper” antes do envio.

### Injeção de Pacotes

- O módulo commander aciona o plano criado pelo manager.  
- Controller sincroniza injectors; cada injector abre socket raw e chama `ReForgeDnsPacket` para gerar novas queries.  
- Em _multithread_ e múltiplos IPs, faz cópias profundas (`malloc` + `memcpy`) para cada origem distinta, garantindo independência dos pacotes.

## Usabilidade

### Interface por linha de comando

Saída no terminal exibe nível, pacotes enviados, drop rate, IPs de origem/destino:

![image](https://github.com/user-attachments/assets/0eac747a-ad2c-4d67-934c-6dec29c4da4f)


### Interface gráfica

- Parâmetro visualização em GUI:

![image](https://github.com/user-attachments/assets/15eef3a6-da39-4340-843e-858b2b53ef26)


- Aba **Output** (logs iguais ao CLI):

![image](https://github.com/user-attachments/assets/2e30d41d-46e1-4729-90aa-e6a2d1ba2837)


- Aba **Results** (gráfico de desempenho):

![image](https://github.com/user-attachments/assets/d37dba7d-c62b-4d32-b7aa-14ae0494c88d)


## Disponibilização

Disponível como código-fonte no GitLab e imagem Docker no Docker Hub, facilitando uso sem configurações adicionais.


# Testes realizados

## Ambiente para teste bare metal

O ambiente para teste foi montado utilizando 5 máquinas físicas, todas conectadas de forma wireless ao mesmo roteador TP-Link TL-WR829N. Para isolar a rede, o roteador foi configurado como **Access Point** isolado (sem uplink para Internet).

![image](https://github.com/user-attachments/assets/56be3c6b-b6fc-488c-93b1-a261138afd17)


Assim como no ambiente virtual, o serviço de DNS, em ambos os servidores, foi configurado usando o software BIND9. O servidor DNS recursivo possui a recursão ativada e consulta o servidor autoritativo em busca dos registros definitivos do domínio requisitado.

Os usuários legítimos foram utilizados para medir os tempos de resposta dos servidores DNS sob ataque, realizando uma _query_ DNS por segundo cada.

| Nome                         | Processador                           | Memória                |
|------------------------------|---------------------------------------|------------------------|
| Atacante                     | Intel Core i7-14700K @ 3,40 GHz       | 32 GB DDR4 @ 3600 MHz  |
| Servidor DNS recursivo       | Intel Core i5-4200U @ 1,60 GHz        | 6 GB DDR3 @ 1600 MHz   |
| Servidor DNS autoritativo    | Intel Core i3-3217U @ 1,80 GHz        | 4 GB DDR3 @ 1600 MHz   |
| Usuário 1                    | Apple M2 chip (2022)                  | 8 GB                   |
| Usuário 2                    | Intel Core i5 @ 1,40 GHz              | 8 GB DDR3 @ 2400 MHz   |
| Roteador TP-Link TL-WR829N   | Single-Core CPU                       | 32 MB DDR2             |

O ambiente bare metal foi utilizado para a execução do cenário 2 deste capítulo.

---

## Cenário de teste

Neste cenário, usou-se o **modo RAID** em ambiente bare metal. Dois usuários legítimos fizeram 1 query/s cada: um para o DNS recursivo e outro para o autoritativo.

**Objetivos do teste:**
1. **Rampa de subida ao throughput máximo:** medir o tempo para atingir a capacidade plena de envio.
2. **Realismo do tráfego:** verificar se as queries são tratadas como legítimas, forçando o recursivo a encaminhar ao autoritativo (efeito dominó).
3. **Capacidade de resiliência:** volume máximo de requisições simultâneas atendidas aos usuários legítimos.

**Coleta e métricas (via `tshark`):**

| Tráfego                         | Ponto de Captura                              | Destino                   | Objetivo da Medição                                               |
|---------------------------------|-----------------------------------------------|---------------------------|-------------------------------------------------------------------|
| Tráfego do Atacante             | Pacotes saindo da máquina do atacante         | Servidor DNS recursivo    | Quantificar requisições maliciosas geradas em modo RAID           |
| Resposta do Recursivo           | Pacotes saindo do servidor DNS recursivo      | Máquina do usuário 1      | Avaliar até quando o recursivo atende as requisições legítimas    |
| Resposta do Autoritativo        | Pacotes saindo do servidor DNS autoritativo   | Máquina do usuário 2      | Avaliar até quando o autoritativo responde às requisições legítimas |

Cada fluxo foi registrado em um PCAP separado (número de pacotes, picos de PPS, intervalos sem resposta).

**Critério de falha:**  
Falha quando o DNS deixa de responder de maneira continua a (1 req/s).

**Resultados esperados:**  
- Pico de PPS quase imediato em modo RAID  
- Tráfego malicioso superior ao do ambiente virtual  
- Efeito dominó: recursivo e autoritativo sobrecarregados  
- Tempo até falha mensurado pelas capturas de resposta  


## Resultados obtidos

O resultado da execução da ferramenta em modo RAID no ambiente bare metal pode ser observado na imagem abaixo, onde o gráfico vermelho representa o volume de pacotes que deixa a máquina do atacante e o azul representa o tráfego recebido pelo servidor DNS recursivo.

![image](https://github.com/user-attachments/assets/b6ddae0b-d881-430e-b544-f560ba90fbd8)


Pode-se observar que a ferramenta atingiu a saturação da taxa de envio de pacotes entre o segundo 0 e 1 de execução. A maior taxa observada foi de 126.791 packets/s no primeiro segundo do ataque, confirmando os resultados esperados para esse cenário de uma taxa de saturação mais alta que a do anterior.

No entanto, também foi observada uma grande discrepância entre a quantidade de pacotes enviados e a quantidade recebida pelo alvo, como mostrado na tabela abaixo:

| Pacotes                                 | Ambiente bare metal    |
|-----------------------------------------|------------------------|
| Maior taxa enviada pelo Atacante        | 126.791 packets/s      |
| Maior taxa recebida pelo Alvo           | 7.958 packets/s        |

Nesse cenário, o DNS recursivo deixa de responder ao usuário 1 logo nos primeiros segundos do ataque, conforme registrado no gráfico abaixo:

![image](https://github.com/user-attachments/assets/455c5032-5b73-457d-b5e9-c94e2924e5d8)

O servidor autoritativo deixa de responder ao usuário 2 um segundo após o recursivo, como mostrado no gráfico abaixo:

![image](https://github.com/user-attachments/assets/601d9df2-a465-430b-97ea-1e761b1cc35a)


A resposta dos servidores confirma que o tráfego malicioso está sendo propagado entre os servidores alvo.  


