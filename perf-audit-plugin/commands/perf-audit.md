---
description: Audit performance a imbuto per siti Master Laravel Enesi (TTFB, carico, N+1, DB, cache, frontend) con report finale
argument-hint: "[url-prod] [url-staging-o-locale]  — es. https://www.sito.it http://127.0.0.1:8000"
allowed-tools: Bash, Read, Grep, Glob, Write, Edit
---

Stai eseguendo un **audit di performance** su un sito ecommerce **Master Laravel Enesi** (Laravel 12, UUID, tabelle di traduzione, URL SEO `rewurl_*`). Il codice Laravel vive in `private/`.

**Target indicati dall'utente:** `$ARGUMENTS`
- Se vuoto: usa il locale `http://127.0.0.1:8000` (avvialo con `php artisan serve` da `private/` se serve) e avvisa che senza l'URL di **produzione** la diagnosi del problema reale del cliente è parziale.
- Se più URL: il **primo** è il bersaglio principale (di solito produzione), gli altri servono per confronto (staging/locale).

## Principio guida (NON saltarlo)
Procedi **a imbuto: misura → isola → scava**. Non ottimizzare a caso. Prima localizza *dove* sta il tempo (server vs DB vs rete vs frontend) con misure black-box, poi scendi nel dettaglio solo sul layer colpevole. Ad ogni fase confronta i numeri con le **soglie di riferimento** e annota i *finding* (problema → evidenza → impatto → fix → sforzo).

## Sicurezza / autorizzazioni
- I **load test** (Fase 2) generano traffico reale. Eseguili contro produzione **solo** se l'utente ha confermato l'autorizzazione, e tieni volumi bassi (`-n 200 -c 20`). Su ambienti non tuoi, chiedi prima.
- Fasi 0,1,3–7 sono sostanzialmente in sola lettura/non distruttive.
- Non modificare l'`.env` di produzione. Le prove invasive (abilitare Debugbar, slow log) falle su locale/staging.

---

## Fase 0 — Setup e detect (sempre)
```bash
cd private
echo "ENV:" ; grep -E '^(APP_ENV|APP_DEBUG|CACHE_STORE|CACHE_DRIVER|SESSION_DRIVER|QUEUE_CONNECTION|DB_CONNECTION|SCOUT_DRIVER|MEILISEARCH_HOST|REDIS_HOST|REDIS_CLIENT|FRONT_CACHE_PRODUCT_TTL|OCTANE)' .env | sed -E 's/(PASSWORD|SECRET|KEY)=.*/\1=***/'
echo "PHP:" ; php -v | head -1
echo "TOOLS (nativi):" ; for t in curl ab wrk jq docker; do printf "%s=%s " "$t" "$(command -v $t || echo no)"; done; echo
echo "TOOLS (via Docker):" ; docker image inspect williamyeh/hey >/dev/null 2>&1 && echo -n "hey=ok " || echo -n "hey=manca(docker pull williamyeh/hey) "; docker image inspect femtopixel/google-lighthouse >/dev/null 2>&1 && echo "lighthouse=ok" || echo "lighthouse=manca(docker pull femtopixel/google-lighthouse)"
TS=$(date +%Y%m%d-%H%M%S); mkdir -p storage/perf-audit; echo "Report dir: storage/perf-audit (TS=$TS)"
```
Annota: `APP_DEBUG` deve essere **false** in produzione (se è `true` è già un finding grosso). `jq` torna utile per leggere i JSON di Debugbar/Lighthouse — se manca, usa `php -r`/Read.

> **Stack di questa macchina (già predisposto):** `curl`, `ab`, `wrk` nativi · `hey` via Docker (`williamyeh/hey`) · `lighthouse` via Docker (`femtopixel/google-lighthouse`). `docker` e `sudo` funzionano senza password.
> **Nota network per target LOCALE:** i tool in container non vedono `127.0.0.1:8000` dell'host → aggiungi `--network host` al `docker run`. Per URL pubblici (produzione) non serve.

