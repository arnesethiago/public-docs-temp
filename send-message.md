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

## Table of Contents

1. [List Channels](#1-list-channels)
2. [List Contacts](#2-list-contacts-by-channel)
3. [List Conversations](#3-list-conversations)
4. [Send Text Message](#4-send-text-message)
5. [Send WhatsApp Template](#5-send-whatsapp-template)
6. [Send Message by Phone Number](#6-send-message-by-phone-number) ⭐ NEW
7. [Verify Phone on WhatsApp](#7-verify-phone-on-whatsapp) ⭐ NEW

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
      "identifier": "5511999999999",
      "identifierType": "PHONE",
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
    "status": "queued",
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
    "authType": "apikey"
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
    "status": "queued",
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
    "authType": "apikey"
  }
}
```

---

## 6. Send Message by Phone Number ⭐ NEW

Send a message directly to a contact using their phone number, without needing to know the conversation UID. The system will automatically find or create a conversation with the contact.

### Endpoint

```
POST /v1/channels/{channelUid}/contacts/{phoneNumber}/messages
```

### Path Parameters

- `channelUid` (required): The channel UID
- `phoneNumber` (required): Phone number in format: {countryCode}{areaCode}{number} (e.g., `5511999999999`)

### Request Body

Same as [Send Text Message](#4-send-text-message).

### Phone Number Format

The phone number will be automatically normalized before lookup:

| Input Format | Normalized | Valid |
|--------------|-----------|-------|
| `5511999999999` | `5511999999999` | ✅ |
| `+5511999999999` | `5511999999999` | ✅ |
| `55 11 99999-9999` | `5511999999999` | ✅ |

**Important Notes:**
- Contact must already exist in the system with this phone number
- Phone number must be in format: {countryCode}{areaCode}{number} (no +, spaces, or hyphens)
- Conversation is automatically created or reused if it exists
- If conversation was inactive for less than 24 hours, it will be reopened automatically

### Example Request

```bash
curl -X POST "https://beta.plati.ai/v1/channels/550e8400-e29b-41d4-a716-446655440000/contacts/5511999999999/messages" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "messageType": "text",
    "contents": [
      {
        "type": "text",
        "data": {
          "text": "Hello! This message was sent using phone number."
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
    "status": "queued",
    "createdAt": "2024-01-15T10:30:00Z",
    "contents": [
      {
        "type": "text",
        "data": {
          "text": "Hello! This message was sent using phone number."
        },
        "position": 0
      }
    ]
  },
  "conversationUid": "550e8400-e29b-41d4-a716-446655440002",
  "contactUid": "550e8400-e29b-41d4-a716-446655440003",
  "meta": {
    "authType": "apikey"
  }
}
```

### Error Responses

| Status Code | Description |
|-------------|-------------|
| 400 | Invalid phone number or contact inactive |
| 404 | Channel or contact not found with this phone number |
| 403 | Insufficient permissions |

### Rate Limits

- **Users**: 100 requests per minute
- **API Keys**: 10,000 requests per hour

---

## 7. Verify Phone on WhatsApp ⭐ NEW

Check if a phone number is registered on WhatsApp before sending messages. Useful to validate numbers and avoid sending messages to invalid numbers.

### Endpoint

```
GET /v1/whatsapp/verify/{phoneNumber}
```

### Path Parameters

- `phoneNumber` (required): Phone number to verify (accepts various formats)

### Phone Number Formats Accepted

The system will normalize the phone number automatically:

| Input | Normalized | Valid |
|-------|-----------|-------|
| `5511999999999` | `5511999999999` | ✅ |
| `+5511999999999` | `5511999999999` | ✅ |
| `55 11 99999-9999` | `5511999999999` | ✅ |
| `+55 11 9 9999-9999` | `5511999999999` | ✅ |

### Example Request

```bash
curl -X GET "https://beta.plati.ai/v1/whatsapp/verify/5511999999999" \
  -H "x-api-key: your-api-key-here"
```

### Response (200 OK)

```json
{
  "exists": true,
  "phone": "5511999999999@c.us",
  "lid": "123456789012345@lid",
  "normalizedPhone": "5511999999999"
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `exists` | boolean | `true` if number has WhatsApp, `false` otherwise |
| `phone` | string | Phone number formatted according to WhatsApp response |
| `lid` | string | WhatsApp's unique identifier for the contact (null if not exists) |
| `normalizedPhone` | string | Phone number after normalization |

### Example: Number Not on WhatsApp

```json
{
  "exists": false,
  "phone": "5511000000000@c.us",
  "lid": null,
  "normalizedPhone": "5511000000000"
}
```

### Error Responses

| Status Code | Description |
|-------------|-------------|
| 400 | Invalid phone number format |
| 429 | Rate limit exceeded (try again later) |
| 503 | Phone verification service not configured or unavailable |
| 408 | Request timeout (verification took too long) |

### Rate Limits

- **Users**: 30 requests per minute
- **API Keys**: 500 requests per hour

### Use Case Example

Before sending a message, verify the number:

```bash
# Step 1: Verify phone
curl -X GET "https://beta.plati.ai/v1/whatsapp/verify/5511999999999" \
  -H "x-api-key: your-api-key-here"

# If exists: true, then send message
curl -X POST "https://beta.plati.ai/v1/channels/{channelUid}/contacts/5511999999999/messages" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "messageType": "text",
    "contents": [
      {
        "type": "text",
        "data": {
          "text": "Hello!"
        }
      }
    ]
  }'
```

---

## Best Practices

1. **API Key Security**: Never expose your API key in client-side code or public repositories
2. **Error Handling**: Always implement proper error handling for API responses
3. **Rate Limiting**: Be mindful of rate limits when sending messages in bulk
4. **Template Validation**: Always verify template names and parameters before sending
5. **Phone Verification**: Use the `/whatsapp/verify` endpoint to validate phone numbers before sending messages
6. **Phone Format**: Always send phone numbers in normalized format: {countryCode}{areaCode}{number}
7. **Conversation Management**: Use the phone number endpoint for simpler integration - it handles conversation creation automatically
8. **Content Array**: Remember that `contents` is an array and can contain multiple content items (max: 10)
9. **Pagination**: Use cursor-based pagination for listing resources efficiently

## Notes

- When using API Key authentication, messages are sent as system messages
- Templates must be pre-approved by WhatsApp before they can be used
- Only WhatsApp Business channels support template messages
- The `shouldPublishInExternalChannel` flag allows you to store messages without sending them externally
- All list endpoints support cursor-based pagination for efficient data retrieval
- Phone verification uses Z-API service and requires proper configuration
- Conversations are automatically created or reused when sending messages by phone number
- Inactive conversations (< 24h) are automatically reopened when a new message is sent

## Support

For more information, refer to the full API documentation or contact support.
```
