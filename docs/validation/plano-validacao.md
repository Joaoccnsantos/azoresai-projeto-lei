# Plano de Validação — Sistema Fusão Multimodal

**Projeto:** AzoresAI - Sistema Fusão Multimodal para Estimativa de Visibilidade  
**Estudante:** João Santos · 1802935  
**Semana validação:** Semana 13 (2-6 junho 2026)

---

## Objetivo

Validar 10-12 casos reais comparando a classificação automática do sistema com observação humana direta via webcams SpotAzores, calculando métricas de accuracy, precision e recall por classe de visibilidade.

---

## Metodologia

Para cada caso de validação:

1. **Aceder** à webcam SpotAzores da localização: https://spotazores.com/webcams/sao-miguel/
2. **Observar** visualmente as condições atmosféricas (ground truth humano)
3. **Verificar** classificação do sistema na app AzoresAI ou Firestore
4. **Registar** concordância/discordância + justificação
5. **Screenshot** da webcam como evidência (guardar em `docs/validation/screenshots/`)
6. **Anotar** o confidence score do Firestore (`weather_data > [localização] > confidence`)

---

## Casos Planeados

| Nº | Localização | Condição esperada | Estado | Data |
|----|-------------|-------------------|--------|------|
| 01 | Ponta Delgada | clear | ⏳ | sem. 13 |
| 02 | Nordeste | clear | ⏳ | sem. 13 |
| 03 | Mosteiros | clear | ⏳ | sem. 13 |
| 04 | Ribeira Grande | clear | ⏳ | sem. 13 |
| 05 | Ponta Delgada | cloudy | ⏳ | sem. 13 |
| 06 | Mosteiros | cloudy | ⏳ | sem. 13 |
| 07 | Nordeste | cloudy | ⏳ | sem. 13 |
| 08 | Sete Cidades | fog | ⏳ | sem. 13 |
| 09 | Lagoa do Fogo | fog | ⏳ | sem. 13 |
| 10 | Furnas | fog | ⏳ | sem. 13 |
| 11 | qualquer | offline | ⏳ | sem. 13 |
| 12 | qualquer | night | ⏳ | sem. 13 |

---

## Template Registo de Caso

```
## Caso XXX — [Localização]

**Data/Hora:** dd/mm/2026 HH:MMh  
**Localização:** [nome]  
**Contexto geográfico:** montanha / costa  

**Observação humana (ground truth):**
- Condições: clear / cloudy / fog / offline / night
- Justificação: [o que vejo na webcam]
- Screenshot: [nome ficheiro]

**Classificação do sistema:**
- skyState: [valor Firestore]
- skyStateFromCam: [valor Firestore]
- confidence: [valor]%
- degraded: true / false

**Resultado:** ✅ CORRETO / ❌ INCORRETO

**Análise:**
[Porquê correto ou incorreto? Limitação técnica ou ambiguidade?]
```

---

## Resultados (a preencher semana 13)

| Caso | Localização | Ground Truth | Sistema | Correto | Confidence |
|------|-------------|--------------|---------|---------|------------|
| 01 | Ponta Delgada | | | | |
| 02 | Nordeste | | | | |
| 03 | Mosteiros | | | | |
| 04 | Ribeira Grande | | | | |
| 05 | Ponta Delgada | | | | |
| 06 | Mosteiros | | | | |
| 07 | Nordeste | | | | |
| 08 | Sete Cidades | | | | |
| 09 | Lagoa do Fogo | | | | |
| 10 | Furnas | | | | |
| 11 | | | | | |
| 12 | | | | | |

---

## Métricas (a calcular semana 13)

### Accuracy Global
```
Accuracy = Casos Corretos / Total Casos
```

### Por Classe
| Classe | True Positive | False Positive | False Negative | Precision | Recall |
|--------|--------------|----------------|----------------|-----------|--------|
| clear | | | | | |
| cloudy | | | | | |
| fog | | | | | |
| offline | | | | | |
| night | | | | |  |

### Correlação Confidence Score
```
Alta confiança (>80%): X/Y casos corretos = Z%
Média confiança (60-80%): X/Y casos corretos = Z%
Baixa confiança (<60%): X/Y casos corretos = Z%
```

---

## Links Úteis

- SpotAzores webcams: https://spotazores.com/webcams/sao-miguel/
- Firebase Console (Firestore): https://console.firebase.google.com/project/mindful-span-429621-i7/firestore
- App AzoresAI: App Store / Play Store
