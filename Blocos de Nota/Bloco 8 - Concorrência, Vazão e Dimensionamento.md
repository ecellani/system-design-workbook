# Checklist Prático de Decisão Arquitetural: Concorrência, Paralelismo e Escalabilidade
**Base de Referência:** Teoria das Filas (Lei de Little), Concorrência & Paralelismo, Estratégias de Bloqueio, Scale Cube e Práticas de SRE  
**Finalidade:** Validação técnica pré-implementação, sizing de capacidade, revisão de System Design e governança para entrada em produção (Go-Live).

---

## 1. Checklist de Questionamentos de Entrada (Validação de Premissas)

Antes de aprovar qualquer decisão arquitetural sobre concorrência, paralelismo, dimensionamento de recursos ou escalabilidade, valide as seguintes premissas quantitativas e operacionais:

- [ ] **Concorrência Interna e Lei de Little ($L = \lambda \times W$)**
  - **Pergunta:** Qual é a taxa média de chegada ($\lambda$) e o tempo médio de processamento/permanência ($W$) esperados para definir o limite de concorrência interna ($L$)?
  - **Validação:** A aplicação da Lei de Little ($L = \lambda \times W$) é mandatória para determinar a concorrência interna (processos ou mensagens simultâneas em trânsito). O time deve estabelecer um $L(\text{Alvo})$ como guardrail de engenharia; se $L$ ultrapassar o limite saudável da solução sob picos de demanda ($\lambda$), o tempo de permanência ($W$) deve ser otimizado ($W = \frac{L(\text{Alvo})}{\lambda} \times 1000$) para evitar o acúmulo descontrolado de filas.

- [ ] **Latência de Cauda ($p95$, $p99$) e Dispersão de Payload**
  - **Pergunta:** Qual é o comportamento do sistema nos percentis de latência de cauda ($p95$, $p99$) e qual é o desvio padrão do tamanho do payload?
  - **Validação:** Basear o planejamento apenas no tempo médio de resposta oculta outliers graves. É necessário mapear a dispersão e variabilidade do payload. Payloads grandes ampliam o consumo de memória, pressão de Garbage Collection, buffers de rede e latência de serialização, gerando caudas longas de latência e antecipando o enfileiramento interno.

- [ ] **Margem de Saturação e Curva do Joelho (Knee Curve)**
  - **Pergunta:** O sistema opera antes da "Curva do Joelho" (*Knee Curve*) e dentro das margens seguras de saturação de CPU/Memória?
  - **Validação:** A latência cresce de forma não linear quando o sistema ultrapassa o Ponto Saudável e se aproxima da Curva do Joelho. O uso de CPU e memória deve ser mantido dentro da margem de 80–85% de utilização, pois incrementos marginais de carga acima desse limiar provocam explosão desproporcional de filas e saturação do sistema.

- [ ] **Gargalo Dominante e Throughput Sistêmico**
  - **Pergunta:** Qual é o componente que dita o gargalo e a capacidade do Throughput Sistêmico de ponta a ponta?
  - **Validação:** O throughput observado externamente é estritamente limitado pelo menor gargalo ativo da cadeia transacional:
    $$\text{Throughput Sistêmico} = \min(s_1, s_2, s_3, \dots)$$
    O engenheiro deve mapear se o gargalo reside na aplicação, no banco de dados, no cache ou no broker de mensageria antes de tentar escalar nós isolados.

- [ ] **Compartilhamento de Estado: Intra-processo vs. Inter-processo**
  - **Pergunta:** As operações concorrentes compartilham estado em memória local ou exigem coordenação entre múltiplas instâncias/containers?
  - **Validação:** Deve-se definir se o isolamento de raia exige sincronização síncrona intra-processo (Paralelismo Interno via Mutexes/Semáforos) ou coordenação inter-processual (Paralelismo Externo via locks distribuídos em Redis ou Zookeeper com suporte a *Last-Write-Wins* e timestamps atômicos).

---

## 2. Matriz de Trade-offs e Dilemas Técnicos

