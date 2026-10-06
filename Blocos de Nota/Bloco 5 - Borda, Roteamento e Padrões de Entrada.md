### 1\. CHECKLIST DE QUESTIONAMENTOS DE ENTRADA (VALIDAÇÃO DE PREMISSAS)

Antes de selecionar qualquer padrão de balanceamento de carga, gateway de exposição ou camada de BFF, o engenheiro líder **deve** validar as seguintes premissas operacionais e limites quantitativos:

* [ ] **Qual é o perfil das conexões de rede e o protocolo primário dos clientes consumidores (curta duração HTTP/REST vs. longa duração keep-alive, WebSockets ou gRPC)?** *Justificativa:* Protocolos de longa duração e conexões persistentes exigem algoritmos de balanceamento orientados a estado de conexão (como **Least Connection**)[1] ou inspeção de requisições pendentes (**Least Outstanding Requests - LOR**)[2]. Utilizar algoritmos estáticos como **Round Robin** ou **Least Request** em conexões persistentes causará acúmulo desigual de carga e saturação severa de nós específicos[3].
* [ ] **O orçamento de latência (** **latency budget** **) permite inspeção de payload em Camada 7 ou exige encaminhamento transparente em Camada 4?** *Justificativa:* Balanceadores em **Layer 4 (Transporte - TCP/UDP)** operam apenas com endereços IP e portas, oferecendo latência extremamente reduzida e alto throughput por não inspecionarem o conteúdo do pacote[6][7]. Se o roteamento exigir verificação de headers, cookies, query strings ou URLs, o sistema pagará o custo computacional do processamento em **Layer 7 (Aplicação)**[8].
* [ ] **Qual é o modelo de consistência e a política de flexibilidade tolerados para o controle de taxa (Rate Limiting)?** *Justificativa:* Se o negócio exigir vazão estritamente constante e rígida sem permitir picos de tráfego, a arquitetura deve implementar o algoritmo **Leaky Bucket**[9]. Se a aplicação precisar absorver picos curtos e imprevisíveis de tráfego (*bursts*) usando contadores centralizados em memória (como Redis), deve-se adotar o **Token Bucket**, aceitando uma consistência eventual na contagem de tokens[10].
* [ ] **Os clientes de entrada possuem requisitos de negócio, segurança e payloads heterogêneos por dispositivo (Mobile, Web, IoT, APIs Públicas)?** *Justificativa:* Tentar atender clientes heterogêneos com um único backend ou gateway genérico força o envio de dados desnecessários para redes móveis e aumenta o acoplamento[13]. Se houver grande discrepância de requisitos de payload, segurança ou protocolos, o design deve adotar **Backend for Frontends (BFFs)** segregados[13].
* [ ] **A topologia de microsserviços exige substituição "em voo" de serviços e estratégias de implantação gradativa (Canary Deployments)?** *Justificativa:* O **API Gateway** permite reescrever caminhos de retaguarda e dividir percentualmente o tráfego entre diferentes versões de um microsserviço (ex: 90% v1 e 10% v2)[14]. Isso isola os clientes externos de mudanças de infraestrutura e viabiliza atualizações sem downtime[14][15].

---

### 2\. MATRIZ DE TRADE-OFFS E DILEMAS TÉCNICOS

#### [ ] Dilema Arquitetural 1: Balanceamento em Layer 4 (Transporte TCP/UDP) vs. Layer 7 (Aplicação HTTP/gRPC/WebSocket)

* **O que você GANHA com Layer 4:** Extrema velocidade, menor consumo de CPU e latência mínima de rede[7]. O balanceador não abre nem inspeciona o payload da aplicação, atuando como um roteador de pacotes transparente focado em alto throughput[6][7].
* **O que você PAGA (Custo oculto):** Falta total de granularidade no roteamento[7]. Não é possível aplicar decisões baseadas em URIs, headers HTTP, métodos, cookies ou payload[6][7]. Fica impossibilitado de realizar SSL/TLS Offloading, cacheamento de respostas ou compressão no balanceador[8].
* **Blast Radius:** Roteamento incorreto no nível de infraestrutura por falha de porta/IP afeta a conectividade bruta de todas as instâncias do pool.
* **Regra Prática de Decisão:**
  * *Escolha* **Layer 4** *se:* O requisito primordial do workload for vazão máxima e baixíssima latência (ex: bancos de dados, streaming de vídeo bruto, ingestão TCP/UDP) sem necessidade de regras de negócio ou cabeçalhos HTTP[7].
  * *Escolha* **Layer 7** *se:* A aplicação exigir roteamento inteligente por caminhos (basepaths), headers, SSL/TLS Offloading, terminação de certificados ou compressão de payload (ex: ecossistemas de microsserviços REST/gRPC)[8][16].

