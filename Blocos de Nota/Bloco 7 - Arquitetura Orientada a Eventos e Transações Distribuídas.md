# Checklist Prático de Decisão Arquitetural
**Base de Referência:** Mensageria, Eventos, Streaming e Arquitetura Assíncrona | CQRS | Saga Pattern | Event Sourcing Pattern  
**Finalidade:** Governança técnica, revisões em sessões de System Design, validação de arquiteturas orientadas a eventos e preenchimento de ADRs (Architectural Decision Records).

---

## 1. Checklist de Questionamentos de Entrada (Validação de Premissas)

Antes de aprovar qualquer arquitetura baseada em segregação de comandos/consultas, gerenciamento de eventos históricos ou orquestração de transações distribuídas, valide as seguintes premissas operacionais e limites quantitativos:

- [ ] **Consistência Eventual vs. Consistência Imediata (ACID)**
  - **Pergunta:** O domínio de negócio aceita operar sob o modelo de Consistência Eventual, ou existe a exigência inegociável de Consistência Imediata entre a gravação e a visualização?
  - **Justificativa:** Aceitar consistência eventual pressupõe que o sistema pode apresentar pequenas divergências temporárias entre o modelo de comando e o modelo de consulta até que os eventos de sincronização sejam propagados. Se o negócio não tolerar atraso na leitura do estado mais recente, a arquitetura deve priorizar modelos transacionais síncronos.

- [ ] **Assimetria de Carga (Write-Intensive vs. Read-Intensive)**
  - **Pergunta:** Existe uma assimetria severa entre a volumetria de escrita e a volumetria de leitura que justifique a separação física do Write Model e do Read Model?
  - **Justificativa:** O padrão CQRS propõe utilizar bancos e modelos otimizados independentes (ex.: relacional normalizado ACID para escrita e NoSQL/Views materializadas para leitura). Se a taxa de leitura e escrita for simétrica e de baixo volume, o overhead de manter sincronização assíncrona cria complexidade operacional sem ganho real de desempenho.

- [ ] **Auditoria, Rastreabilidade e SLO Histórico**
  - **Pergunta:** Qual é o nível de auditabilidade, rastreabilidade e reconstituição histórica exigido pelo negócio para cada entidade (SLO de auditoria, RPO e RTO)?
  - **Justificativa:** Se o negócio exige reconstituir a linha do tempo da evolução de uma entidade (ex.: ledger financeiro, movimentações de conta, prontuários), adota-se Event Sourcing. Caso a necessidade seja apenas armazenar o estado atualizado (State Mutation), a persistência tradicional CRUD é suficiente.

- [ ] **Fronteiras Transacionais e Microserviços Isolados**
  - **Pergunta:** A transação de negócio abrange múltiplos microserviços com bancos de dados isolados (*Database-per-Service*), inviabilizando transações ACID locais?
  - **Justificativa:** Operações distribuídas de longa duração que não podem utilizar bloqueios síncronos (como 2PC) devem ser estruturadas usando o Saga Pattern, garantindo que cada passo possua uma transação compensatória equivalente para efetuar rollbacks em caso de falha.

- [ ] **Perfil de Transporte: Log Imutável vs. Fila de Roteamento**
  - **Pergunta:** O volume de dados e o throughput exigem retenção em log imutável ordenado por chave de particionamento (Event Streaming / Kafka) ou roteamento e entrega por filas (Mensageria AMQP)?
  - **Justificativa:** Plataformas de streaming (como Apache Kafka) são indicadas para alto volume distribuído em partições com garantia de ordenação por chave. Soluções de mensageria tradicionais (AMQP / RabbitMQ) são otimizadas para roteamento granular (Direct, Topic, Fanout) e consumo destrutivo por fila.

---

## 2. Matriz de Trade-offs e Dilemas Técnicos

### Dilema 1: Event Sourcing vs. Persistência de Mutação de Estado (CRUD)
- **Ganhos:** Registro histórico e imutável de todos os fatos ocorridos no sistema (Ledger). Permite auditabilidade total, reconstituição temporal de estados passados (*rehydration*) e reprocessamento determinístico de projeções de leitura.
- **Custos ocultos:** Complexidade estrutural elevada. Consultas ao estado atual do agregado exigem reprocessar sequências de eventos (demandando estratégias de *snapshotting*). Exige governança rigorosa sobre a evolução de schemas de eventos no longo prazo.
- **Blast Radius:** Corrupção na ordem temporal ou perda de eventos no Event Store inviabiliza a reconstituição correta do estado do agregado e de todos os seus Read Models derivados.
- **Regra de Decisão:**
  - *Escolha Event Sourcing se:* O domínio exigir auditoria estrita, rastreabilidade causal, reversão de estados no tempo e alta reatividade (ex.: ledger bancário, fechamento contábil).
  - *Escolha CRUD se:* O domínio se importar apenas com o estado atual da entidade e não houver requisito de histórico ou reconstrução temporal.

