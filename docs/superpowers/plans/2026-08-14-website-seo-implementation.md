# Website SEO Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Improve organic discovery for prompt management while preventing account and user-specific pages from being indexed.

**Architecture:** Extend the centralized metadata helper with an explicit indexability option. Add one shared, server-rendered bilingual prompt-management topic page and connect it to the home page, footer, and docs index. Keep content and JSON-LD in the same localized source module.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind CSS, Vitest, Playwright.

---

## File Structure

- `packages/web-app/lib/i18n/metadata.ts`: canonical, hreflang, Open Graph, and robots metadata.
- `packages/web-app/app/sitemap.ts`: static indexable URLs only.
- `packages/web-app/app/{auth,dashboard,backup,team}/**/page.tsx`: private-page `noindex` declarations.
- `packages/web-app/lib/i18n/messages/prompt-management.ts`: paired copy and schema data.
- `packages/web-app/components/pages/PromptManagementPage.tsx`: topic page and inline JSON-LD.
- `packages/web-app/app/prompt-management/page.tsx` and `packages/web-app/app/en/prompt-management/page.tsx`: paired public routes.
- `packages/web-app/lib/i18n/messages/home.ts`, `components/pages/HomePage.tsx`, `components/layout/Footer.tsx`, and `components/pages/DocsIndexPage.tsx`: public semantic copy and internal links.

### Task 1: Create failing SEO contract tests

**Files:**
- Modify: `packages/web-app/lib/i18n/metadata.test.ts`
- Modify: `packages/web-app/app/sitemap.test.ts`
- Create: `packages/web-app/lib/i18n/messages/prompt-management.test.ts`

- [ ] **Step 1: Add metadata tests**

Add expectations for these exact calls:

```ts
expect(localizedMetadata({ locale: 'zh', pathname: '/', title: '首页', description: '提示词管理' }).robots)
  .toEqual({ index: true, follow: true })
expect(localizedMetadata({ locale: 'zh', pathname: '/dashboard', title: '仪表盘', description: '用户页面', indexable: false }).robots)
  .toEqual({ index: false, follow: false })
```

- [ ] **Step 2: Add sitemap and topic-page contract tests**

Require `/prompt-management` and `/en/prompt-management` in the sitemap; require no authentication route. Create a topic test that checks paired copy structure, Chinese H1 containing `提示词管理`, and the topic component source containing `application/ld+json`, `FAQPage`, and `BreadcrumbList`.

- [ ] **Step 3: Verify red**

Run `npx vitest run lib/i18n/metadata.test.ts app/sitemap.test.ts lib/i18n/messages/prompt-management.test.ts` from `packages/web-app`.

Expected: FAIL because the indexability API, topic page, and sitemap behavior are absent.

- [ ] **Step 4: Commit test baseline**

Run `git add packages/web-app/lib/i18n/metadata.test.ts packages/web-app/app/sitemap.test.ts packages/web-app/lib/i18n/messages/prompt-management.test.ts && git commit -m "test: define website seo contracts"`.

### Task 2: Implement indexability governance

**Files:**
- Modify: `packages/web-app/lib/i18n/metadata.ts`
- Modify: `packages/web-app/app/sitemap.ts`
- Modify: `packages/web-app/app/auth/**/page.tsx`, `packages/web-app/app/en/auth/**/page.tsx`
- Modify: `packages/web-app/app/{dashboard,backup,team}/**/page.tsx`, `packages/web-app/app/en/{dashboard,backup,team}/**/page.tsx`

- [ ] **Step 1: Add the metadata option**

Extend `LocalizedMetadataOptions` with `indexable?: boolean`. Default it to `true` and return `robots: indexable ? { index: true, follow: true } : { index: false, follow: false }`.

- [ ] **Step 2: Mark private pages**

Add `indexable: false` to all existing auth, dashboard, backup, team list/join/detail metadata calls. Do not alter their titles, descriptions, route behavior, or UI.

- [ ] **Step 3: Restrict the sitemap**

Remove `'/auth/login'` and add `'/prompt-management'` to `publicSitemapPaths`. Retain public docs, subscription, contact, privacy, refund, and terms.

- [ ] **Step 4: Verify green and commit**

Run `npx vitest run lib/i18n/metadata.test.ts app/sitemap.test.ts`; expect PASS. Then run `git add packages/web-app/lib/i18n/metadata.ts packages/web-app/app/sitemap.ts packages/web-app/app && git commit -m "feat: govern public page indexing"`, staging only the planned route files and no pre-existing dirty files.

