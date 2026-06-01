# Skip Tracer Pro

> Skip tracing widget for Grist — 14+ search engines, phone validation, and Cloudflare Worker API for batch lookups. Designed for commercial real estate CRM workflows.

## Features
- **URL Launch mode** — Opens people/phone/address search engine URLs in new tabs
- **API Search mode** — Batch search via Cloudflare Worker with aggregated results
- **Phone status tracking** — Detects if a phone is saved, marked wrong, or new
- **One-click save** — Save discovered phones directly to the current Grist record
- **Webex & Phone Link** — Click-to-call buttons (SIP and tel: URIs)
- **14+ search engines** — ZabaSearch, TruePeople, FastPeople, CyberBackground, US PhoneBook, PeekYou, ThatsThem, SpyDialer, and free validators
- **Tier system** — Engines ranked by captcha resistance (Tier 1 = lowest risk)

## Installation
1. In Grist: **Add Widget → Custom → Enter Custom URL**
2. URL: `https://CleanslateKickz.github.io/Grist-Widget-Artefactory/cloudflare-worker/`
3. Grant **"Full document access"**
4. Map columns: `Name`, `Owner_Address`, `Phone`

## Usage

### URL Launch Mode (default)
1. Enter a name, address, phone, or email
2. Select search mode: **Phone**, **Name + Address**, or **Address Only**
3. Click a search engine card to open it in a new tab
4. Click **Launch All Searches** to open all applicable engines

### API Search Mode
1. Click **API Search** toggle
2. Enter your Cloudflare Worker URL
3. Click **Search All Engines** for batch results
4. Aggregated phones appear with status badges (saved/wrong/new)
5. Click a phone to save it to the current record

## CRE CRM Use Cases
- **Owner lookup** — Find hard-to-reach property owners by name + state
- **Phone validation** — Verify tenant contact numbers before sending notices
- **Skip tracing** — Find forwarding numbers for old tenants with outstanding balances
- **Batch search** — Use the Cloudflare Worker to search all engines at once
