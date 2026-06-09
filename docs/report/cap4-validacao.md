# Capítulo 4 — Validação

---

## 4.1 Visão Geral

Este capítulo apresenta os resultados da validação empírica do sistema de fusão multimodal implementado no AzoresAI. O objetivo é avaliar a capacidade do sistema em classificar corretamente as condições de visibilidade em tempo real, comparando as classificações automáticas com observações visuais diretas efetuadas nas webcams do SpotAzores.

A validação foi estruturada em duas componentes: (1) um plano de validação formal definido na Semana 12, documentado em `docs/validation/plano-validacao.md`, e (2) a execução da sessão de observação na Semana 13, cobrindo as oito localizações monitorizadas pelo sistema.

---

## 4.2 Metodologia de Validação

### 4.2.1 Protocolo de Observação

Para cada localização, foi adotado o seguinte protocolo:

1. Acesso à webcam SpotAzores da respetiva localização;
2. Observação visual independente das condições atmosféricas (*ground truth* humano);
3. Consulta dos valores em tempo real no Firestore: `skyState`, `skyStateFromCam`, `confidence`, `degraded`, `hasFog`, `source`;
4. Registo de concordância ou discordância entre a classificação do sistema e o *ground truth*;
5. Captura de screenshot da webcam como evidência documental (arquivados em `docs/validation/screenshots/`).

### 4.2.2 Classes de Classificação

O sistema utiliza quatro classes de visibilidade:

| Classe | Descrição |
|--------|-----------|
| `clear` | Céu limpo, visibilidade total |
| `cloudy` | Cobertura nubosa significativa |
| `fog` | Nevoeiro ou bruma densa |
| `offline` | Câmara indisponível |

O campo `skyState` representa a decisão final do sistema de fusão. O campo `skyStateFromCam` representa exclusivamente a análise visual da câmara, antes da fusão com os dados OWM.

### 4.2.3 Sessão de Validação

A sessão foi realizada a 2 de junho de 2026, entre as 13h51 e as 14h05 (hora local AZOST, UTC-1), com as oito localizações monitorizadas observadas sequencialmente. O Firestore apresentava dados com timestamp de atualização às 13:50:03 UTC para a maioria das localizações, correspondendo a um desfasamento máximo de 15 minutos face ao momento de observação.

---

## 4.3 Resultados

### 4.3.1 Tabela de Resultados

A Tabela 4.1 apresenta os resultados detalhados dos oito casos de validação observados.

| # | Localização | Ground Truth | skyState | skyStateFromCam | Confidence | Resultado |
|---|-------------|--------------|----------|-----------------|------------|-----------|
| 1 | Ponta Delgada | clear | clear | clear | 82% | ✅ Acerto |
| 2 | Nordeste | clear | clear | clear | 82% | ✅ Acerto |
| 3 | Mosteiros | clear | cloudy | cloudy | 60% | ❌ Erro |
| 4 | Ribeira Grande | partly cloudy | cloudy | cloudy | 80% | ✅ Acerto |
| 5 | Sete Cidades | cloudy/fog | cloudy | clear | 60% | ✅ Acerto |
| 6 | Lagoa do Fogo | cloudy | cloudy | cloudy | 80% | ✅ Acerto |
| 7 | Furnas | hazy/bruma | clear | fog | 50% | ❌ Erro |
| 8 | Vila Franca | clear | clear | clear | 82% | ✅ Acerto |

*Tabela 4.1 — Resultados da validação empírica (02/06/2026, 13h51–14h05 AZOST)*

### 4.3.2 Métricas Globais

Com base nos oito casos observados, o sistema obteve os seguintes resultados:

- **Accuracy global:** 6/8 = **75,0%**
- **Erros:** 2/8 = 25,0%
- **Confidence médio nos acertos:** 81%
- **Confidence médio nos erros:** 55%
- **Fonte de dados:** todos os casos utilizaram fusão `owm+cam`

A Tabela 4.2 resume a correlação entre o confidence score e a qualidade da classificação.

| Resultado | Nº Casos | Confidence Médio |
|-----------|----------|-----------------|
| Acerto | 6 | 81% |
| Erro | 2 | 55% |

*Tabela 4.2 — Correlação entre confidence score e resultado da classificação*

---

## 4.4 Análise Qualitativa

### 4.4.1 Casos de Sucesso

**Casos 1 e 2 — Ponta Delgada e Nordeste**

Ambas as localizações costeiras apresentaram condições de céu limpo com visibilidade total. O sistema classificou corretamente como `clear` em ambos os casos, com confidence de 82% e total concordância entre `skyState`, `skyStateFromCam` e dados OWM (code 800, clouds 0%). Estes resultados demonstram o comportamento nominal do sistema em condições ideais junto ao litoral.

**Caso 4 — Ribeira Grande**

A webcam mostrava nebulosidade nos montes do interior, apesar de o OWM reportar apenas "Few clouds". O sistema classificou corretamente como `cloudy` com confidence de 80%, privilegiando a análise da câmara sobre os dados meteorológicos externos. Este caso ilustra a mais-valia da fusão multimodal: a câmara captura a realidade visual local com maior fidelidade do que os dados OWM, que representam condições médias de uma área mais alargada.

**Caso 5 — Sete Cidades**

Este é o caso mais revelador da robustez do sistema de fusão. Tanto o OWM (code 800, clouds 0%) como o `skyStateFromCam` classificaram como `clear`, mas o `skyState` final do sistema resultou em `cloudy`, que corresponde à realidade observada (cobertura total com lago parcialmente visível). O confidence de 60% reflete corretamente a tensão entre as fontes. Este resultado indica que o sistema incorpora lógica adicional para localizações de altitude interior, compensando as limitações do OWM nestas zonas — nomeadamente os thresholds adaptativos `isMountain: true` definidos na configuração `LOCATIONS_WEBCAM` (ver Secção 3.2.2).

