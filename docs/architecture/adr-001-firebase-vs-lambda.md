# ADR-001: Firebase Functions vs AWS Lambda

**Autor:** João Santos  
**Contexto:** Sistema Fusão Multimodal Visibilidade

---

## Contexto

O sistema de fusão multimodal necessita de execução serverless periódica para:
- Scraping de webcams SpotAzores (cada 10 minutos)
- Análise RGB de imagens via Jimp
- Consulta OpenWeatherMap API
- Fusão de dados heterogéneos
- Persistência classificações em Firestore

**Requisitos técnicos:**
- Trigger periódico confiável (cron-like, 144 execuções/dia)
- Runtime Node.js para Jimp (biblioteca JavaScript puro)
- Integração nativa com Firestore (app AzoresAI já usa)
- Escalabilidade automática (execuções concorrentes)
- Tier gratuito generoso (projeto académico, sem budget)
- Deploy simples e CI/CD viável

---

## Decisão

Utilizamos **Google Cloud Functions for Firebase** como plataforma serverless.

**Configuração:**
- Runtime: Node.js 20
- Trigger: Cloud Scheduler integrado (cron syntax)
- Deploy: Firebase CLI (`firebase deploy --only functions`)
- Região: europe-west1 (GDPR compliance)

---

## Alternativas Consideradas

### 1. AWS Lambda
**Prós:**
- Tier gratuito: 1M requests/mês
- Ecosystem maduro (CloudWatch logs, X-Ray tracing)
- Suporte múltiplas linguagens (Node, Python, Go, Rust)
- EventBridge para scheduling

**Contras:**
-  Requer configuração EventBridge separada (não integrado)
-  Integração Firestore via SDK externo (não nativo como Firebase Admin)
-  Curva aprendizagem AWS console (IAM, VPC, CloudFormation)
-  Vendor lock-in Amazon (mesmo problema que Google)
-  Deploy mais complexo (SAM/CDK ou console manual)

**Razão rejeição:** Complexidade configuração > benefícios para projeto académico.

---

### 2. Google Cloud Run
**Prós:**
- Containers Docker (flexibilidade total stack)
- Escala a zero (cost-efficient)
- HTTP e background jobs suportados
- Maior controle sobre ambiente execução

**Contras:**
-  Requer Dockerfile (overhead setup)
-  Cloud Scheduler configuração separada (não integrado como Functions)
-  Overkill para tasks simples (scraping + análise RGB)
-  Cold start potencialmente maior (container vs function)

**Razão rejeição:** Complexidade desnecessária. Functions suficientes para caso de uso.

---

### 3. Vercel Serverless Functions
**Prós:**
- Deploy automático via Git push
- Edge network global (baixa latência)
- Developer experience excelente

**Contras:**
-  Timeout 10s no tier gratuito (insuficiente para scraping + análise)
-  Foco em HTTP requests, não cron jobs nativos
-  Sem integração nativa Firestore (requer SDK externo)
-  Pricing proibitivo para execuções frequentes (144/dia)

**Razão rejeição:** Limitações tier gratuito incompatíveis com requisitos.

---

### 4. Heroku Scheduler
**Prós:**
- Add-on simples
- Dynos gratuitos disponíveis

**Contras:**
-  Heroku free tier descontinuado (Nov 2022)
-  Scheduler intervalo mínimo 10min (OK) mas sem precisão (executa "around" horário)
-  Dyno sleep após 30min inatividade (cold start garantido)

**Razão rejeição:** Tier gratuito inexistente pós-2022.

---

## Consequências

### Positivas ✅

**Integração Cloud Scheduler:**
```javascript
// functions/index.js
const functions = require('firebase-functions');

exports.scrapeCameras = functions.pubsub
  .schedule('every 10 minutes')
  .timeZone('Europe/Lisbon')
  .onRun(async (context) => {
    // Scraping + análise + fusão
  });

→ Zero configuração externa. Cron syntax direto no código.

**SDK Firebase Admin nativo:**
```javascript
const admin = require('firebase-admin');
admin.initializeApp();
const db = admin.firestore();

await db.collection('visibility').doc('lagoa_fogo').set({...});

→ Sem autenticação manual. Credenciais automáticas em ambiente Functions.

**Logs centralizados:**
- Firebase Console > Functions > Logs
- Filtros por função, severidade, timestamp
- Integração Cloud Logging (retenção 30 dias tier gratuito)

**Tier gratuito:**
- 2M invocations/mês
- 400k GB-segundos compute
- 200k GHz-segundos CPU
- Projeto: 4.320 invocations/mês (144/dia × 30 dias) = **0.2%** do limite!

**Deploy simples:**
```bash
firebase deploy --only functions
# Rollback: firebase functions:log --only scrapeCameras


---

### Negativas 

**Vendor lock-in Google:**
- Migração futura para AWS/Azure requer reescrever triggers
- Dependência Firebase Admin SDK (não portável)
- **Mitigação:** Lock-in aceitável para projeto académico (6 meses). Lógica core (scraping, análise RGB, fusão) é independente de plataforma.

**Cold start latência:**
- Primeira execução após inatividade: ~1-2 segundos
- **Mitigação:** Irrelevante para cron job (não user-facing). Execuções cada 10min mantêm função "warm".

**Debugging local:**
- Firebase Emulator Suite necessário para testes locais
- Curva aprendizagem inicial ~30 minutos
- **Mitigação:** Investimento pontual. Emulator permite desenvolvimento offline e CI/CD.

**Timeout 540s (9 minutos):**
- Scraping 6 webcams + análise + fusão deve completar <60s
- **Mitigação:** Se timeout, dividir função (scrapeCameras1-3, scrapeCameras4-6). Não antecipado.

---

## Notas de Implementação

### Setup inicial:
```bash
npm install -g firebase-tools
firebase login
firebase init functions
# Selecionar: JavaScript, ESLint yes
cd functions
npm install jimp cheerio axios


### Estrutura esperada:

functions/
├── index.js          # Entry point, exports scrapeCameras
├── scraper.js        # SpotAzores HTML parsing (Cheerio)
├── analyzer.js       # RGB analysis (Jimp)
├── fusion.js         # Multimodal fusion logic
├── package.json
└── node_modules/


### Dependências críticas:
```json
{
  "engines": {"node": "20"},
  "dependencies": {
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^5.0.0",
    "jimp": "^0.22.0",
    "cheerio": "^1.0.0-rc.12",
    "axios": "^1.6.0"
  }
}


### Ambiente variáveis:
```bash
firebase functions:config:set owm.api_key="YOUR_KEY"
# Acesso: functions.config().owm.api_key
```

---

## Validação Decisão

**Critérios sucesso (validar semana 7):**
- ✅ 144 execuções/dia sem falhas
- ✅ Cold start <2s (medido via logs)
- ✅ Custo $0 (tier gratuito suficiente)
- ✅ Deploy <5 minutos
- ✅ Logs acessíveis e debugáveis

**Se falhar:** Reavaliar AWS Lambda como alternativa.

---

## Referências

- [Firebase Functions Docs](https://firebase.google.com/docs/functions)
- [Cloud Scheduler Cron Syntax](https://cloud.google.com/scheduler/docs/configuring/cron-job-schedules)
- [Firebase Emulator Suite](https://firebase.google.com/docs/emulator-suite)
- Projeto AzoresAI app: já usa Firestore, decisão consistente com stack existente
