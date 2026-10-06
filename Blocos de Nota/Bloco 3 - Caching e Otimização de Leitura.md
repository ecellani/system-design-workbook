### 1\. CHECKLIST DE QUESTIONAMENTOS DE ENTRADA (VALIDAÇÃO DE PREMISSAS)

Antes de aprovar a introdução de qualquer camada de cache ou definir sua topologia, o engenheiro de software líder **deve** responder obrigatoriamente às seguintes validações operacionais de entrada:

* [ ] **Qual é a criticidade e o modelo de consistência aceitável pelo negócio para os dados cacheados?** *Justificativa:* Deve-se avaliar o impacto de leituras desatualizadas (*stale reads*)[2]. Se uma modificação crítica de estado (como a alteração de endereço de remessa ou a desativação de uma conta de usuário com atividade suspeita) exigir consistência imediata em todas as partes do sistema, a arquitetura de escrita tem a obrigação inegociável de invalidar (deletar) as chaves de cache correspondentes ou atualizá-las síncronamente[2]. Isso previne falhas catastróficas, como o envio de mercadorias para o endereço incorreto ou a operação continuada de um usuário bloqueado[2].
* [ ] **A taxa de acertos (Hit Rate) projetada justifica o custo de manutenção e a complexidade da nova camada de infraestrutura?** *Justificativa:* A eficácia real de um sistema de cache é diretamente medida pela relação matemática da Taxa de Acertos: `Hit Rate = (Cache Hits / (Cache Hits + Cache Misses)) * 100`[3][4]. Se os dados sofrem mutações constantes ou são raramente re-acessados pelos clientes, a taxa de acertos será excessivamente baixa[3][5]. Nesses casos, o overhead operacional e de infraestrutura não se justifica, sendo recomendável a completa remoção da camada de cache[3].
* [ ] **Qual é a estratégia de ciclo de vida, expiração (TTL) e evicção recomendada para o volume de dados estimado?** *Justificativa:* O uso de **Time to Live (TTL)** é mandatório em sistemas de larga escala para garantir a reciclagem periódica de informações, prevenir dados obsoletos e evitar o consumo inútil de recursos[6]. Adicionalmente, quando o limite físico de alocação de memória (RAM) do cache for atingido, o sistema precisa de uma política clara de evicção (como LRU ou LFU) para remover itens menos relevantes e abrir espaço para novos dados[7][8].
* [ ] **O cache pretendido operará em escopo isolado de processo (Local) ou requer descentralização e escalabilidade horizontal (Distribuído)?** *Justificativa:* Caches locais baseados em estruturas de memória interna (como Hashmaps) são restritos a uma única execução ou thread[9][10]. Sem lógicas de invalidação e controle estrito, eles induzem vazamentos de memória (*memory leaks*) e saturação de recursos locais[10][11]. Se o sistema for escalável horizontalmente com múltiplos servidores atendendo concorrentemente às mesmas chaves, é obrigatório adotar uma arquitetura de cache distribuído (como Redis ou Memcached)[10].
* [ ] **Os recursos sob alta demanda de tráfego são predominantemente ativos de dados transacionais ou arquivos de conteúdo estático?** *Justificativa:* Se o objetivo é otimizar a entrega de recursos estáticos de alta requisição e baixa taxa de modificação (como imagens, vídeos, arquivos CSS e JS), a solução deve se basear em um **Cache de Conteúdo Distribuído (CDN)**[12][13]. Isso aproxima geograficamente os dados do usuário, reduzindo a latência de rede, aliviando o servidor de origem de picos de tráfego e provendo proteções integradas contra ataques de negação de serviço (DDoS)[13].

---

### 2\. MATRIZ DE TRADE-OFFS E DILEMAS TÉCNICOS

#### [ ] Dilema Arquitetural 1: Cache Local em Memória (HashMap) vs. Caching em Sistemas Distribuídos (Redis / Memcached)

* **O que você GANHA com Cache Local:** Extrema simplicidade de implementação no nível do código, baixíssima latência de acesso ao dado (sem saltos de rede inter-serviços) e alta performance de recuperação local[9][11].
* **O que você PAGA (Custo oculto):** Risco crítico de vazamentos de memória (*memory leaks*) e saturação do host caso não sejam implementados algoritmos rígidos de invalidação[10][11]. Há total isolamento do estado do cache: as réplicas criadas sob escalabilidade horizontal de microsserviços não conseguem compartilhar os dados cacheados entre si de forma transparente[10].
* **Blast Radius:** Saturação de memória no nó da aplicação devido à falta de expiração de dados locais (OOM Killer), derrubando a instância do microsserviço inteiro.
* **Regra Prática de Decisão:**
  * *Escolha* **Cache Local (HashMap)** *se:* O sistema operar em escala simplificada, sob thread ou processo único e autocontido, onde o compartilhamento de dados com nós adjacentes não seja necessário[9][10].
  * *Escolha* **Caching Distribuído** *se:* A aplicação rodar em um ecossistema com escalabilidade horizontal dinâmica e alto paralelismo de leitura concorrente, exigindo que o cache criado por qualquer réplica esteja instantaneamente acessível a todas as outras[10][16].

