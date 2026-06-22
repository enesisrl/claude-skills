---
description: Refactoring verso un'architettura pulita, senza cambiare il comportamento
argument-hint: [file o cartella da rifattorizzare]
---

Comportati come un senior software architect che rifattorizza questo codice con principi di clean architecture. Target: $ARGUMENTS

**Prima di iniziare:** se esiste un report in `.claude/analysis/` o un output di `/audit` su questo target, usalo come base. Se no, leggi il codice per intero prima di proporre qualsiasi struttura — non ridisegnare ciò che non hai capito.

**Vincolo assoluto: NON cambiare il comportamento del prodotto. Solo struttura e qualità del codice.**

Obiettivi:
- Separare le responsabilità
- Aumentare la modularità
- Ridurre l'accoppiamento
- Migliorare la manutenibilità a lungo termine

**Procedi in due tempi — proposta prima, esecuzione poi:**

**Tempo 1 — Proposta (non toccare ancora il codice):**
- Struttura delle cartelle/file proposta con motivazione
- Spiegazione delle scelte architetturali: cosa cambia e perché
- Ordine degli interventi: prima i cambiamenti a maggior impatto e minor rischio
- Cosa NON toccherai e perché

**Tempo 2 — Esecuzione (solo dopo conferma esplicita):**
- Un cambiamento alla volta — mai modifiche parallele sullo stesso codice
- Se un intervento rischia di alterare il comportamento, segnalalo e fermati
- Se durante il refactoring la portata si rivela molto più ampia del previsto, segnalalo prima di continuare e valuta se pianificare il resto con `/dev-plan`

**Output finale:**
- Struttura risultante
- Sommario delle scelte architetturali applicate
- Cosa NON hai toccato e perché