### Dilema 1: Concorrência por Interleaving (Single-Core) vs. Paralelismo Real (Multi-Core / Distribuído)
- **Ganhos (Concorrência / Interleaving):** Habilidade de gerenciar e alternar rapidamente a execução de múltiplas tarefas no mesmo núcleo de CPU via *context switching*. Otimiza o tempo ocioso do processador durante operações lentas de I/O (ex.: chamadas HTTP, leituras de disco) sem exigir hardware multinúcleo de alto custo.
- **Custos (Concorrência / Interleaving):** Ilusão de simultaneidade sem ganho real de velocidade para processamentos CPU-bound. Overhead contínuo decorrente de trocas de contexto entre threads/goroutines e necessidade de sincronização de memória compartilhada.
- **Ganhos (Paralelismo Real):** Execução literal e simultânea de instruções em múltiplos núcleos físicos de CPU (Paralelismo Interno) ou em múltiplos nós/containers (Paralelismo Externo).
- **Custos (Paralelismo Real):** Complexidade estrutural de código elevada. Risco crítico de contenção de recursos, *Race Conditions*, *Deadlocks* e *Starvation*.
- **Blast Radius:** Corrupção de memória e inconsistência de dados devido a acessos simultâneos não sincronizados, ou travamento total de threads em estados de espera permanente (*Deadlock*).
- **Regra de Decisão:**
  - *Escolha Concorrência se:* O workload for predominantemente I/O-bound (espera de rede/disco) e executar em ambientes com recursos computacionais limitados.
  - *Escolha Paralelismo Real se:* A aplicação exigir computação intensiva de dados (CPU-bound, Analytics, ETL, ML) e o hardware fornecer múltiplos núcleos ou nós distribuídos.

---

### Dilema 2: Bloqueio Ativo por Espera (Spinlock) vs. Bloqueio Passivo de Thread (Mutex / Semáforo)
- **Ganhos (Spinlock):** A thread permanece ativa em loop contínuo (*atomic CompareAndSwap*) aguardando a liberação da trava. Elimina o overhead computacional de suspender a thread (*sleep*) e o custo de troca de contexto no sistema operacional.
- **Custos (Spinlock):** Consumo severo e desperdício de ciclos de CPU enquanto a thread gira aguardando a liberação do lock.
- **Ganhos (Mutex / Semáforo):** A thread que não consegue adquirir o recurso é colocada em estado de espera (*sleep*), liberando a CPU para outras tarefas até ser notificada via sinalização (*Signal*).
- **Custos (Mutex / Semáforo):** Latência adicional introduzida pelo ciclo de bloqueio, suspensão e reagendamento de threads pelo Kernel.
- **Blast Radius:** Saturação de 100% de CPU por Spinlocks presos em travas longas, esgotando os recursos da máquina hospedeira.
- **Regra de Decisão:**
  - *Escolha Spinlock se:* O tempo de retenção da trava for comprovadamente ultra-curto (poucos nanossegundos) e o custo da troca de contexto for o gargalo mensurado.
  - *Escolha Mutex / Semáforo se:* O recurso compartilhado for mantido bloqueado por períodos mais longos ou envolver operações de I/O.

---

### Dilema 3: Trava em Memória Local (Mutex / Channels) vs. Lock Distribuído (Redis / Zookeeper)
- **Ganhos (Trava Local):** Altíssimo throughput, execução direta na memória do processo e latência de rede nula.
- **Custos (Trava Local):** Inabilidade de coordenar estados entre réplicas sob escalabilidade horizontal (*Scale Out*). Instâncias distintas acessarão o recurso compartilhado sem coordenação.
- **Ganhos (Lock Distribuído):** Exclusividade mútua, atomicidade e idempotência entre múltiplos containers e servidores distribuídos. O Zookeeper oferece remoção automática de lock via encerramento de sessão (*ephemeral znodes*).
- **Custos (Lock Distribuído):** Latência de rede em cada operação de Lock/Unlock, acoplamento a um cluster centralizado (Ponto Único de Falha) e risco de inconsistências por partições de rede (*split-brain*).
- **Blast Radius:** Falha ou latência elevada no cluster de Lock Distribuído bloqueia a vazão transacional de todas as instâncias da aplicação.
- **Regra de Decisão:**
  - *Escolha Trava Local se:* O recurso compartilhado for estritamente interno a um único processo isolado.
  - *Escolha Lock Distribuído se:* A aplicação rodar em múltiplas réplicas/containers consumindo eventos de filas compartilhadas (Kafka, RabbitMQ, SQS) e exigir garantia de processamento único e ordenado por chave.

---