---

### Dilema 2: CQRS vs. Modelo de Dados Unificado
- **Ganhos:** Escalabilidade e performance dimensionadas de forma independente para leitura e escrita. Permite otimizar o Write Model para consistência/normalização e o Read Model para consultas rápidas (NoSQL/Views desnormalizadas).
- **Custos ocultos:** Surgimento de consistência eventual entre o comando executado e a visão de leitura atualizada. Duplicação de infraestrutura de dados e dependência de esteiras de sincronização assíncronas.
- **Blast Radius:** Falhas na camada de sincronização/mensageria deixam os modelos de leitura desatualizados (*stale data*), expondo dados obsoletos aos consumidores.
- **Regra de Decisão:**
  - *Escolha CQRS se:* A volumetria de leitura for exponencialmente maior que a de escrita ou as consultas exigirem agregações complexas que saturem a base transacional principal.
  - *Escolha Modelo Unificado se:* O domínio for simples, com baixo volume de dados e onde operações de leitura e escrita compartilhem a mesma estrutura sem gargalos.

---

### Dilema 3: Saga Orquestrada vs. Saga Coreografada
- **Ganhos (Orquestrada):** Controle centralizado e visibilidade do estado de cada etapa. Facilidade para gerenciar fluxos complexos, retries, backoff e acionar transações compensatórias em ordem inversa.
- **Custos (Orquestrada):** Risco de criar um componente central acoplado a detalhes de múltiplos domínios. Exige manter tabela de estado persistente da saga.
- **Ganhos (Coreografada):** Desacoplamento entre serviços. Cada participante reage a eventos e publica novos fatos sem coordenador central.
- **Custos (Coreografada):** Dificuldade severa de rastreabilidade do fluxo global; risco de ciclos cíclicos acidentais ou falhas silenciosas de compensação em malhas com muitos participantes.
- **Blast Radius:** Na Orquestrada, falhas no orquestrador paralisam o avanço de todas as sagas ativas. Na Coreografada, uma falha de compensação intermediária pode deixar o sistema em estado parcialmente inconsistente sem notificação imediata.
- **Regra de Decisão:**
  - *Escolha Saga Orquestrada se:* A transação envolver múltiplos passos (3 ou mais), com regras de compensação condicionais e necessidade de monitoramento central de estado.
  - *Escolha Saga Coreografada se:* A transação for curta (2 a 3 passos) e os microserviços já mantiverem relação natural de publicação/assinatura de eventos.

---

### Dilema 4: Transactional Outbox / CDC vs. Dual-Write Direto
- **Ganhos (Outbox / CDC):** Garantia de atomicidade entre a alteração no banco de dados e a publicação do evento no broker. Elimina inconsistências causadas por falhas na rede ou indisponibilidade do broker.
- **Custos (Outbox / CDC):** Necessidade de criar uma tabela de Outbox no mesmo banco transacional e implementar um relay assíncrono (ou ferramentas de CDC como Debezium).
- **Blast Radius:** No Dual-Write direto, se a escrita no banco for confirmada mas a publicação no broker falhar, o evento é perdido, gerando inconsistência irrecuperável entre o Write Model e os sistemas downstream.
- **Regra de Decisão:**
  - *Escolha Outbox / CDC se:* A propagação do evento for crítica para a integridade de domínios downstream ou Read Models.
  - *Escolha Dual-Write se:* **Nunca**. O Dual-Write direto é um anti-pattern em sistemas distribuídos de alta confiabilidade.

---

### Dilema 5: Two-Phase Commit (2PC) vs. Transação Assíncrona Saga
- **Ganhos (2PC):** Consistência forte imediata e atomicidade perfeita entre múltiplos nós ou bancos de dados (fases de Prepare e Commit).
- **Custos (2PC):** Degradação acentuada de throughput devido a bloqueios síncronos estendidos de recursos (*locks*), além de alto risco de travamento sob partição de rede.
- **Blast Radius:** Se um único participante do 2PC falhar ou sofrer timeout na fase de prepare, toda a transação é abortada e conexões ficam bloqueadas aguardando timeout.
- **Regra de Decisão:**
  - *Escolha 2PC se:* A operação for executada dentro do mesmo motor de banco distribuído que forneça suporte nativo sob rede de baixíssima latência.
  - *Escolha Saga se:* A transação cruzar fronteiras de múltiplos microserviços ou bancos de dados heterogêneos na nuvem.

