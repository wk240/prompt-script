# Locale-Based Subscription Payment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route Chinese subscription purchases directly to WeChat Pay and English purchases directly to Creem/USD, with matching server-side enforcement and no payment-method selector.

**Architecture:** Treat the URL-derived `BillingLocale` as the sole payment-channel input. A small pure helper maps `zh → wechat_pay` and `en → creem`; the subscription UI renders one currency and submits directly, while each API route rejects the opposite locale before eligibility or provider calls.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Vitest, Playwright.

---

## File Structure

- `packages/web-app/lib/billing/payment-locale.ts`: pure locale-to-method, currency and display helpers.
- `packages/web-app/lib/billing/payment-locale.test.ts`: channel and price presentation contract.
- `packages/web-app/app/api/billing/subscribe/route.ts`: enforce English-only Creem checkout.
- `packages/web-app/app/api/billing/wechat-pay/route.ts`: require Chinese locale and enforce WeChat-only checkout.
- `packages/web-app/app/subscription/SubscriptionContent.tsx`: send the locale-derived request to the one allowed endpoint.
- `packages/web-app/components/dashboard/subscription/PlanComparison.tsx`: single-currency display and direct checkout state.
- `packages/web-app/components/dashboard/subscription/CurrentPlan.tsx`: Chinese-only WeChat renewal action.
- `packages/web-app/tests/billing.spec.ts`: browser-level direct payment and absence-of-selector coverage.

### Task 1: Locale-to-payment contract

**Files:**
- Create: `packages/web-app/lib/billing/payment-locale.ts`
- Create: `packages/web-app/lib/billing/payment-locale.test.ts`
- Modify: `packages/web-app/package.json`

- [ ] **Step 1: Write failing tests for channel and display selection**

```ts
import { describe, expect, it } from 'vitest'
import { getLocalePaymentPresentation } from './payment-locale'

describe('locale payment presentation', () => {
  it('uses WeChat Pay and CNY for Chinese pages', () => {
    expect(getLocalePaymentPresentation('zh', 900)).toEqual({
      method: 'wechat_pay', currency: 'CNY', display: '¥9',
    })
  })

  it('uses Creem and USD for English pages', () => {
    expect(getLocalePaymentPresentation('en', 900)).toEqual({
      method: 'creem', currency: 'USD', display: '$9',
    })
  })
})
```

- [ ] **Step 2: Run RED**

Run: `cd packages/web-app && npx vitest run lib/billing/payment-locale.test.ts`

Expected: FAIL because the module does not exist.

- [ ] **Step 3: Implement the minimal typed helper**

```ts
import type { PaymentMethod } from '@/components/billing/PaymentMethodSelector'
import type { BillingLocale } from './locale'

export function getLocalePaymentPresentation(locale: BillingLocale, amount: number): {
  method: PaymentMethod; currency: 'CNY' | 'USD'; display: string
} {
  return locale === 'zh'
    ? { method: 'wechat_pay', currency: 'CNY', display: `¥${amount / 100}` }
    : { method: 'creem', currency: 'USD', display: `$${amount / 100}` }
}
```

- [ ] **Step 4: Add the test to the explicit `test:unit` list and verify GREEN**

Run: `cd packages/web-app && npx vitest run lib/billing/payment-locale.test.ts && npm run test:unit`

Expected: focused tests and complete unit suite PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/billing/payment-locale.ts lib/billing/payment-locale.test.ts package.json
git commit -m "feat: map subscription locale to payment channel"
```

### Task 2: Server-side channel enforcement

**Files:**
- Modify: `packages/web-app/app/api/billing/subscribe/route.ts`
- Modify: `packages/web-app/app/api/billing/subscribe/route.test.ts`
- Modify: `packages/web-app/app/api/billing/wechat-pay/route.ts`
- Modify: `packages/web-app/app/api/billing/wechat-pay/route.test.ts`

- [ ] **Step 1: Add failing Creem mismatch tests**

Add a logged-in request with `{ plan: 'pro', interval: 'monthly', locale: 'zh' }`. Assert HTTP 400, `{ error: 'PAYMENT_METHOD_LOCALE_MISMATCH' }`, and that eligibility and `createCreemCheckout` were not called.

- [ ] **Step 2: Add failing WeChat locale tests**

Change valid WeChat payloads to include `locale: 'zh'`. Add an English payload and assert HTTP 400 with `PAYMENT_METHOD_LOCALE_MISMATCH`, before eligibility or `createNativeOrder`. Add missing, extra and invalid locale cases that return `INVALID_WECHAT_ORDER_FIELDS`.

- [ ] **Step 3: Run RED**

Run: `cd packages/web-app && npx vitest run app/api/billing/subscribe/route.test.ts app/api/billing/wechat-pay/route.test.ts`

Expected: mismatch and new three-field WeChat contract tests FAIL.

- [ ] **Step 4: Enforce locale before provider work**

In Creem validation, retain the three-field whitelist and valid `zh | en` parsing, then return `PAYMENT_METHOD_LOCALE_MISMATCH` when the parsed locale is not `en`.

In WeChat validation, require exactly `plan`, `interval`, and `locale`; accept only `zh | en` as structurally valid, then return `PAYMENT_METHOD_LOCALE_MISMATCH` for `en`. Do this before `getPurchaseEligibility` and `createNativeOrder`.

- [ ] **Step 5: Verify GREEN and API regressions**

Run: `cd packages/web-app && npx vitest run app/api/billing/subscribe/route.test.ts app/api/billing/wechat-pay/route.test.ts && npm run test:unit`

Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add app/api/billing/subscribe/route.ts app/api/billing/subscribe/route.test.ts app/api/billing/wechat-pay/route.ts app/api/billing/wechat-pay/route.test.ts
git commit -m "feat: enforce locale-specific payment APIs"
```

