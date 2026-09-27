---
name: seo-audit
description: >
  Use this skill whenever the user asks to run an SEO audit, site audit, page speed audit,
  check Core Web Vitals, check website performance, audit a URL, or "audit my site".
  Also triggers when the user says "run the audit", "check page speed", "check my SEO",
  or pastes any URL and asks how it's performing. This skill spawns multiple parallel
  sub-agents that simultaneously check page speed, meta tags, broken internal links,
  schema markup, orphaned pages, AEO / answer engine readiness, competitor traffic
  and keywords, and keyword clusters, then compiles a full report with programmatic
  SEO opportunities. Powered by Mavek (mavek.ai).
version: 4.0.0
---

# SEO Audit Skill, powered by Mavek

Install and setup: see the README at https://github.com/mavekjeet/claude-seo-audit

---

## What this skill does

Spawns **7 parallel sub-agents** that analyse the target website simultaneously:
- **Agent 1 (Page Speed):** Google Lighthouse scores + Core Web Vitals on every page type (mobile + desktop)
- **Agent 2 (On-Page SEO):** Title tags, meta descriptions, H1s, schema markup, canonical tags
- **Agent 3 (Broken Links):** Crawls every page and checks every internal link for 4xx/5xx errors
- **Agent 4 (Orphaned Pages):** Fetches sitemap, finds pages with zero internal links pointing to them
- **Agent 5 (AEO / AI Readiness):** robots.txt bot access, llms.txt, answer-first content, FAQ/Q&A schema, entity clarity, gated content
- **Agent 6 (Competitors):** DataForSEO traffic estimates + ranked keywords for each competitor vs the client, keyword gap, programmatic URL patterns
- **Agent 7 (Keyword Clusters):** DataForSEO keyword research grouped into the 5-8 clusters the client should focus on

The compiled report also includes a **Programmatic SEO Opportunities** section synthesised from Agents 6 + 7.

Results from all 7 agents are collected and compiled into a single report.

---

## Step 0: Get the Target URL, Competitors & Market

If the user has not provided a URL, ask: *"Which website should I audit?"*

Normalise the URL: ensure it starts with `https://`. Strip trailing slashes.

Set `TARGET_DOMAIN` = the bare domain (e.g. `example.com`).

**Then ask for competitors and market** (before launching anything). Ask BOTH questions outright in chat and WAIT for the user's answers: never assume a market, never default to US:

> "Two quick things before I launch the agents:
> 1. Who are the main competitors? Give me 2-4 competitor domains: or say 'skip' and I'll audit without competitor analysis.
> 2. Which market should I use for keyword and traffic data? (e.g. US, India, UK, UAE...)"

If the user answers only one of the two, ask again for the missing one. Do not proceed to Step 1 until the market is explicitly given.

- Set `COMPETITORS` = list of bare domains (strip `https://`, paths, trailing slashes).
- Set `LOCATION_CODE` from the user's stated market: US 2840 · India 2356 · UK 2826 · UAE 2784 · Saudi Arabia 2682 · Canada 2124 · Australia 2036. For any other market, look up the DataForSEO location code for that country.
- Set `LANGUAGE_CODE` = `en` unless the market implies otherwise.

**Check DataForSEO credentials** (needed for Agents 6 & 7):
```bash
[ -n "$DATAFORSEO_LOGIN" ] && [ -n "$DATAFORSEO_PASSWORD" ] && echo "env ok"
```
If set, pass `DATAFORSEO_LOGIN` and `DATAFORSEO_PASSWORD` to Agents 6 and 7 in their task prompts. If credentials cannot be found anywhere, tell the user competitor + keyword cluster analysis will be skipped, and run only Agents 1-5.

**Understand the business:** from the homepage fetch in Step 1, note in one or two sentences what the client sells, to whom, and in what geography. Pass this `BUSINESS_CONTEXT` to Agents 6 and 7: it drives seed keywords and programmatic SEO relevance.

---

## Step 1: Discover Page Types

Print:
```
🔍 Mavek SEO Audit starting on [domain]...
   Discovering page types...
```

Use `WebFetch` on the homepage with this prompt:
> "List every unique page type linked from this homepage with a representative URL for each. Include: homepage, pricing, a single blog post (not the index), docs/guide, features/product page, comparison pages, about, contact, legal pages, glossary. Return as a plain list of Label: URL pairs only."

