---
name: leaseaccelerator-generate-and-retrieve-reports
description: >-
  Extract data from LeaseAccelerator as reports and ledger exports — check that sweeping and booking have
  settled, submit an asynchronous generation request, poll it, and retrieve the file.
api: leaseaccelerator:leaseaccelerator-api
generated: '2026-07-19'
method: generated
source: >-
  openapi/leaseaccelerator-api-openapi.yml,
  https://docs-leaseaccelerator.insightsoftware.com/hc/en-us/articles/33895853166093-API-Methods
operations:
  - GetReportableStatus
  - GetSweepingStatus
  - GetBookingStatus
  - GetLatestClose
  - GenerateAsynch
  - GetReportStatus
  - GetReportFile
  - Generate
---

# Generate and retrieve LeaseAccelerator reports

Reporting is how you get bulk data *out* of LeaseAccelerator. There is no pagination on the search
operations, so anything larger than a targeted lookup should go through this flow. Ledger exports and
asset-level detail reports are the standard mechanism for feeding an ERP.

This is a read-only flow. Generation produces a file; it does not mutate lease or accounting data.

## Step 1 — check it is safe to report

Background processing moves data through LeaseAccelerator, and a report run mid-flight returns an
inconsistent picture. Check before you generate.

- `GetReportableStatus` — the one to call. Returns the status of sweeping **and** booking tasks together,
  specifically so a caller can determine whether it is safe to run a report. Empty request payload.
- `GetSweepingStatus` — sweeping tasks only. Empty payload.
- `GetBookingStatus` — booking tasks only. Empty payload.
- `GetLatestClose` — the most recent month-end close date. Empty payload. Use this to pin an as-at date
  that will not shift under you.

Do not skip this step for period-end reporting.

## Step 2 — submit the generation request

Prefer `GenerateAsynch` over `Generate`.

- `Generate` runs synchronously. It works, but a large ledger export will hold the connection open for as
  long as it takes.
- `GenerateAsynch` takes the same parameters and returns a request id immediately.

POST to `https://www.leaseaccelerator.com/lease_accelerator/api/LeaseAccelerator/GenerateAsynch` with the
`token` and `file` form fields. The `<Payload>` carries the set of parameters for the document or report
you want.

    <APIRequest>
      <Request>
        <RequestId>your-tracking-id</RequestId>
        <WarningPolicy>Ignore</WarningPolicy>
        <ErrorPolicy>Stop</ErrorPolicy>
      </Request>
      <Payload>...report parameters...</Payload>
    </APIRequest>

Capture the request id from the response payload. Everything downstream keys off it.

## Step 3 — poll

Call `GetReportStatus` with the request id returned by `GenerateAsynch`. Poll until it reports completion.

There are no webhooks and no callback — polling is the only completion signal the API offers. Back off
between polls; the API publishes no rate limits, so apply your own client-side throttle rather than
assuming a generous budget.

## Step 4 — fetch the file

Call `GetReportFile` with the same request id. This is the documented mechanism for transferring data from
LeaseAccelerator to external systems.

## Step 5 — check the envelope, every time

Every one of these operations returns HTTP 200 whether it worked or not. Read `Response/Status` — `0` with
`Context` of `Ok` means success; anything else is an error described by `Context`.

## The file-based alternative

For recurring extraction, the API is not always the right tool. LeaseAccelerator's Reporting Engine can
transfer generated reports automatically over SFTP on a user-defined schedule (daily, weekly, monthly), in
XML, TSV, CSV, pipe-delimited, fixed-width, or XLSX, optionally PGP-encrypted. If the requirement is "the
same ledger export lands in our data lake every night," scheduled SFTP delivery is the intended path and
needs no client code at all. See `conventions/leaseaccelerator-conventions.yml`.

## Rules

- Always call `GetReportableStatus` before a period-end report.
- Use `GenerateAsynch` for anything large; keep `Generate` for small, interactive requests.
- Never assume HTTP 200 means the report generated.
- Do not poll tightly. There is no published rate limit, which is not the same as no limit.
- Retired reports: the User Activity Log and Transaction Report were retired in release 26.3. Use the v2
  versions. Check `changelog/leaseaccelerator-changelog.yml` before wiring a report name into automation.
