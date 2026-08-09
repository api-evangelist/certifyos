---
name: Monitor a network and triage compliance flags
description: >-
  Watch Certify's continuous monitoring workflows, search and triage the
  exclusion/sanction flags they raise, and mark providers reviewed.
api: openapi/certify-application-openapi.json
operations:
- getMonitoringWorkflows
- getMonitoringWorkflow
- bulkAssignMonitoringWorkflows
- FlagsController_searchFlags
- FlagsController_findAllFlags
- FlagsController_markRead
- FlagsController_markUnread
- FlagsController_markProviderReviewedAndApproved
- FlagsController_activateFlag
- FlagsController_deactivateFlag
- calculateFlags
generated: '2026-08-09'
method: generated
source: >-
  Grounded in openapi/certify-application-openapi.json and
  openapi/certify-api-service-openapi.yml; every operationId verified against the
  spec.
---

# Monitor a network and triage compliance flags

Credentialing is point-in-time; **monitoring** is the continuous half of the
product — Certify re-checks providers against primary sources between
credentialing cycles and raises **flags** when something changes (sanction,
exclusion, license lapse). This skill covers reading that stream.

Note the surface split: **monitoring workflows** live on the ProviderHub
`api-service` (`tenant-id` header); **flags** live on the CertifyOS v2
application API (`organization-id` header). You will hold two tokens.

## 1. Enumerate monitoring workflows

**`getMonitoringWorkflows`** (`GET /monitoring-workflows`) lists them;
**`getMonitoringWorkflow`** (`GET /monitoring-workflows/{id}`) reads one. Runs
are exposed as `pipeline-runs` sub-resources — a run is one execution of the
monitoring pipeline against the provider set.

Assign work in bulk with **`bulkAssignMonitoringWorkflows`**
(`POST /monitoring-workflows/assignments`) rather than one call per workflow.

## 2. Find the flags

- **`FlagsController_searchFlags`** — targeted search.
- **`FlagsController_findAllFlags`** — full sweep, paged with `offset` / `limit`.
- **`ProviderFlagsController_findFlags`** — flags for a single provider.

Filter with `flagReadStatus` to separate the new from the already-handled.

## 3. Triage

- **`FlagsController_markRead`** / **`FlagsController_markUnread`** — and the
  bulk forms `FlagsV2Controller_markReadBulk` / `FlagsController_markUnreadBulk`
  when you are clearing a batch.
- **`FlagsController_activateFlag`** / **`FlagsController_deactivateFlag`** —
  change the flag's own state, not just your read marker. Deactivating a flag is
  a compliance-consequential write; log who did it on your side, because Certify
  publishes no audit endpoint for flag state.
- **`FlagsController_markProviderReviewedAndApproved`** — the human sign-off that
  closes the loop on a provider.

## 4. Recalculate

**`calculateFlags`** re-runs flag computation. Treat it as expensive and
non-idempotent — there is no `Idempotency-Key` on this API, and no rate limit is
published, so call it on a schedule you control rather than in a retry loop.

## 5. Prefer events over polling

The v2 webhook vocabulary carries the provider and facility lifecycle
transitions (`PROVIDER_IN_PROGRESS`, `PROVIDER_PSV_COMPLETED`,
`PROVIDER_CRED_APPROVED`, `PROVIDER_TERMINATED`, and the facility equivalents —
34 in total, listed in `asyncapi/certify-webhooks.yml`). Subscribe with
`certify-subscribe-to-credentialing-events` and use these read endpoints to
resolve detail after an event, rather than sweeping on a timer.

## Caveat

There is no published event for a *flag being raised* — the 36 documented event
types cover credentialing and facility workflow status plus form submission
only. Flag discovery is poll-only today; that is a genuine gap in the event
surface, not a missing lookup on your side.
