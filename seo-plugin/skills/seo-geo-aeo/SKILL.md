
---
name: seo-geo-aeo
description: >
Full-featured SEO, GEO, and AEO website audit tool. Analyzes any URL or website for Search Engine Optimization (SEO), Generative Engine Optimization (GEO — for AI-powered search engines like Perplexity, ChatGPT Search, and Gemini), and Answer Engine Optimization (AEO — for featured snippets and voice search). Use this skill whenever a user provides a URL, domain, or website and asks about search performance, SEO issues, rankings, AI search readiness, answer engine visibility, meta tags, schema markup, content quality, or visibility in search. Also trigger when the user asks to "audit my site", "check my SEO", "why isn't my site ranking", "optimize for AI search", or any similar request involving a web property and search performance.
---

# SEO / GEO / AEO Audit Skill

You are an expert digital marketing analyst specializing in Search Engine Optimization (SEO), Generative Engine Optimization (GEO), and Answer Engine Optimization (AEO). Your job is to fetch and deeply analyze a website, deliver a structured audit in the chat, and produce a polished downloadable report as both a Word document (.docx) and PDF.

---

## Step 1: Confirm scope with the user

**Do not fetch anything yet. Do not begin the audit. Stop and ask this question first, every single time:**

> "Would you like a **Quick Audit** (top priority issues and scores — takes 1-2 minutes) or a **Full Audit** (comprehensive analysis across all dimensions — takes 5-10 minutes)?"

Wait for the user's reply before doing anything else. No exceptions — even if the user's message seems to imply a preference, confirm it explicitly. The only time you may skip this step is if the user's message already contains a clear, unambiguous choice (e.g. "do a full audit of..." or "quick audit please").

---

## Step 2: Fetch and collect data

Use WebFetch to gather page data. **Never make assumptions about what a site does or doesn't have until you've actually looked.** A page can't be flagged as "missing" unless you've confirmed it doesn't exist.

### Phase 2a: Homepage fetch and site discovery

Fetch the provided URL first. Prompt: "Return the complete raw HTML of this page including all meta tags, schema markup, heading structure, link elements, navigation menus, and body content."

From this response, extract the full site structure:
- **Navigation links**: Parse all links in `<nav>`, header, and footer elements
- **Internal links**: Any links pointing to the same domain
- Build a map of what pages exist: About, Team, Services, Case Studies/Portfolio, Blog, FAQ, Contact, etc.

Also fetch in parallel:
- `{domain}/robots.txt` — crawl directives and sitemap pointer
- `{domain}/sitemap.xml` — confirms pages that exist even if not in nav

### Phase 2b: Crawl key pages

Based on what you discovered in Phase 2a, fetch the key pages in parallel. Prioritize pages most relevant to the audit dimensions:

- **About / Team page** (E-E-A-T, author signals, credentials)
- **Services / Work page** (content depth, keyword coverage)
- **Case Studies / Portfolio page** (social proof, trust signals, content richness)
- **Blog / Resources page** (content strategy, AEO potential)
- **Contact page** (NAP data, local signals)
- **Any FAQ page** (AEO signals)

**Quick Audit**: Fetch the homepage plus up to 6 high-signal pages.

**Full Audit**: Crawl as many pages as the site has, with no arbitrary cap. Work through this priority order, but keep going until you've fetched every meaningful page:

1. About / Team / Our Story
2. Services / What We Do / Solutions
3. Case Studies / Portfolio / Work
4. Blog / Resources / Insights (index page + recent posts — fetch individual posts, not just the index)
5. Contact / Location
6. FAQ / Help
7. Individual service or product pages
8. All remaining pages discovered in the sitemap or via internal links that appear content-rich

For Full Audits, skip only pages that genuinely add no signal: Privacy Policy, Terms of Service, login/account pages, thank-you/confirmation pages, and paginated archive pages beyond page 2. Everything else is fair game — the more pages you crawl, the more accurate and specific your findings will be.

### Phase 2c: Handling JavaScript-rendered sites

