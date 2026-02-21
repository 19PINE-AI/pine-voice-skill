---
name: pine-voice
description: Give your agent a real phone. It dials, waits on hold, negotiates your bills, and returns a full transcript.
metadata:
  { "openclaw": { "emoji": "📞" } }
command-tool: exec
---

# Pine Voice (Skill-Only)

Make real phone calls via Pine AI's voice agent. The agent calls the specified number, navigates IVR systems, handles verification, conducts negotiations, and returns a full transcript.

This skill uses Pine Voice's REST API directly via `exec` (curl). No plugin installation, no `tools.allow` configuration needed.

## Prerequisites

- A **Pine AI account** with a Pro subscription — sign up at https://19pine.ai

## Authentication

Credentials are stored in `~/.pine-voice/credentials.json`. Before making any call, check if this file exists:

```bash
cat ~/.pine-voice/credentials.json 2>/dev/null || echo "NOT_AUTHENTICATED"
```

If the file exists and contains valid JSON with `access_token` and `user_id`, skip to **How to make a call**.

If the file does not exist or prints `NOT_AUTHENTICATED`, run the authentication flow below. **Ask the user for their Pine AI account email to begin.**

### Step 1: Request a verification code

```bash
curl -s -X POST https://www.19pine.ai/api/v2/auth/email/request \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL_HERE"}'
```

This returns `{"data": {"request_token": "TOKEN"}}`. Save the `request_token` — you will need it in the next step.

Tell the user: *"A verification code has been sent to your email. Please check your inbox (and spam folder) and give me the code."*

### Step 2: Verify the code

Once the user provides the code:

```bash
curl -s -X POST https://www.19pine.ai/api/v2/auth/email/verify \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL_HERE", "request_token": "TOKEN_FROM_STEP_1", "code": "CODE_FROM_USER"}'
```

This returns `{"data": {"access_token": "...", "id": "..."}}`.

### Step 3: Save credentials

Store the credentials locally so they persist across sessions:

```bash
mkdir -p ~/.pine-voice && cat > ~/.pine-voice/credentials.json << 'ENDCRED'
{"access_token": "ACCESS_TOKEN_HERE", "user_id": "USER_ID_HERE"}
ENDCRED
chmod 600 ~/.pine-voice/credentials.json
```

Replace `ACCESS_TOKEN_HERE` with the `access_token` value and `USER_ID_HERE` with the `id` value from the verify response.

Tell the user: *"Authentication successful! Your credentials have been saved. You can now make phone calls."*

## When to use

Use this skill when the user wants you to **make a phone call** on their behalf. The Pine AI voice agent will call the specified number and handle the conversation autonomously.

**Important:** The voice agent can only speak English. Supported countries: US/CA/PR (+1), UK (+44), AU (+61), NZ (+64), SG (+65), IE (+353), HK (+852).

## Best for

- Calling customer service to negotiate bills, request credits, or resolve issues
- Scheduling meetings or appointments by phone
- Making restaurant reservations
- Calling businesses to inquire about services or availability
- Following up with contacts on behalf of the user

## How to make a call

### Step 1: Load credentials

```bash
cat ~/.pine-voice/credentials.json
```

Parse the JSON to extract `access_token` and `user_id`. If the file is missing, run the **Authentication** flow above first.

### Step 2: Gather all required information

Before calling, you **must** collect every piece of information the callee might need. The voice agent **cannot ask a human for missing information during the call**. Anticipate what will be required: authentication details, payment info, negotiation targets, relevant context.

### Step 3: Initiate the call

```bash
curl -s -X POST https://agent3-api-gateway-staging.19pine.ai/api/v2/voice/call \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "X-Pine-User-Id: USER_ID" \
  -d '{
    "dialed_number": "+14155551234",
    "callee_name": "Comcast Customer Service",
    "callee_context": "Cable and internet provider. Account holder: Jane Doe, account #12345. Calling to negotiate monthly bill.",
    "call_objective": "Negotiate monthly bill down to $50/mo. Do not accept above $65/mo.",
    "detailed_instructions": "Mention 10-year customer loyalty. If no reduction, ask for retention department.",
    "caller": "negotiator",
    "voice": "female",
    "max_duration_minutes": 60,
    "enable_summary": false
  }'
```