Deduplicate. Cap at **10 pages**. Always include the homepage.

Priority order:
1. Homepage
2. Pricing
3. Individual blog post
4. Docs / Guide
5. Features / Product
6. Comparison page
7. About
8. Contact
9. Legal (Privacy Policy or Terms)
10. Any other unique template

Once you have the page list, print:
```
   Found [N] page types. Launching 7 agents simultaneously...
```

---

## Step 2: Launch 7 Parallel Sub-Agents

Use the `Agent` tool to spawn all 7 agents **at the same time in a single message** (not sequentially). Pass the full page list and TARGET_DOMAIN to each agent. Additionally pass COMPETITORS, LOCATION_CODE, LANGUAGE_CODE, BUSINESS_CONTEXT, and the DataForSEO credentials to Agents 6 and 7. If the user skipped competitors, skip Agent 6. If DataForSEO credentials are missing, skip Agents 6 and 7.

---

### Agent 1: Page Speed

**Task for this agent:**

For each page in the list, run TWO curl calls (mobile and desktop) against the Google PageSpeed Insights API.

First check for an API key:
```bash
echo $PAGESPEED_API_KEY
```

Then for each page run:
```bash
curl -s "https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=PAGE_URL&strategy=STRATEGY&category=performance&category=seo&category=accessibility&category=best-practices[&key=$PAGESPEED_API_KEY]" | python3 -c "
import json, sys
raw = sys.stdin.buffer.read()
try:
    data = json.loads(raw)
except Exception as e:
    print('ERROR:', e); sys.exit(0)
lh = data.get('lighthouseResult', {})
cats = lh.get('categories', {})
audits = lh.get('audits', {})
p  = round((cats.get('performance',    {}).get('score') or 0) * 100)
s  = round((cats.get('seo',            {}).get('score') or 0) * 100)
ac = round((cats.get('accessibility',  {}).get('score') or 0) * 100)
b  = round((cats.get('best-practices', {}).get('score') or 0) * 100)
lcp     = audits.get('largest-contentful-paint',  {}).get('displayValue', 'N/A')
lcp_raw = audits.get('largest-contentful-paint',  {}).get('numericValue', 0) or 0
cls_    = audits.get('cumulative-layout-shift',   {}).get('displayValue', 'N/A')
cls_raw = audits.get('cumulative-layout-shift',   {}).get('numericValue', 0) or 0
inp     = audits.get('interaction-to-next-paint', {}).get('displayValue', 'N/A')
fcp     = audits.get('first-contentful-paint',    {}).get('displayValue', 'N/A')
tbt     = audits.get('total-blocking-time',       {}).get('displayValue', 'N/A')
opps = []
for aid, av in audits.items():
    d = av.get('details', {})
    if d.get('type') == 'opportunity' and (av.get('score') or 1) < 1:
        ms = d.get('overallSavingsMs', 0) or 0
        opps.append((ms, av.get('title', '')))
opps.sort(reverse=True)
print(f'SCORES  Perf:{p}  SEO:{s}  A11y:{ac}  BP:{b}')
print(f'CWV     LCP:{lcp}({lcp_raw:.0f}ms)  CLS:{cls_}({cls_raw:.3f})  INP:{inp}  FCP:{fcp}  TBT:{tbt}')
for ms, t in opps[:3]:
    print(f'FIX     {t}  saves:{ms:.0f}ms')
"
```

Omit `&key=...` if `PAGESPEED_API_KEY` is empty. If a call returns error or empty, mark that page as `⚠️ PSI unavailable`.

Print progress as each page completes:
```
🔄 Mavek PSI · [PAGE TYPE] mobile ✓  desktop ✓
```

Return all scores, CWV values, and top fixes per page.

---

### Agent 2: On-Page SEO

**Task for this agent:**

For each page in the list, use `WebFetch` with this prompt:
> "Extract exactly: (1) full title tag text, (2) meta description content, (3) every H1 tag on the page and its exact text (list all of them), (4) JSON-LD or schema.org structured data present (yes/no and type(s)), (5) canonical URL if present, (6) approximate count of internal links to the same domain."

Flag:
- **Title:** missing ❌ | >60 chars ⚠️ | <30 chars ⚠️
- **Meta:** missing ❌ | >160 chars ⚠️ | <100 chars ⚠️
- **H1:** missing ❌ | count >1 ⚠️ (list all found) | doesn't match page topic ⚠️
- **Schema:** type(s) present ✅ | missing ❌
- **Canonical:** present ✅ | missing ❌
- **Internal links:** <3 ⚠️

