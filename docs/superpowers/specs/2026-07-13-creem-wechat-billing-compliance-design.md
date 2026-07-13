# Creem + WeChat Pay Billing and Compliance Design

**Date:** 2026-07-13
**Status:** Implemented, pending live payment approval and verified production legal identity

## 1. Objective

Replace Stripe completely with Creem for international subscription payments while retaining the existing WeChat Pay flow for customers paying in China.

The finished product must:

- sell Pro and Team subscriptions through Creem in USD;
- sell the same plans through WeChat Pay in CNY;
- distinguish Creem recurring subscriptions from WeChat Pay manual renewals;
- keep subscription access, payment history, cancellation, refunds, and disputes synchronized;
- satisfy Creem's public website, customer support, and policy requirements;
- provide Chinese as the default language and English through `/en` URLs for payment and compliance pages;
- remove all Stripe code, configuration, dependencies, database fields, and user-facing references.

There are no production Stripe subscriptions to preserve or migrate.

## 2. Confirmed Commercial Rules

| Plan | WeChat Pay | Creem | Included monthly official AI quota |
| --- | ---: | ---: | ---: |
| Pro monthly | CNY 9 | USD 9 | 200 |
| Pro yearly | CNY 99 | USD 99 | 200 per month |
| Team monthly | CNY 49 | USD 49 | 1,000 |
| Team yearly | CNY 499 | USD 499 | 1,000 per month |

The values are deliberately the same in each currency. The application does not calculate or display exchange-rate conversions.

Creem purchases are recurring subscriptions. WeChat Pay purchases are one-time payments that grant access for the purchased period and do not renew automatically.

## 3. Selected Integration Approach

Use the official `@creem_io/nextjs` adapter alongside the existing WeChat Pay implementation.

This is preferred over raw REST calls because the adapter provides the supported Next.js App Router checkout, customer portal, signature verification, and typed webhook lifecycle. It is preferred over fixed payment links because the application must securely associate purchases with authenticated Supabase users and keep access state synchronized.

### 3.1 Creem products

Create four recurring Creem products in the Creem dashboard:

- Pro monthly: USD 9, every month
- Pro yearly: USD 99, every year
- Team monthly: USD 49, every month
- Team yearly: USD 499, every year

Their product IDs are server-only environment variables. A client submits only the allowed `plan` and `interval`; the server maps that pair to a product ID. The client cannot submit an amount, currency, or arbitrary Creem product ID.

### 3.2 Checkout flow

1. The user selects a plan and billing interval.
2. The payment method modal clearly presents:
   - WeChat Pay, priced in CNY and labeled as a manual renewal;
   - Creem international payment, priced in USD and labeled as automatically renewing.
3. A Creem request requires an authenticated user.
4. The server creates the checkout with the user's email and Supabase user ID as `referenceId`.
5. Creem hosts the payment form and acts as Merchant of Record for that transaction.
6. Creem redirects the browser to the locale-matching subscription page after checkout.
7. The UI treats the redirect as pending confirmation. Subscription access is granted only after a verified webhook updates the database.

The existing WeChat Native QR flow remains in place and continues to grant access from its verified payment callback or verified order query.

### 3.3 Subscription management

Authenticated Creem customers receive a **Manage international subscription** action in the subscription area. It opens a server-generated Creem Customer Portal URL where the customer can manage payment details and cancel.

Cancellation at period end preserves access until `current_period_end`. Expired, paused, refunded, or otherwise revoked subscriptions lose paid access according to verified provider events.

WeChat Pay remains non-recurring and therefore has no cancellation action. Its access expires at the end of the purchased period unless the customer manually renews.

## 4. Creem Webhook Design

Add a Creem webhook route that reads the raw body and uses the official adapter to verify the `creem-signature` header with `CREEM_WEBHOOK_SECRET`.

Handle these events:

- `subscription.active`
- `subscription.paid`
- `subscription.update`
- `subscription.scheduled_cancel`
- `subscription.canceled`
- `subscription.past_due`
- `subscription.paused`
- `subscription.expired`
- `refund.created`
- `dispute.created`

The handler must be idempotent. A lightweight billing-event table records provider, external event ID, event type, and processed time. It does not retain the complete webhook payload. Re-delivery of an already processed event returns HTTP 200 without repeating entitlement or payment-history writes.

