# CJ Affiliate (cj-affiliate)

CJ Affiliate (formerly Commission Junction) is one of the largest affiliate
marketing networks, connecting publishers with thousands of advertiser programs.
Its developer platform is **primarily a modern GraphQL API** - the Commission
Detail API at `commissions.api.cj.com` and the Product Search / ads GraphQL API
at `ads.api.cj.com` - covering commission and transaction data, product feeds,
and advertiser discovery. A set of **legacy REST APIs** (Link Search, Advertiser
Lookup, Publisher Lookup, and the legacy Product Search) remains documented for
finding links, advertisers, publishers, and products.

**APIs.json:** [https://raw.githubusercontent.com/api-evangelist/cj-affiliate/refs/heads/main/apis.yml](https://raw.githubusercontent.com/api-evangelist/cj-affiliate/refs/heads/main/apis.yml)

## Access Model

- **Network membership is free to join** as a publisher; the APIs are a benefit
  of holding a CJ account, not a separately-priced product.
- **Authentication is a personal access token (PAT)** created in the CJ developer
  portal and sent as `Authorization: Bearer <token>`. Many calls also need your
  CJ **company id (CID)** (as a `requestor-cid` REST parameter or a `companyId`
  GraphQL argument).
- What data you can read (an advertiser's product feed, your commission records)
  is governed by your **account type and program relationships** - the `products`
  / `shoppingProducts` queries only return advertisers you have joined, while
  `shoppingProductFeeds` lists every feed in the network for cold discovery.
- Constraints on usage are **rate limits and pagination**, not a per-call price.

## GraphQL is primary

CJ's newer surface is GraphQL. Both GraphQL APIs share the same request pattern:
`POST` a `{ "query": "..." }` body with a Bearer PAT.

- **Commission Detail** — `POST https://commissions.api.cj.com/query`
  (`publisherCommissions`, `advertiserCommissions`) — near-real-time commission
  and transaction data.
- **Product Search / Ads** — `POST https://ads.api.cj.com/query` (`products`,
  `shoppingProducts`, `shoppingProductFeeds`) — product discovery across
  advertiser feeds.

See [graphql/cj-affiliate-graphql.md](graphql/cj-affiliate-graphql.md) and the
partial schema at [graphql/cj-affiliate-schema.graphql](graphql/cj-affiliate-schema.graphql).

## Tags

- Affiliate Marketing
- Affiliate Network
- Commission
- Product Search
- Publisher
- Advertiser
- GraphQL
- Ecommerce

## Timestamps

- **Created:** 2026-07-05
- **Modified:** 2026-07-05

## APIs

### CJ Commission Detail API (GraphQL)

Modern GraphQL API serving near-real-time commission and transaction data. The
`publisherCommissions` and `advertiserCommissions` queries return commission
records filtered by posting-date range, action status, advertiser or publisher
IDs, and order / shopper IDs, with monetary amounts and line-item detail.

- **Human URL:** [https://developers.cj.com/graphql/reference/Commission Detail](https://developers.cj.com/graphql/reference/Commission%20Detail)
- **Base URL:** `https://commissions.api.cj.com/query`
- **Properties:** [GraphQL Schema](graphql/cj-affiliate-schema.graphql), [GraphQL Guide](graphql/cj-affiliate-graphql.md), [Postman Collection](collections/cj-affiliate.postman_collection.json), [Open Collection](collections/cj-affiliate.opencollection.json)

### CJ Product Search API (GraphQL)

Modern GraphQL API for product discovery across advertiser product feeds. Search
products by keyword, price range, currency, country / serviceable area, UPC, and
advertiser; `shoppingProductFeeds` lists every advertiser feed in the network
for cold discovery regardless of join status.

- **Human URL:** [https://developers.cj.com/graphql/reference/Product Search](https://developers.cj.com/graphql/reference/Product%20Search)
- **Base URL:** `https://ads.api.cj.com/query`
- **Properties:** [GraphQL Schema](graphql/cj-affiliate-schema.graphql), [GraphQL Guide](graphql/cj-affiliate-graphql.md)

### CJ Link Search API (REST, legacy)

Legacy REST API to search advertiser links by keyword, country, category,
targeted / serviceable area, advertiser relationship status, link type, and
promotion type (Coupon, Free Shipping, Sale).

- **Human URL:** [https://developers.cj.com/docs/rest-apis/link-search](https://developers.cj.com/docs/rest-apis/link-search)
- **Base URL:** `https://link-search.api.cj.com/v2/link-search`
- **Properties:** [OpenAPI](openapi/cj-affiliate-openapi.yml), [Postman Collection](collections/cj-affiliate.postman_collection.json), [Open Collection](collections/cj-affiliate.opencollection.json)

### CJ Advertiser Lookup API (REST)

Legacy REST API letting publishers find advertisers by CID, program name,
program URL, keywords, category, or relationship status, and read program
details such as commission rates, category, and performance-incentive options.

- **Human URL:** [https://developers.cj.com/docs/rest-apis/advertiser-lookup](https://developers.cj.com/docs/rest-apis/advertiser-lookup)
- **Base URL:** `https://advertiser-lookup.api.cj.com/v2/advertiser-lookup`
- **Properties:** [OpenAPI](openapi/cj-affiliate-openapi.yml), [Postman Collection](collections/cj-affiliate.postman_collection.json), [Open Collection](collections/cj-affiliate.opencollection.json)

### CJ Publisher Lookup API (REST)

Legacy REST API letting advertisers search publishers within their program by
criteria such as country or relationship status, and view details about those
publishers.

- **Human URL:** [https://developers.cj.com/docs/rest-apis/publisher-lookup](https://developers.cj.com/docs/rest-apis/publisher-lookup)
- **Base URL:** `https://publisher-lookup.api.cj.com/v2/joined-publisher-lookup`
- **Properties:** [OpenAPI](openapi/cj-affiliate-openapi.yml), [Postman Collection](collections/cj-affiliate.postman_collection.json), [Open Collection](collections/cj-affiliate.opencollection.json)

### CJ Product Search (Legacy) API (REST)

Legacy REST product catalog search across advertiser product feeds. CJ now
steers new integrations to the GraphQL Product Search API; this endpoint remains
documented for existing integrations.

- **Human URL:** [https://developers.cj.com/docs/rest-apis/product-search-(legacy)](https://developers.cj.com/docs/rest-apis/product-search-\(legacy\))
- **Base URL:** `https://product-search.api.cj.com/v2/product-search`
- **Properties:** [OpenAPI](openapi/cj-affiliate-openapi.yml)

## Grounding & Honesty Notes

- **Confirmed:** the two GraphQL endpoints (`commissions.api.cj.com/query`,
  `ads.api.cj.com/query`) and their query names; the Bearer personal access
  token + company id (CID) auth model; and the Advertiser Lookup and Publisher
  Lookup REST endpoints (documented by CJ with `curl` examples).
- **Modeled:** the Link Search and legacy Product Search REST paths follow the
  documented `<service>.api.cj.com/v2/<service>` family pattern, and several
  individual query parameters and response / schema field names in
  `openapi/cj-affiliate-openapi.yml` and `graphql/cj-affiliate-schema.graphql`
  are inferred from CJ knowledge-base articles and shopping-feed conventions.
  The CJ developer portal is a JavaScript-rendered SPA, so exact current field
  names should be confirmed against the live reference at
  [https://developers.cj.com/](https://developers.cj.com/).

## Common Properties

- [Website](https://www.cj.com/)
- [LinkedIn](https://www.linkedin.com/company/cj-affiliate-by-conversant)
- [Documentation](https://developers.cj.com/)
- [Authentication](https://developers.cj.com/account/personal-access-tokens)
- [Plans](plans/cj-affiliate-plans-pricing.yml)
- [Rate Limits](rate-limits/cj-affiliate-rate-limits.yml)
- [Fin Ops](finops/cj-affiliate-finops.yml)
- [Blog](https://junction.cj.com/)

## Maintainers

**FN:** Kin Lane
**Email:** kin@apievangelist.com