Print progress per page:
```
🔄 On-Page · [PAGE TYPE] ✓
```

Return all findings per page.

---

### Agent 3: Broken Internal Links

**Task for this agent:**

For each page in the list:

1. Use `WebFetch` with this prompt to extract all links:
   > "List every internal link href on this page that points to the same domain. Return as a plain list of URLs, one per line, no duplicates."

2. For each internal link found, check its HTTP status using bash:
```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 10 --location "LINK_URL"
```

3. Flag any URL returning 4xx or 5xx as broken.

Cap at **30 links per page** to keep runtime reasonable. If a page has more, check the first 30.

Print progress:
```
🔄 Links · [PAGE TYPE]: checking [N] links...
```

Return: per-page list of broken links with their status code, and a total broken link count across the site.

If no broken links found, return: `✅ No broken internal links found across [N] pages checked`

---

### Agent 4: Orphaned Pages

**Task for this agent:**

1. Fetch the sitemap:
```bash
curl -s "https://TARGET_DOMAIN/sitemap.xml"
```
If that fails, try `https://TARGET_DOMAIN/sitemap_index.xml` then `https://TARGET_DOMAIN/robots.txt` (look for Sitemap: line).

2. Parse all URLs from the sitemap:
```bash
curl -s "https://TARGET_DOMAIN/sitemap.xml" | python3 -c "
import sys, re
content = sys.stdin.read()
urls = re.findall(r'<loc>(.*?)</loc>', content)
for u in urls:
    print(u.strip())
" | head -50
```
Cap at 50 URLs.

3. For each page in the **audit page list** (from Step 1), check if any other page in the sitemap links to it. Use `WebFetch` on a sample of 5 sitemap pages with this prompt:
   > "List every internal link href on this page. Return as a plain URL list."

4. Build a basic inbound link count per page. Any page from the audit list with **0 inbound links** from the crawled sample = potentially orphaned.

Print progress:
```
🔄 Orphans · Fetched sitemap ([N] URLs) · Checking inbound links...
```

Return: list of potentially orphaned pages with their inbound link count. Note that this is a sampled check, not a full crawl.

---

### Agent 5: AEO / AI Readiness

**Task for this agent:**

Audit the site from an **Answer Engine Optimization (AEO)** standpoint: can ChatGPT, Perplexity, Claude, and Google AI Overviews crawl, extract, and cite this site? Run all of these checks:

**1. robots.txt: AI bot access**
```bash
curl -s "https://TARGET_DOMAIN/robots.txt"
```
Check if any of these are blocked: `GPTBot`, `PerplexityBot`, `ClaudeBot`, `anthropic-ai`, `CCBot`, `Google-Extended`.
Flag any that are disallowed.

Also check if the output contains `# BEGIN Cloudflare Managed content`. If present, Cloudflare is auto-injecting AI bot blocks. Flag this separately:
- ❌ **Cloudflare AI block active**: Cloudflare is automatically blocking AI crawlers via its "AI Crawl Control" feature. Disable it: Cloudflare Dashboard → [domain] → AI → Robots.txt → toggle off "Cloudflare managed".

**2. llms.txt**
```bash
curl -s -o /dev/null -w "%{http_code}" "https://TARGET_DOMAIN/llms.txt"
```
200 = exists | 404 = missing. Informational only: no major AI engine has confirmed it reads llms.txt, so never flag a missing file as an issue.

**3. Per-page answer extractability**
For each page, use `WebFetch` with this prompt:
> "Assess this page for answer engine optimization. Report: (1) Does the page contain a clear, direct answer to a specific question an AI assistant like ChatGPT or Perplexity could extract and cite? Yes/no, and if yes, the exact question. (2) Is the answer in the first 2-3 sentences under the relevant heading, or buried mid-page? (3) Are any H2/H3 headings phrased as questions people actually ask? List them. (4) Is there an FAQ section? (5) Are facts stated in extractable form (short definitions, lists, tables, stats with sources) or only in long marketing prose?"

