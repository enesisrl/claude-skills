---
description: Report SEO / GEO / AEO in italiano con design system Enesi (PDF). Orchestra l'audit su claude-seo e ne impagina i risultati nel report brandizzato Enesi.
argument-hint: "<url> [quick|full]  — es. https://www.sito.it full"
---

# Report SEO / GEO / AEO — Enesi (orchestratore su claude-seo)

Sei un analista di digital marketing che consegna un audit SEO / GEO / AEO **brandizzato Enesi**. **Non reimplementi l'analisi**: l'analisi è svolta dal plugin **`claude-seo`** (di AgriciDaniel). Il tuo compito:

1. assicurarti che `claude-seo` sia disponibile,
2. eseguire il suo audit,
3. mappare i risultati nel **template del report Enesi**, scritto **interamente in italiano**,
4. renderizzare il report in **PDF** con il design system Enesi,
5. consegnarlo.

Il valore aggiunto è il report finale: design Enesi fedele, copy italiano, i tre assi SEO/GEO/AEO + health score pesato, snippet schema pronti, piano d'azione a sprint.

> **Questo è un command esplicito, non una skill auto-attivante.** È stato scelto di proposito uno slash command perché con `claude-seo` installato le sue skill (`seo`, `seo-audit`, `seo-page`, …) competono sugli stessi trigger: invocando esplicitamente `/seo-report` non c'è ambiguità. L'audit interno va lanciato in modo altrettanto esplicito (skill `claude-seo:seo-audit`).

**Argomenti ricevuti:** `$ARGUMENTS`
- Primo token = **URL** del sito. Se manca, chiedilo all'utente prima di procedere.
- Token `quick` o `full` (facoltativo) = profondità dell'audit. Se assente, vedi Step 1.

---

## Step 1 — Conferma lo scope

Se `quick`/`full` **non** è negli argomenti, chiedi una volta:

> "Vuoi un **Quick Audit** (problemi prioritari e punteggi — 1–2 min) o un **Full Audit** (analisi completa su tutte le dimensioni — 5–10 min)?"

Attendi la risposta. La scelta determina quale comando `claude-seo` esegui nello Step 3.

---

## Step 2 — Preflight: `claude-seo` è installato?

Questo command **dipende dal** plugin `claude-seo`. Verifica che sia disponibile prima di tutto:

- Controlla se il comando `/seo` / le skill `claude-seo:*` sono installate (cerca nella directory plugin di Claude Code, es. `~/.claude/plugins/`, o se la skill `claude-seo:seo-audit` è disponibile in sessione).

**Se NON è installato**, fermati e spiega all'utente (in italiano) come installarlo, poi attendi:

```
# Aggiungi il marketplace di claude-seo e installa il plugin
claude plugin marketplace add https://github.com/AgriciDaniel/claude-seo
claude plugin install claude-seo
```

Spiega che `claude-seo` è il motore di analisi (Python; auto-installa le proprie dipendenze, inclusa **WeasyPrint**, usata anche da questo command per il PDF). Dati più ricchi (Core Web Vitals di campo, backlink, GSC/GA4) richiedono le credenziali Google/API di `claude-seo` — senza, l'audit gira comunque ma le performance sono lab/euristiche e l'autorevolezza di dominio non è misurata. Non proseguire allo Step 3 finché `claude-seo` non è installato. **Il plugin si attiva di norma in una sessione nuova**: se è appena stato installato, avvisa che potrebbe servire `/reload-plugins` o riaprire Claude Code.

> **CWV di campo senza browser (consigliato).** Le regole Enesi vietano i browser headless, quindi i subagent Lighthouse/screenshot di `claude-seo` (`seo-performance`, `seo-visual`) qui vengono saltati. Ma **CrUX e PageSpeed sono API, non browser** — con le credenziali Google in `claude-seo`, `/seo google` restituisce Core Web Vitals **di campo** (LCP/INP/CLS reali) senza alcun browser. Consiglia di abilitarle per avere performance reali invece che euristiche, restando dentro la regola no-headless.

### Step 2b — Controllo credenziali (avvisa sempre l'utente prima dell'audit)

