# Contextualização teórica
Neste capítulo, são apresentados os principais conceitos relacionados ao *DNS Water Torture*, incluindo o funcionamento do protocolo DNS em si, a modalidade dos ataques DoS e DDoS, e como essa técnica explora a infraestrutura do sistema de nomes de domínio para sobrecarregar servidores e interromper serviços.

## Ataques DoS/DDoS

Um ataque de negação de serviço, ou DoS, tem como objetivo comprometer o pilar da disponibilidade na segurança da informação, tornando deliberadamente um sistema ou infraestrutura inacessível para usuários legítimos, por meio do esgotamento de recursos. Esse objetivo pode ser alcançado, principalmente, por meio de duas estratégias: ataques volumétricos ou abuso de protocolos de rede vulneráveis.

- **Ataques volumétricos**: enviam um grande volume de tráfego que supera a capacidade de processamento do alvo. Quando distribuídos (DDoS), vários nós coordenam o envio, aumentando a eficiência e dificultando a mitigação.  
  - **Diretos**: o tráfego vai diretamente da fonte ao alvo.  
  - **Por reflexão**: utilizam “refletores” (dispositivos que respondem ao tráfego) para amplificar o volume, enviando respostas ao IP da vítima (spoofing).  

- **Ataques de protocolo (low-and-slow)**: enviam tráfego em menor volume, porém cuidadosamente arquitetado para explorar falhas específicas dos protocolos (por exemplo, exaurir conexões ou threads), causando indisponibilidade gradual com pouco consumo de banda [^2].

Neste trabalho, focamos em ataques DoS diretos que exploram o protocolo DNS e suas vulnerabilidades, gerando alto número de requisições por segundo.

## DNS – Domain Name System

O Sistema de Nomes de Domínio (DNS) traduz nomes legíveis (e.g., `www.example.com`) em endereços IP (e.g., `192.0.2.1`), permitindo que usuários acessem serviços sem memorizar números difíceis. Sua arquitetura é hierárquica:

1. **Servidores raiz**: apontam para os servidores de TLD (Top-Level Domain).  
2. **Servidores TLD**: indicam os servidores autoritativos de cada domínio.  
3. **Servidores autoritativos**: armazenam os registros definitivos (A, AAAA, MX, NS, etc.).  
4. **Servidores recursivos**: recebem a consulta do cliente, verificam o cache e, se necessário, fazem buscas iterativas na hierarquia até obter a resposta, armazenando-a em cache para otimizar futuras consultas.

![Fluxo de resolução DNS](imagens/dns_general.png)  
*Figura: Fluxo de resolução DNS.*

![Zonas DNS](imagens/dns_zones.jpeg)  
*Figura: Exemplo de zonas DNS para `sales.com`.*

## Water Torture

A técnica *DNS Water Torture*, popularizada pelo Mirai, consiste em enviar subdomínios pseudoaleatórios a servidores DNS para exaurir recursos de recursão e encaminhamento:

1. O atacante gera queries com *subdomínios inexistentes* (e.g., `abcd1234.example.com`).  
2. A flag **RD** (Recursion Desired) força o recursivo a buscar nos autoritativos.  
3. Como não há resposta em cache, cada query percorre toda a hierarquia, sobrecarregando recursivos e autoritativos.  
4. Em ataques distribuídos, o impacto é exponencial.

![Bytes de uma consulta gerada pela Eris](imagens/raw_byte.jpeg)  
*Figura: Bytes de uma consulta gerada pela Eris.*

- **ID**: identificador da transação DNS.  
- **Flags**: RD = recursão desejada.  
- **QDCOUNT**: número de perguntas (normalmente 1).  
- **QNAME**: domínio consultado (cada rótulo precedido por seu comprimento).  
- **QTYPE**: tipo de registro (A para IPv4).  
- **QCLASS**: classe do registro (IN para Internet).

![Ilustração das consultas realizadas pelo Mirai](imagens/mirai_query.png)  
*Figura: Consultas do Mirai.*  

![Ilustração das consultas realizadas pela Eris](imagens/query_eris.jpg)  
*Figura: Consultas da ferramenta Eris.*  

![Técnica de DNS Water Torture](imagens/water-tortute-akamai.png)  
*Figura: Fluxo da técnica Water Torture.*

## Mirai

O malware *Mirai* recrutou dispositivos IoT (câmeras, roteadores) para formar uma botnet massiva e lançar DDoS, incluindo a técnica de *DNS Water Torture*. Variantes modernas exploram subdomínios pseudoaleatórios para exacerbar o consumo de recursos nos servidores DNS, mostrando que o volume de tráfego e a exploração de falhas de protocolo são igualmente críticos.
