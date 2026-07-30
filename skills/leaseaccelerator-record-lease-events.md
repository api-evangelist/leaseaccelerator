---
name: leaseaccelerator-record-lease-events
description: >-
  Record end-of-term and asset events against LeaseAccelerator leases — renewals, terminations, buyouts,
  returns, evergreen and impairments — and understand what can and cannot be rolled back.
api: leaseaccelerator:leaseaccelerator-api
generated: '2026-07-19'
method: generated
source: >-
  openapi/leaseaccelerator-api-openapi.yml,
  https://docs-leaseaccelerator.insightsoftware.com/hc/en-us/articles/33895853166093-API-Methods
operations:
  - FindDeals
  - FindAssets
  - GetEventsforDeal
  - RecordEvent
  - RecordAssetEvent
  - RollbackEvent
  - GetLatestClose
---

# Record lease events in LeaseAccelerator

Events drive lease state transitions — a lease moves from Active to Renewed, Evergreen, Terminated, or
Disposed because an event was recorded against it. Events generate accounting. Most of them cannot be
undone.

Treat this flow as irreversible unless you have specifically confirmed otherwise in step 5.

## Step 1 — find the target

- `FindDeals` — search deals by `<Criteria>`.
- `FindAssets` — search assets by `<Criteria>`.

There is no pagination. Narrow the criteria until the result set is what you intend to act on, and confirm
the count before proceeding.

## Step 2 — read the existing event history

Call `GetEventsforDeal` before recording anything. Two reasons:

1. You need to know the lease's current state — recording a renewal against an already-terminated lease is
   a data error, not a no-op.
2. If Segregation of Duties (SoD) is enabled for EOT events, an event may already be **pending approval**.
   Since release 26.3, nightly maintenance checks for a pending EOT approval before automatically
   recording a new end-of-term event, precisely to prevent duplicates. Your client should make the same
   check rather than relying on that guardrail.

## Step 3 — check the accounting period

Call `GetLatestClose`. Recording an event dated in a closed period produces catch-up adjusting entries in
the first open period. That may be correct — but it should be a decision, not a surprise discovered at
month end.

## Step 4 — record the event

- `RecordEvent` — records an event, or an intention for a future event, at the deal level.
- `RecordAssetEvent` — records an event at the asset level.

Event types include Terminate without Fee, Evergreen, Renewal, Return, Buyout, and Impairment.

    <APIRequest>
      <Request>
        <RequestId>your-tracking-id</RequestId>
        <WarningPolicy>Ignore</WarningPolicy>
        <ErrorPolicy>Stop</ErrorPolicy>
      </Request>
      <Payload>...event details...</Payload>
    </APIRequest>

Use `ErrorPolicy=Stop`. For an operation this consequential, a clean rollback of the whole request is what
you want — not a partial set of events scattered across a portfolio.

High-asset deals: as of release 26.3, selecting assets and recording all event types on deals with 1,500
or more assets works without timeouts. On earlier releases these calls could fail partway.

## Step 5 — what you can undo

`RollbackEvent` undoes a previously recorded event, but **only** for:

- End-of-term (EOT) events
- Payment adjustment events

Everything else is permanent once recorded. There is no rollback for an impairment, and no bulk undo for a
batch of events.

Before recording any event that is not an EOT or payment adjustment, confirm the intent explicitly. This is
the point in the API where a mistaken agent action cannot be walked back.

## Step 6 — verify

Call `GetEventsforDeal` again and confirm the event landed and the deal state changed as expected.

## Rules

- Always read `GetEventsforDeal` before recording, and check for a pending SoD approval.
- Always use `ErrorPolicy=Stop`.
- Never record a non-EOT, non-payment-adjustment event without explicit human confirmation — it cannot be
  rolled back.
- Never batch events across deals you have not individually verified.
- Read `Response/Status` from the XML envelope; HTTP 200 does not mean the event was recorded.
- Check `GetLatestClose` so you know whether the event will generate catch-up entries.
