# Plati API - Send Messages Guide

This guide explains how to send text messages and WhatsApp templates using the Plati API with API Key authentication.

## Authentication

All requests require an API Key passed in the `x-api-key` header:

```bash
x-api-key: your-api-key-here
```

**Note:** When authenticated with API Key, messages are sent as system messages.

## Base URL

```
https://beta.plati.ai
```

---

## 1. List Channels

Before sending messages, you need to know which channels are available in your workspace.

### Endpoint

```
GET /v1/channels
```

### Query Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `limit` | number | Number of channels to return (1-50) | 20 |
| `cursor` | string | Cursor for pagination (channel ID) | - |
| `direction` | string | Direction for pagination: `before` or `after` | - |

### Example Request

```bash
curl -X GET "https://beta.plati.ai/v1/channels?limit=20" \
  -H "x-api-key: your-api-key-here"
```

### Response (200 OK)

```json
{
  "data": [
    {
      "uid": "550e8400-e29b-41d4-a716-446655440000",
      "name": "WhatsApp Business",
      "type": "whatsapp_business",
      "status": "active",
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ],
  "pagination": {
    "cursor": "12345",
    "hasMore": false
  }
}
```

---

## 2. List Contacts (by Channel)

List all contacts for a specific channel.

### Endpoint

```
GET /v1/channels/{channelUid}/contacts
```

### Path Parameters

- `channelUid` (required): The channel UID

### Query Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `limit` | number | Number of contacts to return (1-100) | 20 |
| `cursor` | string | Cursor for pagination (contact ID) | - |
| `direction` | string | Direction for pagination: `before` or `after` | - |
| `deletedOnly` | boolean | Filter to show only deleted contacts | false |
| `search` | string | Search term | - |
| `segment` | string | Filter by segment | - |
| `marketingOptIn` | boolean | Filter by marketing opt-in status | - |
| `isVerified` | boolean | Filter by verified status | - |
| `isActive` | boolean | Filter by active status | - |
| `status` | string | Filter by status | - |
| `identifierType` | string | Filter by identifier type | - |

### Example Request

```bash
curl -X GET "https://beta.plati.ai/v1/channels/550e8400-e29b-41d4-a716-446655440000/contacts?limit=20" \
  -H "x-api-key: your-api-key-here"
```

### Response (200 OK)

```json
{
  "data": [
    {
      "uid": "550e8400-e29b-41d4-a716-446655440001",
      "name": "John Doe",
      "identifier": "+5511999999999",
      "identifierType": "phone",
      "isVerified": true,
      "isActive": true,
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ],
  "pagination": {
    "cursor": "12345",
    "hasMore": true
  }
}
```

---

## 3. List Conversations

List all conversations in your workspace. You can filter by channel or contact.

### Endpoint

```
GET /v1/conversations
```

### Query Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `channelUid` | string | Channel UID to filter conversations | - |
| `contactUid` | string | Contact UID to filter conversations | - |
| `limit` | number | Number of conversations to return (1-100) | 20 |
| `cursor` | string | Cursor for pagination (conversation ID) | - |
| `direction` | string | Direction for pagination: `before` or `after` | - |

### Example Request

```bash
# List all conversations
curl -X GET "https://beta.plati.ai/v1/conversations?limit=20" \
  -H "x-api-key: your-api-key-here"

# Filter by channel
curl -X GET "https://beta.plati.ai/v1/conversations?channelUid=550e8400-e29b-41d4-a716-446655440000&limit=20" \
  -H "x-api-key: your-api-key-here"

# Filter by contact
curl -X GET "https://beta.plati.ai/v1/conversations?contactUid=550e8400-e29b-41d4-a716-446655440001&limit=20" \
  -H "x-api-key: your-api-key-here"
```

### Response (200 OK)

```json
{
  "data": [
    {
      "uid": "550e8400-e29b-41d4-a716-446655440002",
      "channelUid": "550e8400-e29b-41d4-a716-446655440000",
      "status": "active",
      "createdAt": "2024-01-15T10:00:00Z",
      "lastMessageAt": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "cursor": "12345",
    "hasMore": true
  }
}
```

---

## 4. Send Text Message

Send a text message to an existing conversation.

### Endpoint

```
POST /v1/conversations/{uid}/messages
```

### Path Parameters

- `uid` (required): The conversation UID

### Request Body

```json
{
  "messageType": "text",
  "shouldPublishInExternalChannel": true,
  "contents": [
    {
      "type": "text",
      "data": {
        "text": "Hello, how can I help you?"
      },
      "position": 0
    }
  ]
}
```

### Request Body Fields

| Field | Type | Required | Description | Default |
|-------|------|----------|-------------|---------|
| `messageType` | string | No | Type of message: `text`, `reaction`, `action`, `system`, `media`, `interactive`, `template` | `text` |
| `shouldPublishInExternalChannel` | boolean | No | If false, message only stored in database, not sent to external channel | `true` |
| `contents` | array | Yes | Array of message contents (min: 1, max: 10) | - |
| `parentMessageUid` | string | No | UUID of parent message if this is a reply | - |
| `externalId` | string | No | External ID from channel provider | - |
| `metadata` | object | No | Additional metadata for the message | - |

