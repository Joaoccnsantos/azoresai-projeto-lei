# Sistema de Fusão Multimodal para Estimativa de Visibilidade

> Combina análise visual RGB de webcams com dados meteorológicos para estimativa confiável de visibilidade em tempo real na ilha de São Miguel, Açores — resolvendo o problema de fontes individuais imperfeitas mas complementares.

**Estudante:** João Santos · 1802935  
**Orientador:** Pedro Pestana  
**UC:** Projecto de Engenharia Informática · Universidade Aberta · 2025/26  
**Repositório:** https://github.com/Joaoccnsantos/azoresai-projeto-lei

---

## Estado actual

🟢 **Verde** — A correr conforme planeado. 

---

## Progresso Actual

**Semana 3 (31 mar - 6 abr)** — Documentação arquitetura  **COMPLETA**  
**Próxima fase:** Implementação (inicia semana 5)

**Sistema v1 em produção:** App AzoresAI (App Store/Play Store) já funcional com Jimp + OWM. Projeto académico documenta e melhora sistema existente.

**Completado:**
-  MoSCoW definido ([docs/scope/moscow.md](docs/scope/moscow.md))
-  ADR-001: Firebase Functions vs Lambda
-  ADR-002: Jimp vs OpenCV para análise RGB
-  ADR-003: Firestore vs PostgreSQL
-  ADR-004: Regras Fusão vs Machine Learning
-  Bug produção identificado e corrigido (threshold Ponta Delgada 0.99→0.55)
-  C4 Level 1: Diagrama contexto (ecossistema AzoresAI)
-  C4 Level 2: Diagrama containers (arquitetura interna backend)
-  Changelog semanas 1-3 atualizado

**Próximas etapas (Semana 4):**
- Schema Firestore (diagrama visual collection + documento)
- Polimento documentação (opcional)

---

## MVP Planeado

_Sistema v1 funcional em produção. Projeto académico valida e melhora v1→v2._

**Must Have (v2 melhorias):**
- Sistema fusão multimodal validado cientificamente (webcam RGB + meteorologia OWM)
- Classificador RGB via análise de histogramas de cor e contraste local
- Sunset/sunrise dinâmico (evita ambiguidade noite vs offline)
- Persistência Firestore com execução periódica otimizada

Ver [MoSCoW](docs/scope/moscow.md) para detalhes completos.

---

## Estrutura do Repositório
azoresai-projeto-lei/
├── docs/
│   ├── architecture/        # ADRs
│   │   ├── adr-001-firebase-vs-lambda.md
│   │   ├── adr-002-jimp-vs-opencv.md
│   │   ├── adr-003-firestore-vs-postgresql.md
│   │   └── adr-004-regras-vs-ml.md
│   ├── c4-context.png       # C4 Level 1 
│   ├── c4-containers.png    # C4 Level 2 
│   └── scope/               # MoSCoW, changelog, requisitos
│       ├── moscow.md
│       └── changelog.md
├── functions/               # Firebase Functions (sem. 5+)
└── README.md

---

## Como Instalar e Correr

_Implementação inicia semana 5 (14-25 abril). Instruções serão atualizadas quando houver código._

**Pré-requisitos planeados:**
- Node.js 20+
- Firebase CLI
- Conta Firebase (tier gratuito)
- API key OpenWeatherMap

---

## Decisões de Arquitectura Principais

Ver [docs/architecture/](docs/architecture/) para ADRs completos.

| Decisão | Alternativa considerada | Razão da escolha |
|---------|------------------------|-----------------|
| Firebase Functions | AWS Lambda | Cloud Scheduler integrado, SDK Firestore nativo |
| Jimp | OpenCV | JavaScript puro, sem deps nativas |
| Firestore | PostgreSQL | Document model, tier gratuito, real-time built-in |
| Regras Fusão | Machine Learning | Interpretável, dataset inexistente, zero MLOps |

---

## Referências e IA Utilizada

### Referências Técnicas
- [C4 Model](https://c4model.com) - Modelação arquitetura software
- [MoSCoW Prioritisation](https://www.agilebusiness.org/dsdm-project-framework/moscow-prioritisation.html)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Firebase Functions Docs](https://firebase.google.com/docs/functions)
- [Jimp Documentation](https://github.com/jimp-dev/jimp)

### Ferramentas de IA Utilizadas

| Ferramenta | Para que foi usada |
|-----------|-------------------|
| Claude    | Estruturação documentação técnica, exploração alternativas arquitetura, escrita ADRs |

---

*Última actualização: 6 abril 2026 · Semana 3*
