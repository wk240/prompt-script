# Web App Global Localization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every user-visible web page a complete Chinese route and matching English `/en/...` route, controlled by one accessible language dropdown in the global header.

**Architecture:** Keep the URL as the only locale source: unprefixed routes are Chinese and `/en/...` routes are English. Route files stay thin and pass a typed `BillingLocale` into shared page components; feature-scoped typed dictionaries provide UI copy, while paired Markdown directories provide localized documentation. Existing API contracts remain stable and UI code localizes stable error codes.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind CSS, Vitest, Playwright, gray-matter, marked.

---

## File Structure

Create or extend these focused units:

- `packages/web-app/lib/i18n/locale.ts`: locale type re-export, prefix detection, paired pathname/search conversion, locale-aware URL helpers.
- `packages/web-app/lib/i18n/messages.ts`: shared layout and generic state copy with a type-enforced `zh`/`en` shape.
- `packages/web-app/lib/i18n/messages/*.ts`: feature dictionaries for home, auth, docs, backup, dashboard, and team.
- `packages/web-app/components/layout/LanguageDropdown.tsx`: the only global language control.
- `packages/web-app/components/pages/*.tsx`: shared public/auth/private page bodies receiving `locale`.
- `packages/web-app/app/en/**/page.tsx`: thin English route adapters.
- `packages/web-app/app/docs/content/en/*.md`: seven complete English documents paired by slug with the existing Chinese files.
- `packages/web-app/tests/i18n.spec.ts`: cross-page route and dropdown regression coverage.

Do not modify `packages/web-app/supabase/.temp/cli-latest`; it is a pre-existing user change.

### Task 1: Locale primitives and paired URL behavior

**Files:**
- Create: `packages/web-app/lib/i18n/locale.ts`
- Create: `packages/web-app/lib/i18n/locale.test.ts`
- Modify: `packages/web-app/lib/billing/locale.ts`

- [ ] **Step 1: Write failing tests for locale detection and paired URLs**

```ts
import { describe, expect, it } from 'vitest'
import { localeFromPathname, localizedHref, switchLocaleUrl } from './locale'

describe('web locale routing', () => {
  it.each([
    ['/', 'zh'], ['/docs/getting-started', 'zh'], ['/en', 'en'], ['/en/team/abc', 'en'],
  ] as const)('detects locale for %s', (pathname, locale) => {
    expect(localeFromPathname(pathname)).toBe(locale)
  })

  it('keeps dynamic segments and search params when switching language', () => {
    expect(switchLocaleUrl('/team/team-1?invite=ABC', 'en')).toBe('/en/team/team-1?invite=ABC')
    expect(switchLocaleUrl('/en/team/team-1?invite=ABC', 'zh')).toBe('/team/team-1?invite=ABC')
  })

  it('does not duplicate the English prefix', () => {
    expect(switchLocaleUrl('/en/docs/faq', 'en')).toBe('/en/docs/faq')
  })

  it('localizes an app path without changing external URLs', () => {
    expect(localizedHref('/subscription', 'en')).toBe('/en/subscription')
    expect(localizedHref('https://github.com/wk240/oh-my-prompt', 'en')).toBe('https://github.com/wk240/oh-my-prompt')
  })
})
```

- [ ] **Step 2: Run the focused test and verify RED**

Run: `cd packages/web-app && npx vitest run lib/i18n/locale.test.ts`

Expected: FAIL because `lib/i18n/locale.ts` does not exist.

- [ ] **Step 3: Implement the minimal URL-only locale helpers**

