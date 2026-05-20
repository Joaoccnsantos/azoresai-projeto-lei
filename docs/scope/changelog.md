# Changelog

<!-- Uma entrada por semana, até domingo à noite. -->
<!-- Formato fixo: três linhas por entrada. Não elaborar além do necessário. -->
<!-- O changelog é verificado nas três entregas formais. -->

---

# Changelog

<!-- Uma entrada por semana, até domingo à noite. -->
<!-- Formato fixo: três linhas por entrada. Não elaborar além do necessário. -->
<!-- O changelog é verificado nas três entregas formais. -->

---

## Sem. 1 · 17–21 mar

**Feito:** Proposta projeto redigida: sinopse, MVP com critérios aceitação preliminares, calendário, stack Firebase/Firestore. Submissão 25 março.

**Bloqueou:** Nada

**Próxima semana:** Incorporar feedback professor. Iniciar documentação arquitetura (MoSCoW, ADRs).

---

## Sem. 2 · 24–28 mar

**Feito:** Proposta aprovada com feedback (critérios aceitação formato Given-When-Then para relatórios futuros). MoSCoW completo (Must/Should/Could/Won't). ADR-001 Firebase Functions vs Lambda com análise observabilidade. Estrutura repositório GitHub organizada. README atualizado.

**Bloqueou:** Nada

**Próxima semana:** ADR-002 Jimp vs OpenCV. ADR-003 Firestore vs PostgreSQL. ADR-004 Regras vs ML. C4 Level 1 e 2.

---

## Sem. 3 · 31 mar–6 abr

**Feito:**
- ✅ Bug produção corrigido: Threshold `clearThresh` Ponta Delgada estava em 0.99, causando classificação "nublado" em dias clear com nuvens altas. Ajustado para 0.55 (55% pixels azuis). Deploy 30/03 23:45h, validado 31/03 09:20h.
- ✅ ADR-002: Jimp vs OpenCV completo (incluindo bug fix threshold + ambiguidade noite vs offline)
- ✅ ADR-003: Firestore vs PostgreSQL completo (document model, tier gratuito, comparação quantitativa)
- ✅ ADR-004: Regras Fusão vs ML completo (decisão consciente Won't Have ML fundamentada)
- ✅ C4 Level 1 (Context): Diagrama ecossistema AzoresAI com Firestore ponte desacoplamento
- ✅ C4 Level 2 (Containers): Arquitetura interna backend
- ✅ Schema Firestore: Diagrama NoSQL collection weather_data + document structure
- ✅ README atualizado progresso semana 3

**Bloqueou:** Nada

**Próxima semana:** Polimento documentação

---

## Sem. 4 · 7–11 abr

**Feito:**
- ✅ Organização estrutura: Diagramas C4 movidos para `/docs/diagrams/`
- ✅ Schema Firestore completo
- ✅ README: Estrutura repositório atualizada
- ✅ Bug validação identificado: Classificação "noite" 18:16h quando sunset 20:15h
- ✅ Pasta validation criada: Documentação casos teste com screenshot + análise

**Bloqueou:** Nada

**Próxima semana:** Polimento ADRs (leitura + correções)

---

## Sem. 5 · 14–20 abr

**Feito:**
- ✅ Fix produção: Furnas removida de TEMP_OFFLINE_WEBCAMS - webcam voltou de manutenção
- ✅ Validação caso 002 documentado: Furnas retoma classificação automática corretamente
- ✅ NOTAS.md validação atualizado (caso 001 + caso 002)
- ✅ Polimento ADR-001 a ADR-004 (formatação, datas, nº casos)

**Bloqueou:** Nada

**Próxima semana:** Implementação sunset dinâmico (SunCalc)

---

## Sem. 6 · 22–27 abr

**Feito:**
- ✅ Implementado SunCalc sunset dinâmico (resolve bug noite prematura - Bug 001)
- ✅ Confirmado via logs produção: Sunrise 07:00h, Sunset 20:24h, IsNight: false às 15:03h
- ✅ Fix fetchWithTimeout em falta após refactoring SunCalc
- ✅ NOTAS.md validação atualizado com Fix 001

**Bloqueou:** Nada

**Próxima semana:** Relatório intercalar Cap 1-3

---

## Sem. 7 · 28 abr–2 mai

**Feito:**
- ✅ Relatório intercalar entregue (Cap 1, 2, 3 — 29 abril)
- ✅ Fix app Flutter: remove threshold hora fixa (confia SunCalc backend)
- ✅ Fix sugestões indoor: remove livros como sugestão de dia de chuva
- ✅ Edge-to-edge Android 15 (resolve Android Vitals warning)
- ✅ Build 1.0.10 pronta para submissão

**Bloqueou:** Nada

**Próxima semana:** Submissão lojas + aprovação relatório intercalar

---

## Sem. 8 · 5–11 mai · INTERCALAR

**Feito:**
- ✅ Relatório intercalar aprovado pelo orientador
- ✅ Versão 1.0.10 submetida App Store
- ✅ Versão 1.0.10 submetida Play Store

**Bloqueou:** Nada

**Próxima semana:** Confidence score (Should Have MoSCoW)

---

## Sem. 9 · 7–9 mai

**Feito:**
- ✅ Confidence score implementado no backend (calculateConfidence 0-100%)
- ✅ Confirmado logs produção: cloudy=80%, fog montanha=75% (webcam detecta, OWM não capta)
- ✅ Campo `confidence` persistido no Firestore por localização

**Bloqueou:** Nada

**Próxima semana:** Cache degradação graciosa (Should Have MoSCoW)

---

## Sem. 10 · 12–16 mai

**Feito:**
- ✅ Confidence score validado em produção (logs 15 maio: CONF=80% cloudy, CONF=75% fog montanha)
- ✅ Versão 1.0.10 aprovada App Store e Play Store

**Bloqueou:** Nada

**Próxima semana:** Cache degradação graciosa (Should Have MoSCoW)

---

## Sem. 11 · 19–23 mai

**Feito:**
- ✅ Cache degradação graciosa implementada (Should Have MoSCoW)
- ✅ Fallback automático para cache 24h quando OWM falha
- ✅ Campos `degraded` e `cache_age_minutes` persistidos no Firestore
- ✅ Confirmado logs: Nordeste CAM=offline → OWM fallback → CONF=50% ✅
- ✅ Should Have MoSCoW 100% completo

**Bloqueou:** Nada

**Próxima semana:** +2 webcams (Ribeira Grande + Vila Franca) — Could Have MoSCoW

---

## Sem. 12 · 26–30 mai

**Feito:**
**Bloqueou:**
**Próxima semana:**

---

## Sem. 13 · 2–6 jun

**Feito:**
**Bloqueou:**
**Próxima semana:**

---

## Sem. 14 · 9–13 jun

**Feito:**
**Bloqueou:**
**Próxima semana:**

---

## Sem. 15 · 16–20 jun · PREP. DEFESA

**Feito:**
**Bloqueou:**
**Próxima semana:**

---

## Sem. 16 · 24 jun · ENTREGA FINAL

**Feito:**
**Bloqueou:** —
**Próxima semana:** — Defesa pública.
