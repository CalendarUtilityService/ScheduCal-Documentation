# ScheduCal API Documentation

Official documentation for the ScheduCal calendar scheduling API.

## Documentation

- [API Reference](api-reference.md) - Core API endpoints for appointments
- [Webhooks Guide](webhooks.md) - Real-time event notifications
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