The handler validates that the event's product ID maps to an allowed plan and interval. Unknown products are logged and do not grant access. User association comes from the trusted checkout `referenceId`; email alone is not sufficient to grant access.

Creem retries failed deliveries. The route returns a non-2xx response when a verified, relevant event cannot be persisted so a retry can repair state.

## 5. Data Model Migration

Add a new forward-only Supabase migration. Do not edit historical migrations.

### 5.1 `user_subscriptions`

- change the `payment_provider` constraint to `creem | wechat_pay`;
- remove the legacy `stripe` default so free subscriptions may have a null provider;
- add `creem_customer_id` with an index;
- add unique `creem_subscription_id`;
- add `creem_product_id`;
- add `cancel_at_period_end` with a default of `false`;
- retain provider-neutral `plan_type`, `status`, `billing_interval`, `current_period_start`, and `current_period_end`;
- retain `wechat_order_id`;
- drop `stripe_subscription_id` and `stripe_customer_id`.

Legacy free-plan rows whose provider defaulted to `stripe` are changed to a null provider and retained. The migration must fail with an explicit error if it finds a paid legacy Stripe row, despite the owner's confirmation that none exist. It must not delete or silently convert an unexpected paid subscription.

### 5.2 `payment_history`

Add:

- `payment_provider` constrained to `creem | wechat_pay`;
- `external_transaction_id`;
- a provider + external transaction ID uniqueness rule for provider-originated transactions.

Amounts remain integer minor units. Currency is normalized to uppercase `USD` or `CNY` for new entries. Existing WeChat history is retained and normalized where safe.

### 5.3 Billing event idempotency

Add a service-owned event table containing:

- `payment_provider`;
- `external_event_id`;
- `event_type`;
- `processed_at`.

The provider and external event ID form the unique key. Browser clients receive no insert or update access.

## 6. Refund and Dispute Policy

### 6.1 Eligibility

A refund may be requested within seven calendar days of the first subscription purchase. Renewals, requests after seven days, partial unused time, and plan downgrades are not ordinarily refundable. Mandatory rights under applicable law take precedence.

### 6.2 Consumption-based calculation

For an eligible request:

```text
refundable amount = amount actually paid x
  (1 - official AI quota used in the current monthly quota period / monthly quota limit)
```

The result is clamped between zero and the amount actually paid and rounded to the nearest currency minor unit.

Examples:

- Pro monthly at USD 9 with 50 of 200 uses: USD 6.75 refundable.
- Pro yearly at USD 99 with 50 of 200 uses in the request month: USD 74.25 refundable.
- A fully consumed monthly quota produces a refundable amount of zero.

The annual purchase price is the refund base for an annual plan, while the current month's usage ratio is the consumption ratio. A successfully refunded subscription loses paid access immediately.

### 6.3 Operational flow

Customers request refunds at `support@ohmyprompt.com` and include their account email and transaction reference. Oh My Prompt responds within three business days, verifies eligibility and quota use, calculates the amount, and executes the refund through the original channel.

- Creem refunds are executed through Creem and the Creem subscription is canceled.
- WeChat refunds are executed through the WeChat merchant refund process.
- Arrival time depends on the payment method and financial institution.

Creem may issue a refund within 60 days of purchase at its discretion to reduce chargeback risk. Creem handles disputes and chargebacks for transactions where it is Merchant of Record. A verified refund or dispute event updates payment history and subscription access.

This phase does not add an automated customer refund endpoint.

## 7. Locale and URL Design

This phase localizes only payment and compliance surfaces.

Chinese is the default and has no path prefix:

- `/subscription`
- `/privacy`
- `/terms`
- `/refund`
- `/acceptable-use`
- `/contact`

English uses `/en`:

- `/en/subscription`
- `/en/privacy`
- `/en/terms`
- `/en/refund`
- `/en/acceptable-use`
- `/en/contact`

Every route renders exactly one language. A language menu replaces the locale portion of the current URL. Chinese routes always render Chinese and English routes always render English; there is no browser-language redirect and no query-parameter locale.

