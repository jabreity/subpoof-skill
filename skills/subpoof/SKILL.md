---
name: subpoof
description: Use the subpoof subdomain-intelligence API at https://api.subpoof.com/v1 - submit and poll subdomain-enumeration scans, read scan results, list newly-registered domains, query the subdomain-label dictionary, and pull per-domain scan history. Use when the user wants to enumerate a domain's subdomains, check what changed for a domain, find newly-registered domains for a TLD, or otherwise work with subpoof programmatically.
---

# subpoof API

subpoof is a subdomain-enumeration and domain-monitoring service. This skill
drives its public JSON API. The full machine-readable contract is the OpenAPI
spec at `https://api.subpoof.com/v1/openapi.json` - fetch it when you need exact
parameter names or schemas; this file covers the workflows and the conventions
the spec does not spell out.

## Setup: API key

Every request authenticates with the `X-API-Key` header. The key is on the
account's dashboard Settings page; treat it as a secret and never log it.

Read it from the `SUBPOOF_API_KEY` environment variable. If it is not set, stop
and ask the user for their key rather than guessing or calling unauthenticated.

```bash
curl -sS https://api.subpoof.com/v1/registrations \
  -H "X-API-Key: $SUBPOOF_API_KEY"
```

Base URL: `https://api.subpoof.com/v1`. HTTPS only.

## Conventions

These hold across every endpoint - rely on them instead of re-deriving per call.

### Success envelope (list endpoints)

```json
{
  "data": [ ... ],
  "pagination": {
    "page": 1, "page_size": 25, "total_pages": 7,
    "has_next": true, "has_prev": false
  }
}
```

Pagination params: `page` (1-indexed, default 1) and `page_size`
(default 25, max 100). To walk every page, loop while `pagination.has_next`
is true, incrementing `page`. Do not hardcode a page count.

### Error envelope

Any non-2xx returns:

```json
{ "error": { "code": "bad_request", "message": "...", "request_id": "req_..." } }
```

The `request_id` also rides on the `X-Request-ID` response header - capture it
when surfacing an error to the user.

| HTTP | code            | Meaning / what to do                                            |
|------|-----------------|-----------------------------------------------------------------|
| 400  | `bad_request`   | Bad input. Fix the field named in `message`; do not retry as-is.|
| 401  | `unauthenticated` | Missing/invalid `X-API-Key`. Ask the user to check their key. |
| 404  | `not_found`     | No such resource (or not visible to this key).                  |
| 409  | `conflict`      | Resource not in a usable state yet (e.g. scan not complete).    |
| 429  | `rate_limited`  | Monthly query quota reached. See rate limits below. Do NOT loop-retry. |

### Rate limits / quota

Successful calls return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and
`X-RateLimit-Reset` (an ISO date - the quota resets on the 1st of next month).
A `429 rate_limited` means the monthly quota is spent; back off and tell the
user, do not spin-retry. Polling a scan's status (GET) is cheap, but be
considerate: poll on an interval (see below), not in a tight loop.

## Core workflow: scan a domain end to end

This is the most common task. Submit, poll until complete, then read results.

### 1. Submit - `POST /v1/scans`

```bash
curl -sS -X POST https://api.subpoof.com/v1/scans \
  -H "X-API-Key: $SUBPOOF_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "example.com"}'
```

Returns `202` with a `job_id`:

```json
{ "job_id": "...", "domain": "example.com", "status": "pending",
  "created_at": "...", "idempotent": false }
```

Idempotency: re-submitting the same domain on the same calendar day returns the
existing job (`200`, `"idempotent": true`) and does not consume another credit.
So it is safe to re-run a submit without double-charging.

Optional `enrichments` object tunes what the scan collects. Omit it entirely to
get the sensible defaults (all on). Every flag defaults to `true`:

```json
{
  "domain": "example.com",
  "enrichments": {
    "dns_enrich": true, "ms_tenant": true, "ntlm_probe": true,
    "rbl_check": true, "tls_port_check": true, "tls_cert_enum": true,
    "lookalike_check": true, "http_fingerprint": true, "email_security": true,
    "dict_probe": { "enabled": true, "count": 100, "offset": 0 }
  }
}
```

`dict_probe` may be a bool (shorthand for enabled/disabled at defaults) or an
object: `count` is how many dictionary labels to probe (1-1000, default 100),
`offset` lets you page deeper into the label list on a later scan. To run a fast
scan, set the heavy flags to `false` and `dict_probe.enabled` to `false`.

### 2. Poll - `GET /v1/scans/{job_id}`

```bash
curl -sS https://api.subpoof.com/v1/scans/$JOB_ID -H "X-API-Key: $SUBPOOF_API_KEY"
```

`status` moves through: `pending` -> `claimed` -> `running` -> `complete`
(terminal), or `error` / `stopped` (terminal). Poll roughly every 5-10s; a scan
typically takes from tens of seconds to a few minutes depending on enrichments.

When `status` is `complete` the response includes `subdomain_count`,
a `signal_summary` (counts of notable indicators found), and `results_url`.
Stop polling on any terminal status. On `error`, surface the `error` field.

### 3. Read results - `GET /v1/scans/{job_id}/results`

```bash
curl -sS "https://api.subpoof.com/v1/scans/$JOB_ID/results?page=1&page_size=100" \
  -H "X-API-Key: $SUBPOOF_API_KEY"
```

Returns `409 conflict` if the job is not `complete` yet - poll first. Paginated
like any list endpoint. Pass `category` to fetch only one slice of the findings
(e.g. `subdomains`, `tls_certs`, `lookalikes`, `ntlm`) instead of the full set.

## Other endpoints

### `GET /v1/scans` - list past scans

Filter with `status` and/or exact `domain`. Paginated. Use this to find a prior
`job_id` or summarize recent activity.

### `GET /v1/registrations` - newly-registered domains

A daily feed of freshly-registered domains. Filter with `q` (case-insensitive
domain substring), `tld` (with or without the leading dot, e.g. `.io` or `io`),
and a `date_from`/`date_to` range (inclusive, YYYY-MM-DD). Paginated.

### `GET /v1/dictionary` - subdomain-label dictionary

The account's ranked subdomain-label dictionary (labels seen across prior
scans). Filter with `q` (label substring), `domain_filter` (labels seen on one
root domain), `category`, or `monitored` (`yes`/`no`). Paginated.

### `GET /v1/domains/{domain}/history` - per-domain timeline

The scan-history timeline for a domain, each entry diffed against the prior
scan - what subdomains appeared or disappeared over time.

## How to behave

- Read `SUBPOOF_API_KEY` from the environment; never invent a key, never echo it.
- For "scan domain X" requests, run the full submit -> poll -> results loop and
  summarize the findings (subdomain count, notable signals, new vs. removed),
  not the raw JSON dump, unless the user asks for raw output.
- On `429`, stop and report the quota is spent and when it resets; do not retry.
- When you need an exact parameter or field you are unsure of, fetch
  `https://api.subpoof.com/v1/openapi.json` rather than guessing.
- Paginate to completion when the user asks for "all" of something; otherwise
  fetch the first page and offer to fetch more.