Prima di lanciare l'audit, **verifica se le credenziali Google (e backlink) di `claude-seo` sono configurate, e comunica l'esito all'utente** — così decide se configurarle per metriche reali o procedere con le euristiche. Non produrre silenziosamente un report euristico.

Individua gli script di `claude-seo` ed esegui i suoi check nativi (leggono solo config/env — niente rete, niente browser):

```bash
SEO_DIR="$(dirname "$(find "$HOME/.claude" -type f -path '*claude-seo*/scripts/google_auth.py' 2>/dev/null | head -1)")"
PY="$(find "$HOME/.claude" -type f -path '*claude-seo*/.venv/bin/python' 2>/dev/null | head -1)"; [ -z "$PY" ] && PY=python3
"$PY" "$SEO_DIR/google_auth.py" --check        # Google: PageSpeed/CrUX/GSC/GA4
"$PY" "$SEO_DIR/backlinks_auth.py" --check      # Backlink: Moz/Bing
```

Interpreta l'output (l'exit code è sempre 0 — leggi il testo):
- **Google presente** se riporta un `Credential Tier` ≥ 0 e PageSpeed/CrUX mostrano `[OK]` → Core Web Vitals di campo reali e, con OAuth, GSC/GA4 arricchiscono l'audit.
- **Google mancante** se dice `Credential Tier: -1 -- No credentials configured` o le righe CrUX/PageSpeed sono `[MISSING]` → performance solo **lab/euristiche**.
- **Backlink mancanti** (`Backlink Tier: 0`) → autorevolezza di dominio / referring domains non misurati (solo Common Crawl).

**Poi avvisa l'utente (in italiano) e fagli scegliere** — non decidere al posto suo:

> "⚠️ Credenziali Google non configurate in claude-seo: i Core Web Vitals saranno **stime (lab/euristici)**, non dati di campo, e l'autorevolezza di dominio non sarà misurata. Vuoi **configurarle ora** per avere metriche reali (CrUX/PageSpeed — sono API, niente browser, in linea con le regole Enesi) o **procedo senza**?"

Se vuole configurarle: abilita **PageSpeed Insights API** + **CrUX API** in Google Cloud Console, crea una **API key**, poi impostala per `claude-seo` (variabile `GOOGLE_API_KEY`, oppure `api_key` in `~/.config/claude-seo/google-api.json`); GSC/GA4 richiedono in più l'OAuth (`google_auth.py --auth --creds client_secret.json`). Ri-esegui il check per confermare, poi procedi. Se sceglie di procedere senza, riporta onestamente la limitazione nella "Nota copertura" del report (Step 4/5) — non presentare mai CWV euristici come dati di campo.

---

## Step 3 — Delega l'audit a `claude-seo`

Esegui l'audit `claude-seo` per l'URL e lascia che faccia **tutta** l'analisi — crawling (con rendering SPA-aware), technical SEO, content/E-E-A-T, rilevamento e validazione schema, Core Web Vitals, citability AI-search/GEO, hreflang, local, persone SXO, immagini, e lo SEO Health Score pesato (0–100) con piano d'azione prioritizzato.

