# Creem + WeChat Billing and Compliance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Stripe with secure Creem recurring billing while preserving WeChat manual renewals, making entitlement, ledger, refund, localization, and compliance behavior correct and testable.

**Architecture:** Store provider facts in `billing_subscriptions`, keep one recomputed effective row in `user_subscriptions`, and apply every verified provider mutation through one transactional Supabase RPC. Custom authenticated Next.js routes own all Creem product/customer/reference IDs; the UI consumes provider-neutral billing responses and shared bilingual legal content.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript 5, Supabase PostgreSQL/RLS/RPC, `creem@1.5.3`, `wechatpay-node-v3@2.2.1`, Vitest 4, Playwright 1.59, Vercel Cron.

---

## Execution Preconditions

- Run implementation inside `packages/web-app`, which is a git submodule with its own repository. Use `superpowers:using-git-worktrees` before execution if the existing dirty worktree cannot be preserved cleanly.
- Do not overwrite the existing unrelated edits in `packages/web-app/app/team/[teamId]/page.tsx` and `packages/web-app/tests/team.spec.ts`.
- Public operator name, address, and jurisdiction fields are intentionally omitted. Live readiness remains pending until qualified legal guidance confirms that all required disclosures are satisfied.
- Test and live Creem product IDs must be created before live checkout verification. Unit tests use fixed fake IDs and never call Creem.
- Apply the database migration in a local Supabase instance before running route tests that call the billing RPC.
- Install Docker and the Supabase CLI before Task 2; `npx supabase db reset` and `npx supabase test db` require the local Supabase stack.

## File Map

### Billing core

- Create `packages/web-app/lib/billing/types.ts`: provider-neutral plans, statuses, event and ledger contracts.
- Create `packages/web-app/lib/billing/plans.ts`: CNY/USD prices and server-only Creem product mapping.
- Create `packages/web-app/lib/billing/entitlements.ts`: effective entitlement reads and purchase eligibility.
- Create `packages/web-app/lib/billing/mutations.ts`: typed wrapper around the atomic database RPC.
- Create `packages/web-app/lib/billing/refunds.ts`: refund eligibility and round-half-up calculation.
- Create `packages/web-app/supabase/migrations/025_creem_billing_core.sql`: provider tables, ledger migration, RLS, RPCs, backfill, and Stripe-column removal.
- Create `packages/web-app/supabase/tests/creem_billing_core.test.sql`: pgTAP coverage for duplicate, stale, refund, and entitlement behavior.

### Creem boundary

- Create `packages/web-app/lib/creem/client.ts`: server-only SDK client.
- Create `packages/web-app/lib/creem/checkout.ts`: server-owned checkout creation.
- Create `packages/web-app/lib/creem/webhooks.ts`: verified event normalization without payload logging.
- Create `packages/web-app/lib/creem/reconcile.ts`: stale subscription reconciliation.
- Modify `packages/web-app/app/api/billing/subscribe/route.ts`: authenticated Creem checkout endpoint.
- Create `packages/web-app/app/api/billing/portal/route.ts`: authenticated customer portal endpoint.
- Create `packages/web-app/app/api/webhooks/creem/route.ts`: raw-body verified Creem webhook.
- Create `packages/web-app/app/api/cron/billing-reconcile/route.ts`: protected hourly reconciliation endpoint.

### WeChat boundary

- Modify `packages/web-app/lib/wechat-pay/native.ts`: renewal window and stacked period calculation.
- Modify `packages/web-app/lib/wechat-pay/payment-sync.ts`: replace direct subscription/history writes with billing RPC.
- Modify `packages/web-app/lib/wechat-pay/webhooks.ts`: pass notification ID/time into the billing RPC.
- Create `packages/web-app/lib/wechat-pay/refunds.ts`: refund creation/query/notification normalization.
- Create `packages/web-app/app/api/webhooks/wechat-refund/route.ts`: dedicated verified refund callback.

### Authorization, UI, and legal content

- Modify `packages/web-app/lib/sync-subscription.ts`, `packages/web-app/lib/official-api-quota.ts`, `packages/web-app/app/api/billing/status/route.ts`, and `packages/web-app/app/api/teams/route.ts`: require a future period end.
- Modify `packages/web-app/supabase/migrations/019_has_active_team_subscription_rpc.sql` only through replacement SQL in migration 025; never edit migration 019.
- Create `packages/web-app/lib/billing/locale.ts`, `packages/web-app/lib/legal/content.ts`, and `packages/web-app/components/legal/PolicyPage.tsx`: shared typed bilingual content.
- Create `packages/web-app/app/acceptable-use/page.tsx` plus `/en` wrappers for all six scoped pages.
- Modify subscription components, `Header`, `Footer`, and billing E2E tests for locale, dual prices, disclosure, pending state, portal, and renewal rules.

## Phase 1: Billing Core

### Task 1: Install Creem and Define Billing Contracts

**Files:**
- Modify: `packages/web-app/package.json`
- Modify: `package-lock.json` (parent workspace lockfile)
- Modify: `packages/web-app/.env.example`
- Create: `packages/web-app/lib/billing/types.ts`
- Create: `packages/web-app/lib/billing/plans.ts`
- Create: `packages/web-app/lib/billing/plans.test.ts`

- [ ] **Step 1: Write the failing plan-mapping tests**

```ts
import { afterEach, describe, expect, it } from 'vitest'
import { getCreemProductId, getPlanPrice } from './plans'

describe('billing plan mapping', () => {
  afterEach(() => {
    delete process.env.CREEM_PRO_MONTHLY_PRODUCT_ID
  })

  it('returns fixed minor-unit prices for both providers', () => {
    expect(getPlanPrice('creem', 'pro', 'monthly')).toEqual({ amount: 900, currency: 'USD' })
    expect(getPlanPrice('wechat_pay', 'team', 'yearly')).toEqual({ amount: 49900, currency: 'CNY' })
  })

  it('maps a server-owned plan pair to a Creem product', () => {
    process.env.CREEM_PRO_MONTHLY_PRODUCT_ID = 'prod_test_pro_monthly'
    expect(getCreemProductId('pro', 'monthly')).toBe('prod_test_pro_monthly')
  })

  it('fails closed when a product ID is missing', () => {
    expect(() => getCreemProductId('pro', 'monthly')).toThrow('CREEM_PRODUCT_NOT_CONFIGURED')
  })
})
```

- [ ] **Step 2: Run the tests and confirm the module is missing**

Run: `cd packages/web-app && npx vitest run lib/billing/plans.test.ts`

Expected: FAIL because `lib/billing/plans.ts` does not exist.

- [ ] **Step 3: Install the SDK and add the minimal contracts**

Run: `npm install creem@1.5.3 --workspace=@oh-my-prompt/web-app`

Add to `lib/billing/types.ts`:

```ts
export type BillingProvider = 'creem' | 'wechat_pay'
export type PaidPlan = 'pro' | 'team'
export type BillingInterval = 'monthly' | 'yearly'
export type BillingStatus =
  | 'pending'
  | 'active'
  | 'scheduled_cancel'
  | 'past_due'
  | 'unpaid'
  | 'canceled'
  | 'paused'
  | 'expired'
  | 'refunded'
  | 'disputed'

export type BillingLocale = 'zh' | 'en'
export type LedgerRecordType = 'payment' | 'refund' | 'dispute'
export type LedgerStatus =
  | 'succeeded'
  | 'failed'
  | 'partially_refunded'
  | 'refunded'
  | 'dispute_open'
  | 'dispute_won'
  | 'dispute_lost'

export const ACCESS_ELIGIBLE_STATUSES = new Set<BillingStatus>([
  'active',
  'scheduled_cancel',
  'past_due',
  'unpaid',
])
```

