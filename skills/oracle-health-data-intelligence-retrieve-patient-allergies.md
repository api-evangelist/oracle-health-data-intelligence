---
name: retrieve-patient-allergies
description: >-
  Retrieve the aggregated allergy list for a patient from the Oracle Health Data Intelligence
  longitudinal record, paging safely and handling the platform's throttling and identifier-
  stability rules. Read-only; returns protected health information.
api: Oracle Health Data Intelligence Populations API
spec: openapi/oracle-health-data-intelligence-populations-api-openapi.yml
operations:
  - listAllergies
  - getAllergy
consequence: read
phi: true
generated: '2026-08-27'
method: generated
source: >-
  openapi/oracle-health-data-intelligence-populations-api-openapi.yml plus
  https://docs.healtheintent.com/api/v1/allergy/
---

# Retrieve a patient's allergies

## Before you start

**This returns PHI.** Oracle does not apply user-level authorization or filtering to these APIs.
They are a business-to-business surface and the calling system is responsible for deciding whether
the human on whose behalf you are acting is entitled to see this patient's record. Do not run this
skill without that check having already happened upstream.

You need three things:

1. A system-account credential — either a bearer token or OAuth 1.0a consumer key and secret,
   authorized for the Allergy API on this tenant in the Health Data Intelligence Console.
2. A `populationId`.
3. A `patientId` **for this platform**, which is not the same as the patient's ID in the EHR.

## Step 1 — Resolve the patient ID, every time

Health Data Intelligence patient IDs are not stable. Oracle states that a patient's unique ID may
change as new source data is aggregated into the longitudinal record, and that systems should not
store them locally for long-term use.

Resolve the ID at the start of the workflow from the source system's local person ID plus its data
partition ID, using the Patient API lookup
(`https://docs.healtheintent.com/api/v1/patient#health-data-intelligence-patient-id-lookup`). Do
not reuse a cached ID from a previous run. If your resolution step returns more than one match,
stop and escalate — do not guess which patient is meant.

## Step 2 — List the allergies

Call `listAllergies`:

    GET /populations/{populationId}/patients/{patientId}/allergies?limit=100

Base URL is `https://{tenant}.api.{region}.healtheintent.com/allergy/v1`.

Optional query parameters:

- `limit` — 1 to 100, default 20.
- `status` — repeatable; filters by allergy status.
- `cursor` — supplied only when following a page, see Step 3.

## Step 3 — Page to completion

The response is `{ items, firstLink, nextLink }`. `nextLink` is a fully qualified absolute URL
including the tenant host. **Follow it verbatim.** Do not rebuild it from the cursor value, and do
not assume the host stays the same.

Stop when `nextLink` is absent. That, not an empty `items` array, is the end of the collection.

An allergy list is a clinical safety artifact: a partial list read as a complete one is a real
hazard. If paging is interrupted for any reason, report the result as incomplete rather than
returning what you have.

## Step 4 — Read a single allergy when you need the detail

Call `getAllergy`:

    GET /populations/{populationId}/patients/{patientId}/allergies/{allergyId}

Use an `id` taken from a `listAllergies` response. In the `cernerdemo` sandbox tenant the seeded
data changes, so IDs published in Oracle's own documentation examples are frequently stale — always
list first, then fetch.

## Reading the response

Each `Allergy` carries `id`, `assertedOn`, `onset`, `resolvedOn`, and a set of `CodeableConcept`
fields: `status`, `type`, `category`, `code`, `criticality`. A `CodeableConcept` holds `codings`,
`sourceCodings`, `concepts` and a `text` fallback. Prefer a coding; fall back to `text` only for
display, never for logic.

`reactions[]` pairs a `reaction` with a `severity`. `sourceIdentifier` gives the
`dataPartitionId` and local `id` in the contributing system, which is how you trace a record back
to its source. `provenances[]` records where the assertion came from.

These types resemble FHIR but are not FHIR. Do not hand an `Allergy` to a FHIR-typed consumer.

## Errors

The envelope is `{ code, message, errorDetails[] }`, where each detail carries `domain`, `reason`,
`message`, `locationType` and `location`.

- `401` — the Authorization header is missing or malformed. Do not retry with the same header.
- `403` — the credential is valid but the system account is not authorized for this API or
  resource in the Health Data Intelligence Console. This is an administrative grant, not something
  a retry fixes. Escalate to a human.
- `404` — the population, patient or allergy does not exist. Re-run Step 1 before concluding the
  patient has no record; a stale patient ID surfaces here.
- `429` — Oracle is throttling. Retry the original request with exponential backoff. There is no
  `Retry-After` header to read, so choose your own increasing delays.
- `5xx` — retry with backoff. If it persists, report the `cerner-correlation-id` response header to
  Oracle support; it is the only handle they can trace.

## Do not

- Do not cache patient IDs across sessions.
- Do not write. This API is read-only, and the platform as a whole does not permit writes to core
  clinical resources.
- Do not send a request without an idempotency assumption check — there is no idempotency support
  here, so any retry logic you build must be safe on a read-only path only.
- Do not log the response body anywhere that is not a PHI-approved store.
