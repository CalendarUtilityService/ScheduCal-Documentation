# ScheduCal API Reference

## Overview

ScheduCal provides a REST API for calendar scheduling operations. Upon registration, developers receive an API key and secret along with the REST API endpoint.

**Base URL:** `https://api.scheducal.com`

## Authentication

All API requests require authentication using your API key and secret in the request body:

```json
{
  "apiKey": "your-api-key",
  "apiSecret": "your-api-secret"
}
```

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

When inviting a Gmail user for the first time from your account, the response includes a special field:

| Field | Type | Description |
|-------|------|-------------|
| `gmailFirstInvite` | boolean | `true` if this is the first invitation sent to a Gmail-hosted address from this account. |

**When `gmailFirstInvite: true`:**
- The invitee is using a Gmail-hosted email address
- This is their first invitation from your account
- The invitation was sent with enhanced visibility to improve Gmail calendar integration
- **Recommended**: Display messaging to the user such as "Please check your spam folder and approve adding to calendar"

**When `gmailFirstInvite: false`:**
- Either the invitee is not using Gmail, or they have already received a previous invitation from your account
- Normal invitation flow was used

This field appears in responses for:
- Create Appointment (when `name` and `address` are provided)
- Send Invitation

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