#### [ ] Dilema Arquitetural 2: Cache-Aside (Lazy Loading) vs. Write-Through (Escrita Dupla)

* **O que você GANHA com Cache-Aside:** O cache é construído sob demanda estrita[17]. Os recursos e o espaço em memória só são ocupados quando o dado é efetivamente solicitado pela lógica da aplicação, o que otimiza os custos e evita cachear informações frias e irrelevantes[5][17].
* **O que você PAGA (Custo oculto):** Latência acentuada no primeiro acesso de leitura a qualquer registro (*cache miss* força a busca na base original e gravação síncrona no cache antes de retornar)[17][18] e alta complexidade na gestão e consistência de dados na aplicação principal[17][19].
* **Blast Radius:** Em caso de limpezas parciais ou totais do cache (*flushes*), a ocorrência de um pico violento de *cache misses* direcionará todo o tráfego de leitura de forma simultânea para o banco de dados persistente, que é o gargalo clássico do sistema, gerando indisponibilidade ou lentidão extrema na cauda de latência[3].
* **Regra Prática de Decisão:**
  * *Escolha* **Cache-Aside** *se:* A carga for majoritariamente voltada para leitura de registros que não mudam de estado constantemente e onde um tempo de resposta maior na primeira requisição (*cache miss*) seja perfeitamente tolerável pelo cliente[5][17].
  * *Escolha* **Write-Through** *se:* O sistema requerer leituras com tempos de resposta de latência extremamente previsíveis e estáveis e o domínio exigir que o cache esteja em sincronia absoluta em tempo real com a base persistente a cada nova gravação[19][21].

#### [ ] Dilema Arquitetural 3: Write-Through (Escrita Dupla) vs. Write-Behind (Lazy Writing)

* **O que você GANHA com Write-Through:** Consistência forte contínua[21]. O cache e o banco persistente são atualizados simultaneamente no momento da gravação, eliminando por completo o risco de inconsistências de estado e minimizando dados desatualizados[19][21].
* **O que você PAGA (Custo oculto):** Penalidade severa na performance e no tempo de resposta de escrita. O aplicativo é obrigado a aguardar o término bem-sucedido de duas operações de escrita (no cache e na base de dados principal) antes de confirmar o sucesso para o cliente final[19][21]. Além disso, exige lógicas robustas de rollback e conciliação caso a persistência falhe após a gravação no cache[21].
* **Blast Radius:** Se o banco de dados principal sofrer degradação ou indisponibilidade, todas as operações de escrita da aplicação falharão ou sofrerão timeout imediato de transação.
* **Regra Prática de Decisão:**
  * *Escolha* **Write-Through** *se:* A integridade imediata e a consistência forte entre cache e disco forem requisitos funcionais cruciais do domínio[19][21].
  * *Escolha* **Write-Behind** *se:* O objetivo primordial da arquitetura for maximizar o throughput de escrita e reduzir a latência de gravação imediata para a aplicação[22][23]. O sistema grava instantaneamente no cache e retorna sucesso, delegando a persistência síncrona em disco para um processo ou fila paralela em segundo plano[22][23].

#### [ ] Dilema Arquitetural 4: Políticas de Evicção: Least Recently Used (LRU) vs. Least Frequently Used (LFU) vs. First In, First Out (FIFO)

* **O que você GANHA com LRU:** Alta eficiência para a maioria dos padrões de tráfego, eliminando primeiro o item sem acessos recentes sob a premissa válida de que dados ociosos são menos propensos a consultas futuras[8].
* **O que você PAGA (Custo oculto):** Desempenho reduzido se o padrão de acesso do sistema envolver varreduras completas sequenciais de dados frios que sobrescrevem os dados quentes recentes na memória.
* **O que você GANHA com LFU:** Excelente retenção de registros verdadeiramente populares em termos acumulados de longo prazo, descartando os itens que possuem menor frequência histórica de requisição[8].
* **O que você PAGA (Custo oculto):** Custo de processamento e complexidade computacional acentuados para manter e atualizar continuamente o rastreamento da frequência de acessos de cada chave ativa[8].
* **O que você GANHA com FIFO:** Complexidade de código extremamente baixa e overhead mínimo de execução, descartando dados em ordem estritamente sequencial de inserção[24].
* **O que você PAGA (Custo oculto):** Baixa efetividade prática[24]. Pode excluir precocemente dados cruciais e de altíssima relevância operacional simplesmente porque foram inseridos no início do array de cache, provocando quedas drásticas do Hit Rate[24].
* **Blast Radius:** A escolha inadequada da política de evicção acelera a rotação indevida de dados na memória RAM, provocando picos de escrita concorrente e degradação contínua da capacidade do banco de dados de origem.
* **Regra Prática de Decisão:**
  * *Escolha* **LRU** *se:* Estiver desenhando uma solução de cache geral para dados de comportamento de tráfego dinâmico[8].
  * *Escolha* **LFU** *se:* O sistema operar com um catálogo estável de itens onde a popularidade mude lentamente no tempo e o monitoramento de frequência compense o overhead[8].
  * *Escolha* **FIFO** *se:* O domínio fizer uso de estruturas sequenciais ordenadas de curta duração e onde a frequência ou recência de acesso não tenham relevância técnica[24].

