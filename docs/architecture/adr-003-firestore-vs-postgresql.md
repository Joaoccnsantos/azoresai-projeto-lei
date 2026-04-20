# ADR-003: Firestore vs PostgreSQL para Persistência

**Status:** Accepted  
**Data:** 31 março 2026  
**Autor:** João Santos  
**Contexto:** Sistema Fusão Multimodal Visibilidade

---

## Status Implementação

**Sistema v1 já em produção** utiliza Firestore desde deploy inicial. Esta ADR documenta decisão arquitetural e analisa alternativas consideradas.

**Evidência produção:** Collection `weather_data` com 6 documentos (localizações), ~4.320 writes/mês, custo incluído no tier gratuito Firebase.

---

## Contexto

O sistema fusão multimodal requer persistência de classificações visibilidade em tempo real para:
- Armazenar resultado fusão (webcam RGB + meteorologia OWM) cada 10 minutos
- Consultas rápidas por localização (6 spots: Sete Cidades, Lagoa Fogo, Furnas, Nordeste, Mosteiros, Ponta Delgada)
- Histórico opcional 24h (144 classificações/localização)
- Acesso direto app mobile Flutter (já usa Firebase Auth)

**Modelo de dados necessário:**
```javascript
// Documento por localização
{
  name: "Lagoa do Fogo",              // String
  skyState: "clear",                   // Enum: clear/cloudy/fog/night/offline
  skyStateFromCam: "clear",            // Classificação webcam isolada
  hasFogFromCam: false,                // Boolean
  lastUpdated: Timestamp,              // Firebase Timestamp
  lastCameraCheck: Timestamp,
  webcamOnline: true,                  // Boolean
  tempMin: 12.5,                       // Number (°C)
  tempMax: 18.3,
  confidence: 85                       // 0-100 (Should Have futuro)
}
```

**Requisitos técnicos:**
- Latência read <100ms (queries app mobile)
- Consistência eventual aceitável (dados atmosféricos mudam lentamente)
- Writes baixa frequência: 6 docs × 6 updates/hora = 36 writes/hora
- Relações inexistentes: cada localização é documento independente
- Integração Firebase Functions nativa (já decidido ADR-001)

---

## Decisão

Utilizamos **Google Cloud Firestore** (modo nativo) como base de dados NoSQL document-oriented.

**Estrutura:**
```
weather_data (collection)
  ├── Sete Cidades (document)
  ├── Lagoa do Fogo (document)
  ├── Furnas (document)
  ├── Nordeste (document)
  ├── Mosteiros (document)
  └── Ponta Delgada (document)
```

**Queries típicas:**
```javascript
// Read single location (app mobile)
const doc = await db.collection('weather_data').doc('Lagoa do Fogo').get();

// Read all locations (dashboard admin futuro)
const snapshot = await db.collection('weather_data').get();
snapshot.forEach(doc => console.log(doc.data()));

// Write fusion result (Cloud Function)
await db.collection('weather_data').doc(locationName).set({
  skyState: finalClassification,
  lastUpdated: admin.firestore.Timestamp.now(),
  ...
}, { merge: true });
```

---

## Alternativas Consideradas

### 1. PostgreSQL (Cloud SQL)
**Prós:**
- Base de dados relacional madura, ACID completo
- SQL queries poderosas (JOINs, agregações complexas)
- Integridade referencial via foreign keys
- ORMs maduros (Prisma, TypeORM, Sequelize)
- Backups automáticos, point-in-time recovery

**Contras:**
-  **Setup complexo:** Requer provisionar instância Cloud SQL, configurar VPC, gerir conexões
-  **Custo:** Tier gratuito inexistente. Cloud SQL mínimo ~$10/mês (db-f1-micro)
-  **Overkill:** Projeto não tem relações complexas (sem JOINs necessários)
-  **Latência:** Conexão TCP overhead vs Firestore REST API direto
-  **Integração Firebase Functions:** Requer biblioteca externa (`pg`), connection pooling manual

**Razão rejeição:** Complexidade e custo injustificados para modelo dados plano (key-value essencialmente). Relações inexistentes no domínio (cada localização independente).

---

### 2. MongoDB (Atlas ou self-hosted)
**Prós:**
- Document model similar Firestore (JSON-like)
- Queries flexíveis, agregações poderosas (`$group`, `$match`)
- Indexes compostos, text search
- Tier gratuito Atlas (512MB)