**Scegli le URL di prova reali.** Le rotte sono SEO (`rewurl_*`), non `/product/123`. Recupera esempi veri:
```bash
# dal sitemap (se esiste) -> una home, una categoria, una scheda prodotto, una pagina blog
curl -s "$TARGET/sitemap.xml" | grep -oE '<loc>[^<]+' | sed 's/<loc>//' | head -40
```
Se il sitemap non aiuta, prendili dal DB da `private/`:
```bash
php artisan tinker --execute='echo \DB::table("product_translations")->whereNotNull("rewurl")->value("rewurl");'
```
Definisci un set di **rotte rappresentative**: `home`, `lista-categoria`, `scheda-prodotto`, `ricerca`, `carrello`. Riusa lo stesso set in tutte le fasi.

## Ambiente di misura (CRUCIALE — leggere prima di misurare)
1. **Misura attraverso il web server reale (PHP-FPM), NON `php artisan serve`.** Il built-in server CLI gira **senza OPcache** → numeri non rappresentativi (più lenti del reale). Punta l'URL servito da Nginx/Apache+FPM (es. `https://<host>`); con cert self-signed usa `curl -k`. Su aaPanel/BT il PHP sta in `/www/server/php/<ver>/`.
2. **La config potrebbe essere già in cache.** Se esiste `bootstrap/cache/config.php`, **ogni modifica a `.env` è ignorata** finché non rigeneri: `php artisan config:clear && php artisan config:cache`. Vale anche per `route:cache`/`view:cache`.
3. **Verifica OPcache su FPM** (non sul CLI): cerca `zend_extension=opcache` (non commentato con `;`) e `opcache.enable=1` nella `php.ini` di FPM. OPcache off è quasi sempre il **collo di bottiglia #1** su hardware scarso.
4. **Metodo a DUE PASSAGGI** (non confondere i numeri):
   - **Pass A — diagnostico:** `APP_DEBUG=true` + Debugbar attivo → conta query, scova N+1, query lente (Fasi 3–4). Numeri *gonfiati* dall'overhead di debug, ma la **struttura** (n° query, N+1) è reale.
   - **Pass B — realistico:** `APP_DEBUG=false`, OPcache on, `config/route/view/event:cache` buildate → misura **TTFB/carico/Lighthouse** che il cliente sente davvero (Fasi 1–2, 7). Questo è il numero "vero".
   - La leva di velocità non è `APP_ENV` di per sé, ma **`APP_DEBUG=false` + OPcache + cache buildate + Redis**. Fai sempre backup di `.env` prima.
5. **Sicurezza load test:** prima di flippare a `production`, verifica che chiavi **Stripe/SMTP/marketing** siano sandbox/placeholder, altrimenti rischi addebiti/email reali. In ogni caso **load-test solo pagine in lettura** (home, categoria, prodotto, ricerca) — **mai** submit di checkout/ordini.

## Esecuzione con subagenti (velocità + contesto)
Per audit ampi **non eseguire tutto inline**: il profiling genera output enormi (dump Debugbar, JSON Lighthouse, liste query) che intasano il contesto principale. Delega le fasi **indipendenti e in sola lettura** a **subagenti paralleli** (tool `Agent`); ognuno restituisce **solo un riassunto strutturato**, non i dati grezzi.

**Sequenza:**
1. **Main agent (inline):** Fase 0 (setup/detect) + scelta delle **URL di prova condivise** + Fasi 1–2 (TTFB/carico, brevi). Fissa una volta `$TARGET`, il set di rotte e `$TS` e passali a TUTTI i subagenti, così misurano le stesse URL.
2. **Fan-out: 4 subagenti read-only in parallelo** (un solo messaggio con 4 chiamate `Agent`):

   | Subagente | agentType | Copre | Restituisce |
   |---|---|---|---|
   | **DB** | `general-purpose` | Fase 4 | top query per latenza (`performance_schema`), EXPLAIN delle peggiori, indici mancanti su FK UUID/`meta_rewurl` |
   | **Frontend** | `general-purpose` | Fase 7 | Lighthouse (score, LCP/TBT/CLS) + opportunità in ms |
   | **Codice/N+1** | `Explore` | Fasi 3 + 6 | n° query e N+1 dai dump Debugbar (o da Telescope/Pulse, se presenti), chiamate HTTP/Stripe sincrone nel render |
   | **Infra** | `general-purpose` | Fasi 5 + 8 | OPcache, cache buildate, driver cache/sessione, risorse server, buffer pool |