**4. AEO schema check**
For each page, check the JSON-LD structured data (reuse the WebFetch or fetch raw HTML with curl) for answer-engine-relevant types: `FAQPage`, `HowTo`, `QAPage`, `Article` (with author + dates), `Organization`/`Person` with `sameAs` links. Flag:
- FAQ content on the page but no `FAQPage` schema ⚠️
- No `Organization` schema with `sameAs` (weak entity signal for AI engines) ⚠️
- Articles without visible published/updated dates ⚠️

**5. Entity clarity check**
On the homepage and about page: is it unambiguous in plain text what the company is, what it does, and who it's for (the kind of sentence an LLM would quote as "X is a…")? If an AI engine had to describe this business in one sentence from the homepage alone, could it? Yes/no + the sentence it would likely extract.

**6. Gated content check**
For pricing and features pages: note if real information is hidden behind "contact us" or a login wall.

**7. Statistics check**
For homepage and any blog post: note if the page cites any statistics with sources: original data and cited stats are what AI engines quote.

Print progress:
```
🔄 AEO · robots.txt ✓ · llms.txt ✓ · checking [N] pages...
```

Return all findings.

---

### Agent 6: Competitor Traffic & Keywords

**Task for this agent:** (skip if user provided no competitors, or no DataForSEO credentials)

You are given: TARGET_DOMAIN, COMPETITORS (list of domains), LOCATION_CODE, LANGUAGE_CODE, and DataForSEO credentials (`LOGIN` / `PASSWORD`).

**1. Traffic & visibility overview**: for TARGET_DOMAIN and each competitor:
```bash
curl -s -u "LOGIN:PASSWORD" -H "Content-Type: application/json" \
  -d '[{"target":"DOMAIN","location_code":LOCATION_CODE,"language_code":"LANGUAGE_CODE"}]' \
  "https://api.dataforseo.com/v3/dataforseo_labs/google/domain_rank_overview/live"
```
Extract per domain from `metrics.organic`: `etv` (estimated monthly traffic value), `count` (total ranking keywords), `pos_1`, `pos_2_3`, `pos_4_10` (keywords in top positions).

**2. Top keywords per competitor**: for each competitor (and TARGET_DOMAIN):
```bash
curl -s -u "LOGIN:PASSWORD" -H "Content-Type: application/json" \
  -d '[{"target":"DOMAIN","location_code":LOCATION_CODE,"language_code":"LANGUAGE_CODE","limit":100,"order_by":["keyword_data.keyword_info.search_volume,desc"],"filters":["keyword_data.keyword_info.search_volume",">",50]}]' \
  "https://api.dataforseo.com/v3/dataforseo_labs/google/ranked_keywords/live"
```
For each domain keep: keyword, search volume, position, URL ranking.

**3. Keyword gap**: compare the keyword sets and list the top 15-20 keywords (by volume) where a competitor ranks in the top 10 but TARGET_DOMAIN either doesn't rank at all or ranks below position 20. These are the client's biggest misses.

**4. Programmatic pattern scan**: fetch each competitor's sitemap (`/sitemap.xml`, fall back to robots.txt Sitemap: line). Look for repeated templated URL patterns at scale: `/vs/`, `/alternatives/`, `/integrations/`, `/templates/`, location pages (`/city/`), glossary/definition pages, `/for-[industry]/` pages. Note which competitors run programmatic plays, the pattern, and roughly how many URLs.

Print progress:
```
🔄 Competitors · [domain] overview ✓ keywords ✓
```

Return: per-domain traffic table data, top keywords per domain, the keyword gap list, and any programmatic patterns found. Parse only what you need from the JSON: don't dump raw API responses.

---

### Agent 7: Keyword Cluster Research

**Task for this agent:** (skip if no DataForSEO credentials)

You are given: TARGET_DOMAIN, BUSINESS_CONTEXT, LOCATION_CODE, LANGUAGE_CODE, and DataForSEO credentials.

**1. What the site already ranks for:**
```bash
curl -s -u "LOGIN:PASSWORD" -H "Content-Type: application/json" \
  -d '[{"target":"TARGET_DOMAIN","location_code":LOCATION_CODE,"language_code":"LANGUAGE_CODE","limit":200,"order_by":["keyword_data.keyword_info.search_volume,desc"]}]' \
  "https://api.dataforseo.com/v3/dataforseo_labs/google/ranked_keywords/live"
```