```ts
export type WebLocale = 'zh' | 'en'

export function localeFromPathname(pathname: string): WebLocale {
  return pathname === '/en' || pathname.startsWith('/en/') ? 'en' : 'zh'
}

export function switchLocalePath(pathname: string, locale: WebLocale): string {
  const unscoped = pathname === '/en' ? '/' : pathname.replace(/^\/en(?=\/)/, '')
  if (locale === 'zh') return unscoped || '/'
  return unscoped === '/' ? '/en' : `/en${unscoped}`
}

export function switchLocaleUrl(url: string, locale: WebLocale): string {
  const [pathname, query = ''] = url.split('?')
  const localized = switchLocalePath(pathname, locale)
  return query ? `${localized}?${query}` : localized
}

export function localizedHref(href: string, locale: WebLocale): string {
  return /^(?:https?:|mailto:|#)/.test(href) ? href : switchLocalePath(href, locale)
}
```

Make `lib/billing/locale.ts` re-export `WebLocale as BillingLocale` and `switchLocalePath` so billing imports remain compatible.

- [ ] **Step 4: Verify GREEN and regression compatibility**

Run: `cd packages/web-app && npx vitest run lib/i18n/locale.test.ts lib/billing/locale.test.ts`

Expected: both test files PASS.

- [ ] **Step 5: Commit the locale foundation**

```bash
git -C packages/web-app add lib/i18n/locale.ts lib/i18n/locale.test.ts lib/billing/locale.ts
git -C packages/web-app commit -m "feat: add global locale routing helpers"
```

### Task 2: Typed dictionaries and global language dropdown

**Files:**
- Create: `packages/web-app/lib/i18n/messages.ts`
- Create: `packages/web-app/lib/i18n/messages.test.ts`
- Create: `packages/web-app/components/layout/LanguageDropdown.tsx`
- Create: `packages/web-app/components/layout/LanguageDropdown.test.tsx`
- Modify: `packages/web-app/components/layout/Header.tsx`
- Modify: `packages/web-app/components/legal/PolicyPage.tsx`
- Delete: `packages/web-app/components/legal/LanguageMenu.tsx`

- [ ] **Step 1: Add failing dictionary parity tests**

```ts
import { describe, expect, it } from 'vitest'
import { messages } from './messages'

describe('global messages', () => {
  it('keeps Chinese and English keys identical', () => {
    expect(Object.keys(messages.en).sort()).toEqual(Object.keys(messages.zh).sort())
    expect(Object.keys(messages.en.header).sort()).toEqual(Object.keys(messages.zh.header).sort())
  })

  it('contains no empty localized values', () => {
    expect(JSON.stringify(messages.zh)).not.toContain('""')
    expect(JSON.stringify(messages.en)).not.toContain('""')
  })
})
```

- [ ] **Step 2: Run dictionary tests and verify RED**

Run: `cd packages/web-app && npx vitest run lib/i18n/messages.test.ts`

Expected: FAIL because `messages.ts` does not exist.

- [ ] **Step 3: Add exact shared copy with structural typing**

```ts
const zh = {
  header: { home: '首页', docs: '文档', subscription: '订阅', backup: '备份', team: '团队', logout: '退出', login: '登录', register: '注册', mainNav: '主导航' },
  language: { label: '语言', chinese: '中文', english: 'English' },
  common: { loading: '加载中…', unknownError: '操作失败，请稍后重试', close: '关闭', cancel: '取消', confirm: '确认' },
} as const

type MessageShape = { [K in keyof typeof zh]: { [P in keyof (typeof zh)[K]]: string } }

const en: MessageShape = {
  header: { home: 'Home', docs: 'Docs', subscription: 'Subscription', backup: 'Backup', team: 'Team', logout: 'Sign out', login: 'Sign in', register: 'Register', mainNav: 'Main navigation' },
  language: { label: 'Language', chinese: '中文', english: 'English' },
  common: { loading: 'Loading…', unknownError: 'Something went wrong. Please try again.', close: 'Close', cancel: 'Cancel', confirm: 'Confirm' },
}

export const messages = { zh, en } satisfies Record<'zh' | 'en', MessageShape>
```

- [ ] **Step 4: Add failing component tests for one accessible dropdown**

