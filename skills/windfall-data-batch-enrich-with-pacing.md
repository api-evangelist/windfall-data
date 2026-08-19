---
name: Enrich a list of records with Windfall while respecting rate limits and credits
description: >-
  Walk a list of person records through the Windfall API one at a time — the API has no batch
  endpoint — pacing against the published 5 requests/second ceiling, retrying 429s, and
  tracking usage-token consumption so a run does not silently exhaust the account's contractual
  credit allocation.
api: openapi/windfall-data-windfall-api-api-openapi.yml
operations: [enrichRecord]
---

# Enrich a list of records with Windfall while respecting rate limits and credits

Windfall exposes exactly one operation, `enrichRecord` (`POST /`), and it processes **one record
per request** — there is no batch endpoint. Enriching a list therefore means a paced loop, and
the two things that break that loop are the per-second cap and the account's credit quota.

## Before you start

- Auth: `X-WF-Auth-Token` header on every request, HTTPS only.
- Production base URL: `https://api.windfalldata.com/v1`
- Sandbox base URL: `https://api.windfalldata.com/sandbox/v1` — tokens prefixed `sandbox_`.
  Sandbox requests **do not consume credits**; run the whole loop there first.
- Each record you query consumes one usage token from the allocation set by your purchase
  order. There is no API call that reports remaining balance — track it yourself.

## Steps

1. Pre-filter the list. A record with no name and no address, email or phone will burn a token
   and match nothing. Keep records that satisfy at least one of: email alone; street address
   plus zipcode; or phone plus `first_name` and `last_name`.
2. Assign every record a stable `id` from your system of record. `enrichRecord` echoes `id`
   back on the response — that echo is the only correlation mechanism the API provides.
3. Loop the records sequentially, calling `enrichRecord` (`POST /`) with `first_name`,
   `last_name`, `addresses[]`, `emails[]`, `phones[]`, `company_name`.
4. Pace the loop. The documented ceiling is 5 requests/second; Windfall's own published example
   sleeps **0.2 seconds** between calls. Do not parallelize past the cap — there is no burst
   allowance documented.
5. On `429` (`rate_limit`), sleep **1 second** and retry the same record, then resume pacing.
   Windfall returns no `Retry-After` or `RateLimit-*` headers, so a fixed backoff is all you have.
6. On each `200`, branch on `household_matched` and `career_matched` **before** dereferencing
   `household` or `career` — either may be `null` on a partial match.
7. Persist `windfall_id` on every household match. It is the stable 32-character household key
   and the join surface for any later re-enrichment; data refreshes weekly.
8. Record a per-run tally of requests issued. When responses start failing on quota exhaustion,
   the remedy is contractual — contact your Windfall account representative.

## Rules

- Never call this API from a browser. The token is organization-scoped and must not reach
  client-side code.
- `400` = malformed JSON. `401` = invalid or missing token. `403` = token/environment mismatch
  (a sandbox token on production or the reverse). `429` = rate limit.
- There is no idempotency key. Enrichment is a current-state lookup, so a retry is safe but is
  **not** deduplicated — every retry spends another usage token. Retry only on `429` and `5xx`,
  never on `400`/`401`.
- Confidence is 0.1–1.0. Set a floor appropriate to the downstream use before writing values
  back into a CRM; 1.0 is a very strong match, low scores are weak ones.
- US coverage only; data is refreshed weekly, so re-running a list more often than weekly
  spends tokens for unchanged values.

## References

- Rate limits: rate-limits/windfall-data-rate-limits.yml
- Plans and credits: plans/windfall-data-plans-pricing.yml
- Conventions: conventions/windfall-data-conventions.yml
- Errors: errors/windfall-data-problem-types.yml
- Sandbox: sandbox/windfall-data-sandbox.yml
- Field reference: data-model/windfall-data-data-model.yml
