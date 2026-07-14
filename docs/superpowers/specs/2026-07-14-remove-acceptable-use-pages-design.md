# Remove Acceptable Use Pages

## Goal

Remove the standalone Acceptable Use Policy pages in both supported locales and remove their footer navigation links.

## Scope

- Delete `/acceptable-use` and `/en/acceptable-use` page routes so they return the application's normal 404 response.
- Remove the Chinese and English Acceptable Use links from the shared footer.
- Remove page-only legal content and metadata entries that become unused.
- Update tests that currently expect these routes, links, content, or metadata.

The acceptable-use section inside the Terms of Service remains unchanged. Other legal pages and historical planning documents are outside this change.

## Implementation

Delete the two Next.js page modules. Remove `acceptableUse` from the legal page type/content model and metadata path/description maps, while retaining the `acceptable-use` section identifier used by the Terms of Service. Adjust footer and legal-page tests to match the reduced set of standalone policies.

## Verification

Run the smallest relevant web-app tests, type checking if configured, and a production web-app build. Confirm the two removed routes are no longer included in route expectations and that no live application code links to them.
