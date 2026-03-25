# Scenes

A scene defines the analysis zones and type within a camera frame. **A device must exist before creating a scene.** Only `queue_analysis` is currently supported.

## Create a Scene

```http
POST /api/v1/scenes
Authorization: Bearer <API_KEY>
```

```json
{
  "deviceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Counter Queue Monitor",
  "analysisType": "queue_analysis",
  "config": {
    "zones": {
      "queueZone": {
        "type": "polygon",
        "points": [[0.1, 0.2], [0.5, 0.2], [0.5, 0.9], [0.1, 0.9]]
      },
      "checkoutZone": {
        "type": "polygon",
        "points": [[0.6, 0.3], [0.9, 0.3], [0.9, 0.8], [0.6, 0.8]]
      }
    },
    "parameters": {
      "confidenceThreshold": 0.6,
      "class": "person",
      "entryThresholdSeconds": 3,
      "exitThresholdSeconds": 1
    }
  }
}
```

**Zone coordinate rules:**
- Use normalized coordinates `[x, y]` in the range `0.0–1.0`
- `(0, 0)` is the **top-left** corner of the frame
- Provide at least one of `queueZone` (waiting area) or `checkoutZone` (service area); they must not overlap
- Only `polygon` type is supported; minimum 3 points

**`parameters` fields (all optional):**

| Field | Type | Range | Default | Description |
|-------|------|-------|---------|-------------|
| `confidenceThreshold` | Number | 0.0–1.0 | 0.6 | Minimum detection confidence to count a person/object |
| `class` | String | `person`, `vehicle` | `person` | Detection target class |
| `entryThresholdSeconds` | Number | 1–60 | 3 | Seconds an object must be present before counted as entered |
| `exitThresholdSeconds` | Number | 1–60 | 1 | Seconds after disappearance before counted as exited |

**Response 201:** Returns the scene object. **Save `data.id` — you'll need it to start analysis and create alert rules.**

> **✅ After creating a scene:** Confirm it was saved, then ask: "Would you like to set up alert rules? Rules let you get notified when queue length or wait time exceeds a threshold. (This is optional — you can also start analysis now and add rules later.)"

---

## Other Scene APIs

```http
GET    /api/v1/scenes                      # List (query: ?deviceId=xxx)
GET    /api/v1/scenes/{id}                 # Get details
PATCH  /api/v1/scenes/{id}                 # Update (not allowed while analysis is running)
DELETE /api/v1/scenes/{id}                 # Delete (not allowed while analysis is running)
```
