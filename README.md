# pine-voice

Make real phone calls from your OpenClaw agent using [Pine AI](https://19pine.ai)'s voice agent.

**No plugin required.** This is a standalone skill that calls Pine Voice's REST API directly via curl.

## Install

### Via ClawHub

```bash
clawhub install pine-voice
```

### Manual

Copy `skills/pine-voice/SKILL.md` into `~/.openclaw/skills/pine-voice/SKILL.md` (global) or `./skills/pine-voice/SKILL.md` (workspace).

## Setup

1. Sign up at [19pine.ai](https://19pine.ai) and upgrade to Pro
2. Get your credentials via the email auth flow (see SKILL.md for details)
3. Set environment variables:
   ```bash
   export PINE_ACCESS_TOKEN="your_token"
   export PINE_USER_ID="your_user_id"
   ```
4. Ask your agent: *"Call +14155551234 and make a reservation for 4 at 7pm"*

## What it does

- Calls real phone numbers via Pine AI's voice agent
- Handles IVR navigation, hold times, and extended conversations (up to 2 hours)
- Negotiator mode for bill reduction, rate matching, fee waivers
- Returns full transcripts with speaker labels
- 93% success rate across 50,000+ users

## Skill-only vs Plugin

| | This skill | openclaw-pine-voice plugin |
|---|---|---|
| Install | Copy a file | `openclaw plugins install` + config |
| Config | Env vars only | Edit `openclaw.json` + `tools.allow` |
| Wait mechanism | Agent polls via curl | Native SSE/polling (zero LLM cost) |
| Best for | Quick setup, ClawHub | Heavy usage, cost-sensitive |

For frequent use, the [plugin](https://github.com/19PINE-AI/openclaw-pine-voice) is more cost-efficient because `pine_voice_call_and_wait` blocks at the runtime level without burning LLM tokens during the wait.

## Links

- [Pine for Developers](https://pineclaw.com)
- [Voice API Docs](https://pineclaw.com/voice/api.html)
- [Pine AI](https://19pine.ai)
- [OpenClaw Plugin (alternative)](https://github.com/19PINE-AI/openclaw-pine-voice)

## License

MIT
