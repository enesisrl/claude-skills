# Enesi Claude Skills

Marketplace di **plugin per [Claude Code](https://claude.com/claude-code)** sviluppato da **Enesi srl**.

Questo repository raccoglie un insieme di plugin (skill e command) pensati per automatizzare attività ricorrenti del flusso di lavoro: analisi del codice, redazione di piani di sviluppo, lavorazione dei ticket Jira, audit SEO/GEO/AEO di siti web e audit di performance di siti Master Laravel Enesi. Tutti i plugin sono dichiarati in un unico marketplace e possono essere installati singolarmente.

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
- [Struttura del repository](#struttura-del-repository)
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
claude plugin marketplace add github:enesisrl/claude-skills

# 2. Installa i plugin che ti servono
claude plugin install code-analysis
claude plugin install dev-plan
claude plugin install jira-worker
claude plugin install seo-geo-aeo
claude plugin install perf-audit
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
| [`seo-geo-aeo`](#seo-geo-aeo) | Skill | 1.1.0 | Audit SEO / GEO / AEO di un sito con punteggio deterministico e report Word/PDF |
| [`perf-audit`](#perf-audit) | Command | 1.0.0 | Audit di performance a imbuto per siti Master Laravel Enesi, con report prioritizzato |

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

Skill di **audit di un sito web** su tre dimensioni della visibilità di ricerca moderna:

- **SEO** — ottimizzazione per i motori tradizionali (Google, Bing): title, meta description, struttura degli heading, schema markup, link interni, qualità dei contenuti;
- **GEO** — *Generative Engine Optimization* per i motori AI (Perplexity, ChatGPT Search, Google AI Overviews, Gemini): segnali E-E-A-T, chiarezza delle entità, densità informativa, autorevolezza dell'autore;
- **AEO** — *Answer Engine Optimization* per featured snippet e ricerca vocale: FAQ schema, HowTo schema, heading formulati come domande, risposte dirette.

L'audit usa un **punteggio deterministico basato su checklist** (risultati riproducibili) e produce un **report scaricabile in Word (.docx) e PDF**. Per i siti renderizzati in JavaScript il contenuto viene ri-renderizzato nel browser prima della valutazione.

**Esempi d'uso:**

> - "Can you audit `example.com` for SEO?"
> - "Audit this URL for AI search readiness: `example.com`"
> - "Run a full SEO, GEO, and AEO audit on my website"

Claude chiede se preferisci un **Quick Audit** (problemi principali e punteggi) o un **Full Audit** (analisi completa), quindi effettua il crawl di più pagine prima di consegnare il report.

**Integrazione opzionale PageSpeed Insights** — per misurare i Core Web Vitals e i punteggi Lighthouse, configura una API key gratuita di Google PageSpeed Insights:

1. Su [console.cloud.google.com](https://console.cloud.google.com) crea o seleziona un progetto.
2. **APIs & Services → Library →** cerca **PageSpeed Insights API → Enable**.
3. **APIs & Services → Credentials → Create credentials → API key**, copia la chiave.
4. Nella root del plugin copia `.pagespeed.key.example` in `.pagespeed.key` e incolla la chiave su una sola riga.

`.pagespeed.key` è in `.gitignore`: non va mai committato. L'API è raggiunta tramite il canale browser di Claude-in-Chrome (non dal sandbox); senza un browser connesso l'audit gira comunque, ma senza dati di performance.

> Oltre che come plugin Claude Code, questa skill può essere usata anche nell'app desktop **Claude Cowork** caricando il pacchetto ZIP della cartella `seo-plugin` (Customize → Skills → **+**).

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
│   ├── skills/seo-geo-aeo/SKILL.md
│   ├── skills/README.md
│   ├── .pagespeed.key.example
│   └── .gitignore
├── perf-audit-plugin/
│   ├── .claude-plugin/plugin.json
│   ├── commands/perf-audit.md
│   └── README.md
└── README.md                     ← questo file
```

Ogni plugin ha:

- un `plugin.json` con nome, versione, descrizione e autore;
- una `SKILL.md` (per le skill) o un file in `commands/` (per i command) che contiene le istruzioni operative — la *source of truth* del comportamento;
- un `README.md` dedicato con installazione e dettagli specifici.

---

## Sviluppo e contributi

- **Aggiungere un plugin**: crea una nuova cartella `<nome>-plugin/` con il suo `.claude-plugin/plugin.json` e la skill/command, poi aggiungi la voce corrispondente in `.claude-plugin/marketplace.json`.
- **Versionamento**: aggiorna il campo `version` nel `plugin.json` del plugin modificato.
- **Autore / contatti**: Emanuele Toffolon — `emanuele.toffolon@enesi.it`.

---

*Plugin Claude Code — Enesi srl.*
