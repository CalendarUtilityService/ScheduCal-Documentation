# ScheduCal API Documentation

ScheduCal delivers real calendar appointments — not email attachments. Your recipients receive native calendar invitations that appear in their calendar app, support RSVP, and behave exactly like appointments sent from any calendar system.

You call three endpoints and implement one webhook. That is the entire integration.

---

## Table of Contents

1. [Quickstart](#quickstart)
2. [Authentication](#authentication)
3. [Create Appointment](#create-appointment)
4. [Send Invitation](#send-invitation)
5. [Update Appointment](#update-appointment)
6. [Cancel Appointment](#cancel-appointment)
7. [Webhooks](#webhooks)
8. [Error Handling](#error-handling)

---

## Quickstart

Get from zero to your first appointment in under 10 minutes.

### 1. Get your API credentials

Log in to your ScheduCal dashboard and navigate to **Settings → API Keys**. Copy your `apiKey` and `apiSecret`. Both are always accessible from the dashboard.

### 2. Create your first appointment

```bash
curl -X POST https://api.scheducal.com/api/v1/appointments \
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "YOUR_API_KEY",
    "apiSecret": "YOUR_API_SECRET",
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
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "YOUR_API_KEY",
    "apiSecret": "YOUR_API_SECRET",
    "webhookUrl": "https://yourapp.com/webhooks/scheducal",
    "eventTypes": ["attendee.responded", "appointment.updated", "appointment.canceled"]
  }'
```

Done. Your integration is live.

---

## Authentication

All API requests include your `apiKey` and `apiSecret` in the request body.

```json
{
  "apiKey": "YOUR_API_KEY",
  "apiSecret": "YOUR_API_SECRET",
  ...
}
```

You can manage your credentials from the dashboard under **Settings → API Keys**. Both values are always visible from the dashboard.

**Keep your credentials secret.** Do not commit them to version control or expose them in client-side code.

---

## Create Appointment

```
POST /api/v1/appointments
```

Creates a new appointment. Optionally include an initial invitee — additional invitees can be added afterwards via [Send Invitation](#send-invitation).

### Request

```bash
curl -X POST https://api.scheducal.com/api/v1/appointments \
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "YOUR_API_KEY",
    "apiSecret": "YOUR_API_SECRET",
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
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
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
    "dateCreated": "2026-03-19T14:23:01Z"
  },
  "apiVersion": "v1"
}
```

### JavaScript Example

```javascript
const response = await fetch("https://api.scheducal.com/api/v1/appointments", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    apiKey: process.env.SCHEDUCAL_API_KEY,
    apiSecret: process.env.SCHEDUCAL_API_SECRET,
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
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "YOUR_API_KEY",
    "apiSecret": "YOUR_API_SECRET",
    "name": "Mary Jones",
    "address": "mary@acmecorp.com"
  }'
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
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
    "appointmentSubject": "Quarterly Business Review"
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
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "YOUR_API_KEY",
    "apiSecret": "YOUR_API_SECRET",
    "appointmentStart": "2026-04-15T10:00:00",
    "appointmentEnd": "2026-04-15T11:00:00",
    "appointmentTimeZone": "America/Los_Angeles",
    "appointmentBody": "<p>Rescheduled by 1 hour. Same link.</p>"
  }'
```

### Request Body

All fields are optional except credentials. Include only the fields you want to change.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
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
const response = await fetch(
  `https://api.scheducal.com/api/v1/appointments/${appointmentId}`,
  {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      apiKey: process.env.SCHEDUCAL_API_KEY,
      apiSecret: process.env.SCHEDUCAL_API_SECRET,
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

### Request

```bash
curl -X DELETE https://api.scheducal.com/api/v1/appointments/AAMkAGI2... \
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "YOUR_API_KEY",
    "apiSecret": "YOUR_API_SECRET",
    "comment": "Meeting cancelled — will reschedule next week."
  }'
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
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
await fetch(
  `https://api.scheducal.com/api/v1/appointments/${appointmentId}`,
  {
    method: "DELETE",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      apiKey: process.env.SCHEDUCAL_API_KEY,
      apiSecret: process.env.SCHEDUCAL_API_SECRET,
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
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "YOUR_API_KEY",
    "apiSecret": "YOUR_API_SECRET",
    "webhookUrl": "https://yourapp.com/webhooks/scheducal",
    "eventTypes": ["attendee.responded", "appointment.updated", "appointment.canceled"]
  }'
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
| `webhookUrl` | string | Yes | HTTPS URL to receive webhook events |
| `eventTypes` | array | Yes | List of event types to subscribe to |

### Registration Response

```json
{
  "success": true,
  "message": "Webhook registered successfully",
  "data": {
    "webhookId": "49856464-9c29-43b0-ad45-a8a9812d82bf",
    "secret": "whsec_abc123...",
    "webhookUrl": "https://yourapp.com/webhooks/scheducal",
    "eventTypes": ["attendee.responded", "appointment.updated", "appointment.canceled"]
  },
  "apiVersion": "v1"
}
```

**Save the `secret` from this response** — you will use it to verify webhook signatures.

### Available events

| Event | Fired when |
|-------|------------|
| `attendee.responded` | An attendee accepts, declines, or marks the invitation as tentative |
| `attendee.proposed_new_time` | An attendee proposes an alternate meeting time |
| `appointment.updated` | The appointment time, location, or subject was changed |
| `appointment.canceled` | The appointment was cancelled via API or dashboard |

### Webhook payload

All events share a common envelope:

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

For `attendee.proposed_new_time` events, the `attendee` object also includes:

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

Your webhook endpoint must return a `2xx` HTTP status within **5 seconds**. Any other response — or a timeout — is treated as a delivery failure and will be retried.

**Retry schedule:**

| Attempt | Delay after previous failure |
|---------|------------------------------|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

After 5 failed attempts, the event is marked as undelivered. You can view and replay undelivered events from the dashboard.

### Verifying webhook signatures

Every webhook request includes a `X-ScheduCal-Signature` header. Verify it to confirm the request came from ScheduCal and was not tampered with. Use the `secret` returned when you registered the webhook.

```javascript
const crypto = require("crypto");

function verifyWebhook(rawBody, signatureHeader, webhookSecret) {
  const expected = crypto
    .createHmac("sha256", webhookSecret)
    .update(rawBody)
    .digest("hex");

  const received = signatureHeader.replace("sha256=", "");

  return crypto.timingSafeEqual(
    Buffer.from(expected, "hex"),
    Buffer.from(received, "hex")
  );
}

// Express example
app.post(
  "/webhooks/scheducal",
  express.raw({ type: "application/json" }),
  (req, res) => {
    const valid = verifyWebhook(
      req.body,
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
| List webhooks | GET | `/api/v1/webhooks?apiKey=...&apiSecret=...` |
| Update webhook | PATCH | `/api/v1/webhooks/{webhookId}` |
| Delete webhook | DELETE | `/api/v1/webhooks/{webhookId}?apiKey=...&apiSecret=...` |
| Regenerate secret | POST | `/api/v1/webhooks/{webhookId}/regenerate-secret` |

---

## Error Handling

ScheduCal uses standard HTTP status codes. Error responses include a JSON body.

### Error response format

```json
{
  "success": false,
  "message": "end must be after start",
  "error": {
    "code": "invalid_time_range",
    "param": "appointmentEnd"
  }
}
```

### HTTP status codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `201` | Created |
| `400` | Bad request — check `error.code` and `error.param` |
| `401` | Unauthorized — missing or invalid credentials |
| `404` | Not found — appointment ID does not exist or was already cancelled |
| `409` | Conflict — e.g. duplicate request detected |
| `422` | Unprocessable — request was valid JSON but failed validation |
| `429` | Rate limited — back off and retry |
| `500` | Server error — retry with exponential backoff |

### Common error codes

| Code | Description | Fix |
|------|-------------|-----|
| `invalid_time_range` | `appointmentEnd` is before or equal to `appointmentStart` | Ensure end is after start |
| `missing_required_field` | A required field was not provided | Check required fields for the endpoint |
| `invalid_email` | An email address is malformed | Check address values |
| `appointment_not_found` | The appointment ID does not exist | Verify the ID; it may have been cancelled |
| `appointment_already_cancelled` | Attempting to update or cancel an already-cancelled appointment | No action needed |
| `invalid_credentials` | API key or secret is missing, malformed, or revoked | Check your credentials in the dashboard |
| `rate_limit_exceeded` | Too many requests in a short window | Back off and retry after `Retry-After` header |

### Rate limits

| Tier | Limit |
|------|-------|
| Default | 60 requests / minute |
| High-volume (contact us) | Custom |

When rate limited, the response includes a `Retry-After` header with the number of seconds to wait.

---

## SDK Support

Official SDKs are planned for Node.js and Python. Until then, the REST API works directly with any HTTP client — the examples throughout this documentation use `curl` and the browser `fetch` API.

---

For API support, contact support@scheducal.com.
