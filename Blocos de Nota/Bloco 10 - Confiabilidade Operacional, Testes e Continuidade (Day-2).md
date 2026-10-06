### 1\. CHECKLIST DE QUESTIONAMENTOS DE ENTRADA (VALIDAÇÃO DE PREMISSAS)

Antes de aprovar qualquer topologia de entrega, plano de recuperação de desastres ou estratégia de observabilidade, o engenheiro líder **deve** validar as seguintes premissas operacionais e quantitativas:

* [ ] **Quais são os limites contratuais e operacionais de RPO (Recovery Point Objective) e RTO (Recovery Time Objective) acordados com o negócio?** *Justificativa:* O **RPO** define a perda máxima de dados tolerável (defasagem de backup ou lag de replicação)[1], enquanto o **RTO** especifica o tempo máximo aceitável de indisponibilidade após uma falha[2]. Exigir RPO próximo de zero impõe replicação síncrona contínua e bancos distribuídos fortemente consistentes[3]; RTOs próximos de zero exigem topologias Ativo-Ativo ou Ativo-Passivo com failover automático[4], elevando drasticamente custos e complexidade[2][7].
* [ ] **Quais são os SLOs de latência por percentil (p95 e p99) e os SLIs de taxa de erro aceitos para a jornada do usuário?** *Justificativa:* Basear métricas operacionais na média esconde comportamentos de cauda longa (*outliers*), onde os timeouts disparam e as retentativas amplificam a carga[8][9]. O **SLO** interno da engenharia deve ser mais restritivo que o **SLA** contratual para funcionar como blindagem técnica[10], definindo limites claros de duração e *error rate* que orientam gatilhos de rollback e alertas proativos[8][11].
* [ ] **A arquitetura de dados e os contratos de comunicação garantem retrocompatibilidade para suportar a convivência de duas versões de schema durante o deployment?** *Justificativa:* Estratégias de baixo risco como Rolling Update, Blue-Green e Canary exigem a coexistência temporária de instâncias executando versões antigas e novas em paralelo[12]. A migração de schema e dados (*Schema Migration*) é a etapa mais crítica da entrega e pode inviabilizar o rollback rápido se quebrar a compatibilidade da versão estável[14][15].
* [ ] **O sistema e suas dependências foram submetidos a testes de Breakpoint e Spike para mapear o ponto exato de quebra e o modo de falha sob estresse?** *Justificativa:* Testes de carga convencionais (Average Load) apenas validam as baselines esperadas sob tráfego normal[16][17]. Sem testes de **Breakpoint** (aumento progressivo de carga até a quebra)[18] e **Spike** (picos repentinos)[19], a engenharia desconhece qual componente falhará primeiro (CPU, pools de conexão, I/O ou banco de dados) e como a cascata de falhas afetará o produto[19][20].
* [ ] **Existe algum componente centralizado no fluxo crítico da transação que atue como um Ponto Único de Falha (SPoF)?** *Justificativa:* É mandatório mapear o "fluxo feliz" de cada funcionalidade crítica e identificar SPoFs (bancos sem réplicas, gateways sem redundância, brokers únicos)[21][22]. A falha de um SPoF desprovido de rotas de *fallback* provoca a indisponibilidade total ou parcial do produto[21][23].

---

### 2\. MATRIZ DE TRADE-OFFS E DILEMAS TÉCNICOS

#### [ ] Dilema Arquitetural 1: Disaster Recovery Multi-Região: Arquitetura Ativo-Ativo vs. Arquitetura Ativo-Passivo (ou Pilot Light)

* **O que você GANHA:**
  * *Ativo-Ativo:* Múltiplas regiões/clusters recebem tráfego simultaneamente, elevando significativamente a disponibilidade global e eliminando a ociosidade de infraestrutura[5][24].
  * *Ativo-Passivo / Pilot Light:* Menor custo operacional e menor complexidade de dados, pois apenas a região primária atende o tráfego (ou mantém apenas a camada de dados ativa no Pilot Light), facilitando o controle de consistência[6][25].
