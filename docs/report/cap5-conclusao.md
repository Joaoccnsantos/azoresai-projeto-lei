# Capítulo 5 — Conclusão

---

## 5.1 Síntese do Trabalho Realizado

O presente projeto académico propôs-se a documentar, validar cientificamente e melhorar o sistema de fusão multimodal AzoresAI, já em produção na App Store e Google Play. Ao longo de 16 semanas, o trabalho foi organizado em três fases distintas: documentação arquitetural, implementação de melhorias e validação empírica.

Na fase de documentação (Semanas 1–6), foram produzidos quatro Architecture Decision Records (ADRs) justificando as escolhas tecnológicas do sistema v1 — Firebase Functions vs. AWS Lambda, Jimp vs. OpenCV, Firestore vs. PostgreSQL, e regras de fusão vs. Machine Learning. Foram também produzidos diagramas C4 (contexto e containers) e o schema de dados Firestore, estabelecendo uma base documental rigorosa para o projeto.

Na fase de implementação (Semanas 6–12), foram desenvolvidas e validadas em produção as melhorias v2: detecção dinâmica de período noturno via SunCalc, confidence score de fusão multimodal (0–100%), cache com degradação graciosa, expansão para oito localizações (+2 webcams) e histórico 24h por localização no Firestore. Todas as funcionalidades MoSCoW — Must, Should e Could Have — foram implementadas a 100%.

Na fase de validação (Semana 13), foram realizados oito casos de teste reais, comparando classificações automáticas do sistema com observações visuais diretas, obtendo uma accuracy global de 75% (6/8) e confirmando a correlação entre o confidence score e a qualidade das classificações.

---

## 5.2 Resposta aos Objetivos

Os quatro objetivos específicos definidos no Capítulo 1 foram atingidos:

**Objetivo 1 — Documentação arquitetural via ADRs**
Concluído. Foram produzidos os ADRs 001 a 004, documentando as decisões tecnológicas fundamentais do sistema com análise comparativa de alternativas, contexto de decisão e consequências. Os ADRs estão disponíveis em `docs/architecture/` do repositório académico.

**Objetivo 2 — Identificação e correção de limitações do sistema v1**
Concluído. Foram identificados e corrigidos dois bugs críticos em produção: o threshold `clearThresh` de Ponta Delgada (0.99→0.55), que causava classificações incorretas em condições de boa visibilidade com nuvens cirrus, e o threshold de hora fixa para período noturno, que causava ambiguidade nos equinócios e solstícios. Ambas as correções foram validadas em produção.

**Objetivo 3 — Implementação das melhorias v2**
Concluído. O sistema v2 implementa detecção dinâmica de sunset/sunrise via SunCalc (validada: `IsNight: false` às 15:03h com sunset às 20:24h) e thresholds adaptativos por localização geográfica (`isMountain`, `skyPerc`, `clearThresh` individualizados por câmara). Adicionalmente, foram implementados confidence score, cache com degradação graciosa e expansão para oito localizações.

**Objetivo 4 — Validação científica com casos reais**
Concluído. A sessão de validação de 2 de junho de 2026 produziu oito casos reais com accuracy global de 75%, documentados com screenshots, dados Firestore e análise qualitativa detalhada (Capítulo 4).

---

## 5.3 Contribuições

O projeto produziu as seguintes contribuições concretas:

- **Sistema v2 em produção:** versão 1.0.11 disponível na App Store e Google Play, cobrindo oito localizações de São Miguel com fusão multimodal webcam + OWM;
- **Confidence score:** mecanismo inédito no sistema que quantifica a incerteza de cada classificação, correlacionado empiricamente com a qualidade dos resultados (81% nos acertos vs. 55% nos erros);
- **Base documental:** quatro ADRs, dois diagramas C4, schema Firestore e plano de validação, constituindo documentação técnica reutilizável para evolução futura do sistema;
- **Dataset de validação:** oito casos reais com ground truth humano, screenshots e metadados Firestore, disponíveis em `docs/validation/screenshots/`.

---

## 5.4 Limitações e Trabalho Futuro

### 5.4.1 Limitações Identificadas

A principal limitação técnica identificada na validação é a calibração insuficiente do mecanismo de desempate entre câmara e OWM para localizações com microclimas específicos. Os casos de Mosteiros (câmara prevaleceu incorretamente) e Furnas (OWM prevaleceu incorretamente) partilham esta causa raiz, revelando que a lógica de fusão atual aplica pesos uniformes quando deveria diferenciar por localização.

A dimensão da amostra de validação (oito casos, numa única sessão, com condições predominantemente claras) constitui uma limitação metodológica que impede a extração de métricas de precision e recall por classe de visibilidade com significância estatística.

### 5.4.2 Trabalho Futuro

As melhorias identificadas para versões futuras do sistema são:

1. **Calibração de pesos por localização:** aumentar o peso da câmara para Furnas (microclima de vale geotérmico) e reduzir a sensibilidade da câmara para Mosteiros, através de um parâmetro `camWeight` por localização na configuração `LOCATIONS_WEBCAM`;

2. **Expansão do dataset de validação:** realizar sessões adicionais em condições meteorológicas diversas (nevoeiro, chuva, nebulosidade elevada) para calcular precision e recall por classe com amostra estatisticamente significativa;

3. **Proteção contra debug em produção:** substituir o comentário `// newProStatus = true` por guarda `if (kDebugMode)`, eliminando o risco de release acidental com premium desbloqueado;

4. **Histórico de validação longitudinal:** aproveitar a subcollection `history` implementada (Semana 12) para comparar classificações históricas com registos meteorológicos oficiais do IPMA, expandindo a validação para centenas de casos sem intervenção manual.

---

## 5.5 Reflexão Final

O projeto demonstrou que a fusão de fontes heterogéneas — análise visual de webcams e dados meteorológicos regionais — produz classificações de visibilidade mais robustas do que qualquer fonte isolada, particularmente em localizações de altitude interior onde os dados OWM são insuficientes para capturar microclimas locais. A accuracy de 75% obtida numa única sessão de validação, combinada com a correlação confirmada do confidence score, valida a abordagem arquitetural de regras de fusão explícitas adoptada no ADR-004.

O facto de o sistema estar em produção real, com utilizadores ativos incluindo o primeiro cliente pagante internacional, confere uma dimensão de relevância prática que vai além do contexto académico, demonstrando a viabilidade de sistemas de fusão multimodal de baixo custo para monitorização de condições atmosféricas em destinos turísticos de nicho.
