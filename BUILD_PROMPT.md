# One-shot build prompt

> Paste everything below the line into a fresh Claude session. It is self-contained.

---

Build **Spacecadet**, a meeting attention assistant, as **one self-contained `spacecadet.html` file**.

I open it from disk (`file://`) on a locked-down corporate Windows laptop. I cannot install anything —
no Node, no Python, no build step, no npm, no extensions. Everything must work from that one file.
Write the complete file, no placeholders and no "TODO" stubs.

I have a **Claude Pro subscription, not API credits.** Pro does not pay for API calls, so the app
must be fully useful with **no API key at all**. An API key is an optional upgrade, never a
requirement. Nothing may be gated behind it except the ambient auto-summary.

## What it's for

I lose focus in meetings. When I snap back I need to look at a second screen and know, in about two
seconds, what is being discussed *right now* — and critically, whether someone just asked me
something. Design every UI decision around a two-second glance by someone who is mildly panicking.

## Pipeline

    audio capture → speech-to-text → rolling timestamped buffer → periodic Claude call → dashboard

**Claude cannot accept audio.** The API is text/image/PDF only. Speech-to-text must happen in the
browser; only text is ever sent to the API. Do not attempt to send audio to Anthropic.

Run two independent loops:

- **Fast loop (local, free, instant):** regex-scan every incoming transcript fragment for my name
  and question patterns. Fires the "you're being spoken to" alert with no network round-trip.
- **Slow loop (optional, Claude):** topic, bullets, decisions, action items, talking points. Either
  ambient every 90s with an API key, or on-demand via clipboard handoff to claude.ai with a Pro
  subscription. See Summarizer backends.

Keep them decoupled. The fast loop must still work when the API is down, no key was ever entered,
or local-only mode is on — which is the default state, so this is the normal case, not an edge case.

## Audio capture — two sources, user-selectable

1. **Microphone** — `getUserMedia({audio: {echoCancellation: false, noiseSuppression: false, autoGainControl: true}})`.
   Echo cancellation must be **off**: it actively suppresses meeting audio coming from my own speakers.
2. **System / tab audio** — `getDisplayMedia({video: true, audio: true})`, then keep only the audio track.
   `video: true` is required — Chrome will not offer audio-only display capture. Stop the video track
   immediately after acquiring the stream to save CPU.

Detect and clearly report the case where the user shares a screen but leaves "share system audio"
unchecked — the resulting stream has no audio track. Say exactly that, don't fail silently.

## Speech-to-text — two engines behind one interface

Define a single interface both engines implement:

```js
// start(stream, onFragment) -> stop()
// onFragment({ text: string, isFinal: boolean, t: number /* epoch ms */ })
```

**Engine A — Web Speech API** (default; zero dependencies)
- `webkitSpeechRecognition`, `continuous = true`, `interimResults = true`, `lang = 'en-US'`.
- **Critical gotcha:** it stops on its own after silence. You MUST reattach in `onend` and restart,
  guarded by a `shouldBeRunning` flag so an intentional stop doesn't resurrect it. Without this the
  app silently dies after ~30 seconds and this is the single most common way this build fails.
- Handle `onerror` for `no-speech`, `audio-capture`, `not-allowed`, `network` distinctly.
- **Limitation to surface in the UI:** it only ever listens to the *microphone*. It cannot consume a
  `getDisplayMedia` stream. If the user selects system audio, this engine must be disabled with a
  visible explanation, not silently ignored.
- Note in the UI that this engine sends audio to Google's servers.

**Engine B — Local Whisper** (for system/tab audio, and for keeping audio on-machine)
- `transformers.js` from a CDN, `automatic-speech-recognition`, model `Xenova/whisper-tiny.en`.
- Try WebGPU, fall back to WASM.
- Buffer ~20–30s of audio via `AudioWorklet` or `ScriptProcessorNode`, resample to **16 kHz mono
  Float32** (Whisper requires this — use an `OfflineAudioContext` to resample), transcribe, emit.
- **Verify the current CDN URL and package name yourself** (the package was renamed from
  `@xenova/transformers` to `@huggingface/transformers`) — do not trust a remembered version string.