**2. Expand the space**: from BUSINESS_CONTEXT, pick 3-5 seed keywords describing the core product/service. Seed rules: one seed per CONCEPT, never suffix/synonym variants ("3pl" alone surfaces "3pl company", "3pl services", "3pl provider" in the expansion, so variant seeds waste rows); and NO brand names as seeds (own-brand queries come from GSC, competitor-brand queries come from the competitor ranked-keywords pull). Then:
```bash
curl -s -u "LOGIN:PASSWORD" -H "Content-Type: application/json" \
  -d '[{"keywords":["seed1","seed2","seed3"],"location_code":LOCATION_CODE,"language_code":"LANGUAGE_CODE","limit":200,"order_by":["keyword_info.search_volume,desc"],"filters":["keyword_info.search_volume",">",30]}]' \
  "https://api.dataforseo.com/v3/dataforseo_labs/google/keyword_ideas/live"
```
Extract per keyword: search volume, keyword difficulty (`keyword_properties.keyword_difficulty`), CPC, and search intent if present.

**3. Cluster**: group the combined keyword set into **5-8 thematic clusters** by shared topic/intent (e.g. "comparison queries", "[category] + pricing", "how-to/setup", "[use case] queries"). For each cluster report:
- Cluster name + intent (transactional / commercial / informational)
- 5-8 representative keywords with volume + difficulty
- Total addressable monthly volume
- Client's current coverage: ranking already / thin content / no page at all
- Priority: **P1** (high volume × buying intent × weak current coverage) → **P3**

Print progress:
```
🔄 Keywords · ranked ✓ · ideas ✓ · clustering [N] keywords...
```

Return the clusters ordered by priority. Don't dump raw keyword lists: clusters with representative keywords only.

---

## Step 3: Collect Agent Results

Once all agents complete, collect their outputs. Then print:
```
✅ All agents complete. Compiling report...
```

---

## Step 4: Build the Report

Open with this header:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍  SEO AUDIT: [DOMAIN]
    Powered by Mavek · mavek.ai/tools/page-speed-insights
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Audited: [DATE] · Pages: [N] · Engine: Google Lighthouse v5 + DataForSEO
Agents run: Page Speed · On-Page SEO · Broken Links · Orphaned Pages · AEO · Competitors · Keyword Clusters
```

(List only the agents that actually ran.)

Then output these sections:

---

### 📊 SCORE SUMMARY

| Page | Perf (M) | Perf (D) | SEO | A11y | LCP | CLS | INP |
|------|----------|----------|-----|------|-----|-----|-----|

Verdicts:
- ✅ good: Perf/SEO/A11y ≥ 90 · LCP ≤ 2.5s · CLS ≤ 0.1 · INP ≤ 200ms
- ⚠️ needs work: 50-89 · LCP 2.5-4s · CLS 0.1-0.25 · INP 200-500ms
- ❌ poor: <50 · LCP >4s · CLS >0.25 · INP >500ms

---

### 📄 PER-PAGE BREAKDOWN

For each page:

**[PAGE TYPE]: [URL]**
- Performance: Mobile [score] [emoji] / Desktop [score] [emoji]
- SEO: [score] | A11y: [score] | BP: [score]
- LCP: [value] [emoji] | CLS: [value] [emoji] | INP: [value] [emoji] | FCP: [value] | TBT: [value]
- Title: "[text]" · [✅ / ⚠️ reason / ❌ missing]
- Meta: "[truncated]" · [✅ / ⚠️ reason / ❌ missing]
- H1: [count] found · [list them if >1] · [✅ single / ⚠️ multiple / ❌ missing]
- Schema: [type(s) ✅ / ❌ missing]
- Canonical: [✅ / ❌]
- Internal links: [count] | Broken: [count or ✅ none]
- Orphaned: [✅ has inbound links / ⚠️ no inbound links found in sample]
- AI readiness: [✅ answers "[question]" / ❌ no clear answer]
- Top fix: [plain English one-liner from PSI or link check]

---

### 🚨 CRITICAL ISSUES

Pages with: Perf <50 mobile · LCP >4s · CLS >0.25 · SEO <80 · broken links · missing title or H1

Format: `[Page] · [Issue] · [Fix in one sentence]`

---

### 🔗 BROKEN LINKS

If any broken links found:
```
[Page] · [Broken URL] · HTTP [status code]
```

If none: `✅ No broken internal links detected across [N] pages`

---

### 👻 ORPHANED PAGES

```
[URL]: 0 inbound links found in sample
```
Note: *"Based on a [N]-page sample. Run a full crawl with Screaming Frog for complete coverage."*

If none found: `✅ No orphaned pages detected in sample`

---

### 🏆 QUICK WINS

Top 3-5 fixes by impact:
- **[Page]** · [What] · [One-line fix]

---

### 🤖 AEO / AI SEARCH READINESS

Per-page table + site-wide findings (robots.txt, llms.txt, gated content, stats).

Include a dedicated line for Cloudflare AI block status:
- If `# BEGIN Cloudflare Managed content` was found: `❌ Cloudflare is auto-blocking AI crawlers (GPTBot, ClaudeBot, Google-Extended, etc.): disable in Cloudflare Dashboard → AI → Robots.txt`
- If not found: `✅ No Cloudflare AI block detected`

