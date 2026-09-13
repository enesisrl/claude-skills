# senior-engineer — Plugin Claude Code

Tre comandi di valutazione e indagine del codice, ciascuno che simula il punto di vista di un
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

### `/refactor [file o cartella]`
Refactoring verso clean architecture come un senior software architect. Procede in due tempi:
prima propone la struttura (senza toccare il codice), poi esegue solo dopo conferma. Un
cambiamento alla volta, con escalation se la portata si rivela più ampia del previsto.

```
/senior-engineer:refactor app/Http/Controllers/
/senior-engineer:refactor src/legacy/
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

## Cosa usare per debug e sicurezza

Dalla **2.0.0** questo plugin non include più `/debug` e `/security`: erano duplicati di strumenti
già disponibili, e tenerli significava mantenere due volte la stessa metodologia.

| Ti serve | Usa |
|----------|-----|
| Investigare un bug risalendo alla root cause | La skill **`systematic-debugging`**, che si attiva **da sola** su qualsiasi bug, test rotto o comportamento inatteso — stessa legge ferrea (nessun fix senza root cause), stesse quattro fasi, stessi red flag. Nessun comando da digitare. |
| Audit di sicurezza sul **codice** | `/security-review` (nativo di Claude Code) |
| Audit di sicurezza **completo** (codice + sito live + segreti + dipendenze) | `/security-report` — orchestra SAST, DAST, segreti, dipendenze e container in un report unico |
| Audit di sicurezza sul **sito live** (DAST) | `/vuln-audit` |

---

## Flussi di lavoro consigliati

**Audit → Refactoring pianificato**
```
/senior-engineer:audit src/              → identifica le aree critiche
/senior-engineer:refactor src/Services/  → propone la nuova struttura
/dev-plan refactor del layer Services    → pianifica a fasi rilasciabili
```

**Decisione → Piano**
```
/senior-engineer:techlead "introdurre la queue per le email"
/dev-plan implementazione queue email
```

---

## Struttura

```
senior-engineer-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── audit.md
│   ├── refactor.md
│   └── techlead.md
└── README.md
```

## Installazione (da marketplace)

```
/plugin install senior-engineer@<nome-marketplace>
```
