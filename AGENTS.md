# Documentation project instructions

## About this project

- Public documentation for **Grantiva** (device attestation & intelligence for iOS apps), served at https://docs.grantiva.io
- Built on [Mintlify](https://mintlify.com): pages are MDX files with YAML frontmatter, navigation lives in `docs.json`
- Deploys automatically when changes land on the default branch (Mintlify GitHub app) — **merging is publishing**
- Run `mint dev` to preview locally, `mint broken-links` before every PR (requires Node LTS ≤ 24)

## Source of truth

The backend code wins over any existing docs text, marketing copy, or memory:

- **Tier limits, prices, quotas**: `Sources/App/Models/Tenant.swift` (`ServiceTier`) in the backend repo. Per-minute rate limits: `Sources/App/Services/TenantRateLimiter.swift`.
- **Endpoints and payloads**: the Vapor controllers in `Sources/App/Controllers/` — read the handler before documenting a field.
- **SDK behavior/versions**: the `grantiva/ios-sdk` repo's `CHANGELOG.md`.
- Tier display names: `basic` → **Pro**, `professional` → **Business**. Never expose raw enum values (`basic`, `growth`, `enterprise_plus`) in docs.

## Terminology

- "Monthly Active Devices (MAD)" — the billing unit; never "MAU"
- "attestation JWT" for the token the SDK receives; "API key" only for server-side keys
- Bundle ID + Team ID **identify** a tenant, they do not **authenticate** — most SDK endpoints also require the attestation JWT or an API key
- "sharing unit" / "Subject" for the subscription-claims family concept; distinct from user identity (`identify`)

## Style preferences

- Use active voice and second person ("you")
- Keep sentences concise — one idea per sentence
- Use sentence case for headings
- Bold for UI elements: Click **Settings**
- Code formatting for file names, commands, paths, and code references
- API examples use `https://api.grantiva.io` and realistic field values

## Content boundaries

- Don't document internal admin surfaces (`X-Admin-API-Key` endpoints, `/admin/v1/*`, Railway operations)
- Don't commit internal runbooks or infrastructure details (service names, deploy commands) to this public repo
- URLs the backend emits in emails/error hints must keep resolving: `/quickstart`, `/sdk/testing`, `/sdk/attestation#troubleshooting`, `/sdk/attestation#challenges` — add redirects in `docs.json` rather than breaking them
