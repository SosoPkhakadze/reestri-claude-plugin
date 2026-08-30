---
name: kyc-caucasus
description: Look up and screen companies in the public registers of Georgia (the country), Armenia and Moldova using the Reestri MCP tools. Use when a task mentions a Georgian, Armenian or Moldovan company, a 9-digit Georgian identification code, an 8-digit Armenian UID, beneficial owners in the Caucasus, Armenian public contracts, or KYC, AML, counterparty or supplier checks in these countries.
---

# KYC in Georgia, Armenia and Moldova with Reestri

The Reestri MCP server exposes seven tools. Every call is charged to the user's Apify account (cents per lookup and per record), so make the fewest calls that answer the question and never loop over large name lists without telling the user first.

## Which tool to call

| Situation | Tool | Input |
|---|---|---|
| You have a Georgian 9-digit identification code (also the VAT number) | `lookup_company_by_number` | `companyNumber` (9 digits) |
| You have a Georgian company name only | `search_companies_by_name` | `query`, `maxResults` (default 5) |
| You want every Georgian company a person is involved in | `find_companies_by_person` | `firstName`, `lastName`, `maxResults` (default 25) |
| You want subsidiaries or holdings of a Georgian organisation | `find_companies_by_organisation` | `organisationName`, `maxResults` |
| You have an Armenian 8-digit UID | `lookup_armenian_company` | `uid`, `includeBeneficialOwners` (default true), `maxDeclarations` |
| You have an Armenian company name only | `search_armenian_companies` | `query`, `language` (en, hy, ru), `maxResults` (default 3) |
| You have a name and do not know the country, or want one answer across registers | `screen_company` | `query`, `countries` (default GE and AM), `maxPerCountry`, `includeContracts` |

Rules of thumb:

- Prefer the exact lookups (by number or UID) whenever an identifier is known. They are cheaper and unambiguous.
- Georgian tools match best in Georgian script. If the user gives a Latin name, try it first; if the match is weak or ambiguous, ask the user for the Georgian spelling or the identification code instead of running many variants.
- Armenian name search is capped at 200 candidates by the register itself. Narrow the query if the results look truncated.
- `screen_company` charges one screen event plus one record per matched company. Keep `maxPerCountry` small (the default is 3) unless the user wants an exhaustive list.

## Reading the results

Every record carries `resultType`, `schemaVersion`, a `match` block and an `evidence` block.

`resultType` is one of:

- `company`: a matched company record.
- `participation`: one row of person or organisation, company and role (Georgian reverse searches).
- `contract`: one Armenian public-procurement contract.
- `not_found`: the register answered and had no match. This is a trustworthy negative.
- `unavailable`: the register could not be read (down, blocked, layout changed). This is not a negative. Say so explicitly and offer to retry later rather than reporting "no record".

`match` tells you how the record was found: `method`, `score`, the `normalized` query, `ambiguous` (true when several candidates were close) and sometimes `candidates`. When `ambiguous` is true, show the candidates to the user and ask which one they mean before drawing conclusions.

`evidence` contains `sourceUrl` (the register page that was read), `retrievedAt`, `registryAsOf` (when the register states it) and a `sha256` of the raw payload. Quote the source URL and the retrieval time in any summary you write; that is what makes the finding auditable.

People in Georgian participation rows carry a `personKey`, which is sha256("reestri-person-v1:" + country + ":" + personal number). The raw personal number is never returned. Use `personKey` to group rows that belong to the same individual; the same name with different keys means different people.

## Status semantics differ by country

The `status` field is normalised to `active`, `liquidating`, `liquidated`, `suspended`, `reorganizing` or `unmapped`, and `statusLocal` keeps the register's own wording.

- Georgia: status comes verbatim from NAPR (for example "აქტიური", "რეგისტრირებული", "ლიკვიდირებული"). Registered is treated as active. If you see `unmapped`, report the `statusLocal` wording as-is; do not guess.
- Armenia: status comes from the state register record together with registration number and date, tax ID and address.
- Moldova: the open dataset has no status field. Status is derived: `liquidated` when a liquidation date is present, otherwise `active`, and the record says `statusMethod: "derived"`. Treat a Moldovan "active" as "not recorded as liquidated", nothing stronger.

## Known limits, state them when relevant

- Georgian ownership is not returned. NAPR shows shareholders and directors only behind a captcha-protected extract view, which Reestri deliberately does not automate. `lookup_company_by_number` and `search_companies_by_name` give legal form, status and names; for owners, use `find_companies_by_person` in reverse (person to companies) or advise the user to order an official NAPR extract.
- Armenian beneficial owners come from the company's own BODS declarations filed with the register. They are self-declared, can be missing for companies that have not filed, and can be out of date. Say "declared beneficial owners as of the declaration date", not "verified owners". Home addresses are stripped.
- Moldova is opt-in. `screen_company` only includes Moldova when `countries` contains `MD`. Moldovan data comes from ASP open data (founders, shares, directors) refreshed weekly, so it can lag the live register.
- Armenian public-contract exposure (in `screen_company` with `includeContracts`) is aggregated from the procurement portal and shows how much public money a supplier has won, not whether anything was improper.
- Kazakhstan, Azerbaijan and Uzbekistan are not covered.

## Example: screening a counterparty

User asks: "We are about to sign with Alpha Trade LLC, said to be Georgian. Check them."

1. Call `screen_company` with `query: "Alpha Trade"`, default countries (GE and AM), `maxPerCountry: 3`.
2. Read the summary: matches per country, statuses, owners found, contract exposure, verdict. Check every country block for `unavailable` and say so if one is present.
3. If Georgia returns a `company` with `match.ambiguous: true`, list the candidates with their identification codes and ask the user to confirm. Once confirmed, call `lookup_company_by_number` with that code for the definitive record.
4. For the confirmed company, call `find_companies_by_organisation` with the registered Georgian name if the user wants to know what else it owns.
5. If a named director or owner is known, call `find_companies_by_person` to list their other Georgian companies; group by `personKey`.
6. Write the summary: legal name in local script and Latin, identification code, legal form, status with the register's wording, what could not be established (Georgian ownership without an extract), and one evidence line per record: source URL and retrieval time.

Keep the tone factual. The tools report what a public register showed at a moment in time; they do not judge risk. Leave the verdict to the user.
