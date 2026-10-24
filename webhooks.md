# ScheduCal Webhooks Guide

## Overview

ScheduCal Webhooks allow your application to receive real-time notifications when calendar events change. When an attendee responds to a meeting invitation (accepts, declines, or tentatively accepts), ScheduCal will send an HTTP POST request to your registered webhook URL.

Every request in this guide authenticates with HTTP Basic authentication: your API key as the username and your API secret as the password (`Authorization: Basic base64(apiKey:apiSecret)`). The curl examples use `-u "$API_KEY:$API_SECRET"`; see [Authentication](api-reference.md#authentication) for other languages and for the `401`/`403` responses. Credentials in the request body are no longer accepted.

---

## Quick Start

### 1. Register a Webhook

```bash
curl -X POST "https://api.scheducal.com/api/v1/webhooks" \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://your-app.com/webhooks/scheducal",
    "eventTypes": ["attendee.responded", "appointment.updated"]
  }'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "webhookId": "ced4cc09-eec1-409c-9146-379d8e75c9cd",
    "webhookUrl": "https://your-app.com/webhooks/scheducal",
    "secret": "aujHqc8fuw/dBx6quWO8d92hlHGsrsuOAXXmx2YFDc0=",
    "eventTypes": ["attendee.responded", "appointment.updated"],
    "isActive": true,
    "dateCreated": "2026-01-02T02:40:00Z"
  }
}
```

**Important:** Save the `secret` value. You'll need it to verify webhook signatures.

### 2. Implement Your Webhook Endpoint

Your endpoint must:
- Accept HTTP POST requests
- Return a 2xx status code within 30 seconds
- Optionally verify the signature (recommended)

### 3. Receive Events

When an attendee responds to a calendar invitation, you'll receive:

```http
POST /webhooks/scheducal HTTP/1.1
Host: your-app.com
Content-Type: application/json
X-ScheduCal-Signature: sha256=Xq0XcqOVM6M2ZpBG8Kx1d8m2b9r1yQJ0f4kq4w3wDzA=
X-ScheduCal-Timestamp: 1735812000
X-ScheduCal-Event: attendee.responded
X-ScheduCal-Delivery: 7c9e6679-7425-40de-944b-e07fc1f90ae7

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "attendee.responded",
  "timestamp": "2026-01-02T10:00:00Z",
  "apiVersion": "v1",
  "data": {
    "appointmentId": "AAMkADI3YjRk...",
    "subject": "Project Planning Meeting",
    "start": "2026-01-10T14:00:00",
    "end": "2026-01-10T15:00:00",
    "timeZone": "America/Los_Angeles",
    "attendee": {
      "email": "john.doe@example.com",
      "name": "John Doe",
      "response": "accepted",
      "respondedAt": "2026-01-02T09:45:00Z"
    }
  }
}
```

---

## Event Types

| Event Type | Description |
|------------|-------------|
| `attendee.responded` | An attendee accepted, declined, or tentatively accepted the invitation |
| `attendee.proposed_new_time` | **Coming soon: not sent yet.** An attendee proposed an alternative meeting time |
| `appointment.updated` | The appointment was changed with `PUT /v1/appointments/{id}`. It isn't triggered any other way |
| `appointment.canceled` | The appointment was cancelled with `DELETE /v1/appointments/{id}`. Carries `appointmentId`, `subject`, `start`, `end` and `timeZone` |

`appointment.updated` and `appointment.canceled` come only from API calls. If an appointment is removed any other way (for example, when an attendee's or the account's data is erased), no `appointment.canceled` is sent.

---

## API Reference

### Register Webhook

**POST** `/api/v1/webhooks`

Register a new webhook endpoint for your account.

**Request Body:**
```json
{
  "webhookUrl": "string (required) - URL to deliver events to; use HTTPS",
  "eventTypes": ["string"] (required) - Array of event types
}
```

Registering a URL that is already registered for your account returns `400`.

**Response:** `201 Created`
```json
{
  "success": true,
  "data": {
    "webhookId": "uuid",
    "webhookUrl": "string",
    "secret": "string - Save this for signature verification",
    "eventTypes": ["string"],
    "isActive": true,
    "dateCreated": "ISO 8601 timestamp"
  }
}
```

---

### List Webhooks

**GET** `/api/v1/webhooks`

List all webhooks registered for your account.

This request has no body, so it authenticates with the `Authorization` header only:

```bash
curl "https://api.scheducal.com/api/v1/webhooks" \
  -u "$API_KEY:$API_SECRET"
```

**Response:** `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "webhookId": "uuid",
      "webhookUrl": "string",
      "eventTypes": ["string"],
      "isActive": true,
      "consecutiveFailures": 0,
      "lastDeliveryAt": "ISO 8601 timestamp or null",
      "lastDeliveryStatus": "success | failed | null",
      "dateCreated": "ISO 8601 timestamp",
      "dateUpdated": "ISO 8601 timestamp or null"
    }
  ]
}
```

---

### Update Webhook

**PATCH** `/api/v1/webhooks/{webhookId}`

Update an existing webhook's configuration.

**Request Body:**
```json
{
  "webhookUrl": "string (optional)",
  "eventTypes": ["string"] (optional),
  "isActive": true | false (optional)
}
```

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "webhookId": "uuid",
    "webhookUrl": "string",
    "eventTypes": ["string"],
    "isActive": true,
    "dateUpdated": "ISO 8601 timestamp"
  }
}
```

---

### Delete Webhook

**DELETE** `/api/v1/webhooks/{webhookId}`

Delete a webhook. This request has no body, so it authenticates with the `Authorization` header only:

```bash
curl -X DELETE "https://api.scheducal.com/api/v1/webhooks/{webhookId}" \
  -u "$API_KEY:$API_SECRET"
