# Capítulo 1 — Introdução

**Autor:** João Santos · 1802935  
**Orientador:** Pedro Pestana  
**UC:** Projecto de Engenharia Informática · Universidade Aberta · 2025/26  
**Data:** abril 2026

---

## 1.1 Contexto

A ilha de São Miguel, nos Açores, é um destino turístico de crescente relevância nacional e internacional, atraindo visitantes motivados pela paisagem natural única — lagoas vulcânicas, miradouros de altitude e costa atlântica. A experiência turística nestas localizações é fortemente condicionada pelas condições de visibilidade atmosférica, que nos Açores variam significativamente ao longo do dia devido à orografia da ilha e à influência oceânica.

O projeto surge no contexto de um sistema já em produção — a aplicação móvel AzoresAI (disponível na App Store e Google Play) — que estima condições de visibilidade em tempo real para seis localizações de São Miguel. O sistema v1 combina análise visual de imagens de webcams públicas (SpotAzores) com dados meteorológicos da API OpenWeatherMap, persistindo classificações no Cloud Firestore para consumo pela aplicação móvel Flutter.

O presente projeto académico tem como propósito documentar, validar cientificamente e melhorar o sistema v1, propondo e implementando melhorias arquiteturais fundamentadas — designado sistema v2.

---

## 1.2 Problema

Fontes individuais de dados atmosféricos são imperfeitas e complementares. Webcams podem estar offline, obstruídas ou mal calibradas; dados meteorológicos regionais não capturam microclimas locais — particularmente relevante em localizações de montanha (altitude >400m) onde o nevoeiro pode ser localizado e transitório.

A fusão inteligente de múltiplas fontes heterogéneas, com regras adaptativas ao contexto geográfico (montanha vs. costa), permite classificações mais confiáveis do que qualquer fonte isolada.

---

## 1.3 Objetivos

**Objetivo geral:**  
Documentar, validar e melhorar o sistema de fusão multimodal para estimativa de visibilidade em tempo real na ilha de São Miguel, Açores.

**Objetivos específicos:**
1. Documentar as decisões arquiteturais do sistema v1 através de Architecture Decision Records (ADRs), justificando escolhas tecnológicas face a alternativas consideradas.
2. Identificar e corrigir limitações do sistema v1 em produção, nomeadamente a ambiguidade entre período noturno e webcam offline.
3. Implementar melhorias v2: detecção dinâmica de sunset/sunrise (SunCalc) e thresholds adaptativos por localização geográfica (montanha vs. costa).
4. Validar cientificamente o sistema através de casos de teste reais, comparando classificação automática com observação humana.

---

## 1.4 Metodologia

O projeto segue uma abordagem de engenharia de software iterativa, organizada em 16 semanas. A fase de documentação (semanas 1-6) compreende a análise e registo das decisões arquiteturais do sistema v1, produção de diagramas C4 (contexto e containers) e schema de dados Firestore. A fase de implementação (semanas 6-12) implementa as melhorias v2 identificadas. A fase de validação (semana 13) testa o sistema com 10-12 casos reais, calculando métricas de accuracy, precision e recall por classe de visibilidade.

A priorização de funcionalidades segue o método MoSCoW (Must/Should/Could/Won't Have), documentado em `docs/scope/moscow.md` do repositório do projeto.

---

## 1.5 Estrutura do Documento

O presente relatório está organizado da seguinte forma:

- **Capítulo 2 — Desenho:** Apresenta a arquitetura do sistema, incluindo decisões tecnológicas (ADRs), diagramas C4 e modelo de dados Firestore.
- **Capítulo 3 — Implementação:** Descreve a implementação técnica do classificador RGB, lógica de fusão multimodal e melhorias v2.
- **Capítulo 4 — Testes e Validação:** Apresenta os casos de teste reais e métricas de qualidade do sistema.
- **Capítulo 5 — Conclusão:** Sintetiza os resultados, contribuições e trabalho futuro.
