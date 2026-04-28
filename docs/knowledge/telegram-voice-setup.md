# Telegram Voice (TTS) Setup

How to make the OpenClaw agent reply with voice notes on Telegram, including which provider to choose, full config, and the **traps that silently break voice delivery** (we lost hours debugging these — read the gotchas section before changing anything).

---

## What "voice on Telegram" actually means

Two directions:

- **Inbound (user → agent):** user records a Telegram voice note → OpenClaw transcribes via Whisper (`tools.media.audio.models`) → text reaches the agent. Always works once Whisper is configured.
- **Outbound (agent → user):** agent's reply text → ElevenLabs/OpenAI TTS synth → file saved to `/tmp/openclaw/tts-*/voice-*.opus` → OpenClaw runtime calls Telegram `sendVoice` API → user hears a real voice note (not a file attachment).

This document is mostly about **outbound**, because that's where the breakage happens.

---

## Step 1 — Ask the user which provider

When the user enables voice replies on Telegram, ask:

> "Jakiego providera TTS chcesz użyć?
> - **ElevenLabs** (zalecany — najlepsza jakość, polski głos brzmi naturalnie, możliwość klonowania własnego głosu; płatny)
> - **OpenAI TTS** (taniej, dostępny w ramach OpenAI API; mniej naturalny, ale wystarczający)"

Then:

- **ElevenLabs:** ask for `ELEVENLABS_API_KEY` and a `voiceId` (20-char alphanumeric, get from https://elevenlabs.io/app/voice-lab).
- **OpenAI:** reuse existing `OPENAI_API_KEY`; ask for voice (`alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`).

Validate the ElevenLabs voice ID with a curl ping before saving:

```bash
curl -s "https://api.elevenlabs.io/v1/voices/<VOICE_ID>" \
  -H "xi-api-key: <API_KEY>" | head -c 200
```

A 200 with `voice_id` in the body means it exists on the user's account. A 401/404 means the voice ID or key is wrong — fix before continuing.

---

## Step 2 — Config (`openclaw.json`)

This is the **complete working config** for outbound voice. Every field shown below is required for `sendVoice` to fire.

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy": "allowlist",
      "allowFrom": [123456789],
      "streaming": { "mode": "off" }
    }
  },

  "plugins": {
    "allow": ["telegram", "elevenlabs", "anthropic", "openai"],
    "entries": {
      "telegram": { "enabled": true },
      "elevenlabs": { "enabled": true }
    }
  },

  "messages": {
    "tts": {
      "enabled": true,
      "mode": "final",
      "auto": "inbound",
      "provider": "elevenlabs",
      "providers": {
        "elevenlabs": {
          "voiceId": "<VOICE_ID>"
        }
      }
    }
  },

  "tools": {
    "media": {
      "audio": {
        "maxBytes": 20971520,
        "models": [
          { "provider": "openai", "model": "whisper-1" }
        ]
      }
    }
  }
}
```

**Field meanings (all critical):**

| Field | Required value | Why |
|---|---|---|
| `messages.tts.enabled` | `true` | Master switch. |
| `messages.tts.mode` | `"final"` | TTS runs only on the final reply. The auto-delivery code path that calls `sendVoice` is **gated on `mode === "final"`**. Setting `"all"` looks tempting but actually disables the working path. |
| `messages.tts.auto` | `"inbound"` | Synth fires only when the user message was a voice note. With `"always"` the agent speaks back even on text — usually noisy. |
| `messages.tts.provider` | `"elevenlabs"` or `"openai"` | Picks the synth backend. |
| `plugins.entries.elevenlabs.enabled` | `true` | Without this the plugin does not register, the runtime falls back silently to OpenAI even though the config says ElevenLabs. |
| `channels.telegram.streaming.mode` | `"off"` | `"partial"` chunks the reply as it streams; in some versions this loses the audio attachment. Off is safe. |

The API key for ElevenLabs goes in env vars (`ELEVENLABS_API_KEY` / `XI_API_KEY`) on the VPS, never in the config.

---

## Step 3 — Tell the agent NOT to call the `tts` tool itself

This is the single biggest trap. The agent has access to a tool literally called `tts` whose description says:

> *"Convert text to speech. Audio is delivered automatically from the tool result — reply with NO_REPLY after a successful call to avoid duplicate messages."*

Sounds right. **It is wrong for Telegram.** When the agent calls `tts` manually:

1. Audio is generated and saved to disk (`/tmp/openclaw/tts-*/voice-*.opus`)
2. The toolResult contains `audioPath` and `audioAsVoice: true`
3. The Telegram channel's `sendToolResult` path **does not call `sendVoice`** for tool-attached media — only the auto-synthesis path (in `dispatch-lUjTuuga.js`, gated on `mode === "final"` and triggered by an inbound voice note) actually invokes `sendVoice`
4. Result: file lives on the server, user gets nothing — or just a `NO_REPLY` placeholder

**Fix — add this to `SOUL.md` in the workspace:**

```markdown
## Voice replies (CRITICAL)

NIE wywołuj narzędzia `tts` ręcznie. Kiedy użytkownik napisze do Ciebie głosówką,
odpowiadaj **zwykłym tekstem** — runtime ma ustawione `auto: "inbound"`,
`mode: "final"` i sam zsyntetyzuje Twoją odpowiedź jako głosówkę przez ElevenLabs
i wyśle do Telegrama.