- The CDN may be blocked by corporate proxy. On any load failure, fail gracefully to Engine A with a
  clear message. Never leave the app in a broken state because a CDN was unreachable.
- Say plainly in the UI: with this engine, audio never leaves the machine.

Transcript quality from `whisper-tiny.en` is mediocre. That is acceptable and expected — Claude
compensates. Do not add error-correction logic.

## Summarizer backends — pluggable, no API key required by default

The user has a **Claude Pro subscription, not API credits.** Pro does not pay for API calls. So the
LLM must be optional, and the app must be genuinely useful without one.

Implement the summarizer as a swappable backend chosen in settings. **Default to `handoff`.**

### Backend 1 — `handoff` (DEFAULT, free, uses the Pro subscription)

No API key, no network calls from the app. On demand:

- **"Catch me up" button / `C` hotkey** builds a formatted prompt — the meeting context, the user
  profile, and the rolling transcript — copies it to the clipboard via `navigator.clipboard.writeText`,
  and opens `https://claude.ai/new` in a new tab. The user pastes. Their Pro subscription pays.
- Show a clear toast: "Copied. Paste into the Claude tab that just opened."
- Provide a visible fallback textarea with the text pre-selected, because clipboard writes can fail
  on a `file://` origin without a user-gesture chain.
- Optionally attempt `https://claude.ai/new?q=<encodeURIComponent(text)>` to prefill directly, but
  **treat that as unverified**: if the parameter is unsupported the user still has the clipboard copy,
  so never make the flow depend on it.

In this mode the app's real-time display is driven entirely by the local intelligence layer below.

### Backend 2 — `api` (optional, costs money)

Only if the user supplies an API key. Enables the ambient 90-second auto-summary. Exact request —
these headers are verified working from a `file://` origin:

```js
await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: {
    "content-type": "application/json",
    "x-api-key": apiKey,
    "anthropic-version": "2023-06-01",
    "anthropic-dangerous-direct-browser-access": "true"
  },
  body: JSON.stringify({
    model: selectedModel,              // default "claude-opus-5"
    max_tokens: 2000,
    system: SYSTEM_PROMPT,
    output_config: {
      effort: "low",                   // keep latency down; this is a live loop
      format: { type: "json_schema", schema: RESPONSE_SCHEMA }
    },
    messages: [{ role: "user", content: userContent }]
  })
})
```

Model dropdown: `claude-opus-5` (default), `claude-sonnet-5`, `claude-haiku-4-5`. Exact IDs, never
date-suffixed. Send **incrementally** — previous state plus only transcript since the last call,
never the whole window. Skip the call entirely if fewer than ~15 new words accumulated.

Surface running cost from `usage.input_tokens` / `usage.output_tokens` so the user can see what the
ambient mode actually costs them.

### Backend 3 — `local` (nothing leaves the machine)

Local layer only. No clipboard handoff, no API. One toggle, one choke-point function.

**Architectural requirement:** all three backends feed the same rendering code. The UI must not know
which backend produced the state it is displaying.

## The local intelligence layer — always on, free, instant

This carries the app in `handoff` mode, so build it properly rather than as a stub. All of it is
plain JS over the transcript buffer, no network:

- **Live transcript tail.** The last ~30 seconds of speech, rendered large. This alone often answers
  "what are they talking about" — reading three recent sentences is frequently enough. Do not
  underrate this; it is the highest value-per-line feature in the app.
- **Keyword chips.** Dominant terms over the window, scored by frequency against a stopword list and
  damped by how common the term has been across the whole meeting (a crude TF-IDF). Render as chips.
  Three or four salient nouns communicate the subject at a glance.
- **Topic-change detection.** Build a term-frequency vector per ~60s window; compute cosine
  similarity between consecutive windows; when it drops below a tunable threshold (~0.35), flash
  "TOPIC CHANGED" and push an entry onto the topic timeline. Works surprisingly well and costs nothing.
- **Name and question alerts.** Regex over incoming fragments for configurable name variants, plus
  patterns like "what do you think", "any thoughts", "over to you", "can you", "how about you".
  This is the most time-critical feature in the app and must never depend on a backend.

