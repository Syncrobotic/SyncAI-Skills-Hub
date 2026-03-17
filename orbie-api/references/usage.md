# Usage & Quota

When the user asks "how many analysis hours do I have left?" or "is my quota enough?", call these endpoints.

## Query Usage

```http
GET /api/v1/usage/user/summary          # Overall user usage (analysis / alerts / VLM)
GET /api/v1/usage/devices/{deviceId}    # Usage for a specific device
```

**Response example:**

```json
{
  "analysis": { "used": 54000, "limit": 540000, "balance": 486000, "isExceeded": false },
  "alert":    { "used": 150,   "limit": 3000,   "balance": 2850,   "isExceeded": false }
}
```

**Units:**
- `analysis.used/limit/balance` — in **seconds**. Divide by 3600 to convert to hours.
- `alert.used/limit/balance` — **counts** (number of alert triggers).

**curl example:**
```bash
curl -s "$ORBIE_API_URL/api/v1/usage/user/summary" \
  -H "Authorization: Bearer $ORBIE_API_KEY"
```
