# Capítulo 3 — Implementação

---

## 3.1 Visão Geral

O sistema v2 é implementado sobre a base do sistema v1 já em produção, introduzindo melhorias incrementais documentadas e validadas. A implementação divide-se em dois repositórios distintos: `azores_ai_v2` (backend Firebase Functions + app Flutter) e `azoresai-projeto-lei` (documentação académica).

---

## 3.2 Backend — Firebase Cloud Functions

### 3.2.1 Classificador RGB (Jimp)

O classificador visual analisa imagens das webcams SpotAzores em tempo real, extraindo três métricas da região superior da imagem (30% do topo — zona correspondente ao céu):

```javascript
function classifySkyForWebcam(img, name, isMountain, skyPerc, clearThresh) {
  const { width, height } = img.bitmap;
  const skyHeight = Math.floor(height * skyPerc);

  let blue = 0, neutral = 0, total = 0;
  let brightnessSum = 0, contrastSum = 0;

  for (let y = 0; y < skyHeight; y += 6) {
    for (let x = 0; x < width; x += 6) {
      const { r, g, b } = Jimp.intToRGBA(img.getPixelColor(x, y));
      total++;
      const pixelMean = (r + g + b) / 3;
      brightnessSum += pixelMean;
      if (b > r + 30 && b > g + 15) blue++;
      if (Math.abs(r - g) < 20 && Math.abs(r - b) < 20) neutral++;
      contrastSum += Math.abs(r - g) + Math.abs(r - b) + Math.abs(g - b);
    }
  }

  const blueRatio = blue / total;
  const avgBrightness = brightnessSum / total;
  // ...
}
```

O classificador implementa ainda detecção automática de webcam offline através do threshold de brightness (`avgBrightness < 10`), e detecção de nevoeiro denso em localizações de montanha através da análise da zona média da imagem (25%-50% da altura).

### 3.2.2 Thresholds Adaptativos por Localização

Cada localização possui configuração individualizada, refletindo as suas características geográficas e as especificidades da webcam instalada:

```javascript
const LOCATIONS_WEBCAM = {
  "Sete Cidades":  { isMountain: true,  skyPerc: 0.20, clearThresh: 0.60 },
  "Lagoa do Fogo": { isMountain: true,  skyPerc: 0.20, clearThresh: 0.60 },
  "Furnas":        { isMountain: true,  skyPerc: 0.20, clearThresh: 0.60 },
  "Nordeste":      { isMountain: false, skyPerc: 0.40, clearThresh: 0.60 },
  "Mosteiros":     { isMountain: false, skyPerc: 0.50, clearThresh: 0.60 },
  "Ponta Delgada": { isMountain: false, skyPerc: 0.15, clearThresh: 0.55 },
};
```

O threshold de Ponta Delgada (0.55 vs 0.60 das restantes) resultou da correção de um bug de produção identificado a 30 de março de 2026: o valor original de 0.99 era impossível de alcançar com nuvens cirrus presentes, causando classificações incorretas em condições de boa visibilidade.

### 3.2.3 Detecção Dinâmica de Período Noturno (Melhoria v2)

O sistema v1 utilizava um threshold de hora fixa para determinar período noturno, causando erros nos equinócios e solstícios. O sistema v2 implementa detecção dinâmica via SunCalc:

```javascript
const SunCalc = require('suncalc');

function checkIsNight() {
  const LAT = 37.7412;
  const LON = -25.6756;
  const now = new Date();
  const times = SunCalc.getTimes(now, LAT, LON);
  const sunriseEnd = new Date(times.sunrise.getTime() + 30 * 60 * 1000);
  const sunsetStart = new Date(times.sunset.getTime() - 30 * 60 * 1000);
  const isNight = now < sunriseEnd || now > sunsetStart;
  console.log(`[SUNCALC] Sunrise: ${times.sunrise.toLocaleTimeString()} | Sunset: ${times.sunset.toLocaleTimeString()} | IsNight: ${isNight}`);
  return isNight;
}
```

A correção foi validada em produção a 21 de abril de 2026: logs confirmam `IsNight: false` às 15:03h, com Sunrise 07:00h e Sunset 20:24h. O bug original foi documentado no caso 001 da pasta `docs/validation/` — sistema classificava "noite" às 18:16h quando o pôr do sol estava agendado para as 20:15h.

### 3.2.4 Confidence Score (Melhoria v2)

O sistema calcula um score de confiança (0-100%) para cada classificação, quantificando o grau de concordância entre as fontes disponíveis. O score é persistido no Firestore e utilizado na fase de validação (Capítulo 4) para correlacionar confiança com accuracy:

```javascript
function calculateConfidence(owmState, camState, finalSkyState, isMountain) {
  // NOITE: SunCalc é preciso → alta confiança
  if (finalSkyState === 'night') return 90;
  // FOG: ambas as fontes concordam → muito alta confiança
  if (owmState === 'fog' && camState === 'fog') return 95;
  // FOG: só uma fonte detectou
  if (owmState === 'fog' && camState !== 'fog') return 80;
  if (camState === 'fog' && owmState !== 'fog') return 75;
  // WEBCAM OFFLINE: só OWM disponível → baixa confiança
  if (!camState || camState === 'offline') return 50;
  // MONTANHA: ambas concordam em clear → boa confiança
  if (isMountain && camState === 'clear' && owmState === 'clear') return 85;
  // MONTANHA: discordância → confiança média
  if (isMountain && camState === 'clear' && owmState !== 'clear') return 60;
  // COSTA: confia na webcam → boa confiança
  if (!isMountain && camState === 'clear') return 82;
  // AMBAS CONCORDAM
  if (camState === owmState) return 80;
  return 60;
}
```

Validado em produção a 15 de maio de 2026: classificações `cloudy` com ambas as fontes em concordância = **80%**, `fog` detetado apenas pela webcam (OWM não capta nevoeiro localizado em montanha) = **75%**.

### 3.2.5 Cache com Degradação Graciosa (Melhoria v2)

Quando o OWM falha, o sistema usa a última classificação válida (máximo 24 horas), garantindo disponibilidade contínua do serviço. Após 24 horas sem dados frescos, o sistema marca a localização como offline:

```javascript
// Verifica idade da cache
const cacheAgeMinutes = lastUpdated
  ? Math.floor((Date.now() - lastUpdated.getTime()) / 60000)
  : null;
const cacheValid = cacheAgeMinutes !== null && cacheAgeMinutes < 1440; // 24h

// OWM falhou → fallback para cache
if (cacheValid) {
  batch.set(..., {
    skyState: cachedState,
    degraded: true,
    cache_age_minutes: cacheAgeMinutes
  }, { merge: true });
} else {
  // Cache expirada → offline
  batch.set(..., {
    skyState: 'offline',
    degraded: true,
    cache_age_minutes: cacheAgeMinutes ?? -1
  }, { merge: true });
}
```

**Fluxo de degradação:**
- OWM disponível → classificação normal (`degraded: false`, `cache_age_minutes: 0`)
- OWM falha + cache < 24h → usa cache (`degraded: true`, `cache_age_minutes: X`)
- OWM falha + cache > 24h → marca offline (`degraded: true`)

---

## 3.3 Gestão de Webcams Temporariamente Indisponíveis

O sistema implementa um mecanismo de override manual para webcams em manutenção, evitando scraping desnecessário e erros nos logs:

```javascript
// Lista de webcams offline temporariamente
const TEMP_OFFLINE_WEBCAMS = []; // Vazio = todas ativas
// Exemplo de uso: ['Furnas'] durante manutenção da webcam
```

Este mecanismo foi utilizado durante a manutenção da webcam de Furnas (março-abril 2026), sendo removido após confirmação de retoma operacional a 17 de abril de 2026.

---

## 3.4 Aplicação Móvel Flutter

### 3.4.1 Arquitetura da App

A aplicação móvel AzoresAI (Flutter) consome os dados do Firestore via listener em tempo real, apresentando as classificações de visibilidade para as seis localizações monitorizadas. A integração com o Gemini 2.5 Flash permite geração de roteiros turísticos personalizados com base nas condições de visibilidade em tempo real.

### 3.4.2 Otimização UI para Android 15 (Edge-to-Edge)

Seguindo as diretrizes Google Android Vitals, a versão 1.0.10 implementa apresentação até às extremidades (edge-to-edge display), utilizando o ecrã completo incluindo as zonas da barra de navegação e barra de estado:

```dart
import 'package:flutter/services.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge);
  SystemChrome.setSystemUIOverlayStyle(const SystemUiOverlayStyle(
    systemNavigationBarColor: Colors.transparent,
  ));

  runApp(const MyApp());
}
```

Esta melhoria resolve os avisos Android Vitals reportados na Play Console para a versão 1.0.9, melhorando a experiência visual em dispositivos modernos (Android 15, SDK 35).

### 3.4.3 Correção Sugestões Indoor

O prompt ao Gemini 2.5 Flash para geração de sugestões em dias de mau tempo foi refinado para focar exclusivamente em atividades turísticas locais relevantes (museus, termas, restaurantes típicos, grutas), eliminando sugestões não contextuais identificadas na versão anterior.

---

## 3.5 Controlo de Versões e Deploy

O sistema segue Conventional Commits para versionamento semântico:

| Versão | Data | Principais alterações |
|--------|------|----------------------|
| 1.0.7 | mar 2026 | Sistema v1 inicial em produção |
| 1.0.8 | mar 2026 | Fix threshold clearThresh Ponta Delgada 0.99→0.55 |
| 1.0.9 | abr 2026 | Fix abertura URLs externos Android (AndroidManifest.xml) |
| 1.0.10 | mai 2026 | SunCalc + edge-to-edge + confidence score + cache degradação graciosa |

O deploy do backend é realizado via Firebase CLI:

```bash
firebase deploy --only functions
```

As Cloud Functions são automaticamente executadas via Cloud Scheduler (analyzeWebcamImage a cada 10 minutos, updateWeatherUpdater a cada 15 minutos), sem necessidade de intervenção manual após deploy.
