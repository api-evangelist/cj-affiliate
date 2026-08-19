---
name: cj-affiliate-discover-products
description: >-
  Discover promotable products across CJ advertiser feeds with the GraphQL ads
  API, including cold discovery of feeds you have not joined.
api: cj-affiliate-product-search-api
apis:
- cj-affiliate-product-search-api
operations:
- shoppingProductFeeds
- productFeeds
- products
- shoppingProducts
- travelExperienceProducts
- financeProducts
- financeCreditCardProducts
generated: '2026-08-13'
method: generated
source: >-
  graphql/cj-affiliate-ads-schema.graphql (live introspection 2026-08-13),
  data-model/cj-affiliate-data-model.yml
---

# Discover CJ products to promote

`POST https://ads.api.cj.com/query` with
`Authorization: Bearer <personal-access-token>`.

`companyId` (your CID) is **required** on every query below.

## Step 1 — find the feeds

`shoppingProductFeeds(companyId:, partnerIds:, advertiserCountry:, offset:, limit:)`
lists advertiser product feeds **regardless of join status**, which makes it the
cold-discovery entry point: you can see what is out there before joining a
program. `productFeeds` does the same with a `feedType` filter.

A feed is identified by `adId`. That is the same id space as a link's `link-id`
on the REST Link Search API — in CJ, a product feed *is* an ad.

## Step 2 — search inside them

Pick the query that matches the vertical; CJ models these as separate types, not
one polymorphic Product.

- `products` — the common field set across all verticals. The richest filter
  set: `keywords`, `lowPrice` / `highPrice`, `currency`, `brand`,
  `googleProductCategoryIds`, `availability`, `serviceableAreas` /
  `excludeServiceableAreas`, `advertiserCountries`, `targetCountry`,
  `discountPercentage`, `partnerStatus`, `sortBy` / `sortOrder`, and
  `excludePartnerIds` / `excludeProductIds`.
- `shoppingProducts` — retail, adds `gtin` and `googleProductCategoryNames`.
- `travelExperienceProducts` — travel.
- `financeProducts` / `financeCreditCardProducts` — finance, with a full credit
  card term sheet: APRs (intro, regular, penalty, cash advance, transfer), fees,
  grace periods and up to four reward blocks.

```graphql
{
  shoppingProducts(companyId: "1234567", keywords: ["running","shoes"],
                   lowPrice: 25.00, highPrice: 150.00, currency: "USD",
                   availability: IN_STOCK, limit: 50, offset: 0) {
    totalCount
    count
    resultList {
      id
      adId
      advertiserId
      title
      brand
      price { amount currency }
      salePrice { amount currency }
      availability
      link
      imageLink
      googleProductCategory { id name }
      linkCode { html }
    }
  }
}
```

`linkCode.html` is the placeable creative — the product-query equivalent of
`link-code-html` on the REST Link Search API.

## Pagination

`offset` / `limit` on every query. `products` additionally accepts a `page`
cursor and `disableTotalCount: true`, which is worth setting when you are paging
deep and do not need the total — CJ exposes it precisely because computing the
count is expensive.

## Bulk instead of paging

If you want a whole feed rather than a page of it, CJ exposes GraphQL
**subscriptions** — `shoppingProductCatalog`, `travelExperienceProductCatalog`,
`financeProductCatalog` — that stream an advertiser's entire feed. CJ warns these
"may take some time to complete due to the large number of products". CJ does
not document the subscription transport; see
`asyncapi/cj-affiliate-ads-asyncapi.yml`.

## Restricted queries

`productsFromApplication`, `shoppingProductsFromApplication`,
`travelExperienceProductsFromApplication`, `checksumFromApplication` and
`productFeedsFromApplication` are for **CJ registered applications only**. CJ's
own schema directs you to the CJ Developer Experience Group (dx at cj.com).
Calling them without that arrangement will not work.