Shared typed translation resources hold the content for both locales. Shared page components render those resources so structure, links, prices, and policy sections cannot drift. The payment modal and subscription components receive locale explicitly from the route rather than reading global browser state.

The rest of the web app remains Chinese in this phase.

## 8. Compliance Content

All legal pages are public and require no authentication. The public support identity is the individually operated brand **Oh My Prompt**. The public support address is `support@ohmyprompt.com`. The legal name used for Creem KYC remains private unless Creem account review requires a stronger public identity link.

### 8.1 Privacy Policy

The policy must describe:

- account, prompt, synchronization, team, image, configuration, usage, and billing data;
- purposes and legal bases for processing;
- Supabase, object storage, configured AI providers, Creem, and WeChat Pay as relevant third parties;
- Creem's receipt of email, customer identifiers, order, transaction, and subscription data and its Merchant of Record role;
- the distinction between Creem USD transactions and WeChat Pay CNY transactions;
- international data processing, retention, user access/correction/export/deletion rights, security, and minors;
- that Oh My Prompt does not store full card details;
- the contact and deletion-request address `support@ohmyprompt.com`.

The policy must not claim that third parties are controlled by Oh My Prompt or promise absolute security.

### 8.2 Terms of Service

The terms must describe:

- the service as prompt management, insertion, cloud synchronization, team sharing, image-to-prompt analysis, and text prompt generation;
- the individual operator using the Oh My Prompt brand;
- plan entitlements and visible CNY/USD prices;
- Creem automatic renewal, cancellation, tax, invoice, and Merchant of Record terms;
- WeChat Pay's one-time purchase and manual renewal terms;
- digital service delivery and entitlement timing;
- user content ownership, required rights, acceptable use, account suspension, and termination;
- third-party AI and host-platform dependencies;
- that references to supported platforms or AI models do not imply affiliation, endorsement, or partnership;
- limitations of liability, service changes, governing-law limitations, policy updates, and support.

### 8.3 Refund Policy

The refund page states the eligibility window, exclusions, calculation formula, annual-plan rule, worked example, response time, channel-specific handling, immediate access termination after refund, Creem's 60-day discretion, and mandatory-law exception.

### 8.4 Acceptable Use Policy

Add a public policy prohibiting:

- pornography, NSFW or sexually exploitative content;
- face swaps, deepfakes, impersonation, and deceptive media;
- illegal, fraudulent, harassing, hateful, or violent activity;
- malware, credential theft, surveillance, or unauthorized access;
- intellectual-property infringement and unlicensed content;
- prohibited or regulated goods and services;
- evasion of third-party platform rules or use that violates those platforms' terms;
- abuse of service limits or payment systems.

Oh My Prompt generates and manages text prompts; it does not generate images or videos. Creem's Moderation API is therefore not required by Creem's current AI image/video rule. If the product later generates images or videos from user prompts, production launch of that capability requires a new compliance review and Creem Moderation API integration.

### 8.5 Contact and discoverability

The contact page and authenticated subscription area display `support@ohmyprompt.com` and promise a response within three business days. Footer navigation links to Contact, Privacy, Terms, Refunds, and Acceptable Use in the active locale.

Creem Business Details and receipt settings must use the same support email.

## 9. Subscription UI Requirements

The subscription page must make these facts visible before checkout:

- the exact CNY and USD price for the selected plan and interval;
- Creem purchases renew automatically until canceled;
- WeChat purchases do not renew automatically;
- Creem is Merchant of Record for international transactions;
- purchase implies agreement to the active-locale Terms and Refund Policy.

After checkout, a redirect success indicator must not claim that access is active until the verified webhook state is visible. It may say that payment confirmation is being processed and refresh subscription status.

The Creem management action is visible only when the authenticated user's current subscription provider is Creem. WeChat users receive a renewal action when appropriate.

## 10. Stripe Removal

Remove:

- the `stripe` npm dependency and lockfile entries;
- `lib/stripe`;
- the Stripe webhook route;
- Stripe-specific checkout implementation;
- Stripe environment variables and product IDs;
- Stripe payment-method types, feature flags, icons, comments, tests, and user-facing copy;
- Stripe-specific documentation in the web-app repository.

The existing billing subscribe endpoint may be replaced by the Creem checkout route or retained as a provider-neutral authenticated endpoint. It must not retain Stripe behavior or naming.

