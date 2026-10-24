# ScheduCal API Documentation

Official documentation for the ScheduCal calendar scheduling API.

ScheduCal is a **fully managed calendar invitation service** built on Microsoft Exchange and the Microsoft Graph API. Your integration is three API calls (Create, Update, Cancel) and one webhook. ScheduCal handles Exchange delivery, ICS compliance (UID persistence, SEQUENCE incrementing, cancellation METHOD), and cross-client compatibility — including Gmail priming.

## Documentation

- [Architecture Overview](architecture.md) - How ScheduCal works: Exchange/Graph delivery layer, ICS compliance semantics, Gmail priming
- [API Reference](api-reference.md) - Core API endpoints, including How Invitations Work
- [Webhooks Guide](webhooks.md) - Real-time event notifications
- [Postman Collection](ScheduCal-API.postman_collection.json) - Import into Postman for testing

## Quick Start

1. Register at [scheducal.com](https://scheducal.com) to get your API credentials
2. Authenticate every request with HTTP Basic authentication: your API key as the username and your API secret as the password (`Authorization: Basic base64(apiKey:apiSecret)`). With curl, that is `-u "$API_KEY:$API_SECRET"`
3. Start creating appointments!

```bash
export API_KEY="your-api-key"
export API_SECRET="your-api-secret"

curl -X POST "https://api.scheducal.com/api/v1/appointments" \
  -u "$API_KEY:$API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "appointmentSubject": "Team Meeting",
    "appointmentStart": "2026-01-15T10:00:00",
    "appointmentEnd": "2026-01-15T11:00:00",
    "appointmentTimeZone": "America/Los_Angeles"
  }'
```

See [Authentication](api-reference.md#authentication) for Node.js, Python and C# examples. Sending `apiKey` and `apiSecret` in the JSON request body is deprecated: it still works for now, but will be removed.

## Support

For API support, contact support@scheducal.com
