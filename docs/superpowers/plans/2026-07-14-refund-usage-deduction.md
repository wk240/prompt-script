# Refund Usage Deduction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make both public refund-policy locales clearly state that eligible refunds deduct the current monthly quota usage proportion.

**Architecture:** Keep `legalContent` as the single source rendered by both `/refund` and `/en/refund`. Add focused content assertions first, then replace only the conflicting refund calculation, annual-plan, example, and request-review wording; the existing refund calculator and page components remain unchanged.

**Tech Stack:** TypeScript, Next.js 16, Vitest

---

### Task 1: Lock the consumption-based policy contract

**Files:**
- Modify: `packages/web-app/lib/legal/content.test.ts`
- Test: `packages/web-app/lib/legal/content.test.ts`

- [ ] **Step 1: Write the failing bilingual content test**

Add this test inside the existing `describe('legalContent', ...)` block:

```ts
it('explains usage-based refund deductions in both locales', () => {
  const zhRefund = JSON.stringify(legalContent.zh.refund)
  const enRefund = JSON.stringify(legalContent.en.refund)

  for (const phrase of ['本月已使用的官方 AI 次数', '本月官方 AI 次数上限', '审核退款申请时', '四舍五入', 'USD 6.75', 'USD 74.25']) {
    expect(zhRefund).toContain(phrase)
  }
  for (const phrase of ['official AI quota used in the current monthly quota period', 'monthly quota limit', 'when the refund request is reviewed', 'round-half-up', 'USD 6.75', 'USD 74.25']) {
    expect(enRefund).toContain(phrase)
  }
  expect(zhRefund).not.toContain('我们不按使用量扣减')
  expect(enRefund).not.toContain('We do not deduct usage')
})
```

- [ ] **Step 2: Run the focused test and verify it fails**

Run:

```bash
npx vitest run lib/legal/content.test.ts
```

Run from `packages/web-app`. Expected: FAIL in `explains usage-based refund deductions in both locales` because the current policy says usage is not deducted.

- [ ] **Step 3: Commit the failing contract test**

```bash
git add lib/legal/content.test.ts
git commit -m "test: require usage-based refund disclosure"
```

Run from `packages/web-app` so the commit stays inside the web-app repository.

### Task 2: Align Chinese and English refund wording

**Files:**
- Modify: `packages/web-app/lib/legal/content.ts:52-64`
- Modify: `packages/web-app/lib/legal/content.ts:117-129`
- Test: `packages/web-app/lib/legal/content.test.ts`

- [ ] **Step 1: Replace the Chinese calculation, annual rule, example, and review wording**

Keep all section IDs and untouched policy sections as they are. Replace the four affected Chinese entries with:

```ts
s('calculation', '3. 退款计算', ['符合条件的可退金额 = 首次购买实付金额 ×（1 - 本月已使用的官方 AI 次数 ÷ 本月官方 AI 次数上限）。使用量以审核退款申请时系统记录的当前月度额度周期数据为准。计算结果最低为 ¥0、最高不超过实付金额，并按支付币种的最小单位四舍五入。']),
s('annual-rule', '4. 年付规则', ['年付方案以全年首次购买实付金额作为退款基数，并使用申请退款所在月度额度周期的使用比例进行扣减；未使用的剩余月份不另行按月折算。年付方案同样仅适用首次购买后的 168 小时退款窗口。']),
s('example', '5. 计算示例', ['若 Pro 月付实付 USD 9，本月 200 次额度已使用 50 次，可退 USD 6.75。若 Pro 年付实付 USD 99，申请退款所在月同样已使用 50/200 次，可退 USD 74.25。本月额度全部用完时，可退金额为 0。']),
s('request', '6. 申请信息', ['请提供账户邮箱、交易引用、购买时间、方案和申请原因。我们可能要求用于核验付款的必要资料，并会以审核退款申请时系统记录的本月官方 AI 使用量计算可退金额。']),
```

- [ ] **Step 2: Replace the matching English wording**

Keep all section IDs and untouched policy sections as they are. Replace the four affected English entries with:

```ts
s('calculation', '3. Calculation', ['Eligible refund = amount actually paid for the first purchase × (1 - official AI quota used in the current monthly quota period ÷ monthly quota limit). Usage is measured from the current monthly quota-period record when the refund request is reviewed. The result is clamped between zero and the amount actually paid, then rounded to the nearest currency minor unit using round-half-up.']),
s('annual-rule', '4. Annual plans', ['For an annual plan, the full amount actually paid for the first annual purchase is the refund base, and the usage ratio comes from the monthly quota period containing the refund request. Unused remaining months are not calculated separately. The same 168-hour first-purchase window applies.']),
s('example', '5. Calculation examples', ['If Pro monthly cost USD 9 and 50 of 200 monthly uses have been consumed, the refund is USD 6.75. If Pro yearly cost USD 99 and 50 of 200 uses have been consumed in the request month, the refund is USD 74.25. Fully consuming the monthly quota produces a refund of zero.']),
s('request', '6. Request details', ['Provide your account email, transaction reference, purchase time, plan, and reason. We may request information necessary to verify payment and calculate the refundable amount from the official AI usage recorded when the refund request is reviewed.']),
```

- [ ] **Step 3: Run the focused legal-content test**

Run:

```bash
npx vitest run lib/legal/content.test.ts
```

Run from `packages/web-app`. Expected: all tests in `lib/legal/content.test.ts` PASS.

- [ ] **Step 4: Run the existing refund calculator tests**

Run:

```bash
npx vitest run lib/billing/refunds.test.ts
```

Run from `packages/web-app`. Expected: all refund calculation and eligibility tests PASS, confirming the policy still matches the implemented calculation.

- [ ] **Step 5: Build the web app**

Run:

```bash
npm run build
```

Run from `packages/web-app`. Expected: Next.js production build exits with status 0 and includes `/refund` and `/en/refund` in the generated routes.

- [ ] **Step 6: Commit the policy update**

```bash
git add lib/legal/content.ts
git commit -m "fix: clarify usage-based refund policy"
```

Run from `packages/web-app`, then update the root repository's `packages/web-app` gitlink only if the user asks for root-level staging or committing.

