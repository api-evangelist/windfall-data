---
name: Enrich a person record with Windfall
description: >-
  Submit a person record (name, address, email, phone, company) to the Windfall API and
  interpret the enriched household net worth and career data returned in real time.
api: openapi/windfall-data-openapi-original.json
operations: [enrichRecord]
---

# Enrich a person record with Windfall

Use the Windfall API to enrich a single person record with household wealth and career
intelligence in real time.

## Auth

- Send your token in the `X-WF-Auth-Token` header on every request. HTTPS only.
- Production base URL: `https://api.windfalldata.com/v1`
- Sandbox base URL: `https://api.windfalldata.com/sandbox/v1` (tokens start with `sandbox_`;
  a production token on the sandbox endpoint — or vice versa — returns `403`).

## Steps

1. Build the request body for `enrichRecord` (`POST /`). Provide as many PII fields as you
   have: `first_name`, `last_name`, `addresses[]` (`address`, `city`, `state`, `zipcode`),
   `emails[]`, `phones[]`, `company_name`. Pass your own `id` to correlate the response.
2. To get a match, satisfy at least one combination: an email alone; a full street address
   plus zipcode; or phone plus first and last name.
3. `POST` the body to `/`. Expect a sub-second JSON response.
4. Read the response: `household_matched` and `career_matched` booleans tell you what was
   found. When matched, `household` carries `net_worth`, `windfall_id`, and a `confidence`
   score; `career` carries `linkedin_url` and `confidence`. Your submitted `id` is echoed.

## Rules

- Rate limit is 5 requests/second; on `429` (`rate_limit`) back off and retry after a brief delay.
- `400` = malformed JSON; `401` = invalid/missing token. There is no idempotency key —
  enrichment is a current-state lookup, so re-requesting is safe but not deduplicated.
- Validate integrations against the sandbox personas (e.g. Amanda Taylor, $1M, Software
  Engineer) before going live; sandbox requests consume no credits.
- Handle partial matches: `household` and `career` are independent and either may be absent.

## References

- Conventions: conventions/windfall-data-conventions.yml
- Errors: errors/windfall-data-problem-types.yml
- Sandbox: sandbox/windfall-data-sandbox.yml