3. **Sintesi (main agent):** raccogli i 4 riassunti e scrivi il report unico `storage/perf-audit/report-$TS.md` con i finding prioritizzati.

**Regole obbligatorie nei prompt dei subagenti:** (a) **sola lettura** — non modificare `.env`/`php.ini`/codice; (b) misurare via web server reale con `curl -k`, **non** `artisan serve`; (c) usare le **stesse URL** e `$TS` ricevuti; (d) restituire **solo** una tabella `Problema | Evidenza | Impatto | Fix | Sforzo` + 3 numeri chiave, niente output grezzi; (e) load test leggero e solo su pagine in lettura.

> **Audit molto grandi o ricorrenti:** valuta un **Workflow** (orchestrazione multi-agente con pipeline + verifica adversariale), che però richiede opt-in esplicito ("usa un workflow"). Per il caso standard bastano i subagenti `Agent` paralleli qui sopra.

## Fase 1 — Baseline TTFB (black-box, prima cosa da misurare)
Per ogni rotta, misura cold poi warm (ripeti 2–3 volte e prendi la mediana). Separa le fasi della richiesta:
```bash
URL="https://www.sito.it/<rotta>"
for i in 1 2 3; do
curl -w 'dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total} size:%{size_download} http:%{http_code}\n' \
  -o /dev/null -s "$URL"
done
```
**Lettura:**
- `ttfb` ≈ tempo server-side (DNS/connect/tls esclusi). È la metrica chiave per la lentezza lato app.
- `total - ttfb` ≈ download corpo HTML (se grande → HTML gonfio).
- **Soglie TTFB:** `<200ms` ottimo · `200–500ms` ok · `500ms–1s` lento · `>1s` grave.
- Se TTFB è alto **anche a warm cache** → problema server/DB (vai Fasi 3–6). Se è alto solo a freddo → problema di cache (Fase 5). Se TTFB è basso ma la pagina "sembra" lenta → problema frontend (Fase 7).
- Confronta prod vs locale/staging: se locale è veloce e prod no → infra/carico/config (Fasi 2,5,8).

## Fase 2 — Carico / concorrenza (solo se autorizzato)
Distingue "codice lento" da "saturazione risorse". Tre livelli, dal più leggero al più aggressivo.

**(a) `hey` — percentili affidabili p50/p90/p95/p99** (il più utile per la diagnosi). Via Docker:
```bash
# target pubblico:
docker run --rm williamyeh/hey -n 200 -c 20 "$URL"
# target locale (dev server sull'host):
docker run --rm --network host williamyeh/hey -n 200 -c 20 "http://127.0.0.1:8000/<rotta>"
```
Output da leggere: blocco *Latency distribution* (10%…99%) e *Requests/sec*. **Il p99 è ciò che il cliente "sente".**

**(b) `ab` — fallback rapido** se Docker non c'è:
```bash
ab -n 200 -c 20 -l "$URL"     # -l tollera body di lunghezza variabile
```

**(c) `wrk` — stress fino al limite + flusso realistico** (nativo). Usalo per trovare il *punto di rottura* o simulare navigazione vera, non un singolo URL:
```bash
# stress single-URL ad alta concorrenza con distribuzione latenza:
wrk -t4 -c50 -d30s --latency "$URL"
# flusso realistico: prepara una lista di PATH (non URL completi) e pesca a caso via script Lua
php artisan tinker --execute='foreach(\DB::table("product_translations")->whereNotNull("rewurl")->inRandomOrder()->limit(50)->pluck("rewurl") as $p){ echo "/".$p."\n"; }' > storage/perf-audit/urls.txt
wc -l storage/perf-audit/urls.txt
wrk -t4 -c50 -d30s --latency -s storage/perf-audit/flow.lua "$BASE_URL"   # $BASE_URL = solo schema+host, es. https://www.sito.it
```
Lo script `flow.lua` viene creato dalla Fase 0-bis qui sotto. Adatta la query dei path alla struttura URL reale del sito (categorie, blog, ricerca), non solo prodotti.

