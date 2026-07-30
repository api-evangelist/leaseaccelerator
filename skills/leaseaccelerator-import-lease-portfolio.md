---
name: leaseaccelerator-import-lease-portfolio
description: >-
  Import lease schedules and their assets into LeaseAccelerator safely — authenticate via SAML2, choose the
  right import operation, use external keys so retries do not duplicate, and read the validation results
  out of the XML response envelope.
api: leaseaccelerator:leaseaccelerator-api
generated: '2026-07-19'
method: generated
source: >-
  openapi/leaseaccelerator-api-openapi.yml,
  https://docs-leaseaccelerator.insightsoftware.com/hc/en-us/articles/33895853166093-API-Methods
operations:
  - FederateUser
  - ImportDeals
  - ImportantClassifyDeals
  - ModifyDealImport
  - ImportAssets
  - UpdateAssets
  - GetLatestClose
---

# Import a lease portfolio into LeaseAccelerator

This is the highest-consequence flow in the API. Imported deals become entries in a lease sub-ledger that
generates journal entries for the customer's general ledger. Treat every step as a financial write.

## Before you start

You need a SAML2 security token. There are no API keys.

1. Submit a SAML2 ECP GET request to `https://www.leaseaccelerator.com/auth/api`, identifying your
   identity provider and the credentials of a user already federated for access in LeaseAccelerator.
2. The response body is a text security token.
3. Send it as the `token` form field on every POST below.

See `authentication/leaseaccelerator-authentication.yml` for the full flow.

## Step 1 — check the accounting period

Call `GetLatestClose` (empty payload) to fetch the most recent month-end close date.

If any lease you are about to import has a Lease Start Date before that close date, the import will
succeed but LeaseAccelerator will raise the warning *"Lease starts in a closed period; catch up entries
will be posted in the first open period"* and post catch-up adjusting entries. That is expected behavior,
not an error — but confirm with the accounting owner before proceeding, because those entries land in the
current open period.

## Step 2 — choose the import operation

- `ImportDeals` — imports lease schedule and asset information. Does **not** run the classification
  engine.
- `ImportantClassifyDeals` — identical to `ImportDeals` except it also engages the automated
  LeaseAccelerator Classification Engine. Use this when you want LeaseAccelerator to classify the lease on
  import.
- `ModifyDealImport` — modifies an existing lease schedule in a single operation. Does **not** run the
  Classification Engine.
- `ImportAssets` — refreshes asset-level detail for a specific deal. Only one deal per request.
- `UpdateAssets` — updates existing assets.

Note the spelling of `ImportantClassifyDeals`. That is the operation key as published; it is not a typo you
should correct.

## Step 3 — build the request

POST to `https://www.leaseaccelerator.com/lease_accelerator/api/LeaseAccelerator/{operation}` with form
fields `token` and `file`, where `file` is:

    <APIRequest>
      <Request>
        <RequestId>a-unique-id-you-generate</RequestId>
        <WarningPolicy>Ignore</WarningPolicy>
        <ErrorPolicy>Stop</ErrorPolicy>
      </Request>
      <Payload>...deal and asset records...</Payload>
    </APIRequest>

Set `ErrorPolicy` deliberately:

- `Stop` (the default) — any error halts processing and rolls back everything already done for this
  request. Use this for a first import: a clean all-or-nothing failure is far easier to recover from than a
  partially imported portfolio.
- `Skip` — the offending record is skipped and the rest are processed. Only use this once you have already
  seen the validation output and consciously decided to accept a partial load. Not supported for
  single-record operations.

`WarningPolicy` defaults to `Ignore`, which processes everything and still reports every warning in the
response. `Skip` drops warned records; `Stop` rolls the whole request back on any warning.

## Step 4 — make the import idempotent

There is no idempotency-key header. Instead, put your own identifier on every record:

- `ImportKey` — your system's identifier for this record.
- `ImportedFrom` — a name for your system, maximum 32 characters.

If the LeaseAccelerator surrogate id is omitted but the `ImportKey` resolves to a known record,
LeaseAccelerator updates that record instead of creating a duplicate. **Always send a stable `ImportKey`.**
Without one, a retried import creates a second copy of every lease, and there is no bulk undo.

## Step 5 — read the response

The HTTP status will be 200 even when the import failed. Parse the envelope:

    <APIResponse>
      <Response>
        <RequestId>...</RequestId>
        <Status>0</Status>
        <Context>Ok</Context>
      </Response>
      <Payload>...</Payload>
    </APIResponse>

- `Status` of `0` and `Context` of `Ok` mean success. Any other `Status` is an error, described by
  `Context`.
- The `Payload` details every warning and error per record, regardless of which policies you chose.

Validation messages are text, not codes. Common blocking errors on a portfolio import:

| Message | What to fix |
|---|---|
| `Schedule does not exist` | Wrong Schedule Number, or leading/trailing whitespace. |
| `GL Code must be provided` | Populate the GL Code (Coding Convention). |
| `Cost center is required for assets` | Cost Center is mandatory on every asset. |
| `The entered GL Code is not configured in all ledgers the deal is being booked into` | Configure the Coding Convention in every target ledger, with all five required account fields. |
| `No payment information specified` | Populate either the LRF or the Payment. |
| `Invalid ShipToKey` | The Facility Code does not match a configured ShipTo address. |
| `Both ShipTo key and address specified` | Use a Facility Code **or** address fields, never both. |
| `Number of total payments does not match duration` | Step payments must total the schedule Duration. |
| `Missing first step on step payment` | Step payments must start at Payment Number 1. |

The full catalog is in `errors/leaseaccelerator-problem-types.yml`.

## Step 6 — verify

Call `FindDeals` with a `<Criteria>` element matching what you imported and confirm the count and the deal
states. There is no pagination, so constrain the criteria narrowly rather than expecting to page through
results.

## Rules

- Never retry an import without a stable `ImportKey` on every record.
- Never use `ErrorPolicy=Skip` on a first-time import.
- Never import into a closed period without confirming the catch-up entries are acceptable.
- Do not treat HTTP 200 as success. Read `Response/Status`.
- Prerequisite reference data — companies, people, addresses, cost centers, GL codes — must exist before
  the portfolio import. Load it first with the reference-data skill.
