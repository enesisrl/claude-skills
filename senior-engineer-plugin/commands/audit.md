---
description: Audit del codice come un senior engineer appena arrivato sul progetto
argument-hint: [file o cartella da analizzare]
---

Comportati come un senior engineer appena entrato in questo codebase, che non conosce. Analizza: $ARGUMENTS

**Prima di iniziare:** controlla se esiste già un report in `.claude/analysis/` relativo a questo target. Se esiste, usalo come punto di partenza e concentrati sulla valutazione critica invece di ridescrivere ciò che è già documentato.

Procedi così:

1. Ricostruisci l'architettura reale e il data flow, partendo dal codice (non da quello che dovrebbe fare, da quello che fa davvero).
2. Identifica i problemi concreti: scelte architetturali discutibili, logica duplicata, colli di bottiglia di performance, rischi di scalabilità, punti difficili da mantenere.
3. Per ogni problema indica gravità e perché conta.

**Distingui sempre fatti da ipotesi.** Se un giudizio richiede contesto che non hai (requisiti di business, vincoli infrastrutturali, storia del progetto), dillo invece di assumere.

Poi proponi:

- Una sintesi chiara dell'architettura attuale
- Le aree critiche in ordine di priorità, in formato tabella:

  | Area | Problema | Gravità (alta/media/bassa) | Perché conta |
  |------|----------|---------------------------|--------------|

- Strategie di refactoring concrete per le aree ad alta priorità
- Esempi di codice migliorato dove il beneficio è immediato e verificabile

Vincolo: non cambiare il comportamento del prodotto. Migliora solo qualità, scalabilità e manutenibilità.

**Passo successivo:** per le aree critiche identificate, usa `/refactor` per eseguire il refactoring su un'area specifica, o `/dev-plan` per pianificare l'intervento a fasi rilasciabili.
