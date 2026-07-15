# Creem + WeChat Pay Billing and Compliance Design

**Date:** 2026-07-13
**Status:** Implemented locally, pending live payment approval and verified disclosure readiness

## 1. Objective

Replace Stripe completely with Creem for international subscription payments while retaining the existing WeChat Pay flow for customers paying in China.

The finished product must:

- sell Pro and Team subscriptions through Creem in USD;
- sell the same plans through WeChat Pay in CNY;
- distinguish Creem recurring subscriptions from WeChat Pay manual renewals;
- keep subscription access, payment history, cancellation, refunds, and disputes synchronized;
- prevent stale, duplicated, or out-of-order provider events from corrupting access;
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

Use the official `creem` TypeScript SDK from custom authenticated Next.js App Router routes alongside the existing WeChat Pay implementation. Use Creem's official webhook types and signature-verification utilities, but do not directly export the `Checkout`, `Portal`, or `Webhook` route helpers from `@creem_io/nextjs`.

The current adapter route helpers accept product, customer, reference, success URL, or portal customer identifiers from URL parameters and do not provide Supabase authorization. The current webhook helper also logs the complete `checkout.completed` event. Custom routes are therefore required to enforce server-owned identifiers and the logging rules in this design. Raw, unauthenticated REST calls and fixed payment links are not used.

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
4. The server rejects a cross-provider purchase while the user has an unexpired paid entitlement. A WeChat user may renew the same plan through WeChat during the final seven days of the current period; the renewed period starts at the later of payment time or the existing period end.
5. The server creates the checkout with the user's email and Supabase user ID as `referenceId`.
6. Creem hosts the payment form and acts as Merchant of Record for that transaction.
7. Creem redirects the browser to the locale-matching subscription page after checkout.
8. The UI treats the redirect as pending confirmation. Subscription access is granted only after a verified webhook updates the database.

The existing WeChat Native QR flow remains in place and continues to grant access from its verified payment callback or verified order query.

### 3.3 Subscription management

Authenticated Creem customers receive a **Manage international subscription** action in the subscription area. It opens a server-generated Creem Customer Portal URL where the customer can manage payment details and cancel.

Cancellation at period end preserves access until `current_period_end`. Expired, paused, refunded, or otherwise revoked subscriptions lose paid access according to verified provider events.

WeChat Pay remains non-recurring and therefore has no cancellation action. Its access expires at the end of the purchased period unless the customer manually renews.

### 3.4 Provider records and effective entitlement

Provider subscription records and effective user access are separate concerns:

- `billing_subscriptions` stores each Creem subscription or successful WeChat purchase independently and is updated only by verified provider flows;
- `user_subscriptions` stores one effective entitlement snapshot per user for existing application authorization and quota code;
- every provider event updates only its matching billing record, then recomputes the effective entitlement from all unexpired, non-revoked billing records;
- revoking or refunding one billing record cannot remove access supplied by another valid billing record;
- when multiple valid records exist because of retries or historical data, Team outranks Pro and a later period end breaks ties;
- this phase does not support paid-plan upgrades, downgrades, or cross-provider switching before expiry. The checkout returns `ACTIVE_SUBSCRIPTION_EXISTS` and directs Creem customers to the portal or WeChat customers to the renewal state.

The access-eligible normalized statuses are `active`, `scheduled_cancel`, `past_due`, and `unpaid`, always with non-null `last_paid_at` and `current_period_end > now()`. `subscription.active` maps to the ineligible internal `pending` status until `subscription.paid` records `last_paid_at` and confirms the first paid period. Past-due or unpaid records retain only an already-paid period and never extend it. `pending`, `canceled`, `paused`, `expired`, `refunded`, and `disputed` are ineligible. Status alone is never sufficient.

## 4. Creem Webhook Design

Add a Creem webhook route that reads the raw body and uses Creem's official signature-verification utility to verify the `creem-signature` header with `CREEM_WEBHOOK_SECRET`. Parse the event only after verification. The route does not use the adapter webhook helper because its current implementation logs the complete checkout event.

Handle these events:

