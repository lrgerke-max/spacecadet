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

## Subscription vs. API — the billing reality

**A Claude Pro subscription does not include API access.** They are separately billed products.
Pro covers claude.ai and Claude Code; `api.anthropic.com` draws on pay-as-you-go credits purchased
at console.anthropic.com. No header, endpoint, or auth mode makes a Pro subscription pay for API
calls.

Claude Code spends a subscription via OAuth (`ant auth login` → bearer token + the
`oauth-2025-04-20` beta header), but that requires the CLI installed locally — impossible on a
browser-only machine, so it is moot here.

**Consequence: the LLM must be optional.** The app ships three summarizer backends:

| Backend | Cost | How |
|---|---|---|
| `handoff` (default) | free | Builds the prompt, copies to clipboard, opens claude.ai. Pro pays. |
| `api` | see below | Ambient 90s auto-summary. Needs credits. |
| `local` | free | Nothing leaves the machine. |

All three feed identical rendering code; the UI never knows which produced the state.

## The local layer does more than expected

Because the default backend is on-demand rather than ambient, the free local layer carries the app
between handoffs — and it is more capable than it first appears:

- **Live transcript tail.** The last ~30s in large type. Reading three recent sentences usually
  answers "what are they talking about" outright. Highest value-per-line feature in the app.
- **Keyword chips.** Crude TF-IDF over the window. Three salient nouns communicate a subject fast.
- **Topic-change detection.** Cosine similarity between consecutive 60s term-frequency vectors;
  below ~0.35, flash. No model required.
- **Name and question alerts.** Regex. The most time-critical feature, and it never touches a network.

The genuinely hard problem was always speech-to-text, not summarization — which is why removing the
LLM costs less than intuition suggests.

## Cost, if the `api` backend is enabled

At a 90-second cycle, ~700 input / ~250 output tokens per call, ~40 calls/hour:

| Model | ≈ per hour |
|---|---|
| `claude-sonnet-5` (default) | ~$0.16 — intro pricing, through 2026-08-31 |
| `claude-sonnet-5` | ~$0.24 — standard rate thereafter |
| `claude-opus-5` | $0.50 – $0.80 |
| `claude-haiku-4-5` | ~$0.08 |

Silence triggers no calls, so real usage lands lower. Sonnet 5 is the chosen default: comfortably
capable for rolling summarization at roughly a third of Opus pricing.

## Discretion

This is a tool you use *during* a meeting, on a screen other people may see. Compact mode, a
panic-hide hotkey, and neutral visual styling are functional requirements, not polish. If you
capture system audio by sharing your whole screen, and you are *also* screen-sharing into the
meeting, this app is visible to everyone. Prefer a second monitor.

## Consent

Recording colleagues is a policy question at most companies and a legal one in two-party-consent
jurisdictions. Nothing here is stored off-machine except transcript text sent to the API, but that
does not settle the question. Worth five minutes with your handbook.
