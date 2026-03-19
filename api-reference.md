# ScheduCal API Reference

## Overview

ScheduCal provides a REST API for calendar scheduling operations. Upon registration, developers receive an API key and secret along with the REST API endpoint.

ScheduCal is a **fully managed calendar invitation service** built on Microsoft Exchange and the Microsoft Graph API. Your integration is three API calls (Create, Update, Cancel) and one webhook — ScheduCal handles all Exchange/Graph complexity, ICS compliance semantics, and cross-client delivery.

See [Architecture Overview](architecture.md) for a full explanation of the delivery layer, ICS compliance semantics, and Gmail priming infrastructure.

**Base URL:** `https://api.scheducal.com`

## Authentication

All API requests require authentication using your API key and secret in the request body:

```json
{
  "apiKey": "your-api-key",
  "apiSecret": "your-api-secret"
}
```

## How Delivery Works

When you call Create, Update, or Cancel:

1. ScheduCal creates, updates, or cancels the event directly in Microsoft Exchange via the Graph API
2. Exchange delivers a standards-compliant calendar invitation with correct RFC 5545 ICS metadata (persistent UID, incremented SEQUENCE, proper cancellation METHOD)
3. Attendee responses (accept/decline/tentative) flow back through Exchange and are delivered to your webhook

You do not need an Azure AD application, OAuth setup, or any Exchange infrastructure. ScheduCal manages all of this.

## Date/Time Format Requirements