- `checkout.completed`
- `subscription.active`
- `subscription.paid`
- `subscription.update`
- `subscription.scheduled_cancel`
- `subscription.canceled`
- `subscription.unpaid`
- `subscription.past_due`
- `subscription.paused`
- `subscription.expired`
- `refund.created`
- `dispute.created`

`subscription.paid` is the event that grants or renews access. `subscription.active` synchronizes provider state but does not grant access by itself. `subscription.scheduled_cancel` sets `cancel_at_period_end` while preserving access through the recorded period end. Paused, expired, refunded, and disputed records are ineligible for access. Past-due and unpaid behavior follows Creem's retry state and does not extend the last successfully paid period.

The handler validates that the event's product ID maps to an allowed plan and interval. Unknown products are recorded as ignored and do not grant access. User association comes from the trusted checkout `referenceId`; email alone is not sufficient to grant access.

### 4.1 Atomic idempotency and event ordering

A single service-role database RPC processes each verified event in one transaction:

1. Insert the provider, external event ID, event type, and provider creation time into `billing_events`.
2. If the unique event key already exists as processed, make no mutations and return success.
3. Lock and update only the matching `billing_subscriptions` record.
4. Ignore a lifecycle state mutation whose provider creation time is older than that record's `last_provider_event_at`, while retaining the event audit row. At equal timestamps, refund or dispute outranks expired, paused, or canceled; those terminal states outrank scheduled cancel, past due, unpaid, paid, and active.
5. Insert or update the associated payment, refund, or dispute ledger record regardless of lifecycle event ordering. A verified refund or dispute revokes its billing record even when delivered late; only a later provider-confirmed paid period or resolved dispute may restore it.
6. Recompute `user_subscriptions` from all currently valid billing records for that user.
7. Mark the event processed and commit all writes together.

An event is never marked processed in a separate transaction from its billing and entitlement mutations. The event table does not retain the complete webhook payload. Re-delivery of a processed event returns HTTP 200. Creem retries failed deliveries; the route returns non-2xx when a verified relevant event cannot commit.

### 4.2 Reconciliation and expiry

Webhook delivery is not the only expiry mechanism. All entitlement reads require `current_period_end > now()`, including billing status, cloud sync, official AI quota, team creation, and inherited Team access.

An hourly service-role reconciliation job:

- marks elapsed WeChat billing records expired and recomputes affected entitlements;
- queries Creem for locally active, past-due, paused, or scheduled-cancel subscriptions whose local state has not been confirmed in the last 24 hours;
- repairs provider state through the same transactional billing mutation path;
- records reconciliation failures without granting or extending access.

## 5. Data Model Migration

Add a new forward-only Supabase migration. Do not edit historical migrations.

### 5.1 `billing_subscriptions`

Add a service-owned provider-record table containing:

- `user_id`;
- `payment_provider` constrained to `creem | wechat_pay`;
- `external_subscription_id` for Creem or the merchant order ID for WeChat;
- nullable `external_customer_id` and `external_product_id`;
- provider-neutral `plan_type`, `billing_interval`, `current_period_start`, and `current_period_end`;
- normalized `status` constrained to `pending | active | scheduled_cancel | past_due | unpaid | canceled | paused | expired | refunded | disputed`;
- `cancel_at_period_end` with a default of `false`;
- `last_paid_at`, `last_provider_event_at`, `last_reconciled_at`, `created_at`, and `updated_at`.

Provider plus external subscription ID is unique. Creem subscription and customer IDs receive lookup indexes. Browser clients have read access only to records belonging to the authenticated user and no insert, update, or delete access.

Successful existing WeChat orders are backfilled into this table before effective entitlements are recomputed. Existing paid WeChat subscription rows must match a verified successful order; the migration fails rather than discarding a paid row that cannot be matched.

### 5.2 `user_subscriptions`

`user_subscriptions` becomes the one-row-per-user effective entitlement and quota snapshot:

- consolidate duplicate rows only after their provider facts and quota counters have been preserved, then add a unique constraint on `user_id`;
- change `payment_provider` to nullable `creem | wechat_pay`, indicating the billing record that currently wins entitlement calculation;
- add nullable `source_billing_subscription_id` referencing `billing_subscriptions`;
- retain `plan_type`, `status`, `billing_interval`, `current_period_start`, `current_period_end`, and quota fields;
- add `cancel_at_period_end` with a default of `false`;
- drop `stripe_subscription_id`, `stripe_customer_id`, and provider-specific subscription fields after backfill;
- remove the legacy `stripe` provider default so free entitlements have a null provider and source.

Legacy free-plan rows whose provider defaulted to `stripe` are changed to a null provider and retained. The migration must fail with an explicit error if it finds a paid legacy Stripe row. It must not delete or silently convert an unexpected paid subscription.

The migration also replaces every authorization query and the `has_active_team_subscription` RPC so paid access requires an eligible status and a future `current_period_end`.

### 5.3 `payment_history`

Evolve `payment_history` into an append-oriented billing ledger by adding:

- `payment_provider` constrained to `creem | wechat_pay`;
- `record_type` constrained to `payment | refund | dispute`;
- `external_record_id`, using the transaction, refund, or dispute ID for that record;
- nullable `original_external_transaction_id` for refunds and disputes;
- `status` constrained to `succeeded | failed | partially_refunded | refunded | dispute_open | dispute_won | dispute_lost`.

Provider, record type, and external record ID form the provider-originated uniqueness key for rows with a non-null external ID. A partial refund is a separate refund record and never overwrites the original payment. Amounts are non-negative integer minor units; `record_type` determines their accounting direction. Currency is normalized to uppercase `USD` or `CNY`. Existing history without a recoverable provider ID is retained as legacy data and excluded from the provider-originated uniqueness index.

The migration replaces the existing payment status check constraint and the legacy composite idempotency index with these ledger constraints and uniqueness rules.

### 5.4 `billing_events`

Add a service-owned event table containing:

- `payment_provider`;
- `external_event_id`;
- `event_type`;
- `provider_created_at`;
- `processing_status` constrained to `processed | ignored`;
- `processed_at`.

Provider and external event ID form the unique key. The table participates in the transactional RPC described in section 4.1; a failed transaction rolls back its event row so provider retry can process it again. Bounded failure codes are written to structured server logs. Browser clients receive no read, insert, update, or delete access.

### 5.5 `refund_cases`

Add a service-owned refund-decision table containing the user, original payment ledger record, request time, first-purchase eligibility, quota used and limit, calculated minor-unit amount, decision, and nullable provider refund ID. It stores the inputs needed to reproduce the decision without retaining support email contents. Browser clients receive no read, insert, update, or delete access.

## 6. Refund and Dispute Policy

### 6.1 Eligibility

A refund may be requested within seven calendar days of the user's first successful paid purchase across both providers. For deterministic cross-region handling, the window closes exactly 168 hours after the provider's successful-payment timestamp. Later purchases, including renewals and a purchase made after an earlier entitlement expired, are renewals for this rule. Renewals, requests after the window, partial unused time, and plan downgrades are not ordinarily refundable. Mandatory rights under applicable law take precedence.

### 6.2 Consumption-based calculation

For an eligible request:

```text
refundable amount = amount actually paid x
  (1 - official AI quota used in the current monthly quota period / monthly quota limit)
```

The result is clamped between zero and the amount actually paid and rounded to the nearest currency minor unit using round-half-up.

Examples:

- Pro monthly at USD 9 with 50 of 200 uses: USD 6.75 refundable.
- Pro yearly at USD 99 with 50 of 200 uses in the request month: USD 74.25 refundable.
- A fully consumed monthly quota produces a refundable amount of zero.

The annual purchase price is the refund base for an annual plan, while the current month's usage ratio is the consumption ratio. A successfully refunded subscription loses paid access immediately.

### 6.3 Operational flow

Customers request refunds at `support@oh-my-prompt.com` and include their account email and transaction reference. Oh My Prompt responds within three business days, verifies eligibility and quota use, calculates the amount, and executes the refund through the original channel.