### RESPONSE_SCHEMA — used by the `api` backend

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["topic","topic_changed","bullets","user_addressed","talking_points","decisions","actions","glossary"],
  "properties": {
    "topic": { "type": "string" },
    "topic_changed": { "type": "boolean" },
    "bullets": { "type": "array", "items": { "type": "string" } },
    "user_addressed": {
      "type": ["object","null"],
      "additionalProperties": false,
      "required": ["asked_by","question","kind"],
      "properties": {
        "asked_by": { "type": "string" },
        "question": { "type": "string" },
        "kind": { "type": "string", "enum": ["direct_question","mentioned","assigned"] }
      }
    },
    "talking_points": { "type": "array", "items": { "type": "string" } },
    "decisions": { "type": "array", "items": { "type": "string" } },
    "actions": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["text","owner","mine"],
        "properties": {
          "text": { "type": "string" },
          "owner": { "type": "string" },
          "mine": { "type": "boolean" }
        }
      }
    },
    "glossary": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["term","definition"],
        "properties": { "term": { "type": "string" }, "definition": { "type": "string" } }
      }
    }
  }
}
```

### SYSTEM_PROMPT — used by both the `api` and `handoff` backends

Use this text, with the user profile and pre-meeting context interpolated in. In `handoff` mode
the same text is prepended to the clipboard payload so the pasted prompt behaves identically:

```
You help someone who has lost focus in a live meeting re-enter the conversation quickly.

You receive your own previous state, plus a new chunk of rough speech-to-text transcript.
The transcript is machine-generated: expect misheard words, missing punctuation, no speaker
labels, and dropped words. Infer intent. Never quote transcription errors back verbatim.

USER PROFILE:
{profile}

MEETING CONTEXT (agenda, attendees, projects — may be empty):
{context}

Rules:
- topic: 3-8 words naming what is being discussed at the END of the chunk. Not a summary of the
  whole chunk — what they are on RIGHT NOW.
- topic_changed: true only if meaningfully different from the previous topic. Ignore drift.
- bullets: 3-5 bullets covering the recent window, most recent first, each under 20 words.
  Plain language. No preamble, no "the team discussed".
- user_addressed: non-null ONLY if the user was directly asked something, assigned something, or
  named in a way that expects a response. Someone merely saying their name in passing is
  "mentioned". Be conservative — a false alarm is worse than a miss here.
- talking_points: only when user_addressed is non-null. 2-3 short things the user could actually
  say, grounded in what was discussed. Not generic filler.
- decisions / actions / glossary: only NEW items from this chunk. Never repeat items you have
  already reported. Empty arrays are correct and expected most of the time.
- glossary: only genuinely non-obvious acronyms, jargon, or proper nouns.
- If the chunk is silence, filler, or crosstalk with no substance: repeat the previous topic
  unchanged and return empty arrays.
```


## UI

Dark, high contrast, large type. Three zones, in visual priority order:

1. **NOW** — the current topic in very large text (~48px+), unmissable. Plus a subtle "updated 12s
   ago" and a flash animation when `topic_changed` fires. This is the two-second-glance element.
2. **ALERT** — when `user_addressed` is set, or the local regex fires: a loud banner naming who
   asked, what they asked, how long ago, and the talking points. Optional soft audio chime
   (default off). This must be visually dominant when active and absent when not.
3. **CONTEXT** — the live transcript tail (last ~30s, comfortably readable) and keyword chips,
   always present because they need no backend. When a summary exists, the bullets sit above them.
   Below: collapsible panels for Decisions, My Action Items, Glossary, and Topic Timeline.

Persistent slim status bar: STT engine, capture source, active summarizer backend, connection
state, and — only in `api` mode — last-call latency and running session cost.

## Features

Implement all of these:

- **Rolling buffer** — timestamped fragments, prune older than the window. Window configurable
  5–20 min, default 10.
- **Catch me up** — button and `C` hotkey. In `handoff` mode: build the prompt, copy it, open
  claude.ai. In `api` mode: one richer call over the *entire* buffer rather than the incremental
  delta, rendered in a modal. Same button, same hotkey, backend-appropriate behaviour.
- **While you were away** — track `document.visibilityState`. When the tab regains focus, show
  "3 topics changed while you were away" with the topics I missed.
- **Local name alert (fast loop)** — configurable list of name variants. Regex over incoming
  fragments, plus patterns like "what do you think", "any thoughts", "over to you", "can you".
  Fires instantly, independent of the API.
- **Decisions & my action items** — accumulate for the whole meeting, not just the window.
  Action items assigned to me are visually distinct.
- **Topic timeline** — list of topics with start times, so I can see the arc of the meeting.
- **Glossary** — accumulating acronym/jargon decoder.
- **Export** — download the full meeting as Markdown (transcript + topics + decisions + actions)
  via a `Blob` + object URL. Must work from `file://`.
