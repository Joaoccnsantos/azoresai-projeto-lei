# ADR-004: Regras de Fusão Explícitas vs Machine Learning

**Status:** Accepted  
**Data:** 1 Abril 2026  
**Autor:** João Santos  
**Contexto:** Sistema Fusão Multimodal Visibilidade

---

## Status Implementação

**Sistema v1 já em produção** utiliza regras de fusão explícitas (if-then determinísticas) desde deploy inicial. Esta ADR documenta decisão arquitetural e justifica rejeição consciente de abordagens Machine Learning.

**Evidência produção:** Lógica fusão implementada em `functions/index.js` (linha ~235), executada ~4.320x/mês sem falhas desde deploy.

---

## Contexto

O sistema fusão multimodal combina múltiplas fontes de dados heterogéneas para produzir classificação visibilidade final:

**Inputs disponíveis:**
- **Webcam RGB:** Classificação via métricas Jimp (brightness, contrast, blue ratio) → `clear/cloudy/fog/offline`
- **OpenWeatherMap:** Dados meteorológicos (visibility metros, cloud coverage %, weather code) → `clear/cloudy/fog`
- **Contexto geográfico:** Montanha (altitude >400m) vs costa (<200m)
- **Temporal:** Período noturno (21h-07h) vs diurno

**Objetivo fusão:**
Produzir classificação final **`skyState`** que seja:
1. Mais confiável que qualquer fonte individual isolada
2. Conservadora em segurança (prioriza fog/cloudy quando dúvida)
3. Explicável e auditável (rastreável decisão)
4. Robusta a falhas parciais (webcam offline, OWM timeout)

**Requisitos técnicos:**
- Latência <500ms (execução síncrona Cloud Function)
- Sem treino/retraining necessário (zero MLOps)
- Interpretabilidade completa (júri/professor deve entender lógica)
- Budget €0 (sem custos ML inference)

---

## Decisão

Utilizamos **regras de fusão explícitas determinísticas** implementadas como lógica if-then em JavaScript.

**Lógica implementada (v10.1 produção):**
```javascript
function fusionLogic(camState, owmState, isMountain) {
  // REGRA 1: Fog prioriza (segurança)
  if (owmState === 'fog') return 'fog';
  if (camState === 'fog') return 'fog';
  
  // REGRA 2: Clear em montanha requer concordância (conservador)
  if (camState === 'clear' && !isMountain) return 'clear';
  if (camState === 'clear' && isMountain && owmState === 'clear') return 'clear';
  if (camState === 'clear' && isMountain) return 'cloudy';  // Degradação conservadora
  
  // REGRA 3: Prioriza webcam se online
  if (camState && camState !== 'offline') return camState;
  
  // REGRA 4: Fallback meteorologia
  return owmState;
}
```

**Princípios design:**
1. **Fog prioriza sempre:** Qualquer fonte reporta fog → classificação final fog (segurança turistas)
2. **Montanha conservadora:** Altitude >400m requer concordância ambas sources para "clear" (microclimas imprevisíveis)
3. **Costa permissiva:** Altitude <200m confia mais webcam (condições estáveis)
4. **Degradação cascata:** Webcam (mais precisa) → Meteorologia → Cache (último recurso)

**Rastreabilidade (auditoria):**
```javascript
// Firestore armazena ambos: classificação final + sources originais
{
  skyState: "cloudy",           // Resultado fusão
  skyStateFromCam: "clear",     // Source webcam isolada
  skyStateFromOWM: "cloudy",    // Source OWM isolada (inferido)
  isMountain: true,             // Contexto geográfico
  confidence: 70                // Should Have: 0-100 baseado concordância
}
```
→ Permite auditoria post-mortem: "Porque sistema classificou X quando webcam dizia Y?"

---

## Alternativas Consideradas

### 1. Supervised Learning (Random Forest, XGBoost)
**Descrição:** Treinar modelo classificação multi-class com features extraídas:
```python
# Features hipotéticas
X = [brightness, contrast, blueRatio, owmVisibility, owmClouds, isMountain, hour]
y = ['clear', 'cloudy', 'fog']  # Labels rotulados manualmente
```

**Prós:**
- Aprende padrões não-lineares complexos automaticamente
- Pode descobrir interações features não óbvias (ex: `brightness × hour × isMountain`)
- Performance potencialmente superior se dataset grande (>1000 exemplos rotulados)

**Contras:**
-  **Dataset inexistente:** Projeto não tem 1000+ casos rotulados manualmente ("ground truth")
-  **Rotulação subjetiva:** "Cloudy" vs "Partly Cloudy" - observador humano discorda 20-30% casos
-  **Overfitting risco:** Com 6 localizações × 144 samples/dia = 864 samples/dia, mas alta correlação temporal (condições mudam lentamente). Dataset efetivo menor que aparenta.
-  **Black-box:** Júri/professor não consegue validar decisão modelo ("Por que classificou fog aqui?")
-  **MLOps overhead:** Requer pipeline treino, versionamento modelo, retraining periódico
-  **Infraestrutura:** Precisa TensorFlow/PyTorch runtime (>100MB vs regras 0KB)