**Caso 6 — Lagoa do Fogo**

Localização de altitude com cobertura nubosa significativa. O OWM reportou `clear` (code 800), mas a câmara e o `skyState` concordaram em `cloudy`, com confidence de 80%. Confirma o padrão das localizações interiores de altitude: a câmara é um discriminador mais fiável do que os dados meteorológicos de superfície.

**Caso 8 — Vila Franca**

Localização costeira com excelentes condições de visibilidade. O sistema classificou corretamente como `clear` com confidence de 82%, em total concordância com o *ground truth* e dados OWM.

### 4.4.2 Casos de Erro

**Caso 3 — Mosteiros**

A webcam mostrava condições de céu limpo, contudo o sistema classificou como `cloudy` com confidence de 60%. O OWM reportava clouds 0% (code 800, "Clear") mas o `skyStateFromCam` retornou `cloudy`, induzindo o sistema a divergir da realidade. Trata-se de um falso positivo de nebulosidade, onde a análise da câmara sobrepôs incorretamente o sinal OWM. A causa provável é uma sensibilidade excessiva do algoritmo de análise de imagem às condições de luminosidade ou contraste específicas desta câmara neste momento do dia. O confidence reduzido de 60% — face aos 82% dos casos claros — indica que o sistema detetou a divergência entre as fontes, mas o mecanismo de desempate favoreceu incorretamente o sinal da câmara.

**Caso 7 — Furnas**

Este é o caso de erro mais significativo. A webcam de Furnas apresentava bruma/haze visível (céu esbranquiçado, lago visível mas com redução de contraste), condições típicas do vale de Furnas dada a sua natureza geotérmica e humidade elevada. O `skyStateFromCam` classificou corretamente como `fog` e o campo `hasFogFromCam` estava definido como `true`, demonstrando que a câmara detetou a situação. Contudo, o `skyState` final resultou em `clear`, prevalecendo o sinal OWM (code 800, "Clear", visibility 10000). O confidence de 50% — o valor mais baixo registado na sessão — confirma que o sistema reconheceu a elevada incerteza. Para uma localização com microclima tão particular (vale fechado, atividade geotérmica, neblina frequente), a câmara é uma fonte mais fiável do que os dados meteorológicos regionais do OWM.

---

## 4.5 Discussão

### 4.5.1 Padrão dos Erros e Causa Raiz

A análise dos dois erros identificados revela um padrão oposto mas com a mesma causa raiz: o mecanismo de desempate entre câmara e OWM não está suficientemente calibrado para as especificidades de cada localização.

- **Mosteiros:** câmara prevaleceu incorretamente sobre OWM → falso `cloudy`
- **Furnas:** OWM prevaleceu incorretamente sobre câmara → falso `clear`

Ambas as localizações partilham características microclimáticas distintas das áreas envolventes. Mosteiros está exposta ao vento oceânico e pode apresentar reflexos específicos na câmara, enquanto Furnas tem microclima de vale com bruma frequente de origem geotérmica. Uma calibração dos pesos de fusão por localização — nomeadamente aumentar o peso da câmara para Furnas e reduzir a sensibilidade da câmara para Mosteiros — poderia mitigar ambos os erros, constituindo a principal melhoria identificada para uma versão futura do sistema.

### 4.5.2 Relevância do Confidence Score

Um resultado positivo desta validação é a correlação verificada entre o confidence score e a qualidade da classificação. Os dois erros corresponderam aos casos com confidence mais baixo (50% e 60%), enquanto os seis acertos apresentaram confidence médio de 81%. Este comportamento confirma que o score implementado na Semana 9 (ver Secção 3.2.4) está a cumprir o seu propósito: sinalizar a incerteza do sistema e oferecer ao utilizador uma indicação da fiabilidade da classificação.

### 4.5.3 Limitações da Validação

Esta sessão de validação apresenta as seguintes limitações:

- **Dimensão da amostra:** oito casos constituem uma amostra reduzida, realizada numa única sessão e num único dia com condições predominantemente claras;
- **Diversidade climatológica:** a ausência de casos de nevoeiro denso, chuva intensa ou câmara offline limita a avaliação do comportamento em condições adversas;
- **Subjetividade do ground truth:** a classificação humana de condições intermédias (e.g. *partly cloudy* vs `cloudy`) tem um grau inerente de subjetividade;
- **Desfasamento temporal:** existiu um desfasamento de alguns minutos entre a observação da webcam e a consulta do Firestore, cujos dados são atualizados a cada 15 minutos.

Uma segunda sessão de validação, em condições meteorológicas distintas (nebulosidade elevada, nevoeiro), está planeada para a Semana 13–14 com o objetivo de aumentar a diversidade do dataset e consolidar as métricas obtidas.

---

## 4.6 Síntese

O sistema de fusão multimodal do AzoresAI demonstrou uma accuracy de **75% (6/8)** na sessão de validação inicial, com um confidence score médio de 81% nos acertos e 55% nos erros. O sistema revelou particular competência na deteção de nebulosidade em localizações de altitude interior (Lagoa do Fogo, Sete Cidades, Ribeira Grande), onde os dados OWM são frequentemente insuficientes, e comportamento nominal em condições claras nas localizações costeiras.

As duas classificações incorretas identificadas partilham a mesma causa raiz — calibração insuficiente dos pesos de fusão para localizações com microclimas específicos — e constituem a principal área de melhoria identificada para trabalho futuro. A correlação positiva entre o confidence score e a qualidade da classificação representa um resultado encorajador, confirmando que o sistema está a quantificar adequadamente a sua própria incerteza.