#### [ ] Dilema Arquitetural 2: Algoritmo de Balanceamento Stateless (Round Robin / Least Request) vs. Algoritmo Baseado em Saturação (Least Outstanding Requests - LOR)

* **O que você GANHA com Round Robin / Least Request:** Baixíssima complexidade de computação e execução cíclica simples e equitativa de requisições, sendo ideal para workloads uniformes e de curta duração[17].
* **O que você PAGA (Custo oculto):** Desbalanceamento severo em cargas heterogêneas[3][4]. O Round Robin e o Least Request ignoram o custo de processamento e a duração das requisições (ex: uma busca simples por ID competindo contra a geração de um relatório contábil pesado)[4][21]. Além disso, o Least Request sem reset de contadores pode causar uma negação de serviço involuntária contra novos hosts que entram no pool[4].
* **Blast Radius:** Sobrecarga e travamento por saturação de CPU/memória em um nó específico do pool enquanto os demais permanecem subutilizados[19][21].
* **Regra Prática de Decisão:**
  * *Escolha* **Round Robin / Least Request** *se:* A frota de servidores for homogênea e o tempo de processamento de todas as rotas do serviço for curto, estável e previsível[3][20].
  * *Escolha* **Least Outstanding Requests (LOR)** *se:* O tempo de resposta das requisições for variável e imprevisível, exigindo que o balanceador monitore em tempo real quantas requisições ainda estão pendentes em cada servidor para evitar a saturação de hosts[2][22].

#### [ ] Dilema Arquitetural 3: API Gateway Unificado Centralizador vs. Backend for Frontends (BFFs) Especializados por Canal

* **O que você GANHA com API Gateway Unificado:** Ponto único de contato para governança, padronização corporativa de autenticação/autorização, rate limiting global, proteção de borda e simplificação do catálogo de APIs REST[23].
* **O que você PAGA (Custo oculto):** O Gateway é um componente de infraestrutura pura[26]. Tentar colocar lógicas de agregação complexas ou transformações de UI nele cria um ponto único de falha (SPoF) organizacional, gargalos de deploy entre equipes e acoplamento indevido[24][26].
* **O que você GANHA com BFFs:** Aplicações completas e segregadas[26] projetadas para responder a um frontend específico (Web, Mobile, IoT)[13]. Permite composição de APIs (*API Composition*), minimização de payloads para redes móveis e autonomia total para os times de interface[13][27].
* **Blast Radius:** Falha no API Gateway Unificado pode indisponibilizar a entrada de todos os serviços da organização[23]. Falha em um BFF isola o blast radius apenas ao canal específico (ex: falha no BFF Mobile não afeta o BFF Web)[13].
* **Regra Prática de Decisão:**
  * *Escolha* **API Gateway Unificado** *para:* Governança de infraestrutura de borda, autenticação centralizada, rate limiting global e exposição de APIs públicas/terceiros de forma padronizada[24].
  * *Escolha* **BFFs Especializados** *se:* A aplicação possuir múltiplos canais com requisitos de payload, segurança e desempenho totalmente distintos, onde a agregação de chamadas e a formatação de dados devam ser tratadas por uma aplicação dedicada antes da entrega[13][26].

#### [ ] Dilema Arquitetural 4: Rate Limiting com Algoritmo Token Bucket vs. Leaky Bucket

* **O que você GANHA com Token Bucket:** Capacidade de absorver picos curtos e inesperados de tráfego (*bursts*) reutilizando tokens acumulados no balde até o limite estabelecido[10][12].
* **O que você PAGA (Custo oculto):** Consistência eventual na contagem distribuída de tokens (normalmente via Redis)[11][12]. Por conta da gestão flexível e sincronização entre réplicas, pode haver um vazamento tolerante (*leakage*) que deixa passar pequenas requisições acima da cota[12].
* **O que você GANHA com Leaky Bucket:** Cadência e taxa de saída estritamente constantes em direção ao backend[9]. Elimina por completo qualquer possibilidade de picos ou quebra do limite, garantindo suavização total do tráfego (*traffic shaping*)[9].
* **Blast Radius:** No Token Bucket, um *burst* acumulado muito grande pode causar estresse temporário na CPU do backend[12]. No Leaky Bucket, requisições que excedem a cota fixa são rejeitadas imediatamente (HTTP 429), podendo impactar a experiência do usuário durante picos legítimos[9][29].
* **Regra Prática de Decisão:**
  * *Escolha* **Token Bucket** *se:* O sistema precisar de limites seguros, mas tolerar pequenos picos momentâneos de tráfego dos clientes[10][12].
  * *Escolha* **Leaky Bucket** *se:* O microsserviço de retaguarda possuir uma capacidade rígida e inflexível de processamento que ruirá se receber qualquer tráfego acima da taxa estipulada[9][29].

