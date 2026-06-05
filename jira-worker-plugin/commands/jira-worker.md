---
description: "Lavora i ticket di uno spazio Jira: To Do → In Corso → implementazione → commento → Testing"
argument-hint: <spazio-jira> [note aggiuntive]
---

# Jira Worker — lavorazione automatica dei ticket

Lavora i ticket di uno spazio (progetto) Jira nel codice del progetto corrente, seguendo questo ciclo di vita per ogni ticket: **Da fare → In Corso → implementazione → commento sul ticket → Testing**.

Argomenti ricevuti: `$ARGUMENTS`

## Fase 0 — Raccolta parametri

1. **Spazio Jira** (obbligatorio): il primo argomento è il riferimento allo spazio/progetto Jira (project key, es. `PROJ`, oppure nome del progetto). Se non è stato fornito, chiedilo all'utente con AskUserQuestion prima di procedere (se possibile, recupera la lista dei progetti visibili con `getVisibleJiraProjects` e proponila come opzioni).
2. **Note aggiuntive** (opzionali): tutto ciò che segue il primo argomento. Se non sono state fornite, chiedi all'utente con AskUserQuestion (una sola chiamata, due domande, entrambe facoltative — l'utente può rispondere "nessuno"):
   - **Filtro ticket**: criteri aggiuntivi per selezionare i ticket dello spazio — etichette, priorità, assegnatario, tipo di issue, oppure un frammento JQL custom. Default: nessun filtro.
   - **Vincoli tecnici**: istruzioni implementative da rispettare per tutti i ticket — convenzioni di codice, librerie da usare o evitare, parti del codice da non toccare. Default: nessun vincolo.

Tieni i vincoli tecnici sempre presenti durante tutta la lavorazione: valgono per ogni ticket.

## Fase 1 — Ricerca dei ticket

1. Recupera il `cloudId` con `getAccessibleAtlassianResources` (se ce n'è più di uno, chiedi all'utente quale usare).
2. Cerca i ticket da lavorare con `searchJiraIssuesUsingJql`:
   - JQL base: `project = "<KEY>" AND statusCategory = "To Do" ORDER BY priority DESC, created ASC`
   - Se l'utente ha fornito un filtro, integralo nella JQL (etichette → `labels IN (...)`, priorità → `priority = ...`, assegnatario → `assignee = ...`, JQL custom → aggiungilo in AND).
   - Richiedi i campi: `summary`, `status`, `priority`, `issuetype`, `assignee`, `labels`, `description`.
3. Se la ricerca non restituisce ticket, comunicalo e termina.

## Fase 2 — Conferma della lista

Mostra all'utente la lista dei ticket trovati in una tabella (chiave, titolo, tipo, priorità, assegnatario) e **chiedi conferma di quali lavorare** con AskUserQuestion (multiSelect, un'opzione per ticket; se sono più di 4, mostra la tabella nel messaggio e chiedi all'utente di indicare le chiavi da lavorare o "tutti").

Questa è l'**unica conferma richiesta**: dopo questo punto procedi in completa autonomia su tutti i ticket selezionati, senza ulteriori interruzioni, fino al riepilogo finale.

## Fase 3 — Lavorazione di ogni ticket (in sequenza, in autonomia)

Per ogni ticket selezionato, nell'ordine della lista:

### 3.1 Presa in carico
1. Leggi il ticket completo con `getJiraIssue` (descrizione, commenti esistenti, allegati referenziati).
2. Recupera le transizioni disponibili con `getTransitionsForJiraIssue` e individua quella verso lo stato di lavorazione cercando, in ordine: nome esattamente `In Corso`, poi `In Progress`, poi qualsiasi transizione verso uno stato di categoria `In Progress`. Esegui la transizione con `transitionJiraIssue`.
3. Se la transizione non esiste o fallisce, NON bloccare tutto: salta il ticket, annota il problema per il riepilogo finale e passa al successivo.

### 3.2 Implementazione
1. Analizza la richiesta del ticket e individua nel progetto corrente i file coinvolti.
2. Implementa la modifica richiesta rispettando i **vincoli tecnici** forniti in Fase 0 e le convenzioni del progetto (CLAUDE.md, skill di progetto, stile del codice esistente).
3. Verifica il lavoro: se il progetto ha test, lint o build pertinenti alle modifiche, eseguili. Non dichiarare completato un ticket senza aver verificato che il codice sia almeno sintatticamente valido e coerente.
4. **Non eseguire commit né altri comandi git che modificano lo stato del repository**: le modifiche restano nel working tree, i commit li farà l'utente manualmente.
5. Se la descrizione del ticket è troppo vaga o ambigua per essere implementata con ragionevole certezza, NON inventare: tratta il ticket come "non lavorabile" (vedi 3.4).

### 3.3 Esito positivo
1. Aggiungi un commento al ticket con `addCommentToJiraIssue`, in italiano, con:
   - riepilogo di cosa è stato implementato e delle scelte fatte;
   - elenco dei file modificati/creati;
   - eventuali verifiche eseguite (test, lint) e relativo esito;
   - note utili per chi testerà (come provare la modifica, casi limite, eventuali punti di attenzione).
2. Recupera di nuovo le transizioni con `getTransitionsForJiraIssue` e individua quella verso il testing cercando, in ordine: nome esattamente `Testing`, poi `Test`, `In Test`, `QA`, poi qualsiasi stato il cui nome contenga "test". Esegui la transizione.
3. Se la transizione verso Testing non esiste, lascia il ticket In Corso e annotalo nel riepilogo finale.

### 3.4 Esito negativo o ticket non lavorabile
1. Aggiungi un commento al ticket che spiega in modo chiaro: cosa è stato tentato, cosa è andato storto o quali informazioni mancano, e cosa serve per procedere.
2. **Lascia il ticket nello stato In Corso** (non portarlo in Testing).
3. Se l'implementazione era partita e ha lasciato modifiche incomplete o incoerenti nel working tree, ripristina i file toccati per quel ticket allo stato precedente, così da non sporcare la lavorazione dei ticket successivi.
4. Prosegui con il ticket successivo: un fallimento non interrompe il giro.

## Fase 4 — Riepilogo finale

Al termine, presenta all'utente una tabella riassuntiva con, per ogni ticket:

| Ticket | Titolo | Esito | Stato finale | File toccati | Note |
|---|---|---|---|---|---|

E in coda:
- promemoria che **nessun commit è stato eseguito**, con l'elenco complessivo dei file modificati nel working tree (es. via `git status --short`);
- eventuali ticket saltati o rimasti In Corso, con il motivo;
- suggerimenti su come procedere (review delle modifiche, commit, test manuali).

## Regole generali

- Tutti i commenti sui ticket Jira vanno scritti in italiano, con tono professionale e sintetico.
- Non modificare mai ticket fuori dallo spazio indicato.
- Non cambiare assegnatario, priorità o altri campi dei ticket: solo stato e commenti.
- In caso di errore degli strumenti Atlassian (rete, permessi), riprova una volta; se fallisce ancora, annota e prosegui.