Ręczny `toolCall: tts` blokuje tę automatyczną ścieżkę dostawy — plik audio
powstaje, ale nigdy nie wychodzi do użytkownika. Jeśli odbiorca napisał tekstem,
też odpowiadaj tekstem.
```

The Polish wording is intentional — Anatol's persona is Polish; phrase it in the same language so the rule lands.

---

## Step 4 — Verify end-to-end

After config is written and the gateway restarts, do this **before** declaring it done:

1. Have the user record a short Telegram voice note ("Anatol, słyszysz mnie?")
2. SSH to the VPS:
   ```bash
   ssh root@<VPS> "docker exec <CONTAINER> cat /tmp/openclaw/openclaw-$(date -u +%Y-%m-%d).log" \
     | grep -E "sendVoice|sendMessage|sendAudio"
   ```
3. The line you must see:
   ```
   [gateway/channels/telegram] telegram sendVoice ok chat=<ID> message=<N>
   ```
   If you only see `sendMessage`, voice is broken — see gotchas below.

4. Also verify the audio file exists and is a valid Ogg Opus:
   ```bash
   ssh root@<VPS> "docker exec <CONTAINER> ls -lat /tmp/openclaw/ | head -5"
   ssh root@<VPS> "docker exec <CONTAINER> file /tmp/openclaw/tts-*/voice-*.opus | tail -1"
   # expected: Ogg data, Opus audio, version 0.1, mono, 48000 Hz
   ```

If the file is valid Opus and `sendVoice` did not fire, the bug is in delivery, not TTS — re-read Step 3.

---

## Gotchas we hit (in order of how badly they cost us)

1. **`/data/.openclaw/settings/tts.json` overrides `openclaw.json`.**
   User prefs file silently wins over config. If a user once tested with `tts setProvider openai`, that pref is now permanent and the config-level `provider: "elevenlabs"` is ignored. Always check this file when "the wrong provider keeps being used":
   ```bash
   cat /data/.openclaw/settings/tts.json
   # Wanted: {"tts": {"auto": "inbound"}}
   # Bad:    {"tts": {"auto": "always", "provider": "openai"}}
   ```
   Strip the `provider` field if you want config to take effect.

2. **Agent calls `tts` tool manually → no `sendVoice`.**
   See Step 3. The fix is a SOUL.md rule, not config.

3. **`mode: "all"` looks more permissive but breaks voice.**
   The auto-synth-and-`sendVoice` path is gated on `mode === "final"`. Counterintuitive — leave it `"final"`.

4. **`streaming.mode: "partial"` may drop the audio attachment.**
   We saw this empirically; setting `"off"` made things behave. If you need streaming text in some other channel, scope `streaming` only to that channel.

5. **`/model GLM` (or any small model) can break tool-calling and slow replies to 60–80s.**
   GLM 5.1 took 78 seconds on a 126k-token context and never emitted the right delivery directive. Sonnet 4.6 handles it in 4–8s and triggers correctly. If TTS suddenly stops working, run `cat /data/.openclaw/agents/*/sessions/sessions.json | grep -E "modelOverride"` and remove any user-set override.

6. **Huge un-compacted sessions slow everything down.**
   `compactionCount: 0` on a 1MB+ JSONL session = every reply re-processes the whole context. The user should `/compact` or `/new` periodically. This isn't a voice bug, but it amplifies any latency and looked like "TTS is slow" to the user.

7. **Anatol can hallucinate fake providers/commands.**
   When debugging in-band, the agent tried to invent providers like `papla.media` and commands like `/tts off`. Verify any agent-suggested fix against the actual codebase before applying. The agent does not have access to its own runtime source — Claude Code does (read from `/usr/local/lib/node_modules/openclaw/dist/`).

8. **`FailoverError: Unknown model: openai/Claude Sonnet 4.6` in logs.**
   Cosmetic — the slug-generator subsystem mis-prefixes the alias. Doesn't block voice. Fix later by ensuring aliases include the provider prefix correctly.

---

## What logs to read when voice "doesn't work"

Three places, in order:

1. **`/tmp/openclaw/openclaw-YYYY-MM-DD.log`** — gateway log. Search for `tts`, `voice`, `audio`, `sendVoice`, `sendMessage`, `error`. The structured JSON entries mean you need to grep with `python3 -c "import json,sys; ..."` or `jq`.

2. **`/data/.openclaw/agents/main/sessions/<sessionId>.jsonl`** — the conversation tape. You'll see exactly what the agent did: did it call `tts` tool? did it return `NO_REPLY` or `[[audio_as_voice]]`? did the toolResult contain `audioAsVoice: true`?

3. **`/data/.openclaw/agents/main/sessions/sessions.json`** — current session state per channel/user. Has `modelOverride`, `compactionCount`, `lastTo`. If `modelOverride` is set to anything other than the default, that's why the agent is misbehaving.

---

## Quick Q&A flow for the setup agent

When the user says "I want voice replies on Telegram":

1. "Jakiego providera? **ElevenLabs** (jakość, klon głosu) czy **OpenAI** (tańszy)?"
2. (ElevenLabs) "Wklej `ELEVENLABS_API_KEY` — klucz jest w https://elevenlabs.io/app/settings/api-keys."
3. (ElevenLabs) "Wklej `voiceId` — np. z https://elevenlabs.io/app/voice-lab po wybraniu głosu znajdziesz go w detalach (20 znaków alfanumerycznych)."
4. Validate the voice ID with curl (see Step 1).
5. Write config (see Step 2). Set `streaming: { mode: "off" }` and double-check `mode: "final"`, `auto: "inbound"`.
6. Append the SOUL.md rule (Step 3).
7. Restart gateway.
8. Run the verification (Step 4) — **don't trust "działa" without seeing `sendVoice` in the log**. Users sometimes interpret a fast text reply as "voice works".

If at any point voice doesn't arrive, walk down the gotchas list 1→8 in order.
