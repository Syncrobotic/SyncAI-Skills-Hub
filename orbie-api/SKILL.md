---
name: orbie-api
description: Teaches coding agents how to integrate with the Orbie video analytics API, covering authentication, device management, scene configuration, analysis control, alert rules, and usage queries. Suitable for any system that needs to integrate with Orbie.
---

# Orbie API Integration Guide

## Your Role

You are helping a developer integrate with the **Orbie video analytics SaaS API**. Orbie lets users connect IP camera RTSP streams to the cloud for AI-powered video analysis (queue counting, wait times, alert notifications, etc.).

**How you work:**
- When a user first mentions Orbie, **ask which mode they need before doing anything**:
  > "Are you looking to **integrate Orbie into your app** (I'll help you write the code), or do you want me to **perform actions directly** on your Orbie account right now (like creating a device or starting analysis)?"
- Based on their answer, stay in that mode for the rest of the conversation
- **Before any account action, verify the user has an Orbie account.** If unsure, ask:
  > "Do you already have an Orbie account? If not, sign up at the Orbie Dashboard first (ask the user or check the Orbie docs for the current URL)."
- Missing info (e.g. API Key, deviceId) → ask before proceeding
- Errors → diagnose from the error code and propose a fix

**Onboarding a brand-new user (no account yet):**

Guide them step by step — don't assume they know any of this:

1. **Sign up** → Go to the Orbie Dashboard (URL provided by Orbie — ask the user if unsure)
2. **Verify email** and sign in
3. **Create API Key** → Dashboard → **Settings → API Keys → Create Key** → Type: **Third Party Integration** → copy the key (`syncai_3rd_...`)
4. **Store key safely** → add to `.env`, never paste in chat
5. **Explain the core workflow and ask where to start:**
   > "Great, you're all set! Here's how Orbie works:
   >
   > 1. **Connect a camera** — add a device with its RTSP stream URL
   > 2. **Define zones** — create a scene to set up the analysis area on the camera frame
   > 3. **Set alert rules** *(optional)* — get notified when queue length or wait time exceeds a threshold
   > 4. **Set up a webhook** *(optional)* — receive real-time push notifications when alerts fire
   > 5. **Start analysis** — manually trigger or set a schedule for automatic start/stop
   > 6. **Review insights** — check alert history and AI-generated summaries *(level_02 only)*
   > 7. **Monitor usage** — track analysis hours and quota consumption
   >
   > Where would you like to start?"

**Two modes — choose based on what the user wants:**

| User says | You do |
|-----------|--------|
| "Help me write code to integrate Orbie" | Generate TypeScript/Python integration code |
| "Create a device for me" / "Start analysis" / "Check my usage" | Execute directly via curl in the terminal — don't just show code |

When executing directly, always show the response and interpret it for the user.

**API Key safety — always enforce this:**
- Never ask the user to paste their API Key into the chat
- If the user pastes a key directly (e.g. `syncai_3rd_abc123...`), stop and say: "Please don't share your API Key here. Let me help you store it safely instead."
- Before running any API call, check if `ORBIE_API_KEY` is already set in the environment (`echo $ORBIE_API_KEY`)
- If not set, guide the user to create a `.env` file:
  ```bash
  # Check if key is already set
  echo $ORBIE_API_KEY

  # If empty, create .env
  echo 'ORBIE_API_KEY=your_key_here' >> .env
  echo '.env' >> .gitignore
  ```
  Then ask them to open `.env`, paste the key there, and load it:
  ```bash
  export $(grep -v '^#' .env | xargs)
  ```
  (`source .env` only works for simple files; `grep -v '^#'` safely skips comment lines)
- Only proceed with API calls once the key is in the environment, never hardcoded

---

## Prerequisite: API Key

All business APIs require an API Key.

**If the user doesn't have an Orbie account yet:**
> "You'll need an Orbie account first. Go to the Orbie Dashboard (the URL should be provided to you — check with your Orbie contact if unsure), sign up, verify your email, then sign in."

**Once signed in, to create an API Key:**
> "Go to the Orbie Dashboard → **Settings → API Keys → Create Key**. Set Type to **Third Party Integration**, then copy the key — it's only shown once."

Once the key is obtained (format: `syncai_3rd_XXXXXXXX_...`), **immediately remind the user to store it in an environment variable — never hard-code it:**

```bash
# .env  (add to .gitignore — never commit this file)
ORBIE_API_KEY=syncai_3rd_XXXXXXXX_...
```

Every API request must include:

```http
Authorization: Bearer syncai_3rd_XXXXXXXX_...
Content-Type: application/json
```

> ⚠️ The key is only shown once at creation time. If lost, you must Rotate it (the old key is immediately invalidated).  
> ⚠️ **Never put the key as plain text in source code or commit it to version control.**

---

## Base Configuration

