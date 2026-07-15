# Pro Monthly One-Dollar Price Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Change Pro monthly checkout and localized display from 9 to 1 while leaving every other plan price unchanged.

**Architecture:** Keep `PRICES.pro.monthly` as the trusted application source and update direct public/legal literals that do not consume it. Creem continues to use its configured Product ID, whose dashboard price must be synchronized before production deployment.

**Tech Stack:** Next.js, TypeScript, Vitest, Playwright

---

### Task 1: Lock the new price contract

**Files:**
- Modify: `packages/web-app/lib/billing/plans.test.ts`
- Modify: `packages/web-app/components/dashboard/subscription/PlanComparison.test.tsx`
- Modify: `packages/web-app/tests/billing.spec.ts`

- [ ] Change Pro monthly expectations from `900`, `$9`, and `¥9` to `100`, `$1`, and `¥1` while retaining assertions for all unchanged prices.
- [ ] Run `npm run test:unit -- --run` from `packages/web-app` and confirm the focused price assertions fail because production still returns 900.

### Task 2: Change the trusted price and public literals

**Files:**
- Modify: `packages/web-app/lib/billing/plans.ts`
- Modify: `packages/web-app/components/pages/HomePage.tsx`
- Modify: `packages/web-app/lib/legal/content.ts`
- Modify: `packages/web-app/lib/legal/content.test.ts`

- [ ] Set `PRICES.pro.monthly` to `100`.
- [ ] Render the landing Pro price as `¥1` for Chinese and `$1` for English.
- [ ] Change the English refund example from USD 9/refund USD 6.75 to USD 1/refund USD 0.75.
- [ ] Update legal static assertions so `$1` and `¥1` remain absent from Terms.

### Task 3: Verify and commit

**Files:**
- Verify all files above.

- [ ] Run focused Vitest price and legal tests.
- [ ] Run `npm run test:unit`.
- [ ] Run `npx playwright test tests/billing.spec.ts` against the local test server.
- [ ] Run `npm run build` and `git diff --check`.
- [ ] Commit the Web App change, then update and commit the parent submodule pointer.
