# Restore Pro Monthly 9.9 Price Design

## Goal

Restore the Pro monthly subscription price from 1 to 9.9 for both supported billing locales and deploy the synchronized result to production.

## Scope

- Set the canonical Pro monthly price to 990 minor currency units.
- Display `¥9.9` for Chinese and `$9.9` for English.
- Charge CNY 9.9 through WeChat Pay.
- Synchronize the production Creem Pro monthly product to USD 9.9 before exposing the new production UI.
- Update the landing page, refund examples, and price assertions.
- Keep Pro yearly and all Team prices unchanged.

## Architecture

`packages/web-app/lib/billing/plans.ts` remains the single application price source. Locale presentation formats its value, while the WeChat route resolves the trusted server-side amount from that source. Creem checkout continues to use the configured product ID, so its external product price must match the application display independently.

## Verification

- Focused unit tests prove the canonical amount, localized displays, WeChat plan amount, and refund examples.
- Billing browser tests assert `¥9.9` and `$9.9` on the localized subscription pages.
- The production build must pass before deployment.
- Vercel must report the deployment READY, production pages must render the new localized price, and recent runtime error logs must be clean.

## Deployment Safety

Do not deploy a `$9.9` display while the configured production Creem Pro monthly product charges a different amount. Deploy from an isolated `packages/web-app` worktree so unrelated local edits are excluded.
