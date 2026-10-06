### 1\. CHECKLIST DE QUESTIONAMENTOS DE ENTRADA (VALIDAÇÃO DE PREMISSAS)

Antes de escolher qualquer banco de dados distribuído ou padrão de persistência, o time de engenharia **deve** validar as seguintes premissas operacionais e de negócio:

* [ ] **A regra de negócio tolera Consistência Eventual ou exige Consistência Forte?** *Validação:* Se o sistema for consultado imediatamente após uma escrita, é aceitável que diferentes nós ou réplicas retornem versões temporariamente divergentes (modelo BASE / consistência eventual)[5], ou todos os nós precisam obrigatoriamente exibir o mesmo dado simultaneamente (modelo ACID / consistência forte)[6][7]?
* [ ] **Qual é o impacto financeiro e operacional de leituras desatualizadas (** **stale reads** **)?** *Validação:* Exibir dados obsoletos por alguns milissegundos causará prejuízos graves (ex: saldo bancário negativo, furos de estoque em sistemas transacionais críticos)[8][9] ou apenas uma pequena inconsistência temporária na experiência do usuário (ex: buscas em e-commerce, listagem de redes sociais)[10]?
* [ ] **Qual é a tolerância à latência de escrita em condições normais de rede (sem falhas)?** *Validação:* O sistema opera com um orçamento (*budget*) rígido de tempo de resposta em microssegundos (exigindo otimizar a latência local de escrita)[4], ou aceita pagar o overhead de rede de múltiplos *roundtrips* e replicação síncrona/consenso distribuído (Raft/Paxos) para priorizar a consistência[11][12]?
* [ ] **Qual deve ser o comportamento padrão do sistema durante uma Partição de Rede (P)?** *Validação:* Diante de uma falha física de comunicação de rede entre os nós do cluster, o serviço deve continuar aceitando escritas locais e servindo dados desatualizados nos nós sobreviventes (foco em Disponibilidade - A)[8][13] ou deve rejeitar operações e isolar/desativar nós inconsistentes até que a sincronia seja restabelecida (foco em Consistência - C)[14]?
* [ ] **Temos maturidade de engenharia e algoritmos para resolver conflitos de escrita?** *Validação:* Se optarmos por alta disponibilidade em momentos de partição, a aplicação conta com mecanismos de resolução automática de dados divergentes (como algoritmos CRDTs)[15] ou chaves de idempotência robustas para consolidar as escritas de forma assíncrona pós-reintegração de rede[16]?

---

### 2\. MATRIZ DE TRADE-OFFS E DILEMAS TÉCNICOS

#### [ ] Dilema Arquitetural 1: PC/EC (*Google Spanner, CockroachDB, RDS Multi-AZ*) vs. PA/EL (*DynamoDB, Cassandra, Redis Cluster*)[12]

* **O que você GANHA com PC/EC:** Consistência forte global e integridade transacional absoluta tanto durante partições quanto em funcionamento normal[12]. Evita de forma definitiva a leitura de dados obsoletos, dados inválidos ou corrompidos (*dirty reads*, *non-repeatable reads*, *phantom reads*)[12][22].
* **O que você PAGA com PC/EC (Custo oculto):** Latência de escrita acentuadamente mais alta (toda operação precisa aguardar o consenso distribuído via rede ou replicação síncrona entre réplicas antes de confirmar o commit)[8][11] e potencial indisponibilidade temporária de partes do cluster em caso de falha de comunicação ou perda de quórum[9][14].
* **Blast Radius:** Em caso de falha grave na rede (partição), operações de escrita em nós incapazes de alcançar a maioria do cluster (quórum) falharão imediatamente[14], podendo paralisar pipelines transacionais globais.
* **Regra Prática de Decisão:**
  * *Escolha* **PC/EC** *se:* Estiver projetando sistemas onde a precisão e a atomicidade transacional são inegociáveis (ex: registros financeiros, saldos de carteiras digitais, calculadores de limite de crédito, ou coordenação de clusters de infraestrutura)[9][14].
  * *Escolha* **PA/EL** *se:* O sistema exigir ingestão massiva de dados com tempos de resposta rápidos globalmente (ex: telemetria, streaming, trackers, ou carrinhos de compra) onde a performance e a alta disponibilidade contínua superam a necessidade de sincronização imediata de dados[23][24].

#### [ ] Dilema Arquitetural 2: CP (*MongoDB com majority write concern, etcd, Consul*) vs. AP (*Cassandra, DynamoDB*) sob Partição de Rede[13]

* **O que você GANHA com CP:** Garantia inabalável de consistência das informações. O banco desativa nós incapazes de se comunicar de forma síncrona com o restante do cluster, prevenindo desvios (*drifts*) catastróficos nos dados[14].
* **O que você PAGA com CP (Custo oculto):** Perda de disponibilidade de partes da aplicação. Requisições direcionadas para nós isolados da partição minoritária serão rejeitadas com erro, afetando os SLOs de disponibilidade da camada de serviços[14].
* **Blast Radius:** Indisponibilidade parcial da aplicação no ponto de vista dos usuários que dependem da comunicação com as réplicas que foram isoladas pela falha de rede[4][14].
* **Regra Prática de Decisão:**
  * *Escolha* **CP** *se:* For melhor apresentar um erro ao cliente final do que permitir que ele atualize ou visualize uma informação inconsistente ou desatualizada que violará regras de integridade de domínio[26].
  * *Escolha* **AP** *se:* O negócio exigir que o serviço de escrita e leitura nunca pare de responder, mesmo que isso implique que usuários em locais geograficamente distintos vejam estados diferentes do mesmo registro por algum tempo (consistência eventual temporária)[11][13].

