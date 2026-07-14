# Terms Price Source Design

**Date:** 2026-07-14
**Status:** Approved design

## Objective

Make the subscription page the only user-facing source for current plan prices. The Terms of Service must not duplicate specific prices or contain a standalone pricing section.

## Content Changes

- Remove the Chinese `方案与价格` and English `Plans and prices` sections.
- Remove the four CNY/USD price statements from the Terms content.
- Add one sentence to the existing Service section stating that current plan details and prices are shown on the subscription page.
- Renumber all following Chinese and English Terms headings consecutively.
- Keep renewal, payment-channel, tax, invoice, delivery, refund, and support disclosures unchanged.

## Verification

- Legal-content tests assert that Terms has no `plans-prices` section and no duplicated plan price strings.
- Tests assert that both locales direct users to the subscription page for current plan details and prices.
- Subscription-page price tests remain unchanged and passing.
- The complete web-app unit suite and production build pass.
