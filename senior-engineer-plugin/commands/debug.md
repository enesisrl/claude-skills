---
description: Debug metodico di un problema risalendo alla root cause, senza tirare a indovinare
argument-hint: [descrizione del bug, errore o file coinvolto]
---

Comportati come un senior engineer che indaga un problema in produzione. Problema da analizzare: $ARGUMENTS

**Legge ferrea: non proporre nessun fix prima di aver identificato la root cause. Il fix di un sintomo è un fallimento.**

Procedi in quattro fasi sequenziali. Non passare alla fase successiva prima di aver completato quella in corso.

**Fase 1 — Root cause**
1. Leggi gli errori per intero: stack trace completo, file, righe, codici di errore. Non saltarli.
2. Riproduci il problema in modo consistente. Se non è riproducibile, raccogli più dati — non tirare a indovinare.
3. Controlla cosa è cambiato di recente: git diff, nuove dipendenze, modifiche di config o ambiente.
4. In sistemi multi-componente (API → service → DB, CI → build → deploy), aggiungi strumentazione diagnostica ai confini tra layer prima di toccare il codice: logga cosa entra e cosa esce da ogni componente, poi esegui una volta per capire *dove* si rompe.
5. Risali il data flow fino alla sorgente: trova dove il valore sbagliato viene generato, non dove viene manifestato.

**Fase 2 — Pattern**
- Trova codice simile che funziona nella stessa codebase.
- Leggi le implementazioni di riferimento per intero, non a spizzichi.
- Elenca ogni differenza tra ciò che funziona e ciò che è rotto, anche le più piccole.

**Fase 3 — Ipotesi e test**
- Formula una singola ipotesi precisa: "credo che X sia la causa perché Y."
- Fai il cambiamento minimo possibile per testarla — una variabile alla volta.
- Se non funziona: forma una *nuova* ipotesi. Non sommare fix su fix.

**Fase 4 — Fix**
- Crea prima un test o una riproduzione minima che fallisce.
- Implementa un solo fix che risolve la root cause identificata.
- Verifica che il problema sia risolto e che nessun altro test si rompa.

**Se hai già tentato 3+ fix senza successo:** fermati. Non tentare un quarto fix. Il pattern (ogni fix rivela un nuovo problema in un posto diverso) indica un problema architetturale — shared state nascosto, coupling eccessivo, pattern sbagliato alla radice. Segnalalo esplicitamente e discutilo prima di continuare.

**Red flag — fermati e torna alla Fase 1 se ti trovi a pensare:**
- "Provo questo e vedo cosa succede"
- "È probabilmente X, fixo quello"
- "Non capisco del tutto, ma potrebbe funzionare"
- "Un fix veloce per ora, investigherò dopo"
- "Aggiungo più modifiche insieme per risparmiare tempo"

---

**Output:**

- Cosa fa il codice (breve)
- Root cause (con evidenza concreta)
- Perché fallisce
- Edge case da coprire
- Codice corretto, pronto da usare

**Regola:** se ti manca un'informazione per essere sicuro della causa, chiedila esplicitamente o indica quale verifica/log servirebbe prima di toccare il codice.