**Lettura (tutte e tre):** confronta p95/p99 col TTFB single-shot della Fase 1. Se sotto carico la coda **esplode** (p99 ≫ p50) → **saturazione**: PHP-FPM `pm.max_children`, lock su sessioni-file, cache su DB, CPU/RAM (vedi Fasi 5 e 8). Se invece la latenza resta piatta ma alta anche a 1 utente → **codice/DB lento** (Fasi 3–4). Tieni i volumi bassi su produzione; spingi forte solo su staging.

### Fase 0-bis — crea lo script Lua per wrk (una volta)
```bash
cat > storage/perf-audit/flow.lua <<'LUA'
-- Distribuisce le richieste su una lista di path letti da urls.txt (uno per riga).
local paths = {}
local f = io.open("storage/perf-audit/urls.txt", "r")
if f then for line in f:lines() do if #line > 0 then paths[#paths+1] = line end end f:close() end
if #paths == 0 then paths = {"/"} end
local i = 0
request = function()
  i = (i % #paths) + 1
  return wrk.format("GET", paths[i])
end
LUA
echo "flow.lua creato."
```

## Fase 3 — Profiling server-side: query count & N+1 (su locale/staging)
Master Enesi + tabelle di traduzione + 0 eager-loading nei controller frontend = forte rischio **N+1** sulle liste. Sfrutta **Debugbar** (già nel progetto, storage in `private/storage/debugbar`).
```bash
# abilita Debugbar sull'ambiente di test (NON in prod)
grep -q DEBUGBAR_ENABLED .env || echo 'DEBUGBAR_ENABLED=true' >> .env
php artisan config:clear >/dev/null
# assicurati che il dev server giri (php artisan serve) e colpisci la rotta LOCALE
curl -s -o /dev/null "http://127.0.0.1:8000/<rotta>"
# leggi l'ultimo dump Debugbar
F=$(ls -t storage/debugbar/*.json 2>/dev/null | head -1); echo "$F"
php -r '$d=json_decode(file_get_contents($argv[1]),true);
 $q=$d["__meta"]??null; $qq=$d["queries"]??[];
 printf("queries=%d  tempo_query=%.1fms  memoria=%s  durata=%sms\n",
   $qq["nb_statements"]??0,($qq["accumulated_duration"]??0)*1000,
   $d["memory"]["peak_usage_str"]??"?",$d["time"]["duration_str"]??"?");
 // raggruppa per query normalizzata -> N+1
 $g=[]; foreach(($qq["statements"]??[]) as $s){$k=preg_replace("/\\d+|\x27[^\x27]*\x27/","?",$s["sql"]); $g[$k]=($g[$k]??0)+1;}
 arsort($g); echo "--- query ripetute (sospetto N+1) ---\n";
 foreach(array_slice($g,0,8,true) as $k=>$n){ if($n>1) printf("%4dx  %s\n",$n,substr($k,0,110)); }' "$F"
```
**Lettura / soglie:**
- **n° query per pagina:** `<20` ok · `20–50` da guardare · `>50` quasi sempre N+1.
- Se una stessa query normalizzata compare `Nx` (es. una `select * from product_translations where product_id = ?` ripetuta per ogni prodotto della lista) → **N+1**: il fix è `->with([...])` nel controller/repository o `$with` sul model.
- Annota le **3 query più lente** (Debugbar le ordina) per la Fase 4.

### Strumenti di monitoraggio/profiling alternativi (se Debugbar non basta o non c'è)
Debugbar è il default qui perché è già nel progetto, ma a seconda di cosa è installato/disponibile valuta questi (tutti **solo su locale/staging**, mai abilitati al volo in produzione):