```

**Response:** `200 OK`
```json
{
  "success": true,
  "message": "Webhook deleted successfully"
}
```

---

### Regenerate Secret

**POST** `/api/v1/webhooks/{webhookId}/regenerate-secret`

Generate a new signing secret. The old secret will immediately stop working. No request body is needed:

```bash
curl -X POST "https://api.scheducal.com/api/v1/webhooks/{webhookId}/regenerate-secret" \
  -u "$API_KEY:$API_SECRET"
```

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "webhookId": "uuid",
    "secret": "new-secret-value"
  }
}
```

---

## Webhook Payload Format

All webhook deliveries follow this structure:

```json
{
  "id": "Unique event ID (UUID); the same on every retry",
  "eventType": "Event type string",
  "timestamp": "ISO 8601 timestamp when event occurred",
  "apiVersion": "v1",
  "data": {
    "appointmentId": "ScheduCal appointment ID",
    "subject": "Meeting subject",
    "start": "Start, wall-clock in timeZone (yyyy-MM-ddTHH:mm:ss, no Z or offset)",
    "end": "End, wall-clock in timeZone (yyyy-MM-ddTHH:mm:ss, no Z or offset)",
    "timeZone": "IANA timezone",
    "attendee": {
      "email": "Attendee email",
      "name": "Attendee name (if available)",
      "response": "accepted | declined | tentativelyaccepted",
      "respondedAt": "When they responded (ISO 8601)",
      "proposedStart": "Proposed new start (coming soon: not sent yet)",
      "proposedEnd": "Proposed new end (coming soon: not sent yet)"
    }
  }
}
```

In every payload, `start` and `end` are wall-clock times in the appointment's `timeZone`, formatted `yyyy-MM-ddTHH:mm:ss` with no `Z` or offset.

`attendee` is set only on attendee events; on other events it is `null`. Until `attendee.proposed_new_time` is sent, `proposedStart` and `proposedEnd` are `null`.

---

## Webhook Headers

| Header | Description |
|--------|-------------|
| `X-ScheduCal-Signature` | HMAC-SHA256 signature for verification |
| `X-ScheduCal-Timestamp` | Unix timestamp when the webhook was sent |
| `X-ScheduCal-Event` | The event type |
| `X-ScheduCal-Delivery` | Unique ID for this delivery of the event to this endpoint; the same on every retry |
| `Content-Type` | Always `application/json` |

---

## Verifying Signatures

To ensure webhooks are genuinely from ScheduCal, verify the signature:

### Algorithm

1. Get the timestamp from `X-ScheduCal-Timestamp` header
2. Get the raw request body (as a string)
3. Concatenate: `{timestamp}.{body}`
4. Compute HMAC-SHA256 using your webhook secret
5. Base64 encode the result
6. Compare with the signature in `X-ScheduCal-Signature` (after removing `sha256=` prefix)

### Node.js Example

