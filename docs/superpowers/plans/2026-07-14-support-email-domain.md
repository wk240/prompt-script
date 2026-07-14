# Support Email Domain Update Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the legacy undashed support domain with `support@oh-my-prompt.com` everywhere.

**Architecture:** Keep the support address as the existing server-owned constant in `lib/legal/identity.ts` and make all bilingual legal copy, tests, and current billing documentation agree with it. This is a literal identity update with no new configuration or abstraction.

**Tech Stack:** TypeScript, Next.js, Vitest, Playwright, Markdown

---

### Task 1: Update The Support Address Everywhere

**Files:**
- Modify: `packages/web-app/lib/legal/identity.ts`
- Modify: `packages/web-app/lib/legal/content.ts`
- Modify: `packages/web-app/lib/legal/content.test.ts`
- Modify: `packages/web-app/tests/landing.spec.ts`
- Modify: `docs/superpowers/specs/2026-07-13-creem-wechat-billing-compliance-design.md`
- Modify: `docs/superpowers/plans/2026-07-13-creem-wechat-billing-compliance.md`

- [x] **Step 1: Update test expectations first**

Replace the expected address in `packages/web-app/lib/legal/content.test.ts` and `packages/web-app/tests/landing.spec.ts`:

```ts
'support@oh-my-prompt.com'
```

- [x] **Step 2: Run the focused test and verify it fails**

Run:

```bash
cd packages/web-app
npx vitest run lib/legal/content.test.ts
```

Expected: FAIL because runtime legal content still contains the legacy undashed address.

- [x] **Step 3: Update runtime and documentation references**

Replace every exact legacy support-address occurrence in the scoped runtime and current documentation files with:

```text
support@oh-my-prompt.com
```

Do not change `LEGAL_OPERATOR_NAME`, `LEGAL_OPERATOR_ADDRESS`, or `LEGAL_JURISDICTION`.

- [x] **Step 4: Run focused and complete verification**

Run:

```bash
cd packages/web-app
npx vitest run lib/legal/content.test.ts
npm run test:unit
cd ../..
rg -n --hidden --glob '!node_modules' --glob '!.next' 'support@ohmyprompt\.com' .
git diff --check
```

Expected: focused tests pass, all unit tests pass, `rg` prints no matches, and `git diff --check` exits successfully.

- [ ] **Step 5: Commit only the email update**

```bash
git add docs/superpowers/specs/2026-07-13-creem-wechat-billing-compliance-design.md \
  docs/superpowers/plans/2026-07-13-creem-wechat-billing-compliance.md \
  packages/web-app/lib/legal/identity.ts \
  packages/web-app/lib/legal/content.ts \
  packages/web-app/lib/legal/content.test.ts \
  packages/web-app/tests/landing.spec.ts
git commit -m "fix: update support email domain"
```