- **Laravel Telescope** — il più ricco per il debug server-side: registra ogni richiesta con il suo elenco **query** (evidenzia i duplicati → N+1), **modelli idratati**, **cache hit/miss**, **job**, **mail**, **eventi**, **eccezioni** e **slow query**. Utile quando vuoi ispezionare *a posteriori* più richieste (anche AJAX/API, dove Debugbar non rende). Verifica se è già presente e, in caso, leggi i dati da lì:
  ```bash
  # è installato/abilitato?
  ls app/Providers/TelescopeServiceProvider.php 2>/dev/null && grep -E '^TELESCOPE_ENABLED' .env
  php artisan migrate --pretend 2>/dev/null | grep -i telescope   # tabelle presenti?
  # colpisci la rotta, poi leggi conteggio query e duplicati dell'ULTIMA richiesta registrata
  curl -s -o /dev/null "http://127.0.0.1:8000/<rotta>"
  php artisan tinker --execute='
    $b = \DB::table("telescope_entries")->where("type","request")->latest("created_at")->value("batch_id");
    $q = \DB::table("telescope_entries")->where("type","query")->where("batch_id",$b)->pluck("content");
    echo "query totali: ".$q->count()."\n";
    $g=[]; foreach($q as $c){ $sql=preg_replace("/\d+|\x27[^\x27]*\x27/","?", json_decode($c,true)["sql"] ?? ""); $g[$sql]=($g[$sql]??0)+1; }
    arsort($g); foreach(array_slice($g,0,8,true) as $k=>$n){ if($n>1) printf("%4dx  %s\n",$n,substr($k,0,110)); }'
  ```
  Apri anche la dashboard `/telescope` (dietro auth) per la vista visuale. **Attenzione:** Telescope ha overhead e salva dati sensibili → in produzione tienilo disabilitato o protetto da `gate` + `pruning`; non usarlo per misurare il TTFB "vero" (è un tool del Pass A diagnostico, non del Pass B realistico).
- **Laravel Pulse** — pensato per la **produzione**: dashboard leggera con *slow queries*, *slow requests*, *slow jobs*, *cache*, *exceptions*, uso server. Se il cliente lo ha già, è la fonte migliore per capire cosa è lento **sul traffico reale** senza generare carico tu (complementa la Fase 2). Controlla `ls app/Providers/PulseServiceProvider.php` o le tabelle `pulse_*`.
- **Clockwork** — alternativa a Debugbar/Telescope via estensione browser, overhead minore; comoda anche per richieste API/JSON.
- **Profiler a livello di funzione** (CPU hotspot, oltre il conteggio query): **Blackfire**, **Tideways**, **Xdebug profiler** (`xdebug.mode=profile` → cachegrind in KCachegrind/qcachegrind), **SPX**. Usali in Fase 4/8 quando le query sono già a posto ma il tempo CPU resta alto (logica/render Blade pesante).
- **APM / error+perf monitoring**: **Sentry** (Performance/Tracing), **Bugsnag**, **New Relic**, **Datadog**, **Inspector**. Se uno è già collegato, le sue *transaction traces* in produzione valgono più di qualunque misura sintetica — controlla `grep -iE 'sentry|newrelic|bugsnag|inspector|datadog' composer.json .env`.

Regola: per la **diagnosi** (struttura: n° query, N+1, hotspot) usa Telescope/Debugbar/Clockwork/Blackfire su locale-staging; per il **numero che sente il cliente** usa Pulse/APM sui dati di produzione + le misure black-box delle Fasi 1–2.