Add to `lib/billing/plans.ts`:

```ts
import type { BillingInterval, BillingProvider, PaidPlan } from './types'

const PRICES = {
  pro_monthly: { creem: 900, wechat_pay: 900 },
  pro_yearly: { creem: 9900, wechat_pay: 9900 },
  team_monthly: { creem: 4900, wechat_pay: 4900 },
  team_yearly: { creem: 49900, wechat_pay: 49900 },
} as const

const CREEM_PRODUCT_ENV = {
  pro_monthly: 'CREEM_PRO_MONTHLY_PRODUCT_ID',
  pro_yearly: 'CREEM_PRO_YEARLY_PRODUCT_ID',
  team_monthly: 'CREEM_TEAM_MONTHLY_PRODUCT_ID',
  team_yearly: 'CREEM_TEAM_YEARLY_PRODUCT_ID',
} as const

function planKey(plan: PaidPlan, interval: BillingInterval) {
  return `${plan}_${interval}` as keyof typeof PRICES
}

export function getPlanPrice(provider: BillingProvider, plan: PaidPlan, interval: BillingInterval) {
  return {
    amount: PRICES[planKey(plan, interval)][provider],
    currency: provider === 'creem' ? 'USD' as const : 'CNY' as const,
  }
}

export function getCreemProductId(plan: PaidPlan, interval: BillingInterval): string {
  const value = process.env[CREEM_PRODUCT_ENV[planKey(plan, interval)]]
  if (!value) throw new Error('CREEM_PRODUCT_NOT_CONFIGURED')
  return value
}

export function resolveCreemPlan(productId: string): { plan: PaidPlan; interval: BillingInterval } | null {
  for (const [key, envName] of Object.entries(CREEM_PRODUCT_ENV)) {
    if (process.env[envName] !== productId) continue
    const [plan, interval] = key.split('_') as [PaidPlan, BillingInterval]
    return { plan, interval }
  }
  return null
}
```

Replace Stripe variables in `.env.example` with `CREEM_API_KEY`, `CREEM_WEBHOOK_SECRET`, `CREEM_TEST_MODE`, four `CREEM_*_PRODUCT_ID` values, and `CRON_SECRET`.

- [ ] **Step 4: Run the focused tests**

Run: `cd packages/web-app && npx vitest run lib/billing/plans.test.ts`

Expected: 3 tests PASS.

- [ ] **Step 5: Commit the contract slice**

```bash
cd packages/web-app
git add package.json .env.example lib/billing/types.ts lib/billing/plans.ts lib/billing/plans.test.ts
git commit -m "feat(billing): add Creem billing contracts"
```

### Task 2: Add Provider Tables, Ledger, Entitlement RPC, and RLS

**Files:**
- Create: `packages/web-app/supabase/migrations/025_creem_billing_core.sql`
- Create: `packages/web-app/supabase/tests/creem_billing_core.test.sql`

- [ ] **Step 1: Write failing pgTAP cases for the new schema**

The test SQL must assert these concrete behaviors:

```sql
BEGIN;
SELECT no_plan();

SELECT has_table('public', 'billing_subscriptions');
SELECT has_table('public', 'billing_events');
SELECT has_table('public', 'refund_cases');
SELECT has_function('public', 'recompute_user_entitlement', ARRAY['uuid']);
SELECT has_function('public', 'process_billing_event', ARRAY[
  'text', 'text', 'text', 'timestamptz', 'uuid', 'text', 'text', 'text',
  'text', 'text', 'text', 'timestamptz', 'timestamptz', 'boolean', 'text',
  'text', 'text', 'integer', 'text', 'text', 'text'
]);
SELECT col_is_unique('public', 'user_subscriptions', ARRAY['user_id']);
SELECT col_is_unique('public', 'billing_events', ARRAY['payment_provider', 'external_event_id']);
SELECT index_is_unique('public', 'payment_history', 'payment_history_provider_record_idx');
SELECT policies_are('public', 'billing_events', ARRAY[]::text[]);
SELECT policies_are('public', 'refund_cases', ARRAY[]::text[]);
SELECT has_column('public', 'user_subscriptions', 'source_billing_subscription_id');
SELECT hasnt_column('public', 'user_subscriptions', 'stripe_subscription_id');

SELECT * FROM finish();
ROLLBACK;
```

- [ ] **Step 2: Run the database tests before the migration exists**

Run: `cd packages/web-app && npx supabase db reset && npx supabase test db`

Expected: FAIL on missing `billing_subscriptions` and RPCs.

- [ ] **Step 3: Write migration 025**

The migration must execute in this order:

1. Abort if a paid Stripe row exists.
2. Create `billing_subscriptions`, `billing_events`, and `refund_cases` with the exact checks from the design.
3. Add `source_billing_subscription_id` and `cancel_at_period_end boolean not null default false`; do not add the user uniqueness constraint yet.
4. Replace `payment_history.status` and the legacy idempotency index with `CREATE UNIQUE INDEX payment_history_provider_record_idx ON payment_history(payment_provider, record_type, external_record_id) WHERE external_record_id IS NOT NULL`.
5. Backfill successful `wechat_pay_orders` into `billing_subscriptions` using `out_trade_no` as `external_subscription_id` and `paid_at` as `last_paid_at`. Abort if a paid WeChat subscription row cannot be matched to a verified successful order.
6. Merge duplicate effective rows per user only after preserving the maximum used quota, latest quota reset, and all provider facts; then add the unique `user_subscriptions(user_id)` constraint.
7. Create `recompute_user_entitlement(uuid)` as a `SECURITY DEFINER` function with `SET search_path = public`.
8. Create the 21-argument `process_billing_event(text, text, text, timestamptz, uuid, text, text, text, text, text, text, timestamptz, timestamptz, boolean, text, text, text, integer, text, text, text)` as the only write path for provider events; the final argument is `p_disposition` constrained in the function to `process | ignored`.
9. Replace `has_active_team_subscription(uuid)` so the owner's effective row must have `plan_type = 'team'`, eligible status, and `current_period_end > now()`.
10. Revoke all mutation RPCs from `PUBLIC`, `anon`, and `authenticated`; grant only to `service_role`.
11. Recompute all users, then drop Stripe columns and the legacy provider constraint/default.

Use this selection inside `recompute_user_entitlement`:

```sql
SELECT id, payment_provider, plan_type, status, billing_interval,
       current_period_start, current_period_end, cancel_at_period_end, last_paid_at
INTO v_winner
FROM public.billing_subscriptions
WHERE user_id = p_user_id
  AND status IN ('active', 'scheduled_cancel', 'past_due', 'unpaid')
  AND last_paid_at IS NOT NULL
  AND current_period_end > now()
ORDER BY CASE plan_type WHEN 'team' THEN 2 WHEN 'pro' THEN 1 ELSE 0 END DESC,
         current_period_end DESC,
         created_at DESC
LIMIT 1;
```

When `v_winner` is null, upsert `plan_type = 'free'`, `status = 'inactive'`, null provider/source/period fields, and preserve quota counters. Otherwise upsert the winner and preserve quota counters.

