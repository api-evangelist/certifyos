---
name: Authenticate and page the Certify ProviderHub API
description: >-
  Mint a Certify bearer JWT, set the required tenant scope header, and walk a
  paginated list endpoint safely — the prerequisite for every other Certify
  skill.
api: openapi/certify-api-service-openapi.yml
operations:
- authenticateUser
- clientCredentials
- practitionerFindMany
- AuthTokensController_createApiToken
generated: '2026-08-09'
method: generated
source: >-
  Grounded in openapi/certify-api-service-openapi.yml,
  openapi/certify-application-openapi.json and
  conventions/certify-conventions.yml. Every operationId was verified against the
  spec.
---

# Authenticate and page the Certify ProviderHub API

Base URL: `https://api-service.certifyos.com` (ProviderHub) or
`https://ng-api-production.certifyos.com` (CertifyOS v2).

## 1. Get a token

For a human/service user on ProviderHub, call **`authenticateUser`**
(`POST /auth/login`). For a machine integration, call **`clientCredentials`**
(`POST /auth/client-credentials`). On the v2 application API, mint a long-lived
API token with **`AuthTokensController_createApiToken`**
(`POST /auth-tokens/v2/api`).

## 2. Send the token the way this API expects it

This is the single most common integration failure with Certify:

- **ProviderHub (`api-service`, `roster-service`)** — the `jwt` security scheme
  is documented as *"Provide only the raw token without Bearer prefix"*. Send
  `Authorization: <token>`, **not** `Authorization: Bearer <token>`.
- **CertifyOS v2 (`application`)** — uses conventional
  `Authorization: Bearer <token>`.

A `401` almost always means the prefix is wrong or the token expired; a `403`
means the token is valid but the role lacks the permission, or the tenant header
does not match the token's tenant. Neither is retryable unchanged.

## 3. Always send the tenant scope header

Every ProviderHub request requires `tenant-id`. All 377 api-service operations
and all 64 roster-service operations declare it as a **required** header. The v2
API uses `organization-id` instead. Omit it and the request fails — there is no
default tenant.

## 4. Page the list

Call **`practitionerFindMany`** (`GET /practitioners`).

- Page-number paging: `page` and `size`. The response carries `totalCount` and
  `data`.
- Keyset paging: `startAfterId` / `endAtId` on the higher-volume lists — prefer
  this over deep `page` values.
- Ordering: `order`, and on roster-service `sortBy` / `sortDirection`.
- Filtering: the `filter` query parameter takes **URL-encoded JSON** with
  operators `eq`, `gte`, `contains`, `in`.
- On the v2 API, paging is `offset` / `limit` instead.

## 5. Handle errors

Certify does **not** use RFC 9457. Parse `errors[]` — on ProviderHub each item is
`{ httpStatus, reason, title, detail }`; on `400` responses it is often a flat
list of strings under `errors` plus structured `errorDetails`. See
`errors/certify-problem-types.yml`.

## 6. Retry rules

- No operation accepts an `Idempotency-Key`. **Do not blind-retry a POST** — a
  retried create may duplicate the resource.
- `500` / `502` / `503` are retryable with backoff. `502` and `503` usually mean
  a downstream dependency (the DAL, CAQH, NPPES) is degraded — check
  <https://status.certifyos.com/>, which monitors those upstreams as components.
- No rate-limit headers and no `429` are declared, so back off on `5xx` rather
  than expecting a `Retry-After`.