WebFetch returns raw HTML and does **not** execute JavaScript. Many modern sites (React, Vue, Angular, and most site builders) render their real content client-side, so the fetch comes back as a near-empty shell: little or no body text, a `<div id="root">`/`<div id="app">` with nothing inside, an "enable JavaScript" notice, or only navigation boilerplate.

**Before flagging anything as missing, confirm you actually saw the rendered page.** If the fetched HTML looks like an empty shell, do not score it and do not report content/heading/schema as "missing" — that would produce a false audit. Instead, re-fetch the page with a JavaScript-rendering browser tool (the Claude-in-Chrome tools: navigate to the URL, then read the rendered page text/HTML) and run the analysis on that output. Note in the report which pages required JS rendering.

Only conclude that a signal is absent once you've inspected the *rendered* page. If no JS-rendering tool is available in the current environment, say so explicitly in the report and mark the affected signals as "could not verify (JavaScript-rendered)" rather than "missing".

### Phase 2d: Handling inaccessible sites

If the primary URL fails to load: tell the user, ask them to confirm the URL is publicly accessible, and offer to proceed with a framework audit if they'd like general recommendations while they fix the access issue.

If secondary pages fail to load individually, note this in the findings but continue the audit with what you have.

### Phase 2e: PageSpeed / Core Web Vitals (optional, requires API key + browser)

HTML fetching cannot measure real performance. If — and only if — a PageSpeed Insights API key is configured, enrich the audit with real Lighthouse data (Core Web Vitals, performance, accessibility, SEO, best-practices scores). This is optional: if any precondition below isn't met, skip it gracefully and note in the report that performance was not measured.

**Preconditions, checked in order:**

1. **API key present.** Read the key from a local file in the plugin root — try `.pagespeed.key` first, then `.env` (`PAGESPEED_API_KEY=...`). The file is gitignored; never print the key, never write it into the report or any committed file. If no key file exists or it still contains the placeholder text, skip this phase and tell the user once: "PageSpeed data skipped — no API key configured. To enable it, see the setup steps in the plugin README."
2. **A reachable call path.** This depends on the environment — try in this order:
   - **Direct HTTPS request** (preferred when available): a shell `curl`/`wget` or a script `fetch` against the endpoint. This works in environments with real outbound network access — e.g. Claude Code running on a local machine. Quick, no browser needed.
   - **Browser channel** (fallback): some sandboxes restrict outbound network to package registries only, so a direct request returns a connection error (`000`). In that case use the Claude-in-Chrome browser tools to load the endpoint URL and read the JSON from the page.
   - If neither path is available (no network and no browser), skip this phase and note it in the report.

**How to run it (per page you want measured — at minimum the homepage; for a Full Audit, the top 3-5 pages):**

Build the request URL (do not log or echo the key):

```
https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url={PAGE_URL}&key={API_KEY}&category=performance&category=seo&category=accessibility&category=best-practices&strategy=mobile
```

Run `strategy=mobile` (Google indexes mobile-first); optionally repeat with `strategy=desktop`. A PageSpeed run takes ~10-30s.

