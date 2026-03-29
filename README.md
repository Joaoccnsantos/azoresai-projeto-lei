# Sistema de Fusão Multimodal para Estimativa de Visibilidade

> Combina análise visual RGB de webcams com dados meteorológicos para estimativa confiável de visibilidade em tempo real na ilha de São Miguel, Açores — resolvendo o problema de fontes individuais imperfeitas mas complementares.

**Estudante:** João Santos · 1802935  
**Orientador:** Pedro Pestana  
**UC:** Projecto de Engenharia Informática · Universidade Aberta · 2025/26  
**Repositório:** https://github.com/Joaoccnsantos/azoresai-projeto-lei

---

## Estado actual

🟢 **Verde** — A correr conforme planeado. Proposta aprovada 25 março.

---

## Progresso Actual (Semana 3 de 16)

**Fase:** Documentação arquitetura  
**Próxima fase:** Implementação (inicia semana 5)

**Completado:**
- Proposta aprovada (20 março)
- MoSCoW definido
- ADR-001 (Firebase Functions)
- ADRs adicionais em curso
- Diagramas C4 planeados

**MVP Planeado (implementação sem. 5-12):**
- Sistema fusão multimodal (webcam + meteorologia)
- Classificador RGB (brightness, contrast, blue ratio)
- Persistência Firestore com execução periódica

Ver [MoSCoW](docs/scope/moscow.md) para detalhes completos.

---

## Estrutura do Repositório
```
azoresai-projeto-lei/
├── docs/
│   ├── architecture/        # ADRs
│   │   └── adr-001-firebase-vs-lambda.md
│   ├── diagrams/            # C4 diagramas (L1, L2, L3)
│   └── scope/               # MoSCoW, changelog, requisitos
│       ├── moscow.md
│       └── changelog.md
├── functions/               # Firebase Functions (sem. 5+)
└── README.md
```

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
| Firestore | PostgreSQL | Real-time, já usado na app AzoresAI |

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

*Última actualização: 29 março 2026 · Semana 3*