* **O que você PAGA (Custo oculto):**
  * *Ativo-Ativo:* Complexidade extrema na gestão de consistência distribuída (multi-master), necessidade de estratégias explícitas de resolução de conflitos de escrita (*Last-Write-Wins*, CRDTs) e alto overhead operacional[5].
  * *Ativo-Passivo / Pilot Light:* Tempo de recuperação (RTO) dependente da velocidade de detecção (MTTD) e automatização do chaveamento de DNS/promoção, além do risco de perda de dados recentes (*replication lag* / RPO)[4].
* **Blast Radius:** No Ativo-Ativo, uma corrupção de dados ou erro lógico replicado de forma bilateral pode afetar todas as regiões simultaneamente[5]. No Ativo-Passivo, falhas no mecanismo de detecção ou de promoção manual deixam o sistema indisponível durante toda a janela de intervenção[4][6].
* **Regra Prática de Decisão:** "Escolha **Ativo-Ativo** se o produto exigir alta disponibilidade contínua com RTO/RPO próximos de zero e o domínio aceitar consistência eventual com resolução de conflitos[5]. Escolha **Ativo-Passivo / Pilot Light** se o negócio priorizar consistência forte no dia a dia e aceitar um tempo de aquecimento (*warm-up*) de infraestrutura secundária em troca de custos reduzidos[6][25]."

#### [ ] Dilema Arquitetural 2: Estratégia de Deployment: Blue-Green vs. Canary Release vs. Big Bang (Recreate)

* **O que você GANHA:**
  * *Blue-Green:* Zero downtime e rollback instantâneo via chaveamento de roteador/DNS, permitindo a execução de *smoke tests* e *warm-up* em um ambiente "Green" isolado antes da promoção[13].
  * *Canary Release:* Mitigação de risco progressiva ao direcionar apenas uma pequena porcentagem do tráfego real (ex: 10%) para a nova versão, aumentando a fatia gradualmente conforme métricas e alertas são validados[11].
  * *Big Bang / Recreate:* Simplicidade operacional máxima ao recriar todo o sistema de forma abrupta, evitando a necessidade de manter a coexistência de duas versões distintas em execução[29][30].
* **O que você PAGA (Custo oculto):**
  * *Blue-Green:* Custo duplicado de infraestrutura ao manter dois ambientes idênticos ativos e alta complexidade para gerenciar migrações de schema de banco de dados compativeis com rollback[14][31].
  * *Canary Release:* Exige roteamento inteligente de tráfego (por porcentagem ou cabeçalhos), observabilidade automatizada para acionar rollback e suporte rigoroso a versões de schema em transição[11].
  * *Big Bang / Recreate:* Indisponibilidade/downtime temporário obrigatório e risco máximo para os clientes em caso de falhas na nova versão[12][29].
* **Blast Radius:** No Big Bang, a falha impacta 100% dos usuários imediatamente[12][29]. No Blue-Green, o impacto é total durante a janela entre o chaveamento e a detecção do problema[27]. No Canary, o *blast radius* é restrito à pequena fração de clientes exposta à versão *canary*[14][28].
* **Regra Prática de Decisão:** "Escolha **Canary Release** para aplicações de alto tráfego contínuo que necessitam de validação empírica sob carga real com impacto isolado[14][28]. Escolha **Blue-Green** para validações completas em ambiente espelhado pré-chaveamento com capacidade de reversão imediata[26][27]. Escolha **Big Bang / Recreate** apenas como último recurso em arquiteturas assíncronas onde a coexistência de versões quebra o processamento (ex: rebalanceamento de leasing no Kafka) ou em mudanças destrutivas de schema[12][30]."

#### [ ] Dilema Arquitetural 3: Pré-Validação de Releases: Shadow Deployment (Traffic Mirroring) vs. Feature Flags