Use React DOM test utilities already available in React 19. Assert the trigger is named `语言：中文` or `Language: English`, opening reveals exactly two links, current locale has `aria-current="page"`, Escape closes it, and `PolicyPage` no longer renders a second language navigation.

Run: `cd packages/web-app && npx vitest run components/layout/LanguageDropdown.test.tsx`

Expected: FAIL because the component does not exist.

- [ ] **Step 5: Implement `LanguageDropdown` and replace both old controls**

Implement a client component with `usePathname`, `useSearchParams`, `useState`, `useRef`, and document pointer/keydown listeners. The trigger displays the current full language plus a chevron. Menu items use `switchLocaleUrl`; the current option has a check icon, `aria-current="page"`, and no redundant navigation. Use native buttons/links, `min-h-11`, visible `focus-visible:ring-2`, outside-click close, Escape close, ArrowUp/ArrowDown focus movement, and Enter/Space activation.

In `Header.tsx`, replace the two adjacent links with:

```tsx
<LanguageDropdown locale={locale} />
```

In `PolicyPage.tsx`, remove the `LanguageMenu` import and the title-row instance, then delete `components/legal/LanguageMenu.tsx`.

- [ ] **Step 6: Verify dropdown and dictionary tests**

Run: `cd packages/web-app && npx vitest run lib/i18n/messages.test.ts components/layout/LanguageDropdown.test.tsx`

Expected: PASS with one global language control.

- [ ] **Step 7: Commit the unified global control**

```bash
git -C packages/web-app add lib/i18n components/layout/Header.tsx components/layout/LanguageDropdown.tsx components/layout/LanguageDropdown.test.tsx components/legal/PolicyPage.tsx components/legal/LanguageMenu.tsx
git -C packages/web-app commit -m "feat: unify website language switcher"
```

### Task 3: Shared route adapters and locale-aware metadata

**Files:**
- Create: `packages/web-app/lib/i18n/metadata.ts`
- Create: `packages/web-app/lib/i18n/metadata.test.ts`
- Modify: `packages/web-app/app/layout.tsx`
- Modify: `packages/web-app/app/en/layout.tsx`
- Modify: `packages/web-app/lib/legal/metadata.ts`

- [ ] **Step 1: Write failing metadata tests**

Test that `localizedMetadata({ locale: 'en', pathname: '/docs', title: 'Docs', description: 'Oh My Prompt documentation' })` returns canonical `/en/docs`, alternates for `zh-CN` and `en`, and Open Graph locale `en_US`; the Chinese form returns `/docs` and `zh_CN`.

- [ ] **Step 2: Run the metadata test and verify RED**

Run: `cd packages/web-app && npx vitest run lib/i18n/metadata.test.ts`

Expected: FAIL because `localizedMetadata` does not exist.

- [ ] **Step 3: Implement shared metadata generation**

Create a pure helper returning Next.js `Metadata` with canonical, language alternates, title, description, and Open Graph locale. Make legal metadata delegate to it. Keep the root layout `lang="zh-CN"`; ensure the English layout emits an English-language wrapper without duplicating global providers.

- [ ] **Step 4: Verify metadata tests and typecheck**

Run: `cd packages/web-app && npx vitest run lib/i18n/metadata.test.ts lib/legal/metadata.test.ts && npx tsc --noEmit`

Expected: PASS and exit 0.

- [ ] **Step 5: Commit metadata support**

```bash
git -C packages/web-app add lib/i18n/metadata.ts lib/i18n/metadata.test.ts lib/legal/metadata.ts app/layout.tsx app/en/layout.tsx
git -C packages/web-app commit -m "feat: add localized website metadata"
```

### Task 4: Home page, footer, and login modal

**Files:**
- Create: `packages/web-app/lib/i18n/messages/home.ts`
- Create: `packages/web-app/components/pages/HomePage.tsx`
- Modify: `packages/web-app/app/page.tsx`
- Create: `packages/web-app/app/en/page.tsx`
- Modify: `packages/web-app/components/layout/Footer.tsx`
- Modify: `packages/web-app/components/auth/LoginModal.tsx`
- Create: `packages/web-app/tests/home-localization.spec.ts`