Inside `process_billing_event`, first insert the event with `ON CONFLICT DO NOTHING`; return `{ "result": "duplicate" }` when no row was inserted. Lock the provider record with `FOR UPDATE`. Set `last_paid_at = p_provider_created_at` only when `p_record_type = 'payment'` and `p_ledger_status = 'succeeded'`, and never clear it during later lifecycle updates. Skip older lifecycle mutations; at equal timestamps apply the state precedence from design section 4.1. Always append refund/dispute ledger records and revoke their provider record, even when delivered late. Upsert ledger records on `(payment_provider, record_type, external_record_id)`, call `recompute_user_entitlement`, set the event to `processed` or `ignored`, and return JSON. Any exception must roll back the event insert and every mutation.

- [ ] **Step 4: Add behavioral pgTAP cases**

Extend the test file to call `process_billing_event` with fixed UUIDs and assert:

- `subscription.paid` creates one provider record, one payment record, and a Pro entitlement.
- `subscription.past_due` without an earlier successful payment remains ineligible.
- replaying the same event leaves all row counts unchanged.
- an older `subscription.active` does not overwrite a newer `expired` status.
- a late `refund.created` revokes the provider record and creates a separate refund ledger record.
- a second valid Team provider record keeps Team access when the Pro record is refunded.
- an elapsed period recomputes to free even if the provider status remains active.
- browser roles cannot insert or update provider/event/refund rows.

- [ ] **Step 5: Reset the database and run pgTAP**

Run: `cd packages/web-app && npx supabase db reset && npx supabase test db`

Expected: migration succeeds and all billing pgTAP assertions PASS.

- [ ] **Step 6: Commit the database slice**

```bash
cd packages/web-app
git add supabase/migrations/025_creem_billing_core.sql supabase/tests/creem_billing_core.test.sql
git commit -m "feat(billing): add provider ledger and entitlement RPC"
```

### Task 3: Centralize Effective Entitlement Reads

**Files:**
- Create: `packages/web-app/lib/billing/entitlements.ts`
- Create: `packages/web-app/lib/billing/entitlements.test.ts`
- Modify: `packages/web-app/lib/sync-subscription.ts`
- Modify: `packages/web-app/lib/sync-subscription.test.ts`
- Modify: `packages/web-app/lib/official-api-quota.ts`
- Modify: `packages/web-app/lib/official-api-quota.test.ts`
- Modify: `packages/web-app/app/api/billing/status/route.ts`
- Modify: `packages/web-app/app/api/billing/status/route.test.ts`
- Modify: `packages/web-app/app/api/teams/route.ts`
- Modify: `packages/web-app/app/api/teams/route.test.ts`

- [ ] **Step 1: Write failing tests for elapsed entitlements**

```ts
import { describe, expect, it } from 'vitest'
import { isEffectivePaidEntitlement } from './entitlements'

describe('isEffectivePaidEntitlement', () => {
  const now = new Date('2026-07-13T12:00:00.000Z')

  it('accepts active and scheduled-cancel rows with future periods', () => {
    expect(isEffectivePaidEntitlement({ plan_type: 'pro', status: 'active', current_period_end: '2026-08-01T00:00:00.000Z' }, now)).toBe(true)
    expect(isEffectivePaidEntitlement({ plan_type: 'team', status: 'scheduled_cancel', current_period_end: '2026-08-01T00:00:00.000Z' }, now)).toBe(true)
  })

  it('rejects elapsed active rows', () => {
    expect(isEffectivePaidEntitlement({ plan_type: 'team', status: 'active', current_period_end: '2026-07-01T00:00:00.000Z' }, now)).toBe(false)
  })
})
```

Add regression cases to each existing consumer test proving an active row with a past end date returns free/403 and cannot create a team.

- [ ] **Step 2: Run the focused tests and observe failures**

Run:

```bash
cd packages/web-app
npx vitest run lib/billing/entitlements.test.ts lib/sync-subscription.test.ts lib/official-api-quota.test.ts app/api/billing/status/route.test.ts app/api/teams/route.test.ts
```

Expected: FAIL because elapsed active rows are still accepted.

- [ ] **Step 3: Implement one shared predicate and query shape**

```ts
import { ACCESS_ELIGIBLE_STATUSES } from './types'

export type EffectiveEntitlementRow = {
  plan_type?: string | null
  status?: string | null
  current_period_end?: string | null
  payment_provider?: BillingProvider | null
  cancel_at_period_end?: boolean | null
}

export function isEffectivePaidEntitlement(
  row: EffectiveEntitlementRow | null | undefined,
  now = new Date()
): boolean {
  if (!row || (row.plan_type !== 'pro' && row.plan_type !== 'team')) return false
  if (!ACCESS_ELIGIBLE_STATUSES.has(row.status as never)) return false
  if (!row.current_period_end) return false
  const end = new Date(row.current_period_end)
  return Number.isFinite(end.getTime()) && end > now
}
```

Also add the purchase gate now because Creem checkout in Task 5 depends on it:

```ts
export async function readEffectiveEntitlement(
  client: SupabaseClient,
  userId: string,
  now = new Date()
): Promise<EffectiveEntitlementRow | null> {
  const { data, error } = await client
    .from('user_subscriptions')
    .select('plan_type,status,current_period_end,payment_provider,cancel_at_period_end')
    .eq('user_id', userId)
    .in('status', Array.from(ACCESS_ELIGIBLE_STATUSES))
    .gt('current_period_end', now.toISOString())
    .maybeSingle()
  if (error) throw new Error(`ENTITLEMENT_READ_FAILED:${error.message}`)
  return isEffectivePaidEntitlement(data, now) ? data : null
}

export type PurchaseEligibility =
  | { allowed: true; periodBase: Date }
  | { allowed: false; error: 'ACTIVE_SUBSCRIPTION_EXISTS' }

export async function getPurchaseEligibility(
  client: SupabaseClient,
  userId: string,
  provider: BillingProvider,
  plan: PaidPlan,
  now = new Date()
): Promise<PurchaseEligibility> {
  const entitlement = await readEffectiveEntitlement(client, userId, now)
  if (!entitlement) return { allowed: true, periodBase: now }
  const endsAt = new Date(entitlement.current_period_end!)
  const renewalOpensAt = new Date(endsAt.getTime() - 7 * 24 * 60 * 60 * 1000)
  if (provider === 'wechat_pay' && entitlement.payment_provider === 'wechat_pay' && entitlement.plan_type === plan && now >= renewalOpensAt) {
    return { allowed: true, periodBase: endsAt }
  }
  return { allowed: false, error: 'ACTIVE_SUBSCRIPTION_EXISTS' }
}
```

Update every consumer query to select `current_period_end`, filter `.gt('current_period_end', new Date().toISOString())`, and validate the returned row with the predicate. Do not duplicate an independent status list in consumers.

- [ ] **Step 4: Run the focused authorization tests**

Run the command from Step 2.

Expected: all focused tests PASS.

- [ ] **Step 5: Commit the authorization slice**

```bash
cd packages/web-app
git add lib/billing/entitlements.ts lib/billing/entitlements.test.ts lib/sync-subscription.ts lib/sync-subscription.test.ts lib/official-api-quota.ts lib/official-api-quota.test.ts app/api/billing/status/route.ts app/api/billing/status/route.test.ts app/api/teams/route.ts app/api/teams/route.test.ts
git commit -m "fix(billing): enforce paid period expiry everywhere"
```

### Task 4: Add the Typed Billing Mutation Wrapper

**Files:**
- Create: `packages/web-app/lib/billing/mutations.ts`
- Create: `packages/web-app/lib/billing/mutations.test.ts`

- [ ] **Step 1: Write a failing RPC mapping test**