### Dilema 4: Scale Up (Vertical) vs. Scale Out (Eixo X) vs. Sharding de Dados (Eixo Z)
- **Ganhos (Scale Up - Vertical):** Simplicidade operacional imediata. Aumenta-se CPU, RAM e I/O de uma única máquina sem alterar a arquitetura da aplicação.
- **Custos (Scale Up - Vertical):** Limite físico de hardware e custo financeiro exponencial. Não elimina Pontos Únicos de Falha (SPoF).
- **Ganhos (Scale Out - Eixo X):** Elasticidade dinâmica e resiliência via adição de instâncias idênticas atrás de balanceadores de carga.
- **Custos (Scale Out - Eixo X):** Exige aplicações rigorosamente *stateless* e transfere a pressão de concorrência para a camada de banco de dados.
- **Ganhos (Sharding - Eixo Z):** Particionamento da base de dados em nós independentes por meio de uma chave de partição (*Sharding Key*), escalando escritas de forma quase linear e reduzindo o *blast radius* de falhas.
- **Custos (Sharding - Eixo Z):** Altíssima complexidade de roteamento de chaves na camada de aplicação e perda de transações ACID globais e *joins* eficientes entre partições.
- **Blast Radius:**
  - *Scale Up:* Queda do nó derruba o sistema por completo.
  - *Scale Out (Eixo X):* Queda de um nó é absorvida pelas demais réplicas ativas.
  - *Sharding (Eixo Z):* Falha isolada afeta apenas o subconjunto de dados contido naquela partição específica.
- **Regra de Decisão:**
  - *Escolha Scale Up para:* Mitigações emergenciais de curto prazo ou bases relacionais que não atingiram o teto da máquina.
  - *Escolha Scale Out (Eixo X) para:* Camadas de aplicação, APIs e microserviços sem estado.
  - *Escolha Sharding (Eixo Z) para:* Camadas de persistência que saturaram a capacidade máxima de escrita em nó único.

---

### Dilema 5: Autoscaling Reativo por Métricas vs. Capacity Planning Preditivo
- **Ganhos (Autoscaling Reativo):** Automação simples baseada em regras de utilização de recursos de infraestrutura (ex.: HPA por consumo de CPU/RAM).
- **Custos (Autoscaling Reativo):** Atuação pós-fato (*lag temporal*). A nova capacidade ($\mu$) só é provisionada após a saturação já ter ocorrido, expondo a aplicação a falhas por *bursts* rápidos.
- **Ganhos (Capacity Planning Preditivo):** Dimensionamento preventivo construído via modelagem da taxa de chegada ($\lambda$), latências em cauda ($W$) e margem antes do ponto de inflexão (*pre-knee*).
- **Custos (Capacity Planning Preditivo):** Demanda execução contínua de testes de carga (*stress/chaos testing*), telemetria detalhada e sincronização constante com projeções de tráfego de negócio.
- **Blast Radius:** Falha do escalador reativo durante picos repentinos provoca enfileiramento em cascata, degradação de timeouts e colapso de nós por saturação.
- **Regra de Decisão:**
  - *Escolha Autoscaling Reativo como:* Camada secundária de proteção para variações de tráfego graduais.
  - *Escolha Capacity Planning Preditivo para:* Campanhas sazonais, picos previsíveis de grande escala e proteção de fluxos críticos de receita.

---

## 3. Anti-Patterns e Red Flags de Over-Engineering

- [ ] **Dimensionamento por Média de Latência ou TPS Médio**
  - *Sinal de Alerta:* Planejar capacidade com base em "200ms de latência média" ou "1.000 TPS médios". A média aritmética oculta rajadas em frações de segundo e ignora a gravidade do $p99$.
  - *Remediação:* Projetar dimensionamento baseado nos percentis superiores ($p95$, $p99$, $p99.9$) e na distribuição de frequência dos eventos.

- [ ] **Dependência Cega de Autoscaling contra Bursts**
  - *Sinal de Alerta:* Assumir que o provisionamento automático impede quedas sob tráfego agressivo. O tempo de inicialização de containers/VMs é muito mais lento que a taxa de enfileiramento sob pico.
  - *Remediação:* Associar mecanismos de limitação de taxa (*Rate Limiting*), degradação graciosa e pré-aquecimento de capacidade antes de eventos de alta demanda.

- [ ] **Operação Contínua Próxima a 100% de Utilização**
  - *Sinal de Alerta:* Manter instâncias intencionalmente em 90–95% de CPU/Memória visando redução de custo. Pela Teoria das Filas, acima de 80–85% o tempo de resposta cresce de forma assintótica e exponencial.
  - *Remediação:* Fixar o limiar de dimensionamento nominal entre 70% e 80% de utilização.