The operator records the original transaction, whether it is the user's first paid purchase, quota used and limit at decision time, calculated minor-unit amount, decision, and resulting provider refund ID. This audit data is service-owned and is not exposed to other customers.

- Creem refunds are executed through Creem and the Creem subscription is canceled.
- WeChat refunds are executed through the WeChat merchant refund API with a unique merchant refund ID.
- Arrival time depends on the payment method and financial institution.

Creem may issue a refund within 60 days of purchase at its discretion to reduce chargeback risk. Creem handles disputes and chargebacks for transactions where it is Merchant of Record. A verified refund or dispute event updates the ledger, its billing record, and the effective entitlement atomically.

WeChat refund notifications use a dedicated verified callback route and the refund-notification resource schema; they are not parsed as ordinary payment notifications. Callback and merchant refund-query results enter the same idempotent billing mutation path. Any refund approved under this policy cancels and revokes the affected billing record immediately, including when quota use makes the refund amount partial. Another independently valid billing record may still supply access.

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

All legal pages are public and require no authentication. The public brand is **Oh My Prompt** and the public support address is `support@oh-my-prompt.com`.

Public operator name, address, and jurisdiction fields are intentionally not displayed. Production launch remains blocked until the owner obtains qualified legal guidance and verifies that the public policies, contracting-party disclosures, data-controller disclosures, Creem KYC identity, and WeChat merchant identity satisfy all applicable requirements. Creem's Merchant of Record role applies only to Creem transactions and does not replace Oh My Prompt's other legal obligations.

### 8.1 Privacy Policy

The policy must describe:

- account, prompt, synchronization, team, image, configuration, usage, and billing data;
- purposes and legal bases for processing;
- the confirmed data-controller identity and contact address;
- Supabase, object storage, configured AI providers, Creem, and WeChat Pay as relevant third parties;
- Creem's receipt of email, customer identifiers, order, transaction, and subscription data and its Merchant of Record role;
- the distinction between Creem USD transactions and WeChat Pay CNY transactions;
- international data processing, retention, user access/correction/export/deletion rights, security, and minors;
- that Oh My Prompt does not store full card details;
- the contact and deletion-request address `support@oh-my-prompt.com`.

The policy must not claim that third parties are controlled by Oh My Prompt or promise absolute security.

### 8.2 Terms of Service

The terms must describe:

- the service as prompt management, insertion, cloud synchronization, team sharing, image-to-prompt analysis, and text prompt generation;
- the confirmed contracting operator using the Oh My Prompt brand;
- which party contracts with the customer for Creem and WeChat transactions;
- a reference to the subscription page as the source of current plan details and prices, without duplicating specific amounts in the Terms;
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

The contact page and authenticated subscription area display `support@oh-my-prompt.com` and promise a response within three business days. Footer navigation links to Contact, Privacy, Terms, Refunds, and Acceptable Use in the active locale.

Creem Business Details and receipt settings must use the same support email.

## 9. Subscription UI Requirements

The subscription page must make these facts visible before checkout:

- the exact CNY and USD price for the selected plan and interval;
- Creem purchases renew automatically until canceled;
- WeChat purchases do not renew automatically;
- Creem is Merchant of Record for international transactions;
- purchase implies agreement to the active-locale Terms and Refund Policy.

After checkout, a redirect success indicator must not claim that access is active until the verified webhook state is visible. It may say that payment confirmation is being processed and refresh subscription status.

