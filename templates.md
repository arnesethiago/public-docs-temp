# Plati API - WhatsApp Templates Guide

This guide explains how to create, manage, and send WhatsApp templates using the Plati workspace templates API.

Templates are defined once at the workspace level and automatically synced to every active WhatsApp Business channel. Meta approval happens asynchronously — use the detail and variables endpoints to track status before sending.

## Authentication

All requests require an API Key passed in the `x-api-key` header:

```bash
x-api-key: your-api-key-here
```

Bearer token authentication is also supported.

**Note:** Template management endpoints require workspace admin permissions. API Key callers with workspace context are accepted.

## Base URL

```
https://api.plati.ai
```

---

## Table of Contents

1. [Create Template](#1-create-template)
2. [List Templates](#2-list-templates)
3. [Get Template Detail](#3-get-template-detail)
4. [Get Template Variables](#4-get-template-variables)
5. [Send Template](#5-send-template)
6. [Delete Template](#6-delete-template)
7. [Retry Failed Syncs](#7-retry-failed-syncs)

---

## 1. Create Template

Create a workspace-level template definition. Plati persists the record and enqueues a sync job for each active WhatsApp channel. The Meta API call runs asynchronously — the response returns immediately with `statusMaster: SYNCING`.

### Endpoint

```
POST /v1/workspace/templates
```

### Request Body

```json
{
  "name": "order_confirmation",
  "language": "pt_BR",
  "category": "UTILITY",
  "components": [
    {
      "type": "HEADER",
      "format": "TEXT",
      "text": "Pedido confirmado"
    },
    {
      "type": "BODY",
      "text": "Olá {{customer_name}}, seu pedido {{order_id}} foi confirmado!"
    },
    {
      "type": "FOOTER",
      "text": "Equipe Plati"
    },
    {
      "type": "BUTTONS",
      "buttons": [
        {
          "type": "URL",
          "text": "Rastrear pedido",
          "url": "https://app.example.com/track/{{order_id}}"
        }
      ]
    }
  ],
  "variableExamples": {
    "customer_name": "Ana",
    "order_id": "ABC123"
  }
}
```

### Request Body Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Template name **without** the `plati_` prefix. Lowercase `a-z`, `0-9`, `_` only. Max 80 chars. |
| `language` | string | Yes | Language code (e.g. `pt_BR`, `en_US`) |
| `category` | string | No | `UTILITY` or `MARKETING`. Default: `UTILITY` |
| `components` | array | Yes | Meta-shaped components: `HEADER`, `BODY`, `FOOTER`, `BUTTONS`, `CAROUSEL`, `FLOW` |
| `variableExamples` | object | Yes | Map of variable name → example value. Required for every `{{variable_name}}` placeholder. |
| `headerMedia` | object | No | Required when header format is `IMAGE`, `VIDEO`, or `DOCUMENT` |

### Variable Placeholders

Use **named** placeholders only:

```
✅ {{customer_name}}
✅ {{order_id}}
❌ {{1}}  (positional placeholders are rejected)
```

Variable names must match: `^[a-z][a-z0-9_]*$`

### Header Media

When the header uses media format, provide `headerMedia`:

```json
{
  "headerMedia": {
    "type": "IMAGE",
    "url": "https://cdn.example.com/header.jpg"
  },
  "components": [
    { "type": "HEADER", "format": "IMAGE" },
    { "type": "BODY", "text": "Seu pedido {{order_id}} está a caminho." }
  ],
  "variableExamples": { "order_id": "ABC123" }
}
```

Supported types: `IMAGE`, `VIDEO`, `DOCUMENT`. The URL must be publicly accessible by Meta.

### Example Request

```bash
curl -X POST "https://api.plati.ai/v1/workspace/templates" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "order_confirmation",
    "language": "pt_BR",
    "category": "UTILITY",
    "components": [
      {
        "type": "BODY",
        "text": "Olá {{customer_name}}, seu pedido {{order_id}} foi confirmado!"
      }
    ],
    "variableExamples": {
      "customer_name": "Ana",
      "order_id": "ABC123"
    }
  }'
```

### Response (201 Created)

```json
{
  "uid": "8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789",
  "name": "order_confirmation",
  "metaTemplateName": "plati_order_confirmation",
  "language": "pt_BR",
  "category": "UTILITY",
  "statusMaster": "SYNCING",
  "canDowngrade24h": true,
  "createdAt": "2024-01-15T10:00:00Z"
}
```

### Error Responses

| Status Code | Description |
|-------------|-------------|
| 400 | Validation error (positional placeholders, missing example, invalid name, etc.) |
| 403 | Caller is not a workspace admin |
| 409 | Template with same `(name, language)` already exists in workspace |

---

## 2. List Templates

List all workspace template definitions with cursor-based pagination.

### Endpoint

```
GET /v1/workspace/templates
```

### Query Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `limit` | number | Page size (1–50) | 20 |
| `cursor` | string | Cursor from a previous response (`nextCursor` / `previousCursor`) | - |
| `statusMaster` | string | Filter: `DRAFT`, `SYNCING`, `ACTIVE`, `PARTIAL`, `FAILED`, `DELETING` | - |
| `category` | string | Filter: `UTILITY` or `MARKETING` | - |
| `search` | string | Substring match against template name | - |

### Example Request

```bash
curl -X GET "https://api.plati.ai/v1/workspace/templates?limit=20&statusMaster=ACTIVE" \
  -H "x-api-key: your-api-key-here"
```

### Response (200 OK)

```json
{
  "templates": [
    {
      "uid": "8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789",
      "name": "order_confirmation",
      "metaTemplateName": "plati_order_confirmation",
      "language": "pt_BR",
      "category": "UTILITY",
      "statusMaster": "ACTIVE",
      "canDowngrade24h": true,
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ],
  "hasNext": false,
  "hasPrevious": false,
  "nextCursor": null,
  "previousCursor": null
}
```

---

## 3. Get Template Detail

Get a template definition with per-channel sync status. Use this to monitor Meta approval progress.

### Endpoint

```
GET /v1/workspace/templates/{uid}
```

### Path Parameters

- `uid` (required): Template UID

### Example Request

```bash
curl -X GET "https://api.plati.ai/v1/workspace/templates/8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789" \
  -H "x-api-key: your-api-key-here"
```

### Response (200 OK)

```json
{
  "definition": {
    "uid": "8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789",
    "name": "order_confirmation",
    "metaTemplateName": "plati_order_confirmation",
    "language": "pt_BR",
    "category": "UTILITY",
    "statusMaster": "ACTIVE",
    "canDowngrade24h": true,
    "createdAt": "2024-01-15T10:00:00Z"
  },
  "syncs": [
    {
      "channelUid": "550e8400-e29b-41d4-a716-446655440000",
      "wabaId": "1234567890",
      "status": "APPROVED",
      "metaTemplateId": "99999999",
      "metaTemplateName": "plati_order_confirmation",
      "attempts": 1,
      "approvedAt": "2024-01-15T10:05:00Z"
    }
  ]
}
```

### Sync Status Values

| Status | Description |
|--------|-------------|
| `PENDING_CREATE` | Queued for Meta creation |
| `PENDING_REVIEW` | Submitted, awaiting Meta review |
| `APPROVED` | Ready to send on this channel |
| `REJECTED` | Rejected by Meta (see `rejectionReason`) |
| `FAILED_TO_CREATE` | Transient error during creation (see `lastError`) |
| `PAUSED` / `DISABLED` | Template paused or disabled in Meta |
| `DELETING` / `DELETED` | Removal in progress or complete |

### Master Status Values

| Status | Description |
|--------|-------------|
| `DRAFT` | No active WhatsApp channels to sync |
| `SYNCING` | At least one channel sync in progress |
| `ACTIVE` | All channels approved |
| `PARTIAL` | Some channels approved, others pending or failed |
| `FAILED` | All syncs failed or rejected |
| `DELETING` | Delete in progress |

---

## 4. Get Template Variables

Get the runtime contract for sending: required variables, header media, and sync status. **Call this before `/send`** to know which variables to pass.

### Endpoint

```
GET /v1/workspace/templates/{uid}/variables
```

### Path Parameters

- `uid` (required): Template UID

### Example Request

```bash
curl -X GET "https://api.plati.ai/v1/workspace/templates/8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789/variables" \
  -H "x-api-key: your-api-key-here"
```

### Response (200 OK)

```json
{
  "templateUid": "8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789",
  "name": "order_confirmation",
  "metaTemplateName": "plati_order_confirmation",
  "language": "pt_BR",
  "category": "UTILITY",
  "variables": [
    {
      "name": "customer_name",
      "location": "body",
      "componentIndex": 0,
      "exampleValue": "Ana",
      "required": true
    },
    {
      "name": "order_id",
      "location": "body",
      "componentIndex": 0,
      "exampleValue": "ABC123",
      "required": true
    }
  ],
  "headerMedia": null,
  "canDowngrade24h": true,
  "syncStatus": {
    "statusMaster": "ACTIVE",
    "perChannel": [
      {
        "channelUid": "550e8400-e29b-41d4-a716-446655440000",
        "status": "APPROVED"
      }
    ]
  }
}
```

---

## 5. Send Template

Dispatch an approved template to a contact or an existing conversation (including WhatsApp groups).

### Endpoint

```
POST /v1/workspace/templates/{uid}/send
```

### Path Parameters

- `uid` (required): Template UID

### Request Body

```json
{
  "channelUid": "550e8400-e29b-41d4-a716-446655440000",
  "contactUid": "550e8400-e29b-41d4-a716-446655440001",
  "variables": {
    "customer_name": "Ana",
    "order_id": "ABC123"
  }
}
```

### Request Body Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channelUid` | string (UUID) | Yes | Channel to send through |
| `contactUid` | string (UUID) | Conditional | Contact UID for 1:1 sends. Required unless `conversationUid` targets a group. |
| `conversationUid` | string (UUID) | Conditional | Existing conversation UID. For WhatsApp groups, sends with `recipient_type=group`. |
| `variables` | object | Yes | Map of variable name → value. Must match the template variables schema. |
| `headerMedia` | object | No | Override header media URL at send time (must be publicly accessible by Meta) |

At least one of `contactUid` or `conversationUid` is required.

### 24h Window Optimization

When `canDowngrade24h` is `true` and the contact's last inbound message was within Meta's 24-hour customer service window, Plati may deliver the message as **free-form** (`deliveryMode: FREE_FORM`) instead of a paid template (`deliveryMode: TEMPLATE`).

### Example Request (1:1)

```bash
curl -X POST "https://api.plati.ai/v1/workspace/templates/8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789/send" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "channelUid": "550e8400-e29b-41d4-a716-446655440000",
    "contactUid": "550e8400-e29b-41d4-a716-446655440001",
    "variables": {
      "customer_name": "Ana",
      "order_id": "ABC123"
    }
  }'
```

### Example Request (WhatsApp Group)

```bash
curl -X POST "https://api.plati.ai/v1/workspace/templates/8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789/send" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "channelUid": "550e8400-e29b-41d4-a716-446655440000",
    "conversationUid": "be4a86bb-981a-4603-a4f0-179ba1cbb336",
    "variables": {
      "customer_name": "Equipe",
      "order_id": "ABC123"
    }
  }'
```

### Response (200 OK)

```json
{
  "messageUid": "550e8400-e29b-41d4-a716-446655440002",
  "externalId": "wamid.HBgNNTUxMTk5OTk5OTk5FQIAERgSM...",
  "status": "SENT",
  "deliveryMode": "TEMPLATE"
}
```

### Error Responses

| Status Code | Code | Description |
|-------------|------|-------------|
| 400 | `MISSING_REQUIRED_VARIABLES` | One or more required variables not provided |
| 400 | `UNKNOWN_VARIABLES` | Variables sent that are not declared in the template |
| 400 | `TEMPLATE_NOT_APPROVED_FOR_CHANNEL` | Template not yet approved on the target channel |
| 400 | `CHANNEL_NOT_ACTIVE` | Channel is inactive |
| 404 | `TEMPLATE_NOT_FOUND` | Template UID not found in workspace |
| 404 | `CONTACT_NOT_FOUND` | Contact UID not found |
| 404 | `CHANNEL_NOT_FOUND` | Channel UID not found |

---

## 6. Delete Template

Delete a template from the workspace and remove it from every WABA where it was synced. Deletion is asynchronous.

### Endpoint

```
DELETE /v1/workspace/templates/{uid}
```

### Path Parameters

- `uid` (required): Template UID

### Example Request

```bash
curl -X DELETE "https://api.plati.ai/v1/workspace/templates/8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789" \
  -H "x-api-key: your-api-key-here"
```

### Response (202 Accepted)

```json
{
  "status": "accepted"
}
```

**Important:** Meta enforces a **30-day cooldown** before reusing a deleted template name.

---

## 7. Retry Failed Syncs

Re-enqueue sync jobs for channels where creation failed (`FAILED_TO_CREATE`) or was rejected (`REJECTED`). Useful after fixing a content policy issue or recovering from a transient error.

### Endpoint

```
POST /v1/workspace/templates/{uid}/retry
```

### Path Parameters

- `uid` (required): Template UID

### Example Request

```bash
curl -X POST "https://api.plati.ai/v1/workspace/templates/8b1f2c3d-4e5f-6789-a0b1-c2d3e4f56789/retry" \
  -H "x-api-key: your-api-key-here"
```

### Response (202 Accepted)

```json
{
  "retried": 2
}
```

---

## Typical Workflow

```bash
# 1. Create template
curl -X POST "https://api.plati.ai/v1/workspace/templates" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{ "name": "order_confirmation", "language": "pt_BR", ... }'

# 2. Poll until approved
curl -X GET "https://api.plati.ai/v1/workspace/templates/{uid}" \
  -H "x-api-key: your-api-key-here"

# 3. Get send contract
curl -X GET "https://api.plati.ai/v1/workspace/templates/{uid}/variables" \
  -H "x-api-key: your-api-key-here"

# 4. Send
curl -X POST "https://api.plati.ai/v1/workspace/templates/{uid}/send" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{ "channelUid": "...", "contactUid": "...", "variables": { ... } }'
```

---

## Best Practices

1. **Named variables only**: Always use `{{variable_name}}` — positional `{{1}}` placeholders are rejected at creation time.
2. **Examples required**: Every variable must have an entry in `variableExamples` when creating the template.
3. **Poll before sending**: Check `GET /{uid}` or `GET /{uid}/variables` until sync status is `APPROVED` on the target channel.
4. **Prefix is automatic**: Pass `order_confirmation` as `name` — Plati registers it as `plati_order_confirmation` in Meta.
5. **Public media URLs**: Header media URLs must be reachable by Meta's servers (no auth-gated URLs).
6. **24h optimization**: Templates with `canDowngrade24h: true` may be sent as free-form within the customer service window, reducing template fees.
7. **Retry on failure**: Use `POST /{uid}/retry` after fixing rejection reasons instead of creating a duplicate template.
8. **Name reuse cooldown**: After deletion, wait 30 days before reusing the same template name in Meta.

## Notes

- Templates sync automatically to all **active** WhatsApp Business channels in the workspace.
- If no active WhatsApp channels exist at creation time, the template stays in `DRAFT` until one is added.
- Only `UTILITY` and `MARKETING` categories are supported.
- The `plati_` prefix is added automatically — do not include it in the `name` field.
- For listing templates already registered directly in Meta (without Plati workspace definitions), use `GET /v1/channels/{channelUid}/templates`.

## Support

For more information, refer to the full API documentation or contact support.
