# ScheduCal Architecture

## How ScheduCal Works

ScheduCal is a **fully managed calendar invitation service** built on Microsoft Exchange and the Microsoft Graph API. Customers integrate at the points in their code where they would send an email with an ICS file attachment — and instead call ScheduCal's simple REST API.

**Customer surface:**
- Call `POST /api/v1/appointments` to create an appointment
- Call `PUT /api/v1/appointments/{id}` to update it
- Call `DELETE /api/v1/appointments/{id}` to cancel it
- Implement one webhook endpoint to receive attendee responses

That's the entire integration. ScheduCal handles everything behind the scenes.

---

## The Delivery Layer: Microsoft Exchange via Graph API

ScheduCal's delivery layer is **Microsoft Exchange**, accessed via the **Microsoft Graph API**. Exchange is not a workaround — it is the architecture.

When you call the ScheduCal API to create an appointment:

1. ScheduCal creates a calendar event directly in Microsoft Exchange via the Graph API (`POST /me/events`)
2. Exchange generates and delivers a standards-compliant calendar invitation with correct ICS metadata to all attendees
3. Attendee responses (accept, decline, tentative) flow back through Exchange and are surfaced via your webhook

When you update an appointment (`PUT`), Exchange sends a proper meeting update notification to all attendees — no re-sending of ICS files required. When you cancel (`DELETE`), Exchange sends a cancellation with the correct `METHOD:CANCEL` iTIP header.

### Why Exchange

Microsoft Exchange is the enterprise calendar standard:
- ~350 million business users worldwide operate on Exchange / Microsoft 365
- Exchange enforces iTIP (iCalendar Transport-Independent Interoperability Protocol) natively, handling the full meeting lifecycle: invite → accept/decline → update → cancel
- Exchange interoperates natively with Google Calendar, Apple Calendar, and all CalDAV clients — no special-casing required
- ScheduCal integrates into your attendees' existing Exchange environment rather than requiring any client-side setup

---

## ICS Calendar Compliance

Exchange delivers calendar invitations as standards-compliant ICS (iCalendar, RFC 5545) internally. ScheduCal ensures the correct RFC 5545 semantics are maintained across the full appointment lifecycle:

### UID Persistence

Every appointment has a single, persistent UID assigned at creation time. This UID is carried through all subsequent updates and the final cancellation. Per RFC 5545, the UID links all versions of an event (create, update, cancel) so attendee calendar clients can correctly correlate them.

If UID consistency breaks — a common mistake in DIY ICS implementations — calendar clients create duplicate appointments instead of updating the existing one.

### SEQUENCE Incrementing

The `SEQUENCE` property is incremented on every update per RFC 5545 §3.8.7.4. This tells attendee calendar clients that a newer version of the event has arrived, causing them to replace the old version rather than create a duplicate entry.

ScheduCal manages SEQUENCE automatically. Customers never need to track or pass a sequence number.

### Cancellation METHOD

When an appointment is cancelled, ScheduCal issues a calendar message with `METHOD:CANCEL` per iTIP (RFC 5546). This is the correct mechanism for propagating a cancellation to attendee calendars — it removes the event from the attendee's calendar rather than leaving a phantom entry.

Sending a plain cancellation email without the correct iTIP METHOD header is the most common implementation error in calendar invitation systems. ScheduCal handles this correctly.

---

## Gmail Priming (Required Infrastructure)

When inviting a Gmail address for the first time from a given sending account, ScheduCal performs a **Gmail priming step** before sending the calendar invitation. This is required infrastructure, not an optional enhancement.

**Why it exists:** The Microsoft Exchange → Gmail calendar delivery path requires an initial email handshake to establish the sending relationship. Without this priming step, Gmail's spam filters and calendar integration heuristics will frequently suppress the calendar invitation, and the event will not appear in the attendee's Google Calendar.

**What it does:** ScheduCal sends a pre-invitation email to the Gmail address before the calendar invitation. This establishes trust between the sending Exchange account and the Gmail recipient, ensuring the calendar invitation lands correctly.

**The `gmailFirstInvite` response field** indicates when this priming step was executed. When `gmailFirstInvite: true`, you should display guidance to the attendee:

> "Please check your spam folder and confirm the calendar invitation from ScheduCal."

After the first successful invitation, subsequent invitations to the same Gmail address from the same account do not require priming (`gmailFirstInvite: false`).

---

## What ScheduCal Hides From You

Integrating directly with the Microsoft Graph API requires:
- Registering an Azure AD application
- Implementing OAuth 2.0 consent flows for each Microsoft 365 tenant
- Managing token refresh and per-tenant credential storage
- Handling Graph API throttling and retry logic (10,000 requests per 10 minutes per tenant)
- Maintaining Exchange subscription management for response tracking
- Enforcing UID/SEQUENCE/METHOD semantics correctly across all operations

ScheduCal handles all of this. Your integration is three API calls and one webhook.

---

## Cross-Calendar Compatibility

ScheduCal's Exchange-based delivery works natively across:

| Calendar Client | Compatibility |
|----------------|---------------|
| Microsoft Outlook (365 / Exchange) | Native — full iTIP lifecycle |
| Google Calendar (Gmail) | Supported — Gmail priming required on first invite |
| Apple Calendar / iCloud | Supported — Exchange-native ICS delivery |
| Any CalDAV client | Supported — Exchange interoperates with CalDAV |

---

## Further Reading

- [API Reference](api-reference.md) — Endpoint documentation
- [Webhooks Guide](webhooks.md) — Receiving attendee responses
- [Strategy: MS Graph & Exchange-Native Delivery](strategy-pivot-msgraph-exchange.md) — Architecture deep dive