**Contras:**
-  **Setup externo:** Atlas separado de Firebase, credenciais manuais
-  **Integração Firebase:** SDK externo (`mongodb`), não nativo
-  **Sem real-time:** Requer Change Streams (overhead) para updates live
-  **Latência:** Network hop extra (Firebase Functions → Atlas cluster)

**Razão rejeição:** Firestore oferece mesmas vantagens document model + integração nativa Firebase + real-time built-in. MongoDB seria escolha se projeto fosse AWS/Azure (não Google Cloud).

---

### 3. MySQL (Cloud SQL)
**Prós:**
- Base de dados relacional open-source popular
- Tier gratuito limitado via Always Free (OCI, não GCP)
- ORMs maduros

**Contras:**
-  Mesmos problemas PostgreSQL: setup, custo, overkill
-  Menos features que PostgreSQL (JSON support inferior)
-  Não oferece vantagens vs PostgreSQL para caso de uso

**Razão rejeição:** Inferior a PostgreSQL para novos projetos. PostgreSQL já rejeitado.

---

### 4. Redis (Memorystore)
**Prós:**
- Latência ultra-baixa (<1ms)
- Ideal para cache, dados temporários
- Estruturas dados ricas (hashes, sets, sorted sets)

**Contras:**
-  **In-memory only:** Persistência secundária, não é base dados primária
-  **Custo:** Memorystore mínimo ~$50/mês
-  **Volatilidade:** Dados históricos perdidos se instância reinicia
-  **Overkill:** Latência Firestore (<100ms) já suficiente

**Razão rejeição:** Designed para cache, não persistência primária. Custo proibitivo para tier gratuito projeto.

---

## Consequências

### Positivas 

**Integração Firebase Admin SDK nativa:**
```javascript
const admin = require('firebase-admin');
admin.initializeApp();
const db = admin.firestore();

// Zero configuração: credenciais automáticas em Cloud Functions
await db.collection('weather_data').doc('Lagoa do Fogo').set({...});
```
→ Sem connection strings, sem gestão credenciais, sem pooling manual.

**Real-time listeners (bonus não usado ainda):**
```javascript
// App mobile pode subscrever updates live (Could Have futuro)
db.collection('weather_data').doc('Lagoa do Fogo')
  .onSnapshot(snapshot => {
    console.log('Nova classificação:', snapshot.data().skyState);
  });
```
→ WebSockets automáticos, sem implementar polling.

**Document model natural para dados heterogéneos:**
```javascript
// Campos opcionais sem schema rígido
{
  skyState: "clear",
  confidence: 85,          // Adicionado v2, não existe v1
  degraded: false,         // Should Have futuro
  cache_age_minutes: null  // Só presente quando usa cache
}
```
→ Schema evolution sem migrations. Adicionar campos não quebra código existente.

**Tier gratuito generoso:**
- 50k reads/dia = 1.5M/mês (projeto: ~17k/mês queries app)
- 20k writes/dia = 600k/mês (projeto: ~4.3k/mês updates Functions)
- 1GB storage (projeto: <1MB - 6 docs × ~500 bytes)
- **Custo atual: €0** ✅

**Queries simples suficientes:**
```javascript
// Caso uso 95%: read single doc by ID
const doc = await db.collection('weather_data').doc(locationId).get();

// Caso uso 5%: read all (dashboard admin)
const all = await db.collection('weather_data').get();
```
→ Sem JOINs complexos, sem agregações pesadas. Firestore perfeito para isto.

---

### Negativas 

**Queries limitadas vs SQL:**
- Sem JOINs nativos (não relevante: sem relações no domínio)
- Sem agregações complexas (SUM, AVG, GROUP BY distribuído)
- Inequality filters limitados (máx 1 campo `<`, `>` por query)
- **Mitigação:** Modelo dados plano evita necessidade queries complexas. Se futuro requerer analytics (ex: "média confidence últimas 24h por região"), implementar agregação em Cloud Function ou BigQuery export.

**Vendor lock-in Google:**
- Migração para PostgreSQL/MongoDB requer reescrever queries + data export
- API Firestore específica Google (não é MongoDB Wire Protocol)
- **Mitigação:** Lock-in já aceite em ADR-001 (Firebase Functions). Consistência stack justifica. Dados volume baixo (~MB) torna export/import trivial se necessário.

