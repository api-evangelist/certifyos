---
name: Credential a practitioner end to end
description: >-
  Create or find a practitioner, open a credentialing workflow, watch its
  timeline and steps, and pull the PSV packet when it completes.
api: openapi/certify-api-service-openapi.yml
operations:
- practitionerFindMany
- practitionerCreate
- practitionerGet
- createCredentialingWorkflow
- createBulkCredentialingWorkflows
- getCredentialingWorkflows
- getCredentialingWorkflow
- getTimelineEvents
- getPractitionerGroups
generated: '2026-08-09'
method: generated
source: >-
  Grounded in openapi/certify-api-service-openapi.yml; every operationId verified
  against the spec. Conventions from conventions/certify-conventions.yml.
---

# Credential a practitioner end to end

Prerequisite: **`certify-authenticate-and-page`** — a raw JWT in `Authorization`
and a `tenant-id` header on every call.

## 1. Find the practitioner before you create one

Call **`practitionerFindMany`** (`GET /practitioners`) filtered on `npi` or your
own `externalId` before writing. There is **no request idempotency** on this API,
so a duplicate create is a real risk. Use the `filter` parameter with
URL-encoded JSON, e.g. an `eq` on `npi`.

## 2. Create the practitioner if absent

Call **`practitionerCreate`** (`POST /practitioners`). Carry your own key in
`externalId` so the lookup in step 1 is cheap on every subsequent run. The
Practitioner entity's published schema is
`json-schema/entities/Practitioner.schema.json` (canonical URL
`https://schemas.certifyos.com/entities/Practitioner.schema.json`) — validate
your payload against it locally before sending.

A `409` here means a uniqueness conflict: re-read and reconcile, do not retry.

## 3. Open the credentialing workflow

Call **`createCredentialingWorkflow`** (`POST /credentialing-workflows`). For a
batch, use **`createBulkCredentialingWorkflows`**
(`POST /credentialing-workflows/bulk-create`) rather than looping — it is a
single tenant-scoped call.

A `422` at this step commonly means *"Initiating tenant outreach template missing
or incomplete"* — the tenant's outreach template must exist before a workflow
can start. Fix the precondition; do not retry.

## 4. Track progress

- **`getCredentialingWorkflow`** (`GET /credentialing-workflows/{id}`) for
  current state.
- **`getTimelineEvents`** (`GET /credentialing-workflows/{workflowId}/timeline-events`)
  for the append-only audit trail. A new timeline event is exactly what fires the
  `credential_workflow.status.changed` webhook, so the timeline and the event
  stream are the same fact viewed two ways.
- **`getCredentialingWorkflows`** (`GET /credentialing-workflows`) to sweep the
  queue by status.

**Do not poll hard.** Subscribe instead — see
`certify-subscribe-to-credentialing-events`. Certify publishes no rate limits, so
a polling loop has no documented ceiling to respect and will look like abuse.

## 5. Retrieve outputs

The workflow exposes generated artifacts as sub-resources under
`/credentialing-workflows/{id}` — PSV generation, the latest PSV PDF, NPDB,
board-certification and sanction PDFs, attachments, and supporting documents.
Read them from the workflow's sub-paths rather than reconstructing them.

## 6. Reconcile affiliations

Call **`getPractitionerGroups`** (`GET /practitioners/{practitionerId}/groups`)
to confirm the practitioner is attached to the right groups. Certify models
participation at the intersection of practitioner x location x group x network x
specialty — see `data-model/certify-data-model.yml` before assuming a flat
provider-to-group relationship.
