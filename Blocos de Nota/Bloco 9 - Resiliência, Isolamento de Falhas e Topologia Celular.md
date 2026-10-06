# Checklist Prático de Decisão Arquitetural: Resiliência, Isolamento de Falhas e Arquitetura Celular
**Base de Referência:** Engenharia de Resiliência, Bulkheads, Cell-Based Architecture, Retries/Backpressure, Circuit Breakers e Degradação Graciosa  
**Finalidade:** Governança técnica, revisões de System Design, isolamento de Blast Radius e validação de requisitos de Day-2 para Go-Live.

---

## 1. Checklist de Questionamentos de Entrada (Validação de Premissas)

Antes de aprovar qualquer estratégia de resiliência, isolamento de falhas ou divisão celular de workloads, valide as seguintes premissas operacionais e limites quantitativos:

- [ ] **Limite de Blast Radius e Disponibilidade Percentual**
  - **Pergunta:** Qual é o limite aceitável de Raio de Impacto (*Blast Radius*) e a disponibilidade percentual esperada em caso de falha de um componente ou nó da infraestrutura?
  - **Validação:** No modelo tradicional, a falha de um serviço afeta 100% dos clientes. Com a aplicação de $N$ bulkheads ou partições, o blast radius é reduzido para $1/N$ dos clientes ($10\text{ bulkheads} = 10\%$ de impacto; $100\text{ bulkheads} = 1\%$ de impacto). Se for utilizada arquitetura celular combinada com Shuffle Sharding, o cálculo do risco estatístico passa a ser:
    $$P(\text{impacto}) \approx \left(\frac{f}{N}\right)^k$$
    Onde $f$ é o número de células com falha, $N$ o total de células e $k$ o fator de replicação.

- [ ] **Chaves de Idempotência em Rotas de Modificação de Estado**
  - **Pergunta:** A aplicação possui mecanismos maduros e obrigatórios de Chaves de Idempotência em todas as rotas de escrita/modificação de estado?
  - **Validação:** A idempotência é o pré-requisito fundamental para a implementação segura de retentativas síncronas ou assíncronas. O time deve garantir que a aplicação identifique a intenção de domínio e armazene a chave de idempotência para evitar pagamentos duplicados, inconsistências financeiras ou gravações redundantes sob cenários de reconexão.

- [ ] **Modelo de Consistência Cross-Cell/Cross-Shard (Eventual vs. Imediato/2PC)**
  - **Pergunta:** O modelo de consistência aceito pelo negócio na replicação entre células/shards ativos e passivos é Eventual ou Imediato (ACID/2PC)?
  - **Validação:** Replicar dados entre células ativas e passivas de forma assíncrona reduz a latência e remove dependências no caminho crítico transacional, exigindo contudo a aceitação de consistência eventual. Se o domínio não tolerar divergência de estado entre células ativas e passivas, adota-se replicação síncrona via Two-Phase Commit (2PC) ou algoritmos de consenso, arcando com maior latência de escrita e acoplamento temporal.

- [ ] **Isolamento Físico de Recursos Compartilhados (*Shared Fate*)**
  - **Pergunta:** Os recursos computacionais finitos (threads, pools de conexão, bancos, filas) estão isolados fisicamente por funcionalidade/tenant ou há compartilhamento implícito (*shared fate*)?
  - **Validação:** O compartilhamento de recursos como bancos de dados, tópicos, filas ou caches globais invalida a estratégia de bulkhead e a arquitetura celular. É necessário validar se a saturação de um fluxo secundário ou tenant anômalo (*noisy neighbor*) pode esgotar o pool de conexões ou threads do fluxo principal.

- [ ] **Orçamento de Latência (*Latency Budget*) e Roteamento em Borda**
  - **Pergunta:** O orçamento de latência (*latency budget*) comporta o salto adicional de rede introduzido pela camada de roteamento inteligente (*Edge Cells* / *Cell Routers*)?
  - **Validação:** Toda requisição em uma arquitetura celular passa obrigatoriamente por um roteador de borda ou Edge Cell que inspeciona deterministicamente a chave de negócio (`customerId`, `tenantId`, `accountId`) via DNS, Hashing Consistente ou Service Mesh. Deve-se validar se a latência introduzida nessa camada é compatível com os SLOs de resposta síncrona.

