# ScheduCal API Documentation

Official documentation for the ScheduCal calendar scheduling API.

ScheduCal is a **fully managed calendar invitation service** built on Microsoft Exchange and the Microsoft Graph API. Your integration is three API calls (Create, Update, Cancel) and one webhook. ScheduCal handles Exchange delivery, ICS compliance (UID persistence, SEQUENCE incrementing, cancellation METHOD), and cross-client compatibility — including Gmail priming.

## Documentation

- [Architecture Overview](architecture.md) - How ScheduCal works: Exchange/Graph delivery layer, ICS compliance semantics, Gmail priming
- [API Reference](api-reference.md) - Core API endpoints for appointments
- [Webhooks Guide](webhooks.md) - Real-time event notifications
- [MS Graph & Exchange Strategy](strategy-pivot-msgraph-exchange.md) - Architecture deep dive
- [Postman Collection](ScheduCal-API.postman_collection.json) - Import into Postman for testing

## Quick Start

1. Register at [scheducal.com](https://scheducal.com) to get your API credentials
2. Use your `apiKey` and `apiSecret` in all API requests
3. Start creating appointments!

```bash
curl -X POST "https://api.scheducal.com/api/v1/appointments" \
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "your-api-key",
    "apiSecret": "your-api-secret",
    "appointmentSubject": "Team Meeting",
    "appointmentStart": "2026-01-15T10:00:00",
    "appointmentEnd": "2026-01-15T11:00:00",
    "appointmentTimeZone": "America/Los_Angeles"
  }'
```

## Support

For API support, contact support@scheducal.com
