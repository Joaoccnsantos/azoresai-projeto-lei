# Capítulo 2 — Desenho

---

## 2.1 Visão Geral da Arquitetura

O sistema de fusão multimodal é implementado como uma arquitetura serverless baseada em Google Firebase, composta por dois componentes principais: um backend de análise e fusão (Cloud Functions) e uma base de dados em tempo real (Cloud Firestore).

O backend executa autonomamente a cada 10 minutos via Cloud Scheduler, realizando três operações sequenciais: (1) scraping de imagens das webcams SpotAzores e análise RGB via Jimp, (2) consulta de dados meteorológicos à API OpenWeatherMap, e (3) fusão multimodal das classificações obtidas, persistindo o resultado no Firestore. A aplicação móvel AzoresAI (Flutter) e o motor de IA Gemini 2.5 Flash consomem os dados de forma assíncrona através de listeners em tempo real, garantindo desacoplamento total entre produtor (backend) e consumidores (frontend/IA).

Esta arquitetura assíncrona garante que o sistema de fusão opera independentemente, sem conhecimento dos consumidores finais, seguindo o princípio de separação de responsabilidades.

---

## 2.2 Decisões Arquiteturais

As principais decisões arquiteturais do sistema foram documentadas em Architecture Decision Records (ADRs), disponíveis em `docs/architecture/` do repositório. Cada ADR documenta o contexto, a decisão tomada, as alternativas consideradas e as consequências (positivas e negativas).

### 2.2.1 Plataforma Serverless: Firebase Functions vs AWS Lambda (ADR-001)

O sistema utiliza **Google Cloud Functions for Firebase** (Node.js 20) como plataforma serverless. A decisão foi fundamentada em três critérios principais: integração nativa com Cloud Scheduler (trigger periódico sem configuração externa), SDK Firebase Admin nativo para acesso ao Firestore (sem autenticação manual), e tier gratuito generoso (2M invocações/mês vs 4.320 necessárias).

A alternativa AWS Lambda foi rejeitada por requerer configuração separada do EventBridge para scheduling e integração não-nativa com Firestore, introduzindo complexidade desnecessária para projeto académico de 6 meses. O vendor lock-in Google foi aceite conscientemente, dado que a lógica core (scraping, análise RGB, fusão) é independente de plataforma.

### 2.2.2 Análise de Imagem: Jimp vs OpenCV (ADR-002)

O classificador RGB utiliza **Jimp** (JavaScript Image Manipulation Program) para extração de métricas visuais. A escolha foi determinada pela ausência de dependências nativas (JavaScript puro), bundle size reduzido (~2MB vs >50MB OpenCV) e compatibilidade com ambiente serverless Firebase Functions.

O classificador extrai três métricas da região superior da imagem (30% do topo — zona correspondente ao céu):
- **Brightness médio:** distingue offline (<10), fog (80-150) e clear (>170)
- **Contrast (desvio padrão):** identifica uniformidade de fog (σ<25) vs variabilidade de nuvens (σ>40)
- **Blue ratio:** percentagem de pixels azuis — >55% (costa) ou >60% (montanha) indica céu limpo

A decisão incluiu a identificação e correção de um bug de produção real: o threshold `clearThresh` de Ponta Delgada estava configurado em 0.99 (99% pixels azuis — valor impossível com nuvens cirrus presentes), causando classificação "nublado" em condições de boa visibilidade. O threshold foi corrigido para 0.55, validado em produção a 31 de março de 2026.

### 2.2.3 Persistência: Firestore vs PostgreSQL (ADR-003)

O sistema utiliza **Cloud Firestore** (NoSQL document-oriented) para persistência das classificações. O modelo de dados é naturalmente plano — seis documentos independentes (um por localização), sem relações entre entidades — tornando desnecessários os JOINs característicos de bases de dados relacionais.

As métricas de utilização em produção confirmam a adequação da escolha: ~4.320 writes/mês e ~17.000 reads/mês, representando menos de 1% dos limites do tier gratuito (20k writes/dia e 50k reads/dia), com custo operacional de €0.

