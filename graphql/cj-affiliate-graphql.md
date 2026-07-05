# CJ Affiliate GraphQL APIs

CJ Affiliate's modern developer surface is **GraphQL**. There are two documented
GraphQL endpoints, each authenticated with a **personal access token** (Bearer)
created in the CJ developer portal.

| API | Endpoint | Purpose |
| --- | --- | --- |
| Commission Detail | `POST https://commissions.api.cj.com/query` | Near-real-time commission / transaction data (`publisherCommissions`, `advertiserCommissions`). |
| Product Search / Ads | `POST https://ads.api.cj.com/query` | Product discovery across advertiser feeds (`products`, `shoppingProducts`, `shoppingProductFeeds`). |

- **Documentation:** https://developers.cj.com/
- **GraphQL Reference:** https://developers.cj.com/graphql/reference/
- **Authentication:** https://developers.cj.com/account/personal-access-tokens
- **Schema (this repo):** [cj-affiliate-schema.graphql](cj-affiliate-schema.graphql)

## Authentication

All GraphQL requests carry a Bearer personal access token (PAT). Many queries
also require your CJ company id (CID), passed as a query argument (for example
`companyId` on the product queries) or implied by the token's account.

```
Authorization: Bearer <your-personal-access-token>
Content-Type: application/json
```

## Commission Detail — publisherCommissions

Grounded example (query name, endpoint, and `records` fields are documented by CJ;
issue as a POST body `{ "query": "..." }`):

```graphql
{
  publisherCommissions(
    forPublishers: ["1234567"]
    sincePostingDate: "2026-06-01T00:00:00Z"
    beforePostingDate: "2026-07-01T00:00:00Z"
  ) {
    count
    payloadComplete
    records {
      actionTrackerName
      websiteName
      advertiserName
      orderId
      shopperId
      postingDate
      pubCommissionAmountUsd
      items {
        quantity
        perItemSaleAmountPubCurrency
        totalCommissionPubCurrency
      }
    }
  }
}
```

## Product Search — shoppingProductFeeds / products

```graphql
{
  shoppingProductFeeds(companyId: "7654321", limit: 25) {
    totalCount
    resultList {
      advertiserId
      advertiserName
      feedName
      currency
      productCount
      lastUpdated
    }
  }
}
```

> **Grounding note.** The endpoints (`commissions.api.cj.com/query`,
> `ads.api.cj.com/query`), the Bearer-PAT + CID auth model, and the query names
> (`publisherCommissions`, `advertiserCommissions`, `products`,
> `shoppingProducts`, `shoppingProductFeeds`) are grounded in CJ documentation
> and CJ knowledge-base articles. Individual field and argument names in the
> accompanying schema are **modeled** from those sources and CJ shopping-feed
> conventions — confirm exact names against the live reference at
> https://developers.cj.com/graphql/reference/ before building against them.
