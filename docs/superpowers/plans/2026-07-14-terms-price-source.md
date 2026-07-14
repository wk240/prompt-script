# Terms Price Source Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove duplicated prices and the standalone pricing section from both Terms locales while making the subscription page the stated source of current plan details and prices.

**Architecture:** Keep canonical prices and their tests in the billing plan/subscription modules. The legal-content model retains payment and renewal disclosures but no longer copies current price values, preventing Terms text from drifting from the subscription page.

**Tech Stack:** TypeScript, Vitest, Next.js

---

### Task 1: Remove Terms Price Duplication

**Files:**
- Modify: `packages/web-app/lib/legal/content.test.ts`
- Modify: `packages/web-app/lib/legal/content.ts`
- Modify: `docs/superpowers/specs/2026-07-13-creem-wechat-billing-compliance-design.md`
- Modify: `docs/superpowers/plans/2026-07-13-creem-wechat-billing-compliance.md`

- [x] **Step 1: Write failing Terms assertions**

Remove `plans-prices` from the expected Terms section IDs and remove the `getPlanPrice` import and loop that require prices inside all legal content. Add:

```ts
it('uses the subscription page as the only current price source', () => {
  const zhTerms = JSON.stringify(legalContent.zh.terms)
  const enTerms = JSON.stringify(legalContent.en.terms)
  expect(zhTerms).toContain('具体方案内容和价格以订阅页面展示为准')
  expect(enTerms).toContain('Current plan details and prices are shown on the subscription page')
  for (const terms of [zhTerms, enTerms]) {
    expect(terms).not.toContain('plans-prices')
    for (const price of ['¥9', '¥49', '¥99', '¥499', '$9', '$49', '$99', '$499']) {
      expect(terms).not.toContain(price)
    }
  }
})
```

- [x] **Step 2: Run the focused test and verify RED**

Run:

```bash
cd packages/web-app
npx vitest run lib/legal/content.test.ts
```

Expected: FAIL because both Terms documents still contain `plans-prices` and exact prices.

- [x] **Step 3: Update bilingual Terms content**

In `lib/legal/content.ts`:

- append `具体方案内容和价格以订阅页面展示为准。` to the Chinese Service paragraph;
- append `Current plan details and prices are shown on the subscription page.` to the English Service paragraph;
- delete both `plans-prices` sections;
- renumber the remaining headings from Creem renewal through Support from 2 through 11;
- leave renewal, tax, invoice, delivery, refund, and support disclosures unchanged.

- [x] **Step 4: Align current billing documentation**

Update the current billing design and implementation plan so Terms is described as linking plan details and prices to the subscription page rather than duplicating exact prices. Keep canonical-price and subscription-page verification requirements unchanged.

- [x] **Step 5: Run focused and complete verification**

Run:

```bash
cd packages/web-app
npx vitest run lib/legal/content.test.ts lib/billing/plans.test.ts
npm run test:unit
env -u LEGAL_OPERATOR_NAME -u LEGAL_OPERATOR_ADDRESS -u LEGAL_JURISDICTION \
  NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321 \
  NEXT_PUBLIC_SUPABASE_ANON_KEY=test-anon \
  CREEM_TEST_MODE=true CREEM_CHECKOUT_ENABLED=false CREEM_API_KEY=test \
  CREEM_WEBHOOK_SECRET=test CREEM_PRO_MONTHLY_PRODUCT_ID=test \
  CREEM_PRO_YEARLY_PRODUCT_ID=test CREEM_TEAM_MONTHLY_PRODUCT_ID=test \
  CREEM_TEAM_YEARLY_PRODUCT_ID=test CRON_SECRET=test npm run build
cd ../..
git diff --check
```

Expected: focused tests pass, all registered unit tests pass, production build passes, and diff check succeeds.

- [x] **Step 6: Preserve unrelated worktree changes**

Do not stage or modify the pre-existing team deletion page/test work. Commit only files whose complete diff belongs to this change; otherwise leave the verified changes unstaged and report that constraint.