- **Cost meter** — `api` mode only. Accumulate `usage.input_tokens` / `usage.output_tokens`, price
  per selected model, show running session total. Hidden entirely in the other backends, where the
  honest number is zero.
- **Backend switch** — a prominent three-way control: `handoff` (default) / `api` / `local`. Every
  outbound call routes through a single choke-point function, so the network boundary is auditable
  at a glance. `local` hard-disables that function.
- **Discreet mode** — compact low-key layout, and an `Esc` panic-hide that instantly blanks the
  screen to something innocuous. Second `Esc` restores.
- **Settings panel** — summarizer backend, capture source, STT engine, window size, name variants,
  user profile (free text), pre-meeting context (free text, for agenda/attendees). API key, model
  and call interval appear only when the `api` backend is selected — do not show a key field to
  someone who will never use one.
- **Persistence** — all settings plus in-flight meeting state to `localStorage`, auto-restored on
  reload so an accidental refresh mid-meeting is not fatal.
- **Resilience** — API failures retry with exponential backoff, never kill the capture loop, and
  surface as a status-bar state rather than an alert box.

## Non-goals — do not build these

Speaker diarization. Meeting-platform integrations or bots. Server components. Auth. Cloud storage.
Multi-user anything. A build step or package manager.

## Constraints

- One file. Inline all CSS and JS. The only permitted network calls at runtime are the optional
  Anthropic API, the optional transformers.js CDN, and opening a claude.ai tab in `handoff` mode.
- The app must be fully functional with no API key. If a key is used it lives in `localStorage`,
  entered at runtime, never hardcoded — warn that a `file://` page offers no real secret protection
  and that a spend-limited personal key is advised.
- Must run from `file://` in Chrome/Edge on Windows. Note in a comment that Chrome on macOS cannot
  capture system audio at all (tab audio only).
- Plain vanilla JS. No frameworks, no bundler, no CDN UI libraries.
- Comment the non-obvious parts — especially the `onend` restart loop, the 16 kHz resampling, and
  the incremental-summarization state machine.

## Priority, if you must cut

Core, never cut: capture → STT → buffer → local intelligence layer → NOW zone → ALERT zone →
`handoff` catch-me-up → settings → backend switch.
Cut first, in this order: glossary, cost meter, while-you-were-away, export, topic timeline. Cut the
entire `api` backend before cutting anything in the local layer — the app has to earn its keep with
no key at all.

## Acceptance checklist

Before you finish, verify each:

1. Web Speech engine survives 5+ minutes of intermittent silence without dying (`onend` restart).
2. Selecting system audio disables the Web Speech engine with a visible explanation.
3. A display-capture stream with no audio track produces a clear, specific error.
4. Silence produces no API calls.
5. **With no API key ever entered, the app is fully usable**: live transcript tail, keyword chips,
   topic-change flash, name alerts, and a working catch-me-up handoff.
6. Catch-me-up in `handoff` mode copies real text to the clipboard and opens claude.ai; the visible
   fallback textarea works when the clipboard write is refused.
7. Local-only mode makes zero network requests — verifiable in the Network tab.
8. `Esc` hides the screen instantly.
9. Reloading mid-meeting restores state.
10. A malformed or failed API response does not stop audio capture.
11. The current topic is readable from arm's length, at a glance.
