# Analysis

## Start / Stop Analysis

> ⚠️ **There is NO `status` field, NO `PATCH /devices/{id}` activation, and NO separate "activate" endpoint.**  
> The **only** way to start analysis is `POST /api/v1/devices/{deviceId}/analysis/start`.  
> Never guess or invent endpoints — if unsure, re-read this file.

If the device has a `schedule` configured, the system starts and stops automatically — **no manual calls needed**. Only use these APIs for on-demand, manual control.

### Start Analysis

```http
POST /api/v1/devices/{deviceId}/analysis/start
Authorization: Bearer <API_KEY>
```

```json
{
  "sceneId": "scene-uuid"
}
```

Response is `202 Accepted` with `data.status` set to `initializing`. It transitions to `running` in ~3 seconds.

### Stop Analysis

```http
POST /api/v1/devices/{deviceId}/analysis/stop
Authorization: Bearer <API_KEY>
```

```json
{ "reason": "user_request" }
```

### Query Analysis Status

```http
GET /api/v1/devices/{deviceId}/analysis/status
```

### Query Analysis Task List

```http
GET /api/v1/analysis/tasks?deviceId=xxx&status=running
```

> **✅ After starting analysis:** Check status once to confirm it's `running`, then tell the user:
> "The system is now monitoring your camera zones for queue activity. Would you like to:
> - **Set up alert rules** — get notified when queue length or wait time exceeds a threshold
> - **Check alert history** — see if any alerts have already fired
> - **Check usage** — see your quota consumption"
>
> **If the device plan is `level_02`**, also add:
> - **"Review AI insights** — each alert automatically triggers a VLM snapshot analysis with a plain-English summary of what the camera saw (e.g. '12 people queuing near counter 3, estimated wait ~8 minutes'). You can check these any time."
>
> If you don't know the device plan, check it first:
> ```http
> GET /api/v1/devices/{deviceId}
> ```
> and read `data.plan`. If `level_02`, surface the insights option.

---

## Review Alerts & AI Insights

### Query Alert History

When the user wants to see what alerts have fired:

```http
GET /api/v1/analysis/alerts?deviceId=xxx&startDate=2026-01-01T00:00:00Z
```

Optional filters: `severity=warning|critical`, `sceneId=xxx`, `endDate=xxx`.

After showing results, offer: "Would you like me to fetch the AI insight (VLM summary) for any of these alerts?"

### AI Insights (level_02 only)

When an alert fires on a `level_02` device, Orbie automatically runs a Vision Language Model (VLM) analysis on the camera snapshot — generating a human-readable summary like *"12 people queuing near counter 3, estimated wait ~8 minutes"*.

**List insights:**
```http
GET /api/v1/analysis/insights?deviceId=xxx&startDate=2026-01-01T00:00:00Z
```

Optional filters: `sceneId`, `severity`, `vlmStatus=pending|completed|failed`, `hasVlmSummary=true`

**Get full insight detail (includes VLM summary + snapshot):**
```http
GET /api/v1/analysis/insights/{insightId}
```

**Natural language search across insights:**
```http
POST /api/v1/analysis/insights/search
Authorization: Bearer <API_KEY>
```
```json
{ "q": "people crowding near entrance after 10pm" }
```
This calls an LLM to parse the query into filters — consumes VLM tokens (~200–400 per search).

**`vlmStatus` values:**

| Status | Meaning |
|--------|---------|
| `pending` | Alert fired, VLM analysis queued |
| `completed` | AI summary ready to read |
| `failed` | VLM failed (no quota charged) |
| `skipped` | Device not on level_02 or VLM not enabled |

> **✅ After showing insights:** Offer next steps: "Would you like to adjust your alert rules, check usage, or is there anything else you'd like to monitor?"