---

## 2. Matriz de Trade-offs e Dilemas Técnicos

### Dilema 1: Bulkheads Lógicos (Thread Pools / Filas) vs. Bulkheads Físicos (Arquitetura Celular)
- **Ganhos (Bulkheads Lógicos):** Baixa complexidade de infraestrutura e custo reduzido. Isola a concorrência via código ou runtime (pools de threads separados para leitura e escrita, filas independentes por prioridade ou integrações externas).
- **Custos (Bulkheads Lógicos):** Vulnerabilidade ao isolamento incompleto. Um comportamento anômalo sob carga (vazamento de memória, pressão de GC ou exaustão do kernel) em um único processo/container pode travar a máquina inteira e derrubar outros fluxos lógicos.
- **Ganhos (Bulkheads Físicos / Celular):** Isolamento estrutural absoluto e blast radius deterministicamente limitado. Cada célula contém seus próprios microsserviços, bancos de dados, caches e filas, impedindo que falhas em um compartimento afetem os demais.
- **Custos (Bulkheads Físicos / Celular):** Alto custo financeiro de infraestrutura e complexidade elevada de gerenciamento no Day-2 (deploys segmentados por célula, monitoramento e tracing descentralizados).
- **Blast Radius:**
  - *Bulkheads Lógicos:* Uma falha de sistema operacional ou hardware no host afeta 100% dos fluxos contidos naquele nó.
  - *Arquitetura Celular:* A falha afeta estritamente a fração $1/N$ alocada àquela célula.
- **Regra de Decisão:**
  - *Escolha Bulkheads Lógicos se:* O sistema operar em escopo de microsserviço único ou monolito com restrição orçamentária, visando evitar que integrações externas lentas esgotem threads principais.
  - *Escolha Arquitetura Celular se:* Estiver projetando plataformas SaaS multi-tenant críticas, sistemas bancários ou de pagamentos em grande escala onde a indisponibilidade total seja inaceitável.

---

### Dilema 2: Retries Síncronos (Backoff + Jitter) vs. Retries Assíncronos (HTTP 202 via Fila)
- **Ganhos (Retries Síncronos):** Simplicidade de implementação diretamente em memória/clientes HTTP. O Backoff Exponencial aumenta progressivamente o tempo de espera (ex.: 1s, 2s, 4s, 8s), enquanto o Jitter dispersa bursts concorrentes de retentativas.
- **Custos (Retries Síncronos):** Risco de amplificação de carga (*Retry Storms*) sobre dependências degradadas se os limites não forem rígidos, além de perda do evento se a aplicação for encerrada durante as tentativas.
- **Ganhos (Retries Assíncronos):** Alta resiliência e desacoplamento temporal. Transforma chamadas síncronas em assíncronas ao retornar HTTP 202 Accepted, enfileirando o payload para processamento tardio com garantias de entrega gerenciadas via ACK/NACK.
- **Custos (Retries Assíncronos):** Perda da confirmação síncrona imediata para o cliente final, exigindo padrões de consulta posterior (*polling* ou webhooks) e adição de brokers de mensageria à infraestrutura.
- **Blast Radius:**
  - *Retry Síncrono:* Falhas travam threads de chamadas síncronas, retendo recursos e pools de conexão.
  - *Retry Assíncrono:* Falhas retêm mensagens na fila ou DLQ sem derrubar a API de ingestão de entrada.
- **Regra de Decisão:**
  - *Escolha Retries Síncronos (Backoff + Jitter) para:* Chamadas *service-to-service* rápidas de baixa latência onde intermitências momentâneas de rede são comuns.
  - *Escolha Retries Assíncronos (202 Accepted) para:* Processamentos de negócio pesados ou integrações de terceiros instáveis onde o cliente aceita processamento em segundo plano.

---