- [ ] **Step 1: Write failing Playwright coverage**

```ts
import { expect, test } from '@playwright/test'

test('home page switches between complete Chinese and English routes', async ({ page }) => {
  await page.goto('/')
  await expect(page.getByRole('heading', { name: /一键插入你的提示词/ })).toBeVisible()
  await page.getByRole('button', { name: '语言：中文' }).click()
  await page.getByRole('link', { name: 'English' }).click()
  await expect(page).toHaveURL('/en')
  await expect(page.getByRole('heading', { name: /Insert your prompts in one click/ })).toBeVisible()
  await expect(page.getByText('核心特性')).toHaveCount(0)
})
```

- [ ] **Step 2: Run the E2E test and verify RED**

Run: `cd packages/web-app && npx playwright test tests/home-localization.spec.ts`

Expected: FAIL because `/en` and the dropdown do not yet provide a complete English home page.

- [ ] **Step 3: Extract a shared localized home page**

Move the existing landing markup from `app/page.tsx` into `components/pages/HomePage.tsx`. Give it `locale: WebLocale`, source every visible string and image alt from `messages/home.ts`, pass locale into Header/Footer/LoginModal, and localize the subscription destination with `localizedHref('/subscription', locale)`. Keep all existing layout classes and business behavior unchanged.

`app/page.tsx` becomes `<HomePage locale="zh" />`; `app/en/page.tsx` becomes `<HomePage locale="en" />`.

Translate all current home copy faithfully, including hero, feature cards, usage steps, calls to action, FAQ-like copy if present, and loading state. Do not translate product names or user data.

- [ ] **Step 4: Verify home localization**

Run: `cd packages/web-app && npx playwright test tests/home-localization.spec.ts && npx tsc --noEmit`

Expected: PASS and exit 0.

- [ ] **Step 5: Commit the bilingual home page**

```bash
git -C packages/web-app add app/page.tsx app/en/page.tsx components/pages/HomePage.tsx components/layout/Footer.tsx components/auth/LoginModal.tsx lib/i18n/messages/home.ts tests/home-localization.spec.ts
git -C packages/web-app commit -m "feat: localize website home page"
```

### Task 5: Documentation index and all seven documents

**Files:**
- Modify: `packages/web-app/app/docs/lib.ts`
- Create: `packages/web-app/app/docs/lib.test.ts`
- Create: `packages/web-app/components/pages/DocsIndexPage.tsx`
- Create: `packages/web-app/components/pages/DocPage.tsx`
- Modify: `packages/web-app/app/docs/page.tsx`
- Modify: `packages/web-app/app/docs/[slug]/page.tsx`
- Create: `packages/web-app/app/en/docs/page.tsx`
- Create: `packages/web-app/app/en/docs/[slug]/page.tsx`
- Create: `packages/web-app/app/docs/content/en/faq.md`
- Create: `packages/web-app/app/docs/content/en/getting-started.md`
- Create: `packages/web-app/app/docs/content/en/import-export.md`
- Create: `packages/web-app/app/docs/content/en/platform-support.md`
- Create: `packages/web-app/app/docs/content/en/prompt-agent.md`
- Create: `packages/web-app/app/docs/content/en/team-sharing.md`
- Create: `packages/web-app/app/docs/content/en/vision-api.md`
- Create: `packages/web-app/tests/docs-localization.spec.ts`

- [ ] **Step 1: Write failing unit tests for document parity**

```ts
import { describe, expect, it } from 'vitest'
import { getDoc, getDocs } from './lib'

describe('localized docs', () => {
  it('has an English document for every Chinese slug', () => {
    expect(getDocs('en').map(({ slug }) => slug).sort()).toEqual(getDocs('zh').map(({ slug }) => slug).sort())
  })

  it('loads localized content without Chinese fallback', () => {
    expect(getDoc('getting-started', 'en')?.title).toBe('Getting Started')
    expect(getDoc('getting-started', 'en')?.content).not.toContain('快速开始')
  })
})
```