**Razão rejeição:** Custo-benefício negativo. Dataset pequeno + rotulação subjetiva + falta interpretabilidade > ganho potencial precisão.

---

### 2. Neural Networks (CNN para imagem, MLP para fusão)
**Descrição:** 
- CNN analisa imagem webcam diretamente (sem Jimp) → embedding 512D
- MLP funde embedding + dados OWM → classificação final

**Prós:**
- End-to-end learning: aprende features automáticas (não depende brightness/contrast manual)
- Transfer learning possível (MobileNet pré-treinado em ImageNet)

**Contras:**
-  **Massive overkill:** Problema não requer deep learning (features RGB manuais suficientes)
-  **Dataset gigante necessário:** CNNs requerem >10k imagens rotuladas para não overfit
-  **Custo computacional:** Inference ~500ms (vs regras <1ms), requer GPU ou TPU
-  **Bundle size:** TensorFlow.js >10MB (vs Jimp 2MB)
-  **Interpretabilidade zero:** "Neurónio camada 3 ativou" não explica nada ao júri

**Razão rejeição:** Solução à procura de problema. Projeto Won't Have ML conforme MoSCoW.

---

### 3. Ensemble Voting (Majority Vote)
**Descrição:** Consultar múltiplas sources e votar:
```javascript
const votes = [camState, owmState];
const fogVotes = votes.filter(v => v === 'fog').length;
const clearVotes = votes.filter(v => v === 'clear').length;
if (fogVotes >= 1) return 'fog';  // Fog veto
if (clearVotes === 2) return 'clear';
return 'cloudy';
```

**Prós:**
- Simples, interpretável
- Democrático: todas sources peso igual

**Contras:**
-  **Perde nuance:** Não considera contexto geográfico (montanha vs costa)
-  **Empates:** Com 2 sources, empate frequente (1 clear + 1 cloudy)
-  **Não conservador:** Se 1 fog + 1 clear → empate, não necessariamente fog
-  **Inferior a regras explícitas:** Regras capturam conhecimento domínio (fog prioriza, montanha conservadora)

**Razão rejeição:** Regras explícitas são "ensemble votação melhorado" com conhecimento domínio injetado.

---

### 4. Bayesian Fusion (Probabilistic)
**Descrição:** Modelar incerteza via probabilidades:
```python
P(clear | camClear, owmClear, mountain) = 
  P(camClear | clear) × P(owmClear | clear) × P(clear | mountain) / Z
```

**Prós:**
- Quantifica incerteza explicitamente
- Teoricamente ótimo (Bayes theorem)
- Confidence score natural (posterior probability)

**Contras:**
-  **Priors desconhecidos:** Não sabemos `P(clear | mountain)` real (requer estatísticas históricas anos)
-  **Likelihood estimação:** `P(camClear | clear)` = taxa acerto webcam - desconhecida sem validação extensa
-  **Complexidade matemática:** Júri Engenharia Software não necessariamente familiar Bayesian inference
-  **Overkill:** Problema não requer probabilidades contínuas (classificação discreta suficiente)

**Razão rejeição:** Elegante teoricamente mas impraticável sem dados históricos para calibrar priors/likelihoods.

---

## Consequências

### Positivas 

**Interpretabilidade total:**
```javascript
// Qualquer pessoa lê código e entende decisão
if (owmState === 'fog') return 'fog';  // "Se meteorologia diz fog, é fog"
```

**Zero overhead infraestrutura:**
- Sem modelos serializados (.pkl, .h5, .onnx)
- Sem runtime ML (TensorFlow, PyTorch)
- Sem GPU/TPU necessários
- Bundle size: 0 bytes (lógica já no código JavaScript)
- **Latência:** <1ms execução (vs 100-500ms inference ML)

**Manutenção simples:**
```javascript
// Ajustar regra = editar 1 linha código
if (camState === 'clear' && isMountain && owmState === 'clear') return 'clear';
//                         ^^^^^^^^^^^^^ Adicionar contexto = trivial
```
→ Sem retraining, sem dataset, sem validação modelo. Git commit + deploy.

**Robustez a edge cases explícitos:**
```javascript
// Tratar casos especiais = adicionar if
if (brightness < 10 && isNight()) return 'night';  // Ambiguidade resolvida ADR-002
```
→ ML pode "esquecer" edge cases raros após retrain. Regras mantêm casos especiais explicitamente.