* **O que você GANHA:**
  * *Shadow Deployment:* Duplica uma porcentagem do tráfego real em tempo real e o envia para uma versão de sombra (que processa a requisição sem responder ao cliente), permitindo avaliar performance, latência e logs sem afetar a experiência do usuário[33][34].
  * *Feature Flags:* Habilita ou desabilita funcionalidades dinamicamente em produção para segmentos específicos de clientes sem a necessidade de realizar novos deploys ou alterações de código[35][36].
* **O que você PAGA (Custo oculto):**
  * *Shadow Deployment:* Risco de duplicação involuntária de dados e inconsistências se o ambiente sombra não rodar em modo *dry-run* (sem commit transacional) ou sem idempotência estrita[37][38].
  * *Feature Flags:* Dívida técnica e complexidade cognitiva causadas pela proliferação de condicionais no código-fonte, além da dependência de um serviço centralizador de gerenciamento de flags[36].
* **Blast Radius:** No Shadow Deployment, o *blast radius* é zero para o usuário final (quando operado em *dry-run*)[34][37]. Nas Feature Flags, uma falha na lógica da flag pode vazar instabilidades para os segmentos ativados[36].
* **Regra Prática de Decisão:** "Escolha **Shadow Deployment** para pré-validar capacidade computacional, latência e gargalos de infraestrutura sob tráfego real em fluxos de leitura ou em modo *dry-run* antes da release oficial[34]. Escolha **Feature Flags** para lançamentos gerenciados por produto, testes A/B e para desacoplar a implantação de código da liberação funcional[35][36]."

#### [ ] Dilema Arquitetural 4: Abordagem de Testes de Carga: Teste de Carga Média (Average Load) vs. Teste de Estresse e Breakpoint

* **O que você GANHA:**
  * *Average Load:* Avalia o comportamento do sistema sob a carga média esperada por períodos prolongados, sendo ideal para validar baselines de SLO e identificar vazamentos de memória (*memory leaks*)[17][39].
  * *Estresse e Breakpoint:* Identifica o ponto exato de quebra (*breakpoint*), os limites de saturação de recursos (CPU, memória, I/O, conexões) e o comportamento do sistema sob condições extremas não convencionais[18].
* **O que você PAGA (Custo oculto):**
  * *Average Load:* Exige execuções longas e não revela como a arquitetura reage a picos repentinos ou sobrecargas além da baseline[19][39].
  * *Estresse e Breakpoint:* Requer um ambiente isolado altamente controlado e fiel ao produtivo para evitar derrubar dependências compartilhadas ou indisponibilizar serviços reais[18][20].
* **Blast Radius:** Se executados indevidamente em produção sem o devido isolamento, testes de estresse e breakpoint podem causar falhas em cascata, travamentos de banco de dados e indisponibilidade real[20][42].
* **Regra Prática de Decisão:** "Escolha **Average Load** para homologar contratos de serviço (SLOs) e identificar degradações lentas em execuções de longa duração[17][39]. Escolha **Estresse / Breakpoint** antes de grandes eventos sazonais de pico para encontrar gargalos ocultos de infraestrutura e reajustar políticas de autoscaling e rate limit[18]."

---

### 3\. ANTI-PATTERNS E RED FLAGS DE OVER-ENGINEERING

Fique atento aos seguintes sinais de alerta e práticas inadequadas identificados nas fontes:

