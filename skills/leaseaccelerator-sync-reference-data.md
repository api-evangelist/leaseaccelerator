---
name: leaseaccelerator-sync-reference-data
description: >-
  Keep LeaseAccelerator's companies, people, addresses, cost centers, and FX rates in sync with an upstream
  master-data system, using external keys for upsert and controlling how far back changes are applied.
api: leaseaccelerator:leaseaccelerator-api
generated: '2026-07-19'
method: generated
source: >-
  openapi/leaseaccelerator-api-openapi.yml,
  https://docs-leaseaccelerator.insightsoftware.com/hc/en-us/articles/33895853166093-API-Methods
operations:
  - AddUpdateCompany
  - AddUpdatePerson
  - AddUpdateAddress
  - AddUpdateCostCenters
  - ImportCompanies
  - ImportPeople
  - ImportAddresses
  - ImportExchangeRates
  - DefineReferenceData
  - FindContacts
---

# Synchronize reference data into LeaseAccelerator

Reference data — companies, people, addresses, cost centers, GL codes, FX rates — must exist in
LeaseAccelerator before any lease portfolio referencing it can be imported. This skill covers keeping it
current from an upstream system of record.

The distinguishing risk here is **retroactivity**: a reference-data edit can rewrite history across every
deal the record participates in. Read step 3 before writing anything.

## Step 1 — pick Add/Update, not Import

Two families exist and they are not interchangeable.

| Family | Behavior |
|---|---|
| `AddUpdateCompany`, `AddUpdatePerson`, `AddUpdateAddress`, `AddUpdateCostCenters` | Adds new records **and updates existing ones**. |
| `ImportCompanies`, `ImportPeople`, `ImportAddresses` | Adds new records only. Mirrors the CIW workbook tabs. |

For an ongoing sync you want the `AddUpdate*` family. `ImportCompanies` on an existing company returns the
error `Company Exists` — you cannot update company information through the import path.

`ImportExchangeRates` loads FX rates. `DefineReferenceData` defines configured reference values.

## Step 2 — key every record externally

This is what makes the sync idempotent. On every record send:

- `ImportKey` — the upstream system's identifier for this record.
- `ImportedFrom` — a name for the upstream system, maximum 32 characters.

Omit the LeaseAccelerator surrogate id (`CompanyId`, `AddressId`, `PartyId`). LeaseAccelerator will look up
the `ImportKey`; if it resolves to a known record it updates that record, otherwise it creates a new one.
This is the documented mechanism for external systems to perform adds and updates using their own
identifiers, and it means a replayed sync converges rather than duplicating.

If you send neither a surrogate id nor an `ImportKey`, every run creates new records.

## Step 3 — decide how far back the change applies

Reference-data writes accept `UpdateDealStatus`, which declares which deal states an update should
propagate to:

- `None` — the change applies to future deals only.
- `All` — the change is applied **retroactively to every deal** in which the record is a participant.
- A comma-separated list of `PreOrigination`, `Active`, `Renewed`, `Evergreen`, `Terminated`, `Disposed`.

`All` on a company or address record can rewrite participant data across a closed accounting period. Do not
default to it. For a routine master-data sync, `None` or a narrow list such as `PreOrigination,Active` is
almost always the right answer, and anything wider should be an explicit, reviewed decision.

## Step 4 — respect the field constraints

Consistency rules that produce hard errors:

- A company can only be configured with one parent company. Every time you list a company, all information
  except Company Role Type, Ledger Code, and Functional Currency must be identical — including Parent Name.
- Company Role Type must be one of Lessee, Entity, Funder, Vendor, or SBU, and the Company name must match
  the Companies tab exactly.
- An address needs at minimum City and Country, or you get `Insufficient ship to address specified`.
- The country must already be configured as a GEO, or you get `Invalid Geographic Area`.
- An existing address used as a ship-to must have the ShipTo box checked, or you get
  `Ship to address already exists but is not configured as a ship to address`.
- Custom participant roles must be configured before use, or you get `Invalid role`.

Field limits worth honoring upstream: `ImportedFrom` 32 characters; company name 150; address lines 250;
city 100; postal code 16; AncillaryField Attribute and Value 32 each.

## Step 5 — carry your own attributes

Use `<AncillaryFields>` to attach data LeaseAccelerator has no native field for — repeated
`<AncillaryField>` entries of `<Attribute>` and `<Value>`, 32 characters each. This is where the
chart-of-accounts code for a participant company usually lives. Custom attribute types can be configured as
needed.

## Step 6 — verify

`FindContacts` retrieves contact information by criteria. Use it to confirm a sync round-trip before
trusting the next portfolio import against it.

## Rules

- Always send `ImportKey` and `ImportedFrom`; never rely on name matching.
- Never send `UpdateDealStatus=All` without an explicit reason and sign-off.
- Use `AddUpdate*` for sync; `Import*` only for genuinely new records.
- Load reference data before the portfolio import that depends on it, not after.
- Read `Response/Status` from the XML envelope. HTTP 200 does not mean the record was written.
