# Casos Validação Sistema v1

## Bug 001: Classificação "noite" prematura

**Data:** 12 abril 2026, 18:16h  
**Localização:** Ponta Delgada  
**Condições reais:**
- Hora: 18:16 (6:16 PM)
- Pôr do sol: 20:15 (8:15 PM) - **FALTAM 2h para anoitecer!**
- Visibilidade: 30 km ("Muito boa visibilidade")
- Tempo: Sol, dia claro
- Temperatura: 15°C, pouco nublado

**Sistema classificou:** "Noite"   
**Correto seria:**"clear" / "Vista limpa" (PT/EN) (baseado em visibilidade 30km)

**Causa raiz:** Sistema usa threshold hora fixa sem considerar sunset dinâmico (varia com estações)

**Solução v2:** Implementar SunCalc para sunset/sunrise dinâmico (Should Have MoSCoW)

**Evidência:** `bug-noite-prematura.png`

---

## Próximos casos (Semana 13):

- 3 casos "clear" 
- 3 casos "cloudy"️
- 3 casos "fog" 
- 2 casos "offline"

**Total necessário:** 11 casos validação Cap 4

## Caso 002: Furnas - Retoma após manutenção

**Data:** 17 abril 2026  
**Localização:** Furnas  
**Condições reais:** Webcam voltou de manutenção  
**Sistema classificou:** "nublado"   
**Correto:** Sistema retomou classificação automática corretamente  
**Notas:** Removido de TEMP_OFFLINE_WEBCAMS após confirmação webcam online