---

### 3\. ANTI-PATTERNS E RED FLAGS DE OVER-ENGINEERING

Como revisor de arquitetura principal, acione o alerta máximo ou bloqueie a proposta de design caso identifique os seguintes comportamentos indevidos:

* [ ] **Anti-pattern 1: Implementação de Cache Local (HashMap) sem limites físicos de expiração (TTL) ou Invalidação ativa.** *Red Flag:* Salvar objetos dinâmicos de domínio em memória local sem definir limites de volume, TTL de expiração ou deleção de chaves após mutações[10][11]. Em ambientes de produção contínuos, essa ausência de descarte causará vazamentos de memória (*memory leaks*) e queda inevitável por esgotamento de recursos (*Out-of-Memory*)[6][10].
* [ ] **Anti-pattern 2: Caching de dados com perfil de alta mutabilidade ou de baixíssima frequência de requisição.** *Red Flag:* Implementar camadas de cache para informações que mudam constantemente a cada segundo ou que raramente são consultadas pelos usuários[3][5]. O sistema acumulará um volume excessivo de *cache misses* e um baixíssimo *Hit Rate*[3][18], gerando custos de infraestrutura e complexidade de conciliação de consistência para nenhum ganho real de desempenho[3].
* [ ] **Anti-pattern 3: Ausência de fluxos automatizados de invalidação de cache de CDN nas pipelines de CI/CD.** *Red Flag:* Atualizar arquivos estáticos cruciais do sistema (Assets, CSS ou scripts JS) na base original durante um deploy e não acionar a invalidação programática de chaves ou grupos na CDN distribuída[15]. Usuários finais continuarão recebendo arquivos obsoletos de servidores geográficos mais próximos, provocando inconsistências operacionais e quebras de fluxo na interface do cliente[14][15].
* [ ] **Anti-pattern 4: Negligenciar a remoção/atualização de chaves de cache em escritas originais de domínio crítico.** *Red Flag:* Alterar dados críticos na base persistente original (ex: redefinir o status de um usuário desativado por fraude ou atividade suspeita) e não realizar a deleção ou atualização mandatória correspondente na chave do cache[2]. O usuário continuará operando normalmente em nós replicados devido a leituras inconsistentes de estado válidos no cache[2].

---

### 4\. CHECKLIST DE GO-LIVE E OPERAÇÃO (DAY-2)

Para autorizar a entrada em produção de qualquer topologia com suporte a caching, os times de Engenharia de Confiabilidade de Sites (SRE) devem atestar a cobertura dos seguintes monitoramentos, alertas e mitigações:

* [ ] **Configuração e Alerta sobre a Métrica de Taxa de Acertos (Hit Rate):** *Operação:* Rastrear em tempo real os eventos fundamentais de **Cache Hit** (sucessos diretamente na camada rápida) e **Cache Miss** (falhas que exigiram varreduras lentas no banco de dados)[18][25]. *Alerta Day-2:* Disparar alarmes se a Taxa de Acertos (**Hit Rate**) calculada continuamente cair abaixo de um limiar mínimo tolerável de eficiência (ex: abaixo de 60%), sinalizando a necessidade urgente de revisão de evicção, ampliação de TTL ou remoção da camada[3].
* [ ] **Monitoramento de Saturação de Memória e Taxas de Evicção:** *Operação:* Capturar e expor o percentual de uso de memória física dedicada em relação ao limite contratado na instância de cache[7][10]. *Alerta Day-2:* Alertas de saturação quando o espaço atingir 80% do limite físico, integrados com contadores ativos de evicção sequencial de dados para evitar que chaves ativas quentes sejam descartadas prematuramente para a entrada de registros novos[7].
* [ ] **Proteção contra Surtos de Cache Misses Massivos (Avalanche / Stampede):** *Operação:* Implementar playbooks ativos de aquecimento prévio de chaves (*cache warmup*) e lógicas de contenção de latência de cauda (p95 e p99) no banco original[3]. *Mecanismo de Fallback:* Em momentos de reinicialização de instâncias ou limpezas totais de cache (*flushes*), o sistema deve contar com mecanismos de degradação graciosa ou travas de taxa para prevenir que picos súbitos de *cache misses* destruam o banco de dados de origem[3][18].
* [ ] **Monitoramento de Lag de Ingestion e Tamanho de Filas no Write-Behind:** *Operação:* Se o modelo de persistência adotado for o Write-Behind, medir o volume acumulado de operações pendentes na fila assíncrona ou broker de mensageria que aguardam gravação física em disco[23]. *Alerta Day-2:* Alarme prioritário se o tamanho da fila de sincronização ou a taxa de latência de processamento de gravação crescer de forma exponencial, sinalizando lentidão crônica ou falha total na base de dados original[23].
