# CJ Affiliate GraphQL APIs

CJ Affiliate's current developer surface is **GraphQL**. There are **three**
endpoints, each authenticated with the same Bearer personal access token.

| API | Endpoint | Purpose | Schema in this repo |
| --- | --- | --- | --- |
| Commission Detail | `POST https://commissions.api.cj.com/query` | Near-real-time commission / transaction data. | [cj-affiliate-commissions-schema.graphql](cj-affiliate-commissions-schema.graphql) |
| Product Search / Ads | `POST https://ads.api.cj.com/query` | Product discovery, product feeds, and catalog writes. | [cj-affiliate-ads-schema.graphql](cj-affiliate-ads-schema.graphql) |
| Advertiser Tracking | `POST https://tracking.api.cj.com/graphql` | Order create / restate / cancel. Test Mode at `/graphqltest`. | [cj-affiliate-tracking-schema.graphql](cj-affiliate-tracking-schema.graphql) |

- **Documentation:** https://developers.cj.com/
- **GraphQL Reference:** https://developers.cj.com/graphql/reference/
- **GraphiQL:** https://developers.cj.com/graphql/graphiql/ · **Voyager:** https://developers.cj.com/graphql/voyager/
- **Authentication:** https://developers.cj.com/account/personal-access-tokens

## Provenance

Every schema in this directory was captured by a full `__schema` introspection
query issued against CJ's own endpoint on **2026-08-13**, and rendered to SDL
verbatim. Nothing is modelled or inferred.

**Introspection is open and unauthenticated on all three endpoints.** An
anonymous `POST {"query":"{__schema{queryType{name}}}"}` returns HTTP 200 on
each. The *data* behind them is not open — every real query needs a Bearer
personal access token. This is a deliberate discoverability posture and it is
unusual: most production GraphQL APIs disable introspection.

## Authentication

```
Authorization: Bearer <your-personal-access-token>
Content-Type: application/json
```

Tokens are minted by a human at
https://developers.cj.com/account/personal-access-tokens. There is no OAuth
flow, no token endpoint and no refresh.

Most queries additionally require you to name the account: `companyId` on the
product queries, `forPublishers` / `forAdvertisers` on the commission queries,
`enterpriseId` on Tracking API mutations.

## Commission Detail — `commissions.api.cj.com/query`

Two root queries, symmetric across the two sides of the network:

- `publisherCommissions(forPublishers, sincePostingDate, beforePostingDate, sinceEventDate, beforeEventDate, sinceLockingDate, beforeLockingDate, sinceCommissionId, commissionIds, advertiserIds, adIds, websiteIds, actionStatuses, actionTypes, lockingMethods, validationStatuses)`
- `advertiserCommissions(forAdvertisers, …, publisherIds, …)`

Responses carry `count`, `maxCommissionId` and `payloadComplete`. The documented
polling pattern is a **watermark**: keep the highest `maxCommissionId` you have
seen and pass it back as `sinceCommissionId`.

```graphql
{
  publisherCommissions(forPublishers: ["1234567"], sinceCommissionId: "987654321") {
    count
    maxCommissionId
    payloadComplete
    records {
      commissionId
      orderId
      actionStatus
      actionType
      postingDate
      saleAmount
      commissionAmount
      currency
      original
      originalActionId
      items { commissionItemId }
    }
  }
}
```

Enums as served: `ActionStatus` (NEW, LOCKED, CLOSED, EXTENDED),
`LockingMethod` (IMMEDIATE, FIXED_DATE, OPEN_ENDED, FIXED_DURATION),
`ValidationStatus` (PENDING, ACCEPTED, DECLINED, AUTOMATED), and an
eleven-value `CorrectionReason`.

No mutations. This endpoint is read-only.

## Product Search / Ads — `ads.api.cj.com/query`

**10 queries:** `products`, `shoppingProducts`, `travelExperienceProducts`,
`financeProducts`, `financeCreditCardProducts`, `shoppingProductFeeds`,
`productFeeds`, plus `productsFromApplication`,
`shoppingProductsFromApplication` and `travelExperienceProductsFromApplication`
(restricted to CJ registered applications — contact dx@cj.com).

**5 mutations:** `createShoppingProducts`, `updateShoppingProducts`,
`deleteProducts`, `createCreditCardProducts`, `updateCreditCardProducts`.

**5 subscriptions** for bulk whole-feed download: `shoppingProductCatalog`,
`travelExperienceProductCatalog`, `financeProductCatalog`,
`productFeedsFromApplication`, `checksumFromApplication`. CJ does not document
the subscription transport — see
[../asyncapi/cj-affiliate-ads-asyncapi.yml](../asyncapi/cj-affiliate-ads-asyncapi.yml).

CJ models three product verticals as separate types rather than one polymorphic
Product: `Shopping` (retail), `TravelExperience` (travel) and `CreditCard`
(finance, carrying a full term sheet of APRs, fees, grace periods and rewards).
Pagination is `offset` / `limit`; `products` additionally accepts a `page`
cursor and `disableTotalCount`.

```graphql
{
  shoppingProducts(companyId: "1234567", keywords: ["running","shoes"], limit: 50) {
    totalCount
    resultList {
      id
      adId
      advertiserId
      title
      brand
      price { amount currency }
      availability
      linkCode { html }
      googleProductCategory { id name }
    }
  }
}
```

## Advertiser Tracking — `tracking.api.cj.com/graphql`

One query (`getBuildNumber`) and three mutations: `createOrders`,
`restateOrders`, `cancelOrders`.

- Idempotent on **Order ID + Action ID + Enterprise ID**.
- `restateOrders` is **full-state replacement** — anything omitted is dropped.
- Max 100 items per order; batch at most 10,000 orders per request.
- Open-ended locking requires `status` on restate and cancel.
- **Test Mode:** `https://tracking.api.cj.com/graphqltest`, same token, same
  validation, results not posted to production reporting.

See [../skills/cj-affiliate-submit-orders.md](../skills/cj-affiliate-submit-orders.md).

## Error handling

CJ processes errors in a fixed precedence and stops at the first failing stage:

1. Authentication (`Missing Token`, `Invalid Token`)
2. Schema validation (`Invalid Syntax`, `Validation error of type WrongType: …`)
3. Authorization (`Unauthorized`)
4. Field-level validation (`Order may not have more than 100 items`, …)

Tracking API **processing** errors are not returned at all — they surface up to
an hour later as the absence of the record from Commission Detail. Full catalog
in [../errors/cj-affiliate-problem-types.yml](../errors/cj-affiliate-problem-types.yml).