```ts
import { describe, expect, it, vi } from 'vitest'
import { applyBillingEvent } from './mutations'

it('maps a normalized event to process_billing_event', async () => {
  const rpc = vi.fn(async () => ({ data: { result: 'processed' }, error: null }))
  const result = await applyBillingEvent({ rpc } as never, {
    provider: 'creem',
    eventId: 'evt_1',
    eventType: 'subscription.paid',
    providerCreatedAt: '2026-07-13T12:00:00.000Z',
    userId: '00000000-0000-4000-8000-000000000001',
    externalSubscriptionId: 'sub_1',
    externalCustomerId: 'cust_1',
    externalProductId: 'prod_1',
    plan: 'pro',
    status: 'active',
    interval: 'monthly',
    periodStart: '2026-07-13T12:00:00.000Z',
    periodEnd: '2026-08-13T12:00:00.000Z',
    cancelAtPeriodEnd: false,
    disposition: 'process',
    ledger: { type: 'payment', externalId: 'tran_1', originalTransactionId: null, amount: 900, currency: 'USD', status: 'succeeded' },
  })
  expect(result).toEqual({ result: 'processed' })
expect(rpc).toHaveBeenCalledWith('process_billing_event', expect.objectContaining({ p_event_id: 'evt_1', p_external_record_id: 'tran_1', p_disposition: 'process' }))
})
```

- [ ] **Step 2: Run the test and verify failure**

Run: `cd packages/web-app && npx vitest run lib/billing/mutations.test.ts`

Expected: FAIL because `applyBillingEvent` does not exist.

- [ ] **Step 3: Implement the typed wrapper**

Define the shared normalized contract:

```ts
export type NormalizedBillingEvent = {
  provider: BillingProvider
  eventId: string
  eventType: string
  providerCreatedAt: string
  userId: string | null
  externalSubscriptionId: string | null
  externalCustomerId: string | null
  externalProductId: string | null
  plan: PaidPlan | null
  status: BillingStatus | null
  interval: BillingInterval | null
  periodStart: string | null
  periodEnd: string | null
  cancelAtPeriodEnd: boolean
  disposition: 'process' | 'ignored'
  ledger: {
    type: LedgerRecordType
    externalId: string
    originalTransactionId: string | null
    amount: number
    currency: 'USD' | 'CNY'
    status: LedgerStatus
  } | null
}
```

Convert absent fields to null RPC arguments. The SQL function must short-circuit to an ignored event audit before requiring user/provider-record fields when `disposition === 'ignored'`. Call `process_billing_event`, throw `BILLING_EVENT_PERSIST_FAILED` on Supabase error, and accept only `{ result: 'processed' | 'duplicate' | 'ignored' }`.

```ts
export async function applyBillingEvent(client: SupabaseClient, event: NormalizedBillingEvent) {
  const { data, error } = await client.rpc('process_billing_event', toRpcArgs(event))
  if (error) throw new Error(`BILLING_EVENT_PERSIST_FAILED:${error.message}`)
  const result = data as { result?: string } | null
  if (!result || !['processed', 'duplicate', 'ignored'].includes(result.result || '')) {
    throw new Error('BILLING_EVENT_RESULT_INVALID')
  }
  return result as { result: 'processed' | 'duplicate' | 'ignored' }
}
```

- [ ] **Step 4: Run the wrapper tests**

Run: `cd packages/web-app && npx vitest run lib/billing/mutations.test.ts`

Expected: PASS.

- [ ] **Step 5: Commit the RPC wrapper**

```bash
cd packages/web-app
git add lib/billing/mutations.ts lib/billing/mutations.test.ts
git commit -m "feat(billing): add atomic event mutation wrapper"
```

### Task 5: Implement Secure Creem Checkout

**Files:**
- Create: `packages/web-app/lib/creem/client.ts`
- Create: `packages/web-app/lib/creem/checkout.ts`
- Create: `packages/web-app/lib/creem/checkout.test.ts`
- Modify: `packages/web-app/app/api/billing/subscribe/route.ts`
- Create: `packages/web-app/app/api/billing/subscribe/route.test.ts`

- [ ] **Step 1: Write failing checkout service and route tests**

Cover all four mappings, unauthenticated rejection, invalid plan/interval/locale rejection, active entitlement rejection, and ignored client fields. The decisive assertion is:

```ts
expect(mockCreate).toHaveBeenCalledWith({
  requestId: expect.stringMatching(/^checkout:user-1:/),
  productId: 'prod_server_pro_monthly',
  units: 1,
  customer: { email: 'user@example.com' },
  successUrl: 'https://oh-my-prompt.com/subscription?checkout=pending',
  metadata: { referenceId: 'user-1', plan: 'pro', interval: 'monthly' },
})
expect(mockCreate).not.toHaveBeenCalledWith(expect.objectContaining({ productId: 'prod_attacker' }))
```

- [ ] **Step 2: Run the checkout tests and confirm failure**

Run: `cd packages/web-app && npx vitest run lib/creem/checkout.test.ts app/api/billing/subscribe/route.test.ts`

Expected: FAIL because the route still imports Stripe.

- [ ] **Step 3: Implement the server-only Creem client and checkout**

```ts
import 'server-only'
import { Creem } from 'creem'

export function createCreemClient() {
  const apiKey = process.env.CREEM_API_KEY
  if (!apiKey) throw new Error('CREEM_API_KEY_MISSING')
  return new Creem({ apiKey, server: process.env.CREEM_TEST_MODE === 'true' ? 'test' : 'prod' })
}
```

`createCreemCheckout` accepts only `{ plan, interval, locale, userId, email }`. Build the success URL from `NEXT_PUBLIC_WEB_APP_URL` and the validated locale, obtain the product ID from `getCreemProductId`, and call `creem.checkouts.create` with exactly the fields asserted in Step 1.

In `app/api/billing/subscribe/route.ts`, parse only `plan`, `interval`, and `locale`; reject extra fields with `INVALID_CHECKOUT_FIELDS`, call `getPurchaseEligibility`, return `ACTIVE_SUBSCRIPTION_EXISTS` with HTTP 409 when blocked, and return `{ success: true, data: { url } }`.

- [ ] **Step 4: Run checkout tests**

Run the Step 2 command.

Expected: all checkout service and route cases PASS.

- [ ] **Step 5: Commit the checkout slice**

```bash
cd packages/web-app
git add lib/creem/client.ts lib/creem/checkout.ts lib/creem/checkout.test.ts app/api/billing/subscribe/route.ts app/api/billing/subscribe/route.test.ts
git commit -m "feat(billing): add authenticated Creem checkout"
```

### Task 6: Verify and Normalize Creem Webhooks

**Files:**
- Create: `packages/web-app/lib/creem/webhooks.ts`
- Create: `packages/web-app/lib/creem/webhooks.test.ts`
- Create: `packages/web-app/app/api/webhooks/creem/route.ts`
- Create: `packages/web-app/app/api/webhooks/creem/route.test.ts`

- [ ] **Step 1: Write failing event-normalization tests**

Create fixtures for `subscription.active`, `subscription.paid`, `subscription.scheduled_cancel`, `subscription.expired`, `refund.created`, and `dispute.created`. Assert these mappings:

```ts
expect(normalizeCreemEvent(activeFixture)).toMatchObject({ event: { status: 'pending', ledger: null } })
expect(normalizeCreemEvent(paidFixture)).toMatchObject({
  event: {
    status: 'active',
    userId: 'user-1',
    externalSubscriptionId: 'sub_1',
    ledger: { type: 'payment', externalId: 'tran_1', amount: 900, currency: 'USD', status: 'succeeded' },
  },
})
expect(normalizeCreemEvent(refundFixture)).toMatchObject({
  event: {
    status: 'refunded',
    ledger: { type: 'refund', externalId: 'ref_1', originalTransactionId: 'tran_1', status: 'partially_refunded' },
  },
})
```

