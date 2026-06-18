
# SEO / GEO / AEO Audit — Skill for Claude

Automatically audits any website across three dimensions of modern search visibility:

- **SEO** — Traditional search engine optimization (Google, Bing): title tags, meta descriptions, heading structure, schema markup, internal links, content quality
- **GEO** — Generative Engine Optimization for AI-powered search (Perplexity, ChatGPT Search, Google AI Overviews, Gemini): E-E-A-T signals, entity clarity, factual density, author authority
- **AEO** — Answer Engine Optimization for featured snippets and voice search: FAQ schema, HowTo schema, question-phrased headings, direct answer formatting

---

## How to use

Once installed, just give Claude a URL and ask about search performance:

> "Can you audit burningstickcreative.com for SEO?"
> "Check my site example.com — why isn't it ranking?"
> "Audit this URL for AI search readiness: example.com"
> "Run a full SEO, GEO, and AEO audit on my website"

Claude will ask whether you want a **Quick Audit** (top issues and scores) or a **Full Audit** (comprehensive breakdown), then crawl the site across multiple pages before delivering a structured report with a downloadable Word doc and PDF.

---

## Installation

This tool is distributed as a ZIP archive for use in **Claude's Cowork desktop app**.

1. Download the ZIP from this repository (click **Code → Download ZIP** on GitHub)
2. Open the Claude desktop app or website
3. Navigate to Customize
4. Go to **Skills** and click the **+** icon
5. Upload the ZIP here and it should install the Skill for you

No unzipping required. The skill will be active for that session and any subsequent sessions where you provide the ZIP.

---

## Optional: real performance data (PageSpeed Insights)

By default the audit reads only static HTML and cannot measure real speed. To add Core Web Vitals and Lighthouse performance/SEO/accessibility scores, configure a **free Google PageSpeed Insights API key**:

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create or select a project.
2. **APIs & Services → Library →** search **PageSpeed Insights API → Enable**.
3. **APIs & Services → Credentials → Create credentials → API key**, then copy it.
4. In the plugin root, copy `.pagespeed.key.example` to `.pagespeed.key` and paste your key on a single line.

`.pagespeed.key` is gitignored — never commit it. The free tier allows ~25,000 requests/day, far more than any audit needs.

**Note on connectivity:** the API is reached through the Claude-in-Chrome browser channel, not the sandbox (which has no general outbound network access). A connected browser is required for this feature; if none is available, the audit runs without performance data and says so.

---

## Repository structure

```
SEO-GEO-AEO-Skill/
├── SKILL.md             ← Audit instructions (source of truth)
└── README.md
```

---

## Version history

**1.1.0** — Reliability & data improvements
- Removed all hardcoded absolute paths; output/script locations now resolved at runtime (portable across environments)
- Deterministic, checklist-based scoring (reproducible scores instead of by-impression)
- JavaScript-rendered sites: re-render via browser before judging; never flag content as "missing" from an empty shell
- Optional PageSpeed Insights integration (Core Web Vitals + Lighthouse) via browser channel + local API key

**1.0.0** — Initial release
- Quick and Full audit modes
- Multi-page site crawl (up to 15 pages for Quick, unlimited for Full)
- SEO, GEO, and AEO scoring with priority recommendations matrix
- Downloadable audit report as both Word (.docx) and PDF