Then the AEO-specific findings:
- Answer extractability per page (question answered ✅ / answer buried ⚠️ / no clear answer ❌)
- Question-format headings + FAQ presence
- Schema gaps (FAQPage / HowTo / Organization + sameAs / article dates)
- Entity clarity verdict: the one sentence an AI engine would use to describe the business, and whether that's accurate

2-3 sentence summary + top 3 AEO fixes ordered by impact.

---

### ⚔️ COMPETITOR LANDSCAPE

(Only if Agent 6 ran.)

| Domain | Est. monthly traffic | Ranking keywords | Top 3 positions |
|--------|---------------------|------------------|-----------------|

Client row first, then competitors. One sentence on where the client stands.

**Keyword gap: where competitors win and the client doesn't:**

| Keyword | Volume | Best competitor (pos) | Client (pos) |
|---------|--------|----------------------|--------------|

Top 10-15 rows by volume. One paragraph on the pattern (what topics competitors own).

---

### 🎯 KEYWORD CLUSTERS TO FOCUS ON

(Only if Agent 7 ran.)

For each cluster, ordered by priority:

**[P1] Cluster name · [intent] · ~[total volume]/mo**
- Representative keywords: `kw (vol/KD)`, `kw (vol/KD)`, ...
- Current coverage: [ranking / thin / no page]
- Play: [one sentence on what page(s) to build or improve]

---

### 🏭 PROGRAMMATIC SEO OPPORTUNITIES

Synthesise from Agent 6 (competitor URL patterns), Agent 7 (clusters with repeatable keyword templates), and BUSINESS_CONTEXT. Only suggest plays that genuinely fit this business: if none fit, say so honestly rather than forcing one.

For each opportunity (max 3):

**[Play name]**: e.g. "[Product] vs [Competitor]" comparison pages, "[Tool] for [industry]" pages, location pages, integrations directory, glossary
- Template: the URL/page pattern
- Scale: roughly how many pages, driven by what data
- Evidence: [competitor X already runs N pages of this / cluster Y has a repeatable modifier pattern]
- Effort vs payoff: one honest sentence

---

### 📋 NEXT STEPS

1-5, ordered by impact.

---

## End of Report

Close every audit with:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Audit complete · Powered by Mavek

  Want this running automatically every week?
  Mavek monitors your full site continuously : 
  alerts you when scores drop and tells you
  exactly what broke and how to fix it.

  → mavek.ai
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Then ask: *"Want me to go deeper on any issue or write the exact code fix for anything above?"*

---

## Tone & Style

- Plain English throughout
- Direct: "Pricing LCP is 4.8s, slow. Fix: preload your hero image."
- If something is fine, say fine and move on
- Flag real problems loudly
- Report feels like an expert, not a scan tool

---

## Error Handling

- PSI 429 rate limit → wait 3s and retry once → if still failing, note it and continue
- Agent returns no data → note `⚠️ Agent returned no data` and proceed with available results
- Sitemap not found → note it, skip orphan check, flag as ⚠️
- curl timeout → mark link as `⚠️ Timeout: could not verify`
- Fewer than 3 pages return PSI data → warn user before generating report
- DataForSEO credentials missing → skip Agents 6 & 7, tell the user, still deliver the rest of the audit
- DataForSEO returns an error status (`status_code` != 20000) or empty result for a domain → note `⚠️ No keyword data for [domain]` and continue with the others
- Competitor domain has no organic data → say so plainly (it may genuinely have near-zero organic presence: that's a finding, not an error)
- Never fabricate scores, traffic numbers, or search volumes