### Content Object Structure

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Content type: `text`, `image`, `video`, `audio`, `document`, `footer`, `header`, `contact`, `location`, `button`, `list`, `template`, `sticker`, `reaction`, `request_location` |
| `data` | object | Yes | Content data (structure varies by type) |
| `position` | number | No | Position of this content in the message (min: 0) |
| `metadata` | object | No | Additional metadata for this content |

### Example Request

```bash
curl -X POST "https://beta.plati.ai/v1/conversations/550e8400-e29b-41d4-a716-446655440000/messages" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "messageType": "text",
    "contents": [
      {
        "type": "text",
        "data": {
          "text": "Hello! This is a test message from the API."
        },
        "position": 0
      }
    ]
  }'
```

### Response (201 Created)

```json
{
  "message": {
    "uid": "550e8400-e29b-41d4-a716-446655440001",
    "messageType": "text",
    "status": "SENT",
    "createdAt": "2024-01-15T10:30:00Z",
    "sentAt": "2024-01-15T10:30:01Z",
    "contents": [
      {
        "type": "text",
        "data": {
          "text": "Hello! This is a test message from the API."
        },
        "position": 0
      }
    ]
  },
  "meta": {
    "queuedForDelivery": true,
    "estimatedDeliveryTime": "2024-01-15T10:30:05Z"
  }
}
```

### Error Responses

| Status Code | Description |
|-------------|-------------|
| 400 | Invalid message data |
| 403 | User cannot send messages to this conversation |
| 404 | Conversation not found |

---

## 5. Send WhatsApp Template

Send a WhatsApp template message to an existing conversation.

### Step 1: Get Available Templates

First, list available templates for your channel:

```
GET /v1/channels/{channelUid}/templates
```

#### Query Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `limit` | number | Maximum number of templates to return (1-100) | 20 |
| `offset` | number | Offset for pagination | 0 |

#### Example Request

```bash
curl -X GET "https://beta.plati.ai/v1/channels/your-channel-uid/templates?limit=20&offset=0" \
  -H "x-api-key: your-api-key-here"
```

#### Example Response

```json
{
  "templates": [
    {
      "name": "welcome_message",
      "language": "en_US",
      "status": "APPROVED",
      "category": "MARKETING",
      "components": [
        {
          "type": "BODY",
          "text": "Welcome {{1}}! Thank you for contacting us."
        }
      ]
    }
  ],
  "total": 5,
  "limit": 20,
  "offset": 0
}
```

### Step 2: Send Template Message

Use the same endpoint as text messages, but set `messageType` to `template`.

### Endpoint

```
POST /v1/conversations/{uid}/messages
```

### Request Body

```json
{
  "messageType": "template",
  "shouldPublishInExternalChannel": true,
  "contents": [
    {
      "type": "template",
      "data": {
        "name": "welcome_message",
        "language": "en_US",
        "components": [
          {
            "type": "body",
            "parameters": [
              {
                "type": "text",
                "text": "John"
              }
            ]
          }
        ]
      },
      "position": 0
    }
  ]
}
```

### Template Data Structure

The `data` object for template content typically includes:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Template name (from your approved templates) |
| `language` | string | Template language code (e.g., `en_US`, `pt_BR`) |
| `components` | array | Array of template components with parameters |

### Example Request

```bash
curl -X POST "https://beta.plati.ai/v1/conversations/550e8400-e29b-41d4-a716-446655440000/messages" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "messageType": "template",
    "contents": [
      {
        "type": "template",
        "data": {
          "name": "welcome_message",
          "language": "en_US",
          "components": [
            {
              "type": "body",
              "parameters": [
                {
                  "type": "text",
                  "text": "John"
                }
              ]
            }
          ]
        },
        "position": 0
      }
    ]
  }'
```

### Response (201 Created)

```json
{
  "message": {
    "uid": "550e8400-e29b-41d4-a716-446655440002",
    "messageType": "template",
    "status": "SENT",
    "createdAt": "2024-01-15T10:30:00Z",
    "sentAt": "2024-01-15T10:30:01Z",
    "contents": [
      {
        "type": "template",
        "data": {
          "name": "welcome_message",
          "language": "en_US",
          "components": [
            {
              "type": "body",
              "parameters": [
                {
                  "type": "text",
                  "text": "John"
                }
              ]
            }
          ]
        },
        "position": 0
      }
    ]
  },
  "meta": {
    "queuedForDelivery": true,
    "estimatedDeliveryTime": "2024-01-15T10:30:05Z"
  }
}
```

---

## Best Practices

1. **API Key Security**: Never expose your API key in client-side code or public repositories
2. **Error Handling**: Always implement proper error handling for API responses
3. **Rate Limiting**: Be mindful of rate limits when sending messages in bulk
4. **Template Validation**: Always verify template names and parameters before sending
5. **Conversation UID**: Ensure the conversation exists before attempting to send messages
6. **Content Array**: Remember that `contents` is an array and can contain multiple content items (max: 10)
7. **Pagination**: Use cursor-based pagination for listing resources efficiently

## Notes

- When using API Key authentication, messages are sent as system messages
- Templates must be pre-approved by WhatsApp before they can be used
- Only WhatsApp Business channels support template messages
- The `shouldPublishInExternalChannel` flag allows you to store messages without sending them externally
- All list endpoints support cursor-based pagination for efficient data retrieval

## Support

For more information, refer to the full API documentation or contact support.
