# Reestri plugin for Claude Code

Public company registers of Georgia (the country), Armenia and Moldova as MCP tools inside Claude Code, plus a skill that tells Claude when to use which tool and how to read the results.

Disclosure: the plugin author (Soso Pkhakadze) builds and sells the underlying tools on the Apify Store under the name Reestri. This repository is open source (MIT); the lookups themselves are paid per call through your own Apify account.

## What you get

One remote MCP server, `reestri`, proxied by mcp.apify.com, with seven tools:

| Tool | Register | What it does |
|---|---|---|
| `lookup_company_by_number` | Georgia (NAPR) | Exact lookup by 9-digit identification code |
| `search_companies_by_name` | Georgia (NAPR) | Name search, Georgian or Latin script |
| `find_companies_by_person` | Georgia (NAPR) | Person to every company they take part in, with roles |
| `find_companies_by_organisation` | Georgia (NAPR) | Organisation to its subsidiaries and holdings |
| `lookup_armenian_company` | Armenia (e-register.moj.am) | Exact lookup by 8-digit UID, with beneficial owners from BODS declarations |
| `search_armenian_companies` | Armenia (e-register.moj.am) | Name search in English, Armenian or Russian |
| `screen_company` | Georgia, Armenia, Moldova (opt-in) | One call across registers: statuses, owners, Armenian public-contract exposure |

Every record carries a `match` block (how it was found, ambiguity) and an `evidence` block (source URL, retrieval time, sha256 of the raw page). Registers that could not be read are reported as `unavailable`, never as a negative.

One skill, `kyc-caucasus`, loads automatically when a task touches these countries and explains tool choice, status semantics, known limits and a counterparty screening workflow. Read it at `skills/kyc-caucasus/SKILL.md`.

## Install

Requires Claude Code 2.1 or later.

From this repository (it doubles as a single-plugin marketplace):

```
claude plugin marketplace add SosoPkhakadze/reestri-claude-plugin
claude plugin install reestri@reestri
```

Or inside a session:

```
/plugin marketplace add SosoPkhakadze/reestri-claude-plugin
/plugin install reestri@reestri
```

Once the plugin is listed in the community marketplace you can also use:

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install reestri@claude-community
```

To try it without installing, clone the repo and run `claude --plugin-dir ./reestri-claude-plugin`.

## Token setup

The server authenticates with your Apify API token, read from the `APIFY_TOKEN` environment variable.

1. Create a free Apify account at https://apify.com and open https://console.apify.com/settings/integrations.
2. Copy your personal API token (or create one scoped to this use).
3. Export it before starting Claude Code:

```
export APIFY_TOKEN=apify_api_...        # macOS, Linux
$env:APIFY_TOKEN = "apify_api_..."      # Windows PowerShell
```

If the variable is missing, Claude Code loads the plugin but the server reports a missing-variable warning in `claude mcp list` and calls fail with 401. Set the variable and run `/reload-plugins`.

## Pricing

The tools are pay per event, billed by Apify to your account. Indicative prices at the time of writing:

| Tool | Query | Record |
|---|---|---|
| Georgia company lookup | $0.05 | $0.25 per company |
| Georgia person search | $0.05 | $0.10 per participation row |
| Armenia company lookup | $0.05 | $0.50 per company incl. owners |
| Company screen | $0.50 | $0.10 per matched company |

A query is charged even when the result is `not_found`, because a verified negative is the answer. Current prices are on each tool's Apify Store page. Apify's free plan includes monthly credit that covers a few dozen lookups.

## Data notes

- Georgia: ownership is behind a captcha-protected extract view on NAPR and is not automated. You get legal form, status, names, and reverse person and organisation searches.
- Armenia: beneficial owners are the company's own BODS declarations, self-declared and sometimes missing or stale. Home addresses are removed.
- Moldova: ASP open data refreshed weekly; status is derived from the liquidation date. Included only when you pass `countries: ["MD"]` to `screen_company`.
- Not covered: Kazakhstan, Azerbaijan, Uzbekistan.

Personal numbers of individuals are never returned; a stable `personKey` hash (sha256 of `reestri-person-v1:<country>:<number>`) lets you group rows by person.

## Links

- Tools on Apify: https://apify.com/reestri
- MCP endpoint used by this plugin: https://mcp.apify.com/?tools=reestri/ge-company-lookup,reestri/ge-person-search,reestri/am-company-lookup,reestri/company-screen
- Claude Code plugin docs: https://code.claude.com/docs/en/plugins
- Issues and requests: https://github.com/SosoPkhakadze/reestri-claude-plugin/issues

## License

MIT for this repository. Register data is subject to the terms of the respective public registers.
