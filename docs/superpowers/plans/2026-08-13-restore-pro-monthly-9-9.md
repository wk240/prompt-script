# Restore Pro Monthly 9.9 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore Pro monthly to 9.9 in both currencies and deploy the synchronized web app to Vercel production.

**Architecture:** Keep `PRICES.pro.monthly` as the trusted amount shared by localized presentation and WeChat order creation. Update direct landing/legal literals and tests, then synchronize the external Creem product before deploying the isolated web-app worktree.

**Tech Stack:** Next.js 16, TypeScript, Vitest, Playwright, Creem, WeChat Pay, Vercel

---

### Task 1: Establish an isolated baseline

**Files:**
- No production files modified

- [ ] **Step 1: Create an isolated web-app worktree**

Run `git worktree add <isolated-path> -b codex/restore-pro-price-9-9 d4dd2d1` from `packages/web-app`.

- [ ] **Step 2: Install dependencies and verify current billing tests**

Run `npm install`, then:

```bash
npx vitest run lib/billing/plans.test.ts lib/billing/payment-locale.test.ts components/dashboard/subscription/PlanComparison.test.tsx lib/legal/content.test.ts
```

Expected: all baseline tests pass.

### Task 2: Drive the price restoration through tests

**Files:**
- Modify: `packages/web-app/lib/billing/plans.test.ts`
- Modify: `packages/web-app/components/dashboard/subscription/PlanComparison.test.tsx`
- Modify: `packages/web-app/lib/legal/content.test.ts`
- Modify: `packages/web-app/tests/billing.spec.ts`

- [ ] **Step 1: Change price expectations before production code**

Assert `990` minor units, localized `¥9.9` / `$9.9`, refund results of `7.43`, and browser-visible localized prices.

- [ ] **Step 2: Verify RED**

Run the focused Vitest command. Expected: failures show the application still returns `100`, `¥1`, `$1`, and the old refund example.

### Task 3: Implement the canonical and direct-literal changes

**Files:**
- Modify: `packages/web-app/lib/billing/plans.ts`
- Modify: `packages/web-app/components/pages/HomePage.tsx`
- Modify: `packages/web-app/lib/legal/content.ts`

- [ ] **Step 1: Set the trusted price**

Change `PRICES.pro.monthly` from `100` to `990`.

- [ ] **Step 2: Update landing and refund literals**

Render `¥9.9` / `$9.9` on the landing page. Update the monthly refund example to a 9.9 base and a 7.43 rounded result in each locale.

- [ ] **Step 3: Verify GREEN**

Run the focused Vitest command. Expected: all focused tests pass.

### Task 4: Synchronize billing and deploy production

**Files:**
- External: production Creem Pro monthly product
- External: Vercel production deployment

- [ ] **Step 1: Verify billing configuration**

Confirm the linked Vercel project is `oh-my-prompt-web-app`, its production domain is `oh-my-prompt.com`, and the configured production Creem Pro monthly product is or can be set to USD 9.9.

- [ ] **Step 2: Run full preflight**

Run the web-app unit suite and `npm run build`. Expected: exit code 0.

- [ ] **Step 3: Deploy from the isolated web-app directory**

Run `vercel --cwd . --prod`. Expected: deployment reaches READY.

- [ ] **Step 4: Verify production**

Inspect the deployment, scan recent error logs, and verify `/subscription` shows `¥9.9` while `/en/subscription` shows `$9.9`.