- **Dates**: Must use ISO 8601 format (e.g., `"2024-01-09T13:00:00"`)
- **Time Zones**: Only Microsoft Graph-supported time zones are accepted. See [Microsoft's supported time zones](https://docs.microsoft.com/en-us/graph/api/resources/datetimetimezone).

## Optional Fields

Contact information (name and email address) is optional during appointment creation. This enables use cases like webinars where attendees are added later.

---

## Endpoints

### Create Appointment

Creates a new calendar appointment.

**Endpoint**: `POST /api/v1/appointments`

**Request Body**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
| `appointmentSubject` | string | No | Subject/title of the appointment |
| `appointmentBody` | string | No | HTML body content |
| `appointmentStart` | string | Yes | Start time in ISO 8601 format |
| `appointmentEnd` | string | Yes | End time in ISO 8601 format |
| `appointmentTimeZone` | string | Yes | Time zone (e.g., "America/Los_Angeles") |
| `appointmentLocation` | string | No | Location of the appointment |
| `name` | string | No | Initial invitee's name |
| `address` | string | No | Initial invitee's email address |

**Example Request**:

```json
{
  "apiKey": "your-api-key",
  "apiSecret": "your-api-secret",
  "appointmentSubject": "Team Meeting",
  "appointmentBody": "<p>Weekly sync meeting</p>",
  "appointmentStart": "2024-01-15T10:00:00",
  "appointmentEnd": "2024-01-15T11:00:00",
  "appointmentTimeZone": "America/Los_Angeles",
  "appointmentLocation": "Conference Room A",
  "name": "John Doe",
  "address": "john.doe@example.com"
}
```

**Example Response**:

```json
{
  "success": true,
  "message": "Appointment created successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "subject": "Team Meeting",
    "start": "2024-01-15T10:00:00",
    "end": "2024-01-15T11:00:00",
    "timeZone": "America/Los_Angeles",
    "location": "Conference Room A",
    "inviteeCount": 1,
    "hasInitialInvitee": true,
    "gmailFirstInvite": true,
    "dateCreated": "2024-01-10T15:30:00Z"
  },
  "apiVersion": "v1"
}
```

---

### Update Appointment

Updates an existing appointment. All invitees are notified of changes.

**Endpoint**: `PUT /api/v1/appointments/{appointmentId}`

**Request Body**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
| `appointmentSubject` | string | No | Updated subject |
| `appointmentBody` | string | No | Updated body content |
| `appointmentStart` | string | No | Updated start time |
| `appointmentEnd` | string | No | Updated end time |
| `appointmentTimeZone` | string | No | Updated time zone |
| `appointmentLocation` | string | No | Updated location |

**Example Request**:

```json
{
  "apiKey": "your-api-key",
  "apiSecret": "your-api-secret",
  "appointmentSubject": "Team Meeting (Rescheduled)",
  "appointmentStart": "2024-01-16T10:00:00",
  "appointmentEnd": "2024-01-16T11:00:00",
  "appointmentTimeZone": "America/Los_Angeles"
}
```

**Example Response**:

```json
{
  "success": true,
  "message": "Appointment updated successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "subject": "Team Meeting (Rescheduled)",
    "dateUpdated": "2024-01-11T09:15:00Z"
  },
  "apiVersion": "v1"
}
```

---

### Cancel Appointment

Cancels an appointment. All participants are notified.

**Endpoint**: `DELETE /api/v1/appointments/{appointmentId}`

**Request Body**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
| `comment` | string | No | Optional cancellation message |

**Example Request**:

```json
{
  "apiKey": "your-api-key",
  "apiSecret": "your-api-secret",
  "comment": "Meeting cancelled due to scheduling conflict"
}
```

**Example Response**:

```json
{
  "success": true,
  "message": "Appointment cancelled successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "subject": "Team Meeting",
    "dateCanceled": "2024-01-11T14:00:00Z",
    "isActive": false,
    "inviteeCount": 0,
    "originalInviteeCount": 3
  },
  "apiVersion": "v1"
}
```

---

### Send Invitation

Adds an attendee to an existing appointment.

**Endpoint**: `POST /api/v1/appointments/{appointmentId}/invitations`

**Request Body**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | Your API key |
| `apiSecret` | string | Yes | Your API secret |
| `name` | string | Yes | Invitee's name |
| `address` | string | Yes | Invitee's email address |

**Example Request**:

```json
{
  "apiKey": "your-api-key",
  "apiSecret": "your-api-secret",
  "name": "Jane Smith",
  "address": "jane.smith@example.com"
}
```

**Example Response**:

```json
{
  "success": true,
  "message": "Invitation sent successfully",
  "data": {
    "appointmentId": "AAMkAGI2...",
    "invitee": "Jane Smith",
    "email": "jane.smith@example.com",
    "totalInvitees": 2,
    "appointmentSubject": "Team Meeting",
    "gmailFirstInvite": false
  },
  "apiVersion": "v1"
}
```

---

## Response Fields

### Gmail First Invite Indicator

ScheduCal includes required Gmail priming infrastructure for invitations to Gmail addresses. When a Gmail address is invited for the first time from a given account, ScheduCal executes a priming step before delivering the calendar invitation. This step is required to ensure the invitation lands in the attendee's Google Calendar rather than being suppressed by Gmail's spam filters.

| Field | Type | Description |
|-------|------|-------------|
| `gmailFirstInvite` | boolean | `true` if Gmail priming was executed for this invitation. This is required infrastructure, not an enhancement. |

**When `gmailFirstInvite: true`:**
- The invitee is using a Gmail-hosted email address
- This is their first invitation from your account
- ScheduCal executed a priming step to ensure correct Gmail calendar delivery
- **Required**: Display the following message to your user: "Please check your spam folder and confirm the calendar invitation"

**When `gmailFirstInvite: false`:**
- The invitee is not using Gmail, or they have already received a prior invitation from your account
- No priming step was needed; standard Exchange delivery was used

This field appears in responses for:
- Create Appointment (when `name` and `address` are provided)
- Send Invitation

See [Architecture: Gmail Priming](architecture.md#gmail-priming-required-infrastructure) for a full explanation.

---

## ICS Calendar Compliance

ScheduCal enforces correct RFC 5545 (iCalendar) semantics automatically. You do not need to track or pass any of these values — ScheduCal manages them transparently.

### UID Persistence

Every appointment is assigned a persistent UID at creation time. This UID is preserved across all updates and the final cancellation. Per RFC 5545, the UID ties all lifecycle events (create → update → cancel) to the same calendar entry. Without UID persistence, attendee calendar clients create duplicate appointments on every update.

### SEQUENCE Incrementing

The `SEQUENCE` counter is incremented on every update per RFC 5545 §3.8.7.4. This signals to attendee calendar clients that a newer version of the event has arrived, so they replace the existing entry rather than create a duplicate.

### Cancellation METHOD

Cancellations are issued with `METHOD:CANCEL` per iTIP (RFC 5546). This is the correct mechanism for removing an event from attendee calendars. A plain cancellation email without the correct iTIP METHOD header is the most common calendar implementation error — it leaves the event as a phantom entry on the attendee's calendar. ScheduCal handles this correctly via Exchange.

---

## Error Responses

All error responses follow this format:

```json
{
  "success": false,
  "error": "Error message description",
  "apiVersion": "v1"
}
```

### Common Error Codes

| HTTP Code | Description |
|-----------|-------------|
| 400 | Bad Request - Invalid parameters |
| 401 | Unauthorized - Invalid API key or secret |
| 404 | Not Found - Appointment not found |
| 409 | Conflict - Resource conflict (e.g., already cancelled) |
| 500 | Internal Server Error |
| 503 | Service Unavailable - Temporary outage |

---

## Support

For API support, contact support@scheducal.com.