### Task 3: Build the topic page and inline schema

**Files:**
- Create: `packages/web-app/lib/i18n/messages/prompt-management.ts`
- Create: `packages/web-app/components/pages/PromptManagementPage.tsx`
- Create: `packages/web-app/app/prompt-management/page.tsx`
- Create: `packages/web-app/app/en/prompt-management/page.tsx`
- Modify: `packages/web-app/app/layout.tsx`
- Delete: `packages/web-app/public/scripts/schema-org.json`

- [ ] **Step 1: Define localized content**

Export paired metadata, headings, introductory copy, four workflow entries, FAQs, and docs link labels. Limit claims to existing functionality: organizing prompts, one-click insertion beside supported AI editors, resource library, image-to-prompt, Prompt Agent, and team sharing.

- [ ] **Step 2: Build a server-rendered page**

Render `Header`, a semantic main with one H1, workflow H2 sections, FAQ H2 section, localized documentation links using `localizedHref`, and `Footer`. Inline a JSON-LD graph that contains `WebPage`, `BreadcrumbList`, and `FAQPage` from the same copy object.

- [ ] **Step 3: Add paired thin routes**

Each route exports `localizedMetadata({ locale, pathname, ...promptManagementMessages[locale].metadata })` and renders `PromptManagementPage` for its locale.

- [ ] **Step 4: Replace external homepage JSON-LD**

Replace the root-layout external JSON script with an inline `SoftwareApplication` script using `JSON.stringify`. It must identify Oh My Prompt as a browser extension for prompt management, preserve current supported browsers and download URL, and not use client-only generation. Delete the JSON file after verifying it has no references.

- [ ] **Step 5: Verify green and commit**

Run `npx vitest run lib/i18n/messages/prompt-management.test.ts`; expect PASS. Then run `git add packages/web-app/lib/i18n/messages/prompt-management.ts packages/web-app/lib/i18n/messages/prompt-management.test.ts packages/web-app/components/pages/PromptManagementPage.tsx packages/web-app/app/prompt-management/page.tsx packages/web-app/app/en/prompt-management/page.tsx packages/web-app/app/layout.tsx packages/web-app/public/scripts/schema-org.json && git commit -m "feat: add prompt management topic pages"`.

### Task 4: Improve public-page copy and internal linking

**Files:**
- Modify: `packages/web-app/lib/i18n/messages/home.ts`
- Modify: `packages/web-app/components/pages/HomePage.tsx`
- Modify: `packages/web-app/components/layout/Footer.tsx`
- Modify: `packages/web-app/components/pages/DocsIndexPage.tsx`
- Modify: `packages/web-app/app/docs/page.tsx`
- Modify: `packages/web-app/app/en/docs/page.tsx`
- Modify: `packages/web-app/app/subscription/metadata.ts`
- Modify: `packages/web-app/lib/i18n/messages/home.test.ts`

- [ ] **Step 1: Add failing semantic-copy tests**

Assert that Chinese homepage metadata and visible hero copy contain `提示词管理`, English equivalent content uses `prompt management`, and each locale exposes a localized topic-page link through shared home/footer code.

- [ ] **Step 2: Verify red**

Run `npx vitest run lib/i18n/messages/home.test.ts`; expect FAIL because current copy and links do not meet those assertions.

- [ ] **Step 3: Write natural public copy**

Make prompt management the Chinese/English homepage topic without repetition. Add contextual localized links from home, footer, and docs index. Update docs-index and subscription metadata descriptions to state their real supporting role, preserving actual product claims and locale pairing.

- [ ] **Step 4: Verify green and commit**

Run `npx vitest run lib/i18n/messages/home.test.ts lib/i18n/messages/prompt-management.test.ts lib/i18n/metadata.test.ts app/sitemap.test.ts`; expect PASS. Then stage only files listed in this task and run `git commit -m "feat: strengthen prompt management seo"`.

### Task 5: Final verification

**Files:**
- Modify only when a failed verification identifies a focused correction.

- [ ] **Step 1: Run web-app unit tests**

Run `npm run test:unit --workspace=@oh-my-prompt/web-app`; expect exit code 0.

- [ ] **Step 2: Build production output**

Run `npm run web:build`; expect exit code 0 and public static routes for `/prompt-management` and `/en/prompt-management`.

- [ ] **Step 3: Inspect final scope**

Run `git status --short` and `git diff --check`. Confirm no changes to `packages/web-app/.env.example` or `packages/web-app/lib/vision-proxy.ts`.
