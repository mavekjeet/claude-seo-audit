# claude-seo-audit

Type "audit example.com" in Claude Code and get a full SEO and AI search audit back in one report.

It's a Claude Code skill that spins up 7 agents in parallel, each checking one part of the site, then stitches everything into a single report with the fixes ranked by impact. I've been doing SEO for 10 years and this is the audit I actually run on client sites before a first call.

## What it checks

| Agent | What it does | Needs |
|---|---|---|
| Page speed | Lighthouse scores + Core Web Vitals (LCP, CLS, INP) on up to 10 page types, mobile and desktop | nothing (API key optional) |
| On-page | titles, meta descriptions, H1s, schema, canonicals, internal link counts | nothing |
| Broken links | crawls internal links on every page and flags 4xx/5xx | nothing |
| Orphan pages | reads your sitemap and finds pages nothing links to | nothing |
| AEO / AI search | can ChatGPT, Perplexity, Claude and AI Overviews crawl and quote you? AI bot blocks in robots.txt, Cloudflare's silent AI block, answer-first content, FAQ/Organization schema, entity clarity | nothing |
| Competitors | traffic, ranking keywords, keyword gap vs 2-4 competitors, their programmatic SEO plays | DataForSEO |
| Keyword clusters | 5-8 keyword clusters you should own, prioritised by volume, intent and your current coverage | DataForSEO |

The first 5 agents work with zero setup. The last 2 switch on when you add DataForSEO credentials.

## What you get

1. A full text report in Claude Code: scores, critical issues, quick wins, AEO findings, competitor gap, keyword clusters, next steps.
2. Up to 7 visual cards (PNG) built from your real numbers: orphan pages, keyword clusters, AI readiness, page speed, on-page issues, broken links, competitor traffic. Ready to post or drop in a deck (needs Chrome for the PNGs, otherwise open `cards.html`).

## Install

As a plugin (inside Claude Code):

```
/plugin marketplace add mavekjeet/claude-seo-audit
/plugin install seo-audit@claude-seo-audit
```

Or just copy the skill files:

```bash
mkdir -p ~/.claude/skills/seo-audit && curl -sL https://raw.githubusercontent.com/mavekjeet/claude-seo-audit/main/skills/seo-audit/SKILL.md -o ~/.claude/skills/seo-audit/SKILL.md && curl -sL https://raw.githubusercontent.com/mavekjeet/claude-seo-audit/main/skills/seo-audit/cards-template.html -o ~/.claude/skills/seo-audit/cards-template.html
```

## Use

```
audit https://example.com
```

It asks for your competitors and target market (US, India, UK, UAE...), then runs. A full audit takes a few minutes.

## Optional setup

**PageSpeed API key** (avoids Google rate limits on bigger sites). Get one free from Google Cloud, then:

```bash
export PAGESPEED_API_KEY=your_key
```

**DataForSEO** (turns on competitor and keyword cluster agents). Pay-as-you-go, an audit costs cents:

```bash
export DATAFORSEO_LOGIN=your_login
export DATAFORSEO_PASSWORD=your_password
```

Put these in your shell profile or in `env` in `~/.claude/settings.json`.

## What you get

- score table for every page type
- per-page breakdown (speed, meta, H1, schema, links, AI readiness, top fix)
- critical issues, broken links, orphan pages
- quick wins
- AI search readiness, including the one sentence an AI engine would use to describe your business
- competitor table + keyword gap
- keyword clusters with P1-P3 priority
- programmatic SEO opportunities (only if they genuinely fit)
- next steps

It never makes up numbers. If an API returns nothing, the report says so.

## Contributing

Issues and PRs welcome. If a check gave you a wrong result on a real site, open an issue with the URL.

## License

MIT. Built by [Jeet](https://instagram.com/iojeet), powered by [Mavek](https://mavek.ai), which runs this kind of audit on your site every week and tells you when something breaks.