## Fase 4 — Database (il sospettato n.1)
Da `private/`:
```bash
# top query per latenza totale su PRODUZIONE (richiede performance_schema attivo)
php artisan tinker --execute='foreach(\DB::select("SELECT DIGEST_TEXT, COUNT_STAR c, ROUND(SUM_TIMER_WAIT/1e12,2) tot_s, ROUND(AVG_TIMER_WAIT/1e9,2) avg_ms FROM performance_schema.events_statements_summary_by_digest WHERE SCHEMA_NAME=DATABASE() ORDER BY SUM_TIMER_WAIT DESC LIMIT 15") as $r){ printf("%6.1fs  %6dx  %7.2fms  %s\n",$r->tot_s,$r->c,$r->avg_ms,substr($r->DIGEST_TEXT,0,90)); }'
```
Poi sulle query peggiori (da Fase 3/4):
```bash
php artisan tinker --execute='foreach(\DB::select("EXPLAIN <QUERY>") as $r){ var_dump($r); }'
```
**Cosa cercare:**
- `type=ALL` / `key=NULL` in EXPLAIN → **full table scan** = indice mancante.
- **Indici sui lookup tipici Master:** FK UUID (`product_id`, `language_id`, `*_id`) e colonne slug `rewurl`. Verifica:
  ```bash
  php artisan tinker --execute='foreach(["products","product_translations","product_models","orders","carts"] as $t){ try{ $idx=collect(\DB::select("SHOW INDEX FROM $t"))->pluck("Column_name")->unique()->implode(","); echo "$t: $idx\n"; }catch(\Throwable $e){} }'
  ```
  Se `product_translations` non ha indice su `(product_id)`/`(language_id)`/`(rewurl)` → finding ad alto impatto (35k+ righe).
- **Tipo PK UUID:** `SHOW CREATE TABLE products` — se gli UUID sono `char(36)` gli indici sono grandi e i join lenti; `binary(16)` è molto meglio (nota come miglioria strutturale, non quick-win).
- **Slow query log** (su staging): `SHOW VARIABLES LIKE 'slow_query%'; SHOW VARIABLES LIKE 'long_query_time';` — abilita, riproduci, leggi.

