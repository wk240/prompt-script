# Pro Monthly One-Dollar Price Design

## Goal

Change the Pro monthly subscription price from 900 to 100 minor currency units. The localized checkout rule remains unchanged: English uses Creem and displays USD `$1`; Chinese uses WeChat Pay and displays CNY `¥1`.

## Scope

- Change only the Pro monthly price.
- Keep Pro yearly, Team monthly, and Team yearly prices unchanged.
- Use 100 for UI presentation, trusted server-side price lookup, and WeChat order creation.
- Update public price assertions and the legal refund example that names the Pro monthly price.
- Keep the existing Creem Product ID contract. The matching Creem production Product must charge USD $1 before deployment.

## Data Flow

`PRICES.pro.monthly` remains the single application price source. Locale presentation formats that value as `$1` or `¥1`. WeChat resolves the trusted plan price on the server and submits 100 CNY cents. Creem receives the existing trusted Product ID, so its dashboard price must be synchronized separately.

## Safety And Verification

- Tests must prove both localized displays use 1, not 9.
- Plan tests must prove the trusted Pro monthly amount is 100.
- Billing browser tests must prove the Chinese QR order reports amount 100.
- Other plan prices must remain unchanged.
- Unit, billing browser, internationalization, and production build checks must pass before deployment.

## Deployment Constraint

Do not deploy the price change until the production Creem Pro monthly Product is confirmed at USD $1. A UI/provider mismatch would charge a different amount than the displayed price.
