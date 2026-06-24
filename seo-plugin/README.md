
# seo-geo-aeo — Report SEO / GEO / AEO Enesi

Command `/seo-report` che produce un **report SEO / GEO / AEO in italiano con il design system Enesi** (PDF A4 brandizzato).

- **SEO** — motori tradizionali (Google, Bing): title, meta, heading, schema, link interni, contenuti, performance
- **GEO** — *Generative Engine Optimization* per i motori AI (Perplexity, ChatGPT Search, AI Overviews, Gemini): E-E-A-T, entità, densità informativa, autorevolezza
- **AEO** — *Answer Engine Optimization* per featured snippet e ricerca vocale: FAQ/HowTo schema, heading-domanda, risposte dirette

## Perché un command e non una skill

Con il plugin **[`claude-seo`](https://github.com/AgriciDaniel/claude-seo)** installato (il motore di analisi), le sue skill (`seo`, `seo-audit`, `seo-page`, …) si attivano sugli stessi trigger naturali ("SEO", "audit", una URL). Una nostra skill auto-attivante competerebbe con quelle, col rischio concreto di eseguire la skill sbagliata. Per questo il plugin espone uno **slash command esplicito**: lo invochi tu, e lui delega in modo altrettanto esplicito alla skill `claude-seo:seo-audit`. Nessuna ambiguità di routing.

## Architettura: orchestratore + report Enesi

```
/seo-report <url> [quick|full]
        │  1. verifica che claude-seo sia installato
        │  2. controlla le credenziali Google/API e AVVISA l'utente (CWV reali vs euristici)
        │  3. esegue l'audit: claude-seo:seo-audit  ← motore di analisi
        │  4. ingerisce l'envelope completo (audit-data.json, findings/*, action plan)
        │  5. compila assets/report-template.html (italiano) e lo converte in PDF (WeasyPrint)
        ▼
  report-seo-<dominio>-<data>.pdf  (design system Enesi)
```

`claude-seo` fa l'analisi (technical SEO, content/E-E-A-T, schema, CWV, AI-search/GEO, hreflang, local, SXO, immagini, health score /100 e action plan). Il command aggiunge i tre assi SEO/GEO/AEO, il design Enesi, la lingua italiana e il PDF da consegnare al cliente.

## Requisiti

- Plugin **`claude-seo`** installato (motore di analisi):
  ```bash
  claude plugin marketplace add https://github.com/AgriciDaniel/claude-seo
  claude plugin install claude-seo
  ```
- **WeasyPrint** per la conversione HTML→PDF (già installata da `claude-seo`; in alternativa `pip install weasyprint`). Dipendenze native: `cairo`, `pango`.
- *(Opzionale, consigliato)* credenziali Google/API in `claude-seo`. **CrUX e PageSpeed sono API, non browser**: con le credenziali, `/seo google` restituisce Core Web Vitals **di campo** reali senza alcun browser headless — l'unico modo per avere performance reali rispettando le regole Enesi. Prima di ogni audit il command **verifica le credenziali** (`google_auth.py --check` / `backlinks_auth.py --check`) e, se mancano, **avvisa l'utente** lasciando scegliere se configurarle o procedere con le stime.

> **Generazione PDF senza browser.** Il command usa **WeasyPrint** perché le regole globali Enesi vietano l'avvio di un browser in modalità headless. Chrome `--headless --print-to-pdf` resta solo come fallback, da autorizzare esplicitamente.

## Uso

```
/seo-report <url> [quick|full]
```

- `<url>`: il sito da auditare (se omesso, viene chiesto).
- `quick` | `full` (facoltativo): profondità dell'audit. Se omesso, il command chiede.

Esempi:

```
/seo-report https://www.esempio.it full
/seo-report esempio.it
```

## Struttura

```
seo-plugin/
├── .claude-plugin/plugin.json
├── assets/
│   └── report-template.html     ← template report Enesi (fonte di verità del design)
├── commands/
│   └── seo-report.md            ← istruzioni del command (source of truth del comportamento)
└── README.md
```

Il template `assets/report-template.html` è la versione autonoma (senza runtime design-component) del template `templates/report-seo/ReportSeo.dc.html` del progetto **Enesi Design System** su claude.ai/design: fino a 14 pagine A4 stampabili (inclusa una pagina **SXO/persone** opzionale e le criticità paginate), token colori/tipografia inline, font Geist via CDN con fallback di sistema, logo Enesi inline. Le pagine interne riportano il numero di pagina nell'header; il footer (indirizzo legale Enesi + data) è solo sulla cover.

## Storico versioni

**3.0.0** — Da skill a **command esplicito** (`/seo-report`)
- Convertito da skill auto-attivante a slash command per evitare conflitti di routing con le skill di `claude-seo`
- Ingestione dell'envelope completo di `claude-seo` (severità complete, SXO, business intelligence), campionamento della lingua del cliente, verifica HTTP degli header reali
- Controllo credenziali Google/backlink con avviso all'utente prima dell'audit
- Template: pagina SXO opzionale, paginazione robusta (pagine ad altezza fissa, footer solo in cover, numero pagina nell'header), indirizzo legale Enesi in cover

**2.0.0** — Riscrittura come orchestratore + report Enesi (delega a `claude-seo`, report PDF italiano)

**1.x** — Skill standalone precedente (scoring deterministico interno, output Word/PDF generico)
