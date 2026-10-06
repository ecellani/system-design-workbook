# 🏛️ System Design & Architecture Decision Playbook

Um framework consultivo e operacional de tomada de decisão arquitetural para sistemas distribuídos de alta escala. 

Este repositório consolida requisitos não-funcionais, validações de premissas, matrizes de trade-offs com impacto real (*Blast Radius* e custos operacionais de *Day-2*) e alertas de *anti-patterns* estruturados no formato de checklists práticos para o dia a dia da engenharia.

---

## 🎯 Objetivo

Eliminar a tomada de decisão intuitiva ("feeling") e o *over-engineering* em projetos de software. Este material atua como um **Design Review Gate** e guia para redação e aprovação de **ADRs (Architecture Decision Records)** e **RFCs**, garantindo que nenhuma escolha técnica seja feita sem ponderar explicitamente:

1. **Premissas quantitativas** (SLAs, SLOs, RPO, RTO, proporção leitura/escrita).
2. **Custo real da escolha** (complexidade operacional, latência de rede e governança).
3. **Blast Radius** (raio de impacto em cenários de falha parcial ou catastrófica).
4. **Prontidão operacional (Day-2)** (Golden Signals, alarmes e mecanismos de degradação graciosa).

---

## 🧭 Como Usar no Dia a Dia

Para cada novo componente, refatoração estrutural ou desenho de sistema:

1. **Identifique as Dimensões Críticas do Problema:** Consulte os blocos temáticos correspondentes na tabela abaixo.
2. **Execute o Checklist de Premissas:** Responda às perguntas provocativas antes de desenhar qualquer diagrama.
3. **Pese a Matriz de Trade-offs:** Avalie os custos ocultos e verifique se a justificativa de negócio suporta o preço da solução.
4. **Valide os Red Flags de Over-Engineering:** Certifique-se de que a complexidade da solução não é superior ao problema de negócio.
5. **Documente o Racional (ADR):** Use as conclusões para fundamentar o registro formal da decisão técnica.

---

## 📚 Estrutura do Repositório (Blocos de Decisão)

O repositório está organizado em **10 dimensões arquiteturais essenciais**:

| Bloco | Domínio Arquitetural | Questões & Trade-offs Chave |
| :--- | :--- | :--- |
| **01** | **Consistência Distribuída & Teoremas** | CAP vs. PACELC, Consistência Estrita vs. Eventual, Latência vs. Integridade sob partição. |
| **02** | **Persistência, Dados & Sharding** | SQL vs. NoSQL, Índices (B-Tree vs. LSM), Sharding (Hash vs. Range) e Replicação. |
| **03** | **Caching & Estratégias de Leitura** | Cache-Aside vs. Write-Through/Behind, expiração, *Thundering Herd* e consistência de leitura. |
| **04** | **Estilos Arquiteturais & Domínios** | Monólito Modular vs. Microsserviços, Bounded Contexts, Débito Técnico e Janelas Quebradas. |
| **05** | **Borda, Roteamento & Gateways** | L4 vs. L7 Load Balancing, Reverse Proxies, API Gateways vs. BFFs (Backend for Frontend). |
| **06** | **Comunicação Síncrona & Service Mesh** | REST vs. gRPC, HTTP/2 e HTTP/3, mTLS, Service Mesh em Sidecar vs. Client Libraries. |
| **07** | **Event-Driven & Transações Distribuídas** | Sagas Orquestradas vs. Coreografadas, Event Sourcing, CQRS, Filas vs. Streams (Kafka/RabbitMQ). |
| **08** | **Concorrência, Throughput & Capacidade** | Multithreading, I/O Não-Bloqueante, Capacity Planning, Teoria das Filas (Little's Law, M/M/1). |
| **09** | **Resiliência, Falhas & Topologia Celular** | Circuit Breaker, Retries/Backoff/Jitter, Bulkhead Pattern e Arquitetura em Células (Cell-Based). |
| **10** | **Confiabilidade Operacional & Day-2** | Canary vs. Blue-Green, Testes de Carga/Estresse, Golden Signals, SPOF e Disaster Recovery (RPO/RTO). |
