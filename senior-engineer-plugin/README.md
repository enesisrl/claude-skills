# senior-engineer — Plugin Claude Code

Cinque comandi di valutazione e indagine del codice, ciascuno che simula il punto di vista di un
ruolo senior specifico. Tutti pensati per essere usati **prima di toccare il codice**: capire,
valutare e decidere prima di scrivere o cambiare.

## Comandi

### `/audit [file o cartella]`
Audit architetturale come un senior engineer appena arrivato sul progetto. Ricostruisce
l'architettura reale dal codice (non da ciò che dovrebbe fare), identifica i problemi concreti
con gravità e priorità, propone strategie di refactoring. Usa `.claude/analysis/` come punto
di partenza se esiste.

```
/senior-engineer:audit src/
/senior-engineer:audit app/Services/OrderService.php
```

---

### `/debug [descrizione del bug, errore o file coinvolto]`
Debug metodico come un senior engineer in produzione. Segue quattro fasi sequenziali (root cause
→ pattern → ipotesi → fix) con la legge ferrea: nessun fix prima di aver identificato la causa.
Include red flag espliciti e la regola dei 3+ fix falliti come segnale di problema architetturale.

```
/senior-engineer:debug "ordini duplicati in produzione da ieri mattina"
/senior-engineer:debug app/Jobs/ProcessPayment.php
```

---

### `/refactor [file o cartella]`
Refactoring verso clean architecture come un senior software architect. Procede in due tempi:
prima propone la struttura (senza toccare il codice), poi esegue solo dopo conferma. Un
cambiamento alla volta, con escalation se la portata si rivela più ampia del previsto.

```
/senior-engineer:refactor app/Http/Controllers/
/senior-engineer:refactor src/legacy/
```

---

### `/security [file, endpoint o area]`
Audit di sicurezza come un senior security engineer. Parte da una baseline di igiene fondamentale
(SSL, CSRF, hashing, segreti, errori), poi scende sulle vulnerabilità OWASP. Ogni finding include
posizione, gravità, scenario di attacco concreto e fix pronto da usare. Distingue falle certe da
sospetti da verificare.

```
/senior-engineer:security app/Http/Controllers/AuthController.php
/senior-engineer:security "endpoint /api/orders"
```

---

### `/techlead [decisione tecnica o feature]`
Valutazione da tech lead prima di scrivere codice. Chiede le domande di chiarimento che mancano,
contesta le decisioni deboli, applica YAGNI, analizza i tradeoff per ogni opzione (pro / contro /
rischi a lungo termine / quando ha senso). Suggerisce `/dev-plan` come passo successivo naturale
dopo l'allineamento sulla direzione.

```
/senior-engineer:techlead "usare Redis per le sessioni invece del DB"
/senior-engineer:techlead "introdurre un event bus per disaccoppiare i moduli"
```

---

## Flussi di lavoro consigliati

**Audit → Refactoring pianificato**
```
/senior-engineer:audit src/        → identifica le aree critiche
/senior-engineer:refactor src/Services/  → propone la nuova struttura
/dev-plan refactor del layer Services    → pianifica a fasi rilasciabili
```

**Decisione → Piano**
```
/senior-engineer:techlead "introdurre la queue per le email"
/dev-plan implementazione queue email
```

**Bug in produzione**
```
/senior-engineer:debug "timeout sulle pagine prodotto dopo il deploy delle 14"
```

**Prima di un rilascio**
```
/senior-engineer:security app/Http/Controllers/
/senior-engineer:security config/
```

---

## Struttura

```
senior-engineer-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── audit.md
│   ├── debug.md
│   ├── refactor.md
│   ├── security.md
│   └── techlead.md
└── README.md
```

## Installazione (da marketplace)

```
/plugin install senior-engineer@<nome-marketplace>
```
