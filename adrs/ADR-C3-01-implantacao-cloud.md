# ADR-C3-01 — Estratégia de Implantação Cloud

**Projeto:** HealthSync — Plataforma de Telemedicina Inteligente  
**Autora:** Gênnifer Santos Carvalho — 2212569  
**Status:** Proposto  
**Data:** Maio 2026  
**Relacionado a:** ADR-002 (Clean Architecture Hexagonal — Ciclo 2)

---

## Contexto

O HealthSync é uma plataforma de telemedicina inteligente em fase de MVP acadêmico. A arquitetura definida no Ciclo 2 utiliza Clean Architecture Hexagonal com os seguintes componentes de infraestrutura: Apache Kafka (3 brokers) para ingestão IoT, TimescaleDB (PostgreSQL + extensão de séries temporais) para persistência clínica, e múltiplos serviços em Node.js, Java Spring, Python FastAPI e Go.

As restrições que pesaram na decisão de implantação:

- **Stack heterogêneo e stateful:** Kafka com 3 brokers e TimescaleDB não são suportados nativamente por PaaS simples (Railway/Heroku) nem por arquiteturas serverless sem estado persistente.
- **Conformidade LGPD/HIPAA:** Dados clínicos exigem controle de rede (VPC), criptografia em repouso e em trânsito (KMS/TLS) e auditabilidade de acesso — requisitos documentados no ADR-002.
- **Time enxuto em fase MVP:** Time acadêmico sem SRE/DevOps dedicado. A solução precisa ser operável com complexidade proporcional ao estágio do projeto.
- **SLA esperado ≥ 99,5%:** Alertas clínicos em tempo real (SpO₂ < 90%, FC > 120 bpm) exigem alta disponibilidade e capacidade de failover.
- **Budget estimado:** R$800–R$1.800/mês para MVP/staging. Kubernetes gerenciado (EKS) ultrapassaria esse limite sem benefício proporcional nesta fase.

---

## Decisão

**IaaS (Infrastructure as a Service) — Amazon Web Services (AWS)**

| Componente | Serviço |
|---|---|
| Orquestração | Docker Compose em instâncias EC2 (sem Kubernetes no MVP) |
| Banco de dados | TimescaleDB em EC2 dedicado com volume EBS criptografado (gp3) |
| Message broker | Apache Kafka self-managed em 3 instâncias EC2 (1 por broker) |
| Rede | VPC com subnets públicas (API Gateway) e privadas (Kafka, BD, serviços) |
| Identidade | Auth0 externo via IIdentityPort (conforme ADR-002) |
| Monitoramento | Amazon CloudWatch + Prometheus/Grafana |

---

## Trade-off Aceito

**Ganhamos** controle total de rede (VPC, IAM, KMS) viabilizando conformidade LGPD/HIPAA, suporte nativo ao stack heterogêneo (Kafka + TimescaleDB + 4 linguagens) e custo proporcional ao MVP (~R$1.200–R$1.600/mês),

**abrindo mão de** auto-scaling granular por serviço, rolling deployments nativos e observabilidade distribuída automática (sem service mesh),

**porque o contexto prioriza** viabilidade técnica e conformidade regulatória: um PaaS sem VPC configurável comprometeria os requisitos LGPD/HIPAA antes mesmo de considerar custo ou escalabilidade. IaaS + Docker é o ponto de equilíbrio entre controle suficiente para dados clínicos e complexidade proporcional ao estágio de validação do produto.

---

## Consequências

**O que muda na arquitetura (próximos 30 dias):**

1. Containerizar todos os serviços com Dockerfile e compor o ambiente via Docker Compose com redes isoladas por domínio.
2. Provisionar VPC na AWS com subnets públicas (API Gateway) e privadas (Kafka, TimescaleDB, serviços de domínio).
3. Configurar volume EBS gp3 criptografado (KMS) para o TimescaleDB — requisito do RNF01 (Segurança) e da conformidade HIPAA.
4. Implantar CloudWatch para logs e métricas. Configurar alertas de disponibilidade (threshold SLA 99,5%).
5. Documentar runbook de deploy e rollback manual — compensação pela ausência de rolling deployment automático.

**Que nova decisão essa escolha vai forçar no futuro:**

- **Migração para Amazon MSK:** Operar 3 brokers Kafka em EC2 introduz overhead operacional crescente. Quando o volume superar ~2M eventos IoT/min, MSK eliminará a gestão manual sem alterar os adaptadores Kafka (Clean Architecture garante isso).
- **Decisão de orquestração (EKS vs. Docker Compose):** Quando o custo de escalar serviços manualmente superar a complexidade de Kubernetes, a migração para EKS estará habilitada — os contêineres Docker já existem e o domínio hexagonal não depende de infraestrutura.
- **Política de disaster recovery:** IaaS exige definição explícita de backup point-in-time para TimescaleDB (snapshots EBS), retenção de logs Kafka e plano de failover multi-AZ.

---

## Referências

- PRESSMAN, Roger S. *Engenharia de Software: Uma Abordagem Profissional*. 7ª ed. McGraw-Hill, 2011.
- MARTIN, Robert C. *Clean Architecture*. Prentice Hall, 2017.
- COCKBURN, Alistair. Hexagonal Architecture. https://alistair.cockburn.us/hexagonal-architecture
- NYGARD, Michael T. Architecture Decision Records. https://adr.github.io/
- Amazon Web Services. *AWS Well-Architected Framework — Security Pillar*. 2024.