- [ ] **Step 2: Run doc tests and verify RED**

Run: `cd packages/web-app && npx vitest run app/docs/lib.test.ts`

Expected: FAIL because the loader has no locale argument and English documents do not exist.

- [ ] **Step 3: Make the loader locale-aware**

Change `getDocs(locale)` and `getDoc(slug, locale)` to resolve Chinese files from `app/docs/content/*.md` and English files from `app/docs/content/en/*.md`. Filter directories out of the Chinese file list. Return `null` for a missing localized file; never fall back across locales.

- [ ] **Step 4: Translate all seven documents completely**

Create paired English Markdown with the same filenames and frontmatter fields. Translate every heading, paragraph, list, table, callout, image alt, and internal link label. Preserve code, command names, product/platform names, URLs, anchors, and technical identifiers. Rewrite internal `/docs/...` links as `/en/docs/...` in English documents.

- [ ] **Step 5: Add shared docs pages and thin route adapters**

Both shared pages accept `locale`. The index uses localized title, description, empty state, and locale-prefixed document links. The detail page loads the matching localized Markdown and passes locale into Header. Both language variants generate static params from their own locale set.

- [ ] **Step 6: Verify unit and browser behavior**

Run: `cd packages/web-app && npx vitest run app/docs/lib.test.ts && npx playwright test tests/docs-localization.spec.ts`

Expected: all seven slugs PASS in both languages and `/en/docs/getting-started` contains English content only.

- [ ] **Step 7: Commit complete bilingual documentation**

```bash
git -C packages/web-app add app/docs app/en/docs components/pages/DocsIndexPage.tsx components/pages/DocPage.tsx tests/docs-localization.spec.ts
git -C packages/web-app commit -m "feat: translate website documentation"
```

### Task 6: Authentication pages and locale-preserving redirects

**Files:**
- Create: `packages/web-app/lib/i18n/messages/auth.ts`
- Create: `packages/web-app/components/pages/auth/LoginPage.tsx`
- Create: `packages/web-app/components/pages/auth/RegisterPage.tsx`
- Create: `packages/web-app/components/pages/auth/ForgotPasswordPage.tsx`
- Create: `packages/web-app/components/pages/auth/ResetPasswordPage.tsx`
- Create: `packages/web-app/components/pages/auth/SetPasswordPage.tsx`
- Create: `packages/web-app/components/pages/auth/VerifyPage.tsx`
- Create: `packages/web-app/components/pages/auth/VerifyOtpPage.tsx`
- Modify: `packages/web-app/app/auth/*/page.tsx`
- Create: `packages/web-app/app/en/auth/*/page.tsx`
- Modify: `packages/web-app/lib/auth/login-redirect.ts`
- Modify: `packages/web-app/lib/auth/redirect-url.ts`
- Modify: `packages/web-app/lib/auth/login-error.ts`
- Modify: `packages/web-app/lib/auth/verify-error.ts`
- Modify: corresponding `*.test.ts` files
- Create: `packages/web-app/tests/auth-localization.spec.ts`

- [ ] **Step 1: Add failing tests for locale-preserving auth**

Extend auth helper tests so an English login return URL stays under `/en`, callback redirects reject external origins while preserving `/en/...`, and stable errors map to exact Chinese and English messages. Add Playwright assertions for English login, registration, forgot/reset/set password, verify, and OTP headings and form labels.

- [ ] **Step 2: Run auth tests and verify RED**

Run: `cd packages/web-app && npx vitest run lib/auth/login-redirect.test.ts lib/auth/redirect-url.test.ts lib/auth/login-error.test.ts lib/auth/verify-error.test.ts && npx playwright test tests/auth-localization.spec.ts`