---

### 3\. ANTI-PATTERNS E RED FLAGS DE OVER-ENGINEERING

Fique atento aos seguintes sinais de desvio de arquitetura e superdimensionamento identificados nas fontes:

* [ ] **Red Flag 1: Confundir BFF com Componente de Infraestrutura ou API Gateway.** *Sinal de Alerta:* Tratar o BFF como se fosse um proxy reverso ou tentar programar lógicas de agregação complexas e transformações de UI dentro de um API Gateway de infraestrutura[24][26]. BFFs são aplicações completas com ciclo de vida próprio e devem atuar, se necessário, como backends posicionados atrás do Gateway ou Load Balancer[26].
* [ ] **Red Flag 2: Uso do Algoritmo Least Request em Ambientes com Escalabilidade Horizontal Dinâmica (Auto-scaling) sem Reset de Contador.** *Sinal de Alerta:* Implementar o algoritmo Least Request sem um mecanismo para "zerar" os contadores de requisições[4]. Quando novos hosts entram no pool do balanceador durante um pico de carga, eles iniciam com contador zero e recebem uma enxurrada violenta de requisições concentradas, sofrendo uma "negação de serviço" involuntária[4].
* [ ] **Red Flag 3: Aplicação de IP Hash Balancing para Usuários Acessando via NATs ou Proxies Compartilhados.** *Sinal de Alerta:* Tentar usar o IP do cliente para garantir persistência de sessão em cenários onde milhares de usuários estão atrás do mesmo IP público de um NAT ou proxy corporativo[30][31]. Isso direcionará todo o volume de tráfego do NAT para um único servidor do pool, destruindo o balanceamento de carga[31]. A alternativa é usar hash baseado em headers ou cookies[31].
* [ ] **Red Flag 4: API Gateway Unificado em Arquiteturas Monolíticas Simples.** *Sinal de Alerta:* Adicionar um API Gateway completo na frente de uma aplicação monolítica de pequeno porte que possui apenas uma única URL e poucas APIs[24][32]. Para monolitos, um Proxy Reverso ou Load Balancer simples em relação 1:1 é suficiente para gerenciar SSL/TLS, compressão e conexões[33][34].

---

### 4\. CHECKLIST DE GO-LIVE E OPERAÇÃO (DAY-2)

Para autorizar a subida em produção de uma infraestrutura baseada nestes padrões, os seguintes indicadores e mecanismos de contingência precisam estar ativos:

* [ ] **Monitoramento das Quatro Golden Signals na Borda e nos Pools:** *Operação:* Mapeamento contínuo de **Latência** (p95 e p99 medidos no Gateway, BFF e no salto dos microsserviços), **Tráfego** (TPS/RPS), **Taxa de Erros** (monitorando respostas HTTP 429 Too Many Requests, 5xx e falhas de roteamento) e **Saturação**[2].
* [ ] **Métricas de Saturação de Conexões e Requisições Pendentes (LOR Metrics):** *Day-2:* Dashboards expondo o número de conexões ativas (keep-alive, WebSockets, gRPC) e o contador de requisições pendentes (*outstanding requests*) por host no pool do balanceador para identificar degradação de desempenho antes que ocorra timeout[1][2].
* [ ] **Configuração de Health Checks Ativos e Passivos nos Balanceadores:** *Day-2:* O Load Balancer / Proxy Reverso deve realizar checagens de saúde (*health checks*) contínuas no pool de hosts para isolar e remover automaticamente instâncias com falha ou lentidão, prevenindo a degradação da experiência do usuário[38][39].
* [ ] **Throttling Reativo de Emergência no API Gateway:** *Day-2:* Configurar gatilhos de **Throttling** reativo no Gateway[37][40]. Caso a soma de todas as requisições ultrapasse a capacidade máxima de suporte do próprio Gateway ou da malha (ex: 10.000 TPS), o Throttling deve reter/rejeitar parcialmente o tráfego em excesso para baixar a "temperatura" da infraestrutura e evitar uma pane geral do ecossistema[37][40].
* [ ] **Estratégias de Fallback e Resiliência na Camada de BFF:** *Day-2:* Implementação de retries exponenciais, timeouts e Circuit Breakers nas chamadas síncronas efetuadas pelo BFF para os microsserviços downstream. Caso um serviço secundário falhe, o BFF deve executar um fallback gracioso (ex: retornar dados em cache ou payload parcial) sem quebrar a interface do usuário final[13][24].