### Dilema 3: Circuit Breaker Tradicional (Fail Fast - 503) vs. Circuit Breaker com Fallback Proativo
- **Ganhos (Tradicional / Fail Fast):** Proteção imediata de dependências sobrecarregadas. Nos estados Fechado, Aberto e Semiaberto, o disjuntor intercepta chamadas e falha rapidamente durante o resfriamento, permitindo a recuperação da dependência a jusante.
- **Custos (Tradicional / Fail Fast):** A requisição do usuário é rejeitada com erro imediato, repassando o impacto funcional para o cliente caso não haja tratamento na borda.
- **Ganhos (Fallback Proativo / Graceful Degradation):** Continuidade operacional contínua. Em vez de retornar erro imediato ao abrir o circuito, o sistema redireciona a consulta para *Data Snapshots* mantidos em cache ou desativa temporariamente fluxos não-críticos.
- **Custos (Fallback Proativo / Graceful Degradation):** Sacrifício temporário da consistência forte por consistência eventual (ex.: risco calculado ao aprovar transações com base no último snapshot de saldo/limite) e complexidade de manter pipelines de snapshot ativos.
- **Blast Radius:**
  - *Circuit Breaker Tradicional:* Interrompe a jornada do usuário quando o circuito abre.
  - *Circuit Breaker com Fallback:* Absorve a falha e mantém a operação ativa em modo degradado.
- **Regra de Decisão:**
  - *Escolha Tradicional se:* Não houver rota alternativa viável ou for estritamente proibido operar com dados desatualizados (ex.: liquidação financeira de risco zero).
  - *Escolha Fallback Proativo se:* A continuidade do fluxo de venda/negócio for prioridade sobre a consistência estrita em tempo real.

---

### Dilema 4: Replicação Assíncrona Cross-Cell vs. Replicação Consistente Cross-Cell (2PC)
- **Ganhos (Assíncrona):** Desempenho e throughput máximo no caminho crítico da transação. A célula ativa grava os dados em sua base primária e propaga logs/eventos assincronamente para as células passivas.
- **Custos (Assíncrona):** Janela de desincronização (*Replication Lag* / *Stale Reads*). Em caso de failover imediato para a célula passiva, transações muito recentes podem não ter sido replicadas.
- **Ganhos (Consistente - 2PC):** Garantia de estado idêntico e consistente em todas as células participantes antes da confirmação do commit transacional.
- **Custos (Consistente - 2PC):** Alto acoplamento temporal inter-celular, aumento substancial da latência de resposta e perda de autonomia individual das células. Falha ou timeout em uma única célula participante aborta a transação global.
- **Blast Radius:**
  - *Replicação Assíncrona:* Indisponibilidade ou lentidão da célula passiva não afeta a taxa de escrita da célula ativa.
  - *Replicação Consistente (2PC):* Indisponibilidade da célula passiva trava ou aborta as gravações na célula ativa.
- **Regra de Decisão:**
  - *Escolha Replicação Assíncrona para:* A grande maioria dos sistemas distribuídos resilientes baseados no modelo ativo/passivo.
  - *Escolha Replicação Consistente (2PC) para:* Casos estritos de conciliação financeira onde a tolerância à divergência for nula e o acoplamento temporal for aceito.

---

## 3. Anti-Patterns e Red Flags de Over-Engineering

- [ ] **Dependências Globais Compartilhadas (*Shared Fate / Hidden Channels*)**
  - *Sinal de Alerta:* Implementar arquitetura celular ou bulkheads mantendo bancos de dados, clusters de cache, brokers de mensagens ou serviços de configuração compartilhados entre todas as células.
  - *Remediação:* Garantir isolamento estrito de recursos por célula; a falha de um recurso global compartilhado derrubaria todas as células simultaneamente.

- [ ] **Retries Síncronos Indiscriminados (*Retry Storms*)**
  - *Sinal de Alerta:* Clientes HTTP/gRPC realizando retentativas síncronas consecutivas sem limite rígido de tentativas, sem Jitter e sem suporte a Chaves de Idempotência no destino.
  - *Remediação:* Configurar teto baixo de retries (ex.: 2 a 3 tentativas), aplicar Backoff Exponencial com Jitter completo e exigir headers de idempotência em endpoints de mutação.