Expected: FAIL on missing `/en/auth/*` routes and English messages.

- [ ] **Step 3: Extract each auth page into one shared component**

Preserve each page's current validation, Supabase calls, timers, and security checks. Replace only visible literals with `messages/auth.ts`, accept `locale`, and localize internal links and redirect targets. Chinese and English route files each render the same component with a fixed locale.

- [ ] **Step 4: Localize errors by stable code**

Change error helpers to accept `locale` and return exact locale copy. Unknown errors use `messages[locale].common.unknownError`. Never display raw provider errors when the existing code currently sanitizes them.

- [ ] **Step 5: Verify all auth flows and typecheck**

Run: `cd packages/web-app && npx vitest run lib/auth/*.test.ts && npx playwright test tests/auth-localization.spec.ts && npx tsc --noEmit`

Expected: PASS and exit 0.

- [ ] **Step 6: Commit bilingual authentication**

```bash
git -C packages/web-app add app/auth app/en/auth components/pages/auth lib/i18n/messages/auth.ts lib/auth tests/auth-localization.spec.ts
git -C packages/web-app commit -m "feat: localize authentication flows"
```

### Task 7: Backup and dashboard localization

**Files:**
- Create: `packages/web-app/lib/i18n/messages/dashboard.ts`
- Create: `packages/web-app/components/pages/BackupPage.tsx`
- Modify: `packages/web-app/app/backup/page.tsx`
- Create: `packages/web-app/app/en/backup/page.tsx`
- Modify: `packages/web-app/app/dashboard/page.tsx`
- Create: `packages/web-app/app/en/dashboard/page.tsx`
- Modify: `packages/web-app/components/dashboard/TabNav.tsx`
- Modify: `packages/web-app/components/dashboard/UserMenu.tsx`
- Modify: `packages/web-app/components/dashboard/backup/SyncHistory.tsx`
- Modify: `packages/web-app/components/dashboard/backup/SyncStats.tsx`
- Modify: `packages/web-app/components/dashboard/subscription/*.tsx`
- Create: `packages/web-app/tests/dashboard-localization.spec.ts`

- [ ] **Step 1: Add failing dashboard route tests**

Test `/en/backup` and `/en/dashboard` for English navigation, loading, empty/error states, table headings, dates, plan labels, and internal links. Mock authenticated API responses using Playwright route handlers so tests do not require a live account.

- [ ] **Step 2: Run focused E2E tests and verify RED**

Run: `cd packages/web-app && npx playwright test tests/dashboard-localization.spec.ts`

Expected: FAIL because English routes and copy are missing.

- [ ] **Step 3: Localize backup and dashboard components**

Extract `BackupPage(locale)` from the current route. Pass locale through every dashboard child. Move all fixed copy into `messages/dashboard.ts`; use `Intl.DateTimeFormat(locale === 'zh' ? 'zh-CN' : 'en-US')` for timestamps and existing currency helpers with the matching locale. Keep API request payloads and storage semantics unchanged.

- [ ] **Step 4: Verify dashboard behavior**

Run: `cd packages/web-app && npx playwright test tests/dashboard-localization.spec.ts && npx tsc --noEmit`

Expected: PASS and exit 0.

- [ ] **Step 5: Commit dashboard localization**

```bash
git -C packages/web-app add app/backup app/en/backup app/dashboard app/en/dashboard components/pages/BackupPage.tsx components/dashboard lib/i18n/messages/dashboard.ts tests/dashboard-localization.spec.ts
git -C packages/web-app commit -m "feat: localize backup and dashboard pages"
```

### Task 8: Team pages, dialogs, and dynamic routes