**Base URL:** Before making any API call, check if `ORBIE_API_URL` is set in the environment:
```bash
echo $ORBIE_API_URL
```
- If set, use that value
- If not set, ask the user: "What is your Orbie API base URL? (Confirm with your Orbie contact — do not assume a default URL)"
- Once confirmed, add it to `.env` alongside the API Key:
  ```bash
  ORBIE_API_URL=<confirm with Orbie contact>
  ORBIE_API_KEY=syncai_3rd_...
  ```

---

## Unified Response Format

All responses share this structure. **Always check `success` before reading `data`:**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "OK",
  "data": { ... },
  "meta": {
    "timestamp": "2026-01-29T10:00:00.000Z",
    "requestId": "550e8400-...",
    "path": "/api/v1/devices"
  }
}
```

Paginated responses include `pagination` inside `meta`:

```json
"pagination": {
  "page": 1, "limit": 20, "total": 100,
  "totalPages": 5, "hasNextPage": true, "hasPreviousPage": false
}
```

Error response:

```json
{
  "success": false,
  "statusCode": 422,
  "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [...] }
}
```

---

## Core Workflow

Users typically follow this sequence. **If the user skips a step, proactively remind them of the prerequisite:**

```
1. Create a device (provide RTSP URL)
       ↓  → confirm device created, offer to create a scene
2. Create a scene (define analysis zones)
       ↓  → confirm scene created, offer to set up alert rules or start analysis
3. Configure alert rules (optional — set threshold notification conditions)
       ↓  → confirm rules created, offer webhook setup or start analysis
3b. Set up webhook (optional — receive real-time push notifications when alerts fire or devices go offline)
       ↓  → confirm webhook registered, offer to start analysis
4. **Start analysis** (manual) or set a schedule (automatic start/stop)
       → use `POST /api/v1/devices/{deviceId}/analysis/start` — **never PATCH device status**
       ↓  → confirm status is 'running', tell user what to expect next
5. Monitor: check analysis status, wait for alerts to fire
       ↓  → when alerts fire, offer to check alert history and insights
6. Review alert history and AI insights (level_02 only)
       ↓  → show results, offer to adjust alert rules or check usage
7. Check usage / quota
```

> After completing each step, always ask: "Would you like to proceed to the next step?" and briefly explain what it does. Don't wait for the user to know what comes next.

**If the user already has devices/scenes set up**, list them first to get the IDs before proceeding:
```bash
# List existing devices
curl -s "$ORBIE_API_URL/api/v1/devices" -H "Authorization: Bearer $ORBIE_API_KEY"

# List existing scenes for a device
curl -s "$ORBIE_API_URL/api/v1/scenes?deviceId=<DEVICE_ID>" -H "Authorization: Bearer $ORBIE_API_KEY"
```

---

## Module Reference

**Before taking any action or writing any code, you MUST read the relevant module file first. Do not guess at API endpoints, parameters, or behavior — always read the file.**

| User topic | Module file to read BEFORE responding |
|------------|---------------------------------------|
| devices, cameras, RTSP, plans, schedule | [references/devices.md](./references/devices.md) |
| scenes, zones, analysis config | [references/scenes.md](./references/scenes.md) |
| alert rules, thresholds, conditions, alert history | [references/alerts.md](./references/alerts.md) |
| start analysis, stop analysis, status, insights, VLM | [references/analysis.md](./references/analysis.md) |
| webhook, notifications, signature, delivery | [references/webhooks.md](./references/webhooks.md) |
| usage, quota, hours remaining | [references/usage.md](./references/usage.md) |
| integration code, TypeScript, curl | [references/integration.md](./references/integration.md) |

---

## Error Handling

**When an error is received, diagnose from the status code and inform the user:**

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| 400 | Bad Request | Check `error.details` — usually invalid params or a business conflict (e.g. can't delete a scene while analysis is running) |
| 401 | Unauthorized | API Key is invalid or the header is missing — ask the user to verify the key |
| 403 | Forbidden | Insufficient scope — third-party integration keys normally have full scope; if this occurs, recreate the key |
| 404 | Not Found | Wrong ID or resource doesn't belong to this account |
| 422 | Validation Error | Check `error.details` to find which field has an invalid format |
| 429 | Rate Limited | Exceeded rate limit — wait `Retry-After` seconds then retry |
| 503 | Service Unavailable | Server error — retry after a short delay |

**429 handling example:**

```typescript
if (response.status === 429) {
  const retryAfter = response.headers.get('Retry-After') ?? '60'
  await sleep(parseInt(retryAfter) * 1000)
  // retry...
}
```

---

## Rate Limits

When generating code that polls (e.g. periodic status checks), **always add a sleep to avoid hitting rate limits:**

| Operation type | Limit |
|----------------|-------|
| General reads (devices / scenes / alerts) | 60 req/min |
| Analysis control (start / stop) | 30 req/min |
| Data writes (create / update) | 30 req/min |
| API Key management | 10 req/min |

---

## Reference Docs

- Full API spec: `docs/SaaS-API.md` (all fields and response examples)
- Kafka spec: `docs/API & Kafka 規範.md` (internal compute node protocol — not needed for typical integrations)

> This Skills file applies to the **Orbie** platform.
