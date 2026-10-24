# ScheduCal API Documentation

ScheduCal delivers real calendar appointments — not email attachments. Your recipients receive native calendar invitations that appear in their calendar app, support RSVP, and behave exactly like appointments sent from any calendar system.

You call three endpoints and implement one webhook. That is the entire integration.

---

## Table of Contents

1. [Quickstart](#quickstart)
2. [Authentication](#authentication)
3. [How Invitations Work](#how-invitations-work)
4. [Create Appointment](#create-appointment)
5. [Send Invitation](#send-invitation)
6. [Update Appointment](#update-appointment)
7. [Cancel Appointment](#cancel-appointment)
8. [Webhooks](#webhooks)
9. [Error Handling](#error-handling)

---

## Quickstart

Get from zero to your first appointment in under 10 minutes.

### 1. Get your API credentials

Log in to your ScheduCal dashboard and navigate to **Settings → API Keys**. Copy your API key and API secret. Both are always accessible from the dashboard.

The examples in this guide read them from environment variables:

```bash
export API_KEY="your-api-key"
export API_SECRET="your-api-secret"
```

### 2. Create your first appointment

```bash
curl -X POST https://api.scheducal.com/api/v1/appointments \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "appointmentSubject": "Intro Call — Acme Corp",
    "appointmentStart": "2026-04-01T14:00:00",
    "appointmentEnd": "2026-04-01T14:30:00",
    "appointmentTimeZone": "America/New_York",
    "name": "John Doe",
    "address": "contact@acmecorp.com"
  }'
```

The recipient receives a calendar invitation in their inbox. They can accept, decline, or propose a new time — all from their existing calendar app.

### 3. Set up your webhook

Register a webhook endpoint to receive real-time RSVP updates:

```bash
curl -X POST https://api.scheducal.com/api/v1/webhooks \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://yourapp.com/webhooks/scheducal",
    "eventTypes": ["attendee.responded", "appointment.updated", "appointment.canceled"]
  }'
```

Done. Your integration is live.

---

## Authentication

Authenticate every request with HTTP Basic authentication: send your API key as the username and your API secret as the password in the `Authorization` header.

```
Authorization: Basic base64(apiKey:apiSecret)
```

Join the key and the secret with a single colon, Base64-encode the result, and prefix it with `Basic `. Most HTTP clients do this for you, as the examples below show. The request body carries only the fields of the operation itself.

You can manage your credentials from the dashboard under **Settings → API Keys**. Both values are always visible from the dashboard.

**Keep your credentials secret.** Do not commit them to version control or expose them in client-side code.

### curl

`-u` sends the `Authorization: Basic` header:

```bash
curl -X POST https://api.scheducal.com/api/v1/appointments \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "appointmentStart": "2026-04-15T09:00:00",
    "appointmentEnd": "2026-04-15T10:00:00",
    "appointmentTimeZone": "America/Los_Angeles"
  }'
```

### Node.js (fetch)

```javascript
const auth =
  "Basic " +
  Buffer.from(`${process.env.API_KEY}:${process.env.API_SECRET}`).toString("base64");

const response = await fetch("https://api.scheducal.com/api/v1/appointments", {
  method: "POST",
  headers: { "Content-Type": "application/json", Authorization: auth },
  body: JSON.stringify({
    appointmentStart: "2026-04-15T09:00:00",
    appointmentEnd: "2026-04-15T10:00:00",
    appointmentTimeZone: "America/Los_Angeles",
  }),
});
```

### Python (requests)

Passing a `(key, secret)` tuple as `auth` sends HTTP Basic authentication:

```python
import os
import requests

response = requests.post(
    "https://api.scheducal.com/api/v1/appointments",
    auth=(os.environ["API_KEY"], os.environ["API_SECRET"]),
    json={
        "appointmentStart": "2026-04-15T09:00:00",
        "appointmentEnd": "2026-04-15T10:00:00",
        "appointmentTimeZone": "America/Los_Angeles",
    },
)
```

### C# (HttpClient)

```csharp
using System.Net.Http.Headers;
using System.Net.Http.Json;
using System.Text;

var apiKey = Environment.GetEnvironmentVariable("API_KEY");
var apiSecret = Environment.GetEnvironmentVariable("API_SECRET");

var client = new HttpClient { BaseAddress = new Uri("https://api.scheducal.com") };
client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue(
    "Basic",
    Convert.ToBase64String(Encoding.UTF8.GetBytes($"{apiKey}:{apiSecret}")));

var response = await client.PostAsJsonAsync("/api/v1/appointments", new
{
    appointmentStart = "2026-04-15T09:00:00",
    appointmentEnd = "2026-04-15T10:00:00",
    appointmentTimeZone = "America/Los_Angeles"
});
```

### Authentication errors

| Status | When |
|--------|------|
| `401 Unauthorized` | No credentials were sent, the `Authorization` header is malformed (not `Basic`, not valid Base64, or missing the `key:secret` colon), or the key and secret do not match an account |
| `403 Forbidden` | The key and secret are valid, but the account is suspended and API access is disabled |

Every `401` response carries a `WWW-Authenticate: Basic realm="ScheduCal"` header. See [Error Handling](#error-handling) for the response bodies.

When an `Authorization` header is present, it is the only credential ScheduCal reads: a wrong or malformed header returns `401` even if the request body also contains credentials.

### Deprecated: credentials in the request body

Earlier versions of this API took `apiKey` and `apiSecret` as fields in the JSON request body. Requests without an `Authorization` header that still send them are accepted for now, but this is deprecated and will be removed. Move to the `Authorization` header. The webhook list and delete endpoints (`GET` and `DELETE`) have no request body and accept only the header.

---

## How Invitations Work

ScheduCal is built on **Microsoft Exchange**, accessed via the **Microsoft Graph API**. When you create an appointment:

1. ScheduCal creates a calendar event in Exchange via the Graph API
2. Exchange delivers a standards-compliant calendar invitation to each attendee's inbox
3. Your attendee accepts, declines, or proposes a new time from any calendar app
4. ScheduCal surfaces accepts, declines and tentative responses through your webhook (proposed new times are coming soon)

Exchange handles the complete meeting lifecycle — invite, update, cancel — using correct RFC 5545 ICS semantics (UID persistence, SEQUENCE incrementing, `METHOD:CANCEL`). Your attendees receive a native calendar event, not an email with an attachment.

**Cross-calendar compatibility is built in.** Exchange interoperates natively with Google Calendar, Apple Calendar, and all CalDAV clients. For first-time invitations to Gmail addresses, ScheduCal automatically performs a priming step to ensure reliable delivery — reflected in the `gmailFirstInvite` field returned in the response.

---

## Create Appointment

```
POST /api/v1/appointments
```

Creates a new appointment. Optionally include an initial invitee — additional invitees can be added afterwards via [Send Invitation](#send-invitation).

### Request

```bash
curl -X POST https://api.scheducal.com/api/v1/appointments \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "appointmentSubject": "Quarterly Business Review",
    "appointmentBody": "<p>Q1 2026 review with the Acme team.</p>",
    "appointmentStart": "2026-04-15T09:00:00",
    "appointmentEnd": "2026-04-15T10:00:00",
    "appointmentTimeZone": "America/Los_Angeles",
    "appointmentLocation": "https://meet.yourcompany.com/qbr-acme",
    "name": "John Doe",
    "address": "john@acmecorp.com"
  }'
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `appointmentStart` | string | Yes | Start time, ISO 8601 format (e.g. `2026-04-15T09:00:00`) |
| `appointmentEnd` | string | Yes | End time, ISO 8601 format. Must be after start. |
| `appointmentTimeZone` | string | Yes | IANA time zone name (e.g. `America/New_York`) |
| `appointmentSubject` | string | No | Appointment title shown to all attendees |
| `appointmentBody` | string | No | HTML body content shown in the invitation |
| `appointmentLocation` | string | No | Physical address or meeting URL |
| `name` | string | No | Initial invitee display name |
| `address` | string | No | Initial invitee email address |

### Response

```json
{
  "success": true,
  "message": "Appointment created successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "subject": "Quarterly Business Review",
    "start": "2026-04-15T09:00:00",
    "end": "2026-04-15T10:00:00",
    "timeZone": "America/Los_Angeles",
    "location": "https://meet.yourcompany.com/qbr-acme",
    "inviteeCount": 1,
    "hasInitialInvitee": true,
    "gmailFirstInvite": false,
    "dateCreated": "2026-03-19T14:23:01Z"
  },
  "apiVersion": "v1"
}
```

`gmailFirstInvite` is `true` when this is the first invitation sent to a Gmail address from your account. ScheduCal performs a priming step automatically to ensure reliable delivery. When `true`, display guidance to the attendee: _"Please check your spam folder and confirm the calendar invitation."_

### JavaScript Example

```javascript
// auth is "Basic " + base64(apiKey:apiSecret); see Authentication.
const response = await fetch("https://api.scheducal.com/api/v1/appointments", {
  method: "POST",
  headers: { "Content-Type": "application/json", Authorization: auth },
  body: JSON.stringify({
    appointmentSubject: "Quarterly Business Review",
    appointmentStart: "2026-04-15T09:00:00",
    appointmentEnd: "2026-04-15T10:00:00",
    appointmentTimeZone: "America/Los_Angeles",
    name: "John Doe",
    address: "john@acmecorp.com",
  }),
});

const result = await response.json();
const appointmentId = result.data.appointmentId;
```

---

## Send Invitation

```
POST /api/v1/appointments/{appointmentId}/invitations
```

Sends a calendar invitation to an additional attendee for an existing appointment.

### Request

```bash
curl -X POST https://api.scheducal.com/api/v1/appointments/AAMkAGI2.../invitations \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Mary Jones",
    "address": "mary@acmecorp.com"
  }'
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Invitee display name |
| `address` | string | Yes | Invitee email address |

### Response

```json
{
  "success": true,
  "message": "Invitation sent successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "invitee": "Mary Jones",
    "email": "mary@acmecorp.com",
    "totalInvitees": 2,
    "appointmentSubject": "Quarterly Business Review",
    "gmailFirstInvite": false
  },
  "apiVersion": "v1"
}
```

---

## Update Appointment

```
PUT /api/v1/appointments/{appointmentId}
```

Updates an existing appointment and notifies all invitees. ScheduCal handles all sequencing internally — attendees receive a proper update notification that replaces the original event in their calendar. Include only the fields you want to change.

### Request

```bash
curl -X PUT https://api.scheducal.com/api/v1/appointments/AAMkAGI2... \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "appointmentStart": "2026-04-15T10:00:00",
    "appointmentEnd": "2026-04-15T11:00:00",
    "appointmentTimeZone": "America/Los_Angeles",
    "appointmentBody": "<p>Rescheduled by 1 hour. Same link.</p>"
  }'
```

### Request Body

All fields are optional. Include only the fields you want to change.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `appointmentSubject` | string | No | Updated appointment title |
| `appointmentBody` | string | No | Updated HTML body content |
| `appointmentStart` | string | No | Updated start time (ISO 8601) |
| `appointmentEnd` | string | No | Updated end time (ISO 8601) |
| `appointmentTimeZone` | string | No | Updated IANA time zone |
| `appointmentLocation` | string | No | Updated location or meeting URL |

### Response

```json
{
  "success": true,
  "message": "Appointment updated successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "subject": "Quarterly Business Review",
    "startDateTime": "2026-04-15T10:00:00",
    "endDateTime": "2026-04-15T11:00:00",
    "timeZone": "America/Los_Angeles",
    "lastUpdated": "2026-03-19T15:44:22Z"
  },
  "apiVersion": "v1"
}
```

### JavaScript Example

```javascript
// auth is "Basic " + base64(apiKey:apiSecret); see Authentication.
const response = await fetch(
  `https://api.scheducal.com/api/v1/appointments/${appointmentId}`,
  {
    method: "PUT",
    headers: { "Content-Type": "application/json", Authorization: auth },
    body: JSON.stringify({
      appointmentStart: "2026-04-15T10:00:00",
      appointmentEnd: "2026-04-15T11:00:00",
      appointmentTimeZone: "America/Los_Angeles",
    }),
  }
);

const result = await response.json();
```

---

## Cancel Appointment

```
DELETE /api/v1/appointments/{appointmentId}
```

Cancels the appointment and notifies all invitees. The event is removed from attendees' calendars. This action is irreversible — to reschedule, create a new appointment.

Cancelling sends `appointment.canceled` to subscribed webhooks.

If the cancellation can't be made in the calendar system, the call returns an error and the appointment stays active. It is safe to retry. The call never returns `200` when nothing was cancelled.

### Request

```bash
curl -X DELETE https://api.scheducal.com/api/v1/appointments/AAMkAGI2... \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "comment": "Meeting cancelled — will reschedule next week."
  }'
```

### Request Body

The body is optional; a `DELETE` with no body is valid.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `comment` | string | No | Cancellation message shown to attendees in the cancellation notice |

### Response

```json
{
  "success": true,
  "message": "Appointment cancelled successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "subject": "Quarterly Business Review",
    "dateCanceled": "2026-03-19T16:02:11Z",
    "isActive": false,
    "inviteeCount": 0,
    "originalInviteeCount": 3
  },
  "apiVersion": "v1"
}
```

### What attendees see

Attendees receive a cancellation notification from their calendar system. The appointment is automatically removed from their calendar — they do not need to take any action. If you included a `comment`, it appears in the notification body.

### JavaScript Example

```javascript
// auth is "Basic " + base64(apiKey:apiSecret); see Authentication.
await fetch(
  `https://api.scheducal.com/api/v1/appointments/${appointmentId}`,
  {
    method: "DELETE",
    headers: { "Content-Type": "application/json", Authorization: auth },
    body: JSON.stringify({
      comment: "Meeting cancelled — will reschedule next week.",
    }),
  }
);
```

---

## Webhooks

ScheduCal sends webhook events when attendees respond to invitations or appointments change. Configure one or more endpoints to receive these events in real time.

### Register a webhook

```bash
curl -X POST https://api.scheducal.com/api/v1/webhooks \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://yourapp.com/webhooks/scheducal",
    "eventTypes": ["attendee.responded", "appointment.updated", "appointment.canceled"]
  }'
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `webhookUrl` | string | Yes | URL to receive webhook events. Use HTTPS. |
| `eventTypes` | array | Yes | List of event types to subscribe to |

### Registration Response

`201 Created`

```json
{
  "success": true,
  "data": {
    "webhookId": "49856464-9c29-43b0-ad45-a8a9812d82bf",
    "webhookUrl": "https://yourapp.com/webhooks/scheducal",
    "secret": "aujHqc8fuw/dBx6quWO8d92hlHGsrsuOAXXmx2YFDc0=",
    "eventTypes": ["attendee.responded", "appointment.updated", "appointment.canceled"],
    "isActive": true,
    "dateCreated": "2026-03-19T14:23:01Z"
  }
}
```

**Save the `secret` from this response** — you will use it to verify webhook signatures.

### Available events

| Event | Fired when |
|-------|------------|
| `attendee.responded` | An attendee accepts, declines, or marks the invitation as tentative |
| `attendee.proposed_new_time` | **Coming soon: not sent yet.** An attendee proposes an alternate meeting time |
| `appointment.updated` | The appointment was changed with `PUT /v1/appointments/{id}`. It isn't triggered any other way |
| `appointment.canceled` | The appointment was cancelled with `DELETE /v1/appointments/{id}`. Carries `appointmentId`, `subject`, `start`, `end` and `timeZone` |

`appointment.updated` and `appointment.canceled` come only from API calls. If an appointment is removed any other way (for example, when an attendee's or the account's data is erased), no `appointment.canceled` is sent.

### Webhook payload

All events share a common envelope. In every payload, `start` and `end` are wall-clock times in the appointment's `timeZone`, formatted `yyyy-MM-ddTHH:mm:ss` with no `Z` or offset.

```json
{
  "id": "3f2a1b4c-5d6e-7f8a-9b0c-1d2e3f4a5b6c",
  "eventType": "attendee.responded",
  "timestamp": "2026-04-15T09:03:44Z",
  "apiVersion": "v1",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "subject": "Quarterly Business Review",
    "start": "2026-04-15T09:00:00",
    "end": "2026-04-15T10:00:00",
    "timeZone": "America/Los_Angeles",
    "attendee": {
      "name": "John Doe",
      "email": "john@acmecorp.com",
      "response": "accepted",
      "respondedAt": "2026-04-15T09:03:44Z"
    }
  }
}
```

For `attendee.proposed_new_time` events (coming soon: not sent yet), the `attendee` object will also include:

```json
{
  "attendee": {
    "name": "John Doe",
    "email": "john@acmecorp.com",
    "response": "tentativelyAccepted",
    "respondedAt": "2026-04-15T09:03:44Z",
    "proposedStart": "2026-04-15T11:00:00",
    "proposedEnd": "2026-04-15T12:00:00"
  }
}
```

### Confirming receipt

Your webhook endpoint must return a `2xx` HTTP status within **30 seconds**. Any other response — or a timeout — is treated as a delivery failure and will be retried.

**Retry schedule:**

| Attempt | Delay after previous failure |
|---------|------------------------------|
| 1 | Immediate |
| 2 | 30 seconds |
| 3 | 1 minute |
| 4 | 2 minutes |
| 5 | 5 minutes |
| 6 | 15 minutes |

After 6 failed attempts, the delivery is marked as permanently failed. After 10 consecutive failures across deliveries, the webhook is disabled automatically; re-enable it with `PATCH /api/v1/webhooks/{webhookId}` and `"isActive": true`. See the [Webhooks Guide](webhooks.md#retry-behavior).

### Verifying webhook signatures

Every webhook request includes `X-ScheduCal-Signature` and `X-ScheduCal-Timestamp` headers. Verify them to confirm the request came from ScheduCal and was not tampered with. Use the `secret` returned when you registered the webhook.

The signature is `sha256=` followed by the Base64-encoded HMAC-SHA256 of `{timestamp}.{rawBody}`, keyed with your webhook secret, where `{timestamp}` is the value of `X-ScheduCal-Timestamp`. The [Webhooks Guide](webhooks.md#verifying-signatures) has Python and C# versions.

```javascript
const crypto = require("crypto");

function verifyWebhook(rawBody, timestamp, signatureHeader, webhookSecret) {
  const expected = crypto
    .createHmac("sha256", webhookSecret)
    .update(`${timestamp}.${rawBody}`)
    .digest("base64");

  const received = (signatureHeader || "").replace("sha256=", "");

  return (
    expected.length === received.length &&
    crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received))
  );
}

// Express example
app.post(
  "/webhooks/scheducal",
  express.raw({ type: "application/json" }),
  (req, res) => {
    const valid = verifyWebhook(
      req.body.toString(),
      req.headers["x-scheducal-timestamp"],
      req.headers["x-scheducal-signature"],
      process.env.SCHEDUCAL_WEBHOOK_SECRET
    );

    if (!valid) {
      return res.status(401).send("Invalid signature");
    }

    const event = JSON.parse(req.body);

    switch (event.eventType) {
      case "attendee.responded":
        // update your CRM or booking system
        break;
      case "appointment.canceled":
        // handle cancellation
        break;
    }

    res.sendStatus(200);
  }
);
```

### Managing webhooks

| Action | Method | Endpoint |
|--------|--------|----------|
| List webhooks | GET | `/api/v1/webhooks` |
| Update webhook | PATCH | `/api/v1/webhooks/{webhookId}` |
| Delete webhook | DELETE | `/api/v1/webhooks/{webhookId}` |
| Regenerate secret | POST | `/api/v1/webhooks/{webhookId}/regenerate-secret` |

All of them authenticate with the `Authorization` header, like every other endpoint:

```bash
# List webhooks
curl https://api.scheducal.com/api/v1/webhooks \
  -u "$API_KEY:$API_SECRET"

# Delete a webhook
curl -X DELETE https://api.scheducal.com/api/v1/webhooks/49856464-9c29-43b0-ad45-a8a9812d82bf \
  -u "$API_KEY:$API_SECRET"
```

The [Webhooks Guide](webhooks.md) documents each request and response.

---

## Error Handling

ScheduCal uses standard HTTP status codes. Error responses include a JSON body.

### Error response format

`success` is `false` and `error` holds a human-readable message:

```json
{
  "success": false,
  "error": "Invalid API key or secret",
  "apiVersion": "v1"
}
```

### HTTP status codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `201` | Created |
| `400` | Bad request — the `error` message names the problem |
| `401` | Unauthorized — missing, malformed, or invalid credentials |
| `403` | Forbidden — valid credentials, but the account is suspended |
| `404` | Not found — appointment ID does not exist |
| `409` | Conflict — e.g. the appointment is already cancelled |
| `429` | Rate limited — back off and retry |
| `500` | Server error — retry with exponential backoff |
| `503` | Service temporarily unavailable — retry with exponential backoff |

### Authentication error messages

| Status | `error` message | Fix |
|--------|-----------------|-----|
| `401` | `API credentials are required. Send Authorization: Basic base64(apiKey:apiSecret)` | Send the `Authorization` header |
| `401` | `Malformed Authorization header. Send Authorization: Basic base64(apiKey:apiSecret)` | Use the `Basic` scheme with Base64 of `apiKey:apiSecret`; send the header once |
| `401` | `Invalid API key or secret` | Check your credentials in the dashboard |
| `403` | `This account is suspended. Its credentials are valid, but API access is disabled.` | Contact support@scheducal.com |

Every `401` includes the response header `WWW-Authenticate: Basic realm="ScheduCal"`. A `403` is returned only after the key and secret have been verified, so it never reveals anything to a caller without valid credentials.

### Rate limits

| Tier | Limit |
|------|-------|
| Default | 60 requests / minute |
| High-volume (contact us) | Custom |

When rate limited, the response includes a `Retry-After` header with the number of seconds to wait.

---

## SDK Support

Official SDKs are planned for Node.js and Python. Until then, the REST API works directly with any HTTP client — the examples throughout this documentation use `curl` and Node.js `fetch`, and [Authentication](#authentication) also shows Python `requests` and C# `HttpClient`.

---

For API support, contact support@scheducal.com.