**Files:**
- Create: `packages/web-app/lib/i18n/messages/team.ts`
- Create: `packages/web-app/components/pages/TeamListPage.tsx`
- Create: `packages/web-app/components/pages/TeamJoinPage.tsx`
- Create: `packages/web-app/components/pages/TeamDetailPage.tsx`
- Modify: `packages/web-app/app/team/page.tsx`
- Modify: `packages/web-app/app/team/join/page.tsx`
- Modify: `packages/web-app/app/team/[teamId]/page.tsx`
- Create: `packages/web-app/app/en/team/page.tsx`
- Create: `packages/web-app/app/en/team/join/page.tsx`
- Create: `packages/web-app/app/en/team/[teamId]/page.tsx`
- Modify: `packages/web-app/components/dashboard/team/*.tsx`
- Create: `packages/web-app/tests/team-localization.spec.ts`

- [ ] **Step 1: Add failing team localization tests**

Cover team list empty/loading/error states, create/join/invite/delete dialogs, dynamic team ID preservation, invitation query preservation, member roles, and localized dates. Assert switching `/team/team-1?invite=ABC` produces `/en/team/team-1?invite=ABC`.

- [ ] **Step 2: Run focused team tests and verify RED**

Run: `cd packages/web-app && npx playwright test tests/team-localization.spec.ts`

Expected: FAIL because `/en/team*` routes are missing.

- [ ] **Step 3: Extract shared pages and localize all team UI**

Each shared page receives `locale` plus existing route/search params. Pass locale through dialogs and cards. Translate fixed role names and system feedback but leave team names, member emails, invite codes, and user-entered text unchanged. Preserve existing delete confirmation and authorization behavior exactly.

- [ ] **Step 4: Verify team flows and typecheck**

Run: `cd packages/web-app && npx playwright test tests/team-localization.spec.ts && npx tsc --noEmit`

Expected: PASS and exit 0.

- [ ] **Step 5: Commit team localization**

```bash
git -C packages/web-app add app/team app/en/team components/pages/TeamListPage.tsx components/pages/TeamJoinPage.tsx components/pages/TeamDetailPage.tsx components/dashboard/team lib/i18n/messages/team.ts tests/team-localization.spec.ts
git -C packages/web-app commit -m "feat: localize team management pages"
```

### Task 9: Subscription, legal, contact, and checkout continuity

**Files:**
- Modify: `packages/web-app/components/subscription/SubscriptionPage.tsx`
- Modify: `packages/web-app/app/subscription/SubscriptionContent.tsx`
- Modify: `packages/web-app/components/billing/*.tsx`
- Modify: `packages/web-app/components/legal/PolicyPage.tsx`
- Modify: `packages/web-app/lib/legal/content.ts`
- Modify: `packages/web-app/lib/creem/checkout.ts`
- Modify: `packages/web-app/tests/billing.spec.ts`

- [ ] **Step 1: Extend failing integration coverage**

Update billing tests to open the new dropdown instead of locating two always-visible links. Assert subscription selection, payment method modal, QR states, cancellation, history, legal links, success URLs, and return navigation stay in the selected locale.

- [ ] **Step 2: Run billing tests and verify RED**

Run: `cd packages/web-app && npx playwright test tests/billing.spec.ts`

Expected: FAIL where tests expect old language links or flows lose locale.

- [ ] **Step 3: Finish locale propagation in existing bilingual pages**

Replace remaining fixed Chinese/English branches with the feature dictionary where practical, but keep the existing billing request locale contract. Ensure English checkout uses `/en/subscription`, English legal links use `/en/...`, and all visible QR/payment status copy is localized. Remove any remaining legal-page local switcher markup.

- [ ] **Step 4: Verify billing and legal unit tests**

Run: `cd packages/web-app && npx playwright test tests/billing.spec.ts && npx vitest run lib/legal/content.test.ts lib/legal/metadata.test.ts lib/creem/checkout.test.ts app/api/billing/subscribe/route.test.ts`

Expected: PASS.

- [ ] **Step 5: Commit checkout continuity**

```bash
git -C packages/web-app add components/subscription app/subscription components/billing components/legal lib/legal lib/creem/checkout.ts tests/billing.spec.ts
git -C packages/web-app commit -m "fix: preserve locale across billing and legal flows"
```

