---
name: cj-affiliate-submit-orders
description: >-
  Submit, restate and cancel advertiser transactions through CJ's GraphQL
  Tracking API, safely — including Test Mode and the idempotency contract.
api: cj-affiliate-tracking-api
apis:
- cj-affiliate-tracking-api
operations:
- createOrders
- restateOrders
- cancelOrders
generated: '2026-08-13'
method: generated
source: >-
  graphql/cj-affiliate-tracking-schema.graphql (live introspection 2026-08-13),
  https://production-docs-assets.p.cjpowered.com/Advertiser%20API%20Tracking/API%20Overview.md,
  conventions/cj-affiliate-conventions.yml, sandbox/cj-affiliate-sandbox.yml
---

# Submit CJ orders (advertiser side)

**This flow moves money.** Every mutation here creates or changes a
commissionable transaction. Require human confirmation before running it
unattended.

- Live: `POST https://tracking.api.cj.com/graphql`
- **Test Mode: `POST https://tracking.api.cj.com/graphqltest`**

Same token, same schema, same validation. Mode is chosen by URL — there is no
test key prefix — so a wrong base URL writes to production.

## Idempotency

CJ defines a unique order as **`orderId` + `actionTrackerId` + `enterpriseId`**.
`createOrders` applies your program's duplicate-order logic against that key, so
a retry of an identical create is safe. There is no `Idempotency-Key` header and
none is needed.

## createOrders

```graphql
mutation {
  createOrders(newOrders: [{
    enterpriseId: "1234567"
    actionTrackerId: 987654
    orderId: "ORDER-0001"
    eventTime: "2026-08-13T10:00:00Z"
    amount: "129.99"
    currency: "USD"
    cjEvent: "f5c7245cacf421yf81435f7f0b82c836"
    items: [{ ... }]
  }]) { ... }
}
```

- Max **100 items** per order — exceeding it returns "Order may not have more
  than 100 items".
- `eventTime` / `updateTime` may not be in the future.
- Batch at most **10,000 orders** per request; larger batches can return a 504.
- `customParameters` takes arbitrary name/value pairs; `verticalParameters`
  takes CJ's typed per-vertical block (Retail, Finance, Travel,
  NetworkServices).

## restateOrders — full-state replacement

`restateOrders` **completely overwrites** the order with exactly what you send.
Anything you omit is dropped, not preserved. Always build a restatement from the
order's current full state, never from a diff.

Constraints CJ enforces:

- The order must exist, be commissionable, and be neither locked nor closed.
- A restatement cannot make a commissionable order non-commissionable.
- A restatement cannot change which publisher earns the commission.
- The `actionTrackerId` must belong to the same account as `enterpriseId`.
- **Open-ended locking:** you MUST pass `status` (`Pending` or `Accepted`). Omit
  it and the request fails during processing — not at receipt, so you will get a
  clean response and a silent failure an hour later.

## cancelOrders

Supports `status: Declined` only, and CJ notes that cancellations sent with a
status will not process for **non**-open-ended actions. `correctionReason` takes
CJ's PascalCase vocabulary (`ReturnedMerchandise`, `DuplicateOrder`,
`BookingCanceled`, `Compliance`, …) — which is spelled differently from the
SCREAMING_SNAKE vocabulary the Commission Detail API returns, and is not
one-to-one with it.

## Two error surfaces — do not conflate them

**Real-time**, returned on the call, in strict precedence: authentication →
schema validation → authorization → field-level validation. You only ever see
the first failing stage, so fix and re-send rather than assuming you have the
full error list.

**Processing**, up to an hour later, **not returned to you at all**. Duplicate
detection, lock/close state, action-tracker ownership and publisher-change rules
are all evaluated here. The only signal is whether the record appears in
Commission Detail.

## Closing the loop

Poll `publisherCommissions` / `advertiserCommissions` on
`https://commissions.api.cj.com/query` and confirm the record landed. A
successful restatement appears as two records — a zero reversal plus the new
state. See `skills/cj-affiliate-reconcile-commissions.md`.
