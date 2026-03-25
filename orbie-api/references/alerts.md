# Alert Rules

Alert rules define conditions that trigger notifications (e.g. queue exceeds N people). **A scene must exist before creating a rule.**

## Create an Alert Rule

```http
POST /api/v1/alerts/rules
Authorization: Bearer <API_KEY>
```

```json
{
  "name": "Queue Too Long",
  "sceneId": "scene-uuid",
  "severity": "warning",
  "conditions": [
    {
      "metricPath": "current.inQueue",
      "operator": "GT",
      "threshold": 10
    }
  ],
  "cooldownMinutes": 5
}
```

**Common `metricPath` values:**

| metricPath | Description |
|------------|-------------|
| `current.inQueue` | Current number of people waiting |
| `current.inService` | Current number of people being served |
| `rolling5Min.completed.waitTimeAvgSeconds` | 5-minute average wait time (seconds) |
| `rolling5Min.abandonRate` | 5-minute abandonment rate (0–1) |
| `rolling30Min.completed.waitTimeP95Seconds` | 30-minute P95 wait time (seconds) |

**`operator` values:** `GT`, `GTE`, `LT`, `LTE`, `EQ`, `NEQ`

**`logicOperator` values:** `OR` (default — trigger if **any** condition matches) or `AND` (trigger only if **all** conditions match). Can be omitted for a single condition.

**Multi-condition example:**

```json
{
  "conditions": [
    { "metricPath": "current.inQueue", "operator": "GT", "threshold": 15 },
    { "metricPath": "rolling5Min.completed.waitTimeAvgSeconds", "operator": "GT", "threshold": 300 }
  ],
  "logicOperator": "OR"
}
```

> **✅ After creating alert rules:** Confirm, then ask: "Would you like to set up a webhook so you get push-notified in real time when alerts fire? Or if you prefer to poll manually, we can skip this and start analysis now."

---

## Query Alert History

```http
GET /api/v1/analysis/alerts?deviceId=xxx&startDate=2026-01-01T00:00:00Z
```

Optional filters: `severity=warning|critical`, `sceneId=xxx`, `endDate=xxx`.

**`severity` filter:** `warning` (default) or `critical`. Omit to get all. If the user queries `critical` and gets no results, remind them to also check `warning` — it's the default severity when creating rules.