Also assert that an unknown product returns an ignored normalized event with reason `UNKNOWN_PRODUCT`, a missing `referenceId` returns an ignored normalized event with reason `MISSING_REFERENCE_ID`, and no logger receives a full event object. `normalizeCreemEvent` always returns `{ event, reason? }`, so the route can audit ignored events through the same RPC.

- [ ] **Step 2: Run the webhook tests and verify failure**

Run: `cd packages/web-app && npx vitest run lib/creem/webhooks.test.ts app/api/webhooks/creem/route.test.ts`

Expected: FAIL because the normalizer and route do not exist.

- [ ] **Step 3: Implement verified parsing and normalization**

Use the official utility directly:

```ts
import { constructWebhookEventEntity } from 'creem/webhooks'

export async function verifyCreemWebhook(body: string, headers: Headers) {
  const secret = process.env.CREEM_WEBHOOK_SECRET
  if (!secret) throw new Error('CREEM_WEBHOOK_SECRET_MISSING')
  return constructWebhookEventEntity(body, headers, { secret })
}
```

The route must pass the original `Headers` object so verification reads the `creem-signature` header from the unmodified raw-body request.

Implement an exhaustive switch over the required event names. Normalize `subscription.active` to internal `pending`; only `subscription.paid` produces an access-eligible `active` record and payment ledger entry. Resolve product IDs with `resolveCreemPlan`. For refund/dispute events, use the provider refund/dispute ID as `externalId` and the nested transaction ID as `originalTransactionId`.

The route must:

```ts
export async function POST(request: Request) {
  const body = await request.text()
  try {
    const providerEvent = await verifyCreemWebhook(body, request.headers)
    const normalized = normalizeCreemEvent(providerEvent)
    const admin = createServiceRoleClient()
    const result = await applyBillingEvent(admin, normalized.event)
    console.info('[Oh My Prompt] Creem webhook processed', {
      eventId: normalized.event.eventId,
      eventType: normalized.event.eventType,
      result: result.result,
    })
    return Response.json({ success: true })
  } catch (error) {
    const invalidSignature = error instanceof Error && error.name === 'WebhookVerificationError'
    return Response.json(
      { success: false, error: invalidSignature ? 'INVALID_SIGNATURE' : 'WEBHOOK_PROCESSING_FAILED' },
      { status: invalidSignature ? 400 : 500 }
    )
  }
}
```

For ignored unknown products or missing references, call the RPC with an ignored event audit record and no billing/ledger mutation, then return HTTP 200. Never log `body`, `providerEvent`, customer email, or full metadata.

- [ ] **Step 4: Run webhook tests**

Run the Step 2 command.

Expected: signature failure returns 400, persistence failure returns 500, valid/duplicate/ignored events return 200, and all normalizer tests PASS.

- [ ] **Step 5: Commit the webhook slice**

```bash
cd packages/web-app
git add lib/creem/webhooks.ts lib/creem/webhooks.test.ts app/api/webhooks/creem/route.ts app/api/webhooks/creem/route.test.ts
git commit -m "feat(billing): process verified Creem webhooks"
```

### Task 7: Add the Authenticated Creem Customer Portal

**Files:**
- Create: `packages/web-app/app/api/billing/portal/route.ts`
- Create: `packages/web-app/app/api/billing/portal/route.test.ts`

- [ ] **Step 1: Write failing authorization tests**

Test unauthenticated 401, non-Creem entitlement 409, missing customer ID 404, and a successful redirect. Ensure a client query such as `?customerId=cust_attacker` is ignored:

```ts
expect(mockGenerateBillingLinks).toHaveBeenCalledWith({ customerId: 'cust_from_database' })
expect(response.status).toBe(307)
expect(response.headers.get('location')).toBe('https://creem.io/portal/session_1')
```

- [ ] **Step 2: Run the portal tests and verify failure**

Run: `cd packages/web-app && npx vitest run app/api/billing/portal/route.test.ts`

Expected: FAIL because the route does not exist.

- [ ] **Step 3: Implement the portal route**

Authenticate with the Supabase cookie client. Read `source_billing_subscription_id` from the user's future effective entitlement, then read `external_customer_id` from that owned Creem billing record. Call:

```ts
const portal = await createCreemClient().customers.generateBillingLinks({ customerId })
if (!portal.customerPortalLink) {
  return Response.json({ success: false, error: 'PORTAL_URL_UNAVAILABLE' }, { status: 502 })
}
return Response.redirect(portal.customerPortalLink, 307)
```

Do not accept a customer ID from the request.

- [ ] **Step 4: Run the portal tests**

Run the Step 2 command.

Expected: all portal cases PASS.

- [ ] **Step 5: Commit the portal slice**

```bash
cd packages/web-app
git add app/api/billing/portal/route.ts app/api/billing/portal/route.test.ts
git commit -m "feat(billing): add authorized Creem portal"
```

### Task 8: Move WeChat Payments onto the Billing State Machine

**Files:**
- Modify: `packages/web-app/lib/wechat-pay/native.ts`
- Modify: `packages/web-app/lib/wechat-pay/orders.ts`
- Modify: `packages/web-app/lib/wechat-pay/payment-sync.ts`
- Modify: `packages/web-app/lib/wechat-pay/orders.test.ts`
- Modify: `packages/web-app/lib/wechat-pay/webhooks.ts`
- Modify: `packages/web-app/lib/wechat-pay/webhooks.test.ts`
- Modify: `packages/web-app/app/api/billing/wechat-pay/route.ts`
- Create: `packages/web-app/app/api/billing/wechat-pay/route.test.ts`

- [ ] **Step 1: Write failing renewal and RPC tests**

Add tests proving:

- a user without paid access may buy any WeChat plan;
- an active Creem or non-renewal-window WeChat entitlement gets HTTP 409 `ACTIVE_SUBSCRIPTION_EXISTS`;
- same-plan WeChat renewal is allowed only when `current_period_end <= now + 7 days`;
- renewal `subscription_period_end` starts from the existing end, not payment/order time;
- payment callback and order query both use event ID `wechat-payment:<transactionId>`;
- payment sync calls `process_billing_event` and no longer writes `user_subscriptions` or `payment_history` directly.

Key assertion:

```ts
expect(admin.rpc).toHaveBeenCalledWith('process_billing_event', expect.objectContaining({
  p_provider: 'wechat_pay',
  p_event_id: 'wechat-payment:wx-transaction-id',
  p_external_subscription_id: 'OMP-order-1',
  p_external_record_id: 'wx-transaction-id',
  p_currency: 'CNY',
}))
```

- [ ] **Step 2: Run focused WeChat tests and verify failure**

Run:

```bash
cd packages/web-app
npx vitest run lib/wechat-pay/orders.test.ts lib/wechat-pay/webhooks.test.ts app/api/billing/wechat-pay/route.test.ts
```

Expected: FAIL because current sync writes tables directly and does not enforce renewal windows.

- [ ] **Step 3: Implement purchase eligibility and stacked renewal periods**

Use `getPurchaseEligibility(client, userId, provider, plan, now)` from Task 3 with this result contract:

```ts
export type PurchaseEligibility =
  | { allowed: true; periodBase: Date }
  | { allowed: false; error: 'ACTIVE_SUBSCRIPTION_EXISTS' }
```