O Firestore armazena dois níveis de classificação por localização: `skyState` (resultado da fusão multimodal) e `skyStateFromCam` (classificação da webcam isolada), permitindo rastreabilidade e auditoria das decisões do sistema.

### 2.2.4 Lógica de Fusão: Regras Explícitas vs Machine Learning (ADR-004)

O sistema utiliza **regras de fusão explícitas determinísticas** (if-then em JavaScript) em detrimento de abordagens de Machine Learning. Esta decisão foi fundamentada em três argumentos: ausência de dataset rotulado suficiente (>1000 casos necessários para supervised learning), requisito de interpretabilidade total (júri/professor deve conseguir auditar cada decisão), e zero overhead de infraestrutura ML (sem retraining, sem versionamento de modelos).

A precisão estimada de 85-90% das regras explícitas foi considerada suficiente para MVP, sendo a validação científica desta estimativa um dos objetivos do projeto (Cap 4).

---

## 2.3 Diagramas C4

### 2.3.1 Diagrama de Contexto (C4 Nível 1)

O Diagrama de Contexto apresenta o ecossistema completo do sistema AzoresAI, identificando os atores externos e os fluxos de dados entre componentes.

![Diagrama de Contexto C4 Nível 1](https://raw.githubusercontent.com/Joaoccnsantos/azoresai-projeto-lei/main/docs/diagrams/c4-context.png)

**Figura 1:** Diagrama de Contexto (C4 Nível 1) — Ecossistema AzoresAI.

O Sistema de Fusão Multimodal (foco deste projeto) executa autonomamente cada 10 minutos, realizando scraping SpotAzores e consulta OpenWeatherMap, processando métricas RGB e persistindo classificações no Firestore. A aplicação cliente consome estes dados de forma assíncrona via listener em tempo real, enquanto o motor de IA Gemini 2.5 Flash utiliza o Firestore como contexto para geração de roteiros turísticos personalizados (via function calling).

O Firestore atua como ponte assíncrona entre produtor (backend) e consumidores (frontend/IA), garantindo desacoplamento total: o sistema de fusão opera independentemente, sem conhecimento dos consumidores finais.

### 2.3.2 Diagrama de Containers (C4 Nível 2)

O Diagrama de Containers detalha a arquitetura interna do backend, identificando os módulos funcionais e os fluxos de dados entre eles.

![Diagrama de Containers C4 Nível 2](https://raw.githubusercontent.com/Joaoccnsantos/azoresai-projeto-lei/main/docs/diagrams/c4-containers.png)

**Figura 2:** Diagrama de Containers (C4 Nível 2) — Arquitetura interna backend.

O orquestrador principal (Cloud Functions, Node.js 20) coordena quatro módulos: Scraper Module (Cheerio — scraping HTTP de imagens SpotAzores), Weather Module (Axios — consulta REST API OpenWeatherMap), Analyzer Module (Jimp — análise RGB e classificação visual), e Fusion Module (JavaScript — regras de fusão multimodal). O resultado final é persistido no Cloud Firestore via Firebase Admin SDK.

---

## 2.4 Modelo de Dados

O Firestore organiza os dados numa collection `weather_data` com seis documentos, um por localização monitorizada (Sete Cidades, Lagoa do Fogo, Furnas, Nordeste, Mosteiros, Ponta Delgada).

![Schema Firestore](https://raw.githubusercontent.com/Joaoccnsantos/azoresai-projeto-lei/main/docs/diagrams/firestore-schema.png)

**Figura 3:** Modelo de dados NoSQL — Collection `weather_data` e estrutura do documento.

O esquema foi desenhado para garantir rastreabilidade das decisões de classificação. O campo `skyState` armazena o resultado final da fusão multimodal, enquanto `skyStateFromCam` preserva a classificação isolada da webcam — permitindo auditoria post-mortem: "porque é que o sistema classificou cloudy quando a webcam dizia clear?". Os campos `lastUpdated` e `webcamOnline` permitem detetar dados desatualizados e monitorizar o estado operacional das fontes externas.

```typescript
interface WeatherData {
  name: string;                                              // "Lagoa do Fogo"
  skyState: 'clear' | 'cloudy' | 'fog' | 'night' | 'offline'; // Resultado fusão
  skyStateFromCam?: 'clear' | 'cloudy' | 'fog' | 'offline'; // Webcam isolada
  hasFogFromCam: boolean;
  lastUpdated: Timestamp;
  lastCameraCheck: Timestamp;
  webcamOnline: boolean;
  tempMin: number;                                           // °C
  tempMax: number;                                           // °C
  confidence?: number;                                       // 0-100 (v2 futuro)
}
```

---

## 2.5 Lógica de Fusão Multimodal

A lógica de fusão combina a classificação visual da webcam (`camState`) com a classificação meteorológica OWM (`owmState`) e o contexto geográfico (`isMountain`), seguindo quatro regras por ordem de prioridade:

```javascript
function fusionLogic(camState, owmState, isMountain) {
  // REGRA 1: Fog prioriza sempre (segurança)
  if (owmState === 'fog' || camState === 'fog') return 'fog';

  // REGRA 2: Montanha conservadora (>400m altitude)
  if (camState === 'clear' && isMountain && owmState !== 'clear') return 'cloudy';
  if (camState === 'clear' && isMountain && owmState === 'clear') return 'clear';

  // REGRA 3: Costa permissiva — confia na webcam
  if (camState === 'clear' && !isMountain) return 'clear';

  // REGRA 4: Degradação cascata — fallback meteorologia
  if (camState && camState !== 'offline') return camState;
  return owmState;
}
```

**Regra 1 — Fog prioriza:** Qualquer fonte a reportar nevoeiro resulta em classificação final fog, independentemente da outra fonte. Esta regra reflete uma decisão de segurança: um falso positivo (alertar nevoeiro inexistente) é preferível a um falso negativo (não alertar nevoeiro real).

**Regra 2 — Montanha conservadora:** Localizações com altitude >400m (Sete Cidades, Lagoa do Fogo, Furnas) requerem concordância entre webcam e OWM para classificar "clear". Os microclimas de montanha nos Açores são imprevisíveis — nevoeiro localizado abaixo do nível da câmara pode não ser detetado visualmente. Esta assimetria geográfica constitui a principal contribuição do sistema face a abordagens de fusão simples (e.g., majority vote).

**Regra 3 — Costa permissiva:** Localizações costeiras (<200m altitude) confiam predominantemente na análise visual da webcam, dado que as condições atmosféricas são mais estáveis e a câmara tem visibilidade mais representativa.

**Regra 4 — Degradação cascata:** Em caso de webcam offline ou dados indisponíveis, o sistema degrada graciosamente para a classificação meteorológica OWM, garantindo disponibilidade contínua de classificações mesmo com falha parcial de fontes.

### 2.5.1 Detecção de Período Noturno (Melhoria v2)

O sistema v1 utilizava um threshold de hora fixa (≥21h ou ≤6h) para determinar período noturno, causando classificações incorretas nos equinócios e solstícios — particularmente problemático no verão, quando o pôr do sol ocorre após as 21h nos Açores (bug documentado: sistema classificou "noite" às 18:16h quando o pôr do sol estava agendado para as 20:15h).

O sistema v2 implementa detecção dinâmica via biblioteca SunCalc, calculando os horários exatos de nascer e pôr do sol para as coordenadas de São Miguel (37.7412°N, 25.6756°W) com margem de ±30 minutos:

```javascript
const SunCalc = require('suncalc');

function checkIsNight() {
  const times = SunCalc.getTimes(new Date(), 37.7412, -25.6756);
  const sunriseEnd = new Date(times.sunrise.getTime() + 30 * 60 * 1000);
  const sunsetStart = new Date(times.sunset.getTime() - 30 * 60 * 1000);
  return now < sunriseEnd || now > sunsetStart;
}
```

A correção foi validada em produção a 21 de abril de 2026 — logs confirmam `IsNight: false` às 15:03h, com Sunrise: 07:00h e Sunset: 20:24h.
