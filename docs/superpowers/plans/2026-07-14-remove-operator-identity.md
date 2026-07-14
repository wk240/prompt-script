# Remove Public Operator Identity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make all public legal pages render without operator name, address, jurisdiction, or `LEGAL_OPERATOR_*` configuration.

**Architecture:** Legal pages become static bilingual policy documents backed only by `legalContent`; `PolicyPage` no longer accepts runtime identity data. Removing the identity loader eliminates the build/runtime failure while the support email remains literal policy content. Production payment readiness stays pending because this change intentionally does not supply a verified contracting or data-controller identity.

**Tech Stack:** Next.js 16, React, TypeScript, Vitest, Playwright, Markdown

---

### Task 1: Define The Identity-Free Legal Content Contract

**Files:**
- Modify: `packages/web-app/lib/legal/content.test.ts`
- Modify: `packages/web-app/lib/legal/content.ts`

- [ ] **Step 1: Write failing content tests**

Remove the `getLegalIdentity` test block and update `expectedIds` to exclude identity-dependent sections:

```ts
const expectedIds: Record<LegalPage, string[]> = {
  privacy: ['data-collected', 'purposes-bases', 'processors', 'payments', 'international', 'retention', 'rights', 'security', 'minors', 'contact'],
  terms: ['service', 'plans-prices', 'creem-renewal', 'wechat-manual', 'delivery', 'content-rights', 'acceptable-use', 'third-parties', 'termination', 'liability', 'updates', 'support'],
  refund: ['window', 'excluded', 'calculation', 'annual-rule', 'example', 'request', 'channels', 'access', 'creem-discretion', 'mandatory-rights'],
  acceptableUse: ['sexual-content', 'deepfakes', 'illegal-harm', 'malware', 'ip', 'regulated', 'platform-rules', 'abuse'],
  contact: ['support', 'response-time', 'billing-help', 'deletion'],
}
```

Add a test that serializes both locales and rejects public identity labels and removed section IDs:

```ts
it('does not publish operator identity fields', () => {
  const serialized = JSON.stringify(legalContent)
  for (const value of ['运营方信息', 'Operator information', '司法管辖区', 'Jurisdiction']) {
    expect(serialized).not.toContain(value)
  }
})
```

- [ ] **Step 2: Run the content test and verify RED**

Run:

```bash
cd packages/web-app
npx vitest run lib/legal/content.test.ts
```

Expected: FAIL because the content still includes `controller`, `operator`, `governing-law`, and public identity labels.

- [ ] **Step 3: Remove identity-dependent content**

In both locales of `lib/legal/content.ts`:

- delete Privacy `controller`;
- delete Terms `operator` and `governing-law`;
- delete Contact `operator`;
- renumber every remaining visible section heading sequentially from 1;
- retain `support@oh-my-prompt.com` and all payment, refund, privacy, acceptable-use, and support disclosures.

- [ ] **Step 4: Run the content test and verify GREEN**

Run:

```bash
npx vitest run lib/legal/content.test.ts
```

Expected: 10 tests pass after the two identity-loader tests are removed.

### Task 2: Remove Runtime Identity Loading And Rendering

**Files:**
- Delete: `packages/web-app/lib/legal/identity.ts`
- Modify: `packages/web-app/components/legal/PolicyPage.tsx`
- Modify: `packages/web-app/app/privacy/page.tsx`
- Modify: `packages/web-app/app/terms/page.tsx`
- Modify: `packages/web-app/app/refund/page.tsx`
- Modify: `packages/web-app/app/acceptable-use/page.tsx`
- Modify: `packages/web-app/app/contact/page.tsx`
- Modify: `packages/web-app/app/en/privacy/page.tsx`
- Modify: `packages/web-app/app/en/terms/page.tsx`
- Modify: `packages/web-app/app/en/refund/page.tsx`
- Modify: `packages/web-app/app/en/acceptable-use/page.tsx`
- Modify: `packages/web-app/app/en/contact/page.tsx`
- Modify: `packages/web-app/tests/billing.spec.ts`

- [ ] **Step 1: Add a browser regression assertion**

Extend the existing public legal-route test in `tests/billing.spec.ts` to assert Contact renders the support email and no operator labels:

```ts
await page.goto('/contact')
await expect(page.getByText('support@oh-my-prompt.com')).toBeVisible()
await expect(page.getByText('运营方', { exact: true })).toHaveCount(0)
await expect(page.getByText('地址', { exact: true })).toHaveCount(0)
await expect(page.getByText('司法管辖区', { exact: true })).toHaveCount(0)
```

