# Website SEO Design

## Goal

Improve organic discovery for Oh My Prompt, with Chinese search as the primary
market. The principal Chinese topic is `提示词管理`; supporting search intents
are personal prompt libraries, AI image prompt workflows, team prompt sharing,
and Chrome prompt-management extensions. English pages retain independent,
natural `prompt management` copy and paired international SEO metadata.

## Scope

This work covers every public, indexable marketing and documentation page in
both locales. It does not add features to the extension or web app, change
subscription behavior, or make third-party SEO-console changes.

## Information Architecture

The site will use a small topic cluster rather than repeating one term across
unrelated pages:

| Page type | Primary purpose | Primary topic |
| --- | --- | --- |
| Home (`/`, `/en`) | Brand discovery and product conversion | 提示词管理 / prompt management |
| Topic page (`/prompt-management`, `/en/prompt-management`) | Explain the product category and match broad search intent | 提示词管理工具 / prompt management tool |
| Docs index | Route visitors to task-specific guidance | 提示词管理使用指南 / prompt management guides |
| Existing docs | Match implementation and workflow long-tail searches | Install, insert, image-to-prompt, Prompt Agent, team sharing |
| Subscription and legal/support pages | Support purchase and trust decisions | Existing page-specific intent |

The new topic page links to relevant documentation. The home page, header or
footer, and docs index link to the topic page. This produces crawlable internal
links without adding thin keyword-only pages.

## Content Rules

### Chinese

- The home page title, H1, first descriptive paragraph, and feature section
  naturally identify Oh My Prompt as a `提示词管理` tool.
- The topic page explains the lifecycle supported by the real product:
  organize prompt templates, choose and insert them beside supported AI
  editors, reuse a template library, convert images to prompts, and share
  prompts with a team.
- It includes concise FAQs answering category-level questions such as what a
  prompt-management tool is, where prompts are managed, which AI platforms are
  supported, and whether teams can share prompts.
- Claims remain bounded to documented functionality. No rankings, customer
  counts, compatibility, or pricing promises are invented.

### English

- English content is written for `prompt management` and `prompt library`, not
  a literal translation of Chinese keyword phrasing.
- Each English page remains the `hreflang` counterpart of the Chinese page and
  carries its own canonical URL.

## Indexing Policy

Index only public acquisition, product, documentation, purchase, and trust
pages:

- Home, prompt-management topic page, docs index, each static docs article,
  subscription, contact, privacy, refund, and terms.

Keep user-specific and utility pages out of search results:

- All authentication routes, dashboard, backup, team list/join/detail routes,
  and API routes.

The sitemap contains only indexable static public paths in both locales.
Noindex is emitted in page metadata for the non-public UI routes. `robots.txt`
continues to block API and authentication crawling, while metadata protects
against indexing through discovered links.

## Metadata and Structured Data

All indexable pages use the existing centralized metadata helper to emit:

- concise, intent-specific title and description;
- canonical URL and Chinese/English `hreflang` alternates;
- Open Graph and Twitter cards using the existing image asset;
- production-only absolute URLs.

The root layout will no longer load structured data as an external JSON script.
Instead, server-rendered JSON-LD is emitted inline where it belongs:

- Home: `SoftwareApplication` describing the browser extension and its prompt
  management purpose.
- Prompt-management page: `WebPage`, `BreadcrumbList`, and `FAQPage`.

All schema values are generated from in-repository copy so visible page claims
and structured data cannot drift.

## Implementation Boundaries

- Reuse the existing locale, metadata, page-shell, and Tailwind patterns.
- Add only one paired topic route and its shared localized content component.
- Do not use client-only rendering for crawl-critical copy or JSON-LD.
- Do not add SEO keyword meta tags: modern search engines do not use them for
  ranking.
- Preserve the existing dirty changes in `packages/web-app/.env.example` and
  `packages/web-app/lib/vision-proxy.ts`.

## Verification

Before implementation, add focused failing tests that assert:

1. the sitemap includes the paired topic URLs and excludes authentication;
2. public metadata includes intended robots directives and private metadata is
   `noindex`;
3. new localized content retains matched Chinese and English structure;
4. the home and topic page emit the expected server-rendered JSON-LD payloads.

Then run the focused web-app unit suite covering metadata, sitemap, localized
home content, and the new topic page, followed by `npm run web:build` from the
repository root.

## Success Criteria

- Search engines can discover exactly one canonical Chinese and English version
  of every indexable public page.
- User-specific pages do not appear in the sitemap and declare `noindex`.
- The Chinese home and topic pages visibly and semantically target `提示词管理`
  without keyword stuffing.
- The product category, supported workflows, and answers are represented in
  valid inline JSON-LD.
- Existing locale tests and the production build pass after the change.