For no effective row, `periodBase = now`. For WeChat same-plan access ending within seven days, `periodBase = current_period_end`. Every other unexpired paid row is blocked.

Pass `periodBase` into `calculateSubscriptionPeriodEnd(interval, periodBase)`. Store the resulting period end on `wechat_pay_orders`. In `syncWechatPaymentSuccess`, read the order, normalize it to a billing event, and call `applyBillingEvent(createServiceRoleClient(), event)`.

- [ ] **Step 4: Run focused WeChat tests**

Run the Step 2 command.

Expected: all existing QR/query cases and new renewal/RPC cases PASS.

- [ ] **Step 5: Commit the WeChat payment slice**

```bash
cd packages/web-app
git add lib/billing/entitlements.ts lib/billing/entitlements.test.ts lib/wechat-pay/native.ts lib/wechat-pay/orders.ts lib/wechat-pay/payment-sync.ts lib/wechat-pay/orders.test.ts lib/wechat-pay/webhooks.ts lib/wechat-pay/webhooks.test.ts app/api/billing/wechat-pay/route.ts app/api/billing/wechat-pay/route.test.ts
git commit -m "refactor(billing): reconcile WeChat payments atomically"
```

### Task 9: Implement Refund Calculation and WeChat Refund Notifications

**Files:**
- Create: `packages/web-app/lib/billing/refunds.ts`
- Create: `packages/web-app/lib/billing/refunds.test.ts`
- Create: `packages/web-app/lib/wechat-pay/refunds.ts`
- Create: `packages/web-app/lib/wechat-pay/refunds.test.ts`
- Create: `packages/web-app/app/api/webhooks/wechat-refund/route.ts`
- Create: `packages/web-app/app/api/webhooks/wechat-refund/route.test.ts`

- [ ] **Step 1: Write failing refund tests**

```ts
import { describe, expect, it } from 'vitest'
import { calculateRefundMinorUnits, isFirstPurchaseRefundEligible } from './refunds'

describe('refund policy', () => {
  it('uses round-half-up and clamps the result', () => {
    expect(calculateRefundMinorUnits({ paidAmount: 900, used: 50, limit: 200 })).toBe(675)
    expect(calculateRefundMinorUnits({ paidAmount: 99, used: 1, limit: 200 })).toBe(99)
    expect(calculateRefundMinorUnits({ paidAmount: 900, used: 300, limit: 200 })).toBe(0)
  })

  it('allows only the first successful purchase within seven calendar days', () => {
    expect(isFirstPurchaseRefundEligible({ successfulPurchaseCount: 1, purchasedAt: '2026-07-10T00:00:00Z', requestedAt: '2026-07-13T00:00:00Z' })).toBe(true)
    expect(isFirstPurchaseRefundEligible({ successfulPurchaseCount: 1, purchasedAt: '2026-07-10T00:00:00Z', requestedAt: '2026-07-17T00:00:00Z' })).toBe(true)
    expect(isFirstPurchaseRefundEligible({ successfulPurchaseCount: 1, purchasedAt: '2026-07-10T00:00:00Z', requestedAt: '2026-07-17T00:00:00.001Z' })).toBe(false)
    expect(isFirstPurchaseRefundEligible({ successfulPurchaseCount: 2, purchasedAt: '2026-07-10T00:00:00Z', requestedAt: '2026-07-13T00:00:00Z' })).toBe(false)
  })
})
```

For WeChat, test signature failure, decryption, a `SUCCESS` refund mapped to event ID `wechat-refund:<out_refund_no>`, distinct refund ledger ID, and duplicate callback/query idempotency.

- [ ] **Step 2: Run refund tests and verify failure**

Run:

```bash
cd packages/web-app
npx vitest run lib/billing/refunds.test.ts lib/wechat-pay/refunds.test.ts app/api/webhooks/wechat-refund/route.test.ts
```

Expected: FAIL because the modules do not exist.

- [ ] **Step 3: Implement deterministic calculation and WeChat refund operations**

Use integer math for round-half-up:

```ts
export function calculateRefundMinorUnits(input: { paidAmount: number; used: number; limit: number }) {
  if (input.limit <= 0 || input.paidAmount <= 0) return 0
  const remaining = Math.max(input.limit - Math.max(input.used, 0), 0)
  return Math.min(Math.floor((input.paidAmount * remaining * 2 + input.limit) / (2 * input.limit)), input.paidAmount)
}

export function isFirstPurchaseRefundEligible(input: {
  successfulPurchaseCount: number
  purchasedAt: string
  requestedAt: string
}) {
  const purchasedAt = Date.parse(input.purchasedAt)
  const requestedAt = Date.parse(input.requestedAt)
  const elapsed = requestedAt - purchasedAt
  return input.successfulPurchaseCount === 1 && Number.isFinite(elapsed) && elapsed >= 0 && elapsed <= 168 * 60 * 60 * 1000
}
```

Implement `createWechatRefund` with `client.refunds({ out_trade_no, out_refund_no, reason, notify_url, amount: { refund, total, currency: 'CNY' } })` and `queryWechatRefund` with `client.find_refunds(outRefundNo)`.

The dedicated callback must verify the same four WeChat signature headers, decrypt the refund resource, and normalize successful refunds into `status: 'refunded'` plus a refund ledger record. It must not call the ordinary payment-notification parser.

- [ ] **Step 4: Run refund tests**

Run the Step 2 command.

Expected: calculator, provider mapping, signature, duplicate, and route tests PASS.

- [ ] **Step 5: Commit the refund slice**

```bash
cd packages/web-app
git add lib/billing/refunds.ts lib/billing/refunds.test.ts lib/wechat-pay/refunds.ts lib/wechat-pay/refunds.test.ts app/api/webhooks/wechat-refund/route.ts app/api/webhooks/wechat-refund/route.test.ts
git commit -m "feat(billing): add refund policy and WeChat callbacks"
```

### Task 10: Add Hourly Expiry and Provider Reconciliation

**Files:**
- Create: `packages/web-app/lib/creem/reconcile.ts`
- Create: `packages/web-app/lib/creem/reconcile.test.ts`
- Create: `packages/web-app/app/api/cron/billing-reconcile/route.ts`
- Create: `packages/web-app/app/api/cron/billing-reconcile/route.test.ts`
- Modify: `packages/web-app/vercel.json`

- [ ] **Step 1: Write failing reconciliation tests**

Test that the reconciler:

- expires elapsed WeChat records through the billing RPC;
- fetches Creem records whose `last_reconciled_at` is older than 24 hours;
- maps the fetched state through the same normalizer/mutation wrapper;
- uses deterministic event IDs containing subscription ID and provider `updatedAt`;
- checks open Creem disputes through the original transaction and updates the dispute ledger when the provider exposes a won/lost outcome;
- never extends a period after a Creem fetch failure;
- rejects cron requests without `Authorization: Bearer <CRON_SECRET>`.

- [ ] **Step 2: Run reconciliation tests and verify failure**

Run: `cd packages/web-app && npx vitest run lib/creem/reconcile.test.ts app/api/cron/billing-reconcile/route.test.ts`

Expected: FAIL because the reconciler and route do not exist.

- [ ] **Step 3: Implement bounded reconciliation**

Query at most 100 stale subscription rows and 100 open dispute rows per invocation. For Creem subscriptions call `createCreemClient().subscriptions.get(externalSubscriptionId)`; for open disputes call `createCreemClient().transactions.getById(originalExternalTransactionId)` and update only when the provider exposes a terminal result. For elapsed WeChat rows synthesize normalized `expired` events. Process each row independently and return counts:

```ts
type ReconcileResult = { checked: number; repaired: number; failed: number }
```

The route returns 401 for an invalid secret, 200 with `ReconcileResult` when all rows were checked, and 500 only when the job itself cannot initialize. Individual provider failures increment `failed` and emit identifier-only structured logs.

Add to `vercel.json`:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "git": { "deploymentEnabled": false },
  "crons": [{ "path": "/api/cron/billing-reconcile", "schedule": "0 * * * *" }]
}
```

- [ ] **Step 4: Run reconciliation tests**

Run the Step 2 command.

Expected: all reconciliation and cron authorization cases PASS.

- [ ] **Step 5: Commit the reconciliation slice**

```bash
cd packages/web-app
git add lib/creem/reconcile.ts lib/creem/reconcile.test.ts app/api/cron/billing-reconcile/route.ts app/api/cron/billing-reconcile/route.test.ts vercel.json
git commit -m "feat(billing): reconcile provider state hourly"
```

## Phase 2: Billing APIs, UI, and Compliance Surfaces

### Task 11: Expose Provider-Neutral Status and Ledger APIs

**Files:**
- Modify: `packages/web-app/types/dashboard.ts`
- Modify: `packages/web-app/app/api/billing/status/route.ts`
- Modify: `packages/web-app/app/api/billing/status/route.test.ts`
- Modify: `packages/web-app/app/api/billing/history/route.ts`
- Create: `packages/web-app/app/api/billing/history/route.test.ts`
- Modify: `packages/web-app/lib/supabase/queries.ts`
- Modify: `packages/web-app/components/dashboard/subscription/PaymentHistory.tsx`

- [ ] **Step 1: Write failing API contract tests**

The status response must expose:

```ts
type SubscriptionStatus = {
  plan: 'free' | 'pro' | 'team'
  status: 'inactive' | 'active' | 'scheduled_cancel' | 'past_due' | 'unpaid'
  provider: 'creem' | 'wechat_pay' | null
  interval: 'monthly' | 'yearly' | null
  currentPeriodEnd: string | null
  cancelAtPeriodEnd: boolean
  canManageCreem: boolean
  canRenewWechat: boolean
  optimizationQuota?: QuotaInfo
}
```

The history response must expose `recordType`, `externalId`, `originalTransactionId`, uppercase currency, and the expanded ledger status. Add tests that an elapsed active row returns the free response and that a partial refund remains a separate history item linked to its payment.

- [ ] **Step 2: Run the route tests and verify failure**

Run: `cd packages/web-app && npx vitest run app/api/billing/status/route.test.ts app/api/billing/history/route.test.ts`

Expected: FAIL on missing provider/renewal/ledger fields.

- [ ] **Step 3: Implement the stable response contracts**

Use `isEffectivePaidEntitlement` before returning paid access. Compute `canManageCreem` only for a future Creem entitlement. Compute `canRenewWechat` only for a same-plan WeChat entitlement ending within seven days. Update `getPaymentHistory` to select explicit ledger fields rather than `*`.

Render history direction from `recordType`: payment as a positive charge, refund as a returned amount, and dispute as a neutral status entry. Do not infer refund direction from a negative stored amount.

- [ ] **Step 4: Run API and rendering tests**

Run the Step 2 command.

Expected: route tests PASS.

- [ ] **Step 5: Commit the API contract slice**

```bash
cd packages/web-app
git add types/dashboard.ts app/api/billing/status/route.ts app/api/billing/status/route.test.ts app/api/billing/history/route.ts app/api/billing/history/route.test.ts lib/supabase/queries.ts components/dashboard/subscription/PaymentHistory.tsx
git commit -m "feat(billing): expose provider-neutral billing status"
```

### Task 12: Build Shared Bilingual Legal Content

**Revised by:** `docs/superpowers/plans/2026-07-14-remove-operator-identity.md`

- [x] **Step 1: Define typed bilingual policy content**
- [x] **Step 2: Render the shared Chinese and English legal routes**
- [x] **Step 3: Remove public operator name, address, and jurisdiction fields**
- [x] **Step 4: Keep the support email and payment disclosures public**
- [x] **Step 5: Verify all legal routes build without legal identity environment variables**

Public operator identity configuration and runtime validation were removed by owner decision. This does not establish live payment readiness; qualified legal and provider disclosure review remains pending.
The Terms do not duplicate current prices; their Service section directs users to the subscription page as the source of current plan details and prices.

### Task 13: Update Subscription UI for Both Providers and Locales

**Files:**
- Create: `packages/web-app/components/subscription/SubscriptionPage.tsx`
- Modify: `packages/web-app/app/subscription/page.tsx`
- Create: `packages/web-app/app/en/subscription/page.tsx`
- Modify: `packages/web-app/app/subscription/SubscriptionContent.tsx`
- Modify: `packages/web-app/components/billing/PaymentMethodSelector.tsx`
- Modify: `packages/web-app/components/billing/PaymentMethodModal.tsx`
- Modify: `packages/web-app/components/dashboard/subscription/PlanComparison.tsx`
- Modify: `packages/web-app/components/dashboard/subscription/CurrentPlan.tsx`
- Modify: `packages/web-app/components/layout/Header.tsx`
- Create: `packages/web-app/lib/billing/ui-state.ts`
- Create: `packages/web-app/lib/billing/ui-state.test.ts`
- Modify: `packages/web-app/tests/billing.spec.ts`

- [ ] **Step 1: Write failing state and browser tests**

Unit-test a pure pending-state reducer:

```ts
expect(resolveCheckoutBanner({ query: 'pending', status: 'inactive' })).toBe('processing')
expect(resolveCheckoutBanner({ query: 'pending', status: 'active' })).toBe('active')
expect(resolveCheckoutBanner({ query: null, status: 'scheduled_cancel' })).toBe('scheduled_cancel')
```

Expand Playwright coverage for:

- `/subscription` Chinese and `/en/subscription` English headings;
- exact CNY and USD prices;
- automatic renewal and manual renewal disclosures before checkout;
- language menu preserving `/subscription`;
- Terms and Refund links in the active locale;
- pending redirect never displaying “active” until mocked status becomes active;
- Creem portal action only for Creem status;
- WeChat renewal action only when `canRenewWechat` is true.

Use `page.addInitScript` to set `window.__SUPABASE_MOCK__` before navigation, returning a deterministic authenticated user from `auth.getSession()` and a no-op `onAuthStateChange()`. Mock `/api/billing/status`, `/api/billing/history`, `/api/billing/subscribe`, and `/api/billing/wechat-pay` with `page.route`.

- [ ] **Step 2: Run focused tests and verify failure**

Run:

```bash
cd packages/web-app
npx vitest run lib/billing/ui-state.test.ts
npx playwright test tests/billing.spec.ts
```

Expected: FAIL on missing English route, Creem option, disclosures, and state reducer.

- [ ] **Step 3: Implement locale-explicit shared subscription rendering**

Move the page implementation into `components/subscription/SubscriptionPage.tsx` with a required `locale` prop. Keep route wrappers minimal:

```tsx
export default function ChineseSubscriptionRoute() {
  return <SubscriptionPage locale="zh" />
}
```

Change `PaymentMethod` to `'creem' | 'wechat_pay'`. Render both choices without a feature flag. Pass both prices into the modal:

```ts
type PaymentPrices = {
  creem: { amount: number; currency: 'USD'; display: string }
  wechat_pay: { amount: number; currency: 'CNY'; display: string }
}
```

Show “automatically renews until canceled” for Creem and “one-time payment, renew manually” for WeChat next to their exact prices. Include locale-correct Terms and Refund links and require the confirmation action to state agreement.

- [ ] **Step 4: Implement checkout, pending polling, portal, and renewal actions**

For Creem POST only `{ plan, interval, locale }` to `/api/billing/subscribe`, then assign `window.location.href` to the returned URL. For WeChat retain QR behavior. On `?checkout=pending`, poll `/api/billing/status` every two seconds for up to 60 seconds; show processing until the verified API returns paid access. Remove the old `session_id` success logic and never infer activation from a query parameter.

Use a normal link to `/api/billing/portal` when `canManageCreem`. Enable WeChat checkout only when free or `canRenewWechat`; show the existing subscription message for all other paid users.

- [ ] **Step 5: Run UI tests and inspect both routes**

Run the Step 2 commands.

Expected: unit and browser tests PASS with no skipped billing tests.

- [ ] **Step 6: Commit the subscription UI slice**

```bash
cd packages/web-app
git add components/subscription/SubscriptionPage.tsx app/subscription/page.tsx app/en/subscription/page.tsx app/subscription/SubscriptionContent.tsx components/billing/PaymentMethodSelector.tsx components/billing/PaymentMethodModal.tsx components/dashboard/subscription/PlanComparison.tsx components/dashboard/subscription/CurrentPlan.tsx components/layout/Header.tsx lib/billing/ui-state.ts lib/billing/ui-state.test.ts tests/billing.spec.ts
git commit -m "feat(billing): add Creem and bilingual subscription UI"
```

## Phase 3: Removal, Verification, and Release Readiness

### Task 14: Remove Stripe and Update Active Documentation

**Files:**
- Delete: `packages/web-app/lib/stripe/client.ts`
- Delete: `packages/web-app/lib/stripe/checkout.ts`
- Delete: `packages/web-app/lib/stripe/webhooks.ts`
- Delete: `packages/web-app/lib/stripe/index.ts`
- Delete: `packages/web-app/app/api/webhooks/stripe/route.ts`
- Modify: `packages/web-app/package.json`
- Modify: `package-lock.json` (parent workspace lockfile)
- Modify: `packages/web-app/CLAUDE.md`
- Modify: `packages/web-app/README.md`
- Modify: `packages/web-app/.env.example`

- [ ] **Step 1: Add all new unit files to the unit-test script**

Keep every existing entry in `test:unit` and append exactly:

```text
lib/billing/plans.test.ts
lib/billing/entitlements.test.ts
lib/billing/mutations.test.ts
lib/billing/refunds.test.ts
lib/billing/locale.test.ts
lib/billing/ui-state.test.ts
lib/legal/content.test.ts
lib/creem/checkout.test.ts
lib/creem/webhooks.test.ts
lib/creem/reconcile.test.ts
lib/wechat-pay/refunds.test.ts
app/api/billing/subscribe/route.test.ts
app/api/billing/portal/route.test.ts
app/api/billing/wechat-pay/route.test.ts
app/api/billing/history/route.test.ts
app/api/webhooks/creem/route.test.ts
app/api/webhooks/wechat-refund/route.test.ts
app/api/cron/billing-reconcile/route.test.ts
```

- [ ] **Step 2: Remove Stripe dependency and source**

Run: `npm uninstall stripe --workspace=@oh-my-prompt/web-app`

Delete the Stripe library and webhook route. Remove Stripe environment variables, payment types, feature flags, comments, and current documentation. Update architecture and environment sections to describe Creem checkout/webhook/portal/reconciliation and dedicated WeChat refund callback.

- [ ] **Step 3: Search active files for forbidden references**

Run:

```bash
cd packages/web-app
rg -n -i 'stripe' app components lib types package.json .env.example README.md CLAUDE.md
```

Expected: no matches. Historical SQL migrations are intentionally excluded.

- [ ] **Step 4: Run unit, lint, and build checks**

Run:

```bash
cd packages/web-app
npm run test:unit
npm run lint
npm run build
```

Expected: all unit tests PASS, lint exits 0, and production build exits 0.

- [ ] **Step 5: Commit Stripe removal**

```bash
cd packages/web-app
git add package.json .env.example README.md CLAUDE.md
git add -u lib/stripe app/api/webhooks/stripe
git commit -m "chore(billing): remove Stripe integration"
```

### Task 15: Complete End-to-End, Security, and Deployment Verification

**Files:**
- Modify: `packages/web-app/tests/billing.spec.ts`
- Create: `packages/web-app/docs/billing-operations.md`
- Modify: `packages/web-app/README.md`
- Modify: `docs/superpowers/specs/2026-07-13-creem-wechat-billing-compliance-design.md`
- Modify: `docs/superpowers/plans/2026-07-13-creem-wechat-billing-compliance.md`

- [ ] **Step 1: Add the final browser and route abuse cases**

Add tests for client-injected product/customer/reference/success URL fields, duplicate webhook delivery, out-of-order webhook delivery, refund display, expired active status, and public access to all 12 scoped routes. The abuse route tests must assert the server-selected identifiers were used and upstream details were not returned.

- [ ] **Step 2: Write the operations runbook**

Document exact test/live environment variables, four product IDs, Creem webhook events, Creem webhook URL, WeChat payment and refund callback URLs, hourly cron, signature failure response, retry behavior, reconciliation observability, refund-case recording, and rollback procedure. The rollback procedure disables checkout first, leaves webhook/reconciliation enabled to converge existing records, and never restores Stripe.

- [ ] **Step 3: Run the complete verification matrix**

```bash
cd packages/web-app
npx supabase db reset
npx supabase test db
npm run test:unit
npm run lint
npm run build
npx playwright test tests/billing.spec.ts
if rg -n -i 'stripe' app components lib types package.json .env.example README.md CLAUDE.md; then
  echo 'Active Stripe references found'
  exit 1