#### [ ] Dilema Arquitetural 3: PC/EL (*Sistemas de Consistência Eventual com Health Checks Rígidos*) vs. PA/EC (*Sistemas Híbridos com Fallback para Consistência Eventual*)[15]

* **O que você GANHA com PC/EL:** Excelente latência e alto throughput de leitura/escrita em condições normais (operação saudável), abrindo mão de consistência forte no dia a dia, mas aplicando mecanismos que priorizam a consistência (C) e barram operações assim que qualquer mínima falha ou partição de rede seja detectada[24][26].
* **O que você PAGA com PC/EL (Custo oculto):** Overhead operacional no monitoramento contínuo. Exige processos constantes de *health checks* de altíssima frequência e *heartbeats* entre os nós para validar o status do cluster antes de aceitar qualquer fluxo de leitura ou escrita[27]. Sob qualquer partição de rede, o cluster perde instantaneamente sua disponibilidade para evitar inconsistências, mesmo sem haver tentativas de escrita concorrentes[26].
* **Blast Radius:** Perda imediata e total de disponibilidade em cenários de instabilidade de rede intermitente (redes ruidosas), pois o sistema entra frequentemente em modo de proteção ou travamento automático de operações[26][27].
* **Regra Prática de Decisão:**
  * *Escolha* **PC/EL** *se:* O sistema exigir alta performance na maior parte do tempo operacional, mas sua regra de negócio não possui (ou não tolera implementar) algoritmos confiáveis para resolver conflitos de dados persistidos paralelamente durante falhas[26].
  * *Escolha* **PA/EC** *se:* O sistema requerer consistência forte no cotidiano, mas em caso de desastre (partição), a experiência do usuário sob hipótese alguma pode ser interrompida. Exige que sua aplicação conte com algoritmos complexos de tipos de dados replicados livres de conflito (como CRDTs) para realizar o *merge* das escritas concorrentes após o restabelecimento da rede[15].

---

### 3\. ANTI-PATTERNS E RED FLAGS DE OVER-ENGINEERING

Fique atento a estes sinais de alerta de escolhas inadequadas de persistência ou de superdimensionamento de soluções:

* [ ] **Red Flag 1: Bancos PC/EC (Consistência Forte nos cenários normais e de falha) aplicados a fluxos de Logs, Auditorias ou Analytics.** *Sinal de Alerta:* Tentar usar bancos como Google Spanner, CockroachDB ou etcd para armazenar dados de alta volumetria e escrita contínua e imutável[9][23]. Logs e telemetria toleram consistência eventual por natureza[28]. Adotar bancos PC/EC gera gargalos severos de latência, limita o throughput e infla os custos de rede de maneira totalmente desnecessária[11][12].
* [ ] **Red Flag 2: Configuração de persistência distribuída CA (Consistência e Disponibilidade) operando na Nuvem Pública ou em ambientes WAN.** *Sinal de Alerta:* Tentar forçar o modelo CA em sistemas distribuídos geograficamente[10][29]. Partições de rede (P) são eventos inevitáveis devido a falhas físicas, atualizações programadas de provedores e problemas de roteamento[30]. Ignorar a tolerância a partições (P) fará com que o sistema sofra travamentos imprevisíveis e corrupção de dados ao menor sinal de oscilação na rede[10][28].
* [ ] **Red Flag 3: Bancos de dados AP (como Cassandra ou DynamoDB configurados em modo PA/EL) operando sem mecanismos de controle de versão ou tratamento de concorrência na aplicação.** *Sinal de Alerta:* Confiar cegamente em consistência eventual em cenários com intensa concorrência de escrita sobre o mesmo registro[5][31]. Sem chaves de versão, *Last-Write-Wins* explícito ou algoritmos de resolução na leitura, o sistema sofrerá perda silenciosa de dados devido a sobrescritas causadas pela latência de replicação assíncrona entre nós[5][31].

---

### 4\. CHECKLIST DE GO-LIVE E OPERAÇÃO (DAY-2)

Para garantir uma entrada segura em ambiente de produção distribuído, os seguintes mecanismos operacionais e de observabilidade devem estar configurados:

* [ ] **Monitoramento de Replicação e Drifts (Desvios de Sincronia):** *Day-2:* Dashboards ativos medindo o lag de replicação assíncrona entre nós secundários (especialmente em arquiteturas PA/EL e PC/EL)[5]. Alertas baseados em percentis para latência de replicação para identificar quando o dado se tornou obsoleto além do limite aceitável do negócio[5][32].
* [ ] **Monitoramento de Consensus e Heartbeats:** *Day-2:* Monitoramento em tempo real do estado de consenso (líder ativo, quórum do cluster Raft/Paxos)[12][20]. Alarmes configurados para alertar sobre flutuações rápidas na seleção de líderes (cenários de instabilidade de rede ou *split-brain*)[27].
* [ ] **Four Golden Signals de Infraestrutura Distribuída:** *Day-2:* Alarmes integrados de latência p99 (especialmente de rede para conexões inter-nós), taxa de erros de consenso, saturação de threads de replicação e throughput geral de persistência[32].
* [ ] **Mecanismos de Fallback e Degradação Graciosa:** *Day-2:* Em momentos de perda de consistência em sistemas CP, o microserviço deve possuir fallbacks automáticos, como redirecionar leituras críticas de forma temporária para modelos com consistência eventual (BASE) em réplicas de leitura para evitar indisponibilidade total dos clientes[14].
