---
name: Subscribe to Certify credentialing webhooks
description: >-
  Register a webhook on either Certify API surface, verify it with a test event,
  de-duplicate deliveries, and replay events that were missed.
api: openapi/certify-api-service-openapi.yml
operations:
- createWebhook
- webhookFindMany
- webhookDelete
- triggerManualWebhookEvent
- replayWebhooks
- WebhooksController_create
- WebhooksController_fetchAll
- WebhooksController_remove
- WebhooksController_testWebhook
generated: '2026-08-09'
method: generated
source: >-
  Grounded in openapi/certify-api-service-openapi.yml and
  openapi/certify-application-openapi.json; event catalog in
  asyncapi/certify-webhooks.yml. Every operationId verified against the spec.
---

# Subscribe to Certify credentialing webhooks

Certify has **two independent webhook surfaces**. Pick the one that matches the
API you integrate against — a subscription on one does not deliver the other's
events.

| Surface | Register | Event vocabulary |
|---|---|---|
| ProviderHub (`api-service`) | `createWebhook` `POST /webhooks` | 2 dotted types, e.g. `credential_workflow.status.changed` |
| CertifyOS v2 (`application`) | `WebhooksController_create` `POST /v2/webhooks` | 34 SCREAMING_SNAKE types, e.g. `CREDENTIALING_COMPLETED` |

The full catalog of all 36 event types with their published descriptions is in
`asyncapi/certify-webhooks.yml`.

## 1. Register

**ProviderHub** — `createWebhook` takes `webhookUrl` and `eventTypes[]`, plus
optional `oauthConfiguration` and custom headers. You may subscribe to multiple
event types in one webhook. The URL must match `^https?://[^\s]+$`.

**v2** — `WebhooksController_create` takes `url` and `event_types[]`, scoped by
the `organization-id` header.

A `400` here is a validation failure — empty `eventTypes`, mixed event types, or
a malformed URL.

## 2. Secure the callback

Certify supports **OAuth2 client credentials on the subscription itself**: supply
`tokenUrl`, `clientId`, `clientSecret` and optionally `scope`, and Certify fetches
a token before each delivery and sends `Authorization: Bearer <token>` to your
endpoint. Prefer this over a shared secret in the URL. Any additional custom
headers configured on the subscription are sent with every delivery.

Note: Certify does **not** publish an HMAC signature header. Endpoint
authentication is the OAuth configuration above plus TLS.

## 3. Verify before you trust it

- ProviderHub: **`triggerManualWebhookEvent`**
  (`POST /webhooks/test/publish`) publishes a real event to your registered
  subscriptions. `eventData` is validated against the JSON Schema for the event
  type; an invalid payload returns `400` with detailed validation errors.
- v2: **`WebhooksController_testWebhook`**
  (`POST /v2/webhooks/{webhookId}/test-event`).

The published payload schema for `credential_workflow.status.changed` is at
`json-schema/webhook-events/credential-workflow-status-changed.schema.json`
(canonical URL on `schemas.certifyos.com`). The facility equivalent is referenced
by the docs but **returns 404** — do not build against a schema you cannot fetch.

## 4. Handle the delivery

Every delivery arrives as `POST` with:

```
Content-Type: application/json
User-Agent: CertifyOS-Webhook-Delivery/1.0
Idempotency-Key: <SHA-256 hash>
```

Body:

```json
{ "eventType": "credential_workflow.status.changed",
  "timestamp": "2024-01-20T09:15:00Z",
  "data": { }  }
```

**De-duplicate on `Idempotency-Key`.** It is stable across retries of the same
event, and Certify dispatches through Google Cloud Tasks, which retries whenever
your endpoint does not return 2xx. Treat delivery as at-least-once.

## 5. Recover missed events

If your endpoint was down, call **`replayWebhooks`**
(`POST /credentialing-workflows/replay-webhooks`) with the affected
`workflowIds` — 1 to 500 per call. This is the supported backfill path; do not
sweep the API to reconstruct state.

## 6. Housekeeping

List with **`webhookFindMany`** (`GET /webhooks`) or
**`WebhooksController_fetchAll`** (`GET /v2/webhooks`); remove with
**`webhookDelete`** (`DELETE /webhooks/{id}`) or **`WebhooksController_remove`**
(`DELETE /v2/webhooks`).
