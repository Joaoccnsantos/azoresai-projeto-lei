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
-  Bug produção corrigido: Threshold `clearThresh` Ponta Delgada estava em 0.99, causando classificação "nublado" em dias clear com nuvens altas. Ajustado para 0.55 (55% pixels azuis). Deploy 30/03 23:45h, validado 31/03 09:20h - funcionando corretamente. Documentado ADR-002 secção "Lições Produção".
-  ADR-002: Jimp vs OpenCV completo (incluindo bug fix threshold + ambiguidade noite vs offline)
-  ADR-003: Firestore vs PostgreSQL completo (document model, tier gratuito, comparação quantitativa)
-  ADR-004: Regras Fusão vs ML completo (decisão consciente Won't Have ML fundamentada)
-  C4 Level 1 (Context): Diagrama ecossistema AzoresAI com Firestore ponte desacoplamento
-  C4 Level 2 (Containers): Arquitetura interna backend (Cloud Functions, Scraper, Analyzer, Fusion, Firestore)
-  Schema Firestore: Diagrama NoSQL collection weather_data + document structure
-  README atualizado progresso semana 3

**Bloqueou:**
- Nada

**Próxima semana:**
- Polimento documentação

## Sem. 4 · 7–11 abr

**Feito:**
-  Organização estrutura: Diagramas C4 movidos para `/docs/diagrams/` (refactoring repositório)
-  Schema Firestore: Diagrama NoSQL collection `weather_data` + document structure completo
-  README: Estrutura repositório atualizada (reflete nova organização)
-  Bug validação identificado: Classificação "noite" 18:16h quando sunset 20:15h (evidencia necessidade sunset dinâmico Should Have)
-  Pasta validation criada: Documentação casos teste com screenshot + análise

**Bloqueou:**
- Nada

**Próxima semana:**
- Polimento ADRs (leitura + correções)


---

## Sem. 5 · 14–20 abr

**Feito:**
-  Fix produção: Furnas removida de TEMP_OFFLINE_WEBCAMS - webcam voltou de manutenção
-  Validação caso 002 documentado: Furnas retoma classificação automática corretamente
-  NOTAS.md validação atualizado (caso 001 + caso 002)
-  Polimento ADR-001: Status/Data adicionados, backticks código corrigidos
-  Polimento ADR-002: Data produção, nº casos (20→10-12), screenshot path corrigidos
-  Polimento ADR-003: Status Accepted adicionado
-  Polimento ADR-004: Nº casos (20→10-12) corrigidos

**Bloqueou:**
- Nada

**Próxima semana:**
- Implementação sunset dinâmico (SunCalc)

---

## Sem. 6 · 22–27 abr

**Feito:**
-  Implementado SunCalc sunset dinâmico (resolve bug noite prematura)
-  Confirmado via logs: IsNight: false às 15:03h, Sunset 20:24h 
**Bloqueou:**  
**Próxima semana:**

---

## Sem. 7 · 28 abr–2 mai · DEMO INTERNA

**Feito:**  
**Bloqueou:**  
**Próxima semana:**

---

## Sem. 8 · 5–6 mai · INTERCALAR

**Feito:**  
**Bloqueou:**  
**Próxima semana:**

---

## Sem. 9 · 7–9 mai

**Feito:**  
**Bloqueou:**  
**Próxima semana:**

---

## Sem. 10 · 12–16 mai

**Feito:**  
**Bloqueou:**  
**Próxima semana:**

---

## Sem. 11 · 19–23 mai

**Feito:**  
**Bloqueou:**  
**Próxima semana:**

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