- [ ] **Step 2: Simplify `PolicyPage`**

Change the props and function signature to:

```ts
interface PolicyPageProps {
  locale: BillingLocale
  page: LegalPage
}

export function PolicyPage({ locale, page }: PolicyPageProps) {
```

Remove the `LegalIdentity` import, identity labels, `showsIdentity`, `showsJurisdiction`, `<address>` block, and jurisdiction paragraph. Keep ordinary paragraph and list rendering unchanged.

- [ ] **Step 3: Simplify all ten route components**

For each Chinese and English legal route, remove the identity import and render only:

```tsx
return <PolicyPage locale="zh" page="privacy" />
```

Use the route's existing locale and page values. Delete `lib/legal/identity.ts` after no imports remain.

- [ ] **Step 4: Verify routes and type contracts**

Run:

```bash
npx vitest run lib/legal/content.test.ts
npx playwright test tests/billing.spec.ts --grep "serves all scoped subscription and legal routes publicly"
```

Expected: both commands pass without any `LEGAL_*` values.

### Task 3: Remove Configuration And Reconcile Documentation

**Files:**
- Modify: `packages/web-app/.env.example`
- Modify: `packages/web-app/README.md`
- Modify: `packages/web-app/playwright.config.ts`
- Modify: `docs/superpowers/specs/2026-07-13-creem-wechat-billing-compliance-design.md`
- Modify: `docs/superpowers/plans/2026-07-13-creem-wechat-billing-compliance.md`
- Modify: `packages/web-app/docs/billing-operations.md`

- [ ] **Step 1: Remove obsolete configuration**

Delete `LEGAL_OPERATOR_NAME`, `LEGAL_OPERATOR_ADDRESS`, and `LEGAL_JURISDICTION` from `.env.example`, README setup instructions, and Playwright web-server environment configuration.

- [ ] **Step 2: Update current billing documentation**

Remove instructions that require configuring or publishing the three deleted variables. State instead that public operator identity is intentionally not supplied and live payment readiness remains pending until the owner obtains qualified legal guidance and satisfies provider disclosure requirements. Do not mark live readiness complete or enable checkout.

- [ ] **Step 3: Verify obsolete contracts are gone**

Run:

```bash
rg -n --hidden --glob '!node_modules' --glob '!.next' 'LEGAL_OPERATOR_|LEGAL_IDENTITY_NOT_CONFIGURED|getLegalIdentity|operatorName|operatorAddress' packages/web-app
```

Expected: no matches.

### Task 4: Run Complete Verification

**Files:**
- Modify: `docs/superpowers/plans/2026-07-14-remove-operator-identity.md`

- [ ] **Step 1: Run the full unit suite**

Run:

```bash
cd packages/web-app
npm run test:unit
```

Expected: all registered unit tests pass.

- [ ] **Step 2: Run lint**

Run:

```bash
npm run lint
```

Expected: exit 0 with no new errors.

- [ ] **Step 3: Build without legal identity environment variables**

Run with synthetic Supabase and non-live billing values only:

```bash
env -u LEGAL_OPERATOR_NAME -u LEGAL_OPERATOR_ADDRESS -u LEGAL_JURISDICTION \
  NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321 \
  NEXT_PUBLIC_SUPABASE_ANON_KEY=test-anon \
  CREEM_TEST_MODE=true CREEM_CHECKOUT_ENABLED=false CREEM_API_KEY=test \
  CREEM_WEBHOOK_SECRET=test CREEM_PRO_MONTHLY_PRODUCT_ID=test \
  CREEM_PRO_YEARLY_PRODUCT_ID=test CREEM_TEAM_MONTHLY_PRODUCT_ID=test \
  CREEM_TEAM_YEARLY_PRODUCT_ID=test CRON_SECRET=test npm run build
```

Expected: production build passes and generates all scoped legal routes.

- [ ] **Step 4: Run final searches and diff checks**

Run:

```bash
cd ../..
rg -n --hidden --glob '!node_modules' --glob '!.next' 'LEGAL_OPERATOR_|LEGAL_IDENTITY_NOT_CONFIGURED|getLegalIdentity|operatorName|operatorAddress' packages/web-app
git diff --check
```

Expected: search prints no matches and diff check succeeds.

- [ ] **Step 5: Record completion without mixing unrelated changes**

Mark executed checkboxes complete. Commit only files whose entire diff belongs to this change; leave pre-existing team-page and team-test changes untouched and unstaged.