```javascript
const crypto = require('crypto');

function verifySignature(secret, timestamp, body, signature) {
  const signedPayload = `${timestamp}.${body}`;
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(signedPayload)
    .digest('base64');

  const receivedSignature = (signature || '').replace('sha256=', '');

  // timingSafeEqual throws when the lengths differ, so check them first.
  return expectedSignature.length === receivedSignature.length &&
    crypto.timingSafeEqual(
      Buffer.from(expectedSignature),
      Buffer.from(receivedSignature)
    );
}

// Express.js middleware
app.post('/webhooks/scheducal', express.raw({type: 'application/json'}), (req, res) => {
  const signature = req.headers['x-scheducal-signature'];
  const timestamp = req.headers['x-scheducal-timestamp'];
  const body = req.body.toString();

  if (!verifySignature(WEBHOOK_SECRET, timestamp, body, signature)) {
    return res.status(401).send('Invalid signature');
  }

  const event = JSON.parse(body);

  // Process the event...
  console.log(`Received ${event.eventType} for appointment ${event.data.appointmentId}`);

  res.status(200).send('OK');
});
```

### Python Example

```python
import hmac
import hashlib
import base64
from flask import Flask, request

app = Flask(__name__)
WEBHOOK_SECRET = 'your-webhook-secret'

def verify_signature(secret, timestamp, body, signature):
    signed_payload = f"{timestamp}.{body}"
    expected = base64.b64encode(
        hmac.new(
            secret.encode(),
            signed_payload.encode(),
            hashlib.sha256
        ).digest()
    ).decode()

    received = signature.replace('sha256=', '')
    return hmac.compare_digest(expected, received)

@app.route('/webhooks/scheducal', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-ScheduCal-Signature')
    timestamp = request.headers.get('X-ScheduCal-Timestamp')
    body = request.get_data(as_text=True)

    if not verify_signature(WEBHOOK_SECRET, timestamp, body, signature):
        return 'Invalid signature', 401

    event = request.json

    # Process the event...
    print(f"Received {event['eventType']} for {event['data']['appointmentId']}")

    return 'OK', 200
```

### C# Example

```csharp
using System.Security.Cryptography;
using System.Text;

public bool VerifySignature(string secret, string timestamp, string body, string signature)
{
    var signedPayload = $"{timestamp}.{body}";

    using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(signedPayload));
    var expectedSignature = Convert.ToBase64String(hash);

    var receivedSignature = signature.Replace("sha256=", "");

    return CryptographicOperations.FixedTimeEquals(
        Encoding.UTF8.GetBytes(expectedSignature),
        Encoding.UTF8.GetBytes(receivedSignature));
}
```

---

## Retry Behavior

If your endpoint returns a non-2xx status code or does not respond within 30 seconds, ScheduCal will retry with exponential backoff. Each delivery is attempted up to 6 times:

| Attempt | Delay After Previous Failure |
|---------|------------------------------|
| 1 | Immediate |
| 2 | 30 seconds |
| 3 | 1 minute |
| 4 | 2 minutes |
| 5 | 5 minutes |
| 6 | 15 minutes |

After 6 failed attempts, the delivery is marked as permanently failed.

### Auto-Disable

If your webhook endpoint fails **10 consecutive times** (across any deliveries), it will be automatically disabled. You can re-enable it via the Update Webhook API:

```bash
curl -X PATCH "https://api.scheducal.com/api/v1/webhooks/{webhookId}" \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "isActive": true
  }'
```

---

## Best Practices

### 1. Respond Quickly
Return a 200 response as soon as possible (within 30 seconds). Process the webhook asynchronously if needed.

### 2. Handle Duplicates
Webhooks may occasionally be delivered more than once. Use the `id` field to detect and ignore duplicates.

### 3. Verify Signatures
Always verify the signature in production to ensure webhooks are from ScheduCal.

### 4. Check Timestamp
Reject webhooks with timestamps too far in the past (e.g., > 5 minutes) to prevent replay attacks.

### 5. Use HTTPS
Register an HTTPS URL. Plain HTTP URLs are accepted, but payloads and signatures would then travel unencrypted.

### 6. Handle All Event Types
Even if you only subscribe to certain events, gracefully handle unexpected event types in case new ones are added.

---

## Error Responses

| Status Code | Description |
|-------------|-------------|
| 400 | Bad request (invalid payload, missing fields, invalid event type, URL already registered) |
| 401 | Missing, malformed, or invalid API credentials. The response includes `WWW-Authenticate: Basic realm="ScheduCal"` |
| 403 | Valid credentials, but the account is suspended |
| 404 | Webhook not found (including a webhook that belongs to another account) |
| 500 | Internal server error |
| 503 | Service temporarily unavailable |

Error response format:
```json
{
  "success": false,
  "error": "Error message describing the issue"
}
```

See [Error Handling](api-reference.md#error-handling) for the authentication error messages.