## Fase 5 — Cache & configurazione infra
```bash
echo "--- OPcache ---"; php -i | grep -Ei 'opcache.enable|jit|memory_consumption|max_accelerated_files|revalidate_freq' | head
echo "--- caches buildate ---"; ls -la bootstrap/cache/ 2>/dev/null | grep -E 'config|route|events'
```
**Checklist (confronta con la Fase 0):**
- `CACHE_STORE`/`SESSION_DRIVER` su **database/file** mentre Redis (Predis) è installato → ogni hit di pagina martella MySQL anche per cache/sessioni. **Fix ad alto impatto:** spostare cache e sessioni su **Redis**.
- `MEILISEARCH_HOST` placeholder/non valido → ricerca in timeout/fallback lento.
- In prod devono essere attivi: `php artisan config:cache route:cache view:cache event:cache`, `APP_DEBUG=false`, OPcache ON (`revalidate_freq>0`), idealmente JIT.
- Verifica che il cache prodotto (`FRONT_CACHE_PRODUCT_TTL`) venga davvero colpito (in `front/Main/Views/base/layout.blade.php` e nei service c'è uso di `Cache::`).

## Fase 6 — Chiamate esterne sincrone nel render
Centinaia di ms per pagina se Stripe/shipping/TrustPilot/Meili/Facebook girano nel ciclo di richiesta.
```bash
grep -rn "Http::\|Guzzle\|Stripe\|TrustPilot\|FacebookAds\|->search(" front/Main/Controllers front/Main/Composers front/Main/Services 2>/dev/null | head -30
```
**Fix:** spostare in coda (queue) o cache; mai chiamate HTTP bloccanti durante il render di pagine pubbliche.

## Fase 7 — Frontend / delivery
```bash
curl -sI --http2 "$URL" | grep -iE 'HTTP/|content-encoding|cache-control|content-type'   # gzip/brotli? http2? cache headers?
HTML=$(curl -s "$URL"); echo "scripts=$(grep -oc '<script' <<<"$HTML") css=$(grep -oc 'rel=.stylesheet' <<<"$HTML") img=$(grep -oc '<img' <<<"$HTML") bytes=${#HTML}"
```
**Lighthouse via Docker** (Chrome incluso nell'immagine). L'URL va come **primo** argomento; l'output JSON va scritto nella cartella montata. `--user $(id -u):$(id -g)` evita problemi di permessi sul file generato.
```bash
mkdir -p storage/perf-audit/lh
# target pubblico:
docker run --rm --cap-add=SYS_ADMIN --user $(id -u):$(id -g) \
  -v "$(pwd)/storage/perf-audit/lh:/home/chrome/reports" \
  femtopixel/google-lighthouse "$URL" \
  --quiet --only-categories=performance --output=json \
  --output-path=/home/chrome/reports/lh-$TS.json
# target LOCALE: aggiungi  --network host  e usa http://127.0.0.1:8000/<rotta>
jq '{score:(.categories.performance.score*100),
     LCP:.audits["largest-contentful-paint"].displayValue,
     FCP:.audits["first-contentful-paint"].displayValue,
     TBT:.audits["total-blocking-time"].displayValue,
     CLS:.audits["cumulative-layout-shift"].displayValue,
     SpeedIndex:.audits["speed-index"].displayValue,
     pesoByte:.audits["total-byte-weight"].displayValue}' storage/perf-audit/lh/lh-$TS.json
# diagnosi azionabili più pesanti (opportunità in ms risparmiabili):
jq -r '.audits | to_entries[] | select(.value.details.overallSavingsMs // 0 > 100)
       | "\(.value.details.overallSavingsMs|floor)ms  \(.key)"' storage/perf-audit/lh/lh-$TS.json | sort -rn | head
```
**Soglie performance score:** `≥90` ottimo · `50–89` da migliorare · `<50` grave. **LCP** buono `<2.5s`, **TBT** `<200ms`, **CLS** `<0.1`. Le "opportunità" sopra (es. *unminified-javascript*, *uses-responsive-images*, *uses-text-compression*, *render-blocking-resources*) sono i fix frontend ordinati per ms recuperabili.
**Cosa cercare:** asset non minificati/versionati (occhio: il progetto ha `laravel-mix` legacy in `package.json`), niente gzip/brotli, no HTTP/2, immagini servite full-size invece di WebP/responsive (Spatie MediaLibrary), troppe richieste, render-blocking CSS/JS.

## Fase 8 — Server (solo se sei sulla macchina/SSH)
```bash
nproc; free -m | head -2
php artisan tinker --execute='foreach(\DB::select("SHOW GLOBAL STATUS WHERE Variable_name IN (\"Threads_connected\",\"Slow_queries\",\"Created_tmp_disk_tables\")") as $r){ echo $r->Variable_name.": ".$r->Value."\n"; } foreach(\DB::select("SHOW VARIABLES LIKE \"innodb_buffer_pool_size\"") as $r){ echo $r->Variable_name.": ".round($r->Value/1048576)."MB\n"; }'
```
Valuta `pm.max_children` PHP-FPM vs RAM, `innodb_buffer_pool_size` vs dimensione dati, `Created_tmp_disk_tables` alto (query che spillano su disco).

---

## Report finale (sempre)
Scrivi `private/storage/perf-audit/report-$TS.md` con:
1. **Contesto**: target, ambiente, data, strumenti usati, set di rotte testate.
2. **Tabella baseline** (per rotta): TTFB cold/warm, total, n° query, tempo query, peso HTML.
3. **Finding prioritizzati** in tabella `Problema | Evidenza (numeri/EXPLAIN) | Impatto | Fix | Sforzo`, ordinati per **impatto/sforzo** (prima i quick-win ad alto impatto: es. cache→Redis, indice mancante, eager loading N+1).
4. **Confronto prod vs staging/locale** se disponibile.
5. **Quick-win** (≤1h) vs **interventi strutturali** (UUID binary, refactor query, CDN).

Chiudi con un riepilogo di 5 righe in chat: dove sta il collo di bottiglia, i 3 interventi a maggior ritorno, e cosa serve all'utente per la fase successiva (es. URL prod, accesso SSH/DB, autorizzazione load test).

> **Pulizia:** se hai aggiunto `DEBUGBAR_ENABLED=true` a un `.env` di test, ricordati di rimuoverlo a fine audit e rilanciare `php artisan config:clear`.