- [ ] **Fallbacks "Frios" sem Tráfego Contínuo**
  - *Sinal de Alerta:* Rotas de contingência mantidas totalmente ociosas, acionadas exclusivamente em situações de desastre real.
  - *Remediação:* Injetar proativamente uma parcela contínua de tráfego sintético ou real nas rotas de fallback para assegurar sua operacionalidade e evitar surpresas sob estresse.

- [ ] **Equiparar Rate Limiting Global a Bulkhead Lógico**
  - *Sinal de Alerta:* Assumir que limites globais de requisições por minuto no Gateway protegem fluxos internos concorrentes entre si.
  - *Remediação:* O Rate Limiting protege a borda externa contra saturação global, mas deve ser complementado com limites internos de threads/concorrência para que fluxos secundários (ex.: relatórios) não sufoquem o fluxo crítico (ex.: checkout).

- [ ] **Superdimensionamento de Arquitetura Celular em Baixa Volumetria**
  - *Sinal de Alerta:* Projetar infraestrutura baseada em células completas (Edge Cells, shards independentes de banco e pipelines isolados) para sistemas de tráfego modesto.
  - *Remediação:* Manter segmentação simplificada em bulkheads lógicos ou pods compartimentados até que o volume transacional justifique a complexidade do Day-2 celular.

---

## 4. Checklist de Go-Live e Operação (Day-2)

Para autorizar a entrada em produção, assegure que os seguintes mecanismos de observabilidade, contenção e contingência estejam ativos:

- [ ] **Mapeamento e Cálculo de Uptime/Disponibilidade**
  - *Requisito:* Rastreamento formal do tempo operacional sobre o tempo esperado:
    $$\text{Uptime} = \frac{\text{Tempo Total} - \sum \text{Downtime}}{\text{Tempo Total}} \times 100\%$$
  - *Métrica:* Monitoramento agregado e acompanhamento individualizado da disponibilidade por célula ou bulkhead.

- [ ] **Observabilidade do Estado de Circuit Breakers**
  - *Requisito:* Telemetria em tempo real cobrindo os estados *Fechado*, *Aberto* e *Semiaberto*.
  - *Alarme:* Disparo prioritário imediato assim que qualquer disjuntor transitar para o estado *Aberto*, com rastreamento do volume de requisições absorvidas por fallbacks no estado *Semiaberto*.

- [ ] **Gestão de Recursos via Lease Pattern e Heartbeats**
  - *Requisito:* Pools de conexão de banco de dados, consumidores de mensageria e canais gRPC operando com concessões temporárias (*Lease*) renovadas por heartbeat.
  - *Configuração:* Expiração automática e descarte de conexões ociosas para impedir retenção indevida de recursos compartilhados.

- [ ] **Gatilhos de Graceful Degradation e Feature Flags**
  - *Requisito:* Presença de Feature Flags capazes de desativar módulos não-essenciais sob picos de tráfego ou saturação de infraestrutura.
  - *Ação:* Preservação automática ou manual dos recursos computacionais exclusivamente para a jornada transacional prioritária.

- [ ] **Feedback Loop de Backpressure Integrado aos Error Budgets**
  - *Requisito:* Sinais de medição de latência e profundidade de filas calibrados para acionar desaceleração progressiva (*backpressure*).
  - *Alarme:* Se o *Error Budget* de um componente downstream atingir níveis críticos, o adaptador de entrada deve reduzir proativamente a taxa de ingestão.

- [ ] **Roteamento Determinístico e Healthchecks em Edge Cells**
  - *Requisito:* Monitoramento das instâncias de Edge Cells validando que o chaveamento de tráfego baseado em `customerId`/`tenantId` ocorra sem desvios severos (*rehash storms*).
  - *Alarme:* Healthchecks ativos nos balanceadores para isolar e drenar instantaneamente células ou réplicas em degradação.