- [ ] **Acesso Concorrente sem Sincronização Explícita (Race Conditions)**
  - *Sinal de Alerta:* Múltiplas threads/goroutines acessando e modificando variáveis ou registros compartilhados sem Mutex, canais ou operações atômicas.
  - *Remediação:* Adotar primitivas atômicas, modelo de passagem de mensagens ou estratégias *Last-Write-Wins* (LWW) com controle de versão/timestamp atômico.

- [ ] **Aquisição Desordenada de Travas (Risco Crítico de Deadlock)**
  - *Sinal de Alerta:* Código que adquire múltiplos locks em sequências inconsistentes (ex.: Thread A trava Recurso 1 e pede Recurso 2; Thread B trava Recurso 2 e pede Recurso 1).
  - *Remediação:* Estabelecer uma política estrita de aquisição de travas em ordem hierárquica imutável ou uso de locks combinados com timeout (*try-lock*).

- [ ] **Inanição de Processamento (Thread/Task Starvation)**
  - *Sinal de Alerta:* Políticas de priorização agressivas que monopolizam slots de execução, impedindo indefinidamente a alocação de tarefas menos prioritárias.
  - *Remediação:* Implementar balanceamento justo (*fair-share scheduling*), *aging* de prioridades ou algoritmos de fila com garantias de avanço.

---

## 4. Checklist de Go-Live e Operação (Day-2)

Para autorizar a entrada em produção, assegure que os seguintes mecanismos de observabilidade, contenção e controle de capacidade estejam operacionais:

- [ ] **Monitoramento das Quatro Golden Signals (Google SRE)**
  - *Requisito:* Painéis ativos cobrindo:
    1. **Saturação:** Uso de CPU, memória, I/O e esgotamento de *pools* de conexões.
    2. **Tráfego/Throughput:** Volume de requisições por segundo (RPS/TPS) por rota.
    3. **Latência:** Decomposição estrita em percentis ($p50$, $p90$, $p95$ e $p99$).
    4. **Taxa de Erros:** Proporção de respostas 5xx ou falhas de negócio frente ao volume total.
  - *Alarme:* Disparo imediato se a taxa de erros ou a latência de cauda ($p99$) violarem os SLOs estabelecidos.

- [ ] **Métrica de Concorrência Interna ($L$) e Alertas de Guardrail**
  - *Requisito:* Dashboard calculando continuamente a concorrência instantânea ($L = \lambda \times W$).
  - *Alarme:* Alerta preventivo acionado assim que $L$ atingir o teto de $L(\text{Alvo})$, sinalizando represamento de fluxo antes da ocorrência de falhas ativas.

- [ ] **Ajuste Algorítmico do HPA (Horizontal Pod Autoscaler)**
  - *Requisito:* Configurar a equação padrão de dimensionamento horizontal:
    $$\text{Réplicas Desejadas} = \left\lceil \text{Réplicas Atuais} \times \left( \frac{\text{Valor Atual da Métrica}}{\text{Valor Alvo da Métrica}} \right) \right\rceil$$
  - *Alarme:* Alerta se a contagem de réplicas atingir o `maxReplicas` configurado no cluster.

- [ ] **Mapeamento de Backpressure e Saturação a Jusante (Downstream)**
  - *Requisito:* Monitoramento de tempos de fila e rejeição de chamadas entre serviços dependentes na malha.
  - *Alarme:* Notificação imediata quando serviços upstream passarem a reter ou descartar conexões devido à limitação de taxa dos serviços downstream.

- [ ] **Controle de Concorrência via Worker Pools e Semáforos**
  - *Requisito:* Restrição explícita do número máximo de tarefas ativas por instância usando semáforos contadores ou limites de concorrência em memória.
  - *Alarme:* Rejeição controlada de tráfego excedente via *load shedding* (HTTP 429/503) para preservar os recursos vitais da aplicação.

- [ ] **Rastreamento de Custo Unitário por Transação**
  - *Requisito:* Telemetria financeira mapeando a eficiência operacional da solução:
    $$\text{Custo por Transação} = \frac{\text{Custo Total de Infraestrutura}}{\text{Total de Transações Processadas}}$$
  - *Alarme:* Auditoria periódica para assegurar que expansões de escalabilidade mantenham ou reduzam o custo marginal unitário sob ganho de escala.
  