---
name: cj-affiliate-reconcile-commissions
description: >-
  Poll CJ's GraphQL Commission Detail API incrementally to keep a local
  commission ledger current, using the commission-id watermark.
api: cj-affiliate-commission-detail-api
apis:
- cj-affiliate-commission-detail-api
operations:
- publisherCommissions
- advertiserCommissions
generated: '2026-08-13'
method: generated
source: >-
  graphql/cj-affiliate-commissions-schema.graphql (live introspection
  2026-08-13), conventions/cj-affiliate-conventions.yml,
  rate-limits/cj-affiliate-rate-limits.yml
---

# Reconcile CJ commissions

`POST https://commissions.api.cj.com/query` with
`Authorization: Bearer <personal-access-token>` and
`Content-Type: application/json`. Body is `{"query": "..."}`.

Two root queries, and which one you may call depends on which side of the
network your account is on:

- `publisherCommissions(forPublishers: [ID!], ...)` — publisher view. Includes
  `sid` and `shopperId`.
- `advertiserCommissions(forAdvertisers: [ID!], ...)` — advertiser view.
  Includes advertiser-only amounts such as `advCommissionAmountUsd`.

## Filters (identical arg set on both, apart from the forX / counterparty ids)

Date windows: `sincePostingDate` / `beforePostingDate`, `sinceEventDate` /
`beforeEventDate`, `sinceLockingDate` / `beforeLockingDate`.
Selectors: `commissionIds`, `advertiserIds` or `publisherIds`, `adIds`,
`websiteIds`.
Enums: `actionStatuses` (NEW, LOCKED, CLOSED, EXTENDED), `actionTypes`,
`lockingMethods` (IMMEDIATE, FIXED_DATE, OPEN_ENDED, FIXED_DURATION),
`validationStatuses` (PENDING, ACCEPTED, DECLINED, AUTOMATED).
Watermark: **`sinceCommissionId`**.

## The incremental pattern

Do not re-scan date ranges. CJ's documented pattern — carried over from the
legacy REST `commission-id` parameter — is a watermark:

1. **Bootstrap** with a date window: `sincePostingDate` set to your backfill
   start. Keep windows modest; the legacy REST API capped a range at 31 days and
   the same shape of query is being answered.
2. Read `maxCommissionId` off the response and store it.
3. Check `payloadComplete`. If it is `false`, the response did not contain
   everything that matched — page forward on the watermark and call again before
   you consider the window closed.
4. **Every subsequent poll** sends `sinceCommissionId: <stored max>` and nothing
   else. Store the new `maxCommissionId`.

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
      eventDate
      lockingDate
      saleAmount
      commissionAmount
      currency
      advertiserId
      websiteId
      original
      originalActionId
      correctionReason
      items { commissionItemId quantity }
    }
  }
}
```

## Corrections are new records, not edits

A correction never mutates an existing record. It arrives as a NEW commission id
with `original` distinguishing it and `originalActionId` pointing back at the
transaction being corrected. A restatement submitted through the Tracking API
produces **two** records — a zero record fully reversing the order, then a record
carrying the new state. Your ledger must key on `commissionId` and aggregate by
`orderId` / `originalActionId`, never upsert by `orderId` alone.

## Practical notes

- Not every advertiser reports in real time, so a record can appear for an event
  date well in the past. Re-poll rather than assuming a closed window stays closed.
- Introspection is open on this endpoint without credentials, so you can verify
  the current field set yourself before shipping a change.
- The REST equivalent at `commission-detail.api.cj.com/v3/commissions` is
  deprecated. Do not build on it.