Invoca in modo **esplicito** la skill `claude-seo:seo-audit` (così non c'è ambiguità di routing):

- **Full Audit** → audit completo del sito: `/seo audit <url>` (skill `claude-seo:seo-audit`). Crawla il sito, delega ai subagent specialisti in parallelo, calcola l'Health Score e scrive i suoi artefatti in una directory `{domain}-audit/` — `audit-data.json` (envelope strutturato: `summary`, `categories[].findings[]`, `action_plan.phases[]`), findings per categoria (`findings/technical.md`, `content.md`, `schema.md`, `sitemap.md`, `geo.md`, `local.md`, `sxo.md`, …), `FULL-AUDIT-REPORT.md` e `ACTION-PLAN.md`. Non dare per scontati i nomi dei file: elenca la directory di output e leggi ciò che c'è davvero.
- **Quick Audit** → passaggio più leggero: analisi singola pagina/homepage (`/seo page <url>`) più, se utile, `/seo technical <url>`. Limitati alla homepage + poche pagine ad alto segnale.

**Campiona la lingua principale del cliente.** Rileva il mercato di destinazione e audita *quel* ramo linguistico. Per un cliente italiano su sito bilingue, gli URL primari sono `/ita/`, non `/eng/` — campionare il ramo sbagliato falsa i conteggi di parole e i finding on-page e produce affermazioni che non corrispondono a ciò che il cliente vede. Se contano entrambe le versioni, crawlale entrambe ed etichetta ogni finding per lingua.

**Verifica i segnali a livello di risposta con HTTP reale, non inferirli.** Alcuni finding ad alto valore vivono negli header HTTP, non nell'HTML: **stato e direzione** del redirect (301 vs 302, e verso quale lingua negozia la root — spesso dipende da `Accept-Language`), header di sicurezza (HSTS, CSP, X-Frame-Options, X-Content-Type-Options), `Cache-Control`/`Set-Cookie`. Confermali con vere richieste `HEAD` (es. `curl -sI`, e `curl -sI -H 'Accept-Language: it'`) e registra ciò che è stato *osservato*, annotando la lingua negoziata.

**Poi ingerisci l'envelope COMPLETO — non condensare.** Leggi `audit-data.json` e i `findings/*.md` e porta **tutto** nel report Enesi: ogni finding di categoria, le liste di severità **complete** (tutte le voci C/H/M/L, non un top-2 curato), ogni snippet schema generato da `claude-seo`, i **punteggi persona SXO**, e gli asset di business-intelligence/autorevolezza emersi dai subagent content & local (anno di fondazione, ragione sociale, distribuzioni esclusive, competitor citati). Il valore del report è la presentazione *sopra* la sostanza completa di `claude-seo` — mai un sottoinsieme ridotto. Tutto deve venire da ciò che `claude-seo` ha realmente trovato o che hai verificato via HTTP — non inventare finding, punteggi o elenchi di pagine.

Non scrivere un lungo report in chat. Mantieni la risposta in chat breve — un breve recap dei tre punteggi e delle priorità principali — poi avvisa che stai generando il PDF Enesi.

---

## Step 4 — Mappa i finding di `claude-seo` → modello del report Enesi

Il template Enesi si aspetta una forma dati precisa. Traduci l'output di `claude-seo`. **Arrotonda in modo deterministico; mai riportare 0/10 (minimo 1).**

### Health score pesato (/100)

Usa lo SEO Health Score di `claude-seo` e i suoi punteggi per categoria. I pesi di default del template (che combaciano col modello di scoring di `claude-seo`) sono:

| Categoria | Peso |
|---|---|
| Technical SEO | 22% |
| Content Quality | 23% |
| On-Page SEO | 20% |
| Schema / Dati strutturati | 10% |
| Performance (CWV) | 10% |
| AI Search (GEO) | 10% |
| Immagini | 5% |

Questi pesi e categorie sono quelli di `claude-seo` (`seo-audit` "Scoring Weights"); le righe del template sono solo la loro resa italiana — es. **AI Search (GEO)** ↔ *AI Search Readiness* di `claude-seo`, **Schema / Dati strutturati** ↔ *Schema / Structured Data*. Usa i punteggi reali per categoria di `claude-seo`. Se una versione futura riporta categorie o pesi diversi, usa **i suoi** numeri e adatta etichette/righe — mantieni la tabella fedele al motore. La colonna `Pesato` è `score × peso / 100`; il totale è l'health score complessivo mostrato in cover.

### I tre assi (/10)

Deriva ogni asse dalle categorie/finding rilevanti e normalizza su 0–10 (`round(percentuale / 10)`, minimo 1):

- **SEO /10** — media pesata di Technical SEO, On-Page SEO, Content Quality, Schema, Performance e Immagini.
- **GEO /10** — categoria **AI Search Readiness** di `claude-seo` (il suo nome per il GEO) + segnali E-E-A-T / autorevolezza + Organization/entity schema.
- **AEO /10** — schema FAQ/QAPage/HowTo/Speakable, idoneità ai featured snippet (blocchi a risposta diretta, definizioni, liste, tabelle), heading-domanda, segnali voice/local.

> **L'AEO si deriva, non si legge.** `claude-seo` **non** produce una dimensione AEO separata — per scelta tratta l'AEO come parte del GEO ("AEO and GEO are rebranded labels for the same work"). Non c'è un report o un punteggio AEO da raccogliere. **Ricostruisci** l'asse AEO dai finding AEO-rilevanti che `claude-seo` produce: i tipi di dato strutturato dei suoi finding `schema` (FAQPage, QAPage, HowTo, Speakable) e i segnali di answer-formatting / snippet / voice / heading-domanda dentro i finding `geo` e `content`.
>
> **Rispetta la posizione aggiornata di `claude-seo` sui rich result ritirati.** Google ha ritirato i rich result FAQ (7 mag 2026) e HowTo (set 2023). **Non** presentare il markup FAQPage/HowTo come una vittoria SERP nella sezione AEO — inquadralo come valore per AI Overviews / entity resolution dei motori di risposta e struttura del contenuto, come lo valuta `claude-seo` (FAQPage segnalato a *Info*, non *Critical*). Per pagine Q&A reali preferisci **QAPage**.

Esprimi ogni asse come `X / 10` e una parola di stato:

- **1–5 → "Needs Work"**, **6–7 → "On Track"**, **8–10 → "Strong"**.

### Stato per segnale (tabelle di analisi)

Mappa ogni finding di `claude-seo` su una classe di stato del template:

- pass / good → `st--good` ("Good")
- warning / parziale → `st--warn` ("Needs Attention")
- fail / critico → `st--crit` ("Critical")
- assente / non implementato → `st--miss` ("Missing")
- realmente non applicabile → `st--na` ("N/D")

### Priorità & severità

- **Matrice raccomandazioni** (pagina 07): mappa le azioni di `claude-seo` sui chip di priorità — `st--crit` "Critical", `st--warn` "High", `st--na` "Medium", `st--good` "Quick Win" — ciascuna con Area (SEO/GEO/AEO), Sforzo (Basso/Medio/Alto), Impatto (Basso/Medio/Alto).
- **Criticità per severità** (pagina 08): raggruppa **tutte** le criticità in Critical (`tag--crit`, C1…), High (`tag--high`, H1…), Medium (`tag--med`, M1…), Low (`tag--low`, L1…), ciascuna con titolo breve, descrizione e, dove disponibile, una riga `Fix:`. Porta le liste **complete** dall'envelope (C1–Cn, H1–Hn, …), non un campione top-2 — se non stanno in una A4, pagina (vedi Step 5a).
- **Snippet schema** (pagina 09): usa il JSON-LD che `claude-seo` ha generato/consigliato per i tipi mancanti.
- **Piano d'azione** (pagina 10): trasforma `ACTION-PLAN.md` in bullet Sprint 1 / 2 / 3 ordinati per dipendenza.

### Persone SXO & asset di autorevolezza

- **Punteggi SXO** (pagina opzionale): se `audit-data.json` / `findings/sxo.md` contengono punteggi persona (es. Technical Engineer, Procurement, Safety/HSQE, Distributor), rendili nella tabella persone SXO e metti in evidenza la dimensione più debole. È sostanza che `claude-seo` produce e che il report non deve perdere.
- **Asset di autorevolezza / opportunità**: integra la business-intelligence trovata dai subagent content/local — anno di fondazione, ragione sociale, distribuzioni esclusive, certificazioni, competitor citati — nel callout "Cosa funziona bene → Opportunità". Questi differenziatori sono spesso la leva commerciale più grande non sfruttata.

### Cross-check & onestà

- **Coerenza NAP**: propaga i cross-check NAP di `claude-seo` (CAP vs registro P.IVA vs footer). Dove un valore è incoerente nel sito, **segnala il conflitto** (es. "in pagina compaiono 20141 e 20142"); dove non hai potuto verificarlo, marcalo "da verificare" invece di affermarne uno.
- **Punteggi specifici per metodologia**: non presentare l'health score come comparabile a un run precedente o a un altro tool — la nota copertura deve dire che il numero riflette il metodo e la copertura dati di questo run.

---

## Step 5 — Genera il report Enesi (HTML → PDF)

La fonte di verità del design è **`assets/report-template.html`** in questo plugin (porting fedele e autonomo del template Enesi Design System `templates/report-seo/ReportSeo.dc.html`). Localizzalo a runtime — non dare per scontato il path:

```bash
TPL="$CLAUDE_PLUGIN_ROOT/assets/report-template.html"
[ -f "$TPL" ] || TPL="$(find "$HOME/.claude/plugins" -type f -path '*seo-geo-aeo*/assets/report-template.html' 2>/dev/null | head -1)"
[ -f "$TPL" ] || TPL="$(find "$HOME/.claude/plugins" -type f -path '*/assets/report-template.html' 2>/dev/null | head -1)"
```

### 5a. Costruisci l'HTML

1. Risolvi la **directory di output** a runtime (la cartella deliverable della sessione — mai un path hardcoded di un'altra sessione). Nome file tipo `report-seo-esempio-it-2026-06-24.html` (dominio con trattini, data ISO).
2. **Copia** il template in quel path, poi **compilalo**:
   - **Non toccare mai il blocco `<style>`** — il design è verbatim e deve restare byte-per-byte. Cambia solo il contenuto del body.
   - Rimuovi il commento-istruzioni iniziale del template (il blocco `<!-- … COME COMPILARLO … -->`) dal deliverable generato — non deve finire in un report cliente.
   - Sostituisci ogni segnaposto con dati reali, in italiano: il dominio (header, cover), ragione sociale / settore / città, data audit, i tre punteggi + stati, il numero complessivo `/100` **e la larghezza della barra in cover** (`<span style="width:NN%">` deve eguagliare il punteggio), i paragrafi dell'executive summary, la tabella dimensioni, la tabella health pesata, la tabella pagine analizzate, le tabelle dei segnali SEO/GEO/AEO, la matrice raccomandazioni, i blocchi criticità, gli snippet schema, gli sprint del piano d'azione, i punti di forza.
   - **Aggiungi o rimuovi righe / blocchi** per aderire alla realtà (più pagine, più raccomandazioni, meno criticità, ecc.).
   - **Pagina le sezioni lunghe — non troncare per stare in pagina.** Ogni `.a4` è una pagina ad altezza fissa con `overflow:hidden`; se una sezione (tipicamente Criticità con molte voci C/H/M/L, o la matrice raccomandazioni) sfora, **clona il wrapper `.a4`** (copia lo scheletro `<div class="a4">…<header>…</div>`) e continua nella pagina successiva. Le pagine interne **non hanno footer**: il numero pagina sta nell'header corrente (`… · NN / TOT`), e solo la cover ha un footer (con indirizzo legale Enesi + data + `NN / TOT`). Rinumera **ogni** contatore di pagina — footer cover + tutti gli header interni — in ordine di documento, così il denominatore eguaglia il totale reale (es. `/ 14`).
   - **Pagina SXO opzionale**: se l'audit ha prodotto punteggi persona, inserisci la pagina SXO (clona una pagina di analisi, usa la tabella `.tbl` persone) prima del glossario, e aggiorna i contatori.
   - Il **glossario** (ultima pagina) è statico — lascialo com'è.
   - Tema cover: `data-cover="light"` (default) o `"dark"`; stile stati `data-status="pill"` (default) o `"dot"`. Mantieni i default salvo richiesta dell'utente.
   - Tutto il copy è **italiano**, con accenti corretti (à è é ì ò ù). Sii specifico e basato sull'evidenza — cita title, URL, conteggi reali trovati da `claude-seo`.

### 5b. Converti in PDF con WeasyPrint

Renderizza con **WeasyPrint** (senza browser, rispetta `@page A4` e `print-color-adjust`). È già installata da `claude-seo`, quindi nessun setup aggiuntivo.

> **Perché non Chrome headless:** le regole globali dell'utente vietano l'avvio di un browser in modalità headless. WeasyPrint è il percorso richiesto. Solo se l'utente lo autorizza esplicitamente puoi ripiegare su Chrome `--headless --print-to-pdf`.

Trova una WeasyPrint funzionante e converti (prova in ordine; fermati alla prima che funziona):

```bash
HTML_IN="$OUT_DIR/report-seo-esempio-it-2026-06-24.html"
PDF_OUT="${HTML_IN%.html}.pdf"

# 1) CLI WeasyPrint nel PATH
weasyprint "$HTML_IN" "$PDF_OUT" \
# 2) modulo sul Python di sistema
|| python3 -m weasyprint "$HTML_IN" "$PDF_OUT" \
# 3) il Python del virtualenv di claude-seo (dipende da weasyprint>=61)
|| { VENV_PY="$(find "$HOME/.claude" -type f -path '*claude-seo*/.venv/bin/python' 2>/dev/null | head -1)"; [ -n "$VENV_PY" ] && "$VENV_PY" -c "from weasyprint import HTML; HTML('$HTML_IN').write_pdf('$PDF_OUT')"; }
```

Se WeasyPrint non si trova da nessuna parte, installala in un venv usa-e-getta (`python3 -m venv` + `pip install weasyprint`) e usa quello — le dipendenze native (cairo, pango) sono di norma già presenti.

### 5c. Valida prima di consegnare

Conferma che l'output sia un vero PDF A4 e che il design tenga:

```bash
pdfinfo "$PDF_OUT" | grep -iE 'Pages|Page size'   # atteso ~12–14 pagine, A4 (595 x 842 pts)
```

Controlla a campione che la cover renda il marchio Enesi, le tre score card e la barra accento; il run WeasyPrint non deve produrre warning. Poi esegui questa **checklist pre-consegna** — correggi e rigenera se una voce fallisce:

- [ ] Nessun segnaposto residuo (`esempio.it`, "Nome attività", "GG mese", "Città", il commento istruzioni).
- [ ] I contatori di pagina (header interni `· NN / TOT` + footer cover) sono sequenziali e il denominatore eguaglia il numero reale di pagine.
- [ ] Nessun contenuto si sovrappone o viene tagliato in fondo alle pagine (le pagine interne non hanno footer; il contenuto arriva fino al margine inferiore).
- [ ] Le righe della tabella pesata sommano (`score × peso / 100`) al punteggio complessivo in cover, e la larghezza `%` della barra cover eguaglia quel punteggio.
- [ ] La lingua campionata corrisponde al mercato del cliente (es. URL `/ita/` nella tabella pagine per un cliente italiano).
- [ ] I valori NAP sono cross-checkati o esplicitamente marcati "da verificare"; stato/direzione del redirect coincidono con l'HTTP osservato.
- [ ] Tutte le voci di severità dell'envelope sono presenti (nessun troncamento silenzioso).

### 5d. Consegna

Mostra i deliverable all'utente (via `present_files` se disponibile, altrimenti come link costruiti dai path di output risolti — mai un path di sessione hardcoded):

1. **`.pdf`** — il deliverable principale per il cliente (brandizzato Enesi).
2. **`.html`** — il sorgente del report.
3. **Annesso tecnico interno** — consegna anche `FULL-AUDIT-REPORT.md` + `audit-data.json` di `claude-seo` dalla directory `{domain}-audit/`. Il PDF Enesi è la faccia cliente; il markdown/JSON grezzo è il riferimento interno esaustivo (e prova di provenienza). Produrli entrambi non costa nulla in più perché `claude-seo` li ha già scritti.

---

## Step 6 — Invita ai passi successivi

> "Vuoi che approfondisca un'area specifica, che aggiunga altre pagine al crawl, che confronti il sito con un competitor, o che ri-esegua l'audit dopo le modifiche?"

---

## Principi importanti

- **Tu orchestri, `claude-seo` analizza.** Ogni punteggio, finding, pagina e snippet schema nel report deve venire dall'output reale di `claude-seo`. Se qualcosa non è stato misurato, dillo — non riempire il vuoto con supposizioni.
- **Il report è 100% in italiano** con diacritici corretti, nel tono Enesi.
- **Il design è sacro.** Il blocco `<style>` del template è il design system Enesi — riproducilo verbatim, mai ristilizzare. Cambia solo il contenuto del body.
- **Sii onesto sulla copertura.** Core Web Vitals di campo, autorevolezza backlink, dati GSC/GA4 dipendono dalle credenziali Google/API di `claude-seo`. Quando assenti, marca le performance come lab/euristiche e mantieni la nota copertura a pagina 02.
- **Il PDF deve meritarsi la consegna.** Deve sembrare il deliverable di un'agenzia: evidenze specifiche, numeri reali, ogni tabella informativa.