### Task 10: Whole-site route matrix and Chinese-residue guard

**Files:**
- Create: `packages/web-app/lib/i18n/route-matrix.ts`
- Create: `packages/web-app/lib/i18n/route-matrix.test.ts`
- Create: `packages/web-app/tests/i18n.spec.ts`
- Modify: `packages/web-app/package.json`

- [ ] **Step 1: Add a failing route-pair matrix test**

Define all static route pairs plus factories for `/docs/[slug]` and `/team/[teamId]`. The unit test confirms each static English route has a matching page module and each Chinese page in scope appears in the matrix. The Playwright test visits representative public, auth, docs, private, team, subscription, contact, and legal routes and confirms one language dropdown and no 404.

- [ ] **Step 2: Add an explicit English UI residue test**

For each deterministic English page, collect visible body text and reject known fixed Chinese UI strings from the corresponding Chinese dictionary. Exclude user data, product names, Markdown code blocks, and the literal Chinese menu option `中文` so the guard detects interface regressions without false positives.

- [ ] **Step 3: Run the matrix and verify RED**

Run: `cd packages/web-app && npx vitest run lib/i18n/route-matrix.test.ts && npx playwright test tests/i18n.spec.ts`

Expected: FAIL on any unpaired or untranslated route still remaining.

- [ ] **Step 4: Close only reported localization gaps**

Add missing route adapters, dictionary keys, metadata, error mappings, or locale propagation identified by the tests. Do not refactor unrelated code. Add `test:i18n` to `package.json`:

```json
"test:i18n": "vitest run lib/i18n app/docs/lib.test.ts && playwright test tests/i18n.spec.ts tests/home-localization.spec.ts tests/docs-localization.spec.ts tests/auth-localization.spec.ts tests/dashboard-localization.spec.ts tests/team-localization.spec.ts tests/billing.spec.ts"
```

- [ ] **Step 5: Verify the localization suite passes**

Run: `cd packages/web-app && npm run test:i18n`

Expected: all locale unit and E2E tests PASS.

- [ ] **Step 6: Commit the global route guard**

```bash
git -C packages/web-app add lib/i18n/route-matrix.ts lib/i18n/route-matrix.test.ts tests/i18n.spec.ts package.json
git -C packages/web-app commit -m "test: enforce complete website localization"
```

### Task 11: Full verification and parent repository handoff

**Files:**
- Modify only files required by failures directly caused by Tasks 1–10.
- Update: root submodule pointer for `packages/web-app` after all web-app commits.

- [ ] **Step 1: Run the complete unit suite**

Run: `npm run test:unit --workspace=@oh-my-prompt/web-app`

Expected: exit 0 with no failing tests.

- [ ] **Step 2: Run TypeScript and lint checks**

Run: `cd packages/web-app && npx tsc --noEmit && npm run lint`

Expected: both commands exit 0. If pre-existing lint failures remain, record exact files and verify no new failures are introduced.

- [ ] **Step 3: Run the full Playwright suite**

Run: `npm run test --workspace=@oh-my-prompt/web-app`

Expected: exit 0 with no failing tests.

- [ ] **Step 4: Run a production build**

Run: `npm run web:build`

Expected: Next.js production build exits 0 and lists all `/en/...` routes.

- [ ] **Step 5: Inspect the final diff and route coverage**

Run:

```bash
git -C packages/web-app status --short
git -C packages/web-app diff --check HEAD~10..HEAD
find packages/web-app/app/en -name page.tsx | sort
```

Expected: only intended localization files plus the pre-existing `supabase/.temp/cli-latest` change; no whitespace errors; every scoped route has an English entry.

- [ ] **Step 6: Commit the parent submodule pointer only**

```bash
git add packages/web-app
git commit -m "feat: localize website in Chinese and English"
```

Do not stage the existing unrelated root documentation changes.