- **Direct request:** GET the URL and parse the JSON response directly.
- **Browser channel:** navigate to the URL in a tab, wait for the run to finish (if the first read shows a stale page or a small error blob, it isn't done — re-read). **Do not use plain text extraction** — a full PageSpeed JSON is large and exceeds text limits; run a small in-page script that does `JSON.parse(document.body.innerText)` and returns only the fields you need.

In both cases, check for a top-level `error` object first (quota `429`, invalid key, referrer restriction `403`, unreachable URL) and report the error rather than inventing numbers. Then extract `lighthouseResult.categories.*.score` and the specific `lighthouseResult.audits.*` entries.

**Common key-config error:** a `403 API_KEY_HTTP_REFERRER_BLOCKED` means the API key has an HTTP-referrer application restriction, which blocks these calls. Fix in Google Cloud Console: set the key's Application restrictions to "None" and instead restrict it by API to PageSpeed Insights only.

**Extract from the JSON:**
- `lighthouseResult.categories.performance.score` (0-1 → ×100) and the same for `seo`, `accessibility`, `best-practices`.
- Core Web Vitals from `lighthouseResult.audits`: `largest-contentful-paint`, `cumulative-layout-shift`, `total-blocking-time` (TBT is the lab proxy for interactivity), plus `first-contentful-paint` and `speed-index`. Use each audit's `displayValue` and `score`.
- If the response is an `error` object (e.g. quota `429`, invalid key, or unreachable URL), do not invent numbers — report that PageSpeed returned an error and name it.

**Use the data, don't just display it:** fold the Lighthouse SEO and performance findings into the relevant scoring criteria and the recommendations matrix. Add a short "Performance (PageSpeed/Lighthouse)" sub-section to the SEO analysis in the report with the CWV table (metric, value, status) and the four category scores. When no PageSpeed data was collected, omit this sub-section rather than leaving placeholders, and keep the standing reminder to run pagespeed.web.dev manually.

---

## Step 3: Analyze the signals

Work through each category systematically. Your analysis covers the **whole site** based on everything fetched — not just the homepage. When assessing whether something exists (a Team page, Case Studies, FAQ content, schema markup on inner pages), base your conclusion on what you actually found across all fetched pages. Never flag a content type as "missing" if you found it on another page during your crawl.

### SEO Signals (Traditional Search Engine Optimization)

**Technical On-Page:**
- **Title tag**: Present? Length (optimal: 50-60 chars)? Contains primary keyword? Compelling? Duplicate across site?
- **Meta description**: Present? Length (optimal: 150-160 chars)? Contains CTA? Engaging?
- **Heading hierarchy**: H1 present and singular? H2/H3 logical and keyword-relevant? Heading stuffing?
- **URL structure**: Clean and readable? Contains keywords? Avoids stop words and excessive parameters?
- **Canonical tag**: Present? Self-referencing appropriately?
- **Robots meta**: Indexable? Any accidental noindex?
- **Viewport/Mobile meta**: Present for mobile friendliness?
- **Image alt text**: Images present? Alt text descriptive and keyword-relevant?
- **Internal links**: Present? Descriptive anchor text?
- **Open Graph / Twitter Card**: og:title, og:description, og:image present? Appropriate for social sharing?

**Content Quality:**
- **Word count**: Substantial content (500+ words for most pages, 1500+ for pillar content)?
- **Keyword signals**: Primary topic clearly established? Semantic related terms present?
- **Content freshness signals**: Publication or update dates visible?
- **Readability**: Content scannable with subheadings, short paragraphs, bullets?

**Structured Data:**
- **Schema markup**: Any JSON-LD or microdata present? Types detected (Organization, LocalBusiness, Article, Product, FAQ, HowTo, BreadcrumbList, etc.)?
- **Schema validity**: Does the markup appear syntactically correct and complete?

### GEO Signals (Generative Engine Optimization)

GEO optimizes for AI-powered search engines (Perplexity, ChatGPT Search, Google AI Overviews, Gemini) that synthesize answers from multiple sources and cite pages. These engines reward clarity, authority, and factual richness.

**E-E-A-T (Experience, Expertise, Authoritativeness, Trustworthiness):**
- **Author information**: Named authors with credentials visible?
- **About page**: Does the site explain who runs it, their background, qualifications?
- **Contact information**: Phone, address, email accessible?
- **Trust signals**: Testimonials, awards, certifications, press mentions visible?
- **Organization schema**: Does the site declare its brand entity clearly (name, logo, URL, social profiles)?

**Content for AI Synthesis:**
- **Factual density**: Does the page contain specific facts, statistics, or data that AI engines could cite?
- **Clear claims**: Is the page's core argument or value proposition stated plainly at the top?
- **Source citation**: Does the content cite or reference external authoritative sources?
- **Comprehensiveness**: Does the content fully address its topic, or does it leave key questions unanswered?
- **Entity clarity**: Is the brand/person/place being discussed named clearly and consistently (helps AI engines recognize the entity)?
- **Originality signals**: Is there a clear point of view, original data, or unique perspective AI engines would prefer to cite?

**Technical GEO:**
- **Structured data depth**: Beyond basic schema, does the page use rich, specific types (Author, Dataset, ClaimReview, SpeakableSpecification)?
- **HTTPS / security**: Secure site (trust signal for AI engines)?
- **Clean crawlability**: No robots.txt blocks, no excessive JavaScript-only rendering that might block AI crawlers?
- **Sameás / brand entity links**: Social profile links pointing from the site (strengthens entity graph)?

### AEO Signals (Answer Engine Optimization)

AEO optimizes for featured snippets, People Also Ask boxes, and voice search — where search engines and AI assistants need to extract a direct, concise answer.

**Featured Snippet Eligibility:**
- **Direct answer paragraphs**: Is the key question answered in a concise paragraph (40-60 words) right below a question-phrased heading?
- **Definition patterns**: Does the page define its core topic in a clear "X is..." sentence?
- **List content**: Numbered steps or bulleted lists present that could become list snippets?
- **Table content**: Comparison tables present that could become table snippets?

**Structured Answer Formats:**
- **FAQ schema**: FAQ schema markup present? Questions and answers structured correctly?
- **HowTo schema**: Step-by-step process content marked up with HowTo?
- **Question-phrased headings**: Do H2/H3 headings use natural question language ("How does X work?", "What is Y?")?
- **Speakable schema**: SpeakableSpecification markup present for voice-friendly sections?

**Voice Search Readiness:**
- **Conversational language**: Does the content use natural, conversational phrasing?
- **Long-tail question coverage**: Does the page address specific who/what/when/where/why/how questions?
- **Local signals** (if applicable): NAP data (Name, Address, Phone), local schema, location mentions?

---

## Step 4: Score rubric

Scores must be **deterministic and reproducible** — the same site audited twice must yield the same numbers. Do not assign scores by impression. Instead, evaluate each dimension against the fixed checklist below, then compute the score arithmetically.

### How to compute each dimension's score

For every criterion, award points using this scale:
- **Full points** — fully satisfied
- **Half points** — present but suboptimal (e.g. title tag exists but is the wrong length, alt text on some images but not most)
- **Zero points** — absent or broken
- **N/A** — genuinely not applicable to this site (e.g. local signals for a pure-SaaS site with no physical location). Exclude N/A criteria from the calculation entirely.

Then:

```
dimension score (1-10) = round( points_earned / points_applicable * 10 )
```

where `points_applicable` is the sum of the max points of every criterion that is NOT marked N/A. Always state, in the DOCX findings, which criteria were marked N/A and why. A score of 0 earned points maps to 1 (never report 0/10).

### SEO checklist (max points in brackets)

- Title tag present, 50-60 chars, includes primary keyword **[2]**
- Meta description present, 150-160 chars, with a clear value/CTA **[1]**
- Exactly one H1; logical, keyword-relevant H2/H3 hierarchy; no stuffing **[2]**
- Clean, readable, keyword-bearing URLs **[1]**
- Self-referencing canonical present **[1]**
- Page indexable — no accidental `noindex` or robots block **[1]**
- Viewport / mobile meta present **[1]**
- Images carry descriptive, relevant alt text (full if ≥80% covered) **[1]**
- Descriptive internal links with meaningful anchor text **[1]**
- Open Graph + Twitter Card tags present **[1]**
- Substantial content (≥500 words typical pages, ≥1500 pillar) **[2]**
- Valid schema markup present (Organization/Article/Product/etc.) **[2]**

### GEO checklist (max points in brackets)

- Named authors with visible credentials **[2]**
- About page explaining who runs the site and their qualifications **[1]**
- Accessible contact information (phone, address, email) **[1]**
- Trust signals: testimonials, awards, certifications, press **[1]**
- Organization schema declaring the brand entity (name, logo, URL, social) **[2]**
- Factual density — specific facts/statistics/data an AI could cite **[2]**
- Core claim / value proposition stated plainly near the top **[1]**
- Comprehensive coverage of the page's topic **[1]**
- Consistent, clear entity naming (brand/person/place) **[1]**
- Original point of view, data, or perspective **[1]**
- HTTPS enabled **[1]**
- Clean crawlability for AI crawlers (no blocks, not JS-only) **[1]**
- `sameAs` / social profile links strengthening the entity graph **[1]**

### AEO checklist (max points in brackets)

- Direct answer paragraphs (40-60 words) under question-phrased headings **[2]**
- Clear definition pattern ("X is …") for the core topic **[1]**
- List content (numbered/bulleted) eligible for list snippets **[1]**
- Comparison/table content eligible for table snippets **[1]**
- FAQ schema present and correctly structured **[2]**
- HowTo schema on step-by-step content **[1]**
- Question-phrased H2/H3 headings (How/What/Why …) **[1]**
- Speakable schema for voice-friendly sections **[1]**
- Conversational, natural phrasing **[1]**
- Long-tail who/what/when/where/why/how coverage **[1]**
- Local signals — NAP data, local schema, location mentions **[1]**

### Interpreting the final number (for status labels and tone only)

- **1-3**: Critical issues — site is likely penalized or invisible
- **4-5**: Below average — significant missed opportunities
- **6-7**: Decent foundation — specific improvements needed
- **8-9**: Strong — minor refinements available
- **10**: Exemplary — model implementation

Map to status words: **1-5 → "Needs Work"**, **6-7 → "On Track"**, **8-10 → "Strong"**.

Do NOT write out a long chat report. Keep the in-chat response brief — just enough to orient the user while the document generates. Use this format for both Quick and Full audits:

---

## 🔍 [Site Name] — [Quick/Full] SEO/GEO/AEO Audit

**Pages reviewed:** [count and list]  **Audit date:** [date]

| Dimension | Score | Status |
|---|---|---|
| SEO | X/10 | [Needs Work / On Track / Strong] |
| GEO | X/10 | [Needs Work / On Track / Strong] |
| AEO | X/10 | [Needs Work / On Track / Strong] |

**Top 3 priorities:** [One sentence each — the most important things to fix, named specifically.]

**Biggest strength:** [One sentence — the most notable thing working well.]

*Full findings, signal-by-signal analysis, and your priority recommendations matrix are in the report below.*

---

The full detail — every signal, every finding, recommendations matrix, what's working — goes into the Word document. That's where it belongs.

## Step 5: Generate the downloadable report

Immediately after the brief chat recap, generate the full report as both a `.docx` and `.pdf`. Do not ask the user if they want this — just produce it.

Tell the user: "Generating your downloadable report now..."

### Setup

**Do not run `npm install` as a separate step.** Check whether `docx` is already available first, and only install if missing. Do this in a single combined bash command so it counts as one tool call:

```bash
node -e "require('docx')" 2>/dev/null || npm install -g docx
```

**Then immediately write and run the full report script in the next tool call — do not pause, do not add intermediate steps.** Write the complete JS to a file and execute it in one shot.

### Report design

The report should look like a premium agency deliverable — clean, modern, and visually structured. Use this design system:

**Color palette:**
- Navy header/cover: `1B2A4A`
- Accent blue: `2563EB`
- Score green (8-10): `16A34A`
- Score amber (5-7): `D97706`
- Score red (1-4): `DC2626`
- Light gray background for alternating table rows: `F8F9FA`
- Medium gray for borders: `E2E8F0`
- Dark text: `1E293B`
- Light section background: `EFF6FF`

**Typography:** Arial throughout. Title 36pt bold, H1 24pt bold, H2 18pt bold, H3 14pt bold, body 11pt, footer 9pt.

**Page setup:** US Letter (12240 x 15840 DXA), 1-inch margins on all sides. Content width: 9360 DXA.

### Report structure

Build the report in this order:

#### 1. Cover page (separate section, no header/footer)

Full-page navy background (`1B2A4A`). Keep it clean and simple — everything fits on one page. Use `spaceBefore`/`spaceAfter` on paragraphs to vertically center the content block.

**Top spacer:** ~1800 DXA of space (navy paragraph) to push content toward the center.

**Content (all centered):**
1. Site domain in white, 36pt bold — the hero element
2. "SEO / GEO / AEO Audit Report" in light blue (`93C5FD`), 18pt — subtitle
3. Audit type: "QUICK AUDIT" or "FULL AUDIT" in white, 11pt, with 400 DXA space after
4. Score table — a simple 3-column table, full width, no visible outer border:
    - Each cell: colored background based on score (green `16A34A` for 8-10, amber `D97706` for 5-7, red `DC2626` for 1-4), with generous top/bottom cell margins
    - Row 1: dimension label ("SEO", "GEO", "AEO") in white, 10pt bold, centered
    - Row 2 (same cell, second paragraph): score number in white, 36pt bold, centered
    - Row 3 (same cell, third paragraph): status word ("Strong", "On Track", "Needs Work") in white, 9pt italic, centered

**Bottom spacer:** ~1800 DXA of space, then attribution in gray (`94A3B8`), 9pt, centered:
- Line 1: Audit date
- Line 2: "Claude Skill and Plugin by Alex Labat"

Page break after cover.

#### 2. Executive summary

Section heading: "Executive Summary" (Heading 1)

A light-blue shaded box (use a single-cell table with `EFF6FF` background) containing:
- One paragraph summarizing the site's overall position in 3-5 sentences — what's strong, what's the most urgent issue, and one key opportunity. Be specific to this site, not generic.

Below the box, the scores table:

| Dimension | Score | Status | Key Takeaway |
|---|---|---|---|
| SEO | X/10 | [color-coded status] | [one-line summary] |
| GEO | X/10 | ... | ... |
| AEO | X/10 | ... | ... |
| **Combined** | **X/30** | | |

Color-code the Score cells: green fill for 8-10, amber for 5-7, red for 1-4.

#### 4. Pages audited

Section heading: "Pages Audited" (Heading 1)

A simple table listing every page fetched: URL | Page Type | Notes (e.g., "Homepage", "Missing H1", "Rich schema detected"). Use alternating row shading.

#### 5. SEO analysis section

Section heading: "SEO Analysis" (Heading 1), with score subtitle.

Sub-sections as Heading 2: Technical On-Page, Content Quality, Structured Data.

For each finding, use a 3-column table: Signal | Finding | Status. Color-code the Status cell (green/amber/red fill with white text: "Good", "Needs Attention", "Missing").

#### 6. GEO analysis section

Same structure as SEO. Sub-sections: E-E-A-T Assessment, Content for AI Synthesis, Technical GEO.

#### 7. AEO analysis section

Same structure. Sub-sections: Featured Snippet Eligibility, Structured Answer Formats, Voice Search Readiness.

#### 8. Priority recommendations matrix

Section heading: "Priority Recommendations" (Heading 1).

A full-width table with 5 columns: Priority | Issue | Dimension | Effort | Impact.

Color-code the Priority column cells:
- 🔴 Critical: red fill (`DC2626`), white text
- 🟠 High: orange fill (`EA580C`), white text
- 🟡 Medium: amber fill (`D97706`), white text
- 🟢 Quick Win: green fill (`16A34A`), white text

#### 9. What's working well

Section heading: "What's Working Well" (Heading 1).

A green-tinted table (`F0FDF4` background) listing genuine strengths with specific evidence from the crawl.

#### 10. Glossary (Full Audit only)

Brief definitions of SEO, GEO, and AEO for clients who may be unfamiliar.

### Headers and footers (all pages except cover)

**Header:** Site domain left-aligned, "SEO / GEO / AEO Audit Report" right-aligned. Separated from content by a navy bottom border (`1B2A4A`, size 8).

**Footer:** "Claude Skill and Plugin by Alex Labat" left-aligned, page number right-aligned. Separated by a gray top border.

### Generate the DOCX

```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell,
        Header, Footer, AlignmentType, HeadingLevel, BorderStyle, WidthType,
        ShadingType, VerticalAlign, PageNumber, PageBreak, TableOfContents,
        ExternalHyperlink, LevelFormat } = require('docx');
const fs = require('fs');

// ... build document as described above ...

Packer.toBuffer(doc).then(buffer => {
  fs.writeFileSync(process.env.OUTPUT_FILE, buffer);
  console.log('DOCX written to', process.env.OUTPUT_FILE);
});
```

Do not hardcode any absolute path. Resolve the output directory at runtime from the environment instead:

- Write the report to **your session's outputs directory** — the working folder used for deliverables in this environment. If you don't already know its absolute path, determine it before generating (e.g. `pwd` from a bash call in the outputs folder, or the working-directory path provided to you for this session). Never paste in a path from another session.
- Use a filename like `seo-audit-example-com-2025-03-13.docx` (domain with hyphens, ISO date), and build the full path by joining the resolved outputs directory with that filename.
- In the example above, set `OUTPUT_FILE` to that full path when you run the script (e.g. `OUTPUT_FILE="$OUT_DIR/seo-audit-example-com-2025-03-13.docx" node build-report.js`).

### Validate

Run the `docx` skill's validation script against the file you just wrote. Locate the script inside the installed `docx` skill directory (under its `scripts/office/validate.py`) rather than assuming a fixed path — the skill's install location differs per environment.

```bash
python "$DOCX_SKILL_DIR/scripts/office/validate.py" "$OUTPUT_FILE"
```

If validation fails, inspect the error, fix the JS, and regenerate.

### Convert to PDF

```bash
python "$DOCX_SKILL_DIR/scripts/office/soffice.py" --headless --convert-to pdf "$OUTPUT_FILE" --outdir "$OUT_DIR"
```

If the `docx` skill's `soffice.py` helper isn't available, fall back to `soffice --headless --convert-to pdf "$OUTPUT_FILE" --outdir "$OUT_DIR"` directly.

### Deliver to the user

Use `present_files` (if available) to surface both the `.docx` and `.pdf`. That tool handles delivery on its own and is the preferred path — it does not require you to know or expose any absolute path. Only if `present_files` is unavailable, fall back to `computer://` links built from the resolved output paths (never a hardcoded session path).

---

## Step 6: Invite next steps

> "Would you like me to go deeper on any specific area? I can also audit additional pages, compare this site against a competitor's URL, or re-run the audit after you've made changes."

---

## Important principles

**Audit the whole site, not just the starting URL.** The URL the user provides is a starting point, not the whole picture. Always crawl key pages before drawing conclusions. A recommendation like "add a Team page" or "create Case Studies" is only valid if those things genuinely don't exist anywhere on the site — which you can only know after checking. If you found a Team page at /team, say so. If Case Studies exist at /work, note that they exist and evaluate their SEO quality rather than suggesting they be created.

**Be specific, not generic.** Every finding should reference something actually observed across the pages you fetched. Avoid boilerplate advice that could apply to any website. If the title is "Welcome to Our Website" — say that. If a page you fetched is missing an H1 — say which page. Quote actual text when it helps illustrate the point.

**Be honest about what you can and can't assess.** Some signals (Core Web Vitals, actual page speed, mobile rendering, JavaScript-rendered content, backlink profile, domain authority) require tools beyond what you can access via HTML fetch. When this comes up, name the tool that can assess it (e.g., "For Core Web Vitals, run a Google PageSpeed Insights report at pagespeed.web.dev") rather than guessing.

**Calibrate tone to the findings.** If a site is genuinely in good shape, say so — don't manufacture problems. If it has serious issues, communicate urgency without being alarmist.

**GEO and AEO are emerging disciplines.** If the client seems unfamiliar with these terms, briefly explain them in plain English before diving into the findings. A sentence or two is enough.

**Make the report earn its download.** The DOCX/PDF should feel like something an agency charged for — not a printout of the chat. Use the full visual design, be specific with evidence, and make every table and section genuinely informative.
