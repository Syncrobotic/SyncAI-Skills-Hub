# Devices

A device represents a physical IP camera with an RTSP stream. **A device must exist before creating a scene.**

## Create a Device

```http
POST /api/v1/devices
Authorization: Bearer <API_KEY>
```

```json
{
  "name": "Main Entrance Camera",
  "rtspUrl": "rtsp://192.168.1.100:554/stream1",
  "rtspUsername": "admin",
  "rtspPassword": "password123",
  "utcOffset": "UTC+8",
  "plan": "level_01",
  "tags": ["entrance"]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | ✅ | 1–200 characters |
| `rtspUrl` | ✅ | Must start with `rtsp://` or `rtsps://` |
| `rtspUsername` / `rtspPassword` | ❌ | Basic Auth credentials |
| `rtspToken` | ❌ | Bearer Token Auth (use one or the other) |
| `utcOffset` | ❌ | e.g. `UTC+8`, `UTC-5`, `UTC+5:30` — defaults to `UTC+0` |
| `plan` | ❌ | `level_01` (default) or `level_02` |
| `schedule` | ❌ | See Schedule Configuration below |

**Plan comparison:**

| | `level_01` | `level_02` |
|---|---|---|
| Analysis hours | 150 hrs/month | 250 hrs/month |
| Alert quota | 3,000/month | 5,000/month |
| Attribute detection (gender/age) | ❌ | ✅ 5,000/month |
| VLM AI summary | ❌ | ✅ 10M tokens/month |

**Response 201:** Returns the device object. **Save `data.id` — you'll need it to create a scene.**

> **✅ After creating a device:** Confirm success, then ask: "Would you like to set up a scene now? A scene defines which zones in the camera frame to analyze."

---

## Schedule Configuration

If a schedule is set, analysis starts and stops automatically — no manual API calls needed.

Time is expressed in 30-minute slots. Slot 0 = 00:00, slot 16 = 08:00 (device local time):

```json
// Every day 08:00–22:00 (slots 16–43)
{
  "schedule": {
    "dailySlots": [16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43]
  }
}

// Weekdays only
{
  "schedule": {
    "dailySlots": [],
    "weeklySlots": [
      [],                                                              // Sunday
      [16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35], // Monday
      [16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35], // Tuesday
      [16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35], // Wednesday
      [16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35], // Thursday
      [16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35], // Friday
      []                                                               // Saturday
    ]
  }
}
```

---

## Other Device APIs

```http
GET    /api/v1/devices                     # List (query: ?page=1&limit=20&status=active)
GET    /api/v1/devices/{id}                # Get details
PATCH  /api/v1/devices/{id}                # Partial update
DELETE /api/v1/devices/{id}                # Delete
GET    /api/v1/devices/status/online       # List online devices
POST   /api/v1/devices/{id}/probe          # Manually trigger RTSP connection probe
GET    /api/v1/devices/{id}/quota          # Query device quota usage
```