Replace `ACCESS_TOKEN` and `USER_ID` with the values from `~/.pine-voice/credentials.json`.

This returns `{"call_id": "..."}`. The call is now **active** — Pine's voice agent has dialed and is on the line.

### Step 4: Poll for results

Poll every 30 seconds until the call completes:

```bash
curl -s https://agent3-api-gateway-staging.19pine.ai/api/v2/voice/call/CALL_ID_HERE \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "X-Pine-User-Id: USER_ID"
```

Terminal statuses: `completed`, `Completed`, `HungupByPeer`, `HungupByLocal`, `Timeout`, `TooLongSilence` (all mean done), `failed`, `Failed`, `DialFailed`, `NoAnswer`, `Declined` (all mean failed), `cancelled`.

When complete, the response includes `transcript` (array of `{speaker, text}` entries), `summary` (if requested), `duration_seconds`, and `credits_charged`.

**IMPORTANT: Use `sessions_spawn` to run this in a background sub-agent** so you remain available to the user during the call (which can take 5-60+ minutes).

Example task for sessions_spawn:

> Make a phone call using curl to the Pine Voice API. First read credentials from ~/.pine-voice/credentials.json. POST to https://agent3-api-gateway-staging.19pine.ai/api/v2/voice/call with: dialed_number "+14155551234", callee_name "The Restaurant", callee_context "Italian restaurant, making a dinner reservation", call_objective "Reserve a table for 4 at 7pm tonight", caller "communicator". Then poll GET /api/v2/voice/call/{call_id} every 30 seconds until the status is terminal. Report the full transcript and outcome.

### Step 5: Evaluate the transcript

**Do NOT rely on the `status` field** to judge success. Read what the OTHER party actually said.

**Treat the call as a FAILURE if:**
- Only Pine's agent speaks and the other side is silent
- The other party's responses are automated/recorded (voicemail, IVR-only)
- Extended silence from both sides
- The callee hung up before the objective was discussed

## API parameters

### Create call (POST /api/v2/voice/call)

| Parameter | Required | Description |
|---|---|---|
| `dialed_number` | Yes | Phone number in E.164 format (e.g. `+14155551234`) |
| `callee_name` | Yes | Name of the person or business |
| `callee_context` | Yes | All context the agent needs: who they are, auth details, verification info |
| `call_objective` | Yes | Specific goal with targets and constraints |
| `detailed_instructions` | No | Strategy, approach, behavioral instructions |
| `caller` | No | `"negotiator"` (default) or `"communicator"` |
| `voice` | No | `"male"` or `"female"` (default: `"female"`) |
| `max_duration_minutes` | No | 1-120 (default: 120) |
| `enable_summary` | No | `true`/`false` (default: `false`) |

### Get call status (GET /api/v2/voice/call/{call_id})

Returns: `call_id`, `status`, `duration_seconds`, `transcript` (when complete), `summary` (if requested), `credits_charged`.

## Negotiation calls

For negotiations, set `caller` to `"negotiator"` and provide a thorough strategy:

- **Target outcome**: "Reduce monthly bill to $50/mo"
- **Acceptable range**: "Will accept up to $65/mo"
- **Hard constraints**: "Do not change plan tier"
- **Leverage points**: "10-year customer, competitor offers $45/mo"
- **Fallback**: "Request one-time credit of $100"
- **Walk-away**: "Ask for retention department"

## Examples

**Test call:**
"Call my phone at +1XXXXXXXXXX. Tell me that Pine Voice is set up and working."

**Restaurant reservation:**
"Call +14155559876 and make a reservation for 4 tonight at 7pm. If unavailable, try 7:30 or 8pm. Name: Jane Doe."

## Model requirements

Pine Voice works best with models that have thinking/reasoning capabilities.

- **Recommended:** Claude Sonnet/Opus 4.5+, GPT-5.2+, Gemini 3 Pro
- **Not recommended:** Gemini 3 Flash, or models without thinking capabilities
