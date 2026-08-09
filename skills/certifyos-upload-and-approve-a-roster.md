---
name: Upload, map, validate and approve a provider roster
description: >-
  Push a bulk provider roster CSV into Certify, map its columns to a template,
  fix validation failures, and approve the import.
api: openapi/certify-roster-service-openapi.yml
operations:
- uploadCsvFile
- getMappingsForTheUploadInAGivenTemplate
- createTemplateFromMappings
- getAllRosters
- getRosterById
- getRosterRowsPaged
- revalidateRosterDraftRows
- saveRosterRowsDraft
- triggerSmartFix
- approveRejectRosterImport
- reuploadCsvFile
- downloadRosterRecordsCsv
generated: '2026-08-09'
method: generated
source: >-
  Grounded in openapi/certify-roster-service-openapi.yml and
  openapi/certify-api-service-openapi.yml; every operationId verified against the
  spec.
---

# Upload, map, validate and approve a provider roster

Roster ingestion is Certify's bulk path — it is how a payer or health system
loads hundreds of providers at once instead of calling `practitionerCreate` in a
loop. Prerequisite: **`certify-authenticate-and-page`**.

## 1. Upload the CSV

Call **`uploadCsvFile`** (`POST /roster`, `multipart/form-data`). A `415
Unsupported Media Type` means the file type was rejected — the endpoint accepts
CSV.

## 2. Map columns to a template

Call **`getMappingsForTheUploadInAGivenTemplate`**
(`POST /roster-upload/{templateId}/column-mapping`) to align your CSV headers to
an existing roster template. If no template fits, promote your mapping into a
reusable one with **`createTemplateFromMappings`**
(`POST /roster-upload/{templateId}/column-mapping/template`).

Inspect the resolved shape with **`getRosterEffectiveSchema`**
(`GET /roster/{id}/schema`) before you trust the mapping.

## 3. Review the staged rows

- **`getRosterById`** (`GET /roster/{id}`) for the import's state.
- **`getRosterRowsPaged`** (`GET /roster-records/{rosterId}/paged`) to walk the
  staged rows — use paging, these files are large.

## 4. Fix and revalidate

- **`saveRosterRowsDraft`** (`PATCH /roster-records/{rosterId}/draft`) to correct
  rows in place.
- **`triggerSmartFix`** (`POST /roster-records/{rosterId}/smartfix`) to let
  Certify auto-correct what it can.
- **`revalidateRosterDraftRows`** (`POST /roster-records/{rosterId}/revalidate`)
  to re-run validation after edits.

If the source file itself was wrong, replace it with **`reuploadCsvFile`**
(`PUT /roster/{rosterId}/reupload`) rather than starting a new import.

## 5. Approve or reject

Call **`approveRejectRosterImport`** (`GET /roster/{rosterId}/import`). This is
the commit point — approved rows become platform records.

## 6. Export

**`downloadRosterRecordsCsv`** (`GET /roster/{rosterId}/download-records`)
returns `text/csv`. This is one of only five operations across the whole surface
that returns CSV rather than JSON, so set your `Accept` handling accordingly.

## Notes

- Every call needs the `tenant-id` header.
- `roster-service` publishes no production host in its `servers[]` (only
  localhost, staging and test); use `https://api-service.certifyos.com`, which is
  the monitored production component on the status page.
- There is no idempotency key on the upload. Re-POSTing the same CSV creates a
  second import — use `getAllRosters` (`GET /roster`) to check before re-uploading.
