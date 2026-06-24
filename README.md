# Enesi Claude Skills

Marketplace di **plugin per [Claude Code](https://claude.com/claude-code)** sviluppato da **Enesi srl**.

Questo repository raccoglie un insieme di plugin (skill e command) pensati per automatizzare attività ricorrenti del flusso di lavoro: analisi del codice, redazione di piani di sviluppo, lavorazione dei ticket Jira, audit SEO/GEO/AEO di siti web, audit di performance di siti Master Laravel Enesi e investigazione del codice dal punto di vista di un senior engineer. Tutti i plugin sono dichiarati in un unico marketplace e possono essere installati singolarmente.

---

## Indice

- [Cos'è un marketplace Claude Code](#cosè-un-marketplace-claude-code)
- [Installazione del marketplace](#installazione-del-marketplace)
- [Plugin disponibili](#plugin-disponibili)
  - [code-analysis](#code-analysis)
  - [dev-plan](#dev-plan)
  - [jira-worker](#jira-worker)
  - [seo-geo-aeo](#seo-geo-aeo)
  - [perf-audit](#perf-audit)
  - [senior-engineer](#senior-engineer)
- [Struttura del repository](#struttura-del-repository)
- [Changelog](#changelog)
- [Sviluppo e contributi](#sviluppo-e-contributi)

---

## Cos'è un marketplace Claude Code

Un *marketplace* è un repository che raccoglie più plugin Claude Code descritti nel file `.claude-plugin/marketplace.json`. Una volta aggiunto il marketplace, puoi installare i singoli plugin con `claude plugin install <nome>`.

Ogni plugin di questo repository è di uno di questi tipi:

- **Skill**: si attiva automaticamente in base a quello che chiedi a Claude in linguaggio naturale (nessun comando da digitare).
- **Command**: si invoca esplicitamente con uno slash command (es. `/jira-worker`).

---

## Installazione del marketplace

```bash
# 1. Aggiungi il marketplace
claude plugin marketplace add https://github.com/enesisrl/claude-skill

# 2. Installa i plugin che ti servono
claude plugin install code-analysis
claude plugin install dev-plan
claude plugin install jira-worker
claude plugin install seo-geo-aeo
claude plugin install perf-audit
claude plugin install senior-engineer
```

Per aggiornare un plugin già installato:

```bash
claude plugin update <nome-plugin>
```

---

## Plugin disponibili

| Plugin | Tipo | Versione | Descrizione breve |
|--------|------|----------|-------------------|
| [`code-analysis`](#code-analysis) | Skill | 1.0.0 | Report Markdown strutturati di analisi del codice |
| [`dev-plan`](#dev-plan) | Skill | 1.0.0 | Piani di sviluppo strutturati per feature, refactor, bug fix e migrazioni |
| [`jira-worker`](#jira-worker) | Command | 1.0.0 | Lavora i ticket di uno spazio Jira (In Corso → implementazione → commento → Testing) |
| [`seo-geo-aeo`](#seo-geo-aeo) | Command | 3.0.0 | `/seo-report`: report SEO / GEO / AEO orchestrato su `claude-seo`, in italiano con design system Enesi (PDF) |
| [`perf-audit`](#perf-audit) | Command | 1.0.0 | Audit di performance a imbuto per siti Master Laravel Enesi, con report prioritizzato |
| [`senior-engineer`](#senior-engineer) | Command | 1.0.0 | Cinque command di valutazione del codice dal punto di vista di un senior engineer |

---

### code-analysis

Skill che produce automaticamente **report di analisi del codice** in Markdown.

Quando chiedi a Claude di analizzare il codice di una cartella o di un file, la skill:

- salva il report in `.claude/analysis/<nome-descrittivo>.md` all'interno del progetto (default);
- segue una struttura coerente: Overview, Architecture, Key Components, Dependencies, Code Quality, Summary;
- rispetta un percorso di output personalizzato se lo indichi (es. *"salvalo in docs/review.md"*).

**Esempi d'uso** (nessun comando, la skill si attiva da sola):

> - "Analizza il codice in `src/`"
> - "Analyze the auth module and focus on security"
> - "Analyze `src/utils/` and save it to `docs/review.md`"

---

### dev-plan

Skill che redige **piani di sviluppo strutturati** in Markdown per feature, refactor, bug fix, migrazioni e nuovi progetti.

Quando chiedi un piano di sviluppo, la skill:

- salva il piano in `.claude/dev_plans/<nome-descrittivo>.md` (default);
- segue una struttura coerente: obiettivo e contesto, scope, requisiti, stato attuale, approccio proposto, piano implementativo (fasi e task), rischi, testing, domande aperte, riferimenti;
- pone domande di chiarimento mirate quando lo scope è ambiguo, prima di scrivere;
- rispetta un percorso di output personalizzato se lo indichi.

**Esempi d'uso:**

> - "Fammi un piano di sviluppo per la nuova area pagamenti"
> - "Draft an implementation plan for migrating to Postgres 15"
> - "Piano per rifare il modulo di reportistica, salvalo in `docs/reporting-plan.md`"

---

### jira-worker

Command `/jira-worker` che lavora **automaticamente i ticket di uno spazio Jira**.

Per ogni ticket selezionato:

1. lo sposta in stato **In Corso**;
2. lo implementa nel codice del progetto corrente con Claude Code;
3. aggiunge un **commento** al ticket con il riepilogo (file toccati, scelte fatte, note per il test);
4. se tutto va a buon fine lo sposta in **Testing**; in caso di problemi lo lascia In Corso con un commento esplicativo.

Al termine presenta un riepilogo complessivo. **Non esegue mai commit**: le modifiche restano nel working tree.

**Utilizzo:**

```
/jira-worker <spazio-jira> [note aggiuntive]
```

- `<spazio-jira>`: project key (es. `PROJ`) o nome del progetto Jira. Se omesso, viene chiesto.
- `[note aggiuntive]` (opzionali, chieste interattivamente se non fornite):
  - **Filtro ticket**: etichette, priorità, assegnatario o JQL custom;
  - **Vincoli tecnici**: istruzioni implementative valide per tutti i ticket.

Prima di iniziare mostra la lista dei ticket trovati e chiede quali lavorare; da lì procede in autonomia.

**Requisiti:**

- Connettore MCP **Atlassian** attivo nella sessione (claude.ai Atlassian), con permessi su Jira.
- Workflow Jira con stati raggiungibili equivalenti a "In Corso" e "Testing" (le transizioni sono cercate per nome in modo tollerante).

---

### seo-geo-aeo

Command `/seo-report` che produce un **report di audit di un sito** su tre dimensioni della visibilità di ricerca moderna, **in italiano con il design system Enesi** (PDF):

- **SEO** — ottimizzazione per i motori tradizionali (Google, Bing): title, meta description, struttura degli heading, schema markup, link interni, qualità dei contenuti, performance;
- **GEO** — *Generative Engine Optimization* per i motori AI (Perplexity, ChatGPT Search, Google AI Overviews, Gemini): segnali E-E-A-T, chiarezza delle entità, densità informativa, autorevolezza dell'autore;
- **AEO** — *Answer Engine Optimization* per featured snippet e ricerca vocale: FAQ schema, HowTo schema, heading formulati come domande, risposte dirette.

**Perché un command e non una skill.** Con `claude-seo` installato, le sue skill (`seo`, `seo-audit`, `seo-page`, …) si attivano sugli stessi trigger naturali (la parola "SEO", "audit", una URL): una skill auto-attivante nostra competerebbe con quelle, col rischio concreto di eseguire la skill sbagliata. Per questo il plugin espone uno **slash command esplicito** (`/seo-report`), che delega in modo altrettanto esplicito alla skill `claude-seo:seo-audit`.

**Architettura — orchestratore + report Enesi.** Il command **non riesegue** l'analisi: delega l'audit al plugin **[`claude-seo`](https://github.com/AgriciDaniel/claude-seo)** (di AgriciDaniel), ne ingerisce l'envelope completo (`audit-data.json`, findings, action plan) e lo mappa nel **report Enesi**, renderizzato in PDF. `claude-seo` fa l'analisi completa (technical SEO, content/E-E-A-T, schema, Core Web Vitals, AI-search/GEO, hreflang, local, SXO, immagini, health score /100 e action plan); il command aggiunge i tre assi SEO/GEO/AEO, il design system Enesi, la lingua italiana e il PDF da consegnare al cliente.

Il report finale è un PDF A4 (fino a 14 pagine: cover a 3 assi + health score pesato /100, pagine analizzate, analisi per dimensione, raccomandazioni prioritarie, criticità per severità — paginate, snippet schema pronti, piano d'azione a sprint, punti di forza, SXO opzionale, glossario), generato a partire da `assets/report-template.html` — la versione autonoma del template `templates/report-seo/ReportSeo.dc.html` del progetto **Enesi Design System** su claude.ai/design.

**Requisiti:**

- Plugin **`claude-seo`** installato (motore di analisi):
  ```bash
  claude plugin marketplace add https://github.com/AgriciDaniel/claude-seo
  claude plugin install claude-seo
  ```
- **WeasyPrint** per la conversione HTML→PDF (già installato da `claude-seo`; in alternativa `pip install weasyprint`).
- *(Opzionale, consigliato)* credenziali Google/API in `claude-seo` per Core Web Vitals **di campo** (via CrUX/PageSpeed — API, niente browser), backlink e GSC/GA4. Prima di ogni audit il command **verifica le credenziali e avvisa** se mancano (performance euristiche vs reali).

> **Generazione PDF senza browser.** Il command usa **WeasyPrint** (motore HTML→PDF senza browser) perché le regole globali Enesi vietano l'avvio di un browser in modalità headless. Chrome `--headless --print-to-pdf` resta solo come fallback, da autorizzare esplicitamente.

**Utilizzo:**

```
/seo-report <url> [quick|full]
```

- `<url>`: il sito da auditare (se omesso, viene chiesto).
- `quick` | `full` (facoltativo): profondità dell'audit; se omesso, il command chiede.

Esempi: `/seo-report https://www.esempio.it full` · `/seo-report esempio.it`

---

### perf-audit

Command `/perf-audit` che esegue un **audit di performance a imbuto** (misura → isola → scava) su siti **Master Laravel Enesi** (Laravel 12, UUID, tabelle di traduzione, URL SEO `rewurl_*`, codice in `private/`).

Localizza prima *dove* sta il tempo (server vs DB vs rete vs frontend) con misure black-box, poi scende nel dettaglio solo sul layer colpevole, confrontando ogni numero con soglie di riferimento. Copre 8 fasi:

1. **Setup/detect** e scelta delle URL di prova reali (rotte SEO);
2. **Baseline TTFB** (cold/warm, DNS/connect/TLS separati);
3. **Carico/concorrenza** con `hey`/`ab`/`wrk` (percentili p50…p99) — *solo se autorizzato*;
4. **Profiling server-side**: query count & N+1 (Debugbar, **Telescope**, Clockwork);
5. **Database**: top query per latenza, `EXPLAIN`, indici mancanti su FK UUID e `rewurl`;
6. **Cache & config infra**: OPcache, cache buildate, driver cache/sessione, Redis, Meilisearch;
7. **Chiamate esterne sincrone** nel render (Stripe/shipping/social);
8. **Frontend/delivery** (Lighthouse) e **risorse server**.

Adotta un **doppio passaggio** — Pass A diagnostico (`APP_DEBUG=true`, Debugbar/Telescope) per la struttura, Pass B realistico (`APP_DEBUG=false`, OPcache, cache buildate, Redis) per il numero che il cliente sente — e sa attingere anche a **Laravel Pulse** e ad APM (Sentry, New Relic, Datadog) se presenti. Gli audit ampi vengono delegati a subagenti read-only paralleli.

**Utilizzo:**

```
/perf-audit [url-prod] [url-staging-o-locale]
```

- Senza argomenti usa il locale `http://127.0.0.1:8000` e avvisa che la diagnosi è parziale.
- Con più URL, il **primo** è il bersaglio principale (di solito produzione), gli altri per confronto.

Produce un report `private/storage/perf-audit/report-<TS>.md` con finding prioritizzati nel formato `Problema | Evidenza | Impatto | Fix | Sforzo`, separando **quick-win** (≤1h) e **interventi strutturali**.

**Requisiti:** `curl`, `ab`, `wrk` nativi; `hey` e `lighthouse` via Docker; `jq`. I load test contro produzione vanno eseguiti **solo con autorizzazione esplicita** e su pagine in lettura.

---

### senior-engineer

Cinque command che simulano il punto di vista di un ruolo senior specifico. Tutti pensati per essere usati **prima di toccare il codice**: capire, valutare e decidere prima di scrivere o cambiare.

| Command | Ruolo simulato | Quando usarlo |
|---------|---------------|---------------|
| `/senior-engineer:audit` | Senior engineer appena arrivato | Primo approccio a un codebase, identifica problemi architetturali prioritizzati per gravità |
| `/senior-engineer:debug` | Senior engineer in produzione | Bug da investigare metodicamente: 4 fasi, nessun fix senza root cause |
| `/senior-engineer:refactor` | Senior software architect | Refactoring clean architecture: proposta → conferma → esecuzione, un cambiamento alla volta |
| `/senior-engineer:security` | Senior security engineer | Audit OWASP + baseline igiene: ogni finding con scenario di attacco e fix pronto |
| `/senior-engineer:techlead` | Senior technical lead | Valutazione decisioni tecniche con tradeoff espliciti, prima di scrivere codice |

I comandi si integrano tra loro: `/audit` alimenta `/refactor`, `/techlead` porta a `/dev-plan`, `/debug` usa la metodologia delle 4 fasi di `systematic-debugging`.

**Installazione:**

```bash
claude plugin install senior-engineer
```

**Utilizzo:**

```
/senior-engineer:audit src/
/senior-engineer:debug "ordini duplicati in produzione da ieri mattina"
/senior-engineer:refactor app/Http/Controllers/
/senior-engineer:security app/Http/Controllers/AuthController.php
/senior-engineer:techlead "usare Redis per le sessioni invece del DB"
```

---

## Struttura del repository

```
claude-skills/
├── .claude-plugin/
│   └── marketplace.json          ← dichiarazione del marketplace e dei plugin
├── code-analysis-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── skills/code-analysis/SKILL.md
│   └── README.md
├── dev-plan-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── skills/dev-plan/SKILL.md
│   └── README.md
├── jira-worker-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── commands/jira-worker.md
│   └── README.md
├── seo-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── assets/report-template.html   ← template report Enesi (design)
│   ├── commands/seo-report.md        ← command /seo-report
│   ├── README.md
│   └── .gitignore
├── perf-audit-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── commands/perf-audit.md
│   └── README.md
├── senior-engineer-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── commands/
│   │   ├── audit.md
│   │   ├── debug.md
│   │   ├── refactor.md
│   │   ├── security.md
│   │   └── techlead.md
│   └── README.md
└── README.md                     ← questo file
```

Ogni plugin ha:

- un `plugin.json` con nome, versione, descrizione e autore;
- una `SKILL.md` (per le skill) o un file in `commands/` (per i command) che contiene le istruzioni operative — la *source of truth* del comportamento;
- un `README.md` dedicato con installazione e dettagli specifici.

---

## Changelog

Aggiornamenti notevoli dei plugin. La versione corrente di ogni plugin è nella colonna *Versione* della tabella [Plugin disponibili](#plugin-disponibili).

### code-analysis

- **1.0.0** — Release iniziale: skill di analisi del codice che produce report Markdown strutturati (Overview, Architecture, Key Components, Dependencies, Code Quality, Summary), salvati per default sotto `.claude/analysis/`.

### dev-plan

- **1.0.0** — Release iniziale: skill che redige piani di sviluppo strutturati (obiettivo, scope, requisiti, stato attuale, approccio, fasi e task, rischi, testing, domande aperte), salvati per default sotto `.claude/dev_plans/`.

### jira-worker

- **1.0.0** — Release iniziale: command `/jira-worker` che lavora i ticket di uno spazio Jira (In Corso → implementazione → commento → Testing), senza eseguire commit. Richiede il connettore MCP Atlassian.

### seo-geo-aeo

- **3.0.0** — Da skill auto-attivante a **slash command esplicito `/seo-report`**, per evitare conflitti di routing con le skill di `claude-seo`. Ingestione dell'envelope completo di `claude-seo` (severità complete, SXO, business intelligence), campionamento della lingua del cliente, verifica HTTP degli header reali, **controllo credenziali Google/backlink con avviso** all'utente. Template Enesi con pagina SXO opzionale, paginazione robusta (pagine ad altezza fissa, numero pagina nell'header, footer con indirizzo legale solo in cover) e criticità paginate.
- **2.0.0** — Riscrittura come **orchestratore su `claude-seo`** con report PDF in italiano e design system Enesi (3 assi SEO/GEO/AEO + health score pesato /100). Output solo PDF via WeasyPrint (senza browser headless); rimossa la skill standalone e l'integrazione PageSpeed locale.
- **1.x** — Skill standalone precedente (scoring deterministico interno, output Word/PDF generico).

### perf-audit

- **1.0.0** — Release iniziale: command `/perf-audit` per l'audit di performance a imbuto (misura → isola → scava) su siti Master Laravel Enesi (TTFB, carico/concorrenza, profiling N+1, database, cache, chiamate esterne, frontend), con report finale prioritizzato.

### senior-engineer

- **1.0.0** — Release iniziale: cinque command (`audit`, `debug`, `refactor`, `security`, `techlead`) che simulano i ruoli di un senior engineer, pensati per capire e valutare il codice prima di modificarlo.

---

## Sviluppo e contributi

- **Aggiungere un plugin**: crea una nuova cartella `<nome>-plugin/` con il suo `.claude-plugin/plugin.json` e la skill/command, poi aggiungi la voce corrispondente in `.claude-plugin/marketplace.json`.
- **Versionamento**: aggiorna il campo `version` nel `plugin.json` del plugin modificato.
- **Autore / contatti**: Emanuele Toffolon — `emanuele.toffolon@enesi.it`.

---

*Plugin Claude Code — Enesi srl.*