**Conhecimento domínio injetado:**
- Fog prioriza → conhecimento segurança turismo (conservador)
- Montanha vs costa → conhecimento microclimas Açores
- Degradação cascata → conhecimento confiabilidade relativa sources

→ ML teria que "descobrir" isto via dados. Regras injetam conhecimento diretamente.

---

### Negativas 

**Precisão limitada por conhecimento humano:**
- Regras capturam apenas o que engenheiro conhece/codifica
- Padrões sutis não-óbvios (ex: `brightness × contrast × hour` interação não-linear) não descobertos
- **Mitigação:** 10-12 casos teste. Se precisão <85%, considerar ML em trabalho futuro. Threshold 85% suficiente para MVP.

**Manutenção crescente com complexidade:**
- Adicionar 8ª regra, 9ª regra... código fica nested ifs difícil ler
- Não escala para 50+ localizações com regras específicas cada
- **Mitigação:** Projeto scope 6 localizações (controlado). Se escalar >20, considerar rule engine (Drools) ou ML.

**Sem aprendizagem contínua:**
- Sistema não melhora automaticamente com uso
- Erros classificação não alimentam feedback loop
- **Mitigação:** Changelog documenta ajustes manuais (ex: threshold Ponta Delgada 0.99→0.55). Validação Cap 4 identifica padrões erro para ajustes direcionados.

**Dependência conhecimento domínio:**
- Regras assumem engenheiro entende microclimas Açores
- Risco bias pessoal ("acho que montanha deve ser conservadora" - validado?)
- **Mitigação:** semana 13 (10-12 casos) testa se suposições corretas. MoSCoW Could Have: comparar 6 vs 8 localizações valida regras generalizam.

---

## Validação Científica (Planeada Semana 13)

**Metodologia teste:**
1. Coletar 10-12 casos reais (screenshots webcam + dados OWM + observação humana)
2. Executar sistema: classificação regras explícitas
3. Comparar: sistema vs observação humana (ground truth)
4. Calcular: accuracy, precision, recall por classe (clear/cloudy/fog)

**Critério sucesso:**
- Accuracy global >85%
- Fog recall >90% (segurança: não perder nevoeiro real)
- Clear precision >80% (confiabilidade: clear correto)

**Se falhar (<85% accuracy):**
- Analisar padrões erro (quais casos falham?)
- Ajustar thresholds/regras direcionadamente
- **Se ajustes insuficientes:** Considerar ML como trabalho futuro (não dentro scope projeto 16 semanas)

---

## Comparação Quantitativa

| Critério | Regras Explícitas | Random Forest | CNN | Escolha |
|----------|------------------|---------------|-----|---------|
| Dataset necessário |  0 (conhecimento domínio) |  >500 casos rotulados |  >5000 imagens | Regras |
| Interpretabilidade |  Completa (if-then legível) |  Parcial (feature importance) |  Zero (black-box) |  Regras |
| Latência inference |  <1ms |  50-100ms |  200-500ms |  Regras |
| Infraestrutura |  Zero overhead |  Scikit-learn (~50MB) |  TF.js (>100MB) |  Regras |
| Manutenção |  Edit código + deploy |  Retrain periódico |  MLOps pipeline |  Regras |
| Precisão potencial |  ~85-90% (estimado) |  ~90-95% (se dataset bom) |  ~95%+ (se dataset grande) |  Suficiente MVP |

**Conclusão:** Regras ganham 5/6 critérios práticos. Precisão potencialmente inferior aceitável para projeto académico 6 meses.

---

## Decisão Consciente "Won't Have ML"

Conforme MoSCoW (secção Won't Have):

> **Machine Learning:** Não treinar modelos supervised/unsupervised. Não usar redes neurais (CNN, ResNet). **Justificação:** Regras explícitas são auditáveis, interpretáveis, suficientes para MVP. ML requer dataset rotulado extenso (>1000 imagens) indisponível.

**Esta decisão é estratégica:**
- Projeto foca **fusão multimodal** (combinar sources heterogéneas), não ML state-of-art
- Contribuição académica: validar que regras simples + contexto geográfico > sources individuais
- Júri avalia: capacidade tomar decisões arquiteturais fundamentadas, não usar tecnologia hype

**Trabalho futuro (fora scope):**
Se projeto continuar pós-defesa, ML pode ser explorado com dataset >1000 casos. Arquitetura atual (sources rastreadas separadamente) facilita comparação A/B: regras vs ML.

---

## Referências

- [MoSCoW Won't Have ML](../scope/moscow.md#won't-have)
- Domingos, Pedro. "A Few Useful Things to Know About Machine Learning" (2012) - "More data beats clever algorithms"
- Rule-based vs ML: When to use each (Microsoft Azure Architecture Center)
- App AzoresAI produção: lógica fusão `functions/index.js` linha 235-260
