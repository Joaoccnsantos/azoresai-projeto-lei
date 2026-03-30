# MoSCoW - Sistema Fusão Multimodal Visibilidade

**Projeto:** AzoresAI - Sistema Fusão Multimodal
**Data:** 29 março 2026
**Estudante:** João Santos (1802935)
**Orientador:** Pedro Pestana

---

## Must Have (Essencial para MVP)

### Classificação Visual
- Análise RGB imagens webcam: brightness médio, contrast por desvio padrão, blue ratio
- Classificação automática: clear / cloudy / fog / offline
- Deteção webcam offline via brightness <10
- Suporte 6 localizações: Sete Cidades, Lagoa Fogo, Furnas, Nordeste, Ponta Delgada, Mosteiros

### Deteção Dados Desatualizados
- Timestamp check via HTTP headers Last-Modified
- Invalidação automática se idade >30 minutos
- Marcação source como inválida quando desatualizada
- Fallback para cache ou OWM quando webcam inválida

### Integração Meteorológica
- OpenWeatherMap API: visibility (metros) + cloud coverage (%)
- Função dupla: backup quando webcam indisponível + validação cruzada
- Armazenamento dados OWM em Firestore para auditoria

### Fusão Multimodal
- **Regra 1:** Prioriza fog - qualquer source com fog → classificação final fog
- **Regra 2:** Montanha requer concordância para clear - se webcam clear mas OWM cloudy → final cloudy
- **Regra 3:** Degradação cascata - visual → meteorologia → cache (máx 24h)
- **Regra 4:** Marca incerteza quando discordância irreconciliável entre sources

### Persistência Firestore
- Collection "visibility" com documentos por localização
- Campos obrigatórios: timestamp (ISO8601), classification (enum), sources (array), confidence (0-100)
- Metadados: location_id, context (mountain/coast), degraded (boolean)
- Execução periódica via Cloud Scheduler (trigger cada 10 minutos)

 ### Sunset/Sunrise Dinâmico
- Cálculo automático horário pôr-do-sol/nascer-do-sol para coordenadas São Miguel (37.7°N, 25.7°W)
- Suspensão análise RGB durante período noturno (sunset+30min até sunrise-30min)
- Fallback exclusivo OpenWeatherMap durante noite (análise visual inviável sem luz natural)
- Evita false positives "offline" — brightness noturno natural (~5-10) indistinguível de webcam desligada
- Implementação: biblioteca SunCalc (Node.js, cálculo astronómico offline) ou API Sunrise-Sunset.org
- **Justificação elevação Could→Should:** Identificada ambiguidade crítica em ADR-002. Sem validação horário solar, sistema classifica erroneamente todas as noites como "offline", comprometendo confiança. Essencial para operação 24/7 confiável.

---

## Should Have (Importante mas não bloqueante)

### Thresholds Adaptativos
- Montanha (altitude >400m): brightness <150 = fog, >170 = clear
- Costa (altitude <200m): brightness <100 = fog, >140 = clear
- Configuração por localização em JSON: `{isMountain: true, thresholds: {...}}`

### Confidence Score
- Cálculo 0-100% baseado em concordância sources
- Webcam + OWM concordam (ambos fog) = 95%
- Webcam + OWM discordam mas não conflituam = 70%
- Só cache antigo (>12h) = 50%
- Flag confidence em metadados Firestore

### Cache com Degradação Graciosa
- Armazenar última classificação válida por localização
- Usar se todas sources falharem (webcam offline + OWM timeout)
- Idade máxima 24 horas
- Flag "degraded": true quando usa cache
- Campo "cache_age_minutes" nos metadados

### ADRs Documentados
- Mínimo 4 decisões arquitetura em formato ADR
- Formato obrigatório: Contexto / Decisão / Alternativas Consideradas / Consequências
- Ficheiros: `docs/architecture/adr-00X-titulo.md`

---

## Could Have (Desejável se houver tempo)

### Expansão Densidade Dados
- +2 webcams: Ribeira Grande Norte (costa) + Vila Franca Sul (costa)
- Total 8 localizações em vez de 6
- Validação impacto densidade: 8 sources melhoram cobertura microclimas?
- Comparação casos onde 6 sources insuficientes mas 8 resolvem

### Histórico 24 Horas
- Query Firestore: últimas 144 classificações (24h × 6 execuções/hora)
- Estrutura resposta: array timestamps + classifications
- Deteção padrões: nevoeiro recorrente manhã, dissipa tarde
- Usado para contexto IA chat (se integrado)

### Métricas RGB Avançadas
- Contrast detalhado: desvio padrão por quadrante imagem
- Blue ratio região superior específica (top 30% imagem = céu)
- Brightness por zona: separar montanha de primeiro plano
- Validação: métricas avançadas melhoram precisão?

### Validação Observação Direta
- 20 casos documentados: screenshot + classificação sistema + observação humana
- Categorias: 5 clear, 5 cloudy, 5 fog, 5 offline
- Análise discordâncias: limitação técnica vs ambiguidade inerente
- Documentação: tabela comparativa + justificações

---

## Won't Have (Explicitamente fora de scope)

### Machine Learning
- **Não:** treinar modelos supervised/unsupervised
- **Não:** usar redes neurais (CNN, ResNet)
- **Justificação:** Regras explícitas são auditáveis, interpretáveis, suficientes para MVP. ML requer dataset rotulado extenso (>1000 imagens) indisponível.

### Previsão Futura
- **Não:** prever visibilidade próximas 3-6 horas
- **Não:** modelação temporal (LSTM, ARIMA)
- **Justificação:** Projeto foca tempo real. Previsão requer séries temporais históricas (meses/anos) e está fora do scope académico.

### Integração App Mobile Direta
- **Não:** modificar app AzoresAI Flutter existente
- **Não:** criar endpoints API para consumo mobile
- **Justificação:** App AzoresAI é projeto SEPARADO (já funcional). Este projeto é sistema backend fusão isolado. Integração futura possível mas não no scope LEI.

### Computer Vision Avançada
- **Não:** segmentação semântica (identificar céu vs montanha vs água)
- **Não:** deteção objetos (nuvens, nevoeiro, pessoas)
- **Não:** edge detection, SIFT, HOG
- **Justificação:** Métricas RGB agregadas (brightness, contrast, blue ratio) suficientes para problema binário (visível vs não visível). CV avançada é overkill.

### Análise Vídeo / Streaming
- **Não:** processar frames vídeo em tempo real
- **Não:** deteção movimento (nuvens a mover)
- **Justificação:** Webcams fornecem imagens estáticas. Análise vídeo requer bandwidth alto e processamento pesado (GPU) não disponível em Firebase Functions.

---

## Critérios de Priorização

**Must Have:** Sistema não funciona sem isto. Remover = projeto falha completamente.

**Should Have:** Sistema funciona mas qualidade degrada significativamente. Aumenta robustez/confiabilidade.

**Could Have:** Nice to have. Adiciona valor incremental mas não essencial para demonstração conceito.

**Won't Have:** Conscientemente excluído do scope. Documentado para proteger contra scope creep e justificar escolhas na defesa.

---

## Notas de Implementação

**Ordem execução semanas 5-12:**
1. Must Have primeiro (semanas 5-8): classificador + fusão básica
2. Should Have depois (semanas 9-10): thresholds adaptativos + confidence
3. Could Have se tempo (semanas 11-12): +2 webcams + histórico
4. Won't Have: nunca implementar, só justificar porquê não

**Validação entrega:**
- Intercalar (sem. 8): Must Have 80% completo
- Final (sem. 16): Must Have 100% + Should Have >50%
