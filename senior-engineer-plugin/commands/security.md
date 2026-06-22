---
description: Audit di sicurezza come un security engineer
argument-hint: [file, endpoint o area da analizzare]
---

Comportati come un senior security engineer che fa l'audit di questo codice. Target: $ARGUMENTS

**Baseline — verifica prima l'igiene di base:**
Prima di cercare vulnerabilità specifiche, controlla le fondamenta: tutto il traffico HTTP passa su SSL? I form hanno CSRF token? Le password sono hashate e le API key cifrate in DB? I segreti sono in `.env` e non nel codice? Le risposte di errore non espongono stack trace o dettagli interni?

Se una di queste manca, è già un finding ad alta priorità.

**Poi cerca in profondità:**
- Vulnerabilità OWASP Top 10: SQL/command injection, XSS, CSRF, SSRF, IDOR, broken authentication, broken access control, security misconfiguration
- Falle di autenticazione e autorizzazione (chi può fare cosa, e il codice lo verifica davvero?)
- Debolezze nelle API: endpoint esposti, assenza di rate limiting, validazione input insufficiente
- Esposizione di dati sensibili: segreti hardcoded, log verbosi, risposte che restituiscono più del necessario
- Rischi di configurazione e infrastruttura

**Per ogni problema trovato:**

| Campo | Contenuto |
|-------|-----------|
| **Posizione** | File e riga |
| **Gravità** | Critica / Alta / Media / Bassa |
| **Scenario di attacco** | Come un attaccante sfrutta la falla, in concreto |
| **Fix** | Codice sicuro, pronto da usare |

**Distingui sempre:**
- **Falla certa:** verificabile nel codice fornito, sfruttabile senza ipotesi aggiuntive
- **Sospetto da verificare:** richiede contesto che non hai (config di produzione, middleware upstream, comportamento runtime)

Regola: segnala solo problemi reali e verificabili. Non inventare vulnerabilità per riempire il report — un falso positivo è quasi dannoso quanto un falso negativo perché erode la fiducia nei report successivi. Se un'area è ragionevolmente sicura, dillo.
