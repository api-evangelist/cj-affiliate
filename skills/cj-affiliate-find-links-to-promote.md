---
name: cj-affiliate-find-links-to-promote
description: >-
  Find advertiser links a CJ publisher can place, then confirm the advertiser
  relationship and commission terms behind them.
api: cj-affiliate-link-search-api
apis:
- cj-affiliate-link-search-api
- cj-affiliate-advertiser-lookup-api
operations:
- linkSearch
- advertiserLookup
generated: '2026-08-13'
method: generated
source: >-
  openapi/cj-affiliate-link-search-api-openapi.yml,
  openapi/cj-affiliate-advertiser-lookup-api-openapi.yml,
  conventions/cj-affiliate-conventions.yml
---

# Find CJ links to promote

Publisher-side flow. Two REST calls: search links, then look up the advertisers
behind the ones worth placing.

## Before you start

- You need a CJ **Personal Access Token** (`Authorization: Bearer <token>`) and
  your **PID** (website / promotional property id) and **CID** (company id).
- Both APIs are **publishers only** and both are capped at **25 calls per
  minute**. There is no rate-limit response header — count your own calls.
- Both return **XML**, not JSON.

## Step 1 — search links (`linkSearch`)

`GET https://link-search.api.cj.com/v2/link-search`

Send `website-id` (your PID) plus at least one filter. An empty request returns
**zero** results, not everything.

```
curl -s "https://link-search.api.cj.com/v2/link-search?website-id=$PID&advertiser-ids=joined&link-type=banner" \
  -H "Authorization: Bearer $CJ_TOKEN"
```

Useful filters: `keywords`, `category` (sub-category only — parent categories are
not searchable), `link-type`, `promotion-type`, `targeted-country` (one
two-letter code), `language`, `last-updated` (MM/DD/YYYY), `allow-deep-linking`,
`mobile-optimized`, `cross-device-only`, `link-Id`.

Keyword logic: default is OR. `+term` requires it, `-term` excludes it. So
`keywords=%2Bkitchen+-sink` means kitchen AND NOT sink.

**Encoding trap CJ calls out explicitly:** a space must encode to `+` and a
literal `+` must encode to `%2B`. Most standard URI encoders get this wrong.

Read `total-matched`, `records-returned` and `page-number` off the `<links>`
element and page with `page-number` / `records-per-page` (100 per page by
default).

Per link you get `link-id`, `advertiser-id`, `advertiser-name`,
`link-code-html`, `link-code-javascript`, `clickUrl`, `destination`,
`relationship-status`, `promotion-type` with start/end dates, `coupon-code`,
`seven-day-epc` and `three-month-epc`.

## Step 2 — qualify the advertisers (`advertiserLookup`)

`GET https://advertiser-lookup.api.cj.com/v2/advertiser-lookup`

`requestor-cid` is **required**. Pass the advertiser CIDs you collected in step 1.

```
curl -s "https://advertiser-lookup.api.cj.com/v2/advertiser-lookup?requestor-cid=$CID&advertiser-ids=1234567,7654321" \
  -H "Authorization: Bearer $CJ_TOKEN"
```

Check `account-status` (`Active` vs `Deactive` — only shown for advertisers you
are joined to), `relationship-status`, `network-rank`, the EPC figures, and the
`actions` array for default commission per action.

**Do not treat those rates as final.** CJ states this API excludes commission
rates for Situations and Promotional Properties. For accurate rates use the
GraphQL Program Terms API.

`records-per-page` defaults to 25 here and maxes at 100 — different from Link
Search.

## Errors

- `401` with no message usually means a **wrong URL**, not bad credentials.
- `401 "You must provide an Authorization header."` — missing header.
- `401 "Not Authenticated: xxxxxx"` — invalid key. The response echoes your key,
  so scrub it before logging.

See `errors/cj-affiliate-problem-types.yml`.