### Task 3: Direct locale-specific checkout UI

**Files:**
- Modify: `packages/web-app/app/subscription/SubscriptionContent.tsx`
- Modify: `packages/web-app/components/dashboard/subscription/PlanComparison.tsx`
- Modify: `packages/web-app/components/dashboard/subscription/CurrentPlan.tsx`
- Create: `packages/web-app/components/dashboard/subscription/PlanComparison.test.tsx`
- Modify: `packages/web-app/package.json`

- [ ] **Step 1: Write failing presentation and request tests**

Test pure exported helpers used by the real component:

```ts
expect(getPlanPaymentPresentation('zh', 'pro', 'monthly')).toMatchObject({ method: 'wechat_pay', display: '¥9' })
expect(getPlanPaymentPresentation('en', 'pro', 'monthly')).toMatchObject({ method: 'creem', display: '$9' })
expect(getSubscriptionCheckoutRequest('zh', 'pro', 'monthly')).toEqual({ endpoint: '/api/billing/wechat-pay', body: { plan: 'pro', interval: 'monthly', locale: 'zh' } })
expect(getSubscriptionCheckoutRequest('en', 'pro', 'monthly')).toEqual({ endpoint: '/api/billing/subscribe', body: { plan: 'pro', interval: 'monthly', locale: 'en' } })
```

Also assert the active `PlanComparison` source no longer imports or renders `PaymentMethodModal` or `PaymentMethodSelector`.

- [ ] **Step 2: Run RED**

Run: `cd packages/web-app && npx vitest run components/dashboard/subscription/PlanComparison.test.tsx`

Expected: helper expectations and selector-removal assertion FAIL.

- [ ] **Step 3: Make `SubscriptionContent` submit one locale-derived request**

Remove the `method` argument from its checkout callback. Use a pure `getSubscriptionCheckoutRequest(locale, plan, interval)` helper. Both request bodies include locale. Preserve login redirect, HTTPS validation for Creem, WeChat expiry conversion, and localized errors.

- [ ] **Step 4: Replace the modal with direct checkout state**

In `PlanComparison`:

- render only `getLocalePaymentPresentation(locale, amount).display`;
- render free price as `¥0` in Chinese and `$0` in English;
- remove the active `PaymentMethodModal` import and render path;
- on a logged-in paid-plan click, immediately call `onCheckout(plan, interval)`;
- disable the active button and show localized processing text while pending;
- on Chinese success, render the existing `WechatPayQRCode`;
- on English success, allow `SubscriptionContent` to navigate away;
- keep localized retry errors and unauthenticated login navigation.

- [ ] **Step 5: Restrict WeChat renewal by locale**

Only expose and handle `renew-wechat` on `locale === 'zh'`. Keep provider-specific Creem management behavior unchanged.

- [ ] **Step 6: Verify focused and complete unit tests**

Run: `cd packages/web-app && npx vitest run components/dashboard/subscription/PlanComparison.test.tsx lib/billing/payment-locale.test.ts && npm run test:unit`

Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add app/subscription/SubscriptionContent.tsx components/dashboard/subscription/PlanComparison.tsx components/dashboard/subscription/CurrentPlan.tsx components/dashboard/subscription/PlanComparison.test.tsx package.json
git commit -m "feat: start checkout directly from subscription plans"
```

### Task 4: Browser regression and final verification

**Files:**
- Modify: `packages/web-app/tests/billing.spec.ts`

- [ ] **Step 1: Rewrite the payment-selection E2E expectations and verify RED**

Add behavior coverage:

- Chinese plans show `¥` prices and no `$` paid prices.
- English plans show `$` prices and no `¥` paid prices.
- Neither locale renders the payment-method radiogroup or selector labels.
- Chinese plan click posts `{ plan, interval, locale: 'zh' }` to WeChat and renders QR.
- English plan click posts `{ plan, interval, locale: 'en' }` to Creem and navigates to the HTTPS URL.
- Chinese exposes WeChat manual renewal when eligible; English does not.

Run: `cd packages/web-app && NEXT_PUBLIC_SUPABASE_URL=https://example.supabase.co NEXT_PUBLIC_SUPABASE_ANON_KEY=dummy SUPABASE_SERVICE_ROLE_KEY=dummy npx playwright test tests/billing.spec.ts`

Expected: old UI fails the new direct-flow assertions.

- [ ] **Step 2: Make only test-driven UI corrections**

Fix any direct-flow defects exposed by Playwright without reintroducing a selector or changing provider contracts.

- [ ] **Step 3: Run complete verification**

```bash
cd packages/web-app
npm run test:unit
NEXT_PUBLIC_SUPABASE_URL=https://example.supabase.co NEXT_PUBLIC_SUPABASE_ANON_KEY=dummy SUPABASE_SERVICE_ROLE_KEY=dummy npm run test:i18n
NEXT_PUBLIC_SUPABASE_URL=https://example.supabase.co NEXT_PUBLIC_SUPABASE_ANON_KEY=dummy SUPABASE_SERVICE_ROLE_KEY=dummy npm test
npm run lint
NEXT_PUBLIC_SUPABASE_URL=https://example.supabase.co NEXT_PUBLIC_SUPABASE_ANON_KEY=dummy SUPABASE_SERVICE_ROLE_KEY=dummy npm run build
git diff --check
```

Expected: unit, i18n, full Playwright, lint and production build exit 0. Existing explicitly skipped authenticated fixtures may remain skipped; no failures are allowed.

- [ ] **Step 4: Commit**

```bash
git add tests/billing.spec.ts
git commit -m "test: cover locale-specific subscription checkout"
```
