# Spacecadet — design notes

A single-file, no-install browser app that listens to a meeting and tells you, at a glance,
what is being discussed right now — for when you've drifted and need to re-enter the conversation
without admitting you left it.

## The one constraint that shapes everything

**Claude has no audio input.** The Messages API accepts text, images and PDFs only. So this is
necessarily two-stage: something converts speech to text, *then* Claude reasons over the text.
Speech-to-text is the hard part of this project. Summarizing is the easy part.

## Verified facts (checked, not assumed)

| Claim | Status |
|---|---|
| `api.anthropic.com` returns `access-control-allow-origin: *` | ✅ verified via OPTIONS preflight |
| `anthropic-dangerous-direct-browser-access` accepted in preflight | ✅ verified |
| A `file://` page (Origin: `null`) can call the API | ✅ implied by `*` + verified header list |
| Claude API accepts audio input | ❌ **No.** text / image / document only |
| Web Speech API can consume a `getDisplayMedia` stream | ❌ **No.** microphone only |
| Structured output syntax | ✅ `output_config: {format: {type: "json_schema", schema: …}}` |

## STT engine trade-off

| Engine | Install | Captures remote meeting audio | Audio leaves laptop |
|---|---|---|---|
| Web Speech API (`webkitSpeechRecognition`) | none | ❌ mic only | ✅ yes → Google |
| Whisper WASM (`transformers.js`) | ~40 MB one-time CDN fetch | ✅ yes | ❌ no |
| Cloud STT (Deepgram etc.) | none | ✅ yes | ✅ yes → vendor |

**Decision: ship both local engines, pick at runtime.** Web Speech is the zero-dependency path and
the fallback when the CDN is blocked. Whisper is the correct path for remote meetings and the only
one that keeps audio on the machine.

Note the asymmetry that matters for policy: Web Speech sends **audio to Google**. Whisper sends
**nothing** — only the resulting text goes to Anthropic. "Local-only mode" is therefore only truly
local when paired with the Whisper engine.

## Architecture

    capture (mic | display) → STT engine → rolling buffer (timestamped) → every 90s:
      { previous state + new text only } → Claude → structured JSON → dashboard

Summarization is **incremental**: each call sends the previous summary plus only the new transcript,
never the whole window. Cheaper, faster, and the summary stays stable instead of rewriting itself
every cycle.

Two loops run at different speeds:
- **Fast, local, free:** regex name-detection on every incoming transcript fragment → instant alert.
- **Slow, remote, paid:** the 90s Claude cycle → topic, bullets, decisions, talking points.

The fast loop is what saves you when someone asks you a direct question; the slow loop is what
tells you what's going on. Do not couple them.

## Platform caveats

- **macOS: Chrome cannot capture system audio.** Tab audio only. Remote meetings held in the
  *desktop* Teams/Zoom app are therefore uncapturable on a Mac; the browser-tab versions are fine.
  On Windows, whole-screen share exposes a "share system audio" checkbox and works.
- `file://` permissions do not persist — expect to re-grant mic access each launch.
- Corporate proxies may block `api.anthropic.com` or `cdn.jsdelivr.net`. Phase 0 checks both.

## Cost

At a 90-second cycle, ~700 input / ~250 output tokens per call, ~40 calls/hour:

| Model | ≈ per hour |
|---|---|
| `claude-opus-5` | $0.50 – $0.80 |
| `claude-haiku-4-5` | ~$0.08 |

Opus 5 is the default for summary quality; the model dropdown makes switching a one-click
experiment. Set a spend limit on the key regardless.

## Discretion

This is a tool you use *during* a meeting, on a screen other people may see. Compact mode, a
panic-hide hotkey, and neutral visual styling are functional requirements, not polish. If you
capture system audio by sharing your whole screen, and you are *also* screen-sharing into the
meeting, this app is visible to everyone. Prefer a second monitor.

## Consent

Recording colleagues is a policy question at most companies and a legal one in two-party-consent
jurisdictions. Nothing here is stored off-machine except transcript text sent to the API, but that
does not settle the question. Worth five minutes with your handbook.
