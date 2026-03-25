# Integration Reference

## Full TypeScript Example

When the user asks "write me the integration code", use this as the base:

```typescript
const BASE_URL = process.env.ORBIE_API_URL // Must be set in .env — confirm URL with Orbie contact
const API_KEY = process.env.ORBIE_API_KEY  // syncai_3rd_...

async function orbie<T>(method: string, path: string, body?: unknown): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    method,
    headers: {
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: body ? JSON.stringify(body) : undefined,
  })
  const json = await res.json()
  if (!json.success) throw new Error(`[${json.error?.code}] ${json.error?.message}`)
  return json.data
}

// 1. Create device
const device = await orbie('POST', '/api/v1/devices', {
  name: 'Main Entrance Camera',
  rtspUrl: 'rtsp://192.168.1.100:554/stream1',
  utcOffset: 'UTC+8',
  plan: 'level_01',
})

// 2. Create scene
const scene = await orbie('POST', '/api/v1/scenes', {
  deviceId: device.id,
  name: 'Queue Monitor',
  analysisType: 'queue_analysis',
  config: {
    zones: {
      queueZone: {
        type: 'polygon',
        points: [[0.1, 0.2], [0.5, 0.2], [0.5, 0.9], [0.1, 0.9]],
      },
    },
  },
})

// 3. Start analysis
await orbie('POST', `/api/v1/devices/${device.id}/analysis/start`, {
  sceneId: scene.id,
})

// 4. Query status
const status = await orbie('GET', `/api/v1/devices/${device.id}/analysis/status`)
console.log(status)
```

---

## curl Quick Reference

When the user asks you to **do** something (not write code), run these directly in the terminal. Assume `ORBIE_API_KEY` and `ORBIE_API_URL` are set in the environment.

```bash
# Create a device
curl -s -X POST "$ORBIE_API_URL/api/v1/devices" \
  -H "Authorization: Bearer $ORBIE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Main Entrance", "rtspUrl": "rtsp://...", "utcOffset": "UTC+8", "plan": "level_01"}'

# List devices
curl -s "$ORBIE_API_URL/api/v1/devices" \
  -H "Authorization: Bearer $ORBIE_API_KEY"

# Create a scene
curl -s -X POST "$ORBIE_API_URL/api/v1/scenes" \
  -H "Authorization: Bearer $ORBIE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"deviceId": "<DEVICE_ID>", "name": "Queue Monitor", "analysisType": "queue_analysis", "config": {"zones": {"queueZone": {"type": "polygon", "points": [[0.1,0.2],[0.5,0.2],[0.5,0.9],[0.1,0.9]]}}}}'

# Start analysis
curl -s -X POST "$ORBIE_API_URL/api/v1/devices/<DEVICE_ID>/analysis/start" \
  -H "Authorization: Bearer $ORBIE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sceneId": "<SCENE_ID>"}'

# Check analysis status
curl -s "$ORBIE_API_URL/api/v1/devices/<DEVICE_ID>/analysis/status" \
  -H "Authorization: Bearer $ORBIE_API_KEY"

# Stop analysis
curl -s -X POST "$ORBIE_API_URL/api/v1/devices/<DEVICE_ID>/analysis/stop" \
  -H "Authorization: Bearer $ORBIE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"reason": "user_request"}'

# Check usage
curl -s "$ORBIE_API_URL/api/v1/usage/user/summary" \
  -H "Authorization: Bearer $ORBIE_API_KEY"

# Health check (no auth needed)
curl -s "$ORBIE_API_URL/health"
```

After each call, check `"success": true` in the response. If false, read `error.code` and `error.message` and explain the issue to the user.

---

## Health Check (no auth required)

Use these to verify the API server is operational:

```http
GET /health        # Full status (DB + Redis)
GET /health/live   # Liveness probe
GET /health/ready  # Readiness probe
```
