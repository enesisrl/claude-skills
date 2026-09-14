# dev-pipeline — Plugin Claude Code

Command `/pipeline`: lavora **un task di sviluppo** end-to-end in cinque fasi, con tre subagenti a
contesto isolato. Ogni fase è configurabile: puoi scegliere il modello Claude oppure delegarla a un
agente esterno (Codex, Gemini, un altro CLI).

```
/pipeline <requisito del task> [--task-id <slug>] [--resume]
```

## Le cinque fasi

| # | Fase | Dove gira | Produce |
|---|------|-----------|---------|
| 1 | **CHIARISCI** | nel main loop, con te | `REQUIREMENT.md` + `.ai/DECISIONS.md` |
| 2 | **PIANIFICA** | agente `planner` | `PLAN.md` |
| 3 | **IMPLEMENTA** | agente `implementer` (`implement`) | codice + `IMPLEMENTATION.md` |
| 4 | **REVISIONA** | agente `reviewer` | `CODE_REVIEW.md` |
| 5 | **CORREGGI** | agente `implementer` (`fix`) | `FIX_REPORT.<N>.md` |

La fase 1 è l'unica che parla con te, ed è il motivo per cui la pipeline esiste in questa forma: un
fraintendimento chiarito all'inizio costa una domanda, lo stesso fraintendimento scoperto al test
finale costa il giro completo — piano, implementazione, review, fix — più il tempo per capire
perché un risultato ben argomentato è sbagliato.

## Configurare ogni fase

Copia `pipeline.default.json` in `.ai/pipeline.json` nel progetto e adattalo. Ogni fase vuole una
stringa:

```json
{
  "phases": {
    "plan":      "codex",
    "implement": "claude:sonnet",
    "review":    "codex",
    "fix":       "claude:sonnet"
  }
}
```

- `claude:opus` · `claude:sonnet` · `claude:haiku` → subagente Claude su quel modello
- un nome definito in `providers` → CLI esterno

I provider si dichiarano così:

```json
{
  "providers": {
    "codex":  { "cmd": "codex exec -s read-only", "prompt": "stdin", "smoke": "codex --version" },
    "gemini": { "cmd": "gemini -p",               "prompt": "stdin", "smoke": "gemini --version" }
  }
}
```

`smoke` è il controllo di disponibilità. **Essere sul `PATH` non basta**: un CLI che richiede
un'approvazione interattiva o un flag bloccato dai permessi è installato e inutilizzabile. Se lo
smoke test fallisce, la fase ripiega sul modello Claude di default e viene marcata `DEGRADED`.

### Tre livelli di indipendenza

Il valore della fase 4 sta nell'essere **indipendente** da chi ha scritto il codice. Non tutte le
configurazioni danno la stessa indipendenza, e il report dice sempre quale hai ottenuto:

| Livello | Configurazione | Cosa ti dà |
|---|---|---|
| **A** | provider esterno (modello diverso, processo diverso) | punti ciechi diversi da chi ha implementato |
| **B** | `implement: claude:sonnet` + `review: claude:opus` | diversità parziale |
| **C** | stesso modello per implement e review | nessuna diversità di modello — restano contesto fresco, attenzione al solo diff e ruolo avversariale |

Il livello C vale meno di A, non zero. Ma se leggi `DEGRADED` su una review di codice di
autenticazione o migrazioni, sappi che **non** hai avuto una verifica indipendente.

## Perché consuma poco

Il contesto principale è la risorsa cara: quello che leggi lì viene rispedito a ogni turno
successivo della sessione. Quindi:

- **`/pipeline` non apre mai un artefatto.** Ogni agente chiude con un riassunto a forma fissa di
  poche righe, ed è l'unica cosa che entra nel contesto principale.
- Gli artefatti passano da agente ad agente **per path**: li legge il destinatario, nel proprio
  contesto usa-e-getta.
- Il revisore riceve un **diff precalcolato**, mai il repository.
- Il secondo giro di review vede solo il diff del fix e i finding precedenti.
- I ruoli dichiarano un **tetto di righe**: un piano che diventa un inventario del repository costa
  a valle più di quanto faccia risparmiare.

Misura di partenza, dai 10 task storici che hanno ispirato il plugin: **~254.000 token** di soli
artefatti, di cui i tre file più pesanti erano piani preliminari poi scartati per due terzi. Questa
pipeline non ha la fase di piano preliminare.

## Artefatti

Sotto `.ai/tasks/<TASK_ID>/`: `REQUIREMENT.md`, `PLAN.md`, `IMPLEMENTATION.md`, `CODE_REVIEW.md`,
`FIX_REPORT.<N>.md`, `STATUS.json`.

`.ai/DECISIONS.md` sta invece **a livello di progetto** e cresce in append: è la memoria che
permette al task 22 di sapere cosa è stato deciso al task 7.

Consiglio: metti in `.gitignore` gli artefatti di processo e tieni tracciati solo `REQUIREMENT.md` e
`DECISIONS.md` — sono gli unici due che servono a qualcuno fra sei mesi.

## Requisiti

Nessuno obbligatorio: senza provider esterni gira tutto su subagenti Claude. `git` serve per il
diff (senza, la fase 4 lavora sull'intero stato ed è meno efficace e più cara).

## Limiti dichiarati

- **Non committa e non pusha.** Le modifiche restano nel working tree.
- **Lavora un task per volta**, non progetti interi. Un progetto è trenta task, e tenerli coerenti
  fra loro è un problema diverso: `.ai/DECISIONS.md` è il seme di quella coerenza, non la soluzione.
- **Non elimina il test umano finale.** Sposta l'intervento umano all'inizio, dove costa poco.
- **Massimo 2 giri di fix**, poi si ferma. Non è un limite di budget: se dopo due giri emergono
  ancora finding nuovi, la pipeline non sta convergendo e il problema è nel piano o nel requisito.

## Rapporto con `dev-plan`

`/dev-plan` produce un **documento per un umano**: un piano di sviluppo da leggere, discutere e
approvare. `/pipeline` produce un **piano per la macchina successiva** della catena, che non verrà
letto da nessuno se non dall'implementer. Se vuoi decidere *se* e *come* affrontare un lavoro, usa
`/dev-plan`. Se hai già deciso e vuoi che venga fatto, usa `/pipeline`.

## Installazione

```
claude plugin install dev-pipeline
```