fi
cd ../..
if rg -n '"stripe"' package-lock.json; then
  echo 'Stripe remains in the workspace lockfile'
  exit 1
fi
```

Expected:

- Supabase reset and pgTAP pass.
- Unit tests report zero failures.
- Lint and build exit 0.
- Billing Playwright tests report zero skipped or failed tests.
- Both guarded Stripe searches find no matches and the verification block exits 0.

- [ ] **Step 4: Perform live-readiness checks without creating live charges**

Verify that all production secrets exist in Vercel and qualified legal guidance confirms required public disclosures, webhook and refund callback URLs are reachable without authentication challenges, cron authentication works, support email receives a test message, policies are public, and the site displays the same four prices as the Creem dashboard. Do not enable live checkout until Creem account review and required disclosure review are complete.

- [ ] **Step 5: Mark plan/spec status and commit web-app documentation**

Update the spec status to `Implemented, pending live payment approval` only after Step 3 passes. Mark all executed plan checkboxes. Then:

```bash
cd packages/web-app
git add tests/billing.spec.ts docs/billing-operations.md README.md
git commit -m "docs(billing): add operations and release checks"
```

- [ ] **Step 6: Update the parent repository submodule pointer and planning docs**

```bash
cd ../..
git add package-lock.json packages/web-app docs/superpowers/specs/2026-07-13-creem-wechat-billing-compliance-design.md docs/superpowers/plans/2026-07-13-creem-wechat-billing-compliance.md
git commit -m "feat: integrate Creem and compliant billing"
```

## Completion Gate

Implementation is complete only when all Task 15 verification commands have fresh passing output, the active-source Stripe search is empty, required public disclosures are legally verified, and live checkout remains disabled until Creem approves the account. Public operator identity fields are intentionally omitted; live readiness remains pending.
