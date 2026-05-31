# subpoof skill for Claude Code

A [Claude Code](https://claude.com/claude-code) Agent Skill that teaches Claude
to drive the [subpoof](https://subpoof.com) subdomain-intelligence API: submit
and poll subdomain-enumeration scans, read scan results, list newly-registered
domains, query the subdomain-label dictionary, and pull per-domain scan history.

Once installed, Claude invokes it automatically when you ask things like
"enumerate the subdomains of example.com", "what changed for example.com since
last scan", or "show me newly-registered .io domains".

## Install

In Claude Code:

```
/plugin marketplace add jabreity/subpoof-skill
/plugin install subpoof@subpoof-skill
```

This repo is its own plugin marketplace, so the first command points Claude Code
straight at it; the second installs the `subpoof` plugin (which contains the
skill). Run `/plugin` any time to manage installed plugins.

## Prerequisites

You need a subpoof account and an API key. The key lives on your dashboard
Settings page at [subpoof.com](https://subpoof.com). Expose it to Claude Code as
an environment variable before you start a session:

```bash
export SUBPOOF_API_KEY="your_key_here"
```

The skill reads `SUBPOOF_API_KEY` from the environment and never logs or echoes
it. If the variable is not set, Claude will ask you for the key rather than
making unauthenticated calls.

## What it covers

- **Scan lifecycle** - submit a scan (`POST /v1/scans`), poll the job to
  completion, and read the analyzed results, including the enrichment options
  and the dictionary-probe controls.
- **Registrations** - the daily feed of newly-registered domains, filterable by
  substring, TLD, and date range.
- **Dictionary** - the ranked subdomain-label dictionary built from prior scans.
- **History** - the per-domain timeline of what subdomains appeared or
  disappeared over time.
- **Conventions** - the `data`/`pagination` envelope, the `error` envelope with
  `request_id`, and the monthly-quota rate-limit headers, so Claude paginates,
  handles errors, and backs off on `429` correctly.

The skill points Claude at the live OpenAPI spec
(`https://api.subpoof.com/v1/openapi.json`) as the authoritative reference for
exact parameters and schemas.

## Updating

```
/plugin marketplace update subpoof-skill
```

then reinstall or let Claude Code pick up the new version.

## License

[MIT](LICENSE)