---

## 3. Anti-Patterns e Red Flags de Over-Engineering

- [ ] **Dual-Write Direto sem Tratamento Transacional**
  - *Sinal de Alerta:* Gravar no banco de dados e, na mesma thread de execução, publicar diretamente no broker. Falhas de conexão deixam o banco persistido sem a emissão do evento correspondente.
  - *Remediação:* Implementar Transactional Outbox Pattern ou Change Data Capture (CDC).

- [ ] **Event Sourcing e CQRS em CRUDs Triviais**
  - *Sinal de Alerta:* Adotar Event Store, projections e bases desnormalizadas de leitura para cadastros simples com baixa taxa de mutação.
  - *Remediação:* Manter arquitetura CRUD relacional unificada até que surjam requisitos reais de histórico e assimetria de carga.

- [ ] **Hot Partitions por Chave Inadequada no Kafka**
  - *Sinal de Alerta:* Uso de chaves de partição de baixa cardinalidade ou com valores desproporcionais, sobrecarregando uma única partição e deixando as demais ociosas.
  - *Remediação:* Escolher chaves de alta cardinalidade e distribuição uniforme (ex.: hash composto ou UUID de entidade com alta distribuição).

- [ ] **Reconstrução de Aggregates sem Snapshots**
  - *Sinal de Alerta:* Recompor o estado de um agregado relendo todos os eventos desde a origem, degradando a latência da API em complexidade linear $O(N)$.
  - *Remediação:* Configurar rotinas de *Snapshotting* periódico a cada $N$ eventos (ex.: a cada 1.000 ou 10.000 eventos).

- [ ] **Retentativas Infinitas sem Dead Letter Queue (DLQ)**
  - *Sinal de Alerta:* Reenviar mensagens com erro irrecuperável de volta para a fila principal, criando *poison pills* que travam o pipeline de consumo.
  - *Remediação:* Estabelecer política de retries com backoff exponencial associada a desvio para DLQ após o limite de tentativas.

- [ ] **Transações Síncronas Bloqueantes (2PC) entre Microsserviços**
  - *Sinal de Alerta:* Tentar manter consistência ACID distribuída com coordenadores síncronos entre serviços independentes, exaurindo pools de conexão.
  - *Remediação:* Redesenhar o fluxo transacional para consistência eventual utilizando o padrão Saga.

---

## 4. Checklist de Go-Live e Operação (Day-2)

Para autorizar o deploy em produção, garanta que os seguintes mecanismos de observabilidade, alertas e tolerância a falhas estejam configurados:

- [ ] **Métricas de Defasagem de Sincronização e Projeção (Projection Lag / Sync Lag)**
  - *Requisito:* Monitoramento da diferença temporal (latência em ms) e volume de eventos pendentes entre a escrita no Write Model e a atualização do Read Model.
  - *Alarme:* Disparo imediato se o lag violar os limites contratuais de SLO acordados com o negócio.

- [ ] **Consumer Group Lag e Frequência de Rebalances (Kafka)**
  - *Requisito:* Painéis ativos com métricas de lag de offsets por partição em cada Consumer Group.
  - *Alarme:* Gatilhos configurados para acúmulo sustentado de offset pendente e taxa anormal de rebalanceamentos de grupo.

- [ ] **Vazão e Saúde do Processo de Outbox Relay**
  - *Requisito:* Métricas de contagem de registros não processados na tabela de Outbox e taxa de publicação do relay/CDC.
  - *Alarme:* Alerta de retenção excessiva ou falha no conector de publicação.

- [ ] **Gestão e Monitoramento de DLQs (DLQ Depth)**
  - *Requisito:* Monitoramento contínuo da profundidade das DLQs e esteira definida para auditoria e reprocessamento pós-correção.
  - *Alarme:* Disparo prioritário para qualquer entrada de mensagem em fila morta.

- [ ] **Observabilidade e Tabela de Estado de Sagas**
  - *Requisito:* Logs estruturados e métricas categorizadas por `saga_id` cobrindo os estados: *Iniciada*, *Concluída*, *Em Compensação*, *Compensada* e *Erro Fatal*.
  - *Alarme:* Detecção de transações estagnadas em estados intermediários para intervenção operacional.

- [ ] **Garantia de Idempotência nos Consumidores**
  - *Requisito:* Implementação explícita de verificação de chaves de idempotência (baseadas em `event_id` ou versionamento do aggregate) em todos os handlers de eventos.
  - *Alarme:* Validação de resiliência contra entregas duplicadas (*at-least-once*) sem duplicação de efeitos colaterais.