* [ ] **Monitoramento por Acúmulo ("Coleção Infinita de Dashboards"):** Colecionar centenas de métricas e painéis desconexos sem adotar um modelo mental padronizado (como Four Golden Signals, RED ou USE)[43]. Isso gera "poluição visual", causa fadiga de alertas e impede que o time responda rapidamente se o produto está saudável[43].
* [ ] **Uso Exclusivo da Média de Latência para Medir Desempenho:** Avaliar o tempo de resposta apenas pela média, ignorando distribuições em percentis (p50, p95, p99) e histogramas[8][9]. A média oculta comportamentos de cauda longa, onde timeouts ocorrem e retries acentuam a sobrecarga no sistema[8][9].
* [ ] **Blue-Green Deployment com Alterações Destrutivas e Incompatíveis no Schema do Banco:** Promover alterações de banco de dados sem garantir retrocompatibilidade com a versão estável[14]. Modificar tabelas/colunas de forma que a versão Blue deixe de funcionar inviabiliza o rollback instantâneo, anulando o principal benefício do padrão[14].
* [ ] **Shadow Deployment / Mirror Traffic sem Isolamento de Transações (Modo Dry-Run):** Espelhar tráfego real de escrita para uma versão de sombra sem desativar a gravação transacional ou sem suporte a idempotência estrita[37][38]. Isso causa duplicação indesejada de registros, corrupção da camada de dados e prejuízos operacionais[37].
* [ ] **Dependência de Failover Manual para Sistemas com RTO Agressivo:** Estipular um RTO curto (ex: minutos) enquanto se mantêm processos manuais de decisão e chaveamento de ambientes[2]. A demora na detecção (MTTD) e na intervenção humana inevitavelmente violará o RTO e o SLA contratual[2].
* [ ] **Simular Carga por Simular (Testes de Performance sem Objetivos Claros):** Injetar tráfego em testes de carga sem definir previamente as perguntas a serem respondidas, as jornadas críticas priorizadas e os alvos de SLO (TPS, latência p95, error rate)[46]. O teste gerará dados dispersos e sem valor prático para a engenharia[46].

---

### 4\. CHECKLIST DE GO-LIVE E OPERAÇÃO (DAY-2)

Para autorizar a entrada em produção de um serviço, os seguintes indicadores, alarmes e automações precisam estar ativos:

* [ ] **Four Golden Signals Instrumentados na Borda e nos Serviços (SRE Baseline):**
  * *Latency (Latência):* Medida em percentis (p50, p95, p99) diferenciando o tempo de resposta de requisições bem-sucedidas e com erro[8][9].
  * *Traffic (Tráfego):* Medição de vazão (RPS/TPS) por endpoint, rota, tenant e versão[49].
  * *Errors (Taxa de Erros):* Percentual de falhas em relação ao tráfego total (incluindo falhas de protocolo 5xx e erros semânticos 4xx como timeouts e validações)[8][51].
  * *Saturation (Saturação):* Medição da ocupação de recursos críticos (CPU, memória, thread pools, connection pools e I/O) utilizando o método USE (Utilization, Saturation, Errors)[51].
* [ ] **Observabilidade Estruturada e Correlação dos Três Pilares (Logs, Métricas e Traces):**
  * *Logs Estruturados:* Emissão de registros em JSON contendo obrigatoriamente `correlation_id`, `level` (TRACE, DEBUG, INFO, WARN, ERROR, FATAL) e metadados de domínio para reconstruir a história cronológica da transação[54].
  * *Distributed Tracing e APM:* Injeção de contexto e propagação de cabeçalhos de *trace* entre serviços para localizar exatamente em qual dependência se concentram os gargalos de latência[58][59].
* [ ] **Alertas Proativos Baseados no Consumo do Error Budget e Limiares de SLO:**
  * Alertas configurados sobre métricas conhecidas que indicam sintomas de degradação, disparados antes do comprometimento do SLA do cliente, acelerando o MTTD (Mean Time to Detect)[48].
* [ ] **Automações de Rollback Integradas às Pipelines de CI/CD:**
  * Gatilhos operacionais de rollback automatizado que interrompem o Canary Release ou Blue-Green caso limiares toleráveis de *error rate* ou latência p95 sejam violados durante a janela de observação[11].
* [ ] **Mecanismos de Fallback e Degradação Graciosa em SPoFs Mapeados:**
  * Circuit breakers e feature toggles configurados para chavear o tráfego automaticamente para fluxos alternativos ou desativar recursos não-essenciais sob degradação de um SPoF[4].
