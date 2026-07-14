# Remove Public Operator Identity Design

**Date:** 2026-07-14
**Status:** Approved design

## Objective

Remove public operator name, address, and jurisdiction fields from the scoped legal pages. Legal pages must render without `LEGAL_OPERATOR_NAME`, `LEGAL_OPERATOR_ADDRESS`, or `LEGAL_JURISDICTION`.

## Runtime Changes

- Remove the `LegalIdentity` contract and `getLegalIdentity()` fail-closed loader.
- Remove the `identity` property and operator/jurisdiction rendering from `PolicyPage`.
- Stop passing legal identity data from all Chinese and English legal page routes.
- Remove operator-information sections and jurisdiction-dependent copy from the bilingual legal-content model.
- Keep `support@oh-my-prompt.com` as the public support, privacy, and legal contact.

## Configuration And Documentation

- Remove the three `LEGAL_OPERATOR_*` variables from `.env.example` and web-app setup documentation.
- Update current billing design, plan, and operations documentation so they do not instruct operators to configure or publish those fields.
- Keep production payment readiness explicitly pending. Removing these fields does not establish that Creem, WeChat Pay, consumer-law, privacy, or contracting-party disclosure requirements have been satisfied.

## Verification

- Unit tests assert that bilingual content has no operator-information section or operator identity labels.
- All scoped legal routes render without any `LEGAL_*` environment variables.
- Repository search finds no active `LEGAL_OPERATOR_*`, `getLegalIdentity`, `LEGAL_IDENTITY_NOT_CONFIGURED`, or public operator-field rendering references.
- The complete web-app unit suite, lint, and production build pass.