Historical design documents and already-applied SQL migration files remain unchanged because they describe prior architecture and database history.

## 11. Error Handling and Security

- Checkout and portal creation require an authenticated Supabase user.
- Product mapping, prices, and currencies are server-controlled.
- API keys and webhook secrets are server-only environment variables.
- Webhook handlers verify raw request signatures before parsing or mutation.
- Subscription writes use the service-role client and are not writable by browser clients.
- Unknown products, missing user references, invalid signatures, and inconsistent event data never grant access.
- Provider APIs return stable application error codes rather than leaking upstream details or secrets.
- Logging includes useful event and trace identifiers but excludes secrets, full payment data, and complete webhook payloads.
- Test mode and production mode use separate Creem keys and product IDs.

## 12. Verification Plan

### 12.1 Unit and route tests

Cover:

- each allowed plan/interval to Creem product mapping;
- rejection of invalid plans, intervals, and unauthenticated requests;
- checkout reference ID, email, success URL, and server-selected product;
- customer portal authorization and missing-customer behavior;
- webhook signature failure;
- event idempotency;
- grant, scheduled cancellation, cancellation, failed payment, expiration, refund, and dispute state transitions;
- payment-history amount, currency, provider, and external ID;
- refund calculation at zero, partial, full, and boundary usage;
- quota-based annual refund calculation;
- WeChat behavior remaining unchanged.

### 12.2 Browser tests

Cover:

- Chinese default routes and English `/en` routes;
- language-menu URL switching in both directions;
- public access to every legal page;
- locale-correct footer links;
- CNY and USD prices and renewal disclosures;
- Creem checkout and portal actions for authenticated users;
- WeChat QR flow presentation;
- pending confirmation and active subscription states.

### 12.3 Project checks

Run the smallest relevant tests during development, then before completion run:

```bash
npm run test:unit
npm run lint
npm run build
npx playwright test tests/billing.spec.ts
```

Also search the active web-app source, package manifest, environment documentation, and user-facing pages for remaining `Stripe` references. Historical migrations and historical design documents are excluded from that zero-reference requirement.

## 13. Deployment and Creem Review Checklist

Before live payments:

1. Create the four recurring Creem products with the confirmed USD prices.
2. Configure separate test and live API keys, webhook secrets, and product IDs.
3. Register the production webhook URL and subscribe to the required events.
4. Verify checkout, renewal, failed payment, scheduled cancellation, expiration, refund, and duplicate webhook behavior in test mode.
5. Confirm all Chinese and English policy URLs load without authentication.
6. Confirm public pricing and the product description accurately match the live product.
7. Configure `support@ohmyprompt.com` in Creem Business Details and receipt settings and verify that the mailbox works.
8. Complete individual KYC, country-of-tax-residence, and payout-account setup in Creem.
9. Confirm that the production site contains no fake testimonials, inflated customer counts, prohibited products, or misleading third-party affiliation claims.
10. Submit the live product URL for Creem account review.

Account review may require the individual operator to establish a clearer link between the brand and the KYC identity. That is an external review decision, not something the code can guarantee.

## 14. Out of Scope

- migrating existing Stripe customers, because none exist;
- automatic customer-initiated refunds;
- real-time exchange-rate conversion;
- localizing the rest of the web app;
- changing WeChat Pay from manual renewal to automatic renewal;
- adding image or video generation;
- integrating Creem Moderation API while the product remains a text-prompt tool;
- rewriting historical SQL migrations or historical design documents.

## 15. Acceptance Criteria

The implementation is complete when:

- a logged-in user can start each of the four Creem subscriptions in test mode;
- verified Creem events update access and payment history exactly once;
- a Creem subscriber can open the Customer Portal and cancel without contacting support;
- WeChat Pay still sells the four CNY options as manual renewals;
- Stripe is absent from active code, dependencies, configuration, and current user-facing documentation;
- the confirmed refund formula is implemented as a tested calculation and documented in both languages;
- all six scoped pages work at both their default Chinese URL and `/en` English URL;
- support email, prices, renewal behavior, Merchant of Record disclosure, policies, and acceptable-use restrictions are public and internally consistent;
- targeted tests, lint, and production build pass.
