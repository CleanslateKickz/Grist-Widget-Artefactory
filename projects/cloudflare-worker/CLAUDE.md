# Project: Skip Tracer Pro

## Context
Skip tracing widget for commercial real estate CRM — used to find contact information (phones, addresses, emails) for property owners, tenants, and prospects. Has two modes: URL Launch (opens search engines in new tabs) and API Search (batch search via a Cloudflare Worker).

## Architecture
- Single standalone file: `index.html` (~1267 lines)
- Sidebar layout with resizable input panel and main results area
- Two modes: URL mode (opens 14+ search engine URLs) and API mode (POSTs to Cloudflare Worker `/batch` endpoint)
- Phone-record matching logic mirrors Phone Manager widget (shared pattern)
- Webex SIP and Phone Link tel: URI buttons per phone result
- Aggregated phone section in API mode with verified/wrong/new status badges

## Specific conventions
- Grist integration uses `grist.onRecord()` with `columns: ['Name', 'Owner_Address', 'Phone']` mapping
- Phone columns config: `Phone`, `Phone_2`, `Phone_3`, `Phone_4` with notes map
- Wrong phone tracking via `Wrong_Phone` column (semicolon-delimited with optional reason in parens)
- Search mode toggle: phone-only, name+address, address-only
- Phone sanitization strips leading `1` for consistent comparison
- `addPhoneToRecord()` saves discovered phones into the first empty slot on the current record
- API mode requires a Cloudflare Worker URL (default: `osint-inline-search.sjgfroerer.workers.dev`)

## Current state
- URL mode fully functional with 14 engines across 3 tiers (by captcha resistance)
- API mode functional with batch search and aggregated results
- Phone save-to-record working
- Free services: phone validator, carrier lookup, name lookup, caller lookup, email validator
- Engines: ZabaSearch, OfficialUSA, TruePeople, FastPeople, CyberBackground, US PhoneBook, PeekYou, ThatsThem, SpyDialer
- Tier 1 (low captcha), Tier 2 (medium), Tier 3 (free POST services)

## Points of attention
- URL mode opens browser tabs — may be blocked by popup blockers
- API mode requires a deployed Cloudflare Worker with `/health` and `/batch` endpoints
- `grist.onRecord()` mapping may fail if columns aren't named exactly `Name`, `Owner_Address`, `Phone`
- `alert()` used for phone save feedback — consider replacing with toast notifications
- State abbreviation mapping covers all 50 states + DC
- Logo images hosted externally and may break
