# perf-audit — Plugin Claude Code

Audit di performance **a imbuto** (misura → isola → scava) per siti **Master Laravel Enesi**
(Laravel 12, UUID, tabelle di traduzione, URL SEO `rewurl_*`, codice in `private/`).

Il comando guida Claude in un audit completo end-to-end: prima **localizza** dove sta il tempo
(server vs DB vs rete vs frontend) con misure black-box, poi **scende** nel dettaglio solo sul
layer colpevole, confrontando ogni numero con soglie di riferimento e producendo un **report
finale prioritizzato** nel formato `Problema | Evidenza | Impatto | Fix | Sforzo`.

## Comando

```
/perf-audit [url-prod] [url-staging-o-locale]
```

Esempi:
```
/perf-audit https://www.sito.it
/perf-audit https://www.sito.it http://127.0.0.1:8000
```

- **Senza argomenti** usa il locale `http://127.0.0.1:8000` e avvisa che, senza l'URL di
  produzione, la diagnosi del problema reale del cliente è parziale.
- **Con più URL**, il **primo** è il bersaglio principale (di solito produzione), gli altri
  servono per confronto (staging/locale).

## Cosa fa (fasi)

| Fase | Cosa misura | Layer |
|---|---|---|
| **0** | Setup, detect `.env`/tool, scelta delle **URL di prova reali** (rotte SEO `rewurl_*`) | — |
| **1** | Baseline **TTFB** black-box (cold/warm, DNS/connect/TLS separati) | rete + server |
| **2** | **Carico/concorrenza** con `hey`/`ab`/`wrk` (percentili p50…p99, punto di rottura) | infra |
| **3** | Profiling server-side: **query count & N+1** (Debugbar / Telescope / Clockwork) | app + DB |
| **4** | **Database**: top query per latenza, `EXPLAIN`, indici mancanti su FK UUID e `rewurl` | DB |
| **5** | **Cache & config infra**: OPcache, cache buildate, driver cache/sessione, Redis, Meili | infra |
| **6** | **Chiamate esterne sincrone** nel render (Stripe/shipping/Meili/social) | app |
| **7** | **Frontend/delivery**: Lighthouse (LCP/TBT/CLS), gzip/brotli, HTTP/2, asset, immagini | frontend |
| **8** | **Server**: CPU/RAM, PHP-FPM `pm.max_children`, `innodb_buffer_pool_size`, tmp tables | infra |

Audit ampi vengono delegati a **subagenti read-only paralleli** (DB / Frontend / Codice-N+1 /
Infra): ognuno restituisce solo un riassunto strutturato, senza intasare il contesto con i
dump grezzi.

## Metodo di misura (importante)

- **Misura attraverso il web server reale (PHP-FPM)**, non `php artisan serve` (gira senza
  OPcache → numeri non rappresentativi).
- **Doppio passaggio:**
  - **Pass A — diagnostico** (`APP_DEBUG=true` + Debugbar/Telescope): conta query, scova N+1,
    individua le query lente. I tempi sono gonfiati dall'overhead, ma la **struttura** è reale.
  - **Pass B — realistico** (`APP_DEBUG=false`, OPcache on, `config/route/view/event:cache`,
    Redis): misura il **TTFB/carico/Lighthouse che il cliente sente davvero**.
- Le prove invasive (Debugbar/Telescope, slow log, flip a `production`) vanno fatte su
  **locale/staging**, mai modificando l'`.env` di produzione.

## Strumenti di monitoraggio e profiling Laravel

Il comando usa **Debugbar** come default (è già nel progetto Master), ma sa attingere anche agli
altri strumenti di debug/monitoring se presenti — scegliendoli in base alla fase:

- **Laravel Telescope** — debug server-side ricco (query con duplicati → N+1, modelli, cache,
  job, mail, eventi, eccezioni, slow query); ottimo per ispezionare *a posteriori* anche
  richieste AJAX/API. Solo locale/staging; in produzione disabilitato o protetto da gate + pruning.
- **Laravel Pulse** — monitoring leggero per la **produzione** (slow requests/queries/jobs,
  cache, eccezioni): la fonte migliore per capire cosa è lento sul **traffico reale**.
- **Clockwork** — alternativa a Debugbar/Telescope via estensione browser, overhead minore.
- **Profiler CPU** (oltre il conteggio query): **Blackfire**, **Tideways**, **Xdebug profiler**,
  **SPX** — per gli hotspot quando le query sono già a posto ma il tempo CPU resta alto.
- **APM / error+perf**: **Sentry** (tracing), **New Relic**, **Datadog**, **Bugsnag**,
  **Inspector** — se già collegati, le transaction trace di produzione valgono più di ogni
  misura sintetica.

Regola: usa Telescope/Debugbar/Clockwork/Blackfire per la **diagnosi** su locale-staging;
usa Pulse/APM + misure black-box (Fasi 1–2) per il **numero reale** in produzione.

## Sicurezza / autorizzazioni

- I **load test** (Fase 2) generano traffico reale: contro produzione **solo** con autorizzazione
  esplicita e a volumi bassi (`-n 200 -c 20`), **solo su pagine in lettura** (mai checkout/ordini).
- Prima di flippare a `production` per il Pass B, verifica che chiavi Stripe/SMTP/marketing siano
  sandbox/placeholder.
- Le Fasi 0, 1, 3–8 sono in sola lettura/non distruttive.

## Output

Report `private/storage/perf-audit/report-<TS>.md` con: contesto, tabella baseline per rotta,
**finding prioritizzati** (ordinati per impatto/sforzo), confronto prod vs staging, e separazione
tra **quick-win** (≤1h) e **interventi strutturali**. In chat un riepilogo di 5 righe con il collo
di bottiglia, i 3 interventi a maggior ritorno e cosa serve per la fase successiva.

## Requisiti consigliati sull'ambiente

- `curl`, `ab`, `wrk` nativi; `hey` e `lighthouse` via Docker (immagini `williamyeh/hey`,
  `femtopixel/google-lighthouse`); `jq`.
- Accesso al codice del sito in `private/` e, idealmente, al DB di produzione (`performance_schema`
  attivo) e/o SSH per la Fase 8.

## Installazione (da marketplace)

```
/plugin marketplace add <owner>/<repo-marketplace>
/plugin install perf-audit@<nome-marketplace>
```

## Struttura

```
perf-audit-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── perf-audit.md
└── README.md
```
