# Refund Usage Deduction Design

## Goal

Make the Chinese and English refund policies state the existing consumption-based refund rule clearly and consistently.

## Scope

- Update only the refund-policy entries in `packages/web-app/lib/legal/content.ts`.
- Keep the current page layout, refund window, eligibility rules, request process, payment-channel handling, and mandatory-law exception unchanged.
- Do not change the implemented refund calculation.

## Policy Wording

For an eligible first-purchase request, the refundable amount is:

```text
amount actually paid × (1 - official AI quota used in the current monthly quota period / monthly quota limit)
```

The calculation uses quota usage recorded when the refund request is reviewed. The result is clamped between zero and the amount actually paid, then rounded to the nearest currency minor unit using round-half-up.

For an annual plan, the annual amount actually paid remains the refund base. The usage ratio comes from the monthly quota period containing the refund request; unused annual months are not calculated separately.

Both locales will include equivalent monthly and annual examples. For example, using 50 of 200 monthly uses produces a 75% refund: USD 6.75 on a USD 9 monthly purchase and USD 74.25 on a USD 99 annual purchase. Fully consuming the monthly quota produces a refundable amount of zero.

## Verification

- Assert the Chinese and English legal content contains the consumption formula, annual-plan rule, and examples.
- Run the smallest relevant web-app tests and a production build or type check to ensure the legal pages still render.