**Sem transações multi-document robustas:**
- Transações existem mas com limitações (máx 500 docs, sem queries dentro)
- Não é problema para caso uso (writes sempre single-document)
- **Mitigação:** Projeto não requer atomicidade multi-document. Cada localização update independente.

**Custo potencial em escala (não aplicável projeto):**
- Após tier gratuito: $0.06/100k reads, $0.18/100k writes
- **Cenário hipotético:** 10M reads/mês = $6, 1M writes/mês = $1.80
- **Mitigação:** Projeto académico 6 meses bem dentro tier gratuito. App produção (17k reads/mês) também gratuito indefinidamente.

---

## Modelo de Dados Detalhado

### Collection: `weather_data`

**Documento tipo:**
```typescript
interface WeatherData {
  // Identificação
  name: string;                    // "Lagoa do Fogo" (usado como doc ID também)
  
  // Classificação fusão (resultado final)
  skyState: 'clear' | 'cloudy' | 'fog' | 'night' | 'offline';
  
  // Classificação sources individuais (rastreabilidade)
  skyStateFromCam?: 'clear' | 'cloudy' | 'fog' | 'offline';  // Webcam isolada
  hasFogFromCam: boolean;                                     // Flag fog webcam
  
  // Meteorologia OWM
  tempMin: number;                 // °C
  tempMax: number;                 // °C
  
  // Timestamps
  lastUpdated: Timestamp;          // Última atualização fusão
  lastCameraCheck: Timestamp;      // Última tentativa scraping webcam
  
  // Estado operational
  webcamOnline: boolean;           // Webcam respondeu?
  
  // Should Have (futuro)
  confidence?: number;             // 0-100, baseado concordância sources
  degraded?: boolean;              // True se usa cache antigo
  cache_age_minutes?: number;      // Idade cache se degraded=true
}
```

**Indexes:**
- Nenhum necessário (queries sempre por doc ID)
- Firestore cria index automático no doc ID

**Regras segurança (Firebase Rules):**
```javascript
// rules_v1.rules
service cloud.firestore {
  match /databases/{database}/documents {
    match /weather_data/{location} {
      // Leitura: qualquer utilizador autenticado (app mobile)
      allow read: if request.auth != null;
      
      // Escrita: só Cloud Functions (service account)
      allow write: if false;  // Functions usa Admin SDK (bypass rules)
    }
  }
}
```

---

## Comparação Quantitativa

| Critério | Firestore | PostgreSQL | MongoDB | Escolha |
|----------|-----------|------------|---------|---------|
| Setup inicial |  Imediato (já ativo Firebase) |  30min+ (Cloud SQL provision) | 20min (Atlas signup) |  Firestore |
| Integração Functions |  Nativa (Admin SDK) |  Externa (`pg` lib) |  Externa (`mongodb` lib) |  Firestore |
| Latência read |  50-100ms |  100-200ms (TCP) |  100-150ms (network hop) | Firestore |
| Tier gratuito |  50k reads/dia |  Inexistente | 512MB Atlas | Firestore |
| Real-time |  Built-in (WebSockets) |  Manual (polling) |  Change Streams |  Firestore |
| Queries complexas |  Limitadas |  SQL completo |  Agregações ricas |  Não necessário |
| Schema evolution |  Schemaless |  Migrations |  Schemaless |  Firestore |

**Conclusão:** Firestore ganha em 6/7 critérios relevantes para projeto.

---

## Validação Decisão

**Critérios sucesso (validar semana 8):**
-  Latência read <100ms (99th percentile)
-  Zero downtime desde deploy v1
-  Custo €0 (tier gratuito suficiente)
-  Zero configuração adicional além Admin SDK

**Métricas produção atuais (31/03/2026):**
- Writes: ~4.320/mês (36/hora × 24h × 30d)
- Reads: ~17.000/mês (estimativa app mobile)
- Storage: <1MB (6 docs × ~500 bytes)
- **Custo: €0** 

**Se falhar:** Alternativa seria MongoDB Atlas (document model + tier gratuito), não PostgreSQL (relacional desnecessário).

---

## Referências

- [Firestore Documentation](https://firebase.google.com/docs/firestore)
- [Firestore Pricing](https://firebase.google.com/pricing)
- [Choosing Database: Firestore vs Realtime Database](https://firebase.google.com/docs/database/rtdb-vs-firestore)
- NoSQL Data Modeling: Firestore Best Practices (Google Cloud)
- App AzoresAI produção: collection `weather_data` em uso desde deploy inicial