The Creem management action is visible only when the authenticated user's current effective entitlement comes from Creem. A WeChat renewal action is visible only for the same plan during the final seven days of its period. All other paid users see the existing subscription state without a second checkout action.

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
- Custom server routes ignore client-supplied product IDs, customer IDs, reference IDs, amounts, currencies, and success URLs.
- The `@creem_io/nextjs` Checkout, Portal, and Webhook route helpers are not exported directly.
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
- rejection of client-supplied product, customer, reference, amount, currency, and redirect identifiers;
- active-subscription purchase blocking and the final-seven-day WeChat renewal rule;
- customer portal authorization and missing-customer behavior;
- webhook signature failure;
- concurrent event idempotency and transaction rollback after each intermediate write;
- stale events not overwriting newer provider state;
- grant, scheduled cancellation, cancellation, failed payment, expiration, refund, and dispute state transitions;
- `subscription.active` not granting access before `subscription.paid`;
- recomputing effective access when two provider records exist and when either record is revoked;
- every entitlement consumer rejecting an elapsed `current_period_end`, even while status remains active;
- hourly reconciliation repairing missed provider events and expiring WeChat records;
- payment, partial-refund, full-refund, and dispute ledger records with distinct external IDs;
- verified WeChat refund callback and refund-query idempotency;
- refund calculation at zero, partial, full, and boundary usage;
- quota-based annual refund calculation;
- first-purchase refund eligibility across both providers;
- existing WeChat QR payment and verified order-query behavior remaining functional.

### 12.2 Browser tests

Cover:

- Chinese default routes and English `/en` routes;
- language-menu URL switching in both directions;
- public access to every legal page;
- locale-correct footer links;
- CNY and USD prices and renewal disclosures;
- Creem checkout and portal actions for authenticated users;
- WeChat QR flow presentation;
- pending confirmation, active, scheduled-cancel, renewal-eligible, expired, and cross-provider-blocked states.

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
4. Configure and observe the hourly billing reconciliation job in the production environment.
5. Register and verify the dedicated WeChat refund-notification URL.
6. Verify checkout, renewal, failed payment, scheduled cancellation, expiration, refund, dispute, duplicate, and out-of-order webhook behavior in test mode.
7. Confirm all Chinese and English policy URLs load without authentication.
8. Confirm with qualified legal guidance that all required contracting-party, data-controller, merchant, address, and jurisdiction disclosures are published where legally required and match the verified payment identities.
9. Confirm public pricing and the product description accurately match the live product.
10. Configure `support@oh-my-prompt.com` in Creem Business Details and receipt settings and verify that the mailbox works.
11. Complete individual KYC, country-of-tax-residence, and payout-account setup in Creem.
12. Confirm that the production site contains no fake testimonials, inflated customer counts, prohibited products, or misleading third-party affiliation claims.
13. Submit the live product URL for Creem account review.

Account review may require the individual operator to establish a clearer link between the brand and the KYC identity. Creem approval is an external review decision, not something the code can guarantee.

## 14. Out of Scope

- migrating existing Stripe customers, because none exist;
- automatic customer-initiated refunds;
- real-time exchange-rate conversion;
- localizing the rest of the web app;
- changing WeChat Pay from manual renewal to automatic renewal;
- paid-plan upgrades, downgrades, or cross-provider switching before the current entitlement expires;
- adding image or video generation;
- integrating Creem Moderation API while the product remains a text-prompt tool;
- rewriting historical SQL migrations or historical design documents.

## 15. Acceptance Criteria

The implementation is complete when:

- a logged-in user can start each of the four Creem subscriptions in test mode;
- verified Creem and WeChat events update provider records, access, and the billing ledger exactly once in one transaction;
- duplicate and out-of-order provider events cannot restore stale access or duplicate ledger entries;
- expired periods fail authorization without waiting for a provider webhook, and hourly reconciliation repairs missed state;
- revoking one provider record does not remove access supplied by another valid record;
- a Creem subscriber can open the Customer Portal and cancel without contacting support;
- authenticated checkout and portal routes cannot be used with client-selected product, customer, reference, amount, currency, or redirect identifiers;
- WeChat Pay still sells the four CNY options as manual renewals and verified refunds update the ledger and access;
- Stripe is absent from active code, dependencies, configuration, and current user-facing documentation;
- the confirmed refund formula is implemented as a tested calculation and documented in both languages;
- all six scoped pages work at both their default Chinese URL and `/en` English URL;
- the support email, prices, renewal behavior, legally required contracting-party disclosures, Merchant of Record disclosure, policies, and acceptable-use restrictions are public and internally consistent;
- targeted tests, lint, and production build pass.
