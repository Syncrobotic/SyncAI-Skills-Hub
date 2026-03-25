# Webhooks

Set up a webhook to receive **real-time push notifications** when alerts fire, devices go offline, quotas are exceeded, etc. — instead of polling for alert history.

> 💡 Each user account supports one webhook endpoint. Set it up once and subscribe to the events you care about.

## Create a Webhook

```http
POST /api/v1/users/me/webhook
Authorization: Bearer <API_KEY>
```

```json
{
  "url": "https://your-server.com/webhook/orbie",
  "name": "My Orbie Webhook",
  "subscribedTypes": ["alert_triggered", "device_offline", "analysis_error"],
  "timeoutMs": 5000,
  "maxRetries": 3
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `url` | ✅ | HTTPS URL to receive events (max 500 chars) |
| `name` | ❌ | Display label (max 100 chars) |
| `secret` | ❌ | 16–128 char signing secret — auto-generated if omitted |
| `subscribedTypes` | ❌ | Array of event types to subscribe to — omit to receive **all** events |
| `timeoutMs` | ❌ | Request timeout in ms (1000–30000, default 5000) |
| `maxRetries` | ❌ | Retry attempts on failure (0–5, default 3) |

**Event types (`subscribedTypes`):**

| Event | Description |
|-------|-------------|
| `alert_triggered` | An alert rule condition was met |
| `analysis_error` | Analysis encountered an error |
| `quota_warning` | 80% of quota consumed |
| `quota_exceeded` | Quota limit reached |
| `billing_cycle_completed` | Monthly billing cycle finished |
| `maintenance_scheduled` | Planned maintenance notification |

> **✅ After setting up a webhook:** Confirm the endpoint was registered, then ask: "Would you like to send a test delivery to verify your server is receiving events? Or shall we start analysis now?"

---

## Other Webhook APIs

```http
GET    /api/v1/users/me/webhook                    # Get current config
PATCH  /api/v1/users/me/webhook                    # Update URL or subscriptions
DELETE /api/v1/users/me/webhook                    # Remove webhook
POST   /api/v1/users/me/webhook/test               # Send a test delivery
GET    /api/v1/users/me/webhook/secret             # Retrieve signing secret
POST   /api/v1/users/me/webhook/secret/regenerate  # Rotate signing secret (invalidates old)
GET    /api/v1/users/me/webhook/deliveries         # Query delivery history
```

> ⚠️ Rotating the secret immediately invalidates the old one — update `ORBIE_WEBHOOK_SECRET` in your server before rotating.

---

## Incoming Webhook Format

When an event fires, Orbie sends a `POST` to your endpoint with these headers:

```
Content-Type: application/json
X-SaaS-Signature: sha256=<hmac>
X-SaaS-Timestamp: <unix_timestamp>
X-SaaS-Event: alert_triggered
X-SaaS-Delivery-Id: <uuid>
```

**Body (`alert_triggered` example):**

```json
{
  "id": "event-uuid",
  "type": "alert_triggered",
  "timestamp": "2026-01-29T10:30:00Z",
  "data": {
    "ruleName": "Queue Too Long",
    "deviceId": "device-uuid",
    "sceneId": "scene-uuid",
    "severity": "warning",
    "metricPath": "current.inQueue",
    "threshold": 10,
    "actualValue": 15,
    "message": "Queue count (15) exceeded threshold (10)"
  }
}
```

---

## Verify Webhook Signature

**Always verify the signature** before processing the payload (prevents spoofed requests):

```typescript
import crypto from 'crypto'

function verifyWebhook(
  rawBody: string,
  signature: string,   // X-SaaS-Signature header
  timestamp: string,   // X-SaaS-Timestamp header
  secret: string,      // from ORBIE_WEBHOOK_SECRET env var
): boolean {
  const message = `${timestamp}.${rawBody}`
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(message)
    .digest('hex')
  // Use timingSafeEqual to prevent timing attacks
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))
}

// Express example — use raw body, not parsed JSON
app.post('/webhook/orbie', express.text({ type: '*/*' }), (req, res) => {
  const valid = verifyWebhook(
    req.body,
    req.headers['x-saas-signature'] as string,
    req.headers['x-saas-timestamp'] as string,
    process.env.ORBIE_WEBHOOK_SECRET!,
  )
  if (!valid) return res.status(401).send('Invalid signature')

  const event = JSON.parse(req.body)
  if (event.type === 'alert_triggered') {
    // handle alert...
  }
  res.sendStatus(200)
})
```

> The signing secret is shown once when you create the webhook. Retrieve it any time with `GET /api/v1/users/me/webhook/secret` and store it as `ORBIE_WEBHOOK_SECRET` in your `.env`.

---

## Query Delivery History

To debug failed deliveries:

```http
GET /api/v1/users/me/webhook/deliveries?status=failed&startDate=2026-01-01T00:00:00Z
```

Optional filters: `status=success|failed|pending`, `deviceId`, `sceneId`, `startDate`, `endDate`.
