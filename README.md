# Claude SEO Plugin

A comprehensive SEO toolkit for Claude Code with 23 specialized skills.

## Installation

```bash
claude plugin marketplace add gregor-kuhlmann-seo github:gregor-kuhlmann/claude-seo-plugin
claude plugin install seo@gregor-kuhlmann-seo
```

## Skills Included

| Skill | Description |
|-------|-------------|
| `seo` | Main orchestrator — full-site SEO audit |
| `seo-keyword-research` | Keyword discovery via DataForSEO → Airtable |
| `seo-generate-article-ideas` | SERP analysis → article ideas (TOFU/MOFU/BOFU) |
| `seo-article-research` | Competitor analysis + content outline |
| `seo-article-writer` | Full HTML blog article with FAQ, schema, internal links |
| `seo-article-images` | AI-generated OG/hero images for articles |
| `seo-audit` | Full site crawl with parallel sub-skill delegation |
| `seo-page` | Single-page deep analysis |
| `seo-technical` | Crawlability, indexability, Core Web Vitals, mobile |
| `seo-content` | E-E-A-T, readability, thin content detection |
| `seo-schema` | Schema.org detection, validation, generation |
| `seo-sitemap` | XML sitemap analysis + generation |
| `seo-images` | Alt text, file size, format optimization |
| `seo-geo` | AI Overviews, ChatGPT, Perplexity readiness |
| `seo-google` | Search Console, PageSpeed, CrUX, GA4 |
| `seo-local` | GBP, NAP consistency, citations, reviews |
| `seo-maps` | Geo-grid rank tracking, GBP auditing |
| `seo-hreflang` | International SEO / hreflang validation |
| `seo-programmatic` | Programmatic SEO strategy |
| `seo-competitor-pages` | "X vs Y" comparison page generation |
| `seo-plan` | Strategic SEO roadmap (SaaS, local, ecommerce) |
| `seo-dataforseo` | Live SERP, keyword, backlink data |
| `seo-image-gen` | AI image generation for SEO assets |

## Requirements

- [DataForSEO MCP](https://dataforseo.com) for keyword and SERP data
- [Airtable MCP](https://airtable.com) for content workflow (Article Production table)
- Google APIs (optional): Search Console, PageSpeed, GA4
