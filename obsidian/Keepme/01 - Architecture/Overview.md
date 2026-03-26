# Architecture Overview

## System Diagram

```
[keepme-react-v4]  ←→  [Node.js API (auth)]
        ↓
[keepme-v4 PHP API]  ←→  [MySQL (multi-tenant)]
        ↓                ←→  [MongoDB (per-tenant)]
[keepme-services]  ←→  [Kafka]
        ↓
[Data Warehouse Pipeline]  →  [S3 → Glue → Redshift]
```

## Feature Modules

| Module | Description |
|--------|-------------|
| Sales | Leads, funnels, CTA, forecasts |
| Membership | Members, subscriptions, attendance, retention |
| Engagement | Campaigns (email, SMS, WhatsApp), automations |
| Bookings | Class/appointment scheduling, slots |
| Reports | Sales analysis, membership insights, NPS, custom reports |
| Settings | Team, venues, timezones, branding, integrations |

## Multi-Tenancy

- Company code (EU, AU, US, GB) is embedded in JWT
- Middleware dynamically routes each request to the correct MySQL/MongoDB connection
- Connection mapping defined in `keepme-v4/config/db_connections.php`
- Region variants: `EU`, `AU`, `US`, `GB` (legacy) and `V4EU`, `V4AU`, `V4US` (new)

## External Integrations

| Service | Purpose |
|---------|---------|
| SendGrid | Email delivery |
| Twilio | SMS + WhatsApp |
| Stripe (Cashier) | Billing/subscriptions |
| AWS S3 | File storage + data warehouse |
| Mindbody | CRM webhook integration |
| Firebase | Analytics |
| google-libphonenumber | Phone validation |

## Related Notes

- [[Frontend]]
- [[Backend]]
- [[Microservices]]
- [[../02 - Infrastructure/Kubernetes]]
- [[../03 - Data Warehouse/Pipeline Overview]]